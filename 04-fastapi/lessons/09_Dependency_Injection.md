# Dependency Injection

## 9.1 Dependency Basics

### What are Dependencies?

Dependencies are reusable pieces of code that can be shared across multiple path operations. They solve common problems like:
- Database connections
- User authentication
- Input validation
- Configuration access
- Logging and monitoring

**Why Use Dependencies?**
- **Code Reuse**: Write once, use everywhere
- **Separation of Concerns**: Keep logic organized
- **Testing**: Easy to mock and override
- **Type Safety**: Full editor support with type hints
- **Automatic Validation**: FastAPI validates dependency inputs

### Depends() Function

The `Depends()` function tells FastAPI to execute a dependency and inject its result.

**Basic Example**:
```python
from fastapi import FastAPI, Depends

app = FastAPI()

# Simple dependency function
def get_query(q: str = None):
    return q

@app.get("/items/")
async def read_items(query: str = Depends(get_query)):
    return {"query": query}

# FastAPI will:
# 1. Call get_query(q=<value_from_request>)
# 2. Get the result
# 3. Pass it to read_items as 'query'
```

**With Parameters**:
```python
from fastapi import FastAPI, Depends, Query

app = FastAPI()

def common_parameters(
    skip: int = Query(0, ge=0),
    limit: int = Query(10, ge=1, le=100)
):
    return {"skip": skip, "limit": limit}

@app.get("/items/")
async def read_items(commons: dict = Depends(common_parameters)):
    return commons

@app.get("/users/")
async def read_users(commons: dict = Depends(common_parameters)):
    return commons

# Both endpoints share the same pagination logic
# URL: /items/?skip=20&limit=50
# Result: {"skip": 20, "limit": 50}
```

### Function Dependencies

Dependencies can be any callable (function, async function, or class).

**Simple Function Dependency**:
```python
from fastapi import FastAPI, Depends, HTTPException

app = FastAPI()

def verify_token(token: str):
    if token != "secret-token":
        raise HTTPException(status_code=401, detail="Invalid token")
    return token

@app.get("/protected/")
async def protected_route(token: str = Depends(verify_token)):
    return {"message": "Access granted", "token": token}

# Request must include ?token=secret-token
```

**Async Function Dependency**:
```python
import httpx
from fastapi import FastAPI, Depends

app = FastAPI()

async def get_external_data():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
        return response.json()

@app.get("/data/")
async def read_data(external: dict = Depends(get_external_data)):
    return {"external_data": external}
```

**Multiple Dependencies**:
```python
from fastapi import FastAPI, Depends

app = FastAPI()

def get_user_id(user_id: int):
    return user_id

def get_token(token: str):
    return token

@app.get("/items/")
async def read_items(
    user_id: int = Depends(get_user_id),
    token: str = Depends(get_token)
):
    return {
        "user_id": user_id,
        "token": token
    }
```

### Class Dependencies

Classes can be used as dependencies, making them reusable and configurable.

**Basic Class Dependency**:
```python
from fastapi import FastAPI, Depends

app = FastAPI()

class CommonQueryParams:
    def __init__(self, q: str = None, skip: int = 0, limit: int = 10):
        self.q = q
        self.skip = skip
        self.limit = limit

@app.get("/items/")
async def read_items(commons: CommonQueryParams = Depends()):
    # Depends() with no argument uses the type annotation
    return {
        "q": commons.q,
        "skip": commons.skip,
        "limit": commons.limit
    }

# Shorthand (same as above)
@app.get("/users/")
async def read_users(commons: CommonQueryParams = Depends(CommonQueryParams)):
    return commons
```

**Class with Methods**:
```python
from fastapi import FastAPI, Depends, HTTPException

app = FastAPI()

class TokenValidator:
    def __init__(self, token: str):
        self.token = token
    
    def validate(self):
        if self.token != "valid-token":
            raise HTTPException(status_code=401, detail="Invalid token")
        return True
    
    def get_user(self):
        # Simulated user lookup
        return {"username": "alice", "token": self.token}

@app.get("/profile/")
async def get_profile(validator: TokenValidator = Depends()):
    validator.validate()
    return validator.get_user()
```

### Dependency Execution Order

Dependencies are executed in a specific order, which is important for dependencies that depend on each other.

