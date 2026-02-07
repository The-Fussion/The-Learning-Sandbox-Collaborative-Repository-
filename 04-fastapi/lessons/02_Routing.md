# Routing & Path Operations

## 2.1 Basic Routing

### Path Operations (HTTP Methods)

Path operations define how your API responds to different HTTP methods at specific URL paths.

**HTTP Methods Overview**:

| Method | Purpose | Has Body | Idempotent | Common Use |
|--------|---------|----------|------------|------------|
| GET | Retrieve data | ❌ No | ✅ Yes | Read resources |
| POST | Create resource | ✅ Yes | ❌ No | Create new items |
| PUT | Update/Replace | ✅ Yes | ✅ Yes | Full update |
| PATCH | Partial update | ✅ Yes | ❌ No | Partial update |
| DELETE | Remove resource | ❌ No | ✅ Yes | Delete items |

**Basic Examples**:
```python
from fastapi import FastAPI

app = FastAPI()

# GET - Retrieve data
@app.get("/items/")
async def list_items():
    return [{"id": 1, "name": "Item 1"}]

@app.get("/items/{item_id}")
async def get_item(item_id: int):
    return {"id": item_id, "name": "Item"}

# POST - Create resource
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float

@app.post("/items/")
async def create_item(item: Item):
    return {"id": 1, **item.dict()}

# PUT - Full update
@app.put("/items/{item_id}")
async def update_item(item_id: int, item: Item):
    return {"id": item_id, **item.dict()}

# PATCH - Partial update
from typing import Optional

class ItemUpdate(BaseModel):
    name: Optional[str] = None
    price: Optional[float] = None

@app.patch("/items/{item_id}")
async def partial_update_item(item_id: int, item: ItemUpdate):
    update_data = item.dict(exclude_unset=True)
    return {"id": item_id, "updated_fields": update_data}

# DELETE - Remove resource
@app.delete("/items/{item_id}")
async def delete_item(item_id: int):
    return {"message": f"Item {item_id} deleted"}
```

### Creating Endpoints

**Basic Endpoint Structure**:
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/path")           # Decorator: method + path
async def function_name():   # Function: can be sync or async
    """Docstring for API docs."""
    return {"key": "value"}  # Return: auto-converted to JSON
```

**Multiple Endpoints**:
```python
from fastapi import FastAPI, HTTPException

app = FastAPI()

# Simulated database
books_db = []

# CREATE
@app.post("/books/")
async def create_book(title: str, author: str):
    book = {"id": len(books_db) + 1, "title": title, "author": author}
    books_db.append(book)
    return book

# READ ALL
@app.get("/books/")
async def list_books():
    return books_db

# READ ONE
@app.get("/books/{book_id}")
async def get_book(book_id: int):
    for book in books_db:
        if book["id"] == book_id:
            return book
    raise HTTPException(status_code=404, detail="Book not found")

# UPDATE
@app.put("/books/{book_id}")
async def update_book(book_id: int, title: str, author: str):
    for idx, book in enumerate(books_db):
        if book["id"] == book_id:
            books_db[idx] = {"id": book_id, "title": title, "author": author}
            return books_db[idx]
    raise HTTPException(status_code=404, detail="Book not found")

# DELETE
@app.delete("/books/{book_id}")
async def delete_book(book_id: int):
    for idx, book in enumerate(books_db):
        if book["id"] == book_id:
            books_db.pop(idx)
            return {"message": "Book deleted"}
    raise HTTPException(status_code=404, detail="Book not found")
```

### Function-Based Path Operations

**Sync vs Async Functions**:
```python
from fastapi import FastAPI
import time
import asyncio

app = FastAPI()

# Synchronous (blocking)
@app.get("/sync")
def sync_endpoint():
    time.sleep(1)  # Blocks the thread
    return {"type": "sync"}

# Asynchronous (non-blocking)
@app.get("/async")
async def async_endpoint():
    await asyncio.sleep(1)  # Doesn't block
    return {"type": "async"}

