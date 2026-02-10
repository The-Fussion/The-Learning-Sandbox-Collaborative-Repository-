# Middleware

## 12.1 Middleware Basics

### What is Middleware?

Middleware is software that sits between the client request and your application's route handlers. It processes every request before it reaches your endpoints and every response before it's sent back to the client.

**Think of middleware as layers of an onion**:
```
Client Request
    ↓
[Middleware 1] ← First layer
    ↓
[Middleware 2] ← Second layer
    ↓
[Middleware 3] ← Third layer
    ↓
[Route Handler] ← Core
    ↓
[Middleware 3] → Third layer (response)
    ↓
[Middleware 2] → Second layer (response)
    ↓
[Middleware 1] → First layer (response)
    ↓
Client Response
```

**Common Middleware Uses**:
- Authentication and authorization
- Request/response logging
- CORS handling
- Compression
- Rate limiting
- Request timing
- Error handling
- Request ID tracking
- Security headers

**Key Characteristics**:
- Executes on **every request**
- Can **modify requests** before they reach route handlers
- Can **modify responses** before they're sent to clients
- Can **short-circuit** and return early
- Executes in **defined order**

### Middleware Execution Order

Middleware executes in the order it's added to the application.

**Example**:
```python
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware
import time

app = FastAPI()

class FirstMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        print("1. First middleware - before request")
        response = await call_next(request)
        print("6. First middleware - after request")
        return response

class SecondMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        print("2. Second middleware - before request")
        response = await call_next(request)
        print("5. Second middleware - after request")
        return response

class ThirdMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        print("3. Third middleware - before request")
        response = await call_next(request)
        print("4. Third middleware - after request")
        return response

# Add middleware in order
app.add_middleware(FirstMiddleware)
app.add_middleware(SecondMiddleware)
app.add_middleware(ThirdMiddleware)

@app.get("/")
async def root():
    print("   → Route handler executing")
    return {"message": "Hello"}

# Console output when accessing /:
# 1. First middleware - before request
# 2. Second middleware - before request
# 3. Third middleware - before request
#    → Route handler executing
# 4. Third middleware - after request
# 5. Second middleware - after request
# 6. First middleware - after request
```

**Important**: Middleware is added in reverse order of execution. The **last middleware added executes first**.

### Request and Response Flow

**Request Flow** (top to bottom):
```python
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware

app = FastAPI()

class RequestFlowMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # 1. Request comes in
        print(f"Request: {request.method} {request.url.path}")
        
        # 2. You can modify the request
        request.state.custom_data = "Added by middleware"
        
        # 3. You can read headers, body, etc.
        print(f"Headers: {request.headers.get('user-agent')}")
        
        # 4. Pass to next middleware or route handler
        response = await call_next(request)
        
        # 5. Response comes back
        print(f"Status: {response.status_code}")
        
        # 6. You can modify the response
        response.headers["X-Custom-Header"] = "Added by middleware"
        
        # 7. Return response
        return response

app.add_middleware(RequestFlowMiddleware)

@app.get("/test")
async def test(request: Request):
    # Access middleware data
    custom = request.state.custom_data
    return {"message": "Hello", "custom": custom}
```

**Response Flow** (bottom to top):
```python
class ResponseFlowMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Before route handler
        start_time = time.time()
        
        # Call next middleware/handler
        response = await call_next(request)
        
        # After route handler (modify response)
        process_time = time.time() - start_time
        response.headers["X-Process-Time"] = str(process_time)
        
        return response
```

### Built-in Middleware

FastAPI includes several built-in middleware from Starlette:

**1. CORSMiddleware** - Handle Cross-Origin Resource Sharing
**2. GZipMiddleware** - Compress responses
**3. TrustedHostMiddleware** - Enforce allowed hosts
**4. HTTPSRedirectMiddleware** - Force HTTPS
**5. SessionMiddleware** - Cookie-based sessions

**Example of Built-in Middleware**:
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.gzip import GZipMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware

app = FastAPI()

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# GZip compression
app.add_middleware(GZipMiddleware, minimum_size=1000)

