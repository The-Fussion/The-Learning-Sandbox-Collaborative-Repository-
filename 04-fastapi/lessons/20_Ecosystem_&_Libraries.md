# Ecosystem & Libraries

## 20.1 Essential Libraries

### Pydantic for Validation

Pydantic is FastAPI's backbone for data validation, serialization, and
settings management. It uses Python type hints to define data shapes and
validates automatically at runtime.

**Installation**:
```bash
pip install pydantic[email]   # Includes email validation
```

**Core Features**:
```python
from pydantic import (
    BaseModel, Field, field_validator, model_validator,
    EmailStr, HttpUrl, constr, conint, confloat
)
from typing import Optional, List
from datetime import datetime
from enum import Enum

# Basic model
class UserCreate(BaseModel):
    username: str = Field(..., min_length=3, max_length=50)
    email: EmailStr
    age: int = Field(..., ge=0, le=120)
    website: Optional[HttpUrl] = None

# Field validators
class Product(BaseModel):
    name: str
    price: float
    sku: str

    @field_validator("price")
    @classmethod
    def price_must_be_positive(cls, v):
        if v <= 0:
            raise ValueError("Price must be greater than 0")
        return round(v, 2)

    @field_validator("sku")
    @classmethod
    def sku_format(cls, v):
        import re
        if not re.match(r"^[A-Z]{2}\d{6}$", v):
            raise ValueError("SKU must be 2 uppercase letters + 6 digits")
        return v

# Cross-field validation
class DateRange(BaseModel):
    start_date: datetime
    end_date: datetime

    @model_validator(mode="after")
    def end_after_start(self):
        if self.end_date <= self.start_date:
            raise ValueError("end_date must be after start_date")
        return self

# Nested models
class Address(BaseModel):
    street: str
    city: str
    country: str = "US"

class Order(BaseModel):
    id: int
    items: List[str]
    shipping_address: Address  # Nested validation
    total: float

# ORM mode (read from SQLAlchemy objects)
class UserResponse(BaseModel):
    id: int
    username: str
    email: str
    created_at: datetime

    model_config = {"from_attributes": True}   # Pydantic v2

# Settings management
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    app_name: str = "My API"
    debug: bool = False
    database_url: str
    secret_key: str
    redis_url: str = "redis://localhost:6379"
    allowed_origins: List[str] = ["http://localhost:3000"]

    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}

settings = Settings()  # Auto-loads from .env file
```

### SQLAlchemy for Databases

SQLAlchemy is the most popular Python ORM, providing both a high-level ORM
interface and a lower-level Core expression language.

**Installation**:
```bash
pip install sqlalchemy asyncpg        # PostgreSQL async
pip install sqlalchemy aiosqlite      # SQLite async
```

**Setup**:
```python
# database.py
from sqlalchemy.ext.asyncio import (
    AsyncSession, create_async_engine, async_sessionmaker
)
from sqlalchemy.orm import DeclarativeBase

DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/mydb"

engine = create_async_engine(
    DATABASE_URL,
    pool_size=20,
    max_overflow=10,
    pool_pre_ping=True,
    echo=False  # Set True to log SQL in development
)

AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)

class Base(DeclarativeBase):
    pass

async def get_db():
    async with AsyncSessionLocal() as session:
        yield session
```

**Models**:
```python
# models/user.py
from sqlalchemy import Column, Integer, String, Boolean, DateTime, ForeignKey, Text
from sqlalchemy.orm import relationship
from sqlalchemy.sql import func
from database import Base

class User(Base):
    __tablename__ = "users"

    id         = Column(Integer, primary_key=True, index=True)
    username   = Column(String(50), unique=True, index=True, nullable=False)
    email      = Column(String(255), unique=True, index=True, nullable=False)
    password   = Column(String(255), nullable=False)
    is_active  = Column(Boolean, default=True)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())

    posts = relationship("Post", back_populates="author", lazy="selectin")

class Post(Base):
    __tablename__ = "posts"

    id        = Column(Integer, primary_key=True, index=True)
    title     = Column(String(200), nullable=False)
    body      = Column(Text)
    author_id = Column(Integer, ForeignKey("users.id"), nullable=False)

    author = relationship("User", back_populates="posts")
```

**Async Queries**:
```python
from sqlalchemy import select, update, delete, func
from sqlalchemy.orm import selectinload

# SELECT
async def get_user(db: AsyncSession, user_id: int):
    result = await db.execute(select(User).where(User.id == user_id))
    return result.scalar_one_or_none()

# SELECT with eager loading
async def get_user_with_posts(db: AsyncSession, user_id: int):
    result = await db.execute(
        select(User)
        .options(selectinload(User.posts))
        .where(User.id == user_id)
    )
    return result.scalar_one_or_none()

# INSERT
async def create_user(db: AsyncSession, data: dict):
    user = User(**data)
    db.add(user)
    await db.commit()
    await db.refresh(user)
    return user

# UPDATE
async def update_user(db: AsyncSession, user_id: int, data: dict):
    await db.execute(
        update(User).where(User.id == user_id).values(**data)
    )
    await db.commit()

# DELETE
async def delete_user(db: AsyncSession, user_id: int):
    await db.execute(delete(User).where(User.id == user_id))
    await db.commit()

# Aggregation
async def count_users(db: AsyncSession):
    result = await db.execute(select(func.count(User.id)))
    return result.scalar()
```

### Alembic for Migrations

Alembic manages database schema changes over time, generating migration
scripts that can be applied and rolled back reliably.

**Installation**:
```bash
pip install alembic
```

**Setup**:
```bash
alembic init alembic          # Creates alembic/ directory
```