# Choose async for I/O operations
import httpx

@app.get("/external-data")
async def fetch_external():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
        return response.json()
```

### Path Operation Decorators

**Decorator Parameters**:
```python
from fastapi import FastAPI, status

app = FastAPI()

@app.post(
    "/items/",
    status_code=status.HTTP_201_CREATED,  # Custom status code
    tags=["items"],                        # Group in docs
    summary="Create an item",              # Short description
    description="Create a new item with name and price",
    response_description="The created item",
    deprecated=False,                      # Deprecation flag
    responses={                            # Additional responses
        404: {"description": "Not found"},
        409: {"description": "Conflict"}
    }
)
async def create_item(name: str, price: float):
    """
    Create an item with:
    - **name**: item name
    - **price**: item price (must be positive)
    """
    return {"name": name, "price": price}
```

### Operation Order Matters

The order of path operations is critical - FastAPI evaluates them sequentially.

**❌ Wrong Order**:
```python
from fastapi import FastAPI

app = FastAPI()

# Wrong: Variable path first
@app.get("/users/{user_id}")
async def get_user(user_id: str):
    return {"user_id": user_id}

@app.get("/users/me")  # Never reached! "me" matches {user_id}
async def get_current_user():
    return {"user": "current_user"}
```

**✅ Correct Order**:
```python
from fastapi import FastAPI

app = FastAPI()

# Correct: Specific paths first
@app.get("/users/me")
async def get_current_user():
    return {"user": "current_user"}

@app.get("/users/premium")
async def get_premium_users():
    return {"users": "premium"}

@app.get("/users/{user_id}")  # Variable path last
async def get_user(user_id: int):
    return {"user_id": user_id}

# Now:
# /users/me → get_current_user()
# /users/premium → get_premium_users()
# /users/123 → get_user(123)
```

**Complex Example**:
```python
from fastapi import FastAPI

app = FastAPI()

# Order: Most specific → Most general
@app.get("/files/recent")          # Most specific
@app.get("/files/archived")        # Also specific
@app.get("/files/{file_id}")       # Variable (less specific)
@app.get("/files/{category}/{id}") # Multiple variables (most general)
```

### Deprecated Operations

Mark endpoints as deprecated to signal future removal.

```python
from fastapi import FastAPI

app = FastAPI()

# Simple deprecation
@app.get("/old-endpoint", deprecated=True)
async def old_endpoint():
    """
    **DEPRECATED**: This endpoint will be removed in v2.0.
    Please use `/new-endpoint` instead.
    """
    return {"message": "This is deprecated"}

@app.get("/new-endpoint")
async def new_endpoint():
    """New and improved endpoint."""
    return {"message": "Use this instead"}

# With detailed information
@app.get(
    "/api/v1/users",
    deprecated=True,
    summary="Get users (Deprecated)",
    description="""
    **DEPRECATED** as of v2.0.
    
    Removal date: January 2025
    Replacement: `/api/v2/users`
    
    Migration guide: https://docs.example.com/migration
    """
)
async def get_users_v1():
    return {"version": "v1", "users": []}
```

---

## 2.2 Path Parameters

### Basic Path Parameters

Path parameters are variables in the URL path used to identify resources.

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/{item_id}")
async def read_item(item_id: int):
    return {"item_id": item_id}

@app.get("/users/{user_id}/posts/{post_id}")
async def get_user_post(user_id: int, post_id: int):
    return {"user_id": user_id, "post_id": post_id}

# URL: /items/42 → item_id = 42
# URL: /users/10/posts/5 → user_id = 10, post_id = 5
```

### Type Conversion and Validation

FastAPI automatically converts and validates path parameters.

