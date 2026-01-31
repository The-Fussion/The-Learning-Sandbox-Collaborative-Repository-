# Middleware and Request Processing in Python

## Table of Contents
1. [Introduction to Middleware](#introduction-to-middleware)
2. [Request/Response Pipeline](#requestresponse-pipeline)
3. [Middleware in Flask](#middleware-in-flask)
4. [Middleware in FastAPI](#middleware-in-fastapi)
5. [Middleware in Django](#middleware-in-django)
6. [Common Middleware Patterns](#common-middleware-patterns)
7. [Authentication Middleware](#authentication-middleware)
8. [Logging Middleware](#logging-middleware)
9. [Rate Limiting Middleware](#rate-limiting-middleware)
10. [CORS Middleware](#cors-middleware)
11. [Error Handling Middleware](#error-handling-middleware)
12. [Request/Response Modification](#requestresponse-modification)
13. [Performance Monitoring Middleware](#performance-monitoring-middleware)
14. [Security Middleware](#security-middleware)
15. [Custom Middleware Development](#custom-middleware-development)
16. [Best Practices](#best-practices)

---

## Introduction to Middleware

### What is Middleware?

Middleware is software that sits between the client request and the application's route handlers, processing requests and responses. It acts as a pipeline through which every request passes before reaching your application logic.

```
Client Request → Middleware 1 → Middleware 2 → ... → Route Handler
                      ↓              ↓                      ↓
Client Response ← Middleware 1 ← Middleware 2 ← ... ← Response
```

### Why Use Middleware?

- **Cross-cutting concerns**: Handle common functionality across all routes
- **Request preprocessing**: Validate, authenticate, or modify requests
- **Response postprocessing**: Transform, compress, or cache responses
- **Separation of concerns**: Keep business logic separate from infrastructure
- **Reusability**: Write once, apply to multiple routes or applications
- **Order of execution**: Control the exact sequence of request processing

### Common Use Cases

```python
# 1. Authentication
# Verify user identity before accessing protected routes

# 2. Logging
# Record requests, responses, and errors

# 3. Rate Limiting
# Prevent abuse by limiting requests per user

# 4. CORS
# Handle Cross-Origin Resource Sharing

# 5. Request/Response Transformation
# Modify headers, body, or format

# 6. Error Handling
# Catch and format errors consistently

# 7. Performance Monitoring
# Track response times and bottlenecks

# 8. Security
# Add security headers, sanitize input

# 9. Compression
# Compress responses to reduce bandwidth

# 10. Caching
# Cache responses for faster subsequent requests
```

---

## Request/Response Pipeline

### Understanding the Pipeline

```python
"""
Request Flow:
1. Client sends HTTP request
2. Web server receives request
3. Request enters middleware pipeline
4. Each middleware processes request sequentially
5. Request reaches route handler
6. Route handler generates response
7. Response passes back through middleware (reverse order)
8. Each middleware processes response
9. Final response sent to client

Example Flow:
Client
  ↓ Request
[WSGI/ASGI Server]
  ↓
[Middleware 1: CORS]        - Add CORS headers
  ↓
[Middleware 2: Auth]        - Verify JWT token
  ↓
[Middleware 3: Logging]     - Log request details
  ↓
[Route Handler]             - Process business logic
  ↓ Response
[Middleware 3: Logging]     - Log response details
  ↓
[Middleware 2: Auth]        - (no action on response)
  ↓
[Middleware 1: CORS]        - Add CORS headers to response
  ↓
Client
"""
```

### Middleware Execution Order

```python
# Middleware is executed in the order it's registered
# Request: A → B → C → Handler
# Response: Handler → C → B → A

app.add_middleware(MiddlewareA)  # Executes first for requests
app.add_middleware(MiddlewareB)  # Executes second
app.add_middleware(MiddlewareC)  # Executes third (closest to handler)
```

---

## Middleware in Flask

### WSGI Middleware

Flask uses WSGI (Web Server Gateway Interface), which defines how web servers communicate with web applications.

#### Basic WSGI Middleware Structure

```python
class BasicMiddleware:
    """Basic WSGI middleware structure."""
    
    def __init__(self, app):
        self.app = app
    
    def __call__(self, environ, start_response):
        # Pre-process request
        print(f"Request: {environ['REQUEST_METHOD']} {environ['PATH_INFO']}")
        
        # Call the next application
        response = self.app(environ, start_response)
        
        # Post-process response
        print("Response sent")
        
        return response

# Apply middleware
from flask import Flask

app = Flask(__name__)
app.wsgi_app = BasicMiddleware(app.wsgi_app)
```

### Before/After Request Hooks

```python
from flask import Flask, request, g
from datetime import datetime
import time

app = Flask(__name__)

@app.before_request
def before_request():
    """Execute before each request."""
    g.start_time = time.time()
    g.request_id = str(uuid.uuid4())
    
    print(f"[{g.request_id}] Request started: {request.method} {request.path}")
    
    # Example: Authentication check
    token = request.headers.get('Authorization')
    if not token and request.endpoint not in ['login', 'health']:
        return {'error': 'Authentication required'}, 401

@app.after_request
def after_request(response):
    """Execute after each request."""
    duration = time.time() - g.start_time
    
    print(f"[{g.request_id}] Request completed in {duration:.3f}s")
    
    # Add custom headers
    response.headers['X-Request-ID'] = g.request_id
    response.headers['X-Response-Time'] = f"{duration:.3f}s"
    
    return response

@app.teardown_request
def teardown_request(exception=None):
    """Execute after response is returned (even on error)."""
    if exception:
        print(f"[{g.request_id}] Error occurred: {exception}")
    
    # Clean up resources
    # Close database connections, etc.

@app.teardown_appcontext
def teardown_appcontext(exception=None):
    """Execute when application context is torn down."""
    # Clean up application-level resources
    pass
```

### Flask Middleware Class

```python
from flask import Flask, request, jsonify
import time
import logging

class RequestLoggingMiddleware:
    """Middleware to log all requests and responses."""
    
    def __init__(self, app):
        self.app = app
        self.logger = logging.getLogger(__name__)
    
    def __call__(self, environ, start_response):
        # Extract request information
        method = environ.get('REQUEST_METHOD')
        path = environ.get('PATH_INFO')
        query = environ.get('QUERY_STRING')
        
        # Log request
        self.logger.info(f"Request: {method} {path}?{query}")
        
        # Record start time
        start_time = time.time()
        
        # Define custom start_response to capture status
        status_code = None
        
        def custom_start_response(status, headers, exc_info=None):
            nonlocal status_code
            status_code = status.split()[0]
            return start_response(status, headers, exc_info)
        
        # Call the application
        response = self.app(environ, custom_start_response)
        
        # Log response
        duration = time.time() - start_time
        self.logger.info(
            f"Response: {status_code} ({duration:.3f}s) - {method} {path}"
        )
        
        return response

# Apply middleware
app = Flask(__name__)
app.wsgi_app = RequestLoggingMiddleware(app.wsgi_app)
```

### Multiple Middleware Example

```python
from flask import Flask, request, g
import time
import uuid

class TimingMiddleware:
    """Record request timing."""
    
    def __init__(self, app):
        self.app = app
    
    def __call__(self, environ, start_response):
        environ['start_time'] = time.time()
        
        def timing_start_response(status, headers, exc_info=None):
            duration = time.time() - environ['start_time']
            headers.append(('X-Response-Time', f"{duration:.3f}s"))
            return start_response(status, headers, exc_info)
        
        return self.app(environ, timing_start_response)

class RequestIDMiddleware:
    """Add unique ID to each request."""
    
    def __init__(self, app):
        self.app = app
    
    def __call__(self, environ, start_response):
        request_id = str(uuid.uuid4())
        environ['request_id'] = request_id
        
        def id_start_response(status, headers, exc_info=None):
            headers.append(('X-Request-ID', request_id))
            return start_response(status, headers, exc_info)
        
        return self.app(environ, id_start_response)

class CORSMiddleware:
    """Add CORS headers."""
    
    def __init__(self, app, allow_origin='*'):
        self.app = app
        self.allow_origin = allow_origin
    
    def __call__(self, environ, start_response):
        def cors_start_response(status, headers, exc_info=None):
            headers.extend([
                ('Access-Control-Allow-Origin', self.allow_origin),
                ('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS'),
                ('Access-Control-Allow-Headers', 'Content-Type, Authorization'),
            ])
            return start_response(status, headers, exc_info)
        
        return self.app(environ, cors_start_response)

# Apply multiple middleware (order matters!)
app = Flask(__name__)
app.wsgi_app = TimingMiddleware(app.wsgi_app)
app.wsgi_app = RequestIDMiddleware(app.wsgi_app)
app.wsgi_app = CORSMiddleware(app.wsgi_app)
```

---

## Middleware in FastAPI

FastAPI uses ASGI (Asynchronous Server Gateway Interface) and provides both built-in and custom middleware support.

### Basic Middleware Structure

```python
from fastapi import FastAPI, Request
from fastapi.middleware.base import BaseHTTPMiddleware
import time

app = FastAPI()

class TimingMiddleware(BaseHTTPMiddleware):
    """Middleware to track request processing time."""
    
    async def dispatch(self, request: Request, call_next):
        # Pre-process request
        start_time = time.time()
        
        # Process request and get response
        response = await call_next(request)
        
        # Post-process response
        duration = time.time() - start_time
        response.headers["X-Response-Time"] = f"{duration:.3f}s"
        
        return response

# Add middleware
app.add_middleware(TimingMiddleware)
```

### Multiple Middleware

```python
from fastapi import FastAPI, Request, Response
from fastapi.middleware.base import BaseHTTPMiddleware
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.gzip import GZipMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware
import uuid
import logging
import time

app = FastAPI()

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class RequestIDMiddleware(BaseHTTPMiddleware):
    """Add unique request ID."""
    
    async def dispatch(self, request: Request, call_next):
        request_id = str(uuid.uuid4())
        request.state.request_id = request_id
        
        response = await call_next(request)
        response.headers["X-Request-ID"] = request_id
        
        return response

class LoggingMiddleware(BaseHTTPMiddleware):
    """Log all requests and responses."""
    
    async def dispatch(self, request: Request, call_next):
        # Log request
        logger.info(
            f"Request: {request.method} {request.url.path} "
            f"[{request.state.request_id}]"
        )
        
        start_time = time.time()
        
        # Process request
        response = await call_next(request)
        
        # Log response
        duration = time.time() - start_time
        logger.info(
            f"Response: {response.status_code} ({duration:.3f}s) "
            f"[{request.state.request_id}]"
        )
        
        return response

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    """Add security headers."""
    
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
        
        return response

class ErrorHandlingMiddleware(BaseHTTPMiddleware):
    """Catch and format all errors."""
    
    async def dispatch(self, request: Request, call_next):
        try:
            response = await call_next(request)
            return response
        except Exception as e:
            logger.error(
                f"Error processing request [{request.state.request_id}]: {str(e)}",
                exc_info=True
            )
            
            return Response(
                content=json.dumps({
                    "error": {
                        "code": "INTERNAL_ERROR",
                        "message": "An unexpected error occurred",
                        "request_id": request.state.request_id
                    }
                }),
                status_code=500,
                media_type="application/json"
            )

# Add middleware (order matters - first added executes first)
app.add_middleware(RequestIDMiddleware)
app.add_middleware(LoggingMiddleware)
app.add_middleware(SecurityHeadersMiddleware)
app.add_middleware(ErrorHandlingMiddleware)

# Built-in middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.add_middleware(GZipMiddleware, minimum_size=1000)

app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["example.com", "*.example.com"]
)
```

### Dependency Injection for Request Processing

```python
from fastapi import FastAPI, Depends, Request, HTTPException, status
from typing import Optional

app = FastAPI()

# Dependency for extracting and validating token
async def get_current_user(request: Request):
    """Extract and validate user from token."""
    token = request.headers.get("Authorization")
    
    if not token:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Authentication required"
        )
    
    # Remove 'Bearer ' prefix
    if token.startswith("Bearer "):
        token = token[7:]
    
    # Validate token (simplified)
    # In real app, verify JWT, check database, etc.
    user = validate_token(token)
    
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid token"
        )
    
    return user

# Use dependency in routes
@app.get("/protected")
async def protected_route(current_user: dict = Depends(get_current_user)):
    return {"message": f"Hello {current_user['name']}"}

# Optional authentication
async def get_current_user_optional(request: Request) -> Optional[dict]:
    """Get current user if authenticated, None otherwise."""
    try:
        return await get_current_user(request)
    except HTTPException:
        return None

@app.get("/optional-auth")
async def optional_auth_route(current_user: Optional[dict] = Depends(get_current_user_optional)):
    if current_user:
        return {"message": f"Hello {current_user['name']}"}
    return {"message": "Hello guest"}
```

---

## Middleware in Django

Django uses middleware as a framework of hooks into request/response processing.

### Django Middleware Structure

```python
# middleware.py

class SimpleMiddleware:
    """Basic Django middleware structure."""
    
    def __init__(self, get_response):
        self.get_response = get_response
        # One-time configuration and initialization
    
    def __call__(self, request):
        # Code executed for each request before view is called
        print(f"Request: {request.method} {request.path}")
        
        # Call the next middleware or view
        response = self.get_response(request)
        
        # Code executed for each response after view is called
        print(f"Response: {response.status_code}")
        
        return response

# Add to settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
    'myapp.middleware.SimpleMiddleware',  # Custom middleware
]
```

### Complete Django Middleware Example

```python
# middleware.py
from django.utils.deprecation import MiddlewareMixin
from django.http import JsonResponse
import time
import uuid
import logging

logger = logging.getLogger(__name__)

class RequestIDMiddleware:
    """Add unique ID to each request."""
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        # Generate unique request ID
        request.request_id = str(uuid.uuid4())
        
        # Process request
        response = self.get_response(request)
        
        # Add to response headers
        response['X-Request-ID'] = request.request_id
        
        return response

class TimingMiddleware:
    """Track request processing time."""
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        # Start timer
        request.start_time = time.time()
        
        # Process request
        response = self.get_response(request)
        
        # Calculate duration
        duration = time.time() - request.start_time
        response['X-Response-Time'] = f"{duration:.3f}s"
        
        return response

class LoggingMiddleware:
    """Log all requests and responses."""
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        # Log request
        logger.info(
            f"Request: {request.method} {request.path} "
            f"[{getattr(request, 'request_id', 'N/A')}]"
        )
        
        # Process request
        response = self.get_response(request)
        
        # Log response
        duration = time.time() - getattr(request, 'start_time', time.time())
        logger.info(
            f"Response: {response.status_code} ({duration:.3f}s) "
            f"[{getattr(request, 'request_id', 'N/A')}]"
        )
        
        return response

class AuthenticationMiddleware:
    """Custom authentication middleware."""
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        # Extract token from header
        token = request.headers.get('Authorization')
        
        if token and token.startswith('Bearer '):
            token = token[7:]
            # Validate token and attach user to request
            user = self.validate_token(token)
            if user:
                request.custom_user = user
        
        return self.get_response(request)
    
    def validate_token(self, token):
        # Token validation logic
        # Return user object or None
        pass

class ErrorHandlingMiddleware:
    """Catch and format errors."""
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        try:
            response = self.get_response(request)
            return response
        except Exception as e:
            logger.error(
                f"Error processing request: {str(e)}",
                exc_info=True
            )
            
            return JsonResponse({
                'error': {
                    'code': 'INTERNAL_ERROR',
                    'message': 'An unexpected error occurred',
                    'request_id': getattr(request, 'request_id', None)
                }
            }, status=500)
    
    def process_exception(self, request, exception):
        """Called when a view raises an exception."""
        logger.error(
            f"Exception in view: {str(exception)}",
            exc_info=True
        )
        
        return JsonResponse({
            'error': {
                'code': 'INTERNAL_ERROR',
                'message': str(exception)
            }
        }, status=500)

# Using MiddlewareMixin for process_* methods
class CustomMiddleware(MiddlewareMixin):
    """Middleware with all hook methods."""
    
    def process_request(self, request):
        """Called on each request, before Django decides which view to execute."""
        print("process_request")
        # Return None to continue processing
        # Return HttpResponse to short-circuit
        return None
    
    def process_view(self, request, view_func, view_args, view_kwargs):
        """Called just before Django calls the view."""
        print(f"process_view: {view_func.__name__}")
        return None
    
    def process_template_response(self, request, response):
        """Called just after the view has finished executing."""
        print("process_template_response")
        return response
    
    def process_response(self, request, response):
        """Called on all responses before they're returned."""
        print("process_response")
        return response
    
    def process_exception(self, request, exception):
        """Called when a view raises an exception."""
        print(f"process_exception: {str(exception)}")
        return None
```

---

## Common Middleware Patterns

### 1. Chain of Responsibility Pattern

```python
from typing import Callable, Optional
from dataclasses import dataclass

@dataclass
class Request:
    method: str
    path: str
    headers: dict
    body: dict

@dataclass
class Response:
    status_code: int
    headers: dict
    body: dict

class Middleware:
    """Base middleware class."""
    
    def __init__(self, next_middleware: Optional['Middleware'] = None):
        self.next = next_middleware
    
    def handle(self, request: Request) -> Response:
        """Process request and pass to next middleware."""
        # Pre-processing
        self.before_request(request)
        
        # Call next middleware or handler
        if self.next:
            response = self.next.handle(request)
        else:
            response = self.handle_request(request)
        
        # Post-processing
        self.after_response(response)
        
        return response
    
    def before_request(self, request: Request):
        """Hook called before request is processed."""
        pass
    
    def after_response(self, response: Response):
        """Hook called after response is generated."""
        pass
    
    def handle_request(self, request: Request) -> Response:
        """Final handler if no next middleware."""
        return Response(status_code=404, headers={}, body={})

class LoggingMiddleware(Middleware):
    """Log requests and responses."""
    
    def before_request(self, request: Request):
        print(f"→ {request.method} {request.path}")
    
    def after_response(self, response: Response):
        print(f"← {response.status_code}")

class AuthMiddleware(Middleware):
    """Authenticate requests."""
    
    def before_request(self, request: Request):
        token = request.headers.get('Authorization')
        if not token:
            raise Exception("Authentication required")

class RateLimitMiddleware(Middleware):
    """Rate limit requests."""
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.requests = {}
    
    def before_request(self, request: Request):
        # Simple rate limiting logic
        client_ip = request.headers.get('X-Forwarded-For', 'unknown')
        # Check and update rate limit
        pass

# Build middleware chain
handler = Middleware()
rate_limiter = RateLimitMiddleware(next_middleware=handler)
auth = AuthMiddleware(next_middleware=rate_limiter)
logger = LoggingMiddleware(next_middleware=auth)

# Process request through chain
request = Request(
    method="GET",
    path="/api/users",
    headers={"Authorization": "Bearer token123"},
    body={}
)

response = logger.handle(request)
```

### 2. Decorator Pattern for Middleware

```python
from functools import wraps
from typing import Callable
import time

def timing_middleware(func: Callable) -> Callable:
    """Decorator to measure execution time."""
    
    @wraps(func)
    async def wrapper(*args, **kwargs):
        start = time.time()
        result = await func(*args, **kwargs)
        duration = time.time() - start
        print(f"Execution time: {duration:.3f}s")
        return result
    
    return wrapper

def logging_middleware(func: Callable) -> Callable:
    """Decorator to log function calls."""
    
    @wraps(func)
    async def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        result = await func(*args, **kwargs)
        print(f"Finished {func.__name__}")
        return result
    
    return wrapper

def auth_required(func: Callable) -> Callable:
    """Decorator to require authentication."""
    
    @wraps(func)
    async def wrapper(request, *args, **kwargs):
        if not hasattr(request, 'user') or not request.user:
            raise Exception("Authentication required")
        return await func(request, *args, **kwargs)
    
    return wrapper

# Apply multiple middleware decorators
@timing_middleware
@logging_middleware
@auth_required
async def protected_endpoint(request):
    return {"message": "Success"}
```

---

## Authentication Middleware

### JWT Authentication Middleware

```python
from fastapi import FastAPI, Request, HTTPException, status
from fastapi.middleware.base import BaseHTTPMiddleware
import jwt
from datetime import datetime, timedelta
from typing import Optional

app = FastAPI()

SECRET_KEY = "your-secret-key"
ALGORITHM = "HS256"

class JWTAuthMiddleware(BaseHTTPMiddleware):
    """JWT authentication middleware."""
    
    def __init__(self, app, secret_key: str, algorithm: str = "HS256"):
        super().__init__(app)
        self.secret_key = secret_key
        self.algorithm = algorithm
        
        # Endpoints that don't require authentication
        self.public_endpoints = {'/login', '/register', '/health', '/docs', '/openapi.json'}
    
    async def dispatch(self, request: Request, call_next):
        # Skip authentication for public endpoints
        if request.url.path in self.public_endpoints:
            return await call_next(request)
        
        # Extract token from header
        auth_header = request.headers.get('Authorization')
        
        if not auth_header:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Missing authorization header"
            )
        
        try:
            # Parse Bearer token
            scheme, token = auth_header.split()
            if scheme.lower() != 'bearer':
                raise HTTPException(
                    status_code=status.HTTP_401_UNAUTHORIZED,
                    detail="Invalid authentication scheme"
                )
            
            # Decode and verify token
            payload = jwt.decode(
                token,
                self.secret_key,
                algorithms=[self.algorithm]
            )
            
            # Attach user info to request state
            request.state.user_id = payload.get('user_id')
            request.state.username = payload.get('username')
            request.state.email = payload.get('email')
            
        except jwt.ExpiredSignatureError:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Token has expired"
            )
        except jwt.InvalidTokenError:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid token"
            )
        except ValueError:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid authorization header format"
            )
        
        # Process request
        response = await call_next(request)
        return response

# Add middleware
app.add_middleware(JWTAuthMiddleware, secret_key=SECRET_KEY)

# Helper function to create tokens
def create_access_token(data: dict, expires_delta: Optional[timedelta] = None):
    """Create JWT access token."""
    to_encode = data.copy()
    
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(hours=24)
    
    to_encode.update({"exp": expire})
    
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

@app.post("/login")
async def login(username: str, password: str):
    """Login endpoint (public)."""
    # Validate credentials (simplified)
    if username == "admin" and password == "password":
        token = create_access_token(
            data={
                "user_id": 1,
                "username": username,
                "email": "admin@example.com"
            }
        )
        return {"access_token": token, "token_type": "bearer"}
    
    raise HTTPException(status_code=401, detail="Invalid credentials")

@app.get("/protected")
async def protected_route(request: Request):
    """Protected endpoint."""
    return {
        "message": f"Hello {request.state.username}",
        "user_id": request.state.user_id
    }
```

### API Key Authentication Middleware

```python
from fastapi import FastAPI, Request, HTTPException, status
from fastapi.middleware.base import BaseHTTPMiddleware
import secrets

app = FastAPI()

# Mock API key storage (use database in production)
VALID_API_KEYS = {
    "sk_test_123": {"user_id": 1, "name": "Test User", "tier": "free"},
    "sk_prod_456": {"user_id": 2, "name": "Premium User", "tier": "premium"}
}

class APIKeyAuthMiddleware(BaseHTTPMiddleware):
    """API Key authentication middleware."""
    
    def __init__(self, app):
        super().__init__(app)
        self.public_endpoints = {'/health', '/docs', '/openapi.json'}
    
    async def dispatch(self, request: Request, call_next):
        # Skip authentication for public endpoints
        if request.url.path in self.public_endpoints:
            return await call_next(request)
        
        # Extract API key from header or query parameter
        api_key = (
            request.headers.get('X-API-Key') or
            request.query_params.get('api_key')
        )
        
        if not api_key:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="API key required"
            )
        
        # Validate API key
        user_info = VALID_API_KEYS.get(api_key)
        
        if not user_info:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid API key"
            )
        
        # Attach user info to request
        request.state.user_id = user_info['user_id']
        request.state.user_name = user_info['name']
        request.state.tier = user_info['tier']
        
        # Process request
        response = await call_next(request)
        return response

app.add_middleware(APIKeyAuthMiddleware)

@app.get("/api/data")
async def get_data(request: Request):
    """Protected endpoint."""
    return {
        "data": "sensitive information",
        "user": request.state.user_name,
        "tier": request.state.tier
    }
```

---

## Logging Middleware

### Comprehensive Logging Middleware

```python
from fastapi import FastAPI, Request
from fastapi.middleware.base import BaseHTTPMiddleware
import logging
import time
import json
from typing import Callable

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

app = FastAPI()

class RequestLoggingMiddleware(BaseHTTPMiddleware):
    """Comprehensive request/response logging middleware."""
    
    async def dispatch(self, request: Request, call_next: Callable):
        # Generate request ID
        request_id = request.headers.get('X-Request-ID', str(uuid.uuid4()))
        
        # Start timer
        start_time = time.time()
        
        # Extract request details
        request_body = await self.get_request_body(request)
        
        # Log request
        logger.info(
            f"Request started",
            extra={
                'request_id': request_id,
                'method': request.method,
                'url': str(request.url),
                'path': request.url.path,
                'query_params': dict(request.query_params),
                'headers': dict(request.headers),
                'client_host': request.client.host if request.client else None,
                'body': request_body
            }
        )
        
        # Process request
        try:
            response = await call_next(request)
            
            # Calculate duration
            duration = time.time() - start_time
            
            # Log response
            logger.info(
                f"Request completed",
                extra={
                    'request_id': request_id,
                    'status_code': response.status_code,
                    'duration': f"{duration:.3f}s"
                }
            )
            
            # Add request ID to response
            response.headers['X-Request-ID'] = request_id
            response.headers['X-Response-Time'] = f"{duration:.3f}s"
            
            return response
            
        except Exception as e:
            # Log error
            duration = time.time() - start_time
            logger.error(
                f"Request failed",
                extra={
                    'request_id': request_id,
                    'error': str(e),
                    'duration': f"{duration:.3f}s"
                },
                exc_info=True
            )
            raise
    
    async def get_request_body(self, request: Request) -> dict:
        """Extract request body safely."""
        try:
            # Read body
            body = await request.body()
            
            # Try to parse as JSON
            if body:
                return json.loads(body)
            return {}
        except:
            return {"raw": body.decode() if body else ""}

app.add_middleware(RequestLoggingMiddleware)
```

### Structured Logging Middleware

```python
import structlog
from fastapi import FastAPI, Request
from fastapi.middleware.base import BaseHTTPMiddleware

# Configure structured logging
structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.stdlib.PositionalArgumentsFormatter(),
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.UnicodeDecoder(),
        structlog.processors.JSONRenderer()
    ],
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
    cache_logger_on_first_use=True,
)

logger = structlog.get_logger()

app = FastAPI()

class StructuredLoggingMiddleware(BaseHTTPMiddleware):
    """Structured logging middleware."""
    
    async def dispatch(self, request: Request, call_next):
        # Bind request context
        log = logger.bind(
            method=request.method,
            path=request.url.path,
            client=request.client.host if request.client else None
        )
        
        start_time = time.time()
        
        try:
            response = await call_next(request)
            
            duration = time.time() - start_time
            
            # Log successful response
            log.info(
                "request_completed",
                status_code=response.status_code,
                duration=duration
            )
            
            return response
            
        except Exception as e:
            duration = time.time() - start_time
            
            # Log error
            log.error(
                "request_failed",
                error=str(e),
                duration=duration
            )
            raise

app.add_middleware(StructuredLoggingMiddleware)
```

---

## Rate Limiting Middleware

### Simple Rate Limiting

```python
from fastapi import FastAPI, Request, HTTPException, status
from fastapi.middleware.base import BaseHTTPMiddleware
from collections import defaultdict
from datetime import datetime, timedelta
import time

app = FastAPI()

class RateLimitMiddleware(BaseHTTPMiddleware):
    """Simple rate limiting middleware."""
    
    def __init__(
        self,
        app,
        requests_per_minute: int = 60,
        requests_per_hour: int = 1000
    ):
        super().__init__(app)
        self.requests_per_minute = requests_per_minute
        self.requests_per_hour = requests_per_hour
        
        # Store request timestamps per client
        self.minute_requests = defaultdict(list)
        self.hour_requests = defaultdict(list)
    
    async def dispatch(self, request: Request, call_next):
        # Get client identifier
        client_id = self.get_client_id(request)
        
        # Current time
        now = datetime.utcnow()
        minute_ago = now - timedelta(minutes=1)
        hour_ago = now - timedelta(hours=1)
        
        # Clean old requests
        self.minute_requests[client_id] = [
            req_time for req_time in self.minute_requests[client_id]
            if req_time > minute_ago
        ]
        self.hour_requests[client_id] = [
            req_time for req_time in self.hour_requests[client_id]
            if req_time > hour_ago
        ]
        
        # Check limits
        minute_count = len(self.minute_requests[client_id])
        hour_count = len(self.hour_requests[client_id])
        
        if minute_count >= self.requests_per_minute:
            raise HTTPException(
                status_code=status.HTTP_429_TOO_MANY_REQUESTS,
                detail={
                    "error": "Rate limit exceeded",
                    "limit": self.requests_per_minute,
                    "window": "1 minute",
                    "retry_after": 60
                },
                headers={"Retry-After": "60"}
            )
        
        if hour_count >= self.requests_per_hour:
            raise HTTPException(
                status_code=status.HTTP_429_TOO_MANY_REQUESTS,
                detail={
                    "error": "Rate limit exceeded",
                    "limit": self.requests_per_hour,
                    "window": "1 hour",
                    "retry_after": 3600
                },
                headers={"Retry-After": "3600"}
            )
        
        # Record request
        self.minute_requests[client_id].append(now)
        self.hour_requests[client_id].append(now)
        
        # Process request
        response = await call_next(request)
        
        # Add rate limit headers
        response.headers["X-RateLimit-Limit-Minute"] = str(self.requests_per_minute)
        response.headers["X-RateLimit-Remaining-Minute"] = str(
            self.requests_per_minute - minute_count - 1
        )
        response.headers["X-RateLimit-Limit-Hour"] = str(self.requests_per_hour)
        response.headers["X-RateLimit-Remaining-Hour"] = str(
            self.requests_per_hour - hour_count - 1
        )
        
        return response
    
    def get_client_id(self, request: Request) -> str:
        """Get unique client identifier."""
        # Try to get from API key
        api_key = request.headers.get('X-API-Key')
        if api_key:
            return f"api_key:{api_key}"
        
        # Fall back to IP address
        forwarded_for = request.headers.get('X-Forwarded-For')
        if forwarded_for:
            client_ip = forwarded_for.split(',')[0].strip()
        else:
            client_ip = request.client.host if request.client else 'unknown'
        
        return f"ip:{client_ip}"

app.add_middleware(
    RateLimitMiddleware,
    requests_per_minute=60,
    requests_per_hour=1000
)
```

### Redis-Based Rate Limiting

```python
from fastapi import FastAPI, Request, HTTPException
from fastapi.middleware.base import BaseHTTPMiddleware
import redis
import time

app = FastAPI()

class RedisRateLimitMiddleware(BaseHTTPMiddleware):
    """Redis-based rate limiting for distributed systems."""
    
    def __init__(
        self,
        app,
        redis_url: str = "redis://localhost:6379",
        requests_per_minute: int = 60
    ):
        super().__init__(app)
        self.redis_client = redis.from_url(redis_url, decode_responses=True)
        self.requests_per_minute = requests_per_minute
    
    async def dispatch(self, request: Request, call_next):
        # Get client ID
        client_id = self.get_client_id(request)
        
        # Redis key for rate limiting
        key = f"rate_limit:{client_id}:{int(time.time() // 60)}"
        
        # Increment counter
        current = self.redis_client.incr(key)
        
        # Set expiry on first request
        if current == 1:
            self.redis_client.expire(key, 60)
        
        # Check limit
        if current > self.requests_per_minute:
            ttl = self.redis_client.ttl(key)
            
            raise HTTPException(
                status_code=429,
                detail={
                    "error": "Rate limit exceeded",
                    "retry_after": ttl
                },
                headers={"Retry-After": str(ttl)}
            )
        
        # Process request
        response = await call_next(request)
        
        # Add rate limit headers
        response.headers["X-RateLimit-Limit"] = str(self.requests_per_minute)
        response.headers["X-RateLimit-Remaining"] = str(
            self.requests_per_minute - current
        )
        
        return response
    
    def get_client_id(self, request: Request) -> str:
        """Get client identifier."""
        return request.client.host if request.client else 'unknown'

# app.add_middleware(RedisRateLimitMiddleware)
```

---

## CORS Middleware

### Custom CORS Implementation

```python
from fastapi import FastAPI, Request
from fastapi.middleware.base import BaseHTTPMiddleware
from fastapi.responses import Response
from typing import List

app = FastAPI()

class CORSMiddleware(BaseHTTPMiddleware):
    """Custom CORS middleware."""
    
    def __init__(
        self,
        app,
        allow_origins: List[str] = ["*"],
        allow_methods: List[str] = ["*"],
        allow_headers: List[str] = ["*"],
        allow_credentials: bool = False,
        max_age: int = 600
    ):
        super().__init__(app)
        self.allow_origins = allow_origins
        self.allow_methods = allow_methods
        self.allow_headers = allow_headers
        self.allow_credentials = allow_credentials
        self.max_age = max_age
    
    async def dispatch(self, request: Request, call_next):
        # Get origin from request
        origin = request.headers.get('origin')
        
        # Handle preflight requests
        if request.method == 'OPTIONS':
            response = Response()
            response.status_code = 200
        else:
            response = await call_next(request)
        
        # Add CORS headers
        if origin and self.is_allowed_origin(origin):
            response.headers['Access-Control-Allow-Origin'] = origin
        elif '*' in self.allow_origins:
            response.headers['Access-Control-Allow-Origin'] = '*'
        
        if self.allow_credentials:
            response.headers['Access-Control-Allow-Credentials'] = 'true'
        
        if request.method == 'OPTIONS':
            # Preflight response
            response.headers['Access-Control-Allow-Methods'] = ', '.join(
                self.allow_methods if '*' not in self.allow_methods
                else ['GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS']
            )
            response.headers['Access-Control-Allow-Headers'] = ', '.join(
                self.allow_headers if '*' not in self.allow_headers
                else ['*']
            )
            response.headers['Access-Control-Max-Age'] = str(self.max_age)
        
        return response
    
    def is_allowed_origin(self, origin: str) -> bool:
        """Check if origin is allowed."""
        if '*' in self.allow_origins:
            return True
        return origin in self.allow_origins

app.add_middleware(
    CORSMiddleware,
    allow_origins=['https://example.com', 'https://app.example.com'],
    allow_methods=['GET', 'POST', 'PUT', 'DELETE'],
    allow_headers=['Content-Type', 'Authorization'],
    allow_credentials=True
)
```

---

## Error Handling Middleware

### Comprehensive Error Handler

```python
from fastapi import FastAPI, Request, status
from fastapi.middleware.base import BaseHTTPMiddleware
from fastapi.responses import JSONResponse
import logging
import traceback

logger = logging.getLogger(__name__)

app = FastAPI()

class ErrorHandlingMiddleware(BaseHTTPMiddleware):
    """Catch and format all errors."""
    
    async def dispatch(self, request: Request, call_next):
        try:
            response = await call_next(request)
            return response
            
        except HTTPException as e:
            # Handle FastAPI HTTP exceptions
            return JSONResponse(
                status_code=e.status_code,
                content={
                    "error": {
                        "code": self.get_error_code(e.status_code),
                        "message": e.detail,
                        "status_code": e.status_code
                    }
                }
            )
            
        except ValidationError as e:
            # Handle validation errors
            logger.warning(f"Validation error: {str(e)}")
            return JSONResponse(
                status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
                content={
                    "error": {
                        "code": "VALIDATION_ERROR",
                        "message": "Request validation failed",
                        "details": e.errors()
                    }
                }
            )
            
        except Exception as e:
            # Handle unexpected errors
            logger.error(
                f"Unexpected error: {str(e)}",
                exc_info=True,
                extra={
                    "path": request.url.path,
                    "method": request.method
                }
            )
            
            # In production, don't expose internal error details
            if app.debug:
                error_detail = {
                    "type": type(e).__name__,
                    "message": str(e),
                    "traceback": traceback.format_exc()
                }
            else:
                error_detail = "An unexpected error occurred"
            
            return JSONResponse(
                status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
                content={
                    "error": {
                        "code": "INTERNAL_ERROR",
                        "message": error_detail
                    }
                }
            )
    
    def get_error_code(self, status_code: int) -> str:
        """Map status code to error code."""
        error_codes = {
            400: "BAD_REQUEST",
            401: "UNAUTHORIZED",
            403: "FORBIDDEN",
            404: "NOT_FOUND",
            409: "CONFLICT",
            422: "VALIDATION_ERROR",
            429: "RATE_LIMIT_EXCEEDED",
            500: "INTERNAL_ERROR",
            503: "SERVICE_UNAVAILABLE"
        }
        return error_codes.get(status_code, "UNKNOWN_ERROR")

app.add_middleware(ErrorHandlingMiddleware)
```

---

## Request/Response Modification

### Request Transformation Middleware

```python
from fastapi import FastAPI, Request
from fastapi.middleware.base import BaseHTTPMiddleware
import json

app = FastAPI()

class RequestTransformMiddleware(BaseHTTPMiddleware):
    """Transform request data."""
    
    async def dispatch(self, request: Request, call_next):
        # Read original body
        body = await request.body()
        
        if body and request.headers.get('content-type') == 'application/json':
            try:
                # Parse JSON
                data = json.loads(body)
                
                # Transform data (example: convert all keys to lowercase)
                transformed_data = self.transform_keys(data)
                
                # Create new request with transformed body
                request._body = json.dumps(transformed_data).encode()
                
            except json.JSONDecodeError:
                pass  # Not valid JSON, skip transformation
        
        response = await call_next(request)
        return response
    
    def transform_keys(self, obj):
        """Recursively transform dictionary keys to lowercase."""
        if isinstance(obj, dict):
            return {k.lower(): self.transform_keys(v) for k, v in obj.items()}
        elif isinstance(obj, list):
            return [self.transform_keys(item) for item in obj]
        else:
            return obj

app.add_middleware(RequestTransformMiddleware)
```

### Response Compression Middleware

```python
from fastapi import FastAPI, Request
from fastapi.middleware.base import BaseHTTPMiddleware
from starlette.responses import StreamingResponse
import gzip
import io

app = FastAPI()

class CompressionMiddleware(BaseHTTPMiddleware):
    """Compress responses with gzip."""
    
    def __init__(self, app, minimum_size: int = 1000):
        super().__init__(app)
        self.minimum_size = minimum_size
    
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        
        # Check if client accepts gzip
        accept_encoding = request.headers.get('accept-encoding', '')
        if 'gzip' not in accept_encoding.lower():
            return response
        
        # Check if response is large enough to compress
        if (
            hasattr(response, 'body') and
            len(response.body) < self.minimum_size
        ):
            return response
        
        # Compress response body
        if hasattr(response, 'body'):
            compressed_body = gzip.compress(response.body)
            
            response.body = compressed_body
            response.headers['Content-Encoding'] = 'gzip'
            response.headers['Content-Length'] = str(len(compressed_body))
        
        return response

app.add_middleware(CompressionMiddleware, minimum_size=1000)
```

---

## Performance Monitoring Middleware

### Performance Tracking Middleware

```python
from fastapi import FastAPI, Request
from fastapi.middleware.base import BaseHTTPMiddleware
import time
import psutil
import logging

logger = logging.getLogger(__name__)

app = FastAPI()

class PerformanceMonitoringMiddleware(BaseHTTPMiddleware):
    """Monitor request performance metrics."""
    
    async def dispatch(self, request: Request, call_next):
        # Record start metrics
        start_time = time.time()
        start_cpu = psutil.cpu_percent()
        process = psutil.Process()
        start_memory = process.memory_info().rss / 1024 / 1024  # MB
        
        # Process request
        response = await call_next(request)
        
        # Calculate metrics
        duration = time.time() - start_time
        end_cpu = psutil.cpu_percent()
        end_memory = process.memory_info().rss / 1024 / 1024  # MB
        
        # Log performance metrics
        logger.info(
            f"Performance metrics",
            extra={
                'path': request.url.path,
                'method': request.method,
                'duration': f"{duration:.3f}s",
                'cpu_usage': f"{end_cpu:.1f}%",
                'memory_delta': f"{end_memory - start_memory:.2f}MB",
                'status_code': response.status_code
            }
        )
        
        # Add headers
        response.headers['X-Response-Time'] = f"{duration:.3f}s"
        response.headers['X-CPU-Usage'] = f"{end_cpu:.1f}%"
        
        # Alert on slow requests
        if duration > 1.0:
            logger.warning(
                f"Slow request detected: {request.method} {request.url.path} "
                f"took {duration:.3f}s"
            )
        
        return response

app.add_middleware(PerformanceMonitoringMiddleware)
```

---

## Security Middleware

### Security Headers Middleware

```python
from fastapi import FastAPI, Request
from fastapi.middleware.base import BaseHTTPMiddleware

app = FastAPI()

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    """Add security headers to all responses."""
    
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        
        # Prevent MIME type sniffing
        response.headers['X-Content-Type-Options'] = 'nosniff'
        
        # Prevent clickjacking
        response.headers['X-Frame-Options'] = 'DENY'
        
        # XSS protection
        response.headers['X-XSS-Protection'] = '1; mode=block'
        
        # HSTS (force HTTPS)
        response.headers['Strict-Transport-Security'] = (
            'max-age=31536000; includeSubDomains; preload'
        )
        
        # Content Security Policy
        response.headers['Content-Security-Policy'] = (
            "default-src 'self'; "
            "script-src 'self' 'unsafe-inline' 'unsafe-eval'; "
            "style-src 'self' 'unsafe-inline'; "
            "img-src 'self' data: https:; "
            "font-src 'self' data:; "
            "connect-src 'self'"
        )
        
        # Referrer policy
        response.headers['Referrer-Policy'] = 'strict-origin-when-cross-origin'
        
        # Permissions policy
        response.headers['Permissions-Policy'] = (
            'geolocation=(), microphone=(), camera=()'
        )
        
        return response

app.add_middleware(SecurityHeadersMiddleware)
```

### Input Sanitization Middleware

```python
from fastapi import FastAPI, Request
from fastapi.middleware.base import BaseHTTPMiddleware
import html
import json

app = FastAPI()

class InputSanitizationMiddleware(BaseHTTPMiddleware):
    """Sanitize user input to prevent XSS."""
    
    async def dispatch(self, request: Request, call_next):
        # Read request body
        body = await request.body()
        
        if body and request.headers.get('content-type') == 'application/json':
            try:
                data = json.loads(body)
                
                # Sanitize data
                sanitized_data = self.sanitize(data)
                
                # Update request body
                request._body = json.dumps(sanitized_data).encode()
                
            except json.JSONDecodeError:
                pass
        
        response = await call_next(request)
        return response
    
    def sanitize(self, obj):
        """Recursively sanitize strings in data structure."""
        if isinstance(obj, dict):
            return {k: self.sanitize(v) for k, v in obj.items()}
        elif isinstance(obj, list):
            return [self.sanitize(item) for item in obj]
        elif isinstance(obj, str):
            # HTML escape to prevent XSS
            return html.escape(obj)
        else:
            return obj

app.add_middleware(InputSanitizationMiddleware)
```

---

## Custom Middleware Development

### Complete Custom Middleware Template

```python
from fastapi import FastAPI, Request
from fastapi.middleware.base import BaseHTTPMiddleware
from typing import Callable
import logging

logger = logging.getLogger(__name__)

app = FastAPI()

class CustomMiddleware(BaseHTTPMiddleware):
    """Template for custom middleware."""
    
    def __init__(
        self,
        app,
        # Add configuration parameters
        param1: str = "default",
        param2: int = 100
    ):
        """Initialize middleware with configuration."""
        super().__init__(app)
        self.param1 = param1
        self.param2 = param2
        
        # Initialize any resources (database connections, caches, etc.)
        self.initialize_resources()
    
    def initialize_resources(self):
        """Initialize external resources."""
        # Example: Connect to database, cache, etc.
        pass
    
    async def dispatch(
        self,
        request: Request,
        call_next: Callable
    ):
        """
        Main middleware logic.
        
        Args:
            request: Incoming request object
            call_next: Callable to invoke next middleware/handler
            
        Returns:
            Response object
        """
        try:
            # 1. PRE-PROCESSING
            # - Validate request
            # - Authenticate/authorize
            # - Transform request
            # - Log request
            await self.pre_process(request)
            
            # 2. CALL NEXT MIDDLEWARE/HANDLER
            response = await call_next(request)
            
            # 3. POST-PROCESSING
            # - Transform response
            # - Add headers
            # - Log response
            await self.post_process(request, response)
            
            return response
            
        except Exception as e:
            # 4. ERROR HANDLING
            return await self.handle_error(request, e)
    
    async def pre_process(self, request: Request):
        """Pre-process the request."""
        logger.info(f"Processing request: {request.method} {request.url.path}")
        
        # Add custom logic here
        pass
    
    async def post_process(self, request: Request, response):
        """Post-process the response."""
        logger.info(f"Sending response: {response.status_code}")
        
        # Add custom logic here
        pass
    
    async def handle_error(self, request: Request, error: Exception):
        """Handle errors that occur during processing."""
        logger.error(f"Error processing request: {str(error)}", exc_info=True)
        
        # Return error response
        from fastapi.responses import JSONResponse
        return JSONResponse(
            status_code=500,
            content={"error": "Internal server error"}
        )
    
    def cleanup(self):
        """Cleanup resources when middleware is destroyed."""
        # Close connections, release resources, etc.
        pass

# Add middleware with configuration
app.add_middleware(
    CustomMiddleware,
    param1="custom_value",
    param2=200
)
```

---

## Best Practices

### 1. Order Matters

```python
# Correct order (outer to inner):
app.add_middleware(ErrorHandlingMiddleware)     # Catch all errors
app.add_middleware(LoggingMiddleware)           # Log everything
app.add_middleware(CORSMiddleware)              # CORS headers
app.add_middleware(AuthenticationMiddleware)    # Authenticate
app.add_middleware(RateLimitMiddleware)         # Rate limit
app.add_middleware(CompressionMiddleware)       # Compress response

# Request flow: Error → Logging → CORS → Auth → RateLimit → Compression → Handler
# Response flow: Handler → Compression → RateLimit → Auth → CORS → Logging → Error
```

### 2. Keep Middleware Focused

```python
# Good: Single responsibility
class AuthMiddleware(BaseHTTPMiddleware):
    """Only handles authentication."""
    async def dispatch(self, request, call_next):
        # Authentication logic only
        pass

class LoggingMiddleware(BaseHTTPMiddleware):
    """Only handles logging."""
    async def dispatch(self, request, call_next):
        # Logging logic only
        pass

# Bad: Multiple responsibilities
class EverythingMiddleware(BaseHTTPMiddleware):
    """Does too much!"""
    async def dispatch(self, request, call_next):
        # Authentication
        # Logging
        # Rate limiting
        # Compression
        # etc.
        pass
```

### 3. Handle Errors Gracefully

```python
class RobustMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        try:
            # Pre-processing
            self.process_request(request)
            
            # Call next
            response = await call_next(request)
            
            # Post-processing
            self.process_response(response)
            
            return response
            
        except Exception as e:
            # Always handle errors
            logger.error(f"Middleware error: {str(e)}")
            # Don't let middleware errors break the application
            return await call_next(request)
```

### 4. Use Async When Possible

```python
# Good: Async middleware
class AsyncMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        # Can use await for I/O operations
        data = await fetch_from_database()
        response = await call_next(request)
        return response

# Avoid: Blocking operations
class BlockingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        # Bad: Blocks event loop
        time.sleep(1)  # Don't do this!
        
        # Good: Use async sleep
        await asyncio.sleep(1)
```

### 5. Document Middleware Behavior

```python
class WellDocumentedMiddleware(BaseHTTPMiddleware):
    """
    Middleware that does X, Y, and Z.
    
    This middleware:
    - Validates incoming requests
    - Adds custom headers
    - Logs all activity
    
    Configuration:
        param1 (str): Description of param1
        param2 (int): Description of param2
    
    Example:
        app.add_middleware(
            WellDocumentedMiddleware,
            param1="value",
            param2=100
        )
    """
    pass
```

### 6. Test Middleware

```python
import pytest
from fastapi.testclient import TestClient

def test_auth_middleware():
    """Test authentication middleware."""
    client = TestClient(app)
    
    # Test without token
    response = client.get("/protected")
    assert response.status_code == 401
    
    # Test with valid token
    response = client.get(
        "/protected",
        headers={"Authorization": "Bearer valid_token"}
    )
    assert response.status_code == 200

def test_rate_limit_middleware():
    """Test rate limiting middleware."""
    client = TestClient(app)
    
    # Make requests up to limit
    for _ in range(60):
        response = client.get("/api/data")
        assert response.status_code == 200
    
    # Next request should be rate limited
    response = client.get("/api/data")
    assert response.status_code == 429
```

---

## Summary

This comprehensive guide covered:

1. **Introduction**: What middleware is and why it's important
2. **Request/Response Pipeline**: How requests flow through middleware
3. **Framework-Specific Implementations**: Flask, FastAPI, and Django middleware
4. **Common Patterns**: Authentication, logging, rate limiting, CORS, error handling
5. **Request/Response Modification**: Transforming data in the pipeline
6. **Performance Monitoring**: Tracking application performance
7. **Security**: Adding security headers and input sanitization
8. **Custom Development**: Building your own middleware
9. **Best Practices**: Order, focus, error handling, async, documentation, testing

Key Takeaways:
- Middleware enables separation of cross-cutting concerns
- Order of middleware execution matters
- Keep middleware focused on single responsibilities
- Handle errors gracefully to prevent application breakage
- Use async operations for I/O-bound tasks
- Always document and test your middleware
- Consider performance impact of middleware chain

Middleware is a powerful pattern for building maintainable, secure, and performant web applications!