# Database Integration with FastAPI

## 6.1 Database Fundamentals

### Synchronous vs Asynchronous Databases

Understanding the differences between sync and async database operations is crucial for FastAPI applications.

```python
# Synchronous approach (blocks the thread)
def get_user_sync(user_id: int):
    user = db.query(User).filter(User.id == user_id).first()
    return user

# Asynchronous approach (non-blocking)
async def get_user_async(user_id: int):
    async with async_session() as session:
        result = await session.execute(select(User).filter(User.id == user_id))
        user = result.scalar_one_or_none()
        return user
```

**When to use:**
- **Sync**: Simple applications, CPU-bound operations, existing sync codebases
- **Async**: High I/O operations, many concurrent connections, scalability needs

### Connection Management

Proper connection management prevents resource leaks and ensures database reliability.

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# Synchronous connection
DATABASE_URL = "postgresql://user:password@localhost/dbname"
engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# Async connection
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

ASYNC_DATABASE_URL = "postgresql+asyncpg://user:password@localhost/dbname"
async_engine = create_async_engine(ASYNC_DATABASE_URL)
AsyncSessionLocal = sessionmaker(
    async_engine, 
    class_=AsyncSession, 
    expire_on_commit=False
)
```

### Connection Pooling

Connection pooling reuses database connections to improve performance.

```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    DATABASE_URL,
    poolclass=QueuePool,
    pool_size=5,           # Number of persistent connections
    max_overflow=10,       # Additional connections when pool is full
    pool_timeout=30,       # Seconds to wait for connection
    pool_recycle=3600,     # Recycle connections after 1 hour
    pool_pre_ping=True     # Test connections before using
)
```

### Database Sessions

Sessions manage transactions and object state.

```python
from fastapi import Depends
from sqlalchemy.orm import Session

# Session dependency
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# Usage in endpoint
@app.get("/users/{user_id}")
def read_user(user_id: int, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.id == user_id).first()
    return user
```

### Transaction Management

Transactions ensure data consistency with ACID properties.

```python
from sqlalchemy.orm import Session

def create_user_with_profile(db: Session, user_data: dict, profile_data: dict):
    try:
        # Start transaction (automatic with session)
        user = User(**user_data)
        db.add(user)
        db.flush()  # Get user.id without committing
        
        profile = Profile(**profile_data, user_id=user.id)
        db.add(profile)
        
        db.commit()  # Commit transaction
        return user
    except Exception as e:
        db.rollback()  # Rollback on error
        raise e

# Manual transaction control
def transfer_funds(db: Session, from_account: int, to_account: int, amount: float):
    with db.begin():  # Explicit transaction
        account1 = db.query(Account).filter(Account.id == from_account).first()
        account2 = db.query(Account).filter(Account.id == to_account).first()
        
        account1.balance -= amount
        account2.balance += amount
        # Commits automatically if no exception
```

## 6.2 SQLAlchemy (Sync)

### SQLAlchemy Core vs ORM

**Core**: Lower-level SQL expression language
**ORM**: Higher-level object-relational mapping

```python
# Core approach
from sqlalchemy import Table, Column, Integer, String, MetaData, select

metadata = MetaData()
users_table = Table(
    'users',
    metadata,
    Column('id', Integer, primary_key=True),
    Column('name', String(50)),
    Column('email', String(100))
)

# ORM approach
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'
    
    id = Column(Integer, primary_key=True)
    name = Column(String(50))
    email = Column(String(100))
```

### Database Setup and Connection

Complete database configuration for a FastAPI application.

```python
# database.py
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

SQLALCHEMY_DATABASE_URL = "postgresql://user:password@localhost/dbname"
# For SQLite: "sqlite:///./sql_app.db"

engine = create_engine(
    SQLALCHEMY_DATABASE_URL,
    connect_args={"check_same_thread": False}  # Only for SQLite
)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

Base = declarative_base()

# Dependency
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### Creating Models

Define database models using SQLAlchemy ORM.