```python
from fastapi import FastAPI, Depends

app = FastAPI()

# Execution order example
def first_dependency():
    print("1. First dependency executed")
    return "first"

def second_dependency(first: str = Depends(first_dependency)):
    print(f"2. Second dependency executed, got: {first}")
    return "second"

def third_dependency(second: str = Depends(second_dependency)):
    print(f"3. Third dependency executed, got: {second}")
    return "third"

@app.get("/order/")
async def test_order(result: str = Depends(third_dependency)):
    print(f"4. Path operation executed, got: {result}")
    return {"result": result}

# When you call GET /order/:
# Console output:
# 1. First dependency executed
# 2. Second dependency executed, got: first
# 3. Third dependency executed, got: second
# 4. Path operation executed, got: third
```

**Dependency Tree**:
```python
from fastapi import FastAPI, Depends

app = FastAPI()

def dep_a():
    print("Dependency A")
    return "A"

def dep_b():
    print("Dependency B")
    return "B"

def dep_c(a: str = Depends(dep_a), b: str = Depends(dep_b)):
    print(f"Dependency C (needs A and B)")
    return f"C({a},{b})"

@app.get("/tree/")
async def test_tree(c: str = Depends(dep_c)):
    return {"result": c}

# Execution:
# Dependency A
# Dependency B
# Dependency C (needs A and B)
```

---

## 9.2 Dependency Types

### Path Operation Dependencies

Dependencies applied to individual path operations.

```python
from fastapi import FastAPI, Depends, HTTPException

app = FastAPI()

def verify_admin(is_admin: bool = False):
    if not is_admin:
        raise HTTPException(status_code=403, detail="Admin access required")

# Applied to single endpoint
@app.delete("/users/{user_id}", dependencies=[Depends(verify_admin)])
async def delete_user(user_id: int):
    return {"deleted": user_id}

# Multiple dependencies
def log_request():
    print("Request logged")

def check_rate_limit():
    print("Rate limit checked")

@app.get("/limited/", dependencies=[
    Depends(log_request),
    Depends(check_rate_limit)
])
async def limited_endpoint():
    return {"message": "Success"}
```

### Global Dependencies

Dependencies applied to all routes in the application.

```python
from fastapi import FastAPI, Depends, Header, HTTPException

async def verify_api_key(x_api_key: str = Header(...)):
    if x_api_key != "secret-api-key":
        raise HTTPException(status_code=401, detail="Invalid API key")

# Global dependency - applies to ALL routes
app = FastAPI(dependencies=[Depends(verify_api_key)])

@app.get("/items/")
async def read_items():
    # Automatically checked for API key
    return {"items": []}

@app.get("/users/")
async def read_users():
    # Also checked for API key
    return {"users": []}

# With multiple global dependencies
def log_all_requests():
    print("Request received")

app = FastAPI(dependencies=[
    Depends(verify_api_key),
    Depends(log_all_requests)
])
```

### Router Dependencies

Dependencies applied to all routes in a router.

```python
from fastapi import APIRouter, Depends, HTTPException

def verify_token(token: str):
    if token != "router-token":
        raise HTTPException(status_code=401, detail="Invalid token")

# Router with dependency
admin_router = APIRouter(
    prefix="/admin",
    tags=["admin"],
    dependencies=[Depends(verify_token)]
)

@admin_router.get("/users/")
async def list_users():
    # Automatically requires token
    return {"users": []}

@admin_router.delete("/users/{user_id}")
async def delete_user(user_id: int):
    # Also requires token
    return {"deleted": user_id}

# Public router without dependencies
public_router = APIRouter(prefix="/public", tags=["public"])

@public_router.get("/items/")
async def list_items():
    # No token required
    return {"items": []}
```

### Application Dependencies

Combining global and router dependencies.

```python
from fastapi import FastAPI, APIRouter, Depends, Header

# Global dependency
def verify_api_key(x_api_key: str = Header(...)):
    return x_api_key

# Router-level dependency
def verify_premium(x_premium: str = Header("false")):
    return x_premium == "true"

# Create app with global dependency
app = FastAPI(dependencies=[Depends(verify_api_key)])

# Create router with additional dependency
premium_router = APIRouter(
    prefix="/premium",
    dependencies=[Depends(verify_premium)]
)

@premium_router.get("/features/")
async def premium_features():
    # Requires BOTH api_key AND premium header
    return {"features": ["advanced", "support"]}

app.include_router(premium_router)

@app.get("/basic/")
async def basic_endpoint():
    # Only requires api_key
    return {"message": "Basic access"}
```

