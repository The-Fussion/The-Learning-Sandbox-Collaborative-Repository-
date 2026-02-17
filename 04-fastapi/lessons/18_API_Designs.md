# API Design & Best Practices

## 18.1 RESTful API Design

### REST Principles

REST (Representational State Transfer) is an architectural style built on six
core constraints that make APIs scalable, maintainable, and predictable.

**The Six REST Constraints**:

1. **Client-Server**: UI and data storage are separated
2. **Stateless**: Each request contains all information needed — no server-side sessions
3. **Cacheable**: Responses must define themselves as cacheable or non-cacheable
4. **Uniform Interface**: Consistent resource identification, manipulation through representations
5. **Layered System**: Client can't tell if it's connected directly to the server
6. **Code on Demand** *(optional)*: Servers can send executable code to clients

```python
from fastapi import FastAPI

app = FastAPI()

# ✅ Stateless: Every request is self-contained
@app.get("/users/{user_id}/orders")
async def get_user_orders(
    user_id: int,
    authorization: str = Header(...)  # Auth included in request
):
    # No server session lookup needed
    user = verify_and_get_user(authorization)
    return get_orders_for_user(user_id)

# ❌ Stateful: Relies on server-side session
@app.get("/my-orders")  # Who is "my"? Server has to remember!
async def get_my_orders(session_id: str):
    user = session_store[session_id]  # Server-side state
    return get_orders_for_user(user.id)
```

### Resource Naming Conventions

Resources are the nouns of your API. URLs should identify **things**, not
actions.

**Rules**:
- Use **nouns**, not verbs
- Use **plural** for collections
- Use **lowercase** with hyphens for multi-word resources
- Nest resources to show relationships (max 2 levels deep)

```python
# ✅ GOOD - Noun-based, plural, lowercase
GET    /users                  # Collection
GET    /users/{id}             # Single resource
POST   /users                  # Create in collection
GET    /users/{id}/orders      # Nested relationship
GET    /blog-posts             # Hyphen for multi-word

# ❌ BAD - Verb-based, singular, mixed case
GET    /getUser
GET    /User/{id}
POST   /createUser
GET    /users/{id}/getOrders
GET    /blogPosts              # camelCase in URL

# Resource hierarchy examples
GET    /organizations/{org_id}/teams/{team_id}/members
GET    /products/{id}/reviews
GET    /articles/{id}/comments/{comment_id}

# ✅ Avoid going deeper than 2 levels
# Deep: /users/{id}/orders/{id}/items/{id}/reviews  ← Too deep
# Better: /order-items/{id}/reviews                 ← Flatten it
```

**Query Strings for Non-Resource Operations**:
```python
from fastapi import FastAPI, Query
from typing import Optional

app = FastAPI()

# Filtering, searching, sorting — not part of the path
@app.get("/products")
async def list_products(
    category: Optional[str] = Query(None),
    min_price: Optional[float] = Query(None),
    max_price: Optional[float] = Query(None),
    sort_by: str = Query("created_at"),
    order: str = Query("desc"),
    q: Optional[str] = Query(None, description="Search query"),
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100)
):
    # URL: /products?category=electronics&min_price=100&sort_by=price
    pass

# Actions that don't fit CRUD — use a sub-resource noun
POST /users/{id}/activation      # ✅ Not: POST /activateUser/{id}
POST /orders/{id}/cancellation   # ✅ Not: POST /cancelOrder/{id}
POST /payments/{id}/refunds      # ✅ Not: POST /refundPayment/{id}
```

### HTTP Method Usage

Each HTTP method has a specific, well-defined purpose. Misusing them
creates unpredictable, hard-to-use APIs.

```python
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

class Product(BaseModel):
    name: str
    price: float
    category: str
    description: Optional[str] = None

class ProductUpdate(BaseModel):
    name: Optional[str] = None
    price: Optional[float] = None
    category: Optional[str] = None
    description: Optional[str] = None

# GET — Read only, safe, idempotent, cacheable
@app.get("/products", summary="List all products")
async def list_products():
    return []

@app.get("/products/{product_id}", summary="Get single product")
async def get_product(product_id: int):
    return {}

# POST — Create a new resource, NOT idempotent
@app.post(
    "/products",
    status_code=status.HTTP_201_CREATED,
    summary="Create a product"
)
async def create_product(product: Product):
    return {"id": 1, **product.dict()}

# PUT — Full replacement, idempotent
# Same request twice = same result
@app.put("/products/{product_id}", summary="Replace a product")
async def replace_product(product_id: int, product: Product):
    # Must provide ALL fields — this is a full replacement
    return {"id": product_id, **product.dict()}

# PATCH — Partial update, only provided fields change
@app.patch("/products/{product_id}", summary="Update product fields")
async def update_product(product_id: int, update: ProductUpdate):
    # Only update what was provided
    changes = update.dict(exclude_unset=True)
    return {"id": product_id, "updated_fields": list(changes.keys())}

# DELETE — Remove resource, idempotent
@app.delete(
    "/products/{product_id}",
    status_code=status.HTTP_204_NO_CONTENT,
    summary="Delete a product"
)
async def delete_product(product_id: int):
    return None  # 204 returns no body

# HEAD — Like GET but no body (check existence/headers)
# OPTIONS — What methods does this endpoint support?
# Both are handled automatically by FastAPI/Starlette
```

### Status Code Usage

Status codes communicate the outcome of a request. Using the right ones
makes your API self-documenting.