# Trusted hosts
app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["example.com", "*.example.com"]
)
```

---

## 12.2 Custom Middleware

### Creating Custom Middleware

There are two ways to create custom middleware in FastAPI:
1. **Middleware classes** (more flexible)
2. **Middleware functions** (simpler)

### Middleware Classes

**Basic Middleware Class**:
```python
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware

app = FastAPI()

class CustomMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Code before route handler
        print("Before request")
        
        # Call the route handler
        response = await call_next(request)
        
        # Code after route handler
        print("After request")
        
        return response

app.add_middleware(CustomMiddleware)
```

**Middleware with Initialization Parameters**:
```python
class ConfigurableMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, prefix: str, debug: bool = False):
        super().__init__(app)
        self.prefix = prefix
        self.debug = debug
    
    async def dispatch(self, request: Request, call_next):
        if self.debug:
            print(f"{self.prefix}: {request.url.path}")
        
        response = await call_next(request)
        return response

# Add with parameters
app.add_middleware(
    ConfigurableMiddleware,
    prefix="API",
    debug=True
)
```

**Middleware that Modifies Requests**:
```python
class RequestModifierMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Add custom data to request
        request.state.request_id = str(uuid.uuid4())
        request.state.start_time = time.time()
        
        # Modify headers (create new request)
        # Note: request is immutable, but state is mutable
        
        response = await call_next(request)
        return response

@app.get("/")
async def root(request: Request):
    # Access middleware data
    return {
        "request_id": request.state.request_id,
        "start_time": request.state.start_time
    }
```

**Middleware that Modifies Responses**:
```python
class ResponseModifierMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        
        # Add custom headers
        response.headers["X-Custom-Header"] = "MyValue"
        response.headers["X-Request-ID"] = request.state.request_id
        
        # Add timing
        if hasattr(request.state, "start_time"):
            process_time = time.time() - request.state.start_time
            response.headers["X-Process-Time"] = f"{process_time:.4f}"
        
        return response
```

**Middleware that Short-Circuits**:
```python
from fastapi.responses import JSONResponse

class AuthMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Check authentication
        api_key = request.headers.get("X-API-Key")
        
        if not api_key or api_key != "secret-key":
            # Short-circuit - don't call route handler
            return JSONResponse(
                status_code=401,
                content={"error": "Invalid API key"}
            )
        
        # Continue to route handler
        response = await call_next(request)
        return response
```

### Middleware Functions

**Pure Function Middleware**:
```python
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware

app = FastAPI()

@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start_time = time.time()
    
    response = await call_next(request)
    
    process_time = time.time() - start_time
    response.headers["X-Process-Time"] = str(process_time)
    
    return response

@app.middleware("http")
async def log_requests(request: Request, call_next):
    print(f"{request.method} {request.url.path}")
    response = await call_next(request)
    print(f"Status: {response.status_code}")
    return response
```

**Function Middleware with Early Return**:
```python
from fastapi.responses import JSONResponse

@app.middleware("http")
async def check_maintenance_mode(request: Request, call_next):
    # Check if in maintenance mode
    if os.getenv("MAINTENANCE_MODE") == "true":
        return JSONResponse(
            status_code=503,
            content={"error": "Service under maintenance"}
        )
    
    return await call_next(request)
```

### Async Middleware

All middleware in FastAPI should be async for best performance.

**Async Middleware with Async Operations**:
```python
import aioredis
from fastapi import FastAPI, Request

app = FastAPI()

class CacheMiddleware(BaseHTTPMiddleware):
    def __init__(self, app):
        super().__init__(app)
        self.redis = None
    
    async def dispatch(self, request: Request, call_next):
        # Only cache GET requests
        if request.method != "GET":
            return await call_next(request)
        
        # Initialize Redis connection if needed
        if not self.redis:
            self.redis = await aioredis.create_redis_pool("redis://localhost")
        
        # Check cache
        cache_key = f"cache:{request.url.path}"
        cached = await self.redis.get(cache_key)
        
        if cached:
            return JSONResponse(
                content=json.loads(cached),
                headers={"X-Cache": "HIT"}
            )
        
        # Get response from handler
        response = await call_next(request)
        
        # Cache response (simplified - real implementation more complex)
        if response.status_code == 200:
            # Would need to read response body here
            pass
        
        return response