### Sub-dependencies

Dependencies can have their own dependencies, creating a chain.

```python
from fastapi import FastAPI, Depends, HTTPException

app = FastAPI()

# Level 1: Base dependency
def get_token(token: str):
    return token

# Level 2: Depends on get_token
def verify_token(token: str = Depends(get_token)):
    if token != "valid-token":
        raise HTTPException(status_code=401, detail="Invalid token")
    return token

# Level 3: Depends on verify_token
def get_current_user(token: str = Depends(verify_token)):
    # Token already validated by verify_token
    return {"username": "alice", "token": token}

# Path operation uses top-level dependency
@app.get("/users/me")
async def read_current_user(user: dict = Depends(get_current_user)):
    return user

# FastAPI executes: get_token → verify_token → get_current_user
```

**Complex Sub-dependency Chain**:
```python
from fastapi import FastAPI, Depends

app = FastAPI()

def get_db():
    print("→ Opening database")
    return "db_connection"

def get_current_user(db: str = Depends(get_db)):
    print(f"→ Getting user from {db}")
    return {"id": 1, "username": "alice"}

def get_user_permissions(
    user: dict = Depends(get_current_user),
    db: str = Depends(get_db)
):
    print(f"→ Getting permissions for {user['username']} from {db}")
    return ["read", "write"]

@app.get("/check-access/")
async def check_access(permissions: list = Depends(get_user_permissions)):
    return {"permissions": permissions}

# Execution flow:
# → Opening database (for get_current_user)
# → Getting user from db_connection
# → Opening database (for get_user_permissions) - called again!
# → Getting permissions for alice from db_connection
```

### Yielding Dependencies

Dependencies can use `yield` to provide cleanup logic (like context managers).

```python
from fastapi import FastAPI, Depends

app = FastAPI()

def get_db():
    db = "database_connection"
    print(f"Opening {db}")
    try:
        yield db  # This is provided to the path operation
    finally:
        print(f"Closing {db}")

@app.get("/items/")
async def read_items(db: str = Depends(get_db)):
    print(f"Using {db}")
    return {"items": []}

# Execution flow:
# Opening database_connection
# Using database_connection
# Closing database_connection
```

---

## 9.3 Common Dependency Patterns

### Database Session Dependency

```python
from fastapi import FastAPI, Depends
from typing import Generator

app = FastAPI()

# Simulated database session
class DatabaseSession:
    def __init__(self):
        self.connection = None
    
    def connect(self):
        print("Database connected")
        self.connection = "active"
    
    def close(self):
        print("Database closed")
        self.connection = None
    
    def query(self, sql: str):
        return f"Result of: {sql}"

# Dependency with cleanup
def get_db() -> Generator:
    db = DatabaseSession()
    try:
        db.connect()
        yield db
    finally:
        db.close()

@app.get("/users/")
async def get_users(db: DatabaseSession = Depends(get_db)):
    result = db.query("SELECT * FROM users")
    return {"result": result}

@app.post("/users/")
async def create_user(username: str, db: DatabaseSession = Depends(get_db)):
    result = db.query(f"INSERT INTO users (username) VALUES ('{username}')")
    return {"result": result}
```

**With SQLAlchemy**:
```python
from fastapi import FastAPI, Depends
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session

DATABASE_URL = "sqlite:///./test.db"
engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(bind=engine)

app = FastAPI()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/users/{user_id}")
async def get_user(user_id: int, db: Session = Depends(get_db)):
    # user = db.query(User).filter(User.id == user_id).first()
    return {"user_id": user_id}
```

### Current User Dependency

```python
from fastapi import FastAPI, Depends, HTTPException, Header
from typing import Optional

app = FastAPI()

# Simulated user database
users_db = {
    "token123": {"id": 1, "username": "alice", "email": "alice@example.com"},
    "token456": {"id": 2, "username": "bob", "email": "bob@example.com"}
}

def get_token(authorization: str = Header(...)):
    """Extract token from Authorization header."""
    if not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Invalid authorization header")
    return authorization.replace("Bearer ", "")

def get_current_user(token: str = Depends(get_token)):
    """Get user from token."""
    user = users_db.get(token)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid token")
    return user

def get_current_active_user(current_user: dict = Depends(get_current_user)):
    """Ensure user is active."""
    if current_user.get("disabled"):
        raise HTTPException(status_code=400, detail="Inactive user")
    return current_user

@app.get("/users/me")
async def read_current_user(user: dict = Depends(get_current_user)):
    return user

@app.get("/users/me/items")
async def read_user_items(user: dict = Depends(get_current_active_user)):
    return {"user": user["username"], "items": []}

# Request example:
# GET /users/me
# Headers: Authorization: Bearer token123
```