```python
from fastapi import FastAPI, HTTPException, status
from fastapi.responses import JSONResponse

app = FastAPI()

# ── 2xx Success ──────────────────────────────────────────────
# 200 OK — General success for GET, PUT, PATCH
# 201 Created — Resource was created (POST)
# 202 Accepted — Request accepted, processing async
# 204 No Content — Success with no body (DELETE)

@app.post("/users", status_code=status.HTTP_201_CREATED)
async def create_user(data: dict):
    return {"id": 1, **data}

@app.delete("/users/{id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(user_id: int):
    return None

@app.post("/reports/generate", status_code=status.HTTP_202_ACCEPTED)
async def generate_report(params: dict):
    # Long operation — accept and process async
    return {"job_id": "abc123", "status": "processing"}

# ── 3xx Redirection ──────────────────────────────────────────
from fastapi.responses import RedirectResponse

@app.get("/old-endpoint")
async def old_endpoint():
    return RedirectResponse(url="/new-endpoint", status_code=301)

# ── 4xx Client Errors ────────────────────────────────────────
# 400 Bad Request — Invalid input
# 401 Unauthorized — Not authenticated
# 403 Forbidden — Authenticated but not authorized
# 404 Not Found — Resource doesn't exist
# 405 Method Not Allowed — Wrong HTTP method
# 409 Conflict — Resource state conflict
# 422 Unprocessable Entity — Validation failed (FastAPI default)
# 429 Too Many Requests — Rate limit hit

@app.get("/items/{item_id}")
async def get_item(item_id: int, user=Depends(get_current_user)):
    item = find_item(item_id)
    if not item:
        raise HTTPException(status_code=404, detail="Item not found")
    if item.owner_id != user.id:
        raise HTTPException(status_code=403, detail="Access denied")
    return item

# ── 5xx Server Errors ────────────────────────────────────────
# 500 Internal Server Error — Unexpected server failure
# 502 Bad Gateway — Upstream server error
# 503 Service Unavailable — Server overloaded or down
# 504 Gateway Timeout — Upstream timed out

@app.get("/health")
async def health_check():
    try:
        await db.execute("SELECT 1")
        return {"status": "healthy"}
    except Exception:
        return JSONResponse(
            status_code=503,
            content={"status": "unhealthy", "reason": "database"}
        )
```

### API Versioning Strategies

```python
# Overview of strategies — covered in depth in Section 18.4

# URL Versioning (most common)
GET /api/v1/users
GET /api/v2/users

# Header Versioning
GET /api/users
Headers: API-Version: 2

# Query Parameter
GET /api/users?version=2

# Content Negotiation
GET /api/users
Accept: application/vnd.myapi.v2+json
```

### HATEOAS

HATEOAS (Hypermedia as the Engine of Application State) enriches responses
with links to related actions, making APIs self-discoverable.

```python
from fastapi import FastAPI
from pydantic import BaseModel
from typing import List, Dict

app = FastAPI()

class Link(BaseModel):
    href: str
    method: str
    rel: str

class HateoasResponse(BaseModel):
    data: dict
    links: List[Link]

@app.get("/orders/{order_id}")
async def get_order(order_id: int):
    order = {
        "id": order_id,
        "status": "pending",
        "total": 59.99
    }

    # Links tell client what they can do next
    links = [
        {"href": f"/orders/{order_id}", "method": "GET", "rel": "self"},
        {"href": f"/orders/{order_id}", "method": "DELETE", "rel": "cancel"},
        {"href": f"/orders/{order_id}/payment", "method": "POST", "rel": "pay"},
        {"href": f"/users/{order['user_id']}", "method": "GET", "rel": "customer"},
    ]

    return {"data": order, "links": links}

# Response:
# {
#   "data": {"id": 1, "status": "pending"},
#   "links": [
#     {"href": "/orders/1", "method": "GET", "rel": "self"},
#     {"href": "/orders/1/payment", "method": "POST", "rel": "pay"}
#   ]
# }
```

---

## 18.2 Project Structure

### Modular Architecture

A well-structured FastAPI project separates concerns cleanly and scales as
the application grows.

```
# Small project
my_api/
├── main.py
├── models.py
├── schemas.py
└── database.py

# Medium project
my_api/
├── main.py
├── database.py
├── dependencies.py
├── routers/
│   ├── __init__.py
│   ├── users.py
│   └── products.py
├── models/
│   ├── __init__.py
│   ├── user.py
│   └── product.py
└── schemas/
    ├── __init__.py
    ├── user.py
    └── product.py

# Large / production project
my_api/
├── main.py
├── config.py
├── database.py
├── dependencies.py
│
├── api/                        # All HTTP layer
│   ├── __init__.py
│   ├── v1/
│   │   ├── __init__.py
│   │   ├── router.py           # Aggregates all v1 routers
│   │   └── endpoints/
│   │       ├── users.py
│   │       ├── products.py
│   │       └── orders.py
│   └── v2/
│       └── ...
│
├── core/                       # Business logic
│   ├── __init__.py
│   ├── security.py
│   ├── exceptions.py
│   └── events.py
│
├── services/                   # Use-case layer
│   ├── __init__.py
│   ├── user_service.py
│   ├── product_service.py
│   └── order_service.py
│
├── repositories/               # Data access layer
│   ├── __init__.py
│   ├── base.py
│   ├── user_repository.py
│   └── product_repository.py
│
├── models/                     # SQLAlchemy ORM models
│   ├── __init__.py
│   ├── user.py
│   └── product.py
│
├── schemas/                    # Pydantic I/O schemas
│   ├── __init__.py
│   ├── user.py
│   └── product.py
│
└── tests/
    ├── conftest.py
    ├── test_users.py
    └── test_products.py
```

### Separation of Concerns

Each layer has one job and does it well.

```python
# ── Layer responsibilities ───────────────────────────────────

# api/v1/endpoints/users.py  → HTTP only (routing, request/response)
# services/user_service.py   → Business logic
# repositories/user_repo.py  → Database queries
# models/user.py             → ORM table definition
# schemas/user.py            → Request/response shape

# This means:
# - Changing your database? Only touch repositories
# - Adding a new business rule? Only touch services
# - Adding a new endpoint? Only touch api layer
# - Changing API shape? Only touch schemas
```

