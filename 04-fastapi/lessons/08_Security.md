# Security in FastAPI

## 8.1 CORS (Cross-Origin Resource Sharing)

### What is CORS?

CORS is a security mechanism that allows or restricts resources on a web page to be requested from another domain. Browsers enforce the Same-Origin Policy, and CORS provides a way to relax this restriction when needed.

**Same-Origin Policy**: Protocol + Domain + Port must match

```
✓ Same Origin: https://example.com:443 → https://example.com:443
✗ Different Origin: https://example.com → http://example.com (different protocol)
✗ Different Origin: https://example.com → https://api.example.com (different subdomain)
✗ Different Origin: https://example.com:443 → https://example.com:8080 (different port)
```

### CORSMiddleware Configuration

FastAPI provides built-in CORS middleware to handle cross-origin requests.

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# Basic CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Allows all origins (not recommended for production)
    allow_credentials=True,
    allow_methods=["*"],  # Allows all methods
    allow_headers=["*"],  # Allows all headers
)

@app.get("/")
async def root():
    return {"message": "CORS enabled"}
```

### Allowed Origins

Specify which origins are allowed to access your API.

```python
# Production configuration - specific origins
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://myapp.com",
        "https://www.myapp.com",
        "https://admin.myapp.com",
    ],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Development configuration with localhost
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://localhost:3000",
        "http://localhost:8080",
        "http://127.0.0.1:3000",
    ],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Dynamic origin configuration
from typing import List

def get_cors_origins() -> List[str]:
    """Load CORS origins from environment or config"""
    import os
    origins_str = os.getenv("CORS_ORIGINS", "http://localhost:3000")
    return [origin.strip() for origin in origins_str.split(",")]

app.add_middleware(
    CORSMiddleware,
    allow_origins=get_cors_origins(),
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### Allowed Methods

Control which HTTP methods are permitted.

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://myapp.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],  # Specific methods only
    allow_headers=["*"],
)

# Or allow all methods
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://myapp.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### Allowed Headers

Specify which headers can be included in requests.

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://myapp.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=[
        "Content-Type",
        "Authorization",
        "X-Requested-With",
        "X-API-Key",
    ],
)

# Allow all headers (less restrictive)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://myapp.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Expose additional headers to JavaScript
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://myapp.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
    expose_headers=["X-Total-Count", "X-Page-Count"],  # Custom headers visible to JS
)
```

### Credentials Support

Enable sending cookies and authentication headers across origins.

```python
# Allow credentials (cookies, authorization headers)
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://myapp.com",
        "https://admin.myapp.com",
    ],
    allow_credentials=True,  # Important for cookie-based auth
    allow_methods=["*"],
    allow_headers=["*"],
)

# Note: When allow_credentials=True, you CANNOT use allow_origins=["*"]
# This will raise an error:
# app.add_middleware(
#     CORSMiddleware,
#     allow_origins=["*"],
#     allow_credentials=True,  # ❌ Error: Cannot use wildcard with credentials
# )
```

## 8.2 Security Headers

### HTTPS Enforcement

Redirect HTTP traffic to HTTPS and enforce secure connections.

```python
from fastapi import FastAPI, Request
from fastapi.responses import RedirectResponse

app = FastAPI()

@app.middleware("http")
async def https_redirect(request: Request, call_next):
    """Redirect HTTP to HTTPS in production"""
    if request.url.scheme == "http" and not request.url.hostname == "localhost":
        url = request.url.replace(scheme="https")
        return RedirectResponse(url, status_code=301)
    
    response = await call_next(request)
    return response
```

### Security Headers Middleware

Add security headers to all responses.

```python
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        
        # Security headers
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        response.headers["Permissions-Policy"] = "geolocation=(), microphone=(), camera=()"
        
        return response

app = FastAPI()
app.add_middleware(SecurityHeadersMiddleware)

@app.get("/")
async def root():
    return {"message": "Security headers enabled"}