```python
from fastapi import FastAPI

app = FastAPI()

# Integer
@app.get("/items/{item_id}")
async def read_item(item_id: int):
    return {"item_id": item_id, "type": "int"}
# Valid: /items/42
# Invalid: /items/abc → 422 Error

# Float
@app.get("/prices/{price}")
async def read_price(price: float):
    return {"price": price, "with_tax": price * 1.08}
# Valid: /prices/19.99, /prices/20

# Boolean
@app.get("/active/{is_active}")
async def read_active(is_active: bool):
    return {"is_active": is_active}
# Valid: /active/true, /active/1, /active/false, /active/0

# String (default)
@app.get("/users/{username}")
async def read_user(username: str):
    return {"username": username}
# Valid: /users/alice, /users/john_doe
```

### Integer, String, Float, UUID Path Parameters

```python
from fastapi import FastAPI
from uuid import UUID
from datetime import datetime

app = FastAPI()

# UUID
@app.get("/items/{item_id}")
async def read_item(item_id: UUID):
    return {"item_id": item_id}
# Valid: /items/123e4567-e89b-12d3-a456-426614174000
# Invalid: /items/123 → 422 Error

# DateTime
@app.get("/reports/{date}")
async def read_report(date: datetime):
    return {"date": date}
# Valid: /reports/2024-01-15T10:30:00
# Valid: /reports/2024-01-15
```

### Enum Path Parameters

Restrict path parameters to predefined values.

```python
from fastapi import FastAPI
from enum import Enum

app = FastAPI()

class ModelName(str, Enum):
    alexnet = "alexnet"
    resnet = "resnet"
    lenet = "lenet"

@app.get("/models/{model_name}")
async def get_model(model_name: ModelName):
    if model_name == ModelName.alexnet:
        return {"model": model_name, "message": "Deep Learning!"}
    elif model_name == ModelName.lenet:
        return {"model": model_name, "message": "LeCNN"}
    return {"model": model_name, "message": "Residuals"}

# Valid: /models/alexnet, /models/resnet, /models/lenet
# Invalid: /models/vgg → 422 with allowed values

class Priority(int, Enum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3

@app.get("/tasks/{priority}")
async def get_tasks(priority: Priority):
    return {"priority": priority.name, "value": priority.value}
```

### Path Parameter Constraints

Use `Path` for validation constraints.

```python
from fastapi import FastAPI, Path

app = FastAPI()

# Numeric constraints
@app.get("/items/{item_id}")
async def read_item(
    item_id: int = Path(..., ge=1, le=1000, description="Item ID")
):
    return {"item_id": item_id}
# Valid: /items/1, /items/500, /items/1000
# Invalid: /items/0, /items/1001

# String constraints
@app.get("/users/{username}")
async def read_user(
    username: str = Path(
        ...,
        min_length=3,
        max_length=20,
        pattern="^[a-zA-Z0-9_]+$",
        example="john_doe"
    )
):
    return {"username": username}
# Valid: /users/alice, /users/john123
# Invalid: /users/ab (too short), /users/john-doe (hyphen)

# Float constraints
@app.get("/prices/{price}")
async def read_price(
    price: float = Path(..., gt=0, le=999999.99)
):
    return {"price": price}
```

### Multiple Path Parameters

```python
from fastapi import FastAPI, Path
from uuid import UUID

app = FastAPI()

@app.get("/users/{user_id}/posts/{post_id}/comments/{comment_id}")
async def read_comment(
    user_id: int = Path(..., ge=1),
    post_id: int = Path(..., ge=1),
    comment_id: int = Path(..., ge=1)
):
    return {
        "user_id": user_id,
        "post_id": post_id,
        "comment_id": comment_id
    }

@app.get("/orgs/{org_id}/projects/{project_id}")
async def get_project(
    org_id: UUID,
    project_id: str = Path(..., min_length=5)
):
    return {"org_id": org_id, "project_id": project_id}
```

### Path Parameters with Validation (gt, lt, ge, le)