**alembic/env.py** — connect to your models:
```python
from logging.config import fileConfig
from sqlalchemy import engine_from_config, pool
from alembic import context

# Import your models so Alembic detects them
from database import Base
import models  # noqa: F401 - ensure all models are imported

config = context.config
fileConfig(config.config_file_name)
target_metadata = Base.metadata     # ← Point to your metadata

def run_migrations_online():
    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    with connectable.connect() as connection:
        context.configure(connection=connection, target_metadata=target_metadata)
        with context.begin_transaction():
            context.run_migrations()
```

**alembic.ini** — set database URL:
```ini
sqlalchemy.url = postgresql://user:pass@localhost/mydb
```

**Common Alembic Commands**:
```bash
# Auto-generate migration from model changes
alembic revision --autogenerate -m "add users table"

# Apply all pending migrations
alembic upgrade head

# Roll back one migration
alembic downgrade -1

# Roll back to specific revision
alembic downgrade abc123

# Show current version
alembic current

# Show migration history
alembic history --verbose
```

**Migration File Example**:
```python
# alembic/versions/abc123_add_users_table.py
from alembic import op
import sqlalchemy as sa

revision = "abc123"
down_revision = None
branch_labels = None
depends_on = None

def upgrade():
    op.create_table(
        "users",
        sa.Column("id", sa.Integer(), primary_key=True),
        sa.Column("username", sa.String(50), nullable=False, unique=True),
        sa.Column("email", sa.String(255), nullable=False, unique=True),
        sa.Column("password", sa.String(255), nullable=False),
        sa.Column("is_active", sa.Boolean(), default=True),
        sa.Column("created_at", sa.DateTime(timezone=True),
                  server_default=sa.func.now()),
    )
    op.create_index("ix_users_email", "users", ["email"])
    op.create_index("ix_users_username", "users", ["username"])

def downgrade():
    op.drop_index("ix_users_email")
    op.drop_index("ix_users_username")
    op.drop_table("users")
```

### python-jose for JWT

`python-jose` creates and validates JSON Web Tokens for stateless
authentication.

**Installation**:
```bash
pip install python-jose[cryptography]
```

```python
# core/security.py
from jose import JWTError, jwt
from datetime import datetime, timedelta
from fastapi import HTTPException, status, Depends
from fastapi.security import OAuth2PasswordBearer

SECRET_KEY    = "your-secret-key-min-32-chars-long"
ALGORITHM     = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES  = 30
REFRESH_TOKEN_EXPIRE_DAYS    = 7

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")

def create_access_token(subject: str | int, extra: dict = {}) -> str:
    payload = {
        "sub": str(subject),
        "exp": datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES),
        "iat": datetime.utcnow(),
        "type": "access",
        **extra
    }
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

def create_refresh_token(subject: str | int) -> str:
    payload = {
        "sub": str(subject),
        "exp": datetime.utcnow() + timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS),
        "type": "refresh"
    }
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

def decode_token(token: str) -> dict:
    try:
        return jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")

async def get_current_user(token: str = Depends(oauth2_scheme)):
    payload = decode_token(token)
    if payload.get("type") != "access":
        raise HTTPException(status_code=401, detail="Invalid token type")
    return payload
```

### passlib for Password Hashing

`passlib` provides secure password hashing using industry-standard
algorithms like bcrypt.

**Installation**:
```bash
pip install passlib[bcrypt]
```

```python
# core/password.py
from passlib.context import CryptContext

# Configure bcrypt (most secure option)
pwd_context = CryptContext(
    schemes=["bcrypt"],
    deprecated="auto",        # Auto-upgrade old hashes
    bcrypt__rounds=12         # Work factor (12 is a good balance)
)

def hash_password(plain: str) -> str:
    """Hash a plain-text password."""
    return pwd_context.hash(plain)

def verify_password(plain: str, hashed: str) -> bool:
    """Verify a plain password against a hash."""
    return pwd_context.verify(plain, hashed)

def needs_rehash(hashed: str) -> bool:
    """Check if hash should be upgraded to newer settings."""
    return pwd_context.needs_update(hashed)

# Usage in auth endpoint
from fastapi import FastAPI, HTTPException, Depends
from sqlalchemy.ext.asyncio import AsyncSession

app = FastAPI()

@app.post("/auth/login")
async def login(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: AsyncSession = Depends(get_db)
):
    user = await get_user_by_username(db, form_data.username)
    if not user or not verify_password(form_data.password, user.password):
        raise HTTPException(status_code=401, detail="Invalid credentials")

    # Silently upgrade hash if needed
    if needs_rehash(user.password):
        await update_password_hash(db, user.id, hash_password(form_data.password))

    return {
        "access_token": create_access_token(user.id),
        "refresh_token": create_refresh_token(user.id),
        "token_type": "bearer"
    }
```

### python-multipart for File Uploads

`python-multipart` parses `multipart/form-data` requests, enabling file
uploads in FastAPI.

**Installation**:
```bash
pip install python-multipart
```