### Pagination Dependency

```python
from fastapi import FastAPI, Depends, Query

app = FastAPI()

class PaginationParams:
    def __init__(
        self,
        skip: int = Query(0, ge=0, description="Number of items to skip"),
        limit: int = Query(10, ge=1, le=100, description="Number of items to return")
    ):
        self.skip = skip
        self.limit = limit

# Simulated database
items_db = [{"id": i, "name": f"Item {i}"} for i in range(100)]

@app.get("/items/")
async def list_items(pagination: PaginationParams = Depends()):
    return {
        "total": len(items_db),
        "skip": pagination.skip,
        "limit": pagination.limit,
        "items": items_db[pagination.skip : pagination.skip + pagination.limit]
    }

# Function-based pagination
def get_pagination(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100)
):
    skip = (page - 1) * page_size
    return {"skip": skip, "limit": page_size, "page": page}

@app.get("/products/")
async def list_products(pagination: dict = Depends(get_pagination)):
    return {
        "page": pagination["page"],
        "items": items_db[pagination["skip"] : pagination["skip"] + pagination["limit"]]
    }
```

### Authentication Dependency

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from datetime import datetime, timedelta
import jwt

app = FastAPI()

SECRET_KEY = "your-secret-key"
ALGORITHM = "HS256"

security = HTTPBearer()

def create_token(user_id: int):
    """Create JWT token."""
    payload = {
        "user_id": user_id,
        "exp": datetime.utcnow() + timedelta(hours=1)
    }
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

def verify_token(credentials: HTTPAuthorizationCredentials = Depends(security)):
    """Verify JWT token."""
    try:
        payload = jwt.decode(credentials.credentials, SECRET_KEY, algorithms=[ALGORITHM])
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.post("/login/")
async def login(username: str, password: str):
    # Validate credentials (simplified)
    if username == "alice" and password == "secret":
        token = create_token(user_id=1)
        return {"access_token": token, "token_type": "bearer"}
    raise HTTPException(status_code=401, detail="Invalid credentials")

@app.get("/protected/")
async def protected_route(payload: dict = Depends(verify_token)):
    return {"message": "Access granted", "user_id": payload["user_id"]}
```

### Authorization Dependency

```python
from fastapi import FastAPI, Depends, HTTPException
from enum import Enum

app = FastAPI()

class Role(str, Enum):
    ADMIN = "admin"
    MODERATOR = "moderator"
    USER = "user"

# Simulated current user
def get_current_user():
    return {"id": 1, "username": "alice", "role": Role.USER}

class RoleChecker:
    def __init__(self, allowed_roles: list[Role]):
        self.allowed_roles = allowed_roles
    
    def __call__(self, user: dict = Depends(get_current_user)):
        if user["role"] not in self.allowed_roles:
            raise HTTPException(
                status_code=403,
                detail="Insufficient permissions"
            )
        return user

# Create reusable permission checkers
allow_admin = RoleChecker([Role.ADMIN])
allow_admin_or_moderator = RoleChecker([Role.ADMIN, Role.MODERATOR])
allow_all = RoleChecker([Role.ADMIN, Role.MODERATOR, Role.USER])

@app.delete("/users/{user_id}", dependencies=[Depends(allow_admin)])
async def delete_user(user_id: int):
    return {"deleted": user_id}

@app.post("/posts/{post_id}/approve")
async def approve_post(
    post_id: int,
    user: dict = Depends(allow_admin_or_moderator)
):
    return {"post_id": post_id, "approved_by": user["username"]}

@app.get("/posts/")
async def list_posts(user: dict = Depends(allow_all)):
    return {"posts": [], "user": user["username"]}
```

### Configuration Dependency

```python
from fastapi import FastAPI, Depends
from pydantic import BaseSettings
from functools import lru_cache

app = FastAPI()

class Settings(BaseSettings):
    app_name: str = "My API"
    debug: bool = False
    database_url: str = "sqlite:///./test.db"
    secret_key: str = "secret"
    
    class Config:
        env_file = ".env"

