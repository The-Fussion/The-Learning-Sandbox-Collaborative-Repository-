23.1 Migrating to FastAPI
From Flask
Systematic steps for migrating a Flask application.
python# ================================================================
# FLASK ORIGINAL
# ================================================================
from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy

flask_app = Flask(__name__)
db        = SQLAlchemy(flask_app)

@flask_app.route("/users", methods=["GET"])
def get_users():
    users = User.query.all()
    return jsonify([u.to_dict() for u in users])

@flask_app.route("/users/<int:user_id>", methods=["GET"])
def get_user(user_id):
    user = User.query.get_or_404(user_id)
    return jsonify(user.to_dict())

@flask_app.route("/users", methods=["POST"])
def create_user():
    data = request.get_json()
    user = User(username=data["username"], email=data["email"])
    db.session.add(user)
    db.session.commit()
    return jsonify(user.to_dict()), 201

@flask_app.errorhandler(404)
def not_found(e):
    return jsonify({"error": "Not found"}), 404


# ================================================================
# FASTAPI EQUIVALENT
# ================================================================
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from pydantic import BaseModel

app = FastAPI()

# Pydantic replaces manual .to_dict() serialization
class UserOut(BaseModel):
    id: int
    username: str
    email: str
    model_config = {"from_attributes": True}

class UserCreate(BaseModel):
    username: str
    email: str

@app.get("/users", response_model=list[UserOut])
async def get_users(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User))
    return result.scalars().all()

@app.get("/users/{user_id}", response_model=UserOut)
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).where(User.id == user_id))
    user   = result.scalar_one_or_none()
    if not user:
        raise HTTPException(404, "User not found")
    return user

@app.post("/users", response_model=UserOut, status_code=201)
async def create_user(user_in: UserCreate, db: AsyncSession = Depends(get_db)):
    user = User(username=user_in.username, email=user_in.email)
    db.add(user)
    await db.commit()
    await db.refresh(user)
    return user

# Flask Blueprint → FastAPI APIRouter
# flask:   @blueprint.route("/items", methods=["GET"])
# fastapi: @router.get("/items")
from fastapi import APIRouter
router = APIRouter(prefix="/items", tags=["Items"])
app.include_router(router)

# Flask g / before_request → FastAPI dependencies
# flask:   @app.before_request \n def auth(): g.user = ...
# fastapi: async def get_current_user(): ...  → Depends(get_current_user)
From Django
Migrating from Django REST Framework to FastAPI.
python# ================================================================
# DJANGO REST FRAMEWORK ORIGINAL
# ================================================================
# views.py
from rest_framework import viewsets, status
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated

class UserViewSet(viewsets.ViewSet):
    permission_classes = [IsAuthenticated]

    def list(self, request):
        users = User.objects.all()
        serializer = UserSerializer(users, many=True)
        return Response(serializer.data)

    def retrieve(self, request, pk=None):
        try:
            user = User.objects.get(pk=pk)
        except User.DoesNotExist:
            return Response(status=status.HTTP_404_NOT_FOUND)
        return Response(UserSerializer(user).data)

    def create(self, request):
        serializer = UserSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


# ================================================================
# FASTAPI EQUIVALENT
# ================================================================
# DRF Serializer      → Pydantic BaseModel
# DRF ViewSet         → FastAPI APIRouter + route functions
# DRF permission_classes → FastAPI Depends()
# Django ORM          → SQLAlchemy (async) or Tortoise ORM
# Django migrations   → Alembic
# Django signals      → FastAPI events / background tasks
# Django middleware   → FastAPI middleware

from fastapi import APIRouter, Depends, HTTPException, status
from pydantic import BaseModel

router = APIRouter(prefix="/users", tags=["Users"])

class UserOut(BaseModel):
    id: int
    username: str
    email: str
    model_config = {"from_attributes": True}

class UserCreate(BaseModel):
    username: str
    email: str
    password: str

# DRF permission_classes = [IsAuthenticated]
# → FastAPI dependency
def is_authenticated(current_user: User = Depends(get_current_user)):
    return current_user

@router.get("/", response_model=list[UserOut], dependencies=[Depends(is_authenticated)])
async def list_users(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User))
    return result.scalars().all()

