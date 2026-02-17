# Advanced Topics

## 21.1 Custom Response Classes

### Creating Custom Responses

FastAPI's default response serializes data through Pydantic then encodes it
with the standard `json` module. Custom response classes let you bypass parts
of that pipeline to control encoding, media type, headers, and performance.

```python
from fastapi import FastAPI
from fastapi.responses import (
    Response, JSONResponse, HTMLResponse,
    PlainTextResponse, FileResponse, StreamingResponse
)

app = FastAPI()

# Base Response — full manual control
@app.get("/raw")
async def raw_response():
    return Response(
        content=b'{"message": "raw bytes"}',
        status_code=200,
        media_type="application/json",
        headers={"X-Custom": "value"}
    )

# Plain JSON — skips Pydantic serialization
@app.get("/json")
async def json_response():
    return JSONResponse(
        content={"message": "hello"},
        status_code=200,
        headers={"X-Source": "custom"}
    )

# How to build your own response class
from starlette.responses import Response
import json

class PrettyJSONResponse(Response):
    """Indented JSON for human-readable APIs / debugging."""
    media_type = "application/json"

    def render(self, content) -> bytes:
        return json.dumps(
            content,
            indent=2,
            ensure_ascii=False,
            default=str          # Fallback: convert datetime, UUID, etc.
        ).encode("utf-8")

@app.get("/pretty", response_class=PrettyJSONResponse)
async def pretty_endpoint():
    return {"user": "Alice", "scores": [1, 2, 3]}
```

### ORJSONResponse

`orjson` is a Rust-backed JSON library — typically 2–10× faster than the
stdlib `json` module and natively handles `datetime`, `UUID`, `numpy`, and
`dataclasses`.

**Installation**:
```bash
pip install orjson
```

```python
from fastapi import FastAPI
from fastapi.responses import ORJSONResponse
from datetime import datetime
from uuid import UUID, uuid4
import orjson

app = FastAPI(default_response_class=ORJSONResponse)  # Apply globally

# Per-endpoint override
@app.get("/items", response_class=ORJSONResponse)
async def list_items():
    return {
        "items": [{"id": 1, "name": "Widget"}],
        "generated_at": datetime.utcnow(),   # orjson serializes natively
        "request_id": uuid4()                 # UUID too
    }

# Custom orjson options
class CustomORJSONResponse(ORJSONResponse):
    def render(self, content) -> bytes:
        return orjson.dumps(
            content,
            option=(
                orjson.OPT_INDENT_2          |   # Pretty print
                orjson.OPT_UTC_Z             |   # Use Z suffix for UTC
                orjson.OPT_NON_STR_KEYS      |   # Allow int dict keys
                orjson.OPT_SERIALIZE_NUMPY       # Serialize numpy arrays
            )
        )

# Benchmark comparison
import time

def benchmark():
    data = {"items": [{"id": i, "ts": datetime.utcnow()} for i in range(1000)]}

    # stdlib json
    start = time.perf_counter()
    for _ in range(10000):
        json.dumps(data, default=str)
    stdlib_time = time.perf_counter() - start

    # orjson
    start = time.perf_counter()
    for _ in range(10000):
        orjson.dumps(data)
    orjson_time = time.perf_counter() - start

    print(f"stdlib: {stdlib_time:.3f}s | orjson: {orjson_time:.3f}s | "
          f"speedup: {stdlib_time/orjson_time:.1f}x")
```

### UJSONResponse

`ujson` (Ultra JSON) is a C extension that's faster than stdlib and easier to
integrate than orjson when you need broader type support.

**Installation**:
```bash
pip install ujson
```

```python
from fastapi import FastAPI
import ujson
from starlette.responses import Response

app = FastAPI()

class UJSONResponse(Response):
    media_type = "application/json"

    def render(self, content) -> bytes:
        return ujson.dumps(
            content,
            ensure_ascii=False,
            encode_html_chars=True,    # Escape HTML characters safely
            escape_forward_slashes=False,
            indent=0
        ).encode("utf-8")

@app.get("/fast", response_class=UJSONResponse)
async def fast_endpoint():
    return {"message": "Ultra-fast JSON", "items": list(range(100))}
```

### MessagePack Response

MessagePack is a binary serialization format that's smaller and faster than
JSON — ideal for internal microservice communication.

**Installation**:
```bash
pip install msgpack
```

```python
import msgpack
from starlette.responses import Response
from fastapi import FastAPI, Request

app = FastAPI()

class MessagePackResponse(Response):
    """Binary MessagePack response — smaller + faster than JSON."""
    media_type = "application/msgpack"

    def render(self, content) -> bytes:
        return msgpack.packb(content, use_bin_type=True)

def accepts_msgpack(request: Request) -> bool:
    return "application/msgpack" in request.headers.get("Accept", "")

@app.get("/data")
async def smart_response(request: Request):
    data = {"users": [{"id": 1, "name": "Alice"}], "total": 1}

    # Content negotiation: respond in format client prefers
    if accepts_msgpack(request):
        return MessagePackResponse(data)
    return data   # Falls back to JSON

# Client that speaks MessagePack
import httpx

async def msgpack_client():
    data = {"name": "Alice", "age": 30}
    packed = msgpack.packb(data, use_bin_type=True)

    async with httpx.AsyncClient() as client:
        response = await client.post(
            "http://localhost:8000/data",
            content=packed,
            headers={
                "Content-Type": "application/msgpack",
                "Accept": "application/msgpack"
            }
        )
    return msgpack.unpackb(response.content, raw=False)
```

### Custom Serialization

```python
from fastapi import FastAPI
from fastapi.responses import Response
from pydantic import BaseModel
from datetime import datetime
from decimal import Decimal
from uuid import UUID
import json

app = FastAPI()

class EnhancedJSONEncoder(json.JSONEncoder):
    """Handle Python types that stdlib json can't serialize."""
    def default(self, obj):
        if isinstance(obj, datetime):
            return obj.isoformat()
        if isinstance(obj, Decimal):
            return float(obj)
        if isinstance(obj, UUID):
            return str(obj)
        if isinstance(obj, set):
            return list(obj)
        if isinstance(obj, bytes):
            return obj.decode("utf-8")
        if hasattr(obj, "__dict__"):
            return obj.__dict__
        return super().default(obj)

class EnhancedJSONResponse(Response):
    media_type = "application/json"

    def render(self, content) -> bytes:
        return json.dumps(
            content,
            cls=EnhancedJSONEncoder,
            ensure_ascii=False
        ).encode("utf-8")

# Conditional serialization based on content
class SmartResponse(Response):
    """Chooses best format based on content type and size."""

    def render(self, content) -> bytes:
        serialized = json.dumps(content, cls=EnhancedJSONEncoder)

        # Compress large payloads automatically
        if len(serialized) > 10_000:
            import gzip
            self.headers["Content-Encoding"] = "gzip"
            return gzip.compress(serialized.encode())

        return serialized.encode()

@app.get("/orders/{order_id}", response_class=EnhancedJSONResponse)
async def get_order(order_id: UUID):
    return {
        "id": order_id,
        "total": Decimal("99.99"),
        "created_at": datetime.utcnow(),
        "tags": {"urgent", "vip"}
    }
```