```python
from fastapi import FastAPI, UploadFile, File, Form, HTTPException
from typing import List
import aiofiles
import os

app = FastAPI()

UPLOAD_DIR   = "uploads"
MAX_FILE_SIZE = 10 * 1024 * 1024   # 10 MB
ALLOWED_TYPES = {"image/jpeg", "image/png", "image/webp", "application/pdf"}

@app.post("/upload/single")
async def upload_single(file: UploadFile = File(...)):
    # Validate content type
    if file.content_type not in ALLOWED_TYPES:
        raise HTTPException(400, f"File type not allowed: {file.content_type}")

    # Read and validate size
    content = await file.read()
    if len(content) > MAX_FILE_SIZE:
        raise HTTPException(413, "File too large (max 10 MB)")

    # Save file
    os.makedirs(UPLOAD_DIR, exist_ok=True)
    path = os.path.join(UPLOAD_DIR, file.filename)

    async with aiofiles.open(path, "wb") as f:
        await f.write(content)

    return {"filename": file.filename, "size": len(content)}

@app.post("/upload/multiple")
async def upload_multiple(files: List[UploadFile] = File(...)):
    results = []
    for file in files:
        content = await file.read()
        path = os.path.join(UPLOAD_DIR, file.filename)
        async with aiofiles.open(path, "wb") as f:
            await f.write(content)
        results.append({"filename": file.filename, "size": len(content)})
    return results

# Mixed form data + file
@app.post("/upload/with-metadata")
async def upload_with_metadata(
    file: UploadFile = File(...),
    title: str = Form(...),
    description: str = Form(""),
):
    content = await file.read()
    return {"title": title, "description": description, "size": len(content)}
```

### aiofiles for Async File Operations

`aiofiles` makes file I/O non-blocking so it doesn't freeze the event loop.

**Installation**:
```bash
pip install aiofiles
```

```python
import aiofiles
import json
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

# Async read
@app.get("/file/{filename}")
async def read_file(filename: str):
    async with aiofiles.open(f"data/{filename}", mode="r") as f:
        content = await f.read()
    return {"content": content}

# Async write
@app.post("/file/{filename}")
async def write_file(filename: str, content: str):
    async with aiofiles.open(f"data/{filename}", mode="w") as f:
        await f.write(content)
    return {"message": "Written"}

# Async JSON
async def read_json(path: str) -> dict:
    async with aiofiles.open(path, mode="r") as f:
        raw = await f.read()
    return json.loads(raw)

async def write_json(path: str, data: dict):
    async with aiofiles.open(path, mode="w") as f:
        await f.write(json.dumps(data, indent=2))

# Stream large files without loading into memory
@app.get("/download/{filename}")
async def download_file(filename: str):
    async def file_chunks():
        async with aiofiles.open(f"files/{filename}", "rb") as f:
            while chunk := await f.read(65536):   # 64 KB chunks
                yield chunk

    return StreamingResponse(
        file_chunks(),
        media_type="application/octet-stream",
        headers={"Content-Disposition": f"attachment; filename={filename}"}
    )
```

---

## 20.2 Useful Extensions

### FastAPI Users (Complete Auth System)

FastAPI Users provides a complete, production-ready authentication system
with registration, login, password reset, OAuth, and more out of the box.

**Installation**:
```bash
pip install fastapi-users[sqlalchemy]
```

```python
from fastapi import FastAPI
from fastapi_users import FastAPIUsers
from fastapi_users.authentication import (
    AuthenticationBackend, BearerTransport, JWTStrategy
)
from fastapi_users.db import SQLAlchemyUserDatabase
from fastapi_users import schemas, models

# 1. Define your user model
from sqlalchemy import Column, String
from fastapi_users.db import SQLAlchemyBaseUserTableUUID

class User(SQLAlchemyBaseUserTableUUID, Base):
    __tablename__ = "users"
    full_name = Column(String(100))          # Custom field

# 2. Pydantic schemas
class UserRead(schemas.BaseUser):
    full_name: str

class UserCreate(schemas.BaseUserCreate):
    full_name: str

class UserUpdate(schemas.BaseUserUpdate):
    full_name: str = None

# 3. Database adapter
async def get_user_db(session: AsyncSession = Depends(get_db)):
    yield SQLAlchemyUserDatabase(session, User)

# 4. JWT strategy
SECRET = "your-secret-key"

def get_jwt_strategy() -> JWTStrategy:
    return JWTStrategy(secret=SECRET, lifetime_seconds=3600)

auth_backend = AuthenticationBackend(
    name="jwt",
    transport=BearerTransport(tokenUrl="auth/jwt/login"),
    get_strategy=get_jwt_strategy,
)

# 5. FastAPIUsers instance
fastapi_users = FastAPIUsers[User, int](
    get_user_db,
    [auth_backend],
)

# 6. Wire up to app
app = FastAPI()

app.include_router(
    fastapi_users.get_auth_router(auth_backend),
    prefix="/auth/jwt",
    tags=["auth"],
)
app.include_router(
    fastapi_users.get_register_router(UserRead, UserCreate),
    prefix="/auth",
    tags=["auth"],
)
app.include_router(
    fastapi_users.get_reset_password_router(),
    prefix="/auth",
    tags=["auth"],
)
app.include_router(
    fastapi_users.get_users_router(UserRead, UserUpdate),
    prefix="/users",
    tags=["users"],
)

# Now you have:
# POST /auth/jwt/login
# POST /auth/jwt/logout
# POST /auth/register
# POST /auth/forgot-password
# POST /auth/reset-password
# GET  /users/me
# PATCH /users/me

# Dependency to get current user
current_user = fastapi_users.current_user()

@app.get("/protected")
async def protected(user: User = Depends(current_user)):
    return {"email": user.email}
```

### FastAPI-Cache for Caching

FastAPI-Cache adds simple decorator-based caching with support for Redis,
Memcached, or in-memory backends.

**Installation**:
```bash
pip install fastapi-cache2[redis]
```