```python
from fastapi import FastAPI, Path

app = FastAPI()

# Greater than (gt)
@app.get("/positive/{number}")
async def positive_number(
    number: int = Path(..., gt=0)
):
    return {"number": number}
# Valid: 1, 2, 3...
# Invalid: 0, -1

# Greater than or equal (ge)
@app.get("/pages/{page}")
async def read_page(
    page: int = Path(..., ge=1)
):
    return {"page": page}
# Valid: 1, 2, 3...
# Invalid: 0, -1

# Less than (lt)
@app.get("/percentage/{percent}")
async def read_percentage(
    percent: float = Path(..., lt=100)
):
    return {"percent": percent}
# Valid: 0, 50, 99.9
# Invalid: 100, 101

# Less than or equal (le)
@app.get("/score/{score}")
async def read_score(
    score: int = Path(..., le=100)
):
    return {"score": score}
# Valid: 0, 50, 100
# Invalid: 101

# Range (combining constraints)
@app.get("/age/{age}")
async def read_age(
    age: int = Path(..., ge=0, le=120)
):
    return {"age": age}
# Valid: 0-120
# Invalid: -1, 121

@app.get("/rating/{rating}")
async def read_rating(
    rating: float = Path(..., ge=0.0, le=5.0)
):
    return {"rating": rating}
# Valid: 0.0-5.0
```

---

## 2.3 Query Parameters

### Basic Query Parameters

Query parameters appear after `?` in the URL and are optional by default.

```python
from fastapi import FastAPI

app = FastAPI()

# Basic query parameters
@app.get("/items/")
async def read_items(skip: int = 0, limit: int = 10):
    return {"skip": skip, "limit": limit}

# URL: /items/ → skip=0, limit=10
# URL: /items/?skip=20 → skip=20, limit=10
# URL: /items/?skip=20&limit=50 → skip=20, limit=50

# String query parameters
@app.get("/search/")
async def search(q: str = ""):
    return {"query": q}

# URL: /search/ → q=""
# URL: /search/?q=python → q="python"
```

### Optional Query Parameters

```python
from fastapi import FastAPI
from typing import Optional

app = FastAPI()

@app.get("/items/{item_id}")
async def read_item(item_id: int, q: Optional[str] = None):
    if q:
        return {"item_id": item_id, "q": q}
    return {"item_id": item_id}

# URL: /items/1 → {"item_id": 1}
# URL: /items/1?q=test → {"item_id": 1, "q": "test"}

# Python 3.10+ syntax
@app.get("/products/{product_id}")
async def read_product(product_id: int, search: str | None = None):
    return {"product_id": product_id, "search": search}
```

### Query Parameters with Defaults

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/")
async def list_items(
    skip: int = 0,
    limit: int = 10,
    sort: str = "created",
    order: str = "desc"
):
    return {
        "skip": skip,
        "limit": limit,
        "sort": sort,
        "order": order
    }

# URL: /items/ → All defaults
# URL: /items/?limit=20 → limit=20, others default
# URL: /items/?sort=name&order=asc → custom sort
```

### Required Query Parameters

```python
from fastapi import FastAPI, Query

app = FastAPI()

# Method 1: No default value
@app.get("/items/")
async def read_items(q: str):
    return {"q": q}
# URL: /items/ → 422 Error (missing required parameter)
# URL: /items/?q=test → Success

# Method 2: Using Query with ...
@app.get("/search/")
async def search(q: str = Query(..., min_length=3)):
    return {"q": q}

# Method 3: Using Query with default
@app.get("/products/")
async def search_products(
    q: str = Query(..., description="Search query")
):
    return {"q": q}
```

### Multiple Values for Same Parameter (List)

```python
from fastapi import FastAPI, Query
from typing import List, Optional

app = FastAPI()

# List of strings
@app.get("/items/")
async def read_items(tags: List[str] = Query([])):
    return {"tags": tags}

# URL: /items/?tags=python&tags=fastapi&tags=api
# Result: {"tags": ["python", "fastapi", "api"]}