```python
# models.py
from sqlalchemy import Column, Integer, String, Boolean, DateTime, ForeignKey, Text
from sqlalchemy.orm import relationship
from datetime import datetime
from .database import Base

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    email = Column(String(100), unique=True, index=True, nullable=False)
    username = Column(String(50), unique=True, index=True, nullable=False)
    hashed_password = Column(String(255), nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    # Relationships
    posts = relationship("Post", back_populates="author", cascade="all, delete-orphan")
    profile = relationship("Profile", back_populates="user", uselist=False)

class Post(Base):
    __tablename__ = "posts"
    
    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(200), nullable=False)
    content = Column(Text)
    published = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    
    author = relationship("User", back_populates="posts")
    tags = relationship("Tag", secondary="post_tags", back_populates="posts")

class Profile(Base):
    __tablename__ = "profiles"
    
    id = Column(Integer, primary_key=True, index=True)
    bio = Column(Text)
    avatar_url = Column(String(500))
    user_id = Column(Integer, ForeignKey("users.id"), unique=True)
    
    user = relationship("User", back_populates="profile")
```

### CRUD Operations

Implement Create, Read, Update, Delete operations.

```python
# crud.py
from sqlalchemy.orm import Session
from . import models, schemas

# CREATE
def create_user(db: Session, user: schemas.UserCreate):
    db_user = models.User(
        email=user.email,
        username=user.username,
        hashed_password=user.password  # Hash this in production!
    )
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user

# READ
def get_user(db: Session, user_id: int):
    return db.query(models.User).filter(models.User.id == user_id).first()

def get_users(db: Session, skip: int = 0, limit: int = 100):
    return db.query(models.User).offset(skip).limit(limit).all()

def get_user_by_email(db: Session, email: str):
    return db.query(models.User).filter(models.User.email == email).first()

# UPDATE
def update_user(db: Session, user_id: int, user_update: schemas.UserUpdate):
    db_user = db.query(models.User).filter(models.User.id == user_id).first()
    if db_user:
        for key, value in user_update.dict(exclude_unset=True).items():
            setattr(db_user, key, value)
        db.commit()
        db.refresh(db_user)
    return db_user

# DELETE
def delete_user(db: Session, user_id: int):
    db_user = db.query(models.User).filter(models.User.id == user_id).first()
    if db_user:
        db.delete(db_user)
        db.commit()
    return db_user
```

### Relationships (One-to-Many, Many-to-Many)

Define and work with database relationships.

```python
from sqlalchemy import Table

# Many-to-Many relationship table
post_tags = Table(
    'post_tags',
    Base.metadata,
    Column('post_id', Integer, ForeignKey('posts.id'), primary_key=True),
    Column('tag_id', Integer, ForeignKey('tags.id'), primary_key=True)
)

class Tag(Base):
    __tablename__ = "tags"
    
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(50), unique=True, nullable=False)
    
    posts = relationship("Post", secondary=post_tags, back_populates="tags")

# Working with relationships
def create_post_with_tags(db: Session, post_data: dict, tag_names: list):
    # Create or get tags
    tags = []
    for tag_name in tag_names:
        tag = db.query(Tag).filter(Tag.name == tag_name).first()
        if not tag:
            tag = Tag(name=tag_name)
            db.add(tag)
        tags.append(tag)
    
    # Create post with tags
    post = Post(**post_data)
    post.tags = tags
    db.add(post)
    db.commit()
    db.refresh(post)
    return post
```

### Querying and Filtering

Advanced querying techniques with SQLAlchemy.