```python
from fastapi import FastAPI
from fastapi_cache import FastAPICache
from fastapi_cache.backends.redis import RedisBackend
from fastapi_cache.decorator import cache
from redis import asyncio as aioredis

app = FastAPI()

@app.on_event("startup")
async def startup():
    redis = aioredis.from_url("redis://localhost")
    FastAPICache.init(RedisBackend(redis), prefix="myapp-cache")

# Cache endpoint response for 5 minutes
@app.get("/products")
@cache(expire=300)
async def list_products():
    # This result is cached — database only hit on cache miss
    products = await db.execute(select(Product))
    return products.scalars().all()

# Cache with custom key
@app.get("/products/{product_id}")
@cache(expire=600)
async def get_product(product_id: int):
    return await db.get(Product, product_id)

# Custom cache key builder
def user_cache_key(func, *args, **kwargs):
    user_id = kwargs.get("user_id")
    return f"user:{user_id}:{func.__name__}"

@app.get("/users/{user_id}")
@cache(expire=120, key_builder=user_cache_key)
async def get_user(user_id: int):
    return await db.get(User, user_id)

# Manually invalidate cache
from fastapi_cache import FastAPICache

@app.put("/products/{product_id}")
async def update_product(product_id: int, data: dict):
    # Update database
    await db.execute(update(Product).where(Product.id == product_id).values(**data))
    await db.commit()

    # Invalidate cached version
    await FastAPICache.clear(namespace=f"product:{product_id}")

    return {"updated": product_id}
```

### FastAPI-Limiter for Rate Limiting

FastAPI-Limiter provides easy per-route rate limiting using Redis.

**Installation**:
```bash
pip install fastapi-limiter
```

```python
from fastapi import FastAPI, Depends, Request
from fastapi_limiter import FastAPILimiter
from fastapi_limiter.depends import RateLimiter
from redis import asyncio as aioredis

app = FastAPI()

@app.on_event("startup")
async def startup():
    redis = await aioredis.from_url("redis://localhost", encoding="utf-8")
    await FastAPILimiter.init(redis)

# 5 requests per minute
@app.get("/public", dependencies=[Depends(RateLimiter(times=5, seconds=60))])
async def public_endpoint():
    return {"message": "Limited to 5/min"}

# 100 requests per minute
@app.get("/api/data", dependencies=[Depends(RateLimiter(times=100, seconds=60))])
async def api_data():
    return {"data": []}

# Custom identifier (by API key instead of IP)
async def api_key_identifier(request: Request):
    api_key = request.headers.get("X-API-Key")
    return api_key or request.client.host

@app.get("/custom-limit")
@app.get("/custom-limit", dependencies=[
    Depends(RateLimiter(times=10, seconds=60, identifier=api_key_identifier))
])
async def custom_limited():
    return {"message": "Limited by API key"}
```

### FastAPI-Pagination

FastAPI-Pagination adds standard pagination to any endpoint with minimal
boilerplate.

**Installation**:
```bash
pip install fastapi-pagination[sqlalchemy]
```

```python
from fastapi import FastAPI
from fastapi_pagination import Page, add_pagination
from fastapi_pagination.ext.sqlalchemy import paginate
from sqlalchemy import select

app = FastAPI()

# Paginated endpoint — returns Page[ItemSchema]
@app.get("/items", response_model=Page[ItemResponse])
async def list_items(db: AsyncSession = Depends(get_db)):
    return await paginate(db, select(Item).order_by(Item.id))

# Cursor-based pagination
from fastapi_pagination import LimitOffsetPage

@app.get("/products", response_model=LimitOffsetPage[ProductResponse])
async def list_products(db: AsyncSession = Depends(get_db)):
    return await paginate(db, select(Product))

add_pagination(app)  # ← Required — adds params to all paginated routes

# GET /items?page=1&size=20
# Response:
# {
#   "items": [...],
#   "total": 150,
#   "page": 1,
#   "size": 20,
#   "pages": 8
# }
```

### FastAPI-Mail for Emails

FastAPI-Mail provides async email sending with templates.

**Installation**:
```bash
pip install fastapi-mail
```

```python
from fastapi import FastAPI, BackgroundTasks
from fastapi_mail import FastMail, MessageSchema, ConnectionConfig, MessageType
from pydantic import EmailStr

app = FastAPI()

# Email configuration
conf = ConnectionConfig(
    MAIL_USERNAME    = "your@email.com",
    MAIL_PASSWORD    = "your-password",
    MAIL_FROM        = "your@email.com",
    MAIL_PORT        = 587,
    MAIL_SERVER      = "smtp.gmail.com",
    MAIL_STARTTLS    = True,
    MAIL_SSL_TLS     = False,
    USE_CREDENTIALS  = True,
    TEMPLATE_FOLDER  = "templates/email"   # Jinja2 templates
)

mail = FastMail(conf)

# Simple email
@app.post("/send-email")
async def send_email(to: EmailStr, subject: str, body: str):
    message = MessageSchema(
        subject=subject,
        recipients=[to],
        body=body,
        subtype=MessageType.html
    )
    await mail.send_message(message)
    return {"message": "Sent"}

# Template email
@app.post("/welcome-email")
async def send_welcome(email: EmailStr, username: str, background_tasks: BackgroundTasks):
    message = MessageSchema(
        subject="Welcome!",
        recipients=[email],
        template_body={"username": username, "app_name": "MyApp"},
        subtype=MessageType.html
    )
    # Send in background so endpoint returns immediately
    background_tasks.add_task(mail.send_message, message, template_name="welcome.html")
    return {"message": "Email queued"}

# templates/email/welcome.html
# <!DOCTYPE html>
# <html>
# <body>
#   <h1>Welcome, {{ username }}!</h1>
#   <p>Thanks for joining {{ app_name }}.</p>
# </body>
# </html>
```

### fastapi-admin for Admin Panel

fastapi-admin provides a ready-made admin interface backed by your SQLAlchemy
models.

**Installation**:
```bash
pip install fastapi-admin
```