```

### Content Security Policy

Control which resources can be loaded by your application.

```python
class CSPMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        
        # Strict CSP policy
        csp_policy = (
            "default-src 'self'; "
            "script-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net; "
            "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; "
            "font-src 'self' https://fonts.gstatic.com; "
            "img-src 'self' data: https:; "
            "connect-src 'self' https://api.myapp.com; "
            "frame-ancestors 'none'; "
            "base-uri 'self'; "
            "form-action 'self'"
        )
        
        response.headers["Content-Security-Policy"] = csp_policy
        
        return response

app.add_middleware(CSPMiddleware)
```

### X-Frame-Options

Prevent clickjacking attacks by controlling iframe embedding.

```python
from fastapi import Response

@app.middleware("http")
async def add_x_frame_options(request: Request, call_next):
    response = await call_next(request)
    
    # DENY: Cannot be displayed in a frame
    response.headers["X-Frame-Options"] = "DENY"
    
    # SAMEORIGIN: Can only be displayed in a frame on the same origin
    # response.headers["X-Frame-Options"] = "SAMEORIGIN"
    
    # ALLOW-FROM: Can only be displayed in a frame on specified origin
    # response.headers["X-Frame-Options"] = "ALLOW-FROM https://trusted.com"
    
    return response
```

### X-Content-Type-Options

Prevent MIME type sniffing.

```python
@app.middleware("http")
async def add_security_headers(request: Request, call_next):
    response = await call_next(request)
    
    # Prevent browsers from MIME-sniffing
    response.headers["X-Content-Type-Options"] = "nosniff"
    
    return response
```

### Strict-Transport-Security

Force HTTPS connections for a specified duration.

```python
@app.middleware("http")
async def add_hsts_header(request: Request, call_next):
    response = await call_next(request)
    
    # HSTS header - force HTTPS for 1 year
    response.headers["Strict-Transport-Security"] = (
        "max-age=31536000; "
        "includeSubDomains; "
        "preload"
    )
    
    return response

# Complete security headers middleware
@app.middleware("http")
async def add_all_security_headers(request: Request, call_next):
    response = await call_next(request)
    
    response.headers.update({
        "X-Content-Type-Options": "nosniff",
        "X-Frame-Options": "DENY",
        "X-XSS-Protection": "1; mode=block",
        "Strict-Transport-Security": "max-age=31536000; includeSubDomains; preload",
        "Referrer-Policy": "strict-origin-when-cross-origin",
        "Permissions-Policy": "geolocation=(), microphone=(), camera=()",
        "Content-Security-Policy": "default-src 'self'",
    })
    
    return response
```

## 8.3 Input Validation & Sanitization

### SQL Injection Prevention

Use parameterized queries and ORMs to prevent SQL injection.

```python
from sqlalchemy.orm import Session
from sqlalchemy import text

# ❌ VULNERABLE - Never do this!
async def get_user_vulnerable(username: str, db: Session):
    query = f"SELECT * FROM users WHERE username = '{username}'"
    result = db.execute(text(query))
    return result.fetchone()

# ✓ SAFE - Use parameterized queries
async def get_user_safe(username: str, db: Session):
    query = text("SELECT * FROM users WHERE username = :username")
    result = db.execute(query, {"username": username})
    return result.fetchone()

# ✓ SAFE - Use ORM (best practice)
from sqlalchemy.orm import Session
from models import User

async def get_user_orm(username: str, db: Session):
    return db.query(User).filter(User.username == username).first()
```

### XSS Prevention

Sanitize user input to prevent Cross-Site Scripting attacks.

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, validator
import bleach
import html

app = FastAPI()

class CommentCreate(BaseModel):
    content: str
    
    @validator('content')
    def sanitize_content(cls, v):
        # Escape HTML entities
        return html.escape(v)

# Using bleach for more control
class PostCreate(BaseModel):
    title: str
    content: str
    
    @validator('content')
    def sanitize_html(cls, v):
        # Allow only specific safe tags
        allowed_tags = ['p', 'br', 'strong', 'em', 'a']
        allowed_attrs = {'a': ['href', 'title']}
        
        cleaned = bleach.clean(
            v,
            tags=allowed_tags,
            attributes=allowed_attrs,
            strip=True
        )
        return cleaned

@app.post("/comments")
async def create_comment(comment: CommentCreate):
    # content is already sanitized by Pydantic validator
    return {"content": comment.content}

# Manual sanitization function
def sanitize_input(text: str, allow_html: bool = False) -> str:
    """Sanitize user input"""
    if not allow_html:
        # Escape all HTML
        return html.escape(text)
    else:
        # Clean HTML with bleach
        return bleach.clean(
            text,
            tags=['p', 'br', 'strong', 'em', 'ul', 'ol', 'li', 'a'],
            attributes={'a': ['href', 'title']},
            protocols=['http', 'https', 'mailto'],
            strip=True
        )
```