# Optional list
@app.get("/products/")
async def read_products(
    categories: Optional[List[str]] = Query(None)
):
    if categories:
        return {"categories": categories}
    return {"message": "No categories filter"}

# List with default values
@app.get("/search/")
async def search(
    tags: List[str] = Query(["python", "web"])
):
    return {"tags": tags}

# List of integers
@app.get("/filter/")
async def filter_items(ids: List[int] = Query([])):
    return {"ids": ids}
# URL: /filter/?ids=1&ids=2&ids=3
```

### Boolean Query Parameters

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/")
async def read_items(active: bool = True):
    return {"active": active}

# URL: /items/ → active=True (default)
# URL: /items/?active=false → active=False
# URL: /items/?active=0 → active=False
# URL: /items/?active=true → active=True
# URL: /items/?active=1 → active=True

@app.get("/users/")
async def list_users(
    include_inactive: bool = False,
    verified_only: bool = False
):
    return {
        "include_inactive": include_inactive,
        "verified_only": verified_only
    }
```

### Query Parameter Validation

```python
from fastapi import FastAPI, Query

app = FastAPI()

@app.get("/items/")
async def read_items(
    q: str = Query(
        None,
        min_length=3,
        max_length=50,
        pattern="^[a-zA-Z0-9 ]+$",
        description="Search query",
        example="python fastapi"
    ),
    skip: int = Query(0, ge=0, description="Items to skip"),
    limit: int = Query(10, ge=1, le=100, description="Max items"),
    price_min: float = Query(None, ge=0),
    price_max: float = Query(None, le=1000000)
):
    return {
        "q": q,
        "skip": skip,
        "limit": limit,
        "price_range": (price_min, price_max)
    }

# Comprehensive validation
@app.get("/search/")
async def search(
    query: str = Query(
        ...,
        title="Search Query",
        description="The search query string",
        min_length=1,
        max_length=100
    ),
    page: int = Query(1, ge=1, le=1000, description="Page number"),
    size: int = Query(20, ge=1, le=100, description="Page size"),
    sort: str = Query(
        "relevance",
        regex="^(relevance|date|price)$",
        description="Sort order"
    )
):
    return {
        "query": query,
        "page": page,
        "size": size,
        "sort": sort
    }
```

### Query Parameter Models with Pydantic

Group related query parameters into Pydantic models (FastAPI 0.115+).

```python
from fastapi import FastAPI, Query
from pydantic import BaseModel, Field
from typing import Optional

app = FastAPI()

class FilterParams(BaseModel):
    skip: int = Field(0, ge=0)
    limit: int = Field(10, ge=1, le=100)
    search: Optional[str] = Field(None, min_length=3, max_length=50)
    min_price: Optional[float] = Field(None, ge=0)
    max_price: Optional[float] = Field(None, le=1000000)
    in_stock: bool = True

@app.get("/items/")
async def list_items(filters: FilterParams = Query()):
    return {
        "filters": filters.dict(),
        "results": []
    }

# Advanced example
class PaginationParams(BaseModel):
    page: int = Field(1, ge=1, description="Page number")
    page_size: int = Field(20, ge=1, le=100, description="Items per page")

class SortParams(BaseModel):
    sort_by: str = Field("created_at", description="Field to sort by")
    order: str = Field("desc", regex="^(asc|desc)$")

@app.get("/products/")
async def list_products(
    pagination: PaginationParams = Query(),
    sorting: SortParams = Query()
):
    return {
        "pagination": pagination.dict(),
        "sorting": sorting.dict(),
        "products": []
    }
```

---

## 2.4 APIRouter

### Creating Routers

APIRouter allows you to organize endpoints into separate modules.

```python
# routers/items.py
from fastapi import APIRouter

router = APIRouter()

@router.get("/")
async def list_items():
    return [{"id": 1, "name": "Item 1"}]

@router.get("/{item_id}")
async def get_item(item_id: int):
    return {"id": item_id, "name": "Item"}

@router.post("/")
async def create_item(name: str):
    return {"id": 1, "name": name}
```