```python
from fastapi import FastAPI
from fastapi_admin.app import app as admin_app
from fastapi_admin.providers.login import UsernamePasswordProvider
from fastapi_admin.resources import Model, Field
import aioredis

app = FastAPI()

# Define admin resources for each model
class UserResource(Model):
    label = "Users"
    model = User
    icon  = "fas fa-users"
    page_pre_title = "User Management"

    fields = [
        Field(name="id",         label="ID"),
        Field(name="username",   label="Username", sortable=True),
        Field(name="email",      label="Email",    sortable=True),
        Field(name="is_active",  label="Active",   sortable=True),
        Field(name="created_at", label="Joined",   sortable=True),
    ]

class ProductResource(Model):
    label = "Products"
    model = Product
    icon  = "fas fa-box"
    fields = [
        Field(name="name",    label="Name",     sortable=True),
        Field(name="price",   label="Price",    sortable=True),
        Field(name="sku",     label="SKU"),
        Field(name="in_stock", label="In Stock"),
    ]

@app.on_event("startup")
async def startup():
    redis = aioredis.from_url("redis://localhost", decode_responses=True)
    await admin_app.configure(
        logo_url="https://example.com/logo.png",
        template_folders=["templates"],
        providers=[
            UsernamePasswordProvider(
                admin_model=AdminUser,
                login_logo_url="https://example.com/logo.png"
            )
        ],
        redis=redis,
        resources=[UserResource, ProductResource]
    )

app.mount("/admin", admin_app)
# Visit http://localhost:8000/admin
```

---

## 20.3 Testing Tools

### pytest

pytest is the standard Python testing framework — simple to use, powerful
to extend.

**Installation**:
```bash
pip install pytest pytest-cov
```

**Configuration** — `pytest.ini`:
```ini
[pytest]
testpaths       = tests
python_files    = test_*.py
python_classes  = Test*
python_functions = test_*
addopts         =
    -v
    --tb=short
    --strict-markers
    --cov=app
    --cov-report=term-missing
    --cov-fail-under=80
markers =
    unit: Unit tests (fast)
    integration: Integration tests (slower)
    slow: Slow tests (skipped in CI by default)
```

**Complete conftest.py**:
```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from app.main import app
from app.database import Base, get_db

# Use SQLite in-memory for tests
TEST_DB_URL = "sqlite:///./test.db"
engine = create_engine(TEST_DB_URL, connect_args={"check_same_thread": False})
TestingSession = sessionmaker(bind=engine)

@pytest.fixture(scope="session", autouse=True)
def create_tables():
    Base.metadata.create_all(bind=engine)
    yield
    Base.metadata.drop_all(bind=engine)

@pytest.fixture
def db():
    session = TestingSession()
    try:
        yield session
    finally:
        session.rollback()   # Roll back after each test
        session.close()

@pytest.fixture
def client(db):
    def override_db():
        yield db
    app.dependency_overrides[get_db] = override_db
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()

@pytest.fixture
def auth_client(client, db):
    """Authenticated test client."""
    # Create user
    from app.models import User
    from app.core.password import hash_password
    user = User(username="tester", email="tester@test.com",
                password=hash_password("testpass"))
    db.add(user)
    db.commit()

    # Login
    resp = client.post("/auth/login", data={
        "username": "tester", "password": "testpass"
    })
    token = resp.json()["access_token"]
    client.headers = {"Authorization": f"Bearer {token}"}
    return client
```

**Usage**:
```bash
pytest                              # All tests
pytest tests/test_users.py         # Single file
pytest -m unit                     # Marked tests only
pytest -m "not slow"               # Exclude slow tests
pytest --cov=app --cov-report=html # Coverage report
pytest -x                          # Stop on first failure
pytest --lf                        # Re-run last failures only
```

### httpx for Async Requests

httpx is a modern async HTTP client — it's what FastAPI's TestClient is
built on, and it supports real async testing.

**Installation**:
```bash
pip install httpx
```

```python
import pytest
import httpx
from app.main import app

# Sync testing with TestClient (standard approach)
from fastapi.testclient import TestClient

client = TestClient(app)

def test_get_products():
    response = client.get("/products")
    assert response.status_code == 200
    assert isinstance(response.json(), list)

# Async testing with AsyncClient
@pytest.mark.asyncio
async def test_get_products_async():
    async with httpx.AsyncClient(app=app, base_url="http://test") as ac:
        response = await ac.get("/products")

    assert response.status_code == 200

# Testing against a running server
@pytest.mark.asyncio
async def test_live_server():
    async with httpx.AsyncClient(base_url="http://localhost:8000") as ac:
        response = await ac.get("/health")

    assert response.status_code == 200

# Concurrent requests
@pytest.mark.asyncio
async def test_concurrent_requests():
    import asyncio
    async with httpx.AsyncClient(app=app, base_url="http://test") as ac:
        responses = await asyncio.gather(*[
            ac.get("/products") for _ in range(10)
        ])
    assert all(r.status_code == 200 for r in responses)
```

### pytest-asyncio

pytest-asyncio enables writing async test functions with native `async/await`
syntax.

**Installation**:
```bash
pip install pytest-asyncio
```

**Configuration** — add to `pytest.ini`:
```ini
[pytest]
asyncio_mode = auto       # auto-detect async tests (recommended)
```

```python
import pytest
import asyncio
from httpx import AsyncClient
from app.main import app

# Auto-detected with asyncio_mode = auto
async def test_create_user():
    async with AsyncClient(app=app, base_url="http://test") as ac:
        response = await ac.post("/users", json={
            "username": "alice",
            "email": "alice@example.com",
            "password": "secure123"
        })

    assert response.status_code == 201
    assert response.json()["username"] == "alice"

# Async fixtures
@pytest.fixture
async def async_db():
    async with AsyncSessionLocal() as session:
        yield session
        await session.rollback()

async def test_with_async_db(async_db):
    from sqlalchemy import select
    from app.models import User

    result = await async_db.execute(select(User))
    users = result.scalars().all()
    assert isinstance(users, list)

# Custom event loop (if needed)
@pytest.fixture(scope="session")
def event_loop():
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()
```