```python
from sqlalchemy import and_, or_, not_, func

# Basic filtering
users = db.query(User).filter(User.is_active == True).all()

# Multiple conditions with and_
users = db.query(User).filter(
    and_(
        User.is_active == True,
        User.email.like('%@gmail.com')
    )
).all()

# OR conditions
users = db.query(User).filter(
    or_(
        User.username == 'admin',
        User.email.endswith('@admin.com')
    )
).all()

# NOT condition
users = db.query(User).filter(not_(User.is_active)).all()

# LIKE and ILIKE (case-insensitive)
users = db.query(User).filter(User.email.like('%@example.com')).all()
users = db.query(User).filter(User.username.ilike('john%')).all()

# IN clause
user_ids = [1, 2, 3, 4, 5]
users = db.query(User).filter(User.id.in_(user_ids)).all()

# Ordering
users = db.query(User).order_by(User.created_at.desc()).all()

# Limiting and offsetting
users = db.query(User).limit(10).offset(20).all()

# Counting
user_count = db.query(func.count(User.id)).scalar()

# Aggregation
from sqlalchemy import func
post_counts = db.query(
    User.username,
    func.count(Post.id).label('post_count')
).join(Post).group_by(User.username).all()
```

### Joins and Eager Loading

Optimize queries with joins and eager loading.

```python
from sqlalchemy.orm import joinedload, selectinload, subqueryload

# Implicit join
posts_with_authors = db.query(Post).join(User).filter(
    User.is_active == True
).all()

# Explicit join
posts = db.query(Post).join(
    User, 
    Post.user_id == User.id
).filter(User.username == 'john').all()

# Left outer join
posts = db.query(Post).outerjoin(Tag, Post.tags).all()

# Eager loading to avoid N+1 queries

# joinedload - uses LEFT OUTER JOIN
users = db.query(User).options(
    joinedload(User.posts)
).all()

# selectinload - uses separate SELECT IN query
users = db.query(User).options(
    selectinload(User.posts).selectinload(Post.tags)
).all()

# subqueryload - uses subquery
users = db.query(User).options(
    subqueryload(User.posts)
).all()
```

### Session Management

Best practices for managing database sessions.

```python
from contextlib import contextmanager

@contextmanager
def get_db_context():
    db = SessionLocal()
    try:
        yield db
        db.commit()
    except Exception:
        db.rollback()
        raise
    finally:
        db.close()

# Usage
def some_operation():
    with get_db_context() as db:
        user = User(username="john", email="john@example.com")
        db.add(user)
        # Automatically commits on success, rolls back on error
```

### Dependency Injection for Database

Using FastAPI's dependency injection system.

```python
from fastapi import Depends, FastAPI, HTTPException
from sqlalchemy.orm import Session

app = FastAPI()

# Simple dependency
@app.get("/users/{user_id}")
def read_user(user_id: int, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.id == user_id).first()
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return user

# Nested dependencies
def get_current_user(db: Session = Depends(get_db), token: str = Header(...)):
    # Verify token and get user
    user = verify_token(db, token)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid token")
    return user

@app.get("/me")
def read_current_user(current_user: User = Depends(get_current_user)):
    return current_user
```

## 6.3 SQLAlchemy (Async)

### AsyncEngine Setup

Configure async database engine for asynchronous operations.

```python
# database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker, declarative_base

DATABASE_URL = "postgresql+asyncpg://user:password@localhost/dbname"
# For MySQL: "mysql+aiomysql://user:password@localhost/dbname"

async_engine = create_async_engine(
    DATABASE_URL,
    echo=True,
    future=True,
    pool_size=5,
    max_overflow=10
)

AsyncSessionLocal = sessionmaker(
    async_engine,
    class_=AsyncSession,
    expire_on_commit=False,
    autocommit=False,
    autoflush=False
)

Base = declarative_base()

# Async dependency
async def get_async_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()
```

### AsyncSession

Working with async sessions.

```python
from sqlalchemy.ext.asyncio import AsyncSession

async def create_tables():
    async with async_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

async def drop_tables():
    async with async_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
```

### Async CRUD Operations

Implement async CRUD operations.