### Routers Organization

```python
# api/v1/endpoints/users.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession

from dependencies import get_db, get_current_user
from services.user_service import UserService
from schemas.user import UserCreate, UserResponse, UserUpdate

router = APIRouter()

@router.get("/", response_model=list[UserResponse])
async def list_users(
    db: AsyncSession = Depends(get_db),
    _: dict = Depends(get_current_user)
):
    service = UserService(db)
    return await service.get_all()

@router.post("/", response_model=UserResponse, status_code=201)
async def create_user(
    data: UserCreate,
    db: AsyncSession = Depends(get_db)
):
    service = UserService(db)
    return await service.create(data)

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    service = UserService(db)
    user = await service.get_by_id(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

# api/v1/router.py — aggregates all routers for v1
from fastapi import APIRouter
from .endpoints import users, products, orders

api_router = APIRouter()
api_router.include_router(users.router, prefix="/users", tags=["Users"])
api_router.include_router(products.router, prefix="/products", tags=["Products"])
api_router.include_router(orders.router, prefix="/orders", tags=["Orders"])

# main.py — clean and minimal
from fastapi import FastAPI
from api.v1.router import api_router

app = FastAPI(title="My API")
app.include_router(api_router, prefix="/api/v1")
```

### Models Organization

```python
# models/base.py — shared columns every model needs
from sqlalchemy import Column, Integer, DateTime, func
from database import Base

class TimestampMixin:
    created_at = Column(DateTime, server_default=func.now(), nullable=False)
    updated_at = Column(DateTime, onupdate=func.now())

class BaseModel(TimestampMixin, Base):
    __abstract__ = True
    id = Column(Integer, primary_key=True, index=True)

# models/user.py
from sqlalchemy import Column, String, Boolean
from sqlalchemy.orm import relationship
from .base import BaseModel

class User(BaseModel):
    __tablename__ = "users"

    username  = Column(String(50), unique=True, index=True, nullable=False)
    email     = Column(String(255), unique=True, index=True, nullable=False)
    password  = Column(String(255), nullable=False)
    is_active = Column(Boolean, default=True)
    is_admin  = Column(Boolean, default=False)

    orders  = relationship("Order", back_populates="user")
    profile = relationship("UserProfile", uselist=False, back_populates="user")

# models/__init__.py — import all models here so Alembic finds them
from .user import User
from .product import Product
from .order import Order
```

### Schemas Organization

```python
# schemas/user.py — separate schema for each operation
from pydantic import BaseModel, EmailStr, Field
from datetime import datetime
from typing import Optional

# Base — shared fields
class UserBase(BaseModel):
    username: str = Field(..., min_length=3, max_length=50)
    email: EmailStr

# Create — what the client sends on POST
class UserCreate(UserBase):
    password: str = Field(..., min_length=8)

# Update — all optional for PATCH
class UserUpdate(BaseModel):
    username: Optional[str] = Field(None, min_length=3, max_length=50)
    email: Optional[EmailStr] = None

# Response — what the API returns (no password!)
class UserResponse(UserBase):
    id: int
    is_active: bool
    created_at: datetime

    class Config:
        from_attributes = True  # Pydantic v2 (was orm_mode in v1)

# Nested — used inside other schemas
class UserSummary(BaseModel):
    id: int
    username: str

    class Config:
        from_attributes = True
```

### Services Layer

```python
# services/user_service.py — all business logic lives here
from fastapi import HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession

from repositories.user_repository import UserRepository
from schemas.user import UserCreate, UserUpdate
from core.security import hash_password, verify_password

class UserService:
    def __init__(self, db: AsyncSession):
        self.repo = UserRepository(db)

    async def create(self, data: UserCreate):
        # Business rule: usernames must be unique
        existing = await self.repo.get_by_email(data.email)
        if existing:
            raise HTTPException(
                status_code=status.HTTP_409_CONFLICT,
                detail="Email already registered"
            )

        # Business rule: hash password before storing
        hashed = hash_password(data.password)
        return await self.repo.create({**data.dict(), "password": hashed})

    async def get_by_id(self, user_id: int):
        return await self.repo.get_by_id(user_id)

    async def get_all(self, skip: int = 0, limit: int = 100):
        return await self.repo.get_all(skip=skip, limit=limit)

    async def update(self, user_id: int, data: UserUpdate):
        user = await self.repo.get_by_id(user_id)
        if not user:
            raise HTTPException(status_code=404, detail="User not found")

        changes = data.dict(exclude_unset=True)
        return await self.repo.update(user_id, changes)

    async def delete(self, user_id: int):
        user = await self.repo.get_by_id(user_id)
        if not user:
            raise HTTPException(status_code=404, detail="User not found")

        # Business rule: can't delete admin users
        if user.is_admin:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Cannot delete admin users"
            )
        await self.repo.delete(user_id)
```

### Repository Pattern