app.add_middleware(CacheMiddleware)
```

**Async Middleware with Background Tasks**:
```python
import asyncio

class AsyncLoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Start async logging
        log_task = asyncio.create_task(
            self.log_request(request)
        )
        
        response = await call_next(request)
        
        # Wait for logging to complete
        await log_task
        
        return response
    
    async def log_request(self, request: Request):
        """Async logging operation."""
        await asyncio.sleep(0.1)  # Simulate async I/O
        print(f"Logged: {request.method} {request.url.path}")
```

### Middleware with State

**Shared State Across Requests**:
```python
class StatefulMiddleware(BaseHTTPMiddleware):
    def __init__(self, app):
        super().__init__(app)
        self.request_count = 0
        self.request_times = []
    
    async def dispatch(self, request: Request, call_next):
        # Update shared state
        self.request_count += 1
        start_time = time.time()
        
        # Add to request
        request.state.request_number = self.request_count
        
        response = await call_next(request)
        
        # Track timing
        elapsed = time.time() - start_time
        self.request_times.append(elapsed)
        
        # Add stats to response
        response.headers["X-Request-Number"] = str(self.request_count)
        if self.request_times:
            avg_time = sum(self.request_times) / len(self.request_times)
            response.headers["X-Avg-Response-Time"] = f"{avg_time:.4f}"
        
        return response

app.add_middleware(StatefulMiddleware)

@app.get("/stats")
async def get_stats():
    # Access middleware state (would need reference to middleware instance)
    return {"message": "See response headers for stats"}
```

**Application State Access**:
```python
class AppStateMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Access app state
        if not hasattr(request.app.state, "request_count"):
            request.app.state.request_count = 0
        
        request.app.state.request_count += 1
        
        response = await call_next(request)
        response.headers["X-Total-Requests"] = str(request.app.state.request_count)
        
        return response
```

---

## 12.3 Common Middleware

### CORS Middleware

CORS (Cross-Origin Resource Sharing) allows your API to be accessed from different domains.

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# Basic CORS - allow all
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Specific CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://localhost:3000",
        "https://myapp.com",
        "https://*.myapp.com"  # Wildcard subdomain
    ],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["*"],
    expose_headers=["X-Request-ID"],
    max_age=3600,  # Cache preflight requests for 1 hour
)

@app.get("/")
async def root():
    return {"message": "CORS enabled"}
```

**Custom CORS Logic**:
```python
from starlette.middleware.base import BaseHTTPMiddleware

class CustomCORSMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        origin = request.headers.get("origin")
        
        # Custom logic for allowed origins
        allowed_origins = ["http://localhost:3000"]
        if request.headers.get("x-api-key") == "premium-key":
            allowed_origins.append("https://premium.example.com")
        
        response = await call_next(request)
        
        if origin in allowed_origins:
            response.headers["Access-Control-Allow-Origin"] = origin
            response.headers["Access-Control-Allow-Credentials"] = "true"
        
        return response
```

### GZip Middleware

Compress responses to reduce bandwidth.

```python
from fastapi import FastAPI
from fastapi.middleware.gzip import GZipMiddleware

app = FastAPI()

# Add GZip compression
app.add_middleware(
    GZipMiddleware,
    minimum_size=1000  # Only compress responses > 1000 bytes
)

@app.get("/large-data")
async def get_large_data():
    # This will be compressed
    return {"data": "x" * 10000}
```

**Custom Compression Middleware**:
```python
import gzip
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import Response

class CustomCompressionMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        
        # Check if client accepts gzip
        accept_encoding = request.headers.get("accept-encoding", "")
        
        if "gzip" not in accept_encoding:
            return response
        
        # Only compress certain content types
        content_type = response.headers.get("content-type", "")
        if not any(ct in content_type for ct in ["application/json", "text/"]):
            return response
        
        # Get response body
        body = b""
        async for chunk in response.body_iterator:
            body += chunk
        
        # Compress if large enough
        if len(body) > 500:
            compressed = gzip.compress(body)
            
            return Response(
                content=compressed,
                status_code=response.status_code,
                headers={
                    **dict(response.headers),
                    "Content-Encoding": "gzip",
                    "Content-Length": str(len(compressed))
                },
                media_type=response.media_type
            )
        
        return Response(
            content=body,
            status_code=response.status_code,
            headers=dict(response.headers),
            media_type=response.media_type
        )
```