@lru_cache()
def get_settings():
    """Cached settings - only loaded once."""
    return Settings()

@app.get("/info/")
async def get_info(settings: Settings = Depends(get_settings)):
    return {
        "app_name": settings.app_name,
        "debug": settings.debug
    }

@app.get("/health/")
async def health_check(settings: Settings = Depends(get_settings)):
    return {
        "status": "healthy",
        "database": settings.database_url
    }
```

---

## 9.4 Advanced Dependencies

### Dependencies with Yield (Context Managers)

```python
from fastapi import FastAPI, Depends
import logging

app = FastAPI()

def get_db():
    """Database connection with automatic cleanup."""
    db = {"connection": "open"}
    logging.info("Opening database connection")
    try:
        yield db
    finally:
        logging.info("Closing database connection")
        db["connection"] = "closed"

def get_cache():
    """Cache connection with automatic cleanup."""
    cache = {"status": "connected"}
    print("Connecting to cache")
    try:
        yield cache
    finally:
        print("Disconnecting from cache")

@app.get("/data/")
async def get_data(
    db: dict = Depends(get_db),
    cache: dict = Depends(get_cache)
):
    return {
        "db_status": db["connection"],
        "cache_status": cache["status"]
    }

# Execution:
# Opening database connection
# Connecting to cache
# [Path operation executes]
# Disconnecting from cache
# Closing database connection
```

### Dependencies with Cleanup

```python
from fastapi import FastAPI, Depends, HTTPException

app = FastAPI()

class ResourceManager:
    def __init__(self):
        self.resources = []
    
    def allocate(self, resource: str):
        print(f"Allocating {resource}")
        self.resources.append(resource)
        return resource
    
    def cleanup(self):
        print(f"Cleaning up {len(self.resources)} resources")
        for resource in self.resources:
            print(f"  - Releasing {resource}")
        self.resources.clear()

def get_resource_manager():
    manager = ResourceManager()
    try:
        yield manager
    finally:
        manager.cleanup()

@app.get("/process/")
async def process_data(manager: ResourceManager = Depends(get_resource_manager)):
    manager.allocate("memory")
    manager.allocate("file_handle")
    manager.allocate("network_connection")
    return {"status": "processing", "resources": len(manager.resources)}

# Output:
# Allocating memory
# Allocating file_handle
# Allocating network_connection
# Cleaning up 3 resources
#   - Releasing memory
#   - Releasing file_handle
#   - Releasing network_connection
```

### Cached Dependencies

```python
from fastapi import FastAPI, Depends
from functools import lru_cache
import time

app = FastAPI()

@lru_cache()
def get_expensive_config():
    """This is only calculated once and cached."""
    print("Loading expensive configuration...")
    time.sleep(2)  # Simulate expensive operation
    return {"setting1": "value1", "setting2": "value2"}

call_count = 0

def get_database():
    """Not cached - called every time."""
    global call_count
    call_count += 1
    print(f"Creating database connection #{call_count}")
    return {"connection": f"db_{call_count}"}

@app.get("/config/")
async def read_config(config: dict = Depends(get_expensive_config)):
    # First call: waits 2 seconds
    # Subsequent calls: instant (cached)
    return config

@app.get("/db/")
async def read_db(db: dict = Depends(get_database)):
    # Called every time, not cached
    return db
```

**Manual Caching**:
```python
from fastapi import FastAPI, Depends

app = FastAPI()

_cache = {}

def get_cached_data(key: str):
    """Manually cached dependency."""
    def dependency():
        if key not in _cache:
            print(f"Cache miss for {key}")
            _cache[key] = f"Data for {key}"
        else:
            print(f"Cache hit for {key}")
        return _cache[key]
    return dependency

@app.get("/user/{user_id}")
async def get_user(
    user_id: int,
    data: str = Depends(get_cached_data("user_data"))
):
    return {"user_id": user_id, "data": data}
```

### Optional Dependencies

```python
from fastapi import FastAPI, Depends
from typing import Optional

app = FastAPI()

def get_optional_user(user_id: Optional[int] = None):
    """Optional dependency - doesn't fail if not provided."""
    if user_id:
        return {"id": user_id, "username": f"user_{user_id}"}
    return None