```python
from sqlalchemy import select, update, delete
from sqlalchemy.ext.asyncio import AsyncSession

# CREATE
async def create_user(db: AsyncSession, user_data: dict):
    user = User(**user_data)
    db.add(user)
    await db.commit()
    await db.refresh(user)
    return user

# READ
async def get_user(db: AsyncSession, user_id: int):
    result = await db.execute(select(User).filter(User.id == user_id))
    return result.scalar_one_or_none()

async def get_users(db: AsyncSession, skip: int = 0, limit: int = 100):
    result = await db.execute(select(User).offset(skip).limit(limit))
    return result.scalars().all()

# UPDATE
async def update_user(db: AsyncSession, user_id: int, update_data: dict):
    stmt = update(User).where(User.id == user_id).values(**update_data)
    await db.execute(stmt)
    await db.commit()
    return await get_user(db, user_id)

# DELETE
async def delete_user(db: AsyncSession, user_id: int):
    stmt = delete(User).where(User.id == user_id)
    await db.execute(stmt)
    await db.commit()
```

### Async Querying

Advanced async queries.

```python
from sqlalchemy import select, and_, or_, func

async def get_active_users(db: AsyncSession):
    stmt = select(User).filter(User.is_active == True)
    result = await db.execute(stmt)
    return result.scalars().all()

async def search_users(db: AsyncSession, search_term: str):
    stmt = select(User).filter(
        or_(
            User.username.ilike(f'%{search_term}%'),
            User.email.ilike(f'%{search_term}%')
        )
    )
    result = await db.execute(stmt)
    return result.scalars().all()

async def count_posts_by_user(db: AsyncSession, user_id: int):
    stmt = select(func.count(Post.id)).filter(Post.user_id == user_id)
    result = await db.execute(stmt)
    return result.scalar()
```

### Async Relationships

Work with relationships in async context.

```python
from sqlalchemy.orm import selectinload, joinedload

async def get_user_with_posts(db: AsyncSession, user_id: int):
    stmt = select(User).options(
        selectinload(User.posts)
    ).filter(User.id == user_id)
    result = await db.execute(stmt)
    return result.scalar_one_or_none()

async def get_posts_with_author_and_tags(db: AsyncSession):
    stmt = select(Post).options(
        joinedload(Post.author),
        selectinload(Post.tags)
    )
    result = await db.execute(stmt)
    return result.scalars().unique().all()
```

### Connection Pooling with asyncpg/aiomysql

Configure connection pooling for async databases.

```python
# PostgreSQL with asyncpg
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine(
    "postgresql+asyncpg://user:password@localhost/dbname",
    pool_size=20,
    max_overflow=0,
    pool_recycle=3600,
    pool_pre_ping=True,
    echo_pool=True
)

# MySQL with aiomysql
engine = create_async_engine(
    "mysql+aiomysql://user:password@localhost/dbname",
    pool_size=10,
    max_overflow=5
)
```

## 6.4 Alembic Migrations

### Installing and Configuring Alembic

Set up Alembic for database migrations.

```bash
# Install Alembic
pip install alembic

# Initialize Alembic
alembic init alembic
```

```python
# alembic/env.py
from logging.config import fileConfig
from sqlalchemy import engine_from_config, pool
from alembic import context
import sys
from pathlib import Path

# Add your project to the path
sys.path.append(str(Path(__file__).resolve().parents[1]))

from app.database import Base
from app.models import User, Post, Profile  # Import all models

# this is the Alembic Config object
config = context.config

# Interpret the config file for Python logging
fileConfig(config.config_file_name)

# Set target metadata
target_metadata = Base.metadata

# Get database URL from environment or config
from app.config import settings
config.set_main_option('sqlalchemy.url', settings.DATABASE_URL)

def run_migrations_offline():
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    
    with context.begin_transaction():
        context.run_migrations()

def run_migrations_online():
    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    
    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata
        )
        
        with context.begin_transaction():
            context.run_migrations()

if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

### Creating Migrations

Generate migration files for database changes.

```bash
# Create a new migration
alembic revision --autogenerate -m "Create users table"

# Create empty migration
alembic revision -m "Add custom index"
```

```python
# Example migration file: alembic/versions/xxx_create_users_table.py
from alembic import op
import sqlalchemy as sa

revision = 'xxx'
down_revision = None
branch_labels = None
depends_on = None