---

## 21.2 Custom Request Classes

### Custom Request Handling

By subclassing Starlette's `Request`, you can add computed properties,
business-logic helpers, and preprocessing directly to the request object.

```python
from starlette.requests import Request
from starlette.datastructures import Headers
from fastapi import FastAPI
import time

class EnrichedRequest(Request):
    """Request with convenience properties for your domain."""

    @property
    def request_id(self) -> str:
        return self.headers.get("X-Request-ID", "")

    @property
    def client_ip(self) -> str:
        # Respect reverse-proxy headers
        forwarded = self.headers.get("X-Forwarded-For")
        if forwarded:
            return forwarded.split(",")[0].strip()
        return self.client.host if self.client else "unknown"

    @property
    def is_ajax(self) -> bool:
        return self.headers.get("X-Requested-With") == "XMLHttpRequest"

    @property
    def preferred_language(self) -> str:
        accept_lang = self.headers.get("Accept-Language", "en")
        return accept_lang.split(",")[0].split(";")[0].strip()

    @property
    def api_version(self) -> str:
        return self.headers.get("API-Version", "1")

    def get_bearer_token(self) -> str | None:
        auth = self.headers.get("Authorization", "")
        if auth.startswith("Bearer "):
            return auth[7:]
        return None

# Tell FastAPI to use your custom request class globally
class CustomFastAPI(FastAPI):
    def build_middleware_stack(self):
        # Inject our custom request class
        return super().build_middleware_stack()

app = FastAPI()

@app.get("/info")
async def request_info(request: EnrichedRequest):
    return {
        "client_ip":    request.client_ip,
        "request_id":   request.request_id,
        "language":     request.preferred_language,
        "api_version":  request.api_version,
        "is_ajax":      request.is_ajax,
    }
```

### Request Preprocessing

```python
from starlette.requests import Request
from starlette.middleware.base import BaseHTTPMiddleware
import json
import time

class PreprocessingMiddleware(BaseHTTPMiddleware):
    """Enrich every request before it reaches route handlers."""

    async def dispatch(self, request: Request, call_next):
        # 1. Attach metadata
        request.state.start_time  = time.perf_counter()
        request.state.request_id  = request.headers.get("X-Request-ID", str(uuid4()))

        # 2. Parse and cache the body once (avoid double-reading)
        if request.method in ("POST", "PUT", "PATCH"):
            body = await request.body()
            request.state.raw_body = body

            try:
                request.state.json_body = json.loads(body)
            except (json.JSONDecodeError, UnicodeDecodeError):
                request.state.json_body = None

        # 3. Resolve and cache user (avoid repeated token lookups)
        token = request.headers.get("Authorization", "")[7:]   # strip "Bearer "
        if token:
            request.state.current_user = await resolve_user_from_token(token)
        else:
            request.state.current_user = None

        response = await call_next(request)

        # 4. Post-process
        elapsed = (time.perf_counter() - request.state.start_time) * 1000
        response.headers["X-Request-ID"]    = request.state.request_id
        response.headers["X-Response-Time"] = f"{elapsed:.2f}ms"
        return response

app.add_middleware(PreprocessingMiddleware)

@app.post("/orders")
async def create_order(request: Request):
    # Body already parsed — no double await
    body = request.state.json_body
    user = request.state.current_user
    return {"user": user["id"] if user else None, "body": body}
```

### Custom Validators

```python
from fastapi import FastAPI, Request, HTTPException
from starlette.middleware.base import BaseHTTPMiddleware
import re

app = FastAPI()

class RequestValidationMiddleware(BaseHTTPMiddleware):
    """Validate requests before they ever reach route handlers."""

    MAX_BODY_SIZE   = 5 * 1024 * 1024   # 5 MB
    ALLOWED_CONTENT = {"application/json", "multipart/form-data",
                        "application/x-www-form-urlencoded"}

    async def dispatch(self, request: Request, call_next):
        # 1. Content-type validation
        if request.method in ("POST", "PUT", "PATCH"):
            ct = request.headers.get("content-type", "").split(";")[0].strip()
            if ct and ct not in self.ALLOWED_CONTENT:
                return JSONResponse(
                    status_code=415,
                    content={"error": f"Unsupported content type: {ct}"}
                )

        # 2. Body size validation
        content_length = request.headers.get("content-length")
        if content_length and int(content_length) > self.MAX_BODY_SIZE:
            return JSONResponse(
                status_code=413,
                content={"error": "Request body too large (max 5 MB)"}
            )

        # 3. SQL injection check on query params
        for key, value in request.query_params.items():
            if self._looks_like_sqli(value):
                return JSONResponse(
                    status_code=400,
                    content={"error": f"Invalid characters in parameter: {key}"}
                )

        return await call_next(request)

    def _looks_like_sqli(self, value: str) -> bool:
        patterns = [r"(;|--|\bUNION\b|\bSELECT\b|\bDROP\b)", r"(\bOR\b\s+\d+=\d+)"]
        return any(re.search(p, value, re.IGNORECASE) for p in patterns)

# Route-level custom validator via dependency
from fastapi import Depends

async def require_json_body(request: Request):
    """Dependency that ensures a parseable JSON body exists."""
    if not request.headers.get("content-type", "").startswith("application/json"):
        raise HTTPException(415, "Content-Type must be application/json")
    try:
        body = await request.json()
        if not body:
            raise HTTPException(400, "Request body cannot be empty")
        return body
    except Exception:
        raise HTTPException(400, "Invalid JSON body")

@app.post("/strict-endpoint")
async def strict(body: dict = Depends(require_json_body)):
    return {"received": body}
```

---

## 21.3 Metaprogramming

### Dynamic Route Generation

Generate routes at runtime from configuration, database, or other data
sources — useful for plugin systems, CMS-like apps, and auto-generated APIs.