### Trusted Host Middleware

Protect against HTTP Host header attacks.

```python
from fastapi import FastAPI
from fastapi.middleware.trustedhost import TrustedHostMiddleware

app = FastAPI()

app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=[
        "example.com",
        "www.example.com",
        "*.example.com",  # Wildcard for subdomains
        "localhost",
        "127.0.0.1"
    ]
)

@app.get("/")
async def root():
    return {"message": "Trusted host"}
```

### Session Middleware

Cookie-based sessions.

```python
from fastapi import FastAPI, Request
from starlette.middleware.sessions import SessionMiddleware

app = FastAPI()

app.add_middleware(
    SessionMiddleware,
    secret_key="your-secret-key-here",  # Use strong secret in production
    max_age=3600,  # Session expires after 1 hour
    same_site="lax",
    https_only=False  # Set to True in production with HTTPS
)

@app.get("/set-session")
async def set_session(request: Request):
    request.session["user_id"] = 123
    request.session["username"] = "alice"
    return {"message": "Session set"}

@app.get("/get-session")
async def get_session(request: Request):
    user_id = request.session.get("user_id")
    username = request.session.get("username")
    return {"user_id": user_id, "username": username}

@app.get("/clear-session")
async def clear_session(request: Request):
    request.session.clear()
    return {"message": "Session cleared"}
```

### Logging Middleware

Log all requests and responses.

```python
import logging
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

app = FastAPI()

class LoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Log request
        logger.info(
            f"Request: {request.method} {request.url.path} "
            f"from {request.client.host}"
        )
        
        # Log headers (be careful with sensitive data)
        logger.debug(f"Headers: {dict(request.headers)}")
        
        # Process request
        start_time = time.time()
        response = await call_next(request)
        process_time = time.time() - start_time
        
        # Log response
        logger.info(
            f"Response: {response.status_code} "
            f"in {process_time:.4f}s"
        )
        
        return response

app.add_middleware(LoggingMiddleware)
```

**Structured Logging Middleware**:
```python
import json
from datetime import datetime

class StructuredLoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        request_id = str(uuid.uuid4())
        start_time = time.time()
        
        # Add request ID to request
        request.state.request_id = request_id
        
        response = await call_next(request)
        
        # Structured log entry
        log_entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "request_id": request_id,
            "method": request.method,
            "path": request.url.path,
            "query_params": str(request.query_params),
            "client_ip": request.client.host,
            "status_code": response.status_code,
            "duration_ms": (time.time() - start_time) * 1000,
            "user_agent": request.headers.get("user-agent")
        }
        
        logger.info(json.dumps(log_entry))
        
        return response
```

### Authentication Middleware

```python
from fastapi import FastAPI, Request, HTTPException
from starlette.middleware.base import BaseHTTPMiddleware
import jwt

app = FastAPI()

SECRET_KEY = "your-secret-key"

class AuthenticationMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, exclude_paths: list = None):
        super().__init__(app)
        self.exclude_paths = exclude_paths or ["/login", "/docs", "/openapi.json"]
    
    async def dispatch(self, request: Request, call_next):
        # Skip authentication for excluded paths
        if request.url.path in self.exclude_paths:
            return await call_next(request)
        
        # Get token from header
        auth_header = request.headers.get("Authorization")
        
        if not auth_header or not auth_header.startswith("Bearer "):
            return JSONResponse(
                status_code=401,
                content={"error": "Missing or invalid authorization header"}
            )
        
        token = auth_header.replace("Bearer ", "")
        
        try:
            # Verify token
            payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
            
            # Add user info to request
            request.state.user = payload
            
        except jwt.ExpiredSignatureError:
            return JSONResponse(
                status_code=401,
                content={"error": "Token expired"}
            )
        except jwt.InvalidTokenError:
            return JSONResponse(
                status_code=401,
                content={"error": "Invalid token"}
            )
        
        response = await call_next(request)
        return response

app.add_middleware(
    AuthenticationMiddleware,
    exclude_paths=["/login", "/register", "/docs", "/openapi.json"]
)

@app.get("/protected")
async def protected_route(request: Request):
    return {"user": request.state.user}
```