```python
# main.py
from fastapi import FastAPI
from routers import items

app = FastAPI()

# Include the router
app.include_router(items.router, prefix="/items", tags=["items"])
```

### Router Groups and Organization

**Project Structure**:
```
app/
├── main.py
├── routers/
│   ├── __init__.py
│   ├── users.py
│   ├── items.py
│   └── auth.py
```

**routers/users.py**:
```python
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel

router = APIRouter()

class User(BaseModel):
    username: str
    email: str

users_db = []

@router.get("/")
async def list_users():
    return users_db

@router.get("/{user_id}")
async def get_user(user_id: int):
    if user_id < len(users_db):
        return users_db[user_id]
    raise HTTPException(status_code=404, detail="User not found")

@router.post("/")
async def create_user(user: User):
    users_db.append(user)
    return user

@router.delete("/{user_id}")
async def delete_user(user_id: int):
    if user_id < len(users_db):
        deleted = users_db.pop(user_id)
        return {"message": "User deleted", "user": deleted}
    raise HTTPException(status_code=404, detail="User not found")
```

**routers/items.py**:
```python
from fastapi import APIRouter
from pydantic import BaseModel
from typing import Optional

router = APIRouter()

class Item(BaseModel):
    name: str
    price: float
    description: Optional[str] = None

items_db = []

@router.get("/")
async def list_items(skip: int = 0, limit: int = 10):
    return items_db[skip : skip + limit]

@router.post("/")
async def create_item(item: Item):
    items_db.append(item)
    return item
```

### Including Routers in Main App

```python
# main.py
from fastapi import FastAPI
from routers import users, items, auth

app = FastAPI(title="My API", version="1.0.0")

# Include routers
app.include_router(users.router, prefix="/users", tags=["users"])
app.include_router(items.router, prefix="/items", tags=["items"])
app.include_router(auth.router, prefix="/auth", tags=["authentication"])

@app.get("/")
async def root():
    return {"message": "Welcome to the API"}

# Now you have:
# /users/
# /users/{user_id}
# /items/
# /items/{item_id}
# /auth/login
# /auth/logout
```

### Router Prefixes

```python
from fastapi import FastAPI, APIRouter

# Create router without prefix
users_router = APIRouter()

@users_router.get("/")
async def list_users():
    return []

@users_router.get("/{user_id}")
async def get_user(user_id: int):
    return {"user_id": user_id}

app = FastAPI()

# Add prefix when including
app.include_router(users_router, prefix="/api/v1/users")

# Routes become:
# /api/v1/users/
# /api/v1/users/{user_id}

# Or set prefix in router
api_v2_router = APIRouter(prefix="/api/v2")

@api_v2_router.get("/users/")
async def list_users_v2():
    return []

app.include_router(api_v2_router)
# Routes: /api/v2/users/
```

### Router Tags

Tags organize endpoints in the documentation.

```python
from fastapi import FastAPI, APIRouter

# Router with tags
items_router = APIRouter(tags=["items"])

@items_router.get("/")
async def list_items():
    return []

@items_router.post("/")
async def create_item():
    return {}

# Router without tags
users_router = APIRouter()

@users_router.get("/", tags=["users", "public"])
async def list_users():
    return []

@users_router.get("/{user_id}", tags=["users"])
async def get_user(user_id: int):
    return {}

app = FastAPI()

# Include with additional tags
app.include_router(
    items_router,
    prefix="/items",
    tags=["inventory"]  # Adds to existing tags
)

app.include_router(
    users_router,
    prefix="/users"
)
```

### Router Dependencies

Apply dependencies to all routes in a router.