def upgrade():
    op.create_table(
        'users',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('email', sa.String(100), nullable=False),
        sa.Column('username', sa.String(50), nullable=False),
        sa.Column('hashed_password', sa.String(255), nullable=False),
        sa.Column('is_active', sa.Boolean(), default=True),
        sa.Column('created_at', sa.DateTime(), server_default=sa.func.now()),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('email'),
        sa.UniqueConstraint('username')
    )
    op.create_index('ix_users_email', 'users', ['email'])

def downgrade():
    op.drop_index('ix_users_email', 'users')
    op.drop_table('users')
```

### Running Migrations

Apply migrations to the database.

```bash
# Upgrade to latest version
alembic upgrade head

# Upgrade to specific revision
alembic upgrade abc123

# Upgrade by one version
alembic upgrade +1

# Show current revision
alembic current

# Show migration history
alembic history
```

### Rollback Migrations

Revert database changes.

```bash
# Downgrade by one version
alembic downgrade -1

# Downgrade to specific revision
alembic downgrade abc123

# Downgrade to base (remove all migrations)
alembic downgrade base
```

### Auto-generating Migrations

Let Alembic detect model changes automatically.

```bash
# Auto-generate migration from model changes
alembic revision --autogenerate -m "Add user profile table"

# Review the generated migration file before applying!
# File: alembic/versions/xxx_add_user_profile_table.py
```

### Managing Migration History

Track and manage migrations.

```bash
# Show detailed history
alembic history --verbose

# Show specific range
alembic history -r abc123:def456

# Stamp database with revision (without running migration)
alembic stamp head

# Generate SQL without executing
alembic upgrade head --sql
```

## 6.5 Other ORMs

### Tortoise ORM (Async Native)

Tortoise ORM is designed specifically for async Python.

```python
from tortoise import fields
from tortoise.models import Model
from tortoise.contrib.fastapi import register_tortoise

class User(Model):
    id = fields.IntField(pk=True)
    username = fields.CharField(max_length=50, unique=True)
    email = fields.CharField(max_length=100, unique=True)
    is_active = fields.BooleanField(default=True)
    created_at = fields.DatetimeField(auto_now_add=True)
    
    posts: fields.ReverseRelation["Post"]
    
    class Meta:
        table = "users"

class Post(Model):
    id = fields.IntField(pk=True)
    title = fields.CharField(max_length=200)
    content = fields.TextField()
    author = fields.ForeignKeyField("models.User", related_name="posts")
    created_at = fields.DatetimeField(auto_now_add=True)
    
    class Meta:
        table = "posts"

# FastAPI integration
app = FastAPI()

register_tortoise(
    app,
    db_url="postgres://user:password@localhost/dbname",
    modules={"models": ["app.models"]},
    generate_schemas=True,
    add_exception_handlers=True,
)

# Usage
@app.post("/users")
async def create_user(username: str, email: str):
    user = await User.create(username=username, email=email)
    return user

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    user = await User.get(id=user_id).prefetch_related("posts")
    return user
```

### Piccolo ORM

A modern async ORM with a focus on developer experience.

```python
from piccolo.table import Table
from piccolo.columns import Varchar, Integer, Boolean, Timestamp, ForeignKey

class User(Table):
    username = Varchar(length=50, unique=True)
    email = Varchar(length=100, unique=True)
    is_active = Boolean(default=True)
    created_at = Timestamp()

class Post(Table):
    title = Varchar(length=200)
    content = Varchar()
    author = ForeignKey(User)

# Usage
async def create_user():
    user = User(username="john", email="john@example.com")
    await user.save()
    return user

async def get_users():
    return await User.select().where(User.is_active == True)
```

### SQLModel (FastAPI + SQLAlchemy)

SQLModel combines Pydantic and SQLAlchemy for seamless FastAPI integration.

```python
from sqlmodel import Field, SQLModel, create_engine, Session, select