### Request ID Middleware

Add unique ID to each request for tracing.

```python
import uuid
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware

app = FastAPI()

class RequestIDMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Generate or get request ID
        request_id = request.headers.get("X-Request-ID")
        
        if not request_id:
            request_id = str(uuid.uuid4())
        
        # Add to request state
        request.state.request_id = request_id
        
        # Process request
        response = await call_next(request)
        
        # Add to response headers
        response.headers["X-Request-ID"] = request_id
        
        return response

app.add_middleware(RequestIDMiddleware)

@app.get("/")
async def root(request: Request):
    return {"request_id": request.state.request_id}
```

### Timing Middleware

Track request processing time.

```python
import time
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware

app = FastAPI()

class TimingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Start timer
        start_time = time.time()
        
        # Process request
        response = await call_next(request)
        
        # Calculate time
        process_time = time.time() - start_time
        
        # Add timing headers
        response.headers["X-Process-Time"] = f"{process_time:.4f}"
        response.headers["X-Process-Time-Ms"] = f"{process_time * 1000:.2f}"
        
        # Log slow requests
        if process_time > 1.0:
            logger.warning(
                f"Slow request: {request.method} {request.url.path} "
                f"took {process_time:.4f}s"
            )
        
        return response

app.add_middleware(TimingMiddleware)
```

**Advanced Timing with Breakdown**:
```python
class DetailedTimingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        timings = {}
        
        # Overall start
        overall_start = time.time()
        
        # Middleware processing
        middleware_start = time.time()
        timings["middleware_pre"] = time.time() - middleware_start
        
        # Route handler
        handler_start = time.time()
        response = await call_next(request)
        timings["handler"] = time.time() - handler_start
        
        # Middleware post-processing
        middleware_post_start = time.time()
        timings["middleware_post"] = time.time() - middleware_post_start
        
        # Total time
        timings["total"] = time.time() - overall_start
        
        # Add to response
        response.headers["X-Timing-Total"] = f"{timings['total']:.4f}"
        response.headers["X-Timing-Handler"] = f"{timings['handler']:.4f}"
        
        return response
```

---

## Complete Example: Production-Ready Middleware Stack