```python
# repositories/base.py — generic CRUD for any model
from typing import TypeVar, Generic, Type, Optional, List
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

ModelType = TypeVar("ModelType")

class BaseRepository(Generic[ModelType]):
    def __init__(self, model: Type[ModelType], db: AsyncSession):
        self.model = model
        self.db = db

    async def get_by_id(self, id: int) -> Optional[ModelType]:
        result = await self.db.execute(
            select(self.model).where(self.model.id == id)
        )
        return result.scalar_one_or_none()

    async def get_all(self, skip: int = 0, limit: int = 100) -> List[ModelType]:
        result = await self.db.execute(
            select(self.model).offset(skip).limit(limit)
        )
        return result.scalars().all()

    async def create(self, data: dict) -> ModelType:
        instance = self.model(**data)
        self.db.add(instance)
        await self.db.commit()
        await self.db.refresh(instance)
        return instance

    async def update(self, id: int, data: dict) -> Optional[ModelType]:
        instance = await self.get_by_id(id)
        if not instance:
            return None
        for key, value in data.items():
            setattr(instance, key, value)
        await self.db.commit()
        await self.db.refresh(instance)
        return instance

    async def delete(self, id: int) -> bool:
        instance = await self.get_by_id(id)
        if not instance:
            return False
        await self.db.delete(instance)
        await self.db.commit()
        return True

# repositories/user_repository.py — model-specific queries
from models.user import User
from .base import BaseRepository

class UserRepository(BaseRepository[User]):
    def __init__(self, db):
        super().__init__(User, db)

    async def get_by_email(self, email: str) -> Optional[User]:
        result = await self.db.execute(
            select(User).where(User.email == email)
        )
        return result.scalar_one_or_none()

    async def get_active_users(self) -> list[User]:
        result = await self.db.execute(
            select(User).where(User.is_active == True)
        )
        return result.scalars().all()
```

---

## 18.3 Error Handling Patterns

### Consistent Error Responses

Every error your API returns should have the same predictable shape. This
makes it trivial for clients to handle errors regardless of what went wrong.

```python
# core/exceptions.py — define the error shape
from pydantic import BaseModel
from typing import Optional, List, Any

class ErrorDetail(BaseModel):
    field: Optional[str] = None   # Which field caused the error
    message: str                   # Human-readable message
    code: str                      # Machine-readable error code

class ErrorResponse(BaseModel):
    status: str = "error"
    message: str                   # Top-level message
    errors: Optional[List[ErrorDetail]] = None
    request_id: Optional[str] = None

# Every error looks like this:
# {
#   "status": "error",
#   "message": "Validation failed",
#   "errors": [
#     {"field": "email", "message": "Invalid email format", "code": "INVALID_FORMAT"},
#     {"field": "password", "message": "Too short", "code": "TOO_SHORT"}
#   ],
#   "request_id": "abc-123"
# }
```

**Global Exception Handlers**:
```python
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from sqlalchemy.exc import IntegrityError
import uuid

app = FastAPI()

@app.middleware("http")
async def add_request_id(request: Request, call_next):
    request.state.request_id = str(uuid.uuid4())
    response = await call_next(request)
    response.headers["X-Request-ID"] = request.state.request_id
    return response

# Handle FastAPI/Pydantic validation errors
@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    errors = []
    for error in exc.errors():
        field = ".".join(str(loc) for loc in error["loc"] if loc != "body")
        errors.append({
            "field": field or None,
            "message": error["msg"],
            "code": error["type"].upper()
        })

    return JSONResponse(
        status_code=422,
        content={
            "status": "error",
            "message": "Validation failed",
            "errors": errors,
            "request_id": getattr(request.state, "request_id", None)
        }
    )

# Handle HTTPException with consistent shape
@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "status": "error",
            "message": exc.detail,
            "request_id": getattr(request.state, "request_id", None)
        }
    )

# Handle database integrity errors
@app.exception_handler(IntegrityError)
async def integrity_error_handler(request: Request, exc: IntegrityError):
    return JSONResponse(
        status_code=409,
        content={
            "status": "error",
            "message": "Resource already exists",
            "code": "DUPLICATE_ENTRY",
            "request_id": getattr(request.state, "request_id", None)
        }
    )

# Catch-all for unexpected errors
@app.exception_handler(Exception)
async def unhandled_exception_handler(request: Request, exc: Exception):
    # Log the real error internally
    logger.exception(f"Unhandled error: {exc}")

    return JSONResponse(
        status_code=500,
        content={
            "status": "error",
            "message": "An unexpected error occurred",
            "request_id": getattr(request.state, "request_id", None)
            # Never expose internal details in production
        }
    )
```

### Error Codes

Machine-readable error codes let clients handle specific errors
programmatically without parsing English error messages.

```python
# core/error_codes.py
from enum import Enum

class ErrorCode(str, Enum):
    # Authentication
    INVALID_TOKEN       = "INVALID_TOKEN"
    TOKEN_EXPIRED       = "TOKEN_EXPIRED"
    NOT_AUTHENTICATED   = "NOT_AUTHENTICATED"

    # Authorization
    PERMISSION_DENIED   = "PERMISSION_DENIED"
    ACCOUNT_SUSPENDED   = "ACCOUNT_SUSPENDED"

    # Resource
    NOT_FOUND           = "NOT_FOUND"
    ALREADY_EXISTS      = "ALREADY_EXISTS"
    CONFLICT            = "CONFLICT"

    # Validation
    INVALID_INPUT       = "INVALID_INPUT"
    INVALID_FORMAT      = "INVALID_FORMAT"
    MISSING_FIELD       = "MISSING_FIELD"
    OUT_OF_RANGE        = "OUT_OF_RANGE"

    # Business logic
    INSUFFICIENT_FUNDS  = "INSUFFICIENT_FUNDS"
    ORDER_NOT_EDITABLE  = "ORDER_NOT_EDITABLE"
    QUOTA_EXCEEDED      = "QUOTA_EXCEEDED"

    # System
    SERVICE_UNAVAILABLE = "SERVICE_UNAVAILABLE"
    RATE_LIMIT_EXCEEDED = "RATE_LIMIT_EXCEEDED"

# core/exceptions.py — custom exception classes
from fastapi import HTTPException

class AppException(HTTPException):
    def __init__(self, status_code: int, code: ErrorCode, message: str):
        super().__init__(status_code=status_code, detail={
            "code": code,
            "message": message
        })

class NotFoundException(AppException):
    def __init__(self, resource: str, id: any):
        super().__init__(404, ErrorCode.NOT_FOUND, f"{resource} {id} not found")

class ConflictException(AppException):
    def __init__(self, message: str):
        super().__init__(409, ErrorCode.ALREADY_EXISTS, message)

class ForbiddenException(AppException):
    def __init__(self, message: str = "Access denied"):
        super().__init__(403, ErrorCode.PERMISSION_DENIED, message)

# Usage in services
async def get_order(order_id: int):
    order = await repo.get(order_id)
    if not order:
        raise NotFoundException("Order", order_id)  # Consistent, typed
    return order
```