### Faker for Test Data

Faker generates realistic fake data — names, emails, addresses, phone
numbers — so you don't have to hardcode test values.

**Installation**:
```bash
pip install faker
```

```python
from faker import Faker

fake = Faker()
Faker.seed(42)  # Reproducible data across runs

# Basic data generation
print(fake.name())             # "Alice Johnson"
print(fake.email())            # "alice@example.com"
print(fake.phone_number())     # "+1-555-123-4567"
print(fake.address())          # "123 Main St, Springfield, IL"
print(fake.text(max_nb_chars=200))  # Random paragraph
print(fake.url())              # "https://example.com/path"
print(fake.uuid4())            # "550e8400-e29b-41d4-a716-446655440000"
print(fake.past_date())        # datetime.date(2023, 3, 15)
print(fake.random_int(1, 100)) # 42

# Use in tests
def test_create_user(client):
    response = client.post("/users", json={
        "username": fake.user_name(),
        "email": fake.email(),
        "password": fake.password(length=12)
    })
    assert response.status_code == 201

# Bulk test data
def generate_test_products(count: int = 10):
    return [
        {
            "name": fake.catch_phrase(),
            "price": round(fake.pyfloat(min_value=1, max_value=500), 2),
            "description": fake.text(max_nb_chars=200),
            "sku": fake.bothify("??######").upper()
        }
        for _ in range(count)
    ]

# Locale-specific data
fake_de = Faker("de_DE")
fake_jp = Faker("ja_JP")

print(fake_de.name())  # "Hans Müller"
print(fake_jp.name())  # "田中太郎"
```

### factory-boy for Test Fixtures

factory-boy creates model instances declaratively — cleaner than building
dicts by hand and works directly with SQLAlchemy models.

**Installation**:
```bash
pip install factory-boy
```

```python
import factory
from factory.alchemy import SQLAlchemyModelFactory
from faker import Faker
from app.models import User, Product, Order
from app.core.password import hash_password

fake = Faker()

# Base factory with shared session
class BaseFactory(SQLAlchemyModelFactory):
    class Meta:
        abstract = True
        sqlalchemy_session_persistence = "commit"

# User factory
class UserFactory(BaseFactory):
    class Meta:
        model = User

    username   = factory.LazyFunction(fake.user_name)
    email      = factory.LazyAttribute(lambda o: f"{o.username}@example.com")
    password   = factory.LazyFunction(lambda: hash_password("testpass"))
    is_active  = True
    is_admin   = False

# Product factory
class ProductFactory(BaseFactory):
    class Meta:
        model = Product

    name  = factory.LazyFunction(fake.catch_phrase)
    price = factory.LazyFunction(
        lambda: round(fake.pyfloat(min_value=1, max_value=999), 2)
    )
    sku   = factory.LazyFunction(
        lambda: fake.bothify("??######").upper()
    )

# Order factory with relationship
class OrderFactory(BaseFactory):
    class Meta:
        model = Order

    user  = factory.SubFactory(UserFactory)  # Creates user automatically
    total = factory.LazyFunction(
        lambda: round(fake.pyfloat(min_value=10, max_value=500), 2)
    )

# Use in tests
def test_list_products(client, db):
    # Create 5 products
    ProductFactory._meta.sqlalchemy_session = db
    ProductFactory.create_batch(5)

    response = client.get("/products")
    assert len(response.json()) == 5

def test_get_user_orders(client, db):
    UserFactory._meta.sqlalchemy_session = db
    OrderFactory._meta.sqlalchemy_session = db

    user   = UserFactory.create()
    orders = OrderFactory.create_batch(3, user=user)

    response = client.get(f"/users/{user.id}/orders")
    assert len(response.json()) == 3

# Traits for variations
class AdminUserFactory(UserFactory):
    is_admin = True

class InactiveUserFactory(UserFactory):
    is_active = False
```

---

## 20.4 Development Tools

### Black for Code Formatting

Black is an opinionated, zero-configuration code formatter that enforces a
consistent style across your entire codebase. You never argue about formatting
again — Black decides.

**Installation**:
```bash
pip install black
```

**Configuration** — `pyproject.toml`:
```toml
[tool.black]
line-length    = 88       # Black's default — good for modern screens
target-version = ["py311"]
include        = '\.pyi?$'
extend-exclude = '''
/(
  migrations
  | .venv
  | __pycache__
)/
'''
```

**Usage**:
```bash
black .                    # Format everything
black app/                 # Format specific directory
black app/main.py          # Format single file
black --check .            # Check without changing (CI)
black --diff .             # Show what would change
```

**Before Black**:
```python
# Inconsistent, hard to read
user_data={'name':'Alice','email':'alice@example.com','age':30,'is_active':True}
def create_user(name:str,email:str,age:int=0,is_active:bool=True)->dict:
    return {'name':name,'email':email,'age':age,'is_active':is_active}
```

**After Black**:
```python
# Clean, consistent, readable
user_data = {
    "name": "Alice",
    "email": "alice@example.com",
    "age": 30,
    "is_active": True,
}

def create_user(
    name: str,
    email: str,
    age: int = 0,
    is_active: bool = True,
) -> dict:
    return {"name": name, "email": email, "age": age, "is_active": is_active}
```

### isort for Import Sorting

isort automatically sorts and groups imports into the standard order:
stdlib → third-party → local. It's compatible with Black.