### Path Traversal Prevention

Prevent directory traversal attacks in file operations.

```python
from pathlib import Path
from fastapi import HTTPException
import os

UPLOAD_DIR = Path("/var/app/uploads")

def validate_file_path(filename: str) -> Path:
    """Validate and sanitize file path to prevent traversal"""
    # Remove any path separators and parent directory references
    safe_filename = os.path.basename(filename)
    
    # Construct full path
    file_path = UPLOAD_DIR / safe_filename
    
    # Ensure path is within upload directory
    try:
        file_path.resolve().relative_to(UPLOAD_DIR.resolve())
    except ValueError:
        raise HTTPException(
            status_code=400,
            detail="Invalid file path"
        )
    
    return file_path

@app.get("/files/{filename}")
async def download_file(filename: str):
    # ❌ VULNERABLE
    # file_path = f"/var/app/uploads/{filename}"
    # An attacker could use: ../../../etc/passwd
    
    # ✓ SAFE
    file_path = validate_file_path(filename)
    
    if not file_path.exists():
        raise HTTPException(status_code=404, detail="File not found")
    
    return FileResponse(file_path)

# Additional validation
def is_safe_path(base_dir: Path, user_path: str) -> bool:
    """Check if user path is safe and within base directory"""
    try:
        full_path = (base_dir / user_path).resolve()
        base_dir = base_dir.resolve()
        
        # Check if path is within base directory
        return str(full_path).startswith(str(base_dir))
    except (ValueError, OSError):
        return False

@app.get("/download/{filepath:path}")
async def download(filepath: str):
    if not is_safe_path(UPLOAD_DIR, filepath):
        raise HTTPException(status_code=400, detail="Invalid path")
    
    file_path = UPLOAD_DIR / filepath
    
    if not file_path.is_file():
        raise HTTPException(status_code=404, detail="File not found")
    
    return FileResponse(file_path)
```

### Regex Validation

Use regex patterns to validate input format.

```python
from pydantic import BaseModel, validator, Field
import re

class UserInput(BaseModel):
    username: str = Field(..., min_length=3, max_length=20)
    email: str
    phone: str
    
    @validator('username')
    def validate_username(cls, v):
        # Only alphanumeric and underscores
        if not re.match(r'^[a-zA-Z0-9_]+$', v):
            raise ValueError('Username must contain only letters, numbers, and underscores')
        return v
    
    @validator('phone')
    def validate_phone(cls, v):
        # US phone format: XXX-XXX-XXXX
        if not re.match(r'^\d{3}-\d{3}-\d{4}$', v):
            raise ValueError('Phone must be in format XXX-XXX-XXXX')
        return v

class URLInput(BaseModel):
    url: str
    
    @validator('url')
    def validate_url(cls, v):
        url_pattern = re.compile(
            r'^https?://'  # http:// or https://
            r'(?:(?:[A-Z0-9](?:[A-Z0-9-]{0,61}[A-Z0-9])?\.)+[A-Z]{2,6}\.?|'  # domain
            r'localhost|'  # localhost
            r'\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3})'  # IP
            r'(?::\d+)?'  # optional port
            r'(?:/?|[/?]\S+)$', re.IGNORECASE
        )
        
        if not url_pattern.match(v):
            raise ValueError('Invalid URL format')
        return v

class IPAddress(BaseModel):
    ip: str
    
    @validator('ip')
    def validate_ip(cls, v):
        # IPv4 validation
        ipv4_pattern = r'^(\d{1,3}\.){3}\d{1,3}$'
        
        if not re.match(ipv4_pattern, v):
            raise ValueError('Invalid IPv4 address')
        
        # Check each octet is 0-255
        octets = v.split('.')
        if not all(0 <= int(octet) <= 255 for octet in octets):
            raise ValueError('Invalid IPv4 address range')
        
        return v
```

### Input Length Limits

Enforce maximum input lengths to prevent DoS attacks.

```python
from pydantic import BaseModel, Field, validator

class PostCreate(BaseModel):
    title: str = Field(..., min_length=1, max_length=200)
    content: str = Field(..., min_length=1, max_length=10000)
    tags: list[str] = Field(..., max_items=10)
    
    @validator('tags')
    def validate_tag_length(cls, v):
        for tag in v:
            if len(tag) > 50:
                raise ValueError('Each tag must be 50 characters or less')
        return v

class UserBio(BaseModel):
    bio: str = Field(..., max_length=500)
    
    @validator('bio')
    def validate_bio(cls, v):
        # Count actual characters, not bytes
        if len(v) > 500:
            raise ValueError('Bio must be 500 characters or less')
        
        # Limit newlines
        if v.count('\n') > 10:
            raise ValueError('Bio cannot have more than 10 line breaks')
        
        return v

# Middleware to limit request body size
from fastapi import Request, HTTPException
from starlette.middleware.base import BaseHTTPMiddleware

class RequestSizeLimitMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, max_size: int = 10 * 1024 * 1024):  # 10MB default
        super().__init__(app)
        self.max_size = max_size
    
    async def dispatch(self, request: Request, call_next):
        if request.method in ['POST', 'PUT', 'PATCH']:
            content_length = request.headers.get('content-length')
            
            if content_length and int(content_length) > self.max_size:
                raise HTTPException(
                    status_code=413,
                    detail=f"Request body too large. Max size: {self.max_size} bytes"
                )
        
        return await call_next(request)

app.add_middleware(RequestSizeLimitMiddleware, max_size=5 * 1024 * 1024)  # 5MB
```

### File Upload Validation

Validate uploaded files for type, size, and content.

```python
from fastapi import UploadFile, File, HTTPException
from PIL import Image
import io
import magic  # python-magic library

ALLOWED_IMAGE_TYPES = {'image/jpeg', 'image/png', 'image/gif', 'image/webp'}
MAX_FILE_SIZE = 5 * 1024 * 1024  # 5MB

async def validate_image_upload(file: UploadFile) -> UploadFile:
    """Validate image file upload"""
    
    # Check content type
    if file.content_type not in ALLOWED_IMAGE_TYPES:
        raise HTTPException(
            status_code=400,
            detail=f"Invalid file type. Allowed types: {ALLOWED_IMAGE_TYPES}"
        )
    
    # Read file content
    contents = await file.read()
    
    # Check file size
    if len(contents) > MAX_FILE_SIZE:
        raise HTTPException(
            status_code=400,
            detail=f"File too large. Max size: {MAX_FILE_SIZE} bytes"
        )
    
    # Verify actual file type (not just extension)
    mime = magic.from_buffer(contents, mime=True)
    if mime not in ALLOWED_IMAGE_TYPES:
        raise HTTPException(
            status_code=400,
            detail="File content does not match declared type"
        )
    
    # Validate image can be opened
    try:
        image = Image.open(io.BytesIO(contents))
        image.verify()  # Verify it's a valid image
    except Exception:
        raise HTTPException(status_code=400, detail="Invalid image file")
    
    # Check image dimensions
    image = Image.open(io.BytesIO(contents))
    width, height = image.size
    
    if width > 4000 or height > 4000:
        raise HTTPException(
            status_code=400,
            detail="Image dimensions too large. Max: 4000x4000"
        )
    
    # Reset file pointer
    await file.seek(0)
    
    return file

@app.post("/upload/image")
async def upload_image(file: UploadFile = File(...)):
    validated_file = await validate_image_upload(file)
    
    # Process file
    contents = await validated_file.read()
    
    # Save file
    file_path = f"uploads/{validated_file.filename}"
    with open(file_path, "wb") as f:
        f.write(contents)
    
    return {"filename": validated_file.filename, "size": len(contents)}

# Validate file extension
ALLOWED_EXTENSIONS = {'.jpg', '.jpeg', '.png', '.gif', '.pdf', '.doc', '.docx'}

def validate_file_extension(filename: str) -> bool:
    """Check if file extension is allowed"""
    ext = os.path.splitext(filename)[1].lower()
    return ext in ALLOWED_EXTENSIONS

@app.post("/upload/document")
async def upload_document(file: UploadFile = File(...)):
    if not validate_file_extension(file.filename):
        raise HTTPException(
            status_code=400,
            detail=f"Invalid file extension. Allowed: {ALLOWED_EXTENSIONS}"
        )
    
    # Sanitize filename
    safe_filename = os.path.basename(file.filename)
    safe_filename = re.sub(r'[^\w\s.-]', '', safe_filename)
    
    return {"filename": safe_filename}
```

## 8.4 Rate Limiting

### slowapi Library

Install and configure slowapi for rate limiting.

```bash
pip install slowapi
```

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from fastapi import FastAPI, Request

# Initialize limiter
limiter = Limiter(key_func=get_remote_address)

app = FastAPI()
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

# Apply rate limit to endpoint
@app.get("/limited")
@limiter.limit("5/minute")
async def limited_route(request: Request):
    return {"message": "This endpoint is rate limited"}

# Different limits for different endpoints
@app.post("/api/search")
@limiter.limit("10/minute")
async def search(request: Request, query: str):
    return {"results": []}

@app.post("/api/create")
@limiter.limit("3/minute")
async def create_item(request: Request, item: dict):
    return {"created": True}
```

### Rate Limiting Strategies

Implement different rate limiting strategies.

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

# Fixed window
@app.get("/fixed-window")
@limiter.limit("10/minute")
async def fixed_window(request: Request):
    return {"message": "10 requests per minute"}

# Multiple limits
@app.get("/multiple-limits")
@limiter.limit("5/second")
@limiter.limit("100/minute")
@limiter.limit("1000/hour")
async def multiple_limits(request: Request):
    return {"message": "Multiple rate limits applied"}

# Different limits for different HTTP methods
@app.get("/resource")
@limiter.limit("100/minute")
async def get_resource(request: Request):
    return {"data": "resource"}

@app.post("/resource")
@limiter.limit("10/minute")
async def create_resource(request: Request, data: dict):
    return {"created": True}

# Exempt certain routes
@app.get("/health")
@limiter.exempt
async def health_check():
    return {"status": "healthy"}
```

### Per-User Rate Limiting

Rate limit based on authenticated user.

```python
from fastapi import Depends, HTTPException
from slowapi import Limiter

def get_user_id(request: Request) -> str:
    """Extract user ID from JWT token or API key"""
    auth_header = request.headers.get("Authorization")
    
    if not auth_header:
        return get_remote_address(request)
    
    # Extract token and decode (simplified)
    try:
        token = auth_header.replace("Bearer ", "")
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload.get("sub", get_remote_address(request))
    except:
        return get_remote_address(request)

# Create limiter with custom key function
user_limiter = Limiter(key_func=get_user_id)

app = FastAPI()
app.state.limiter = user_limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.post("/api/expensive-operation")
@user_limiter.limit("5/hour")
async def expensive_operation(request: Request):
    return {"message": "Limited to 5 per hour per user"}

# Different limits for different user tiers
def get_rate_limit_for_user(request: Request) -> str:
    """Return rate limit based on user tier"""
    user = get_current_user_from_request(request)
    
    if user.tier == "premium":
        return "100/minute"
    elif user.tier == "pro":
        return "50/minute"
    else:
        return "10/minute"

@app.get("/api/data")
async def get_data(request: Request):
    limit = get_rate_limit_for_user(request)
    
    @limiter.limit(limit)
    async def _get_data():
        return {"data": "results"}
    
    return await _get_data()
```

### Per-Endpoint Rate Limiting

Configure different limits for different endpoints.

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

app = FastAPI()
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

# Public endpoints - stricter limits
@app.get("/public/data")
@limiter.limit("10/minute")
async def public_data(request: Request):
    return {"data": "public"}

# Authenticated endpoints - more lenient
@app.get("/api/user/data")
@limiter.limit("100/minute")
async def user_data(request: Request, current_user: User = Depends(get_current_user)):
    return {"data": "user specific"}

# Admin endpoints - very lenient
@app.get("/admin/data")
@limiter.limit("1000/minute")
async def admin_data(
    request: Request,
    current_user: User = Depends(require_role([Role.ADMIN]))
):
    return {"data": "admin"}

# Write operations - stricter than read
@app.get("/posts")
@limiter.limit("100/minute")
async def get_posts(request: Request):
    return {"posts": []}

@app.post("/posts")
@limiter.limit("10/minute")
async def create_post(request: Request, post: dict):
    return {"created": True}

@app.delete("/posts/{post_id}")
@limiter.limit("5/minute")
async def delete_post(request: Request, post_id: int):
    return {"deleted": True}
```

### Custom Rate Limit Handlers

Create custom responses for rate limit errors.

```python
from slowapi.errors import RateLimitExceeded
from fastapi.responses import JSONResponse

@app.exception_handler(RateLimitExceeded)
async def custom_rate_limit_handler(request: Request, exc: RateLimitExceeded):
    return JSONResponse(
        status_code=429,
        content={
            "error": "Rate limit exceeded",
            "message": "Too many requests. Please try again later.",
            "retry_after": exc.detail
        },
        headers={
            "Retry-After": str(exc.detail)
        }
    )

# Include rate limit info in response headers
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.middleware import SlowAPIMiddleware

app.add_middleware(SlowAPIMiddleware)

@app.get("/with-headers")
@limiter.limit("10/minute")
async def with_headers(request: Request, response: Response):
    # Add custom headers showing rate limit status
    response.headers["X-RateLimit-Limit"] = "10"
    response.headers["X-RateLimit-Remaining"] = "5"
    response.headers["X-RateLimit-Reset"] = "60"
    
    return {"message": "Check response headers for rate limit info"}
```

### Distributed Rate Limiting with Redis

Use Redis for rate limiting across multiple servers.

```bash
pip install redis
```

```python
import redis
from datetime import datetime, timedelta
from fastapi import Request, HTTPException

# Redis connection
redis_client = redis.Redis(host='localhost', port=6379, db=0, decode_responses=True)

class RedisRateLimiter:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
    
    def is_allowed(
        self,
        key: str,
        max_requests: int,
        window_seconds: int
    ) -> tuple[bool, int]:
        """
        Check if request is allowed.
        Returns (is_allowed, remaining_requests)
        """
        current_time = datetime.now()
        window_key = f"rate_limit:{key}:{current_time.minute // (window_seconds // 60)}"
        
        # Increment counter
        current_count = self.redis.incr(window_key)
        
        # Set expiration on first request
        if current_count == 1:
            self.redis.expire(window_key, window_seconds)
        
        remaining = max(0, max_requests - current_count)
        is_allowed = current_count <= max_requests
        
        return is_allowed, remaining

# Initialize rate limiter
rate_limiter = RedisRateLimiter(redis_client)

async def rate_limit_dependency(request: Request):
    """Rate limit dependency using Redis"""
    # Get client identifier
    client_id = request.client.host
    
    # Check rate limit (10 requests per minute)
    is_allowed, remaining = rate_limiter.is_allowed(
        key=client_id,
        max_requests=10,
        window_seconds=60
    )
    
    if not is_allowed:
        raise HTTPException(
            status_code=429,
            detail="Rate limit exceeded",
            headers={"Retry-After": "60"}
        )
    
    # Add rate limit info to request state
    request.state.rate_limit_remaining = remaining

@app.get("/redis-limited")
async def redis_limited(request: Request, _=Depends(rate_limit_dependency)):
    return {
        "message": "Success",
        "rate_limit_remaining": request.state.rate_limit_remaining
    }

# Sliding window rate limiter
class SlidingWindowRateLimiter:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
    
    def is_allowed(
        self,
        key: str,
        max_requests: int,
        window_seconds: int
    ) -> bool:
        """Sliding window rate limiter"""
        now = datetime.now().timestamp()
        window_start = now - window_seconds
        
        # Redis sorted set key
        rate_key = f"rate_limit:{key}"
        
        # Remove old entries
        self.redis.zremrangebyscore(rate_key, 0, window_start)
        
        # Count requests in window
        request_count = self.redis.zcard(rate_key)
        
        if request_count < max_requests:
            # Add current request
            self.redis.zadd(rate_key, {str(now): now})
            self.redis.expire(rate_key, window_seconds)
            return True
        
        return False

sliding_limiter = SlidingWindowRateLimiter(redis_client)

async def sliding_window_limit(request: Request):
    client_id = request.client.host
    
    if not sliding_limiter.is_allowed(
        key=client_id,
        max_requests=10,
        window_seconds=60
    ):
        raise HTTPException(status_code=429, detail="Rate limit exceeded")

@app.get("/sliding-window")
async def sliding_window_route(request: Request, _=Depends(sliding_window_limit)):
    return {"message": "Sliding window rate limiting"}
```

## 8.5 HTTPS & SSL

### SSL/TLS Certificates

Understanding SSL/TLS certificates for HTTPS.

```python
# Types of certificates:
# 1. Self-signed (development only)
# 2. Let's Encrypt (free, automated)
# 3. Commercial CA (paid, extended validation)

# Generate self-signed certificate for development
"""
openssl req -x509 -newkey rsa:4096 -nodes \
  -out cert.pem \
  -keyout key.pem \
  -days 365 \
  -subj "/CN=localhost"
"""
```

### Running FastAPI with HTTPS

Configure FastAPI/Uvicorn to use SSL certificates.

```python
# Run with Uvicorn and SSL
"""
uvicorn main:app --host 0.0.0.0 --port 443 \
  --ssl-keyfile=./key.pem \
  --ssl-certfile=./cert.pem
"""

# In code
import uvicorn

if __name__ == "__main__":
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=443,
        ssl_keyfile="./private.key",
        ssl_certfile="./certificate.crt",
        ssl_ca_certs="./ca_bundle.crt"  # Optional: CA bundle
    )

# Production configuration
if __name__ == "__main__":
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=443,
        ssl_keyfile="/etc/letsencrypt/live/example.com/privkey.pem",
        ssl_certfile="/etc/letsencrypt/live/example.com/fullchain.pem",
        workers=4
    )
```

### Let's Encrypt Integration

Automate SSL certificate management with Let's Encrypt.

```bash
# Install Certbot
sudo apt-get update
sudo apt-get install certbot

# Obtain certificate
sudo certbot certonly --standalone -d example.com -d www.example.com

# Certificates will be saved to:
# /etc/letsencrypt/live/example.com/fullchain.pem
# /etc/letsencrypt/live/example.com/privkey.pem
```

```python
# Automatic certificate renewal script
# /etc/cron.d/certbot-renew

"""
0 3 * * * certbot renew --quiet --post-hook "systemctl reload fastapi"
"""

# FastAPI app with Let's Encrypt
import uvicorn
from pathlib import Path

CERT_DIR = Path("/etc/letsencrypt/live/example.com")

if __name__ == "__main__":
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=443,
        ssl_keyfile=str(CERT_DIR / "privkey.pem"),
        ssl_certfile=str(CERT_DIR / "fullchain.pem"),
        workers=4
    )
```

### Certificate Management

Implement certificate rotation and monitoring.

```python
from pathlib import Path
from datetime import datetime
import ssl
import socket

def check_certificate_expiry(cert_path: str) -> int:
    """Check days until certificate expires"""
    import OpenSSL
    
    with open(cert_path, 'r') as f:
        cert_data = f.read()
    
    cert = OpenSSL.crypto.load_certificate(
        OpenSSL.crypto.FILETYPE_PEM,
        cert_data
    )
    
    # Get expiration date
    expires = datetime.strptime(
        cert.get_notAfter().decode('ascii'),
        '%Y%m%d%H%M%SZ'
    )
    
    days_remaining = (expires - datetime.now()).days
    return days_remaining

def validate_certificate(cert_path: str, key_path: str) -> bool:
    """Validate that certificate and key match"""
    import OpenSSL
    
    with open(cert_path, 'r') as f:
        cert = OpenSSL.crypto.load_certificate(
            OpenSSL.crypto.FILETYPE_PEM,
            f.read()
        )
    
    with open(key_path, 'r') as f:
        key = OpenSSL.crypto.load_privatekey(
            OpenSSL.crypto.FILETYPE_PEM,
            f.read()
        )
    
    # Create a context and check if they work together
    context = OpenSSL.SSL.Context(OpenSSL.SSL.TLSv1_2_METHOD)
    context.use_certificate(cert)
    context.use_privatekey(key)
    
    try:
        context.check_privatekey()
        return True
    except:
        return False

# Startup check
@app.on_event("startup")
async def check_ssl_certificate():
    """Check SSL certificate on startup"""
    cert_path = "/etc/letsencrypt/live/example.com/fullchain.pem"
    key_path = "/etc/letsencrypt/live/example.com/privkey.pem"
    
    # Check expiration
    days_remaining = check_certificate_expiry(cert_path)
    
    if days_remaining < 30:
        print(f"WARNING: SSL certificate expires in {days_remaining} days")
    
    # Validate certificate
    if not validate_certificate(cert_path, key_path):
        raise Exception("SSL certificate and key do not match!")
    
    print(f"SSL certificate valid. Expires in {days_remaining} days.")

# Nginx configuration for SSL termination
"""
server {
    listen 443 ssl http2;
    server_name example.com;
    
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    
    # Modern SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
    ssl_prefer_server_ciphers off;
    
    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name example.com;
    return 301 https://$server_name$request_uri;
}
"""
```

---

## Complete Security Configuration Example

```python
from fastapi import FastAPI, Request, Depends, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from starlette.middleware.base import BaseHTTPMiddleware
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
import uvicorn

app = FastAPI(
    title="Secure FastAPI Application",
    docs_url=None,  # Disable docs in production
    redoc_url=None,
)

# CORS Configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://myapp.com",
        "https://www.myapp.com",
    ],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["*"],
    expose_headers=["X-Total-Count"],
)

# Security Headers Middleware
class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
        response.headers["Content-Security-Policy"] = "default-src 'self'"
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        
        return response

app.add_middleware(SecurityHeadersMiddleware)

# Rate Limiting
limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

# Request Size Limit
class RequestSizeLimitMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        if request.method in ['POST', 'PUT', 'PATCH']:
            content_length = request.headers.get('content-length')
            if content_length and int(content_length) > 10 * 1024 * 1024:  # 10MB
                raise HTTPException(status_code=413, detail="Request too large")
        
        return await call_next(request)

app.add_middleware(RequestSizeLimitMiddleware)

# Routes
@app.get("/")
@limiter.limit("100/minute")
async def root(request: Request):
    return {"message": "Secure FastAPI Application"}

@app.get("/health")
@limiter.exempt
async def health_check():
    return {"status": "healthy"}

if __name__ == "__main__":
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=443,
        ssl_keyfile="/etc/letsencrypt/live/example.com/privkey.pem",
        ssl_certfile="/etc/letsencrypt/live/example.com/fullchain.pem",
        workers=4
    )
```

---

**Key Takeaways:**
- Always use HTTPS in production
- Configure CORS properly (avoid allow_origins=["*"] with credentials)
- Add security headers to all responses
- Validate and sanitize all user input
- Use parameterized queries to prevent SQL injection
- Implement rate limiting to prevent abuse
- Use Let's Encrypt for free SSL certificates
- Monitor certificate expiration
- Regular security audits and updates

**Additional Resources:**
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [FastAPI Security](https://fastapi.tiangolo.com/tutorial/security/)
- [Let's Encrypt](https://letsencrypt.org/)
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