### Validation Errors

```python
from pydantic import BaseModel, Field, field_validator, model_validator
from fastapi import FastAPI
from typing import Optional

app = FastAPI()

class OrderCreate(BaseModel):
    items: list[dict] = Field(..., min_length=1)
    delivery_date: str
    promo_code: Optional[str] = None
    payment_method: str

    @field_validator("payment_method")
    @classmethod
    def validate_payment_method(cls, v):
        allowed = ["credit_card", "paypal", "bank_transfer"]
        if v not in allowed:
            raise ValueError(f"Must be one of: {', '.join(allowed)}")
        return v

    @field_validator("delivery_date")
    @classmethod
    def validate_delivery_date(cls, v):
        from datetime import date
        try:
            d = date.fromisoformat(v)
        except ValueError:
            raise ValueError("Must be YYYY-MM-DD format")
        if d <= date.today():
            raise ValueError("Must be a future date")
        return v

    @model_validator(mode="after")
    def validate_promo_with_payment(self):
        if self.promo_code and self.payment_method == "bank_transfer":
            raise ValueError("Promo codes cannot be used with bank transfer")
        return self

# Validation errors return a clear, structured 422:
# {
#   "status": "error",
#   "message": "Validation failed",
#   "errors": [
#     {"field": "payment_method", "message": "Must be one of: credit_card, paypal, bank_transfer"},
#     {"field": "delivery_date", "message": "Must be a future date"}
#   ]
# }
```

### Business Logic Errors

```python
# services/order_service.py
from core.exceptions import AppException
from core.error_codes import ErrorCode
from fastapi import status

class OrderService:
    async def place_order(self, user_id: int, order_data: dict):
        user = await self.user_repo.get(user_id)

        # Business rule: user must have verified email
        if not user.email_verified:
            raise AppException(
                status_code=status.HTTP_403_FORBIDDEN,
                code=ErrorCode.PERMISSION_DENIED,
                message="Email verification required before placing orders"
            )

        # Business rule: check inventory
        for item in order_data["items"]:
            product = await self.product_repo.get(item["product_id"])
            if product.stock < item["quantity"]:
                raise AppException(
                    status_code=status.HTTP_409_CONFLICT,
                    code=ErrorCode.CONFLICT,
                    message=f"Insufficient stock for '{product.name}'"
                )

        # Business rule: check wallet balance
        total = sum(i["price"] * i["quantity"] for i in order_data["items"])
        if user.wallet_balance < total:
            raise AppException(
                status_code=status.HTTP_402_PAYMENT_REQUIRED,
                code=ErrorCode.INSUFFICIENT_FUNDS,
                message="Insufficient wallet balance"
            )

        return await self.order_repo.create(order_data)
```

---

## 18.4 API Versioning

### URL Versioning

The most common and visible strategy. Version is part of the URL path.

```python
from fastapi import FastAPI, APIRouter

app = FastAPI(title="My API")

# ── V1 Router ───────────────────────────────────────────────
v1_router = APIRouter(prefix="/api/v1")

@v1_router.get("/users")
async def list_users_v1():
    return {"version": "v1", "users": [{"id": 1, "name": "Alice"}]}

@v1_router.get("/users/{user_id}")
async def get_user_v1(user_id: int):
    return {"id": user_id, "name": "Alice", "email": "alice@example.com"}

# ── V2 Router — new fields, breaking changes ─────────────────
v2_router = APIRouter(prefix="/api/v2")

@v2_router.get("/users")
async def list_users_v2():
    # V2 adds pagination and new fields
    return {
        "version": "v2",
        "data": [{"id": 1, "username": "alice", "full_name": "Alice Smith"}],
        "meta": {"total": 1, "page": 1}
    }

@v2_router.get("/users/{user_id}")
async def get_user_v2(user_id: int):
    # V2 renames 'name' to 'full_name', adds 'username'
    return {
        "id": user_id,
        "username": "alice",
        "full_name": "Alice Smith",
        "avatar_url": "https://cdn.example.com/alice.jpg"
    }

app.include_router(v1_router, tags=["V1"])
app.include_router(v2_router, tags=["V2"])
```

**Structured folder versioning**:
```python
# api/v1/router.py
from fastapi import APIRouter
from .endpoints import users, products

router = APIRouter()
router.include_router(users.router, prefix="/users")
router.include_router(products.router, prefix="/products")

# api/v2/router.py
from fastapi import APIRouter
from .endpoints import users, products, analytics  # New in V2

router = APIRouter()
router.include_router(users.router, prefix="/users")
router.include_router(products.router, prefix="/products")
router.include_router(analytics.router, prefix="/analytics")

# main.py
from api.v1.router import router as v1_router
from api.v2.router import router as v2_router

app.include_router(v1_router, prefix="/api/v1")
app.include_router(v2_router, prefix="/api/v2")
```

### Header Versioning

Version is passed in a custom request header — URLs stay clean.