```python
from fastapi import FastAPI
from pydantic import BaseModel
from typing import Type

app = FastAPI()

# Generate CRUD routes for any model dynamically
def register_crud_routes(
    app: FastAPI,
    prefix: str,
    model_class: Type,
    tag: str
):
    """Auto-generate GET/POST/PUT/DELETE for a resource."""

    @app.get(f"/{prefix}", tags=[tag], summary=f"List {tag}")
    async def list_resources():
        return []

    @app.get(f"/{prefix}/{{id}}", tags=[tag], summary=f"Get {tag}")
    async def get_resource(id: int):
        return {"id": id}

    @app.post(f"/{prefix}", tags=[tag], summary=f"Create {tag}", status_code=201)
    async def create_resource(data: model_class):
        return data

    @app.put(f"/{prefix}/{{id}}", tags=[tag], summary=f"Update {tag}")
    async def update_resource(id: int, data: model_class):
        return {"id": id, **data.dict()}

    @app.delete(f"/{prefix}/{{id}}", tags=[tag], status_code=204)
    async def delete_resource(id: int):
        return None

# Usage
class Product(BaseModel):
    name: str
    price: float

class Category(BaseModel):
    title: str
    slug: str

register_crud_routes(app, "products",   Product,  "Products")
register_crud_routes(app, "categories", Category, "Categories")

# Now you have:
# GET/POST       /products
# GET/PUT/DELETE /products/{id}
# GET/POST       /categories
# GET/PUT/DELETE /categories/{id}

# Dynamic routes from configuration
ROUTE_CONFIG = [
    {"path": "/ping",   "method": "GET",  "response": {"status": "ok"}},
    {"path": "/version","method": "GET",  "response": {"version": "1.0.0"}},
    {"path": "/env",    "method": "GET",  "response": {"env": "production"}},
]

for config in ROUTE_CONFIG:
    response_data = config["response"]   # Capture in closure

    def make_handler(data):
        async def handler():
            return data
        return handler

    app.add_api_route(
        config["path"],
        make_handler(response_data),
        methods=[config["method"]]
    )
```

### Dynamic Model Creation

Create Pydantic models at runtime for situations where the schema isn't known
until the application starts.

```python
from pydantic import BaseModel, create_model
from typing import Optional, Any
from fastapi import FastAPI

app = FastAPI()

# create_model: build Pydantic model from dict spec
def build_model(name: str, fields: dict[str, tuple]) -> type[BaseModel]:
    """
    fields = {"field_name": (type, default_or_...)}
    Example: {"name": (str, ...), "age": (int, 0)}
    """
    return create_model(name, **{k: v for k, v in fields.items()})

# Generate models from a database schema definition
TABLE_SCHEMAS = {
    "users": {
        "username":  (str, ...),
        "email":     (str, ...),
        "age":       (Optional[int], None),
        "is_active": (bool, True),
    },
    "products": {
        "name":     (str, ...),
        "price":    (float, ...),
        "stock":    (int, 0),
        "category": (Optional[str], None),
    }
}

dynamic_models = {
    name: build_model(name.capitalize(), fields)
    for name, fields in TABLE_SCHEMAS.items()
}

# Register endpoints using the dynamic models
for resource, model in dynamic_models.items():
    def make_endpoint(m):
        async def create(data: m):
            return data.dict()
        create.__name__ = f"create_{resource}"
        return create

    app.add_api_route(
        f"/{resource}",
        make_endpoint(model),
        methods=["POST"],
        tags=[resource.capitalize()]
    )

# Extend existing models programmatically
def add_audit_fields(model: type[BaseModel]) -> type[BaseModel]:
    """Add created_at / updated_at to any model."""
    from datetime import datetime
    return create_model(
        f"Audited{model.__name__}",
        __base__=model,
        created_at=(datetime, ...),
        updated_at=(Optional[datetime], None)
    )

BaseProduct = build_model("Product", {"name": (str, ...), "price": (float, ...)})
AuditedProduct = add_audit_fields(BaseProduct)
```

### Code Generation

```python
from fastapi import FastAPI
from typing import Any
import inspect

app = FastAPI()

# Auto-generate OpenAPI-compatible schemas
def generate_schema_from_dataclass(cls) -> dict:
    """Generate JSON Schema from a dataclass."""
    import dataclasses
    schema = {"title": cls.__name__, "type": "object", "properties": {}, "required": []}

    for field in dataclasses.fields(cls):
        prop = {}
        if field.type == int:
            prop["type"] = "integer"
        elif field.type == float:
            prop["type"] = "number"
        elif field.type == str:
            prop["type"] = "string"
        elif field.type == bool:
            prop["type"] = "boolean"

        schema["properties"][field.name] = prop
        if field.default is dataclasses.MISSING:
            schema["required"].append(field.name)

    return schema

# Generate service boilerplate from a router
def generate_service_code(router_module) -> str:
    """Introspect a router and generate service stubs."""
    lines = [
        "# Auto-generated service stubs",
        "from sqlalchemy.ext.asyncio import AsyncSession",
        "",
        "class GeneratedService:",
    ]

    for name, obj in inspect.getmembers(router_module, inspect.iscoroutinefunction):
        if name.startswith("_"):
            continue
        sig = inspect.signature(obj)
        params = [
            f"{p}: {a.annotation.__name__ if a.annotation != inspect.Parameter.empty else 'Any'}"
            for p, a in sig.parameters.items()
            if p not in ("request", "response", "db")
        ]
        lines.append(f"    async def {name}(self, db: AsyncSession, {', '.join(params)}):")
        lines.append( "        raise NotImplementedError")
        lines.append("")

    return "\n".join(lines)

# Endpoint that returns auto-generated API client code
@app.get("/codegen/python-client")
async def generate_python_client():
    routes = []
    for route in app.routes:
        if hasattr(route, "methods"):
            routes.append({
                "path":    route.path,
                "methods": list(route.methods),
                "name":    route.name
            })

    lines = ["import httpx", "", "class APIClient:"]
    for route in routes:
        func_name = route["name"] or route["path"].replace("/", "_").strip("_")
        method = list(route["methods"])[0].lower()
        lines.append(f"    async def {func_name}(self, **kwargs):")
        lines.append(f"        async with httpx.AsyncClient() as c:")
        lines.append(f"            return await c.{method}('{route['path']}', **kwargs)")
        lines.append("")

    return {"client_code": "\n".join(lines)}
```

---

## 21.4 Plugin Systems

