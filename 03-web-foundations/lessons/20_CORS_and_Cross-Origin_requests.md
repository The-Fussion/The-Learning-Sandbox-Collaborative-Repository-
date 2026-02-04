# CORS and Cross-Origin Requests in Python

## Table of Contents
1. [Introduction to CORS](#introduction-to-cors)
2. [How CORS Works](#how-cors-works)
3. [CORS in FastAPI](#cors-in-fastapi)
4. [CORS in Flask](#cors-in-flask)
5. [CORS in Django](#cors-in-django)
6. [Advanced CORS Configuration](#advanced-cors-configuration)
7. [Security Considerations](#security-considerations)
8. [Common Issues and Solutions](#common-issues-and-solutions)
9. [Testing CORS](#testing-cors)
10. [Best Practices](#best-practices)

---

## Introduction to CORS

### What is CORS?

**Cross-Origin Resource Sharing (CORS)** is a security mechanism implemented by web browsers that restricts web pages from making requests to a different domain than the one serving the web page.

### The Same-Origin Policy

Browsers enforce the **Same-Origin Policy (SOP)**, which prevents JavaScript running on one origin from accessing resources from another origin. Two URLs have the same origin if:

- **Protocol** (scheme) is the same (http vs https)
- **Domain** (host) is the same
- **Port** is the same

#### Examples:

```
Origin: https://example.com:443

✅ Same Origin:
- https://example.com:443/page
- https://example.com/api/users

❌ Different Origin:
- http://example.com (different protocol)
- https://api.example.com (different subdomain)
- https://example.com:8080 (different port)
- https://other.com (different domain)
```

### Why CORS Exists

CORS exists to:
1. **Protect users**: Prevent malicious websites from accessing sensitive data
2. **Control access**: Let servers decide which origins can access their resources
3. **Enable legitimate cross-origin requests**: Allow controlled sharing between trusted domains

### Common Scenarios Requiring CORS

```
Scenario 1: Frontend on different domain
Frontend: https://myapp.com
API:      https://api.myapp.com
❌ Blocked without CORS

Scenario 2: Development environment
Frontend: http://localhost:3000 (React/Vue)
API:      http://localhost:8000 (Python)
❌ Blocked without CORS

Scenario 3: CDN-hosted assets
Website:  https://example.com
API:      https://api.example.com
Assets:   https://cdn.example.com
❌ Blocked without CORS
```

---

## How CORS Works

### Simple Requests

A **simple request** meets all these conditions:
- Method: `GET`, `HEAD`, or `POST`
- Headers: Only simple headers (Accept, Accept-Language, Content-Language, Content-Type)
- Content-Type (if used): `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`

#### Simple Request Flow:

```
┌─────────┐                           ┌─────────┐
│ Browser │                           │  Server │
└────┬────┘                           └────┬────┘
     │                                     │
     │  GET /api/users                     │
     │  Origin: https://example.com        │
     │────────────────────────────────────>│
     │                                     │
     │  200 OK                             │
     │  Access-Control-Allow-Origin: *     │
     │<────────────────────────────────────│
     │                                     │
```

### Preflight Requests

A **preflight request** is sent for requests that:
- Use methods other than GET, HEAD, or POST
- Include custom headers
- Use Content-Type other than the simple ones
- Include credentials

#### Preflight Request Flow:

```
┌─────────┐                           ┌─────────┐
│ Browser │                           │  Server │
└────┬────┘                           └────┬────┘
     │                                     │
     │  OPTIONS /api/users (Preflight)     │
     │  Origin: https://example.com        │
     │  Access-Control-Request-Method: PUT │
     │  Access-Control-Request-Headers:    │
     │    Authorization                    │
     │────────────────────────────────────>│
     │                                     │
     │  200 OK                             │
     │  Access-Control-Allow-Origin: *     │
     │  Access-Control-Allow-Methods:      │
     │    GET, POST, PUT, DELETE           │
     │  Access-Control-Allow-Headers:      │
     │    Authorization, Content-Type      │
     │  Access-Control-Max-Age: 86400      │
     │<────────────────────────────────────│
     │                                     │
     │  PUT /api/users/1 (Actual Request)  │
     │  Origin: https://example.com        │
     │  Authorization: Bearer token        │
     │────────────────────────────────────>│
     │                                     │
     │  200 OK                             │
     │  Access-Control-Allow-Origin: *     │
     │<────────────────────────────────────│
     │                                     │
```

### CORS Headers

#### Request Headers:

- **Origin**: The origin making the request
- **Access-Control-Request-Method**: Method for the actual request (preflight)
- **Access-Control-Request-Headers**: Headers for the actual request (preflight)

#### Response Headers:

- **Access-Control-Allow-Origin**: Which origins are allowed
- **Access-Control-Allow-Methods**: Which HTTP methods are allowed
- **Access-Control-Allow-Headers**: Which headers are allowed
- **Access-Control-Allow-Credentials**: Whether credentials are allowed
- **Access-Control-Max-Age**: How long preflight results can be cached
- **Access-Control-Expose-Headers**: Which headers can be accessed by the client

---

## CORS in FastAPI

FastAPI provides built-in CORS middleware for easy configuration.

### Installation

```bash
pip install fastapi uvicorn
```

### Basic CORS Setup

```python
# main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(title="API with CORS")

# Configure CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Allow all origins
    allow_credentials=True,
    allow_methods=["*"],  # Allow all methods
    allow_headers=["*"],  # Allow all headers
)

@app.get("/api/users")
async def get_users():
    return [{"id": 1, "name": "John"}]
```

### Production-Ready CORS Configuration

```python
# config.py
from pydantic_settings import BaseSettings
from typing import List

class Settings(BaseSettings):
    # CORS settings
    CORS_ORIGINS: List[str] = [
        "http://localhost:3000",
        "http://localhost:8080",
        "https://myapp.com",
        "https://www.myapp.com",
    ]
    CORS_ALLOW_CREDENTIALS: bool = True
    CORS_ALLOW_METHODS: List[str] = ["GET", "POST", "PUT", "DELETE", "OPTIONS"]
    CORS_ALLOW_HEADERS: List[str] = [
        "Accept",
        "Accept-Language",
        "Content-Type",
        "Authorization",
        "X-Requested-With",
    ]
    CORS_EXPOSE_HEADERS: List[str] = [
        "Content-Length",
        "X-Request-ID",
    ]
    CORS_MAX_AGE: int = 600  # 10 minutes
    
    class Config:
        env_file = ".env"

settings = Settings()

# main.py
from fastapi import FastAPI, Header, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
from typing import Optional
import logging

from config import settings

# Setup logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(
    title="Secure API with CORS",
    description="API with production-ready CORS configuration"
)

# Add CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.CORS_ORIGINS,
    allow_credentials=settings.CORS_ALLOW_CREDENTIALS,
    allow_methods=settings.CORS_ALLOW_METHODS,
    allow_headers=settings.CORS_ALLOW_HEADERS,
    expose_headers=settings.CORS_EXPOSE_HEADERS,
    max_age=settings.CORS_MAX_AGE,
)

# Middleware to log CORS requests
@app.middleware("http")
async def log_cors_requests(request, call_next):
    origin = request.headers.get("origin")
    if origin:
        logger.info(f"CORS request from origin: {origin}")
    
    response = await call_next(request)
    
    # Log CORS headers in response
    if origin:
        cors_header = response.headers.get("access-control-allow-origin")
        logger.info(f"CORS response header: {cors_header}")
    
    return response

# Sample endpoints
@app.get("/")
async def root():
    return {"message": "API with CORS enabled"}

@app.get("/api/public")
async def public_endpoint():
    """Public endpoint - no authentication required"""
    return {"message": "This is a public endpoint", "data": [1, 2, 3]}

@app.get("/api/private")
async def private_endpoint(
    authorization: Optional[str] = Header(None)
):
    """
    Private endpoint - requires authentication
    Demonstrates CORS with credentials
    """
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(
            status_code=401,
            detail="Missing or invalid authorization header"
        )
    
    # Validate token (simplified)
    token = authorization.split(" ")[1]
    
    return {
        "message": "This is a private endpoint",
        "user": "authenticated_user",
        "token_preview": token[:10] + "..."
    }

@app.post("/api/data")
async def create_data(data: dict):
    """
    POST endpoint that will trigger preflight
    """
    return {
        "message": "Data created successfully",
        "received": data
    }

@app.options("/api/custom")
async def custom_options():
    """
    Custom OPTIONS handler (usually handled by middleware)
    """
    return JSONResponse(
        content={"message": "Custom OPTIONS response"},
        headers={
            "Access-Control-Allow-Origin": "*",
            "Access-Control-Allow-Methods": "GET, POST, OPTIONS",
            "Access-Control-Allow-Headers": "Content-Type, Authorization",
        }
    )

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Dynamic CORS Origins

```python
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
import re

app = FastAPI()

# Define allowed origin patterns
ALLOWED_ORIGIN_PATTERNS = [
    r"https://.*\.myapp\.com",  # Any subdomain of myapp.com
    r"http://localhost:\d+",     # Any localhost port
]

def is_origin_allowed(origin: str) -> bool:
    """Check if origin matches allowed patterns"""
    for pattern in ALLOWED_ORIGIN_PATTERNS:
        if re.match(pattern, origin):
            return True
    return False

# Custom CORS middleware with dynamic origins
@app.middleware("http")
async def custom_cors_middleware(request: Request, call_next):
    origin = request.headers.get("origin")
    
    # Handle preflight requests
    if request.method == "OPTIONS":
        if origin and is_origin_allowed(origin):
            return JSONResponse(
                content={},
                headers={
                    "Access-Control-Allow-Origin": origin,
                    "Access-Control-Allow-Methods": "GET, POST, PUT, DELETE, OPTIONS",
                    "Access-Control-Allow-Headers": "Content-Type, Authorization",
                    "Access-Control-Allow-Credentials": "true",
                    "Access-Control-Max-Age": "600",
                }
            )
    
    # Process request
    response = await call_next(request)
    
    # Add CORS headers to response
    if origin and is_origin_allowed(origin):
        response.headers["Access-Control-Allow-Origin"] = origin
        response.headers["Access-Control-Allow-Credentials"] = "true"
    
    return response

@app.get("/api/data")
async def get_data():
    return {"data": "This endpoint uses dynamic CORS"}
```

### Environment-Based CORS

```python
# config.py
import os
from typing import List

class Config:
    def get_cors_origins(self) -> List[str]:
        env = os.getenv("ENVIRONMENT", "development")
        
        if env == "production":
            return [
                "https://myapp.com",
                "https://www.myapp.com",
            ]
        elif env == "staging":
            return [
                "https://staging.myapp.com",
                "https://dev.myapp.com",
            ]
        else:  # development
            return [
                "http://localhost:3000",
                "http://localhost:8080",
                "http://127.0.0.1:3000",
            ]

config = Config()

# main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from config import config

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=config.get_cors_origins(),
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

---

## CORS in Flask

Flask uses the `flask-cors` extension for CORS support.

### Installation

```bash
pip install flask flask-cors
```

### Basic CORS Setup

```python
# app.py
from flask import Flask, jsonify
from flask_cors import CORS

app = Flask(__name__)

# Enable CORS for all routes
CORS(app)

@app.route('/api/users')
def get_users():
    return jsonify([{"id": 1, "name": "John"}])

if __name__ == '__main__':
    app.run(debug=True)
```

### Advanced Flask CORS Configuration

```python
# app.py
from flask import Flask, jsonify, request
from flask_cors import CORS, cross_origin
from functools import wraps
import logging

# Setup logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = Flask(__name__)

# Global CORS configuration
cors_config = {
    "origins": [
        "http://localhost:3000",
        "http://localhost:8080",
        "https://myapp.com",
    ],
    "methods": ["GET", "POST", "PUT", "DELETE", "OPTIONS"],
    "allow_headers": [
        "Content-Type",
        "Authorization",
        "X-Requested-With",
    ],
    "expose_headers": [
        "Content-Length",
        "X-Request-ID",
    ],
    "supports_credentials": True,
    "max_age": 600,
}

# Apply CORS to entire app
CORS(app, resources={r"/api/*": cors_config})

# Middleware to log CORS requests
@app.before_request
def log_cors_request():
    origin = request.headers.get('Origin')
    if origin:
        logger.info(f"CORS request from origin: {origin}")
        logger.info(f"Request method: {request.method}")
        logger.info(f"Request path: {request.path}")

@app.after_request
def log_cors_response(response):
    origin = request.headers.get('Origin')
    if origin:
        cors_origin = response.headers.get('Access-Control-Allow-Origin')
        logger.info(f"CORS response origin: {cors_origin}")
    return response

# Public endpoint
@app.route('/api/public', methods=['GET'])
def public_endpoint():
    """Public endpoint with CORS"""
    return jsonify({
        "message": "This is a public endpoint",
        "data": [1, 2, 3]
    })

# Private endpoint with authentication
@app.route('/api/private', methods=['GET'])
@cross_origin(supports_credentials=True)
def private_endpoint():
    """Private endpoint requiring authentication"""
    auth_header = request.headers.get('Authorization')
    
    if not auth_header or not auth_header.startswith('Bearer '):
        return jsonify({"error": "Unauthorized"}), 401
    
    token = auth_header.split(' ')[1]
    
    return jsonify({
        "message": "Private data",
        "user": "authenticated_user",
        "token_preview": token[:10] + "..."
    })

# POST endpoint that triggers preflight
@app.route('/api/data', methods=['POST', 'OPTIONS'])
def create_data():
    """POST endpoint with CORS"""
    if request.method == 'OPTIONS':
        return '', 204
    
    data = request.get_json()
    return jsonify({
        "message": "Data created",
        "received": data
    }), 201

# Route-specific CORS
@app.route('/api/specific', methods=['GET'])
@cross_origin(
    origins=["http://localhost:3000"],
    methods=["GET"],
    allow_headers=["Content-Type"],
)
def specific_cors():
    """Endpoint with specific CORS configuration"""
    return jsonify({"message": "Specific CORS rules apply here"})

# Error handlers with CORS
@app.errorhandler(404)
def not_found(error):
    response = jsonify({"error": "Not found"})
    response.status_code = 404
    return response

@app.errorhandler(500)
def internal_error(error):
    response = jsonify({"error": "Internal server error"})
    response.status_code = 500
    return response

if __name__ == '__main__':
    app.run(debug=True, port=5000)
```

### Flask CORS with Blueprints

```python
# app.py
from flask import Flask
from flask_cors import CORS

app = Flask(__name__)

# Configure CORS for different blueprints
cors_config = {
    "api_v1": {
        "origins": ["http://localhost:3000"],
        "methods": ["GET", "POST"],
    },
    "api_v2": {
        "origins": ["https://myapp.com"],
        "methods": ["GET", "POST", "PUT", "DELETE"],
        "supports_credentials": True,
    }
}

# blueprints/api_v1.py
from flask import Blueprint, jsonify

api_v1 = Blueprint('api_v1', __name__, url_prefix='/api/v1')

@api_v1.route('/users')
def get_users_v1():
    return jsonify([{"id": 1, "name": "John", "version": "v1"}])

# blueprints/api_v2.py
from flask import Blueprint, jsonify

api_v2 = Blueprint('api_v2', __name__, url_prefix='/api/v2')

@api_v2.route('/users')
def get_users_v2():
    return jsonify([{"id": 1, "name": "John", "version": "v2"}])

# Register blueprints with different CORS configs
from blueprints.api_v1 import api_v1
from blueprints.api_v2 import api_v2

app.register_blueprint(api_v1)
app.register_blueprint(api_v2)

# Apply CORS to blueprints
CORS(api_v1, resources={r"/*": cors_config["api_v1"]})
CORS(api_v2, resources={r"/*": cors_config["api_v2"]})
```

### Custom CORS Decorator

```python
from flask import request, make_response
from functools import wraps

def custom_cors(
    origins=None,
    methods=None,
    allow_headers=None,
    expose_headers=None,
    max_age=600
):
    """Custom CORS decorator with fine-grained control"""
    
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            # Handle preflight request
            if request.method == 'OPTIONS':
                response = make_response('', 204)
            else:
                response = make_response(f(*args, **kwargs))
            
            # Get request origin
            origin = request.headers.get('Origin')
            
            # Check if origin is allowed
            if origin and (origins is None or origin in origins):
                response.headers['Access-Control-Allow-Origin'] = origin
            
            # Set other CORS headers
            if methods:
                response.headers['Access-Control-Allow-Methods'] = ', '.join(methods)
            
            if allow_headers:
                response.headers['Access-Control-Allow-Headers'] = ', '.join(allow_headers)
            
            if expose_headers:
                response.headers['Access-Control-Expose-Headers'] = ', '.join(expose_headers)
            
            response.headers['Access-Control-Max-Age'] = str(max_age)
            response.headers['Access-Control-Allow-Credentials'] = 'true'
            
            return response
        
        return decorated_function
    return decorator

# Usage
@app.route('/api/custom', methods=['GET', 'POST', 'OPTIONS'])
@custom_cors(
    origins=['http://localhost:3000'],
    methods=['GET', 'POST'],
    allow_headers=['Content-Type', 'Authorization']
)
def custom_endpoint():
    return jsonify({"message": "Custom CORS applied"})
```

---

## CORS in Django

Django uses the `django-cors-headers` package for CORS support.

### Installation

```bash
pip install django django-cors-headers
```

### Django Settings Configuration

```python
# settings.py
INSTALLED_APPS = [
    # ... other apps
    'corsheaders',
    # ... your apps
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',  # Should be as high as possible
    'django.middleware.common.CommonMiddleware',
    # ... other middleware
]

# CORS Configuration

# Option 1: Allow all origins (DEVELOPMENT ONLY)
CORS_ALLOW_ALL_ORIGINS = True

# Option 2: Specific origins (PRODUCTION)
CORS_ALLOWED_ORIGINS = [
    "http://localhost:3000",
    "http://localhost:8080",
    "https://myapp.com",
    "https://www.myapp.com",
]

# Option 3: Origin regex patterns
CORS_ALLOWED_ORIGIN_REGEXES = [
    r"^https://.*\.myapp\.com$",
    r"^http://localhost:\d+$",
]

# Allow credentials (cookies, authorization headers)
CORS_ALLOW_CREDENTIALS = True

# Allowed HTTP methods
CORS_ALLOW_METHODS = [
    'DELETE',
    'GET',
    'OPTIONS',
    'PATCH',
    'POST',
    'PUT',
]

# Allowed HTTP headers
CORS_ALLOW_HEADERS = [
    'accept',
    'accept-encoding',
    'authorization',
    'content-type',
    'dnt',
    'origin',
    'user-agent',
    'x-csrftoken',
    'x-requested-with',
]

# Headers exposed to the browser
CORS_EXPOSE_HEADERS = [
    'content-length',
    'x-request-id',
]

# Preflight cache duration (seconds)
CORS_PREFLIGHT_MAX_AGE = 600
```

### Environment-Based Django CORS

```python
# settings.py
import os
from typing import List

ENVIRONMENT = os.getenv('ENVIRONMENT', 'development')

def get_cors_allowed_origins() -> List[str]:
    """Get CORS allowed origins based on environment"""
    if ENVIRONMENT == 'production':
        return [
            'https://myapp.com',
            'https://www.myapp.com',
            'https://app.myapp.com',
        ]
    elif ENVIRONMENT == 'staging':
        return [
            'https://staging.myapp.com',
            'https://dev.myapp.com',
        ]
    else:  # development
        return [
            'http://localhost:3000',
            'http://localhost:8080',
            'http://127.0.0.1:3000',
            'http://127.0.0.1:8080',
        ]

# Apply configuration
if ENVIRONMENT == 'development':
    CORS_ALLOW_ALL_ORIGINS = True
else:
    CORS_ALLOWED_ORIGINS = get_cors_allowed_origins()

CORS_ALLOW_CREDENTIALS = True
```

### Custom CORS Middleware

```python
# middleware/cors_middleware.py
import re
from django.utils.deprecation import MiddlewareMixin
import logging

logger = logging.getLogger(__name__)

class CustomCorsMiddleware(MiddlewareMixin):
    """
    Custom CORS middleware with logging and dynamic origin validation
    """
    
    ALLOWED_ORIGIN_PATTERNS = [
        r'^https://.*\.myapp\.com$',
        r'^http://localhost:\d+$',
    ]
    
    def is_origin_allowed(self, origin):
        """Check if origin matches allowed patterns"""
        for pattern in self.ALLOWED_ORIGIN_PATTERNS:
            if re.match(pattern, origin):
                return True
        return False
    
    def process_request(self, request):
        """Log incoming CORS requests"""
        origin = request.META.get('HTTP_ORIGIN')
        if origin:
            logger.info(f"CORS request from: {origin}")
            logger.info(f"Method: {request.method}")
            logger.info(f"Path: {request.path}")
        return None
    
    def process_response(self, request, response):
        """Add CORS headers to response"""
        origin = request.META.get('HTTP_ORIGIN')
        
        if origin and self.is_origin_allowed(origin):
            response['Access-Control-Allow-Origin'] = origin
            response['Access-Control-Allow-Credentials'] = 'true'
            response['Access-Control-Allow-Methods'] = 'GET, POST, PUT, DELETE, OPTIONS'
            response['Access-Control-Allow-Headers'] = 'Content-Type, Authorization'
            response['Access-Control-Max-Age'] = '600'
            
            logger.info(f"CORS headers added for origin: {origin}")
        
        return response

# settings.py
MIDDLEWARE = [
    'middleware.cors_middleware.CustomCorsMiddleware',
    # ... other middleware
]
```

### Django Views with CORS

```python
# views.py
from django.http import JsonResponse
from django.views import View
from django.views.decorators.csrf import csrf_exempt
from django.utils.decorators import method_decorator
import logging

logger = logging.getLogger(__name__)

class PublicAPIView(View):
    """Public API view with CORS"""
    
    @method_decorator(csrf_exempt)
    def dispatch(self, *args, **kwargs):
        return super().dispatch(*args, **kwargs)
    
    def options(self, request, *args, **kwargs):
        """Handle preflight requests"""
        response = JsonResponse({}, status=204)
        return response
    
    def get(self, request, *args, **kwargs):
        """GET endpoint"""
        return JsonResponse({
            'message': 'Public API endpoint',
            'data': [1, 2, 3]
        })
    
    def post(self, request, *args, **kwargs):
        """POST endpoint"""
        import json
        try:
            data = json.loads(request.body)
            return JsonResponse({
                'message': 'Data received',
                'received': data
            }, status=201)
        except json.JSONDecodeError:
            return JsonResponse({
                'error': 'Invalid JSON'
            }, status=400)

# urls.py
from django.urls import path
from .views import PublicAPIView

urlpatterns = [
    path('api/public/', PublicAPIView.as_view(), name='public_api'),
]
```

---

## Advanced CORS Configuration

### 1. Handling Credentials

```python
# FastAPI - Credentials with specific origins
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# ⚠️ IMPORTANT: Cannot use "*" with credentials
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://localhost:3000",
        "https://myapp.com",
    ],  # Specific origins required
    allow_credentials=True,  # Enable credentials
    allow_methods=["*"],
    allow_headers=["*"],
)

# Frontend JavaScript example
"""
// Correct way to send credentials
fetch('http://localhost:8000/api/data', {
    method: 'GET',
    credentials: 'include',  // Send cookies
    headers: {
        'Authorization': 'Bearer token123',
        'Content-Type': 'application/json'
    }
})
"""
```

### 2. Exposing Custom Headers

```python
from fastapi import FastAPI, Response
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    expose_headers=[
        "X-Total-Count",      # Custom pagination header
        "X-Request-ID",        # Request tracking
        "X-RateLimit-Limit",   # Rate limiting
        "X-RateLimit-Remaining",
    ],
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/api/users")
async def get_users(response: Response):
    """Endpoint that sets custom headers"""
    users = [{"id": 1, "name": "John"}]
    
    # Set custom headers
    response.headers["X-Total-Count"] = str(len(users))
    response.headers["X-Request-ID"] = "abc123"
    
    return users

# Frontend can now access these headers
"""
fetch('http://localhost:8000/api/users')
    .then(response => {
        const totalCount = response.headers.get('X-Total-Count');
        const requestId = response.headers.get('X-Request-ID');
        console.log(`Total: ${totalCount}, Request ID: ${requestId}`);
    })
"""
```

### 3. Conditional CORS

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()

@app.middleware("http")
async def conditional_cors(request: Request, call_next):
    """Apply CORS only to API routes"""
    response = await call_next(request)
    
    # Only apply CORS to /api/* routes
    if request.url.path.startswith("/api/"):
        origin = request.headers.get("origin")
        
        if origin:
            # Validate origin
            allowed_origins = [
                "http://localhost:3000",
                "https://myapp.com"
            ]
            
            if origin in allowed_origins:
                response.headers["Access-Control-Allow-Origin"] = origin
                response.headers["Access-Control-Allow-Credentials"] = "true"
    
    return response
```

### 4. CORS with Rate Limiting

```python
from fastapi import FastAPI, Request, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from collections import defaultdict
from datetime import datetime, timedelta
import asyncio

app = FastAPI()

# Rate limiting storage
rate_limit_storage = defaultdict(list)
RATE_LIMIT = 100  # requests
RATE_WINDOW = 3600  # seconds (1 hour)

@app.middleware("http")
async def rate_limit_with_cors(request: Request, call_next):
    """Rate limiting that respects CORS"""
    origin = request.headers.get("origin", "unknown")
    
    # Clean old entries
    now = datetime.now()
    rate_limit_storage[origin] = [
        timestamp for timestamp in rate_limit_storage[origin]
        if now - timestamp < timedelta(seconds=RATE_WINDOW)
    ]
    
    # Check rate limit
    if len(rate_limit_storage[origin]) >= RATE_LIMIT:
        # Return CORS-compliant error
        response = JSONResponse(
            content={"error": "Rate limit exceeded"},
            status_code=429
        )
        response.headers["Access-Control-Allow-Origin"] = origin
        response.headers["X-RateLimit-Limit"] = str(RATE_LIMIT)
        response.headers["X-RateLimit-Remaining"] = "0"
        return response
    
    # Add request to rate limit storage
    rate_limit_storage[origin].append(now)
    
    # Process request
    response = await call_next(request)
    
    # Add rate limit headers
    remaining = RATE_LIMIT - len(rate_limit_storage[origin])
    response.headers["X-RateLimit-Limit"] = str(RATE_LIMIT)
    response.headers["X-RateLimit-Remaining"] = str(remaining)
    
    return response

# Configure CORS with exposed rate limit headers
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    expose_headers=["X-RateLimit-Limit", "X-RateLimit-Remaining"],
    allow_methods=["*"],
    allow_headers=["*"],
)
```

---

## Security Considerations

### 1. Never Use Wildcards in Production with Credentials

```python
# ❌ DANGEROUS - Never do this
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,  # This combination is insecure!
    allow_methods=["*"],
    allow_headers=["*"],
)

# ✅ SECURE - Use specific origins
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://myapp.com",
        "https://www.myapp.com",
    ],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Content-Type", "Authorization"],
)
```

### 2. Validate Origins Carefully

```python
from fastapi import FastAPI, Request
from urllib.parse import urlparse

app = FastAPI()

def is_origin_safe(origin: str) -> bool:
    """Validate origin with strict rules"""
    try:
        parsed = urlparse(origin)
        
        # Check protocol
        if parsed.scheme not in ['http', 'https']:
            return False
        
        # Check domain
        allowed_domains = [
            'myapp.com',
            'www.myapp.com',
            'app.myapp.com',
        ]
        
        # Allow localhost in development
        if parsed.hostname in ['localhost', '127.0.0.1']:
            return True
        
        # Check if domain or subdomain is allowed
        if parsed.hostname in allowed_domains:
            return True
        
        # Check subdomain
        for domain in allowed_domains:
            if parsed.hostname.endswith('.' + domain):
                return True
        
        return False
    except Exception:
        return False

@app.middleware("http")
async def secure_cors(request: Request, call_next):
    origin = request.headers.get("origin")
    
    response = await call_next(request)
    
    if origin and is_origin_safe(origin):
        response.headers["Access-Control-Allow-Origin"] = origin
        response.headers["Access-Control-Allow-Credentials"] = "true"
    
    return response
```

### 3. Implement CSRF Protection

```python
from fastapi import FastAPI, Request, HTTPException, Header
from typing import Optional
import secrets
import hmac
import hashlib

app = FastAPI()

# CSRF token storage (use Redis in production)
csrf_tokens = {}

def generate_csrf_token() -> str:
    """Generate a secure CSRF token"""
    return secrets.token_urlsafe(32)

def validate_csrf_token(token: str, session_id: str) -> bool:
    """Validate CSRF token"""
    stored_token = csrf_tokens.get(session_id)
    if not stored_token:
        return False
    return hmac.compare_digest(stored_token, token)

@app.get("/api/csrf-token")
async def get_csrf_token(request: Request):
    """Get CSRF token for forms"""
    session_id = request.cookies.get("session_id", "default")
    token = generate_csrf_token()
    csrf_tokens[session_id] = token
    
    return {"csrf_token": token}

@app.post("/api/protected")
async def protected_endpoint(
    request: Request,
    x_csrf_token: Optional[str] = Header(None)
):
    """Protected endpoint requiring CSRF token"""
    session_id = request.cookies.get("session_id", "default")
    
    if not x_csrf_token or not validate_csrf_token(x_csrf_token, session_id):
        raise HTTPException(status_code=403, detail="Invalid CSRF token")
    
    return {"message": "Protected action completed"}
```

### 4. Security Headers

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

@app.middleware("http")
async def add_security_headers(request, call_next):
    response = await call_next(request)
    
    # Security headers
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"
    response.headers["X-XSS-Protection"] = "1; mode=block"
    response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
    
    # Content Security Policy
    response.headers["Content-Security-Policy"] = (
        "default-src 'self'; "
        "script-src 'self' 'unsafe-inline'; "
        "style-src 'self' 'unsafe-inline'; "
        "img-src 'self' data: https:; "
    )
    
    return response

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://myapp.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Content-Type", "Authorization"],
)
```

---

## Common Issues and Solutions

### Issue 1: CORS Error Despite Configuration

**Problem:**
```
Access to fetch at 'http://localhost:8000/api/users' from origin 
'http://localhost:3000' has been blocked by CORS policy
```

**Solutions:**

```python
# Solution 1: Check middleware order (FastAPI)
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# CORS middleware should be added BEFORE other middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Other middleware after CORS
app.add_middleware(SomeOtherMiddleware)

# Solution 2: Verify origin exactly matches
# ❌ Wrong: "http://localhost:3000/"  (trailing slash)
# ✅ Correct: "http://localhost:3000"

# Solution 3: Check for http vs https mismatch
allowed_origins = [
    "http://localhost:3000",   # Development
    "https://myapp.com",        # Production (HTTPS!)
]
```

### Issue 2: Preflight Request Failing

**Problem:**
```
Preflight request returns 404 or 405
```

**Solutions:**

```python
# Solution 1: Ensure OPTIONS is allowed
from fastapi import FastAPI

app = FastAPI()

@app.options("/api/users")
async def options_users():
    return {"message": "OK"}

# Solution 2: Let CORS middleware handle OPTIONS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_methods=["GET", "POST", "PUT", "DELETE", "OPTIONS"],  # Include OPTIONS
    allow_headers=["*"],
)

# Solution 3: Check if custom middleware blocks OPTIONS
@app.middleware("http")
async def custom_middleware(request, call_next):
    # Don't block OPTIONS requests!
    if request.method == "OPTIONS":
        return await call_next(request)
    
    # Your middleware logic
    response = await call_next(request)
    return response
```

### Issue 3: Credentials Not Working

**Problem:**
```
Credentials flag is 'true', but the 'Access-Control-Allow-Origin' header is '*'
```

**Solutions:**

```python
# ❌ WRONG: Wildcard with credentials
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Cannot use wildcard with credentials
    allow_credentials=True,
)

# ✅ CORRECT: Specific origins with credentials
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://localhost:3000",
        "https://myapp.com",
    ],
    allow_credentials=True,
)

# Frontend must include credentials
"""
fetch('http://localhost:8000/api/data', {
    method: 'POST',
    credentials: 'include',  // Required!
    headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer token'
    },
    body: JSON.stringify(data)
})
"""
```

### Issue 4: Custom Headers Not Accessible

**Problem:**
```javascript
// Frontend cannot access custom header
const customHeader = response.headers.get('X-Custom-Header'); // null
```

**Solution:**

```python
# Expose custom headers
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    expose_headers=[
        "X-Custom-Header",
        "X-Total-Count",
        "X-Request-ID",
    ],  # Must expose custom headers!
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/api/data")
async def get_data(response: Response):
    response.headers["X-Custom-Header"] = "value"
    response.headers["X-Total-Count"] = "100"
    return {"data": []}
```

---

## Testing CORS

### 1. Manual Testing with cURL

```bash
# Test simple GET request
curl -H "Origin: http://localhost:3000" \
     -H "Accept: application/json" \
     -X GET http://localhost:8000/api/users \
     -v

# Test preflight request
curl -H "Origin: http://localhost:3000" \
     -H "Access-Control-Request-Method: POST" \
     -H "Access-Control-Request-Headers: Content-Type" \
     -X OPTIONS http://localhost:8000/api/users \
     -v

# Test with credentials
curl -H "Origin: http://localhost:3000" \
     -H "Authorization: Bearer token123" \
     -b "session_id=abc123" \
     -X GET http://localhost:8000/api/private \
     -v
```

### 2. Python Tests

```python
# test_cors.py
import pytest
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_cors_get_request():
    """Test simple CORS GET request"""
    response = client.get(
        "/api/users",
        headers={"Origin": "http://localhost:3000"}
    )
    
    assert response.status_code == 200
    assert "access-control-allow-origin" in response.headers
    assert response.headers["access-control-allow-origin"] == "http://localhost:3000"

def test_cors_preflight():
    """Test CORS preflight request"""
    response = client.options(
        "/api/users",
        headers={
            "Origin": "http://localhost:3000",
            "Access-Control-Request-Method": "POST",
            "Access-Control-Request-Headers": "Content-Type",
        }
    )
    
    assert response.status_code in [200, 204]
    assert "access-control-allow-methods" in response.headers
    assert "POST" in response.headers["access-control-allow-methods"]

def test_cors_blocked_origin():
    """Test that unauthorized origins are blocked"""
    response = client.get(
        "/api/users",
        headers={"Origin": "http://malicious-site.com"}
    )
    
    # Should not have CORS headers for blocked origin
    assert "access-control-allow-origin" not in response.headers or \
           response.headers["access-control-allow-origin"] != "http://malicious-site.com"

def test_cors_credentials():
    """Test CORS with credentials"""
    response = client.get(
        "/api/private",
        headers={
            "Origin": "http://localhost:3000",
            "Authorization": "Bearer test-token"
        }
    )
    
    assert "access-control-allow-credentials" in response.headers
    assert response.headers["access-control-allow-credentials"] == "true"

def test_exposed_headers():
    """Test that custom headers are exposed"""
    response = client.get(
        "/api/users",
        headers={"Origin": "http://localhost:3000"}
    )
    
    assert "access-control-expose-headers" in response.headers
    exposed = response.headers["access-control-expose-headers"]
    assert "X-Total-Count" in exposed
```

### 3. Frontend Testing

```javascript
// test-cors.js
describe('CORS Tests', () => {
    const API_URL = 'http://localhost:8000';
    
    test('Simple GET request works', async () => {
        const response = await fetch(`${API_URL}/api/users`);
        expect(response.ok).toBe(true);
        
        const data = await response.json();
        expect(Array.isArray(data)).toBe(true);
    });
    
    test('POST request with JSON works', async () => {
        const response = await fetch(`${API_URL}/api/users`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify({
                name: 'Test User',
                email: 'test@example.com'
            })
        });
        
        expect(response.ok).toBe(true);
    });
    
    test('Request with credentials works', async () => {
        const response = await fetch(`${API_URL}/api/private`, {
            method: 'GET',
            credentials: 'include',
            headers: {
                'Authorization': 'Bearer test-token'
            }
        });
        
        expect(response.ok).toBe(true);
    });
    
    test('Custom headers are accessible', async () => {
        const response = await fetch(`${API_URL}/api/users`);
        const totalCount = response.headers.get('X-Total-Count');
        
        expect(totalCount).toBeTruthy();
        expect(Number(totalCount)).toBeGreaterThan(0);
    });
});
```

---

## Best Practices

### 1. Development vs Production Configuration

```python
# config.py
import os
from typing import List

class Settings:
    ENVIRONMENT = os.getenv("ENVIRONMENT", "development")
    
    @property
    def cors_origins(self) -> List[str]:
        if self.ENVIRONMENT == "production":
            return [
                "https://myapp.com",
                "https://www.myapp.com",
            ]
        else:
            return [
                "http://localhost:3000",
                "http://localhost:8080",
                "http://127.0.0.1:3000",
            ]
    
    @property
    def cors_allow_all(self) -> bool:
        return self.ENVIRONMENT == "development"

settings = Settings()

# main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from config import settings

app = FastAPI()

if settings.cors_allow_all:
    # Development: Allow all origins
    app.add_middleware(
        CORSMiddleware,
        allow_origins=["*"],
        allow_methods=["*"],
        allow_headers=["*"],
    )
else:
    # Production: Strict CORS
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.cors_origins,
        allow_credentials=True,
        allow_methods=["GET", "POST", "PUT", "DELETE"],
        allow_headers=["Content-Type", "Authorization"],
    )
```

### 2. Logging CORS Requests

```python
import logging
from fastapi import FastAPI, Request

logger = logging.getLogger(__name__)

@app.middleware("http")
async def log_cors(request: Request, call_next):
    origin = request.headers.get("origin")
    
    if origin:
        logger.info(
            f"CORS request - "
            f"Origin: {origin}, "
            f"Method: {request.method}, "
            f"Path: {request.url.path}"
        )
    
    response = await call_next(request)
    
    if origin:
        cors_origin = response.headers.get("access-control-allow-origin")
        if cors_origin:
            logger.info(f"CORS allowed for origin: {origin}")
        else:
            logger.warning(f"CORS blocked for origin: {origin}")
    
    return response
```

### 3. Documentation

```python
from fastapi import FastAPI

app = FastAPI(
    title="API with CORS",
    description="""
    ## CORS Configuration
    
    This API is configured to accept requests from the following origins:
    - http://localhost:3000 (Development)
    - https://myapp.com (Production)
    
    ### Authentication
    Most endpoints require Bearer token authentication.
    Credentials (cookies, auth headers) are supported.
    
    ### Allowed Methods
    - GET
    - POST
    - PUT
    - DELETE
    - OPTIONS (for preflight)
    
    ### Custom Headers
    The following custom headers are exposed:
    - X-Total-Count
    - X-Request-ID
    - X-RateLimit-Limit
    - X-RateLimit-Remaining
    """,
)
```

### 4. Monitoring and Alerting

```python
from fastapi import FastAPI, Request
from prometheus_client import Counter, Histogram
import time

# Metrics
cors_requests_total = Counter(
    'cors_requests_total',
    'Total CORS requests',
    ['origin', 'method', 'allowed']
)

cors_request_duration = Histogram(
    'cors_request_duration_seconds',
    'CORS request duration',
    ['origin', 'method']
)

@app.middleware("http")
async def monitor_cors(request: Request, call_next):
    origin = request.headers.get("origin")
    
    if origin:
        start_time = time.time()
        response = await call_next(request)
        duration = time.time() - start_time
        
        # Track metrics
        is_allowed = "access-control-allow-origin" in response.headers
        cors_requests_total.labels(
            origin=origin,
            method=request.method,
            allowed=str(is_allowed)
        ).inc()
        
        cors_request_duration.labels(
            origin=origin,
            method=request.method
        ).observe(duration)
        
        return response
    
    return await call_next(request)
```

---

## Summary

### Key Takeaways:

1. **Understanding CORS**: Know the difference between same-origin and cross-origin requests
2. **Security First**: Never use wildcards with credentials in production
3. **Specific Origins**: Always use specific origins in production
4. **Handle Preflight**: Ensure OPTIONS requests are properly handled
5. **Expose Headers**: Explicitly expose custom headers clients need to access
6. **Environment-Based**: Use different configurations for development and production
7. **Monitor**: Log and monitor CORS requests to detect issues
8. **Test**: Write tests to verify CORS configuration works correctly

### Quick Reference:

```python
# Minimal secure CORS setup
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://myapp.com"],  # Specific origins
    allow_credentials=True,                # Enable credentials
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Content-Type", "Authorization"],
    expose_headers=["X-Total-Count"],      # Custom headers
    max_age=600,                           # Cache preflight
)
```

CORS is a critical security feature. Configure it carefully, test thoroughly, and monitor in production!