```python
from fastapi import FastAPI, Header, HTTPException
from typing import Optional

app = FastAPI()

@app.get("/users")
async def list_users(api_version: Optional[str] = Header(None, alias="API-Version")):
    version = api_version or "1"

    if version == "1":
        return [{"id": 1, "name": "Alice"}]
    elif version == "2":
        return {
            "data": [{"id": 1, "username": "alice", "full_name": "Alice Smith"}],
            "meta": {"total": 1}
        }
    else:
        raise HTTPException(status_code=400, detail=f"Unsupported API version: {version}")

# Middleware approach for header versioning
from starlette.middleware.base import BaseHTTPMiddleware

class VersionMiddleware(BaseHTTPMiddleware):
    SUPPORTED_VERSIONS = {"1", "2"}
    DEFAULT_VERSION = "1"

    async def dispatch(self, request, call_next):
        version = request.headers.get("API-Version", self.DEFAULT_VERSION)

        if version not in self.SUPPORTED_VERSIONS:
            return JSONResponse(
                status_code=400,
                content={"error": f"Unsupported version: {version}. Use: {self.SUPPORTED_VERSIONS}"}
            )

        request.state.api_version = version
        response = await call_next(request)
        response.headers["API-Version"] = version
        return response

app.add_middleware(VersionMiddleware)
```

### Query Parameter Versioning

```python
from fastapi import FastAPI, Query
from typing import Literal

app = FastAPI()

@app.get("/users")
async def list_users(
    version: Literal["1", "2"] = Query("1", description="API version")
):
    if version == "1":
        return [{"id": 1, "name": "Alice"}]
    return {
        "data": [{"id": 1, "username": "alice"}],
        "meta": {"total": 1}
    }

# GET /users         → v1 (default)
# GET /users?version=2 → v2
```

### Content Negotiation

Clients declare which version they want using `Accept` headers — the most
HTTP-correct approach.

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
import re

app = FastAPI()

def parse_version_from_accept(accept: str) -> str:
    """Parse: application/vnd.myapi.v2+json → '2'"""
    match = re.search(r"vnd\.myapi\.v(\d+)\+json", accept or "")
    return match.group(1) if match else "1"

@app.get("/users")
async def list_users(request: Request):
    accept = request.headers.get("Accept", "")
    version = parse_version_from_accept(accept)

    if version == "2":
        data = {"data": [{"id": 1, "username": "alice"}], "meta": {"total": 1}}
        media_type = "application/vnd.myapi.v2+json"
    else:
        data = [{"id": 1, "name": "Alice"}]
        media_type = "application/vnd.myapi.v1+json"

    return JSONResponse(content=data, media_type=media_type)

# Requests:
# Accept: application/vnd.myapi.v2+json  → V2 response
# Accept: application/json               → V1 response (default)
```

### Deprecation Strategies

Deprecating an old version gracefully so clients have time to migrate.

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from starlette.middleware.base import BaseHTTPMiddleware
from datetime import date

app = FastAPI()

# Deprecation schedule
VERSION_DEPRECATION = {
    "v1": {
        "deprecated": True,
        "sunset_date": "2025-12-31",
        "successor": "v2",
        "migration_guide": "https://docs.example.com/migration/v1-to-v2"
    }
}

class DeprecationMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)

        # Detect version from URL
        for version, info in VERSION_DEPRECATION.items():
            if f"/{version}/" in request.url.path and info["deprecated"]:
                # RFC 8594 Sunset header
                response.headers["Sunset"] = info["sunset_date"]
                response.headers["Deprecation"] = "true"
                response.headers["Link"] = (
                    f'<{info["migration_guide"]}>; rel="deprecation", '
                    f'<{request.url.path.replace(version, info["successor"])}>; rel="successor-version"'
                )

        return response

app.add_middleware(DeprecationMiddleware)

# Deprecated endpoint with explicit warning in docs
@app.get(
    "/api/v1/users",
    deprecated=True,
    summary="List users (Deprecated)",
    description="""
    ⚠️ **This endpoint is deprecated** and will be removed on **2025-12-31**.

    Please migrate to [`GET /api/v2/users`](/api/v2/users).

    **Migration guide**: https://docs.example.com/migration/v1-to-v2
    """
)
async def list_users_v1():
    return [{"id": 1, "name": "Alice"}]
```

---

## 18.5 Documentation Standards

### Clear Endpoint Descriptions

Good documentation is written in the code itself via docstrings and
Pydantic field descriptions. FastAPI renders it automatically in Swagger UI.

```python
from fastapi import FastAPI, Query, Path
from pydantic import BaseModel, Field
from typing import Optional

app = FastAPI(
    title="E-Commerce API",
    description="""
## Welcome to the E-Commerce API

Manage products, orders, and users for the platform.

### Authentication
All endpoints (except `/auth/login`) require a Bearer token in the
`Authorization` header.

### Versioning
This is **v2** of the API. See the [migration guide](/docs/migration)
if upgrading from v1.
    """,
    version="2.0.0",
    contact={
        "name": "API Support",
        "email": "api@example.com",
        "url": "https://support.example.com"
    },
    license_info={
        "name": "MIT",
        "url": "https://opensource.org/licenses/MIT"
    }
)

class ProductCreate(BaseModel):
    name: str = Field(
        ...,
        min_length=1,
        max_length=200,
        description="Product display name",
        examples=["Wireless Keyboard"]
    )
    price: float = Field(
        ...,
        gt=0,
        description="Price in USD. Must be greater than 0.",
        examples=[49.99]
    )
    sku: str = Field(
        ...,
        pattern=r"^[A-Z]{2}\d{6}$",
        description="Stock Keeping Unit. Format: 2 uppercase letters + 6 digits.",
        examples=["KB123456"]
    )
    category_id: int = Field(..., description="ID of the parent category")
    description: Optional[str] = Field(
        None,
        max_length=5000,
        description="Full product description. Supports Markdown."
    )

@app.post(
    "/products",
    status_code=201,
    response_model=ProductResponse,
    summary="Create a product",
    description="""
Creates a new product in the catalog.

**Permissions required**: `products:write`

**Notes**:
- SKU must be unique across the entire catalog
- Price is stored and returned in USD
- `category_id` must reference an existing, active category
    """,
    responses={
        201: {"description": "Product created successfully"},
        400: {"description": "Invalid input data"},
        401: {"description": "Authentication required"},
        403: {"description": "Insufficient permissions"},
        409: {"description": "SKU already exists"},
        422: {"description": "Validation error in request body"}
    },
    tags=["Products"]
)
async def create_product(product: ProductCreate):
    """
    Create a new product with the following fields:

    - **name**: Display name shown to customers
    - **price**: Selling price in USD (excluding tax)
    - **sku**: Unique stock keeping unit identifier
    - **category_id**: Must be an existing active category
    - **description**: Optional rich text description
    """
    pass
```