@router.get("/{user_id}", response_model=UserOut)
async def retrieve_user(
    user_id: int,
    db: AsyncSession = Depends(get_db),
    _: User = Depends(is_authenticated)
):
    result = await db.execute(select(User).where(User.id == user_id))
    user   = result.scalar_one_or_none()
    if not user:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "User not found")
    return user

@router.post("/", response_model=UserOut, status_code=status.HTTP_201_CREATED)
async def create_user(user_in: UserCreate, db: AsyncSession = Depends(get_db)):
    user = User(username=user_in.username, email=user_in.email,
                hashed_password=hash_password(user_in.password))
    db.add(user)
    await db.commit()
    await db.refresh(user)
    return user
Migration Strategies
python"""
MIGRATION APPROACH OPTIONS

1. BIG BANG — Rewrite everything at once
   ✓ Clean slate, modern architecture
   ✗ Long development time, high risk

2. STRANGLER FIG — Incremental, route-by-route
   ✓ Low risk, continuous delivery
   ✗ Longer overall timeline

3. PARALLEL RUN — Run both apps simultaneously
   ✓ Easy rollback
   ✗ Double maintenance burden

Recommended: Strangler Fig
"""

# ── Strangler Fig with Nginx ───────────────────────────────────
"""
# nginx.conf
upstream legacy_flask {
    server localhost:5000;
}
upstream new_fastapi {
    server localhost:8000;
}

server {
    listen 80;

    # New routes → FastAPI
    location /api/v2/     { proxy_pass http://new_fastapi; }
    location /api/users/  { proxy_pass http://new_fastapi; }   # migrated
    location /api/posts/  { proxy_pass http://new_fastapi; }   # migrated

    # Legacy routes → Flask (still being migrated)
    location /api/        { proxy_pass http://legacy_flask; }
}
"""

# ── Migration checklist per endpoint ───────────────────────────
"""
For each endpoint:
  1.  Write Pydantic schemas (replaces serializers/marshmallow)
  2.  Port business logic (move to service layer)
  3.  Write SQLAlchemy async models (or reuse with sync adapter)
  4.  Write FastAPI route
  5.  Write integration tests
  6.  Switch Nginx proxy rule
  7.  Monitor error rates for 24 h
  8.  Decommission old route
"""
Gradual Migration with Compatibility Layer
python# ── Run Flask and FastAPI side-by-side using WSGI/ASGI adapter ─

from fastapi import FastAPI
from fastapi.middleware.wsgi import WSGIMiddleware
from flask import Flask

# Existing Flask app (legacy)
flask_app = Flask(__name__)

@flask_app.route("/legacy/hello")
def legacy_hello():
    return {"message": "Hello from Flask"}

# New FastAPI app
fastapi_app = FastAPI()

@fastapi_app.get("/new/hello")
async def new_hello():
    return {"message": "Hello from FastAPI"}

# Mount Flask inside FastAPI — route legacy traffic through FastAPI
fastapi_app.mount("/legacy", WSGIMiddleware(flask_app))

# Run: uvicorn main:fastapi_app
# GET /new/hello   → FastAPI handler
# GET /legacy/hello → Flask handler (via WSGIMiddleware)


# ── Shared authentication during migration ─────────────────────
from functools import wraps
from jose import jwt

# Flask decorator
def flask_login_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        token = request.headers.get("Authorization", "").replace("Bearer ", "")
        user  = verify_token(token)   # shared verify_token function
        if not user:
            return jsonify({"error": "Unauthorized"}), 401
        return f(*args, **kwargs)
    return decorated

# FastAPI dependency — same verify_token function
async def fastapi_get_current_user(token: str = Depends(oauth2_scheme)):
    user = verify_token(token)       # reuse same function
    if not user:
        raise HTTPException(401, "Unauthorized")
    return user

23.2 Integration with Existing Systems
Integrating with Legacy Systems
pythonfrom fastapi import FastAPI, HTTPException
import httpx, asyncio, xmltodict

app = FastAPI()