### Creating Plugins

A plugin system lets you extend your FastAPI app with reusable, self-contained
modules that register their own routes, middleware, and startup logic.

```python
# core/plugin.py — the plugin contract
from abc import ABC, abstractmethod
from fastapi import FastAPI

class Plugin(ABC):
    """Base class all plugins must implement."""

    @property
    @abstractmethod
    def name(self) -> str:
        """Unique plugin identifier."""

    @property
    def version(self) -> str:
        return "1.0.0"

    @property
    def description(self) -> str:
        return ""

    def register(self, app: FastAPI) -> None:
        """Called once when plugin is loaded."""

    async def on_startup(self) -> None:
        """Called when the app starts."""

    async def on_shutdown(self) -> None:
        """Called when the app stops."""
```

### Plugin Architecture

```python
# core/plugin_manager.py
from typing import Dict, List, Type
from fastapi import FastAPI
import importlib
import logging

logger = logging.getLogger(__name__)

class PluginManager:
    def __init__(self):
        self._plugins: Dict[str, Plugin] = {}

    def load(self, plugin: Plugin) -> None:
        """Register a plugin instance."""
        if plugin.name in self._plugins:
            raise ValueError(f"Plugin '{plugin.name}' already loaded")
        self._plugins[plugin.name] = plugin
        logger.info(f"Plugin loaded: {plugin.name} v{plugin.version}")

    def load_from_module(self, module_path: str) -> None:
        """Load plugin from a Python module path string."""
        module = importlib.import_module(module_path)
        if not hasattr(module, "plugin"):
            raise AttributeError(f"Module {module_path} has no 'plugin' attribute")
        self.load(module.plugin)

    def register_all(self, app: FastAPI) -> None:
        """Register all loaded plugins with the app."""
        for plugin in self._plugins.values():
            plugin.register(app)
            logger.info(f"Plugin registered: {plugin.name}")

    async def startup_all(self) -> None:
        for plugin in self._plugins.values():
            await plugin.on_startup()

    async def shutdown_all(self) -> None:
        for plugin in self._plugins.values():
            await plugin.on_shutdown()

    def get(self, name: str) -> Plugin | None:
        return self._plugins.get(name)

    @property
    def loaded(self) -> List[str]:
        return list(self._plugins.keys())
```

### Extension Points

```python
# plugins/analytics_plugin.py — a self-contained plugin
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware
from core.plugin import Plugin
from collections import defaultdict
import time

class AnalyticsPlugin(Plugin):
    name        = "analytics"
    version     = "1.2.0"
    description = "Request analytics and timing"

    def __init__(self):
        self._stats = defaultdict(lambda: {"count": 0, "total_ms": 0.0})

    def register(self, app: FastAPI) -> None:
        # 1. Add middleware
        app.add_middleware(BaseHTTPMiddleware, dispatch=self._track)

        # 2. Add routes
        app.add_api_route(
            "/analytics/stats",
            self._stats_endpoint,
            methods=["GET"],
            tags=["Analytics"]
        )
        app.add_api_route(
            "/analytics/reset",
            self._reset_endpoint,
            methods=["POST"],
            tags=["Analytics"]
        )

    async def _track(self, request: Request, call_next):
        start = time.perf_counter()
        response = await call_next(request)
        elapsed = (time.perf_counter() - start) * 1000

        key = f"{request.method} {request.url.path}"
        self._stats[key]["count"] += 1
        self._stats[key]["total_ms"] += elapsed

        return response

    async def _stats_endpoint(self):
        return {
            route: {
                "count":   s["count"],
                "avg_ms":  round(s["total_ms"] / s["count"], 2) if s["count"] else 0,
                "total_ms": round(s["total_ms"], 2)
            }
            for route, s in self._stats.items()
        }

    async def _reset_endpoint(self):
        self._stats.clear()
        return {"message": "Stats reset"}

    async def on_startup(self) -> None:
        print(f"[{self.name}] Analytics plugin started")

    async def on_shutdown(self) -> None:
        print(f"[{self.name}] Final stats: {dict(self._stats)}")

# Export plugin instance (convention)
plugin = AnalyticsPlugin()

# plugins/rate_limit_plugin.py
from core.plugin import Plugin
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from starlette.middleware.base import BaseHTTPMiddleware
from collections import defaultdict
import time

class RateLimitPlugin(Plugin):
    name = "rate_limit"

    def __init__(self, requests_per_minute: int = 60):
        self.rpm = requests_per_minute
        self._windows: dict = defaultdict(list)

    def register(self, app: FastAPI) -> None:
        app.add_middleware(BaseHTTPMiddleware, dispatch=self._limit)

    async def _limit(self, request: Request, call_next):
        ip  = request.client.host
        now = time.time()

        self._windows[ip] = [t for t in self._windows[ip] if now - t < 60]

        if len(self._windows[ip]) >= self.rpm:
            return JSONResponse(
                status_code=429,
                content={"error": "Rate limit exceeded"},
                headers={"Retry-After": "60"}
            )

        self._windows[ip].append(now)
        return await call_next(request)

plugin = RateLimitPlugin(requests_per_minute=100)

# main.py — wiring everything together
from fastapi import FastAPI
from core.plugin_manager import PluginManager

app = FastAPI()
plugins = PluginManager()

# Load plugins
plugins.load_from_module("plugins.analytics_plugin")
plugins.load_from_module("plugins.rate_limit_plugin")

# Register routes + middleware from all plugins
plugins.register_all(app)

@app.on_event("startup")
async def startup():
    await plugins.startup_all()
    print(f"Loaded plugins: {plugins.loaded}")

@app.on_event("shutdown")
async def shutdown():
    await plugins.shutdown_all()

@app.get("/")
async def root():
    return {"plugins": plugins.loaded}
```

---

## 21.5 Performance Profiling

### cProfile Integration

cProfile provides function-level CPU profiling — where time is actually
being spent in your application.