class User(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    username: str = Field(index=True, unique=True)
    email: str = Field(unique=True)
    is_active: bool = Field(default=True)

# Engine setup
engine = create_engine("sqlite:///database.db")
SQLModel.metadata.create_all(engine)

# Usage
@app.post("/users", response_model=User)
def create_user(user: User):
    with Session(engine) as session:
        session.add(user)
        session.commit()
        session.refresh(user)
        return user

@app.get("/users", response_model=list[User])
def read_users():
    with Session(engine) as session:
        users = session.exec(select(User)).all()
        return users
```

### Prisma Python

Modern database toolkit with type safety and auto-completion.

```python
# Install
# pip install prisma

# schema.prisma
"""
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-py"
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  username  String   @unique
  posts     Post[]
  createdAt DateTime @default(now())
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String?
  authorId  Int
  author    User     @relation(fields: [authorId], references: [id])
}
"""

# Generate client
# prisma generate

# Usage
from prisma import Prisma

@app.on_event("startup")
async def startup():
    db = Prisma()
    await db.connect()

@app.post("/users")
async def create_user(username: str, email: str):
    user = await db.user.create(
        data={"username": username, "email": email}
    )
    return user

@app.get("/users")
async def get_users():
    users = await db.user.find_many(include={"posts": True})
    return users
```

## 6.6 NoSQL Databases

### MongoDB with Motor (Async)

Motor is the async MongoDB driver for Python.

```python
from motor.motor_asyncio import AsyncIOMotorClient
from fastapi import FastAPI
from bson import ObjectId

app = FastAPI()

# MongoDB connection
MONGODB_URL = "mongodb://localhost:27017"
client = AsyncIOMotorClient(MONGODB_URL)
db = client.mydatabase

# Pydantic models
from pydantic import BaseModel, Field

class PyObjectId(ObjectId):
    @classmethod
    def __get_validators__(cls):
        yield cls.validate
    
    @classmethod
    def validate(cls, v):
        if not ObjectId.is_valid(v):
            raise ValueError("Invalid ObjectId")
        return ObjectId(v)

class UserModel(BaseModel):
    id: PyObjectId = Field(default_factory=PyObjectId, alias="_id")
    username: str
    email: str
    is_active: bool = True
    
    class Config:
        json_encoders = {ObjectId: str}

# CRUD operations
@app.post("/users", response_model=UserModel)
async def create_user(user: UserModel):
    user_dict = user.dict(by_alias=True)
    result = await db.users.insert_one(user_dict)
    user.id = result.inserted_id
    return user

@app.get("/users/{user_id}", response_model=UserModel)
async def get_user(user_id: str):
    user = await db.users.find_one({"_id": ObjectId(user_id)})
    if user:
        return UserModel(**user)
    raise HTTPException(status_code=404, detail="User not found")

@app.get("/users", response_model=list[UserModel])
async def get_users(skip: int = 0, limit: int = 10):
    users = await db.users.find().skip(skip).limit(limit).to_list(limit)
    return [UserModel(**user) for user in users]

@app.put("/users/{user_id}")
async def update_user(user_id: str, user: UserModel):
    await db.users.update_one(
        {"_id": ObjectId(user_id)},
        {"$set": user.dict(exclude={"id"})}
    )
    return await get_user(user_id)

@app.delete("/users/{user_id}")
async def delete_user(user_id: str):
    result = await db.users.delete_one({"_id": ObjectId(user_id)})
    if result.deleted_count:
        return {"message": "User deleted"}
    raise HTTPException(status_code=404, detail="User not found")
```

### MongoDB with PyMongo

Synchronous MongoDB driver.

```python
from pymongo import MongoClient
from fastapi import FastAPI

app = FastAPI()

# MongoDB connection
client = MongoClient("mongodb://localhost:27017/")
db = client.mydatabase

@app.post("/users")
def create_user(username: str, email: str):
    user = {"username": username, "email": email, "is_active": True}
    result = db.users.insert_one(user)
    user["_id"] = str(result.inserted_id)
    return user

@app.get("/users/{user_id}")
def get_user(user_id: str):
    user = db.users.find_one({"_id": ObjectId(user_id)})
    if user:
        user["_id"] = str(user["_id"])
        return user
    raise HTTPException(status_code=404, detail="User not found")
```

### Redis Integration

Use Redis for caching and session management.

```python
from redis import asyncio as aioredis
from fastapi import FastAPI
import json

app = FastAPI()

# Redis connection
redis = None

@app.on_event("startup")
async def startup():
    global redis
    redis = await aioredis.from_url(
        "redis://localhost",
        encoding="utf-8",
        decode_responses=True
    )

@app.on_event("shutdown")
async def shutdown():
    await redis.close()

# Caching example
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    # Try cache first
    cached = await redis.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)
    
    # Fetch from database
    user = await fetch_user_from_db(user_id)
    
    # Store in cache (expire in 1 hour)
    await redis.setex(
        f"user:{user_id}",
        3600,
        json.dumps(user)
    )
    return user