# ── REST adapter for SOAP / XML legacy API ─────────────────────
class LegacySOAPClient:
    def __init__(self, wsdl_url: str):
        self.wsdl_url = wsdl_url
        self.headers  = {"Content-Type": "text/xml; charset=utf-8"}

    async def call(self, action: str, payload: str) -> dict:
        soap_body = f"""<?xml version="1.0" encoding="utf-8"?>
        <soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
          <soap:Body>{payload}</soap:Body>
        </soap:Envelope>"""

        async with httpx.AsyncClient() as client:
            response = await client.post(
                self.wsdl_url,
                content=soap_body,
                headers={**self.headers, "SOAPAction": action},
                timeout=30.0,
            )
            response.raise_for_status()
            return xmltodict.parse(response.text)

soap_client = LegacySOAPClient("http://legacy-system/soap")

@app.get("/legacy/users/{user_id}")
async def get_legacy_user(user_id: int):
    """Wrap legacy SOAP endpoint as REST"""
    payload = f"<GetUser><UserId>{user_id}</UserId></GetUser>"
    try:
        result = await soap_client.call("GetUser", payload)
        body   = result["Envelope"]["Body"]["GetUserResponse"]
        return {
            "id":    body["UserId"],
            "name":  body["UserName"],
            "email": body["Email"],
        }
    except httpx.HTTPError as e:
        raise HTTPException(503, f"Legacy service unavailable: {e}")


# ── Database adapter for legacy schema ─────────────────────────
from sqlalchemy import Table, Column, Integer, String, MetaData

# Reflect legacy DB schema (don't rewrite it, just read it)
legacy_engine  = create_async_engine("mssql+aioodbc://...")
legacy_meta    = MetaData()

async def reflect_legacy_schema():
    async with legacy_engine.connect() as conn:
        await conn.run_sync(legacy_meta.reflect)

legacy_users = Table("tbl_Users", legacy_meta, autoload=True)

@app.get("/legacy-db/users/{user_id}")
async def get_from_legacy_db(user_id: int):
    async with legacy_engine.connect() as conn:
        result = await conn.execute(
            legacy_users.select().where(legacy_users.c.UserID == user_id)
        )
        row = result.fetchone()
    if not row:
        raise HTTPException(404, "User not found")
    return {"id": row.UserID, "name": row.UserName}


# ── Event sourcing bridge ──────────────────────────────────────
import aio_pika, json

async def listen_to_legacy_events():
    """Read events from a legacy RabbitMQ queue"""
    connection = await aio_pika.connect_robust(settings.LEGACY_RABBITMQ_URL)
    channel    = await connection.channel()
    queue      = await channel.declare_queue("legacy.events", durable=True)

    async for message in queue:
        async with message.process():
            event = json.loads(message.body)
            await handle_legacy_event(event)

async def handle_legacy_event(event: dict):
    if event["type"] == "USER_CREATED":
        # Sync to new system
        async with AsyncSession(engine) as db:
            user = User(
                legacy_id=event["data"]["id"],
                username=event["data"]["username"],
                email=event["data"]["email"],
            )
            db.add(user)
            await db.commit()

@app.on_event("startup")
async def start_event_bridge():
    asyncio.create_task(listen_to_legacy_events())
Reverse Proxy Configuration
nginx# ── Nginx production configuration ────────────────────────────
# /etc/nginx/sites-available/fastapi

upstream fastapi_backend {
    least_conn;
    server 127.0.0.1:8001 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8002 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8003 max_fails=3 fail_timeout=30s;
    keepalive 64;
}

server {
    listen 443 ssl http2;
    server_name api.myapp.com;

    # SSL
    ssl_certificate     /etc/letsencrypt/live/api.myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.myapp.com/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req zone=api burst=20 nodelay;

    # WebSocket support
    location /ws/ {
        proxy_pass http://fastapi_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 3600s;
    }

    # SSE support
    location /events {
        proxy_pass         http://fastapi_backend;
        proxy_buffering    off;
        proxy_cache        off;
        proxy_read_timeout 3600s;
        proxy_set_header   X-Accel-Buffering no;
    }

    # API
    location / {
        proxy_pass       http://fastapi_backend;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_connect_timeout 60s;
        proxy_read_timeout    120s;
        proxy_send_timeout    60s;
    }
}