```python
from fastapi import APIRouter, Depends, HTTPException, Header

# Dependency function
async def verify_token(x_token: str = Header(...)):
    if x_token != "secret-token":
        raise HTTPException(status_code=401, detail="Invalid token")
    return x_token

async def verify_admin(x_admin: str = Header(...)):
    if x_admin != "admin-key":
        raise HTTPException(status_code=403, detail="Not admin")

# Router with dependencies
admin_router = APIRouter(
    prefix="/admin",
    tags=["admin"],
    dependencies=[Depends(verify_token), Depends(verify_admin)]
)

@admin_router.get("/users/")
async def list_all_users():
    """Requires both token and admin verification."""
    return {"users": []}

@admin_router.delete("/users/{user_id}")
async def delete_user(user_id: int):
    """Also requires both token and admin verification."""
    return {"deleted": user_id}

# Public router (no dependencies)
public_router = APIRouter(prefix="/public", tags=["public"])

@public_router.get("/items/")
async def list_public_items():
    """No authentication required."""
    return {"items": []}

app = FastAPI()
app.include_router(admin_router)
app.include_router(public_router)
```

### Nested Routers

Create hierarchical router structures.

```python
from fastapi import FastAPI, APIRouter

# Parent router
api_router = APIRouter(prefix="/api/v1")

# Child routers
users_router = APIRouter(prefix="/users", tags=["users"])
items_router = APIRouter(prefix="/items", tags=["items"])

@users_router.get("/")
async def list_users():
    return []

@users_router.get("/{user_id}")
async def get_user(user_id: int):
    return {"user_id": user_id}

@items_router.get("/")
async def list_items():
    return []

# Include child routers in parent
api_router.include_router(users_router)
api_router.include_router(items_router)

# Include parent in app
app = FastAPI()
app.include_router(api_router)

# Routes:
# /api/v1/users/
# /api/v1/users/{user_id}
# /api/v1/items/

# Complex nested structure
v1_router = APIRouter(prefix="/api/v1")
v2_router = APIRouter(prefix="/api/v2")

# V1 sub-routers
v1_users = APIRouter(prefix="/users", tags=["v1", "users"])
v1_items = APIRouter(prefix="/items", tags=["v1", "items"])

@v1_users.get("/")
async def list_users_v1():
    return {"version": "v1"}

v1_router.include_router(v1_users)
v1_router.include_router(v1_items)

# V2 sub-routers
v2_users = APIRouter(prefix="/users", tags=["v2", "users"])

@v2_users.get("/")
async def list_users_v2():
    return {"version": "v2"}

v2_router.include_router(v2_users)

app = FastAPI()
app.include_router(v1_router)
app.include_router(v2_router)

# Routes:
# /api/v1/users/
# /api/v1/items/
# /api/v2/users/
```

---

## Complete Example: E-commerce API with Routers

**Project Structure**:
```
ecommerce_api/
├── main.py
├── dependencies.py
└── routers/
    ├── __init__.py
    ├── products.py
    ├── users.py
    └── orders.py
```

**dependencies.py**:
```python
from fastapi import Header, HTTPException

async def verify_api_key(x_api_key: str = Header(...)):
    if x_api_key != "secret-api-key":
        raise HTTPException(status_code=401, detail="Invalid API key")
    return x_api_key

async def get_current_user(x_user_id: int = Header(...)):
    # Simulate user lookup
    return {"id": x_user_id, "username": "user"}
```