```python
import cProfile
import pstats
import io
import asyncio
from fastapi import FastAPI, Request, Query
from fastapi.responses import PlainTextResponse

app = FastAPI()

# ── Profiling Middleware ─────────────────────────────────────
class ProfilingMiddleware:
    """Enable with ?profile=true query param."""

    def __init__(self, app):
        self.app = app

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return

        # Check for profile flag
        query = scope.get("query_string", b"").decode()
        if "profile=true" not in query:
            await self.app(scope, receive, send)
            return

        # Profile the request
        pr = cProfile.Profile()
        pr.enable()
        await self.app(scope, receive, send)
        pr.disable()

        # Format results
        s  = io.StringIO()
        ps = pstats.Stats(pr, stream=s)
        ps.sort_stats("cumulative")
        ps.print_stats(30)   # Top 30 functions

        print(f"\n{'='*60}\nPROFILE REPORT\n{'='*60}")
        print(s.getvalue())

app.add_middleware(ProfilingMiddleware)

# ── On-demand endpoint profiler ──────────────────────────────
@app.get("/debug/profile", response_class=PlainTextResponse)
async def profile_endpoint(
    func: str = Query(..., description="Function path to profile"),
    iterations: int = Query(100, ge=1, le=10000)
):
    """Profile a specific function N times and return stats."""
    module_path, func_name = func.rsplit(".", 1)

    import importlib
    module = importlib.import_module(module_path)
    target = getattr(module, func_name)

    pr = cProfile.Profile()
    pr.enable()
    for _ in range(iterations):
        if asyncio.iscoroutinefunction(target):
            await target()
        else:
            target()
    pr.disable()

    s  = io.StringIO()
    ps = pstats.Stats(pr, stream=s)
    ps.sort_stats("cumulative")
    ps.print_stats(20)
    return s.getvalue()

# ── Profiling a specific code block ─────────────────────────
async def profile_block(coro):
    """Profile an async block inline."""
    pr = cProfile.Profile()
    pr.enable()
    result = await coro
    pr.disable()

    s  = io.StringIO()
    ps = pstats.Stats(pr, stream=s)
    ps.sort_stats("tottime")
    ps.print_stats(10)
    print(s.getvalue())

    return result

@app.get("/expensive")
async def expensive_with_profile():
    return await profile_block(do_expensive_work())
```

### Memory Profiling

```bash
pip install memory-profiler tracemalloc psutil
```

```python
import tracemalloc
import psutil
import os
from fastapi import FastAPI
from fastapi.responses import JSONResponse

app = FastAPI()

# ── tracemalloc — Python-level allocations ────────────────────
class MemoryTracingMiddleware:
    def __init__(self, app, top_n: int = 10):
        self.app  = app
        self.top_n = top_n

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return

        query = scope.get("query_string", b"").decode()
        if "memprofile=true" not in query:
            await self.app(scope, receive, send)
            return

        tracemalloc.start()
        snapshot_before = tracemalloc.take_snapshot()

        await self.app(scope, receive, send)

        snapshot_after = tracemalloc.take_snapshot()
        tracemalloc.stop()

        stats = snapshot_after.compare_to(snapshot_before, "lineno")
        print(f"\n{'='*50}\nMEMORY DELTA (top {self.top_n})\n{'='*50}")
        for stat in stats[:self.top_n]:
            print(stat)

app.add_middleware(MemoryTracingMiddleware, top_n=10)

# ── System memory endpoint ────────────────────────────────────
@app.get("/debug/memory")
async def memory_stats():
    process = psutil.Process(os.getpid())
    mem     = process.memory_info()
    vm      = psutil.virtual_memory()

    # Python allocation snapshots
    tracemalloc.start()
    snapshot = tracemalloc.take_snapshot()
    top_stats = snapshot.statistics("lineno")[:5]

    return {
        "process": {
            "rss_mb":  round(mem.rss  / 1024 / 1024, 2),
            "vms_mb":  round(mem.vms  / 1024 / 1024, 2),
            "percent": round(process.memory_percent(), 2)
        },
        "system": {
            "total_gb":     round(vm.total     / 1024**3, 2),
            "available_gb": round(vm.available / 1024**3, 2),
            "used_percent": vm.percent
        },
        "top_allocations": [
            {
                "file":  str(s.traceback[0].filename).split("/")[-1],
                "line":  s.traceback[0].lineno,
                "size_kb": round(s.size / 1024, 2)
            }
            for s in top_stats
        ]
    }
```

### Performance Analysis

```python
from fastapi import FastAPI, Request
import time
import statistics
from collections import defaultdict, deque

app = FastAPI()

class PerformanceAnalyzer:
    """Track latency, throughput, and error rates per endpoint."""

    def __init__(self, window: int = 1000):
        self.window = window
        # Keep last N samples per route
        self._latencies: dict[str, deque] = defaultdict(lambda: deque(maxlen=window))
        self._errors:    dict[str, int]   = defaultdict(int)
        self._counts:    dict[str, int]   = defaultdict(int)

    def record(self, route: str, latency_ms: float, status_code: int):
        self._latencies[route].append(latency_ms)
        self._counts[route] += 1
        if status_code >= 400:
            self._errors[route] += 1

    def report(self) -> dict:
        report = {}
        for route, latencies in self._latencies.items():
            if not latencies:
                continue
            sorted_l = sorted(latencies)
            n        = len(sorted_l)
            report[route] = {
                "requests":   self._counts[route],
                "errors":     self._errors[route],
                "error_rate": round(self._errors[route] / self._counts[route] * 100, 2),
                "latency": {
                    "min_ms":  round(sorted_l[0], 2),
                    "avg_ms":  round(statistics.mean(sorted_l), 2),
                    "median":  round(statistics.median(sorted_l), 2),
                    "p95_ms":  round(sorted_l[int(n * 0.95)], 2),
                    "p99_ms":  round(sorted_l[int(n * 0.99)], 2),
                    "max_ms":  round(sorted_l[-1], 2),
                    "std_dev": round(statistics.stdev(sorted_l), 2) if n > 1 else 0,
                }
            }
        return report

analyzer = PerformanceAnalyzer()

@app.middleware("http")
async def track_performance(request: Request, call_next):
    start    = time.perf_counter()
    response = await call_next(request)
    elapsed  = (time.perf_counter() - start) * 1000

    route = f"{request.method} {request.url.path}"
    analyzer.record(route, elapsed, response.status_code)
    return response

@app.get("/debug/performance")
async def performance_report():
    return analyzer.report()
```

### Optimization Strategies