### Request/Response Examples

```python
from pydantic import BaseModel, Field

class UserCreate(BaseModel):
    username: str
    email: str
    password: str

    model_config = {
        "json_schema_extra": {
            "examples": [
                {
                    "summary": "Standard user",
                    "description": "Creating a regular user account",
                    "value": {
                        "username": "alice_smith",
                        "email": "alice@example.com",
                        "password": "securePassword123!"
                    }
                },
                {
                    "summary": "Admin user",
                    "description": "Creating an admin account",
                    "value": {
                        "username": "admin_bob",
                        "email": "bob@example.com",
                        "password": "adminSecurePass456!"
                    }
                }
            ]
        }
    }

class UserResponse(BaseModel):
    id: int
    username: str
    email: str
    created_at: str

    model_config = {
        "json_schema_extra": {
            "examples": [
                {
                    "id": 42,
                    "username": "alice_smith",
                    "email": "alice@example.com",
                    "created_at": "2024-01-15T10:30:00Z"
                }
            ]
        }
    }

@app.post(
    "/users",
    response_model=UserResponse,
    status_code=201,
    responses={
        201: {
            "description": "User created",
            "content": {
                "application/json": {
                    "example": {
                        "id": 42,
                        "username": "alice_smith",
                        "email": "alice@example.com",
                        "created_at": "2024-01-15T10:30:00Z"
                    }
                }
            }
        },
        409: {
            "description": "Email already in use",
            "content": {
                "application/json": {
                    "example": {
                        "status": "error",
                        "message": "Email already registered",
                        "code": "ALREADY_EXISTS"
                    }
                }
            }
        }
    }
)
async def create_user(user: UserCreate):
    pass
```

### Error Documentation

```python
# Reusable error response schemas for docs
from pydantic import BaseModel

class ErrorModel(BaseModel):
    status: str = "error"
    message: str
    code: str

# Shared error response definitions
COMMON_ERRORS = {
    400: {
        "description": "Bad Request — malformed or invalid data",
        "model": ErrorModel,
        "content": {
            "application/json": {
                "example": {"status": "error", "message": "Invalid date format", "code": "INVALID_FORMAT"}
            }
        }
    },
    401: {
        "description": "Unauthorized — missing or invalid auth token",
        "model": ErrorModel,
        "content": {
            "application/json": {
                "example": {"status": "error", "message": "Token expired", "code": "TOKEN_EXPIRED"}
            }
        }
    },
    403: {
        "description": "Forbidden — authenticated but insufficient permissions",
        "model": ErrorModel,
        "content": {
            "application/json": {
                "example": {"status": "error", "message": "Admin access required", "code": "PERMISSION_DENIED"}
            }
        }
    },
    404: {
        "description": "Not Found — resource doesn't exist",
        "model": ErrorModel,
        "content": {
            "application/json": {
                "example": {"status": "error", "message": "User 42 not found", "code": "NOT_FOUND"}
            }
        }
    },
    429: {
        "description": "Too Many Requests — rate limit exceeded",
        "model": ErrorModel,
        "content": {
            "application/json": {
                "example": {"status": "error", "message": "Rate limit exceeded. Try again in 60s", "code": "RATE_LIMIT_EXCEEDED"}
            }
        },
        "headers": {
            "Retry-After": {
                "description": "Seconds until rate limit resets",
                "schema": {"type": "integer"}
            }
        }
    }
}

@app.get("/orders/{order_id}", responses={**COMMON_ERRORS, 200: {"model": OrderResponse}})
async def get_order(order_id: int):
    pass
```

### Authentication Flows

```python
from fastapi import FastAPI
from fastapi.security import HTTPBearer, OAuth2PasswordBearer

app = FastAPI()

# Clearly document auth scheme
oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="/auth/login",
    description="JWT Bearer token. Obtain via POST /auth/login"
)

@app.post(
    "/auth/login",
    summary="Obtain access token",
    description="""
Authenticate with username and password to receive a JWT access token.

**Token lifetime**: 60 minutes

**Usage**: Include in subsequent requests as:
```
Authorization: Bearer <your_token>
```
    """,
    tags=["Authentication"],
    responses={
        200: {
            "description": "Login successful",
            "content": {
                "application/json": {
                    "example": {
                        "access_token": "eyJhbGci...",
                        "token_type": "bearer",
                        "expires_in": 3600
                    }
                }
            }
        },
        401: {"description": "Invalid credentials"}
    }
)
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    pass

@app.post(
    "/auth/refresh",
    summary="Refresh access token",
    description="Exchange a valid (non-expired) token for a new one with a reset expiry.",
    tags=["Authentication"]
)
async def refresh_token(token: str = Depends(oauth2_scheme)):
    pass

@app.post(
    "/auth/logout",
    summary="Invalidate token",
    description="Revoke the current token. It cannot be used after this call.",
    status_code=204,
    tags=["Authentication"]
)
async def logout(token: str = Depends(oauth2_scheme)):
    pass
```