# Session management
@app.post("/login")
async def login(username: str, password: str):
    # Verify credentials
    user = await verify_credentials(username, password)
    
    # Create session
    session_id = generate_session_id()
    await redis.setex(
        f"session:{session_id}",
        86400,  # 24 hours
        json.dumps({"user_id": user.id, "username": user.username})
    )
    return {"session_id": session_id}

# Rate limiting
from fastapi import Request, HTTPException

async def rate_limit(request: Request):
    client_ip = request.client.host
    key = f"rate_limit:{client_ip}"
    
    requests = await redis.incr(key)
    if requests == 1:
        await redis.expire(key, 60)  # 1 minute window
    
    if requests > 100:  # Max 100 requests per minute
        raise HTTPException(status_code=429, detail="Rate limit exceeded")
```

### Elasticsearch

Full-text search with Elasticsearch.

```python
from elasticsearch import AsyncElasticsearch
from fastapi import FastAPI

app = FastAPI()

# Elasticsearch connection
es = AsyncElasticsearch(["http://localhost:9200"])

@app.on_event("shutdown")
async def shutdown():
    await es.close()

# Index documents
@app.post("/articles")
async def create_article(title: str, content: str, author: str):
    doc = {
        "title": title,
        "content": content,
        "author": author,
        "timestamp": datetime.now()
    }
    result = await es.index(index="articles", document=doc)
    return {"id": result["_id"]}

# Search
@app.get("/search")
async def search_articles(q: str, size: int = 10):
    body = {
        "query": {
            "multi_match": {
                "query": q,
                "fields": ["title^2", "content", "author"]
            }
        },
        "size": size
    }
    result = await es.search(index="articles", body=body)
    hits = result["hits"]["hits"]
    return [{"id": hit["_id"], **hit["_source"]} for hit in hits]

# Aggregations
@app.get("/articles/stats")
async def article_stats():
    body = {
        "aggs": {
            "authors": {
                "terms": {"field": "author.keyword"}
            },
            "by_date": {
                "date_histogram": {
                    "field": "timestamp",
                    "calendar_interval": "day"
                }
            }
        }
    }
    result = await es.search(index="articles", body=body, size=0)
    return result["aggregations"]
```

### Cassandra

Distributed NoSQL database for high scalability.

```python
from cassandra.cluster import Cluster
from cassandra.auth import PlainTextAuthProvider
from fastapi import FastAPI

app = FastAPI()

# Cassandra connection
auth_provider = PlainTextAuthProvider(username='cassandra', password='cassandra')
cluster = Cluster(['127.0.0.1'], auth_provider=auth_provider)
session = cluster.connect()

# Create keyspace and table
session.execute("""
    CREATE KEYSPACE IF NOT EXISTS myapp
    WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 1}
""")

session.set_keyspace('myapp')

session.execute("""
    CREATE TABLE IF NOT EXISTS users (
        user_id UUID PRIMARY KEY,
        username TEXT,
        email TEXT,
        created_at TIMESTAMP
    )
""")

# CRUD operations
from uuid import uuid4
from datetime import datetime

@app.post("/users")
def create_user(username: str, email: str):
    user_id = uuid4()
    session.execute(
        """
        INSERT INTO users (user_id, username, email, created_at)
        VALUES (%s, %s, %s, %s)
        """,
        (user_id, username, email, datetime.now())
    )
    return {"user_id": str(user_id)}