```python
from fastapi import FastAPI
from functools import lru_cache
import asyncio
import time

app = FastAPI()

# ── Strategy 1: Identify hot paths with sampling ─────────────
import random

SAMPLE_RATE = 0.1   # Profile 10% of requests

@app.middleware("http")
async def sampled_profiling(request: Request, call_next):
    if random.random() < SAMPLE_RATE:
        pr = cProfile.Profile()
        pr.enable()
        response = await call_next(request)
        pr.disable()
        # Log top functions to your monitoring tool
        s  = io.StringIO()
        pstats.Stats(pr, stream=s).sort_stats("cumulative").print_stats(5)
        logger.debug(f"Profile sample:\n{s.getvalue()}")
        return response
    return await call_next(request)

# ── Strategy 2: Cache expensive computations ─────────────────
from cachetools import TTLCache
_cache = TTLCache(maxsize=500, ttl=300)

@app.get("/report/{report_id}")
async def get_report(report_id: int):
    if report_id in _cache:
        return {**_cache[report_id], "cached": True}

    result = await generate_expensive_report(report_id)
    _cache[report_id] = result
    return {**result, "cached": False}

# ── Strategy 3: Parallelize independent I/O ──────────────────
@app.get("/dashboard/{user_id}")
async def dashboard(user_id: int):
    # Sequential (slow): 3 seconds total
    # user    = await get_user(user_id)       # 1s
    # orders  = await get_orders(user_id)     # 1s
    # metrics = await get_metrics(user_id)    # 1s

    # Parallel (fast): ~1 second total
    user, orders, metrics = await asyncio.gather(
        get_user(user_id),
        get_orders(user_id),
        get_metrics(user_id)
    )
    return {"user": user, "orders": orders, "metrics": metrics}

# ── Strategy 4: Use __slots__ for data-heavy models ──────────
class SlottedPoint:
    __slots__ = ("x", "y", "z")   # 40% less memory than __dict__
    def __init__(self, x, y, z):
        self.x, self.y, self.z = x, y, z

# ── Strategy 5: Connection pool sizing formula ───────────────
import os
CPU_COUNT     = os.cpu_count() or 1
# Rule of thumb: (2 × CPU cores) + 1 for I/O-bound workloads
POOL_SIZE     = (2 * CPU_COUNT) + 1
MAX_OVERFLOW  = CPU_COUNT
```

---

## 21.6 Distributed Systems

### Message Queues (RabbitMQ, Kafka)

```bash
pip install aio-pika aiokafka
```

**RabbitMQ with aio-pika**:
```python
import aio_pika
import json
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()
rabbitmq_conn = None

@app.on_event("startup")
async def connect_rabbitmq():
    global rabbitmq_conn
    rabbitmq_conn = await aio_pika.connect_robust("amqp://guest:guest@localhost/")

@app.on_event("shutdown")
async def disconnect_rabbitmq():
    if rabbitmq_conn:
        await rabbitmq_conn.close()

async def publish(queue_name: str, message: dict):
    """Publish a message to a RabbitMQ queue."""
    async with rabbitmq_conn.channel() as channel:
        queue = await channel.declare_queue(queue_name, durable=True)
        await channel.default_exchange.publish(
            aio_pika.Message(
                body=json.dumps(message).encode(),
                delivery_mode=aio_pika.DeliveryMode.PERSISTENT,
                content_type="application/json"
            ),
            routing_key=queue.name
        )

async def consume(queue_name: str, handler):
    """Consume messages from a queue."""
    async with rabbitmq_conn.channel() as channel:
        await channel.set_qos(prefetch_count=10)
        queue = await channel.declare_queue(queue_name, durable=True)
        async with queue.iterator() as q:
            async for message in q:
                async with message.process():
                    data = json.loads(message.body)
                    await handler(data)

@app.post("/orders")
async def create_order(order: dict, background_tasks: BackgroundTasks):
    # Persist to database...
    order["id"] = 42

    # Publish events for other services
    await publish("order.created",   {"event": "order.created",   "order_id": order["id"]})
    await publish("email.send",      {"event": "welcome_email",   "to": order["email"]})
    await publish("inventory.reserve", {"event": "reserve",       "items": order["items"]})

    return order
```

**Kafka with aiokafka**:
```python
from aiokafka import AIOKafkaProducer, AIOKafkaConsumer
from fastapi import FastAPI
import json

app = FastAPI()
producer = None

@app.on_event("startup")
async def start_kafka():
    global producer
    producer = AIOKafkaProducer(bootstrap_servers="localhost:9092")
    await producer.start()

@app.on_event("shutdown")
async def stop_kafka():
    if producer:
        await producer.stop()

async def publish_event(topic: str, key: str, value: dict):
    await producer.send_and_wait(
        topic,
        key=key.encode(),
        value=json.dumps(value).encode()
    )

async def consume_events(topic: str, group_id: str, handler):
    consumer = AIOKafkaConsumer(
        topic,
        bootstrap_servers="localhost:9092",
        group_id=group_id,
        auto_offset_reset="earliest"
    )
    await consumer.start()
    try:
        async for msg in consumer:
            event = json.loads(msg.value)
            await handler(event)
    finally:
        await consumer.stop()

@app.post("/events/{event_type}")
async def emit_event(event_type: str, payload: dict):
    await publish_event(
        topic="app-events",
        key=event_type,
        value={"type": event_type, "payload": payload}
    )
    return {"published": event_type}
```

### Event-Driven Architecture

```python
from fastapi import FastAPI
from typing import Callable, Dict, List, Any
import asyncio

app = FastAPI()

# ── In-process event bus ──────────────────────────────────────
class EventBus:
    """Lightweight async pub/sub for decoupling services."""

    def __init__(self):
        self._handlers: Dict[str, List[Callable]] = {}

    def on(self, event: str):
        """Decorator to register an event handler."""
        def decorator(func: Callable):
            self._handlers.setdefault(event, []).append(func)
            return func
        return decorator

    async def emit(self, event: str, payload: Any = None):
        """Fire event — all handlers run concurrently."""
        handlers = self._handlers.get(event, [])
        if handlers:
            await asyncio.gather(*[h(payload) for h in handlers])

    async def emit_async(self, event: str, payload: Any = None):
        """Fire and forget — don't await handlers."""
        asyncio.create_task(self.emit(event, payload))

bus = EventBus()

# Register domain event handlers
@bus.on("user.registered")
async def send_welcome_email(user: dict):
    print(f"→ Sending welcome email to {user['email']}")

@bus.on("user.registered")
async def create_default_settings(user: dict):
    print(f"→ Creating settings for user {user['id']}")

@bus.on("user.registered")
async def notify_crm(user: dict):
    print(f"→ Syncing {user['email']} to CRM")

@bus.on("order.placed")
async def reserve_inventory(order: dict):
    print(f"→ Reserving inventory for order {order['id']}")

@bus.on("order.placed")
async def send_confirmation(order: dict):
    print(f"→ Sending order confirmation")

# Usage in endpoints
@app.post("/users", status_code=201)
async def register_user(data: dict):
    user = {"id": 1, **data}   # persist to db...

    # One emit triggers all registered handlers concurrently
    await bus.emit("user.registered", user)

    return user
```