### Rate Limiting Documentation

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()

# Rate limit tiers — documented per endpoint
RATE_LIMIT_DOCS = """
**Rate Limiting**

Requests are rate limited per API key:

| Tier    | Limit              |
|---------|--------------------|
| Free    | 60 requests/minute |
| Pro     | 600 requests/minute|
| Enterprise | Unlimited       |

When exceeded, you'll receive a `429 Too Many Requests` response with:
- `Retry-After` header: seconds until limit resets
- `X-RateLimit-Limit`: your tier's limit
- `X-RateLimit-Remaining`: requests left this window
- `X-RateLimit-Reset`: Unix timestamp when window resets
"""

@app.get(
    "/search",
    summary="Search products",
    description=f"""
Search the product catalog with full-text search.

{RATE_LIMIT_DOCS}

**Caching**: Results are cached for 60 seconds. Use `Cache-Control: no-cache`
to bypass.
    """,
    responses={
        200: {"description": "Search results"},
        429: {
            "description": "Rate limit exceeded",
            "headers": {
                "Retry-After": {"schema": {"type": "integer"}, "description": "Seconds until reset"},
                "X-RateLimit-Limit": {"schema": {"type": "integer"}},
                "X-RateLimit-Remaining": {"schema": {"type": "integer"}},
                "X-RateLimit-Reset": {"schema": {"type": "integer"}}
            }
        }
    },
    tags=["Search"]
)
async def search(q: str, request: Request):
    pass
```

---

## Complete Example: Production-Grade API

```python
# main.py
from fastapi import FastAPI, Request, HTTPException, Depends
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
import uuid
import logging

from api.v1.router import router as v1_router
from api.v2.router import router as v2_router
from core.exceptions import AppException
from middleware import DeprecationMiddleware, TimingMiddleware

logger = logging.getLogger(__name__)

app = FastAPI(
    title="Commerce API",
    description="Scalable e-commerce backend API",
    version="2.0.0",
    docs_url="/docs",
    redoc_url="/redoc",
    openapi_tags=[
        {"name": "Products",       "description": "Product catalog management"},
        {"name": "Orders",         "description": "Order lifecycle management"},
        {"name": "Users",          "description": "User account management"},
        {"name": "Authentication", "description": "Auth token operations"},
        {"name": "V1",             "description": "⚠️ Deprecated — use V2"},
    ]
)

# ── Middleware ───────────────────────────────────────────────
app.add_middleware(DeprecationMiddleware)
app.add_middleware(TimingMiddleware)

# ── Exception Handlers ──────────────────────────────────────
@app.middleware("http")
async def attach_request_id(request: Request, call_next):
    request.state.request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
    response = await call_next(request)
    response.headers["X-Request-ID"] = request.state.request_id
    return response

@app.exception_handler(RequestValidationError)
async def validation_handler(request: Request, exc: RequestValidationError):
    errors = [
        {
            "field": ".".join(str(l) for l in e["loc"] if l != "body"),
            "message": e["msg"],
            "code": e["type"].upper()
        }
        for e in exc.errors()
    ]
    return JSONResponse(status_code=422, content={
        "status": "error",
        "message": "Validation failed",
        "errors": errors,
        "request_id": request.state.request_id
    })

@app.exception_handler(AppException)
async def app_exception_handler(request: Request, exc: AppException):
    return JSONResponse(status_code=exc.status_code, content={
        "status": "error",
        "message": exc.detail["message"],
        "code": exc.detail["code"],
        "request_id": request.state.request_id
    })

@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    return JSONResponse(status_code=exc.status_code, content={
        "status": "error",
        "message": exc.detail,
        "request_id": request.state.request_id
    })

@app.exception_handler(Exception)
async def unhandled_handler(request: Request, exc: Exception):
    logger.exception(f"[{request.state.request_id}] Unhandled: {exc}")
    return JSONResponse(status_code=500, content={
        "status": "error",
        "message": "An unexpected error occurred",
        "request_id": request.state.request_id
    })

# ── Routers ─────────────────────────────────────────────────
app.include_router(v1_router, prefix="/api/v1")   # Deprecated
app.include_router(v2_router, prefix="/api/v2")   # Current

# ── Meta Endpoints ───────────────────────────────────────────
@app.get("/health", tags=["System"], include_in_schema=False)
async def health():
    return {"status": "healthy"}

@app.get("/", tags=["System"], include_in_schema=False)
async def root():
    return {
        "name": "Commerce API",
        "version": "2.0.0",
        "docs": "/docs",
        "current_version": "/api/v2",
        "deprecated_version": "/api/v1"
    }
```

---

## Summary

You've learned about API Design & Best Practices for FastAPI:

1. **RESTful Design**: REST constraints, resource naming (nouns, plural, lowercase), correct HTTP method usage, proper status codes, and HATEOAS
2. **Project Structure**: Modular architecture for small/medium/large projects, clean separation of concerns, organized routers, models, schemas, a services layer for business logic, and the repository pattern for data access
3. **Error Handling**: Consistent error shape with `status/message/errors/request_id`, machine-readable error codes, global exception handlers, typed custom exceptions, and business logic errors
4. **API Versioning**: URL versioning, header versioning, query parameter versioning, content negotiation, and graceful deprecation with `Sunset` and `Deprecation` headers
5. **Documentation Standards**: Rich endpoint descriptions, multiple request examples, reusable error response schemas, fully documented auth flows, and rate limit documentation

**Golden Rules**:
- Design for the **client**, not the implementation
- Be **consistent** — same patterns everywhere
- **Version** from day one, even if you only have v1
- **Errors** are part of your API contract — document them
- **Structure** code so each layer has exactly one job