server {
    listen 80;
    server_name api.myapp.com;
    return 301 https://$host$request_uri;
}
python# ── FastAPI — trust proxy headers ─────────────────────────────
from fastapi import FastAPI, Request
from uvicorn.middleware.proxy_headers import ProxyHeadersMiddleware

app = FastAPI()
app.add_middleware(ProxyHeadersMiddleware, trusted_hosts="*")

@app.get("/client-ip")
async def client_ip(request: Request):
    # Now returns real IP even behind Nginx
    return {"ip": request.client.host}
API Gateway Integration
pythonfrom fastapi import FastAPI, Request, HTTPException, Depends
from fastapi.responses import JSONResponse
import httpx, time

app = FastAPI()

# ── AWS API Gateway — Lambda handler (via Mangum) ─────────────
from mangum import Mangum

handler = Mangum(app, lifespan="off")   # AWS Lambda entry point

# ── Kong API Gateway — custom plugin headers ───────────────────
@app.middleware("http")
async def handle_kong_headers(request: Request, call_next):
    # Kong injects these headers after auth plugin runs
    user_id   = request.headers.get("X-Consumer-ID")
    user_name = request.headers.get("X-Consumer-Username")
    api_key   = request.headers.get("X-Credential-Identifier")

    # Make them available to routes via request.state
    request.state.user_id   = user_id
    request.state.user_name = user_name

    return await call_next(request)

@app.get("/profile")
async def profile(request: Request):
    return {"user_id": request.state.user_id, "username": request.state.user_name}

# ── Rate limit headers from API Gateway ───────────────────────
@app.middleware("http")
async def forward_rate_limit_headers(request: Request, call_next):
    response = await call_next(request)
    # Forward rate limit headers from gateway to client
    for key in ("X-RateLimit-Limit", "X-RateLimit-Remaining", "X-RateLimit-Reset"):
        if key in request.headers:
            response.headers[key] = request.headers[key]
    return response

# ── Service mesh (Istio / Envoy) — propagate trace headers ─────
TRACE_HEADERS = [
    "x-request-id", "x-b3-traceid", "x-b3-spanid",
    "x-b3-parentspanid", "x-b3-sampled", "x-b3-flags",
]

@app.middleware("http")
async def propagate_trace_headers(request: Request, call_next):
    request.state.trace_headers = {
        h: request.headers[h] for h in TRACE_HEADERS if h in request.headers
    }
    return await call_next(request)

async def call_downstream(url: str, request: Request):
    """Propagate tracing headers to downstream services"""
    async with httpx.AsyncClient() as client:
        return await client.get(url, headers=request.state.trace_headers)
Microservices Communication
pythonfrom fastapi import FastAPI, HTTPException
import httpx, asyncio
from tenacity import retry, stop_after_attempt, wait_exponential

app = FastAPI()

# ── Resilient HTTP client with retry + circuit breaker ─────────
class ResilientClient:
    def __init__(self, base_url: str, service_name: str):
        self.base_url     = base_url
        self.service_name = service_name
        self.failures     = 0
        self.open_until   = 0
        self.threshold    = 5
        self.timeout_sec  = 60

    def _circuit_open(self) -> bool:
        if self.failures >= self.threshold:
            if time.time() < self.open_until:
                return True
            self.failures = 0   # Half-open: allow one attempt
        return False

    @retry(stop=stop_after_attempt(3), wait=wait_exponential(min=1, max=8))
    async def get(self, path: str, **kw) -> dict:
        if self._circuit_open():
            raise HTTPException(503, f"{self.service_name} circuit open")
        try:
            async with httpx.AsyncClient(timeout=10.0) as client:
                r = await client.get(f"{self.base_url}{path}", **kw)
                r.raise_for_status()
                self.failures = 0
                return r.json()
        except Exception:
            self.failures += 1
            self.open_until = time.time() + self.timeout_sec
            raise

user_svc  = ResilientClient(settings.USER_SERVICE_URL,  "UserService")
order_svc = ResilientClient(settings.ORDER_SERVICE_URL, "OrderService")