### CQRS Pattern

CQRS (Command Query Responsibility Segregation) separates read and write
paths so each can be optimized independently.

```python
from fastapi import FastAPI, Depends
from pydantic import BaseModel
from abc import ABC, abstractmethod

app = FastAPI()

# ── Commands (write side) ─────────────────────────────────────
class Command(BaseModel):
    pass

class CreateProductCommand(Command):
    name: str
    price: float
    sku: str

class UpdatePriceCommand(Command):
    product_id: int
    new_price: float

class DeleteProductCommand(Command):
    product_id: int

# ── Queries (read side) ───────────────────────────────────────
class Query(BaseModel):
    pass

class GetProductQuery(Query):
    product_id: int

class SearchProductsQuery(Query):
    term: str
    category: str | None = None
    min_price: float | None = None
    max_price: float | None = None

# ── Handlers ──────────────────────────────────────────────────
class CommandHandler(ABC):
    @abstractmethod
    async def handle(self, command: Command): ...

class QueryHandler(ABC):
    @abstractmethod
    async def handle(self, query: Query): ...

class CreateProductHandler(CommandHandler):
    async def handle(self, cmd: CreateProductCommand):
        # Write to primary (PostgreSQL) database
        product = {"id": 1, "name": cmd.name, "price": cmd.price, "sku": cmd.sku}
        # Optionally publish ProductCreated event
        return product

class SearchProductsHandler(QueryHandler):
    async def handle(self, query: SearchProductsQuery):
        # Read from optimized read model (Elasticsearch, Redis, read replica)
        return [{"id": 1, "name": "Widget", "price": 9.99}]

# ── Bus ───────────────────────────────────────────────────────
class CommandBus:
    def __init__(self):
        self._handlers: dict[type, CommandHandler] = {}

    def register(self, command_type: type, handler: CommandHandler):
        self._handlers[command_type] = handler

    async def dispatch(self, command: Command):
        handler = self._handlers.get(type(command))
        if not handler:
            raise ValueError(f"No handler for {type(command).__name__}")
        return await handler.handle(command)

class QueryBus:
    def __init__(self):
        self._handlers: dict[type, QueryHandler] = {}

    def register(self, query_type: type, handler: QueryHandler):
        self._handlers[query_type] = handler

    async def dispatch(self, query: Query):
        handler = self._handlers.get(type(query))
        if not handler:
            raise ValueError(f"No handler for {type(query).__name__}")
        return await handler.handle(query)

command_bus = CommandBus()
query_bus   = QueryBus()

command_bus.register(CreateProductCommand, CreateProductHandler())
query_bus.register(SearchProductsQuery,   SearchProductsHandler())

@app.post("/products", status_code=201)
async def create_product(cmd: CreateProductCommand):
    return await command_bus.dispatch(cmd)

@app.get("/products/search")
async def search_products(query: SearchProductsQuery = Depends()):
    return await query_bus.dispatch(query)
```

### Event Sourcing

```python
from pydantic import BaseModel
from datetime import datetime
from typing import List, Any
import json

# ── Events ────────────────────────────────────────────────────
class DomainEvent(BaseModel):
    event_id:   str = Field(default_factory=lambda: str(uuid4()))
    event_type: str
    aggregate_id: str
    aggregate_type: str
    payload:    dict
    timestamp:  datetime = Field(default_factory=datetime.utcnow)
    version:    int = 1

# ── Event Store ───────────────────────────────────────────────
class EventStore:
    """Append-only store — the system of record."""

    def __init__(self, db: AsyncSession):
        self.db = db

    async def append(self, event: DomainEvent) -> None:
        """Append event — never update or delete."""
        row = EventRecord(
            event_id=event.event_id,
            event_type=event.event_type,
            aggregate_id=event.aggregate_id,
            aggregate_type=event.aggregate_type,
            payload=json.dumps(event.payload),
            timestamp=event.timestamp,
            version=event.version
        )
        self.db.add(row)
        await self.db.commit()

    async def get_events(
        self,
        aggregate_id: str,
        from_version: int = 0
    ) -> List[DomainEvent]:
        result = await self.db.execute(
            select(EventRecord)
            .where(EventRecord.aggregate_id == aggregate_id)
            .where(EventRecord.version > from_version)
            .order_by(EventRecord.version)
        )
        rows = result.scalars().all()
        return [DomainEvent(**{**r.__dict__, "payload": json.loads(r.payload)}) for r in rows]

# ── Aggregate ────────────────────────────────────────────────
class Order:
    """Rebuilt from events — state is derived, never stored."""

    def __init__(self, order_id: str):
        self.id       = order_id
        self.items    = []
        self.status   = "new"
        self.total    = 0.0
        self._version = 0
        self._pending: List[DomainEvent] = []

    # Rebuild state by replaying events
    def apply(self, event: DomainEvent):
        if event.event_type == "OrderCreated":
            self.items  = event.payload["items"]
            self.total  = event.payload["total"]
            self.status = "created"
        elif event.event_type == "OrderPaid":
            self.status = "paid"
        elif event.event_type == "OrderCancelled":
            self.status = "cancelled"
        self._version = event.version

    # Business operations generate events
    def pay(self, payment_ref: str):
        if self.status != "created":
            raise ValueError(f"Cannot pay order in status: {self.status}")
        self._pending.append(DomainEvent(
            event_type="OrderPaid",
            aggregate_id=self.id,
            aggregate_type="Order",
            payload={"payment_ref": payment_ref},
            version=self._version + 1
        ))
        self.apply(self._pending[-1])

    @classmethod
    async def load(cls, order_id: str, store: EventStore) -> "Order":
        """Reconstruct from event stream."""
        order  = cls(order_id)
        events = await store.get_events(order_id)
        for event in events:
            order.apply(event)
        return order

    async def save(self, store: EventStore):
        for event in self._pending:
            await store.append(event)
        self._pending.clear()
```

### Saga Pattern

The Saga pattern manages long-running, distributed transactions by composing
a series of local transactions with compensating rollbacks.