```python
from fastapi import FastAPI, Request, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.gzip import GZipMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.middleware.sessions import SessionMiddleware
import time
import uuid
import logging
import json
from datetime import datetime

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(title="Production API")

# ========== Custom Middleware ==========

class RequestIDMiddleware(BaseHTTPMiddleware):
    """Add unique request ID to each request."""
    async def dispatch(self, request: Request, call_next):
        request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
        request.state.request_id = request_id
        
        response = await call_next(request)
        response.headers["X-Request-ID"] = request_id
        
        return response

class TimingMiddleware(BaseHTTPMiddleware):
    """Track request processing time."""
    async def dispatch(self, request: Request, call_next):
        start_time = time.time()
        request.state.start_time = start_time
        
        response = await call_next(request)
        
        process_time = time.time() - start_time
        response.headers["X-Process-Time"] = f"{process_time:.4f}"
        
        # Log slow requests
        if process_time > 1.0:
            logger.warning(
                f"Slow request [{request.state.request_id}]: "
                f"{request.method} {request.url.path} took {process_time:.2f}s"
            )
        
        return response

class StructuredLoggingMiddleware(BaseHTTPMiddleware):
    """Log requests in structured JSON format."""
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        
        # Create structured log
        log_data = {
            "timestamp": datetime.utcnow().isoformat(),
            "request_id": getattr(request.state, "request_id", None),
            "method": request.method,
            "path": request.url.path,
            "query": str(request.query_params),
            "client_ip": request.client.host if request.client else None,
            "user_agent": request.headers.get("user-agent"),
            "status_code": response.status_code,
            "duration": getattr(request.state, "start_time", None) and 
                       f"{time.time() - request.state.start_time:.4f}",
        }
        
        # Log based on status code
        if response.status_code >= 500:
            logger.error(json.dumps(log_data))
        elif response.status_code >= 400:
            logger.warning(json.dumps(log_data))
        else:
            logger.info(json.dumps(log_data))
        
        return response

class RateLimitMiddleware(BaseHTTPMiddleware):
    """Simple in-memory rate limiting."""
    def __init__(self, app, requests_per_minute: int = 60):
        super().__init__(app)
        self.requests_per_minute = requests_per_minute
        self.requests = {}  # ip -> [(timestamp, count)]
    
    async def dispatch(self, request: Request, call_next):
        client_ip = request.client.host if request.client else "unknown"
        current_time = time.time()
        
        # Clean old entries
        if client_ip in self.requests:
            self.requests[client_ip] = [
                (ts, count) for ts, count in self.requests[client_ip]
                if current_time - ts < 60
            ]
        
        # Count requests in last minute
        request_count = sum(
            count for ts, count in self.requests.get(client_ip, [])
        )
        
        if request_count >= self.requests_per_minute:
            return JSONResponse(
                status_code=429,
                content={"error": "Rate limit exceeded"},
                headers={"Retry-After": "60"}
            )
        
        # Add current request
        if client_ip not in self.requests:
            self.requests[client_ip] = []
        self.requests[client_ip].append((current_time, 1))
        
        response = await call_next(request)
        
        # Add rate limit headers
        response.headers["X-RateLimit-Limit"] = str(self.requests_per_minute)
        response.headers["X-RateLimit-Remaining"] = str(
            self.requests_per_minute - request_count - 1
        )
        
        return response

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    """Add security headers to all responses."""
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        
        # Security headers
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
        response.headers["Content-Security-Policy"] = "default-src 'self'"
        
        return response

# ========== Add Middleware in Order ==========

# 1. Security headers (first to process, last to execute)
app.add_middleware(SecurityHeadersMiddleware)

# 2. CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 3. Trusted hosts
app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["localhost", "127.0.0.1", "example.com"]
)

# 4. GZip compression
app.add_middleware(GZipMiddleware, minimum_size=1000)

# 5. Sessions
app.add_middleware(
    SessionMiddleware,
    secret_key="your-secret-key-change-this",
    max_age=3600
)

# 6. Rate limiting
app.add_middleware(RateLimitMiddleware, requests_per_minute=60)

# 7. Request ID
app.add_middleware(RequestIDMiddleware)

# 8. Timing
app.add_middleware(TimingMiddleware)

# 9. Logging (last middleware, logs everything)
app.add_middleware(StructuredLoggingMiddleware)

# ========== Routes ==========

@app.get("/")
async def root(request: Request):
    return {
        "message": "Hello World",
        "request_id": request.state.request_id
    }

@app.get("/slow")
async def slow_endpoint():
    """Endpoint that triggers slow request warning."""
    import asyncio
    await asyncio.sleep(1.5)
    return {"message": "This was slow"}

@app.get("/session")
async def session_test(request: Request):
    """Test session middleware."""
    count = request.session.get("count", 0)
    count += 1
    request.session["count"] = count
    return {"visit_count": count}

@app.get("/error")
async def error_endpoint():
    """Test error logging."""
    raise HTTPException(status_code=500, detail="Test error")

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## Summary

You've learned about Middleware in FastAPI:

1. **Middleware Basics**: What middleware is, execution order, request/response flow, and built-in middleware
2. **Custom Middleware**: Creating middleware classes and functions, async middleware, and stateful middleware
3. **Common Middleware**: CORS, GZip, Trusted Host, Sessions, Logging, Authentication, Request ID, and Timing

**Key Takeaways**:
- Middleware executes on **every request**
- Add middleware in **reverse order** of desired execution
- Use middleware for **cross-cutting concerns** (logging, auth, timing)
- Keep middleware **lightweight** - heavy processing slows all requests
- Middleware can **modify requests and responses**
- Use **async middleware** for best performance

Middleware is powerful for implementing application-wide functionality without cluttering your route handlers!