**routers/products.py**:
```python
from fastapi import APIRouter, Query, Path
from pydantic import BaseModel, Field
from typing import Optional, List
from enum import Enum

router = APIRouter()

class Category(str, Enum):
    ELECTRONICS = "electronics"
    CLOTHING = "clothing"
    BOOKS = "books"

class Product(BaseModel):
    name: str = Field(..., min_length=1)
    price: float = Field(..., gt=0)
    category: Category
    description: Optional[str] = None
    in_stock: bool = True

products_db = []

@router.get("/", response_model=List[Product])
async def list_products(
    skip: int = Query(0, ge=0),
    limit: int = Query(10, ge=1, le=100),
    category: Optional[Category] = None,
    in_stock: Optional[bool] = None
):
    """List all products with filters."""
    filtered = products_db
    if category:
        filtered = [p for p in filtered if p.category == category]
    if in_stock is not None:
        filtered = [p for p in filtered if p.in_stock == in_stock]
    return filtered[skip : skip + limit]

@router.get("/{product_id}", response_model=Product)
async def get_product(
    product_id: int = Path(..., ge=1)
):
    """Get a specific product."""
    if product_id <= len(products_db):
        return products_db[product_id - 1]
    return {"error": "Product not found"}

@router.post("/", response_model=Product)
async def create_product(product: Product):
    """Create a new product."""
    products_db.append(product)
    return product
```

**routers/users.py**:
```python
from fastapi import APIRouter, Depends
from pydantic import BaseModel, EmailStr
from dependencies import verify_api_key, get_current_user

router = APIRouter(dependencies=[Depends(verify_api_key)])

class User(BaseModel):
    username: str
    email: EmailStr
    full_name: str

users_db = []

@router.get("/me")
async def read_current_user(current_user: dict = Depends(get_current_user)):
    """Get current user profile."""
    return current_user

@router.post("/")
async def create_user(user: User):
    """Create a new user."""
    users_db.append(user)
    return user

@router.get("/{user_id}")
async def get_user(user_id: int):
    """Get user by ID."""
    if user_id <= len(users_db):
        return users_db[user_id - 1]
    return {"error": "User not found"}
```

**routers/orders.py**:
```python
from fastapi import APIRouter, Depends
from pydantic import BaseModel
from typing import List
from dependencies import verify_api_key

router = APIRouter(dependencies=[Depends(verify_api_key)])

class OrderItem(BaseModel):
    product_id: int
    quantity: int

class Order(BaseModel):
    user_id: int
    items: List[OrderItem]
    total: float

orders_db = []

@router.post("/")
async def create_order(order: Order):
    """Create a new order."""
    orders_db.append(order)
    return {"order_id": len(orders_db), **order.dict()}

@router.get("/{order_id}")
async def get_order(order_id: int):
    """Get order by ID."""
    if order_id <= len(orders_db):
        return orders_db[order_id - 1]
    return {"error": "Order not found"}
```

**main.py**:
```python
from fastapi import FastAPI
from routers import products, users, orders

app = FastAPI(
    title="E-commerce API",
    description="A complete e-commerce API with routers",
    version="1.0.0"
)

# API v1
api_v1 = APIRouter(prefix="/api/v1")

api_v1.include_router(
    products.router,
    prefix="/products",
    tags=["products"]
)

api_v1.include_router(
    users.router,
    prefix="/users",
    tags=["users"]
)

api_v1.include_router(
    orders.router,
    prefix="/orders",
    tags=["orders"]
)

app.include_router(api_v1)

@app.get("/")
async def root():
    return {
        "message": "E-commerce API",
        "docs": "/docs",
        "version": "1.0.0"
    }

# Routes:
# /api/v1/products/
# /api/v1/products/{product_id}
# /api/v1/users/me
# /api/v1/users/{user_id}
# /api/v1/orders/
# /api/v1/orders/{order_id}
```

---

## Summary

You've learned about Routing & Path Operations in FastAPI:

1. **Basic Routing**: HTTP methods (GET, POST, PUT, PATCH, DELETE), creating endpoints, operation order, and deprecation
2. **Path Parameters**: Type conversion, validation, enums, constraints (gt, ge, lt, le), and multiple parameters
3. **Query Parameters**: Optional and required parameters, defaults, lists, validation, and Pydantic models
4. **APIRouter**: Organizing code with routers, prefixes, tags, dependencies, and nested structures

These concepts enable you to build well-organized, scalable APIs with clean separation of concerns and automatic validation!