@app.get("/content/")
async def get_content(user: Optional[dict] = Depends(get_optional_user)):
    if user:
        return {"content": "personalized", "user": user}
    return {"content": "public"}

# URL: /content/ → Public content
# URL: /content/?user_id=1 → Personalized content

def get_optional_token(authorization: Optional[str] = None):
    """Return user if token provided, None otherwise."""
    if authorization and authorization.startswith("Bearer "):
        token = authorization.replace("Bearer ", "")
        # Validate token
        return {"token": token, "user": "alice"}
    return None

@app.get("/data/")
async def get_data(auth: Optional[dict] = Depends(get_optional_token)):
    if auth:
        return {"data": "sensitive", "user": auth["user"]}
    return {"data": "public"}
```

### Dependencies in Path, Query, Body

Dependencies can be combined with path/query parameters and request bodies.

```python
from fastapi import FastAPI, Depends, Path, Query, Body
from pydantic import BaseModel

app = FastAPI()

# Dependency
def verify_api_key(api_key: str = Query(...)):
    return api_key == "valid-key"

class Item(BaseModel):
    name: str
    price: float

@app.post("/items/{item_id}")
async def update_item(
    item_id: int = Path(..., ge=1),           # Path parameter
    q: str = Query(None),                      # Query parameter
    item: Item = Body(...),                    # Request body
    is_valid: bool = Depends(verify_api_key)  # Dependency
):
    if not is_valid:
        return {"error": "Invalid API key"}
    
    return {
        "item_id": item_id,
        "query": q,
        "item": item,
        "api_key_valid": is_valid
    }
```

### Dependency Overrides for Testing

FastAPI allows you to override dependencies during testing.

```python
from fastapi import FastAPI, Depends
from fastapi.testclient import TestClient

app = FastAPI()

# Original dependency
def get_db():
    return {"type": "production", "connection": "real_db"}

@app.get("/data/")
async def get_data(db: dict = Depends(get_db)):
    return {"db": db}

# Testing
client = TestClient(app)

# Test with real dependency
response = client.get("/data/")
assert response.json() == {"db": {"type": "production", "connection": "real_db"}}

# Override dependency for testing
def get_test_db():
    return {"type": "test", "connection": "test_db"}

app.dependency_overrides[get_db] = get_test_db

# Test with overridden dependency
response = client.get("/data/")
assert response.json() == {"db": {"type": "test", "connection": "test_db"}}

# Clear overrides
app.dependency_overrides.clear()
```

**Complete Testing Example**:
```python
from fastapi import FastAPI, Depends, HTTPException
from fastapi.testclient import TestClient
import pytest

app = FastAPI()

# Real dependencies
def get_current_user():
    # In production, this validates token and queries database
    raise HTTPException(status_code=401, detail="Not authenticated")

def get_db():
    # In production, opens real database connection
    return {"type": "production", "query": lambda x: f"SELECT {x} FROM prod"}

@app.get("/users/me")
async def read_user(user: dict = Depends(get_current_user)):
    return user

@app.get("/query/")
async def query_db(
    table: str,
    db: dict = Depends(get_db),
    user: dict = Depends(get_current_user)
):
    result = db["query"](table)
    return {"result": result, "user": user}

# Test fixtures
@pytest.fixture
def test_client():
    # Mock user
    def get_test_user():
        return {"id": 1, "username": "testuser"}
    
    # Mock database
    def get_test_db():
        return {"type": "test", "query": lambda x: f"SELECT {x} FROM test"}
    
    # Override dependencies
    app.dependency_overrides[get_current_user] = get_test_user
    app.dependency_overrides[get_db] = get_test_db
    
    client = TestClient(app)
    yield client
    
    # Cleanup
    app.dependency_overrides.clear()

def test_read_user(test_client):
    response = test_client.get("/users/me")
    assert response.status_code == 200
    assert response.json() == {"id": 1, "username": "testuser"}

def test_query_db(test_client):
    response = test_client.get("/query/?table=users")
    assert response.status_code == 200
    data = response.json()
    assert data["result"] == "SELECT users FROM test"
    assert data["user"]["username"] == "testuser"
```

---

## Complete Example: Blog API with Dependencies

```python
from fastapi import FastAPI, Depends, HTTPException, Header
from pydantic import BaseModel
from typing import Optional, Generator
from enum import Enum
import logging

app = FastAPI(title="Blog API with Dependencies")