**Installation**:
```bash
pip install isort
```

**Configuration** — `pyproject.toml`:
```toml
[tool.isort]
profile                   = "black"    # Black-compatible settings
multi_line_output         = 3
line_length               = 88
known_third_party         = ["fastapi", "pydantic", "sqlalchemy"]
known_first_party         = ["app"]
skip                      = [".venv", "migrations"]
```

**Usage**:
```bash
isort .                    # Sort all files
isort app/                 # Sort directory
isort --check-only .       # CI check
isort --diff .             # Show changes
```

**Before isort**:
```python
from app.models import User
import os
from fastapi import FastAPI
import json
from app.database import get_db
from typing import List
import sqlalchemy
```

**After isort**:
```python
# 1. Standard library
import json
import os
from typing import List

# 2. Third-party
import sqlalchemy
from fastapi import FastAPI

# 3. Local
from app.database import get_db
from app.models import User
```

### Flake8 for Linting

Flake8 catches code style violations, undefined names, unused imports, and
common programming errors.

**Installation**:
```bash
pip install flake8 flake8-bugbear flake8-simplify
```

**Configuration** — `.flake8`:
```ini
[flake8]
max-line-length = 88          # Match Black
extend-ignore   =
    E203,                     # Whitespace before ':' (Black compatible)
    E501,                     # Line too long (Black handles this)
    W503                      # Line break before binary operator
per-file-ignores =
    __init__.py: F401         # Unused imports OK in __init__.py
    tests/*: S101             # Assert OK in tests
exclude =
    .git,
    .venv,
    migrations,
    __pycache__
```

**Usage**:
```bash
flake8 .                       # Lint everything
flake8 app/                    # Lint directory
flake8 app/main.py             # Lint single file
```

**Common Errors Flake8 Catches**:
```python
import os           # F401: imported but unused
from sys import *   # F403: wildcard imports

def bad():
    x = 1           # F841: local variable assigned but never used
    undefined_var   # F821: undefined name

if x == True:       # E712: comparison to True
    pass

if x == None:       # E711: comparison to None (use 'is None')
    pass
```

### MyPy for Type Checking

MyPy performs static type analysis, catching type mismatches before they
become runtime bugs.

**Installation**:
```bash
pip install mypy
```

**Configuration** — `mypy.ini` or `pyproject.toml`:
```toml
[tool.mypy]
python_version         = "3.11"
strict                 = true     # Enable all strict checks
ignore_missing_imports = true     # Skip untyped third-party libs
exclude                = ["migrations/", "tests/"]

# Per-module overrides
[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false
```

**Usage**:
```bash
mypy app/                          # Check directory
mypy app/main.py                   # Check file
mypy --strict app/                 # Maximum strictness
mypy --html-report mypy-report app/ # HTML report
```

**Type Checking in Practice**:
```python
from typing import Optional, List
from fastapi import FastAPI

app = FastAPI()

# ✅ Correctly typed — MyPy happy
def get_user(user_id: int) -> Optional[dict]:
    if user_id == 1:
        return {"id": 1, "name": "Alice"}
    return None

def process_users(users: List[dict]) -> int:
    return len(users)

# ❌ MyPy will catch these
def bad_function(x: int) -> str:
    return x           # error: Incompatible return value type (got int, expected str)

user = get_user(1)
print(user["name"])    # error: Item "None" of "Optional[dict]" has no attribute "__getitem__"

# ✅ Correct — handle None
user = get_user(1)
if user:
    print(user["name"])   # Safe: we checked it's not None
```

### Pre-commit Hooks

Pre-commit runs all your quality tools automatically before every `git commit`.
Bad code never enters your repository.

**Installation**:
```bash
pip install pre-commit
pre-commit install       # Install git hook
```

**Configuration** — `.pre-commit-config.yaml`:
```yaml
repos:
  # Black — code formatting
  - repo: https://github.com/psf/black
    rev: 23.12.1
    hooks:
      - id: black
        language_version: python3.11

  # isort — import sorting
  - repo: https://github.com/PyCQA/isort
    rev: 5.13.2
    hooks:
      - id: isort
        args: ["--profile", "black"]

  # Flake8 — linting
  - repo: https://github.com/PyCQA/flake8
    rev: 7.0.0
    hooks:
      - id: flake8

  # MyPy — type checking
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
        additional_dependencies: [pydantic, fastapi]

  # General hygiene
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace     # Remove trailing spaces
      - id: end-of-file-fixer       # Files end with newline
      - id: check-yaml              # Validate YAML files
      - id: check-json              # Validate JSON files
      - id: check-merge-conflict    # Catch merge conflict markers
      - id: debug-statements        # Catch leftover breakpoint() calls
      - id: check-added-large-files # Prevent large files in git
        args: ["--maxkb=500"]
```

**Usage**:
```bash
pre-commit run --all-files          # Run on all files manually
pre-commit run black                # Run single hook
pre-commit autoupdate               # Update hook versions
git commit -m "feat: add users"     # Hooks run automatically
```

### Poetry for Dependency Management

Poetry manages dependencies, virtual environments, and publishing — all in
one tool with a clean lockfile for reproducible installs.

**Installation**:
```bash
curl -sSL https://install.python-poetry.org | python3 -
```