```python
from enum import Enum
from typing import List, Callable, Awaitable
from pydantic import BaseModel
import asyncio

class StepStatus(str, Enum):
    PENDING     = "pending"
    COMPLETED   = "completed"
    FAILED      = "failed"
    COMPENSATED = "compensated"

class SagaStep(BaseModel):
    name:   str
    status: StepStatus = StepStatus.PENDING

class Saga:
    """Orchestration-based saga with automatic compensation."""

    def __init__(self, name: str):
        self.name       = name
        self._steps:    List[tuple[Callable, Callable]] = []
        self._executed: List[tuple[SagaStep, Callable]] = []

    def step(self, name: str, action: Callable, compensate: Callable):
        self._steps.append((name, action, compensate))
        return self

    async def execute(self) -> bool:
        for name, action, compensate in self._steps:
            step = SagaStep(name=name)
            try:
                print(f"  [{self.name}] Executing: {name}")
                await action()
                step.status = StepStatus.COMPLETED
                self._executed.append((step, compensate))
                print(f"  [{self.name}] ✓ {name}")
            except Exception as e:
                step.status = StepStatus.FAILED
                print(f"  [{self.name}] ✗ {name}: {e}")
                await self._rollback()
                return False
        return True

    async def _rollback(self):
        print(f"  [{self.name}] Rolling back {len(self._executed)} steps...")
        for step, compensate in reversed(self._executed):
            try:
                await compensate()
                step.status = StepStatus.COMPENSATED
                print(f"  [{self.name}] ↩ Compensated: {step.name}")
            except Exception as e:
                print(f"  [{self.name}] ! Compensation failed: {step.name}: {e}")

# ── Order placement saga ──────────────────────────────────────
@app.post("/orders/checkout")
async def checkout(cart: dict):
    order_id = str(uuid4())

    saga = Saga("OrderCheckout")
    saga.step(
        "Reserve inventory",
        action=lambda:     reserve_inventory(cart["items"]),
        compensate=lambda: release_inventory(cart["items"])
    ).step(
        "Charge payment",
        action=lambda:     charge_customer(cart["user_id"], cart["total"]),
        compensate=lambda: refund_customer(cart["user_id"], cart["total"])
    ).step(
        "Create order record",
        action=lambda:     create_order_record(order_id, cart),
        compensate=lambda: delete_order_record(order_id)
    ).step(
        "Send confirmation",
        action=lambda:     send_confirmation_email(cart["email"], order_id),
        compensate=lambda: None   # Email sent — can't unsend, just log
    )

    success = await saga.execute()

    if success:
        return {"order_id": order_id, "status": "confirmed"}
    else:
        return JSONResponse(
            status_code=500,
            content={"error": "Checkout failed — all changes rolled back"}
        )
```

---

## Complete Example: Distributed Order Service

```python
# main.py — brings all advanced patterns together
from fastapi import FastAPI, Depends
from fastapi.responses import ORJSONResponse
from pydantic import BaseModel

# Custom response for performance
app = FastAPI(
    title="Distributed Order Service",
    default_response_class=ORJSONResponse
)

# Plugin manager
plugins = PluginManager()
plugins.load_from_module("plugins.analytics_plugin")
plugins.load_from_module("plugins.rate_limit_plugin")
plugins.register_all(app)

# Event bus
bus = EventBus()

@bus.on("order.created")
async def on_order_created(order: dict):
    await publish("inventory.reserve", {"order_id": order["id"], "items": order["items"]})
    await publish("email.order_confirm", {"to": order["email"], "order_id": order["id"]})

@bus.on("order.paid")
async def on_order_paid(order: dict):
    await publish("fulfillment.start", {"order_id": order["id"]})

# CQRS: separate create from read
@app.post("/orders", status_code=201)
async def create_order(cmd: CreateOrderCommand, db=Depends(get_db)):
    # Saga orchestrates the distributed transaction
    saga = Saga("CreateOrder")
    saga.step("reserve",  lambda: reserve_stock(cmd.items),  lambda: release_stock(cmd.items))
    saga.step("charge",   lambda: charge(cmd.user_id, cmd.total), lambda: refund(cmd.user_id, cmd.total))
    saga.step("persist",  lambda: save_order(db, cmd),       lambda: delete_order(db, cmd))

    success = await saga.execute()
    if not success:
        raise HTTPException(500, "Order creation failed")

    order = await save_order(db, cmd)

    # Append to event store
    store = EventStore(db)
    await store.append(DomainEvent(
        event_type="OrderCreated",
        aggregate_id=str(order.id),
        aggregate_type="Order",
        payload=order.dict(),
        version=1
    ))

    # Emit to in-process bus
    await bus.emit("order.created", order.dict())

    return order

@app.get("/orders/{order_id}")
async def get_order(order_id: str, db=Depends(get_db)):
    # Rebuild state from event store
    store = EventStore(db)
    order = await Order.load(order_id, store)
    return {"id": order.id, "status": order.status, "total": order.total}

@app.get("/debug/performance")
async def perf():
    return analyzer.report()

if __name__ == "__main__":
    import uvicorn
    uvicorn.run("main:app", host="0.0.0.0", port=8000, workers=4)
```

---

## Summary

You've mastered the advanced FastAPI ecosystem:

1. **Custom Responses** — `ORJSONResponse` for 2–10× faster JSON, `UJSONResponse`, binary MessagePack for microservices, custom serializers for complex types (Decimal, UUID, datetime, sets)
2. **Custom Requests** — Enriched request classes with domain properties (client IP, API version, language), preprocessing middleware that parses and caches body once, and route-level validator dependencies
3. **Metaprogramming** — Dynamic route generation from config with proper closure capture, runtime Pydantic model creation with `create_model`, model extension, and API client code generation via introspection
4. **Plugin Systems** — Abstract `Plugin` base class, `PluginManager` with module loading, plugin-owned routes + middleware + lifecycle hooks, and a complete analytics + rate-limit plugin example
5. **Performance Profiling** — cProfile middleware with `?profile=true`, on-demand endpoint profiler, memory tracing with `tracemalloc`, system metrics with `psutil`, and a `PerformanceAnalyzer` with P95/P99 percentiles
6. **Distributed Systems** — RabbitMQ and Kafka integration, decoupled event bus, CQRS with command/query buses, append-only event sourcing with aggregate replay, and saga orchestration with automatic compensation rollback

**When to Reach for These Patterns**:
- **ORJSONResponse**: Any high-traffic API — drop-in, zero-risk improvement
- **Event Sourcing**: Audit trails, financial ledgers, undo/redo systems
- **CQRS**: Read-heavy APIs where the read model can be denormalized
- **Saga**: Multi-service transactions (checkout, booking, onboarding flows)
- **Plugins**: Internal platforms, extensible SaaS backends