# ========== Models ==========
class Role(str, Enum):
    ADMIN = "admin"
    AUTHOR = "author"
    READER = "reader"

class User(BaseModel):
    id: int
    username: str
    role: Role

class Post(BaseModel):
    id: int
    title: str
    content: str
    author_id: int

# ========== Database ==========
class Database:
    def __init__(self):
        self.users = {
            "token123": User(id=1, username="alice", role=Role.ADMIN),
            "token456": User(id=2, username="bob", role=Role.AUTHOR),
            "token789": User(id=3, username="charlie", role=Role.READER)
        }
        self.posts = []
    
    def get_user_by_token(self, token: str) -> Optional[User]:
        return self.users.get(token)
    
    def create_post(self, post: Post):
        self.posts.append(post)
        return post
    
    def get_posts(self, skip: int = 0, limit: int = 10):
        return self.posts[skip:skip + limit]

# ========== Dependencies ==========

# Database dependency
def get_db() -> Generator:
    """Provide database session with cleanup."""
    db = Database()
    logging.info("Database session created")
    try:
        yield db
    finally:
        logging.info("Database session closed")

# Pagination dependency
class PaginationParams:
    def __init__(
        self,
        skip: int = 0,
        limit: int = 10
    ):
        self.skip = skip
        self.limit = limit

# Authentication dependency
def get_token(authorization: str = Header(..., description="Bearer token")):
    """Extract token from header."""
    if not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Invalid authorization format")
    return authorization.replace("Bearer ", "")

def get_current_user(
    token: str = Depends(get_token),
    db: Database = Depends(get_db)
) -> User:
    """Get current user from token."""
    user = db.get_user_by_token(token)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid token")
    return user

# Authorization dependency
class RoleChecker:
    def __init__(self, allowed_roles: list[Role]):
        self.allowed_roles = allowed_roles
    
    def __call__(self, user: User = Depends(get_current_user)):
        if user.role not in self.allowed_roles:
            raise HTTPException(
                status_code=403,
                detail=f"Role {user.role} not permitted. Required: {self.allowed_roles}"
            )
        return user

allow_admin = RoleChecker([Role.ADMIN])
allow_author = RoleChecker([Role.ADMIN, Role.AUTHOR])

# ========== Routes ==========

@app.get("/")
async def root():
    return {"message": "Blog API", "docs": "/docs"}

@app.get("/users/me")
async def read_current_user(user: User = Depends(get_current_user)):
    """Get current user profile."""
    return user

@app.get("/posts/")
async def list_posts(
    pagination: PaginationParams = Depends(),
    db: Database = Depends(get_db)
):
    """List all posts (public)."""
    posts = db.get_posts(pagination.skip, pagination.limit)
    return {
        "total": len(db.posts),
        "skip": pagination.skip,
        "limit": pagination.limit,
        "posts": posts
    }

@app.post("/posts/", dependencies=[Depends(allow_author)])
async def create_post(
    title: str,
    content: str,
    user: User = Depends(get_current_user),
    db: Database = Depends(get_db)
):
    """Create a new post (authors only)."""
    post = Post(
        id=len(db.posts) + 1,
        title=title,
        content=content,
        author_id=user.id
    )
    db.create_post(post)
    return post

@app.delete("/posts/{post_id}")
async def delete_post(
    post_id: int,
    user: User = Depends(allow_admin),
    db: Database = Depends(get_db)
):
    """Delete a post (admins only)."""
    return {
        "deleted": post_id,
        "deleted_by": user.username
    }

# ========== Testing ==========
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)

# Test with:
# curl -H "Authorization: Bearer token456" http://localhost:8000/users/me
# curl -H "Authorization: Bearer token456" -X POST "http://localhost:8000/posts/?title=Test&content=Content"
# curl -H "Authorization: Bearer token123" -X DELETE http://localhost:8000/posts/1
```

---

## Summary

You've learned about Dependency Injection in FastAPI:

1. **Dependency Basics**: `Depends()` function, function and class dependencies, execution order
2. **Dependency Types**: Path operation, global, router, application, sub-dependencies, and yielding dependencies
3. **Common Patterns**: Database sessions, current user, pagination, authentication, authorization, and configuration
4. **Advanced Dependencies**: Context managers with yield, cleanup, caching, optional dependencies, and testing overrides

Dependency injection is one of FastAPI's most powerful features, enabling clean code organization, reusability, and easy testing!