@app.get("/users/{user_id}")
def get_user(user_id: str):
    from uuid import UUID
    rows = session.execute(
        "SELECT * FROM users WHERE user_id = %s",
        (UUID(user_id),)
    )
    row = rows.one()
    if row:
        return {
            "user_id": str(row.user_id),
            "username": row.username,
            "email": row.email,
            "created_at": row.created_at.isoformat()
        }
    raise HTTPException(status_code=404, detail="User not found")

@app.on_event("shutdown")
def shutdown():
    cluster.shutdown()
```

---

## Complete Example: FastAPI with Async SQLAlchemy and Alembic

```python
# File structure:
# app/
#   __init__.py
#   main.py
#   database.py
#   models.py
#   schemas.py
#   crud.py
#   config.py
# alembic/
#   versions/
#   env.py
# alembic.ini

# config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    DATABASE_URL: str = "postgresql+asyncpg://user:password@localhost/dbname"
    
    class Config:
        env_file = ".env"

settings = Settings()

# database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker, declarative_base

engine = create_async_engine(settings.DATABASE_URL, echo=True)
AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
Base = declarative_base()

async def get_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

# models.py
from sqlalchemy import Column, Integer, String, Boolean, DateTime, ForeignKey, Text
from sqlalchemy.orm import relationship
from datetime import datetime
from .database import Base

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    email = Column(String(100), unique=True, index=True)
    username = Column(String(50), unique=True, index=True)
    hashed_password = Column(String(255))
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    posts = relationship("Post", back_populates="author")

class Post(Base):
    __tablename__ = "posts"
    
    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(200))
    content = Column(Text)
    user_id = Column(Integer, ForeignKey("users.id"))
    created_at = Column(DateTime, default=datetime.utcnow)
    
    author = relationship("User", back_populates="posts")

# schemas.py
from pydantic import BaseModel, EmailStr
from datetime import datetime

class UserBase(BaseModel):
    email: EmailStr
    username: str

class UserCreate(UserBase):
    password: str

class UserResponse(UserBase):
    id: int
    is_active: bool
    created_at: datetime
    
    class Config:
        from_attributes = True

class PostCreate(BaseModel):
    title: str
    content: str

class PostResponse(BaseModel):
    id: int
    title: str
    content: str
    user_id: int
    created_at: datetime
    
    class Config:
        from_attributes = True

# crud.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from . import models, schemas

async def create_user(db: AsyncSession, user: schemas.UserCreate):
    db_user = models.User(
        email=user.email,
        username=user.username,
        hashed_password=user.password  # Hash in production!
    )
    db.add(db_user)
    await db.commit()
    await db.refresh(db_user)
    return db_user

async def get_users(db: AsyncSession, skip: int = 0, limit: int = 100):
    result = await db.execute(select(models.User).offset(skip).limit(limit))
    return result.scalars().all()

# main.py
from fastapi import FastAPI, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from . import crud, schemas
from .database import get_db

app = FastAPI()

@app.post("/users", response_model=schemas.UserResponse)
async def create_user(user: schemas.UserCreate, db: AsyncSession = Depends(get_db)):
    return await crud.create_user(db, user)

@app.get("/users", response_model=list[schemas.UserResponse])
async def read_users(skip: int = 0, limit: int = 100, db: AsyncSession = Depends(get_db)):
    return await crud.get_users(db, skip, limit)
```

---

**Key Takeaways:**
- Choose async for high I/O operations, sync for simpler applications
- Use connection pooling for better performance
- Implement proper session management with dependency injection
- Use Alembic for database migrations
- Consider SQLModel for seamless FastAPI integration
- Choose the right database (SQL vs NoSQL) based on your needs
- Always use proper error handling and transactions

**Additional Resources:**
- [SQLAlchemy Documentation](https://docs.sqlalchemy.org/)
- [Alembic Documentation](https://alembic.sqlalchemy.org/)
- [Tortoise ORM Documentation](https://tortoise.github.io/)
- [Motor Documentation](https://motor.readthedocs.io/)