# ── Parallel service calls with graceful degradation ───────────
@app.get("/dashboard/{user_id}")
async def dashboard(user_id: int):
    # Fetch all in parallel; don't let one failure break the page
    user_task   = user_svc.get(f"/users/{user_id}")
    orders_task = order_svc.get(f"/orders?user_id={user_id}")

    user_result, orders_result = await asyncio.gather(
        user_task, orders_task, return_exceptions=True
    )

    return {
        "user":   user_result   if not isinstance(user_result,   Exception) else None,
        "orders": orders_result if not isinstance(orders_result, Exception) else [],
        "errors": [str(e) for e in [user_result, orders_result] if isinstance(e, Exception)],
    }

# ── Service discovery via consul ───────────────────────────────
class ServiceDiscovery:
    def __init__(self, consul_url: str):
        self.consul_url = consul_url
        self._cache: dict[str, str] = {}

    async def get_service_url(self, name: str) -> str:
        if name not in self._cache:
            async with httpx.AsyncClient() as client:
                r = await client.get(f"{self.consul_url}/v1/catalog/service/{name}")
                services = r.json()
                if not services:
                    raise HTTPException(503, f"Service '{name}' not found in registry")
                svc = services[0]
                self._cache[name] = f"http://{svc['Address']}:{svc['ServicePort']}"
        return self._cache[name]

discovery = ServiceDiscovery(settings.CONSUL_URL)

@app.get("/users/{user_id}")
async def get_user_via_discovery(user_id: int):
    url = await discovery.get_service_url("user-service")
    async with httpx.AsyncClient() as client:
        r = await client.get(f"{url}/users/{user_id}")
        return r.json()

Quick Reference Cheat Sheet
python# ── Common error → fix mapping ─────────────────────────────────
"""
ERROR                                  FIX
─────────────────────────────────────────────────────────────────
RuntimeError: no running event loop    Use await / asyncio.run()
coroutine was never awaited            Add await keyword
TimeoutError (DB pool)                 Increase pool_size or fix leaks
422 Unprocessable Entity               Check Pydantic schema field types
CORS blocked by browser                Add CORSMiddleware (before routers)
ImportError: circular import           Use deps.py shared module
ModuleNotFoundError                    Activate virtualenv, pip install
Slow endpoint (> 1s)                   Profile with cProfile, check N+1
Memory growing unbounded               Use TTLCache, bounded deque
Connection refused                     Check service URL & port, health check
"""

# ── Flask → FastAPI mapping ────────────────────────────────────
"""
FLASK                          FASTAPI
─────────────────────────────────────────────────────────────────
@app.route("/", methods=["GET"]) @app.get("/")
request.get_json()             body parameter (Pydantic model)
request.args.get("q")          q: str = Query(None)
request.form.get("field")      field: str = Form(...)
jsonify(data)                  return data  (auto-serialized)
abort(404)                     raise HTTPException(404, "...")
@app.before_request            Depends() dependency
@app.errorhandler(404)         @app.exception_handler(404)
Blueprint                      APIRouter
g.user                         request.state.user
Flask-SQLAlchemy               SQLAlchemy async + Depends(get_db)
Flask-JWT                      python-jose + Depends(get_current_user)
"""

# ── Django → FastAPI mapping ───────────────────────────────────
"""
DJANGO / DRF                   FASTAPI
─────────────────────────────────────────────────────────────────
Serializer                     Pydantic BaseModel
ViewSet                        APIRouter + route functions
permission_classes             Depends(require_permission(...))
django.db.models               SQLAlchemy ORM (async)
makemigrations / migrate       alembic revision / alembic upgrade
signals                        Background tasks / event bus
django-environ                 pydantic-settings BaseSettings
urls.py                        app.include_router()
Django middleware               FastAPI middleware / Depends
"""

Key Takeaways:

Always await coroutines and never block the event loop with sync I/O
Use try/finally in dependencies to guarantee resource cleanup
Add CORSMiddleware before including routers
Profile with cProfile, catch N+1 with eager loading, bound caches with TTL
Migrate incrementally using Strangler Fig + Nginx routing rules
Wrap legacy SOAP/XML APIs with thin FastAPI REST adapters
Use resilient HTTP clients with retry and circuit breaker for microservices

Additional Resources:

FastAPI Debugging
SQLAlchemy Async
Alembic Migrations
Strangler Fig Pattern
Tenacity Retry Library
structlog