**`pyproject.toml`** — the single source of truth:
```toml
[tool.poetry]
name        = "my-fastapi-app"
version     = "1.0.0"
description = "Production FastAPI application"
authors     = ["Your Name <you@example.com>"]

[tool.poetry.dependencies]
python          = "^3.11"
fastapi         = "^0.109.0"
uvicorn         = {extras = ["standard"], version = "^0.27.0"}
pydantic        = {extras = ["email"], version = "^2.5.0"}
pydantic-settings = "^2.1.0"
sqlalchemy      = "^2.0.0"
asyncpg         = "^0.29.0"
alembic         = "^1.13.0"
python-jose     = {extras = ["cryptography"], version = "^3.3.0"}
passlib         = {extras = ["bcrypt"], version = "^1.7.4"}
python-multipart = "^0.0.9"
aiofiles        = "^23.2.1"
redis           = "^5.0.0"
httpx           = "^0.26.0"

[tool.poetry.group.dev.dependencies]
pytest          = "^7.4.0"
pytest-cov      = "^4.1.0"
pytest-asyncio  = "^0.23.0"
faker           = "^22.0.0"
factory-boy     = "^3.3.0"
black           = "^23.12.0"
isort           = "^5.13.0"
flake8          = "^7.0.0"
mypy            = "^1.8.0"
pre-commit      = "^3.6.0"

[build-system]
requires        = ["poetry-core"]
build-backend   = "poetry.core.masonry.api"
```

**Common Poetry Commands**:
```bash
# Project management
poetry new my-project          # Scaffold new project
poetry init                    # Init in existing directory

# Dependencies
poetry add fastapi             # Add package
poetry add pytest --group dev  # Add dev dependency
poetry remove requests         # Remove package
poetry update                  # Update all to latest compatible
poetry update sqlalchemy       # Update specific package

# Environment
poetry install                 # Install all from lockfile
poetry install --without dev   # Install without dev dependencies (production)
poetry shell                   # Activate virtual environment
poetry run pytest              # Run command in venv

# Info
poetry show                    # List installed packages
poetry show --tree             # Dependency tree
poetry check                   # Validate pyproject.toml
poetry env info                # Show venv path and Python version
poetry lock                    # Regenerate poetry.lock
```

---

## Complete Example: Full Development Setup

```
my_fastapi_project/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── database.py
│   ├── config.py
│   ├── api/v1/
│   ├── models/
│   ├── schemas/
│   ├── services/
│   └── repositories/
├── tests/
│   ├── conftest.py
│   ├── test_users.py
│   └── test_products.py
├── alembic/
│   ├── env.py
│   └── versions/
├── .pre-commit-config.yaml
├── .flake8
├── pyproject.toml
├── poetry.lock
└── Makefile
```

**Makefile** — one command for every workflow:
```makefile
.PHONY: install dev test lint format typecheck migrate run

install:          ## Install all dependencies
	poetry install

dev:              ## Start dev server with reload
	poetry run uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

test:             ## Run tests with coverage
	poetry run pytest --cov=app --cov-report=html --cov-report=term-missing

lint:             ## Run flake8
	poetry run flake8 app/ tests/

format:           ## Run black + isort
	poetry run black app/ tests/
	poetry run isort app/ tests/

typecheck:        ## Run mypy
	poetry run mypy app/

check: format lint typecheck test  ## Run full quality suite

migrate:          ## Apply pending migrations
	poetry run alembic upgrade head

migration:        ## Generate new migration (make migration msg="add users")
	poetry run alembic revision --autogenerate -m "$(msg)"

run:              ## Start production server
	poetry run gunicorn app.main:app \
		-w 4 -k uvicorn.workers.UvicornWorker \
		--bind 0.0.0.0:8000
```

**config.py** — centralised settings with all libraries wired in:
```python
from functools import lru_cache
from pydantic_settings import BaseSettings
from typing import List

class Settings(BaseSettings):
    # App
    app_name:    str  = "My FastAPI App"
    debug:       bool = False
    version:     str  = "1.0.0"

    # Database
    database_url: str

    # Redis
    redis_url: str = "redis://localhost:6379"

    # Auth
    secret_key:                  str
    access_token_expire_minutes:  int = 30
    refresh_token_expire_days:    int = 7

    # CORS
    allowed_origins: List[str] = ["http://localhost:3000"]

    # Email
    mail_server:   str = "smtp.gmail.com"
    mail_port:     int = 587
    mail_username: str = ""
    mail_password: str = ""
    mail_from:     str = ""

    model_config = {
        "env_file": ".env",
        "env_file_encoding": "utf-8",
        "case_sensitive": False
    }

@lru_cache()
def get_settings() -> Settings:
    return Settings()
```

---

## Summary

You've learned the complete FastAPI ecosystem:

1. **Essential Libraries** — Pydantic (validation/settings), SQLAlchemy (ORM), Alembic (migrations), python-jose (JWT), passlib (passwords), python-multipart (uploads), aiofiles (async I/O)
2. **Useful Extensions** — FastAPI Users (auth system), FastAPI-Cache (response caching), FastAPI-Limiter (rate limiting), FastAPI-Pagination (paginated responses), FastAPI-Mail (templated emails), fastapi-admin (admin UI)
3. **Testing Tools** — pytest (test framework), httpx (async HTTP client), pytest-asyncio (async tests), Faker (realistic test data), factory-boy (model factories)
4. **Development Tools** — Black (formatting), isort (import sorting), Flake8 (linting), MyPy (type checking), pre-commit (automated checks), Poetry (dependency management)

**Recommended Stack for a New Project**:
```bash
# Core
poetry add fastapi uvicorn[standard] pydantic[email] pydantic-settings
poetry add sqlalchemy[asyncio] asyncpg alembic
poetry add python-jose[cryptography] passlib[bcrypt] python-multipart aiofiles

# Extensions
poetry add fastapi-cache2[redis] fastapi-limiter fastapi-pagination[sqlalchemy]

# Dev
poetry add --group dev pytest pytest-cov pytest-asyncio httpx faker factory-boy
poetry add --group dev black isort flake8 mypy pre-commit
```