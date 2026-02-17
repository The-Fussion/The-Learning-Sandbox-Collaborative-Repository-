# Performance & Optimization

## 16.1 Performance Basics

### FastAPI Performance Characteristics

FastAPI is one of the fastest Python frameworks available, built on two high-performance foundations:

- **Starlette**: ASGI framework handling the web layer
- **Pydantic**: Data validation and serialization

**Performance Benchmarks** (relative to other Python frameworks):
| Framework | Requests/sec | Notes |
|-----------|-------------|-------|
| FastAPI | ~50,000+ | ASGI, async-native |
| Flask | ~10,000 | WSGI, sync |
| Django | ~8,000 | WSGI, sync |
| Django REST | ~7,000 | WSGI, sync |

**FastAPI Performance Strengths**:
- **ASGI**: Handles thousands of concurrent connections
- **Async-native**: Non-blocking I/O by default
- **Pydantic v2**: Compiled validation (Rust-backed)
- **Auto-serialization**: Optimized JSON encoding
- **Minimal overhead**: Thin framework layer

**What Can Slow FastAPI Down**:
- Sync/blocking operations in async handlers
- N+1 database queries
- Unoptimized database queries
- No caching on repeated expensive operations
- Large uncompressed responses
- CPU-bound work on the main thread

### Benchmarking Tools

**wrk - HTTP Benchmarking**:
```bash
# Install
brew install wrk  # macOS
apt install wrk   # Linux

# Basic benchmark
wrk -t12 -c400 -d30s http://localhost:8000/

# Options:
# -t12: 12 threads
# -c400: 400 concurrent connections
# -d30s: 30 second duration

# With Lua script for POST requests
wrk -t4 -c100 -d10s -s post.lua http://localhost:8000/items/
```

**post.lua**:
```lua
wrk.method = "POST"
wrk.body   = '{"name": "item", "price": 10.0}'
wrk.headers["Content-Type"] = "application/json"
```

**ab (Apache Benchmark)**:
```bash
# Basic test - 1000 requests, 50 concurrent
ab -n 1000 -c 50 http://localhost:8000/

# With headers
ab -n 1000 -c 50 -H "Authorization: Bearer token" http://localhost:8000/protected/
```

**locust - Python Load Testing**:
```python
# locustfile.py
from locust import HttpUser, task, between

class APIUser(HttpUser):
    wait_time = between(1, 3)
    
    @task
    def get_items(self):
        self.client.get("/items/")
    
    @task(2)
    def get_single_item(self):
        self.client.get("/items/1")
```

**httpx for micro-benchmarking**:
```python
import httpx
import time
import asyncio

async def benchmark_endpoint(url: str, num_requests: int = 100):
    """Quick endpoint benchmark."""
    times = []
    
    async with httpx.AsyncClient() as client:
        for _ in range(num_requests):
            start = time.perf_counter()
            await client.get(url)
            times.append(time.perf_counter() - start)
    
    avg = sum(times) / len(times)
    p99 = sorted(times)[int(len(times) * 0.99)]
    
    print(f"Avg: {avg * 1000:.2f}ms")
    print(f"P99: {p99 * 1000:.2f}ms")
    print(f"Min: {min(times) * 1000:.2f}ms")
    print(f"Max: {max(times) * 1000:.2f}ms")

asyncio.run(benchmark_endpoint("http://localhost:8000/items/"))
```

### Performance Monitoring

**Prometheus + Grafana**:
```bash
pip install prometheus-fastapi-instrumentator
```

```python
from fastapi import FastAPI
from prometheus_fastapi_instrumentator import Instrumentator

app = FastAPI()

# Auto-instrument with Prometheus metrics
Instrumentator().instrument(app).expose(app)

# Exposes /metrics endpoint with:
# - http_requests_total
# - http_request_duration_seconds
# - http_requests_in_progress
```

**Custom Metrics Middleware**:
```python
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware
import time
from collections import defaultdict

app = FastAPI()

class MetricsMiddleware(BaseHTTPMiddleware):
    def __init__(self, app):
        super().__init__(app)
        self.request_count = defaultdict(int)
        self.request_times = defaultdict(list)
        self.error_count = defaultdict(int)
    
    async def dispatch(self, request: Request, call_next):
        start = time.perf_counter()
        route = f"{request.method} {request.url.path}"
        
        response = await call_next(request)
        
        duration = time.perf_counter() - start
        self.request_count[route] += 1
        self.request_times[route].append(duration)
        
        if response.status_code >= 400:
            self.error_count[route] += 1
        
        response.headers["X-Response-Time"] = f"{duration:.4f}"
        return response

metrics = MetricsMiddleware(app)
app.add_middleware(MetricsMiddleware)

@app.get("/metrics/summary")
async def metrics_summary():
    summary = {}
    for route, times in metrics.request_times.items():
        summary[route] = {
            "count": metrics.request_count[route],
            "avg_ms": sum(times) / len(times) * 1000,
            "errors": metrics.error_count[route]
        }
    return summary
```

### Identifying Bottlenecks

**Profiling with cProfile**:
```python
import cProfile
import pstats
import io

def profile_endpoint():
    pr = cProfile.Profile()
    pr.enable()
    
    # Run your code here
    result = your_function()
    
    pr.disable()
    
    s = io.StringIO()
    ps = pstats.Stats(pr, stream=s).sort_stats("cumulative")
    ps.print_stats(20)  # Top 20 functions
    
    print(s.getvalue())
    return result
```

**Pyinstrument for async profiling**:
```bash
pip install pyinstrument
```

```python
from pyinstrument import Profiler
from fastapi import FastAPI, Request

app = FastAPI()

@app.middleware("http")
async def profile_middleware(request: Request, call_next):
    if request.query_params.get("profile"):
        profiler = Profiler(async_mode="enabled")
        profiler.start()
        
        response = await call_next(request)
        
        profiler.stop()
        print(profiler.output_text(unicode=True, color=True))
        
        return response
    
    return await call_next(request)
```

---

## 16.2 Async Best Practices

### Async vs Sync Route Handlers

FastAPI handles sync and async functions differently. Choosing the wrong one can severely impact performance.

```python
from fastapi import FastAPI
import asyncio
import time

app = FastAPI()

# ✅ CORRECT: Async with async I/O
@app.get("/async-correct")
async def async_correct():
    await asyncio.sleep(1)  # Non-blocking
    return {"type": "async"}

# ❌ WRONG: Sync sleep in async handler - blocks event loop
@app.get("/async-wrong")
async def async_wrong():
    time.sleep(1)  # Blocks entire event loop!
    return {"type": "blocked"}

# ✅ CORRECT: Sync function for CPU-bound work
# FastAPI runs sync functions in threadpool automatically
@app.get("/sync-cpu")
def sync_cpu():
    result = heavy_computation()
    return {"result": result}
```

**Performance Impact**:
```python
import httpx
import asyncio
import time

# Blocking async: 10 concurrent requests take 10 seconds
# Non-blocking async: 10 concurrent requests take ~1 second

async def demo_blocking_vs_nonblocking():
    async with httpx.AsyncClient() as client:
        # 10 requests to blocking endpoint
        start = time.time()
        await asyncio.gather(*[
            client.get("http://localhost:8000/async-wrong")
            for _ in range(10)
        ])
        print(f"Blocking: {time.time() - start:.2f}s")  # ~10 seconds
        
        # 10 requests to non-blocking endpoint
        start = time.time()
        await asyncio.gather(*[
            client.get("http://localhost:8000/async-correct")
            for _ in range(10)
        ])
        print(f"Non-blocking: {time.time() - start:.2f}s")  # ~1 second
```

### Async Database Operations

```python
from fastapi import FastAPI, Depends
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy import select

# Async SQLAlchemy setup
DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/db"
engine = create_async_engine(DATABASE_URL, echo=False)
AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

app = FastAPI()

async def get_db():
    async with AsyncSessionLocal() as session:
        yield session

# ✅ Async database operations
@app.get("/items/")
async def read_items(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(Item))
    items = result.scalars().all()
    return items

# ✅ Async transactions
@app.post("/items/")
async def create_item(item: ItemCreate, db: AsyncSession = Depends(get_db)):
    db_item = Item(**item.dict())
    db.add(db_item)
    await db.commit()
    await db.refresh(db_item)
    return db_item

# ✅ Parallel async queries
@app.get("/dashboard/")
async def get_dashboard(db: AsyncSession = Depends(get_db)):
    # Run multiple queries concurrently
    users_q = db.execute(select(User).limit(10))
    items_q = db.execute(select(Item).limit(10))
    stats_q = db.execute(select(func.count(Order.id)))
    
    users_r, items_r, stats_r = await asyncio.gather(
        users_q, items_q, stats_q
    )
    
    return {
        "users": users_r.scalars().all(),
        "items": items_r.scalars().all(),
        "order_count": stats_r.scalar()
    }
```

### Async I/O Operations

```python
import aiofiles
import httpx
import asyncio
from fastapi import FastAPI

app = FastAPI()

# ✅ Async file operations
@app.get("/read-file/")
async def read_file(filename: str):
    async with aiofiles.open(f"files/{filename}", mode="r") as f:
        content = await f.read()
    return {"content": content}

@app.post("/write-file/")
async def write_file(filename: str, content: str):
    async with aiofiles.open(f"files/{filename}", mode="w") as f:
        await f.write(content)
    return {"message": "Written"}

# ✅ Async HTTP calls
@app.get("/external-data/")
async def fetch_external():
    async with httpx.AsyncClient() as client:
        # Run multiple requests concurrently
        responses = await asyncio.gather(
            client.get("https://api.example.com/users"),
            client.get("https://api.example.com/products"),
            client.get("https://api.example.com/orders"),
        )
    
    return {
        "users": responses[0].json(),
        "products": responses[1].json(),
        "orders": responses[2].json()
    }
```

### Avoiding Blocking Operations

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor
from fastapi import FastAPI

app = FastAPI()
thread_pool = ThreadPoolExecutor(max_workers=10)
process_pool = ProcessPoolExecutor(max_workers=4)

# ❌ WRONG: Blocking sync call in async handler
@app.get("/bad/")
async def bad_endpoint():
    result = slow_sync_function()  # Blocks event loop!
    return result

# ✅ CORRECT: Run blocking code in thread pool
@app.get("/good/thread/")
async def good_thread_endpoint():
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(
        thread_pool,
        slow_sync_function
    )
    return result

# ✅ CORRECT: CPU-bound work in process pool
@app.get("/good/process/")
async def good_process_endpoint():
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(
        process_pool,
        cpu_heavy_function
    )
    return result

# ✅ CORRECT: Use anyio for thread offloading
import anyio

@app.get("/anyio/")
async def anyio_endpoint():
    result = await anyio.to_thread.run_sync(slow_sync_function)
    return result
```

### CPU-Bound vs I/O-Bound Tasks

```python
from fastapi import FastAPI, BackgroundTasks
import asyncio
from concurrent.futures import ProcessPoolExecutor

app = FastAPI()

# I/O-bound: Use async/await
@app.get("/io-bound/")
async def io_bound_task():
    """Database queries, HTTP calls, file I/O - use async."""
    async with httpx.AsyncClient() as client:
        data = await client.get("https://api.example.com/data")
    return data.json()

# CPU-bound: Use process pool
def compute_fibonacci(n: int) -> int:
    """Pure computation - runs in process pool."""
    if n < 2:
        return n
    return compute_fibonacci(n - 1) + compute_fibonacci(n - 2)

process_pool = ProcessPoolExecutor()

@app.get("/cpu-bound/{n}")
async def cpu_bound_task(n: int):
    """Heavy computation - offload to process pool."""
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(process_pool, compute_fibonacci, n)
    return {"fibonacci": result}

# Mixed: Background task for CPU-heavy post-processing
@app.post("/process-image/")
async def process_image(
    file: UploadFile,
    background_tasks: BackgroundTasks
):
    content = await file.read()
    
    # Return immediately, process in background
    background_tasks.add_task(
        heavy_image_processing,
        content,
        file.filename
    )
    
    return {"status": "processing", "filename": file.filename}
```

---

## 16.3 Caching

### Response Caching

**Simple In-Memory Cache**:
```python
from fastapi import FastAPI
from functools import lru_cache
import time

app = FastAPI()

# Cache with TTL
class TTLCache:
    def __init__(self):
        self._cache = {}
        self._expiry = {}
    
    def get(self, key: str):
        if key in self._cache:
            if time.time() < self._expiry[key]:
                return self._cache[key]
            else:
                del self._cache[key]
                del self._expiry[key]
        return None
    
    def set(self, key: str, value, ttl: int = 60):
        self._cache[key] = value
        self._expiry[key] = time.time() + ttl

cache = TTLCache()

@app.get("/expensive/")
async def expensive_operation(query: str):
    cache_key = f"expensive:{query}"
    
    # Check cache first
    cached = cache.get(cache_key)
    if cached:
        return {**cached, "from_cache": True}
    
    # Expensive operation
    result = {"query": query, "data": "computed_value"}
    
    # Store in cache for 5 minutes
    cache.set(cache_key, result, ttl=300)
    
    return {**result, "from_cache": False}
```

### Redis Caching

```bash
pip install redis aioredis
```

```python
from fastapi import FastAPI, Depends
import aioredis
import json

app = FastAPI()

# Redis connection
async def get_redis():
    redis = await aioredis.create_redis_pool("redis://localhost")
    try:
        yield redis
    finally:
        redis.close()
        await redis.wait_closed()

# Cache decorator
def cache_response(ttl: int = 60, key_prefix: str = ""):
    def decorator(func):
        async def wrapper(*args, redis=None, **kwargs):
            # Build cache key
            cache_key = f"{key_prefix}:{func.__name__}:{str(kwargs)}"
            
            # Try cache first
            if redis:
                cached = await redis.get(cache_key)
                if cached:
                    return json.loads(cached)
            
            # Call function
            result = await func(*args, **kwargs)
            
            # Store in cache
            if redis:
                await redis.setex(
                    cache_key,
                    ttl,
                    json.dumps(result)
                )
            
            return result
        return wrapper
    return decorator

@app.get("/users/{user_id}")
async def get_user(user_id: int, redis=Depends(get_redis)):
    cache_key = f"user:{user_id}"
    
    # Check Redis cache
    cached = await redis.get(cache_key)
    if cached:
        return {**json.loads(cached), "cached": True}
    
    # Query database (simulated)
    user = {"id": user_id, "name": "Alice", "email": "alice@example.com"}
    
    # Cache for 5 minutes
    await redis.setex(cache_key, 300, json.dumps(user))
    
    return {**user, "cached": False}

@app.put("/users/{user_id}")
async def update_user(user_id: int, data: dict, redis=Depends(get_redis)):
    # Update database (simulated)
    updated_user = {"id": user_id, **data}
    
    # Invalidate cache
    await redis.delete(f"user:{user_id}")
    
    return updated_user
```

**Redis for Rate Limiting + Caching**:
```python
from fastapi import FastAPI, Request, HTTPException
import aioredis
import time

app = FastAPI()
redis = None

@app.on_event("startup")
async def startup():
    global redis
    redis = await aioredis.create_redis_pool("redis://localhost")

@app.on_event("shutdown")
async def shutdown():
    redis.close()
    await redis.wait_closed()

@app.get("/api/data/")
async def get_data(request: Request):
    client_ip = request.client.host
    rate_key = f"rate:{client_ip}"
    
    # Rate limit: 100 requests per minute
    count = await redis.incr(rate_key)
    if count == 1:
        await redis.expire(rate_key, 60)
    
    if count > 100:
        raise HTTPException(status_code=429, detail="Rate limit exceeded")
    
    # Check response cache
    cache_key = "global:data"
    cached = await redis.get(cache_key)
    if cached:
        return json.loads(cached)
    
    # Compute result
    result = {"data": "expensive_computation_result"}
    await redis.setex(cache_key, 60, json.dumps(result))
    
    return result
```

### In-Memory Caching

**Using cachetools**:
```bash
pip install cachetools
```

```python
from cachetools import TTLCache, LRUCache, cached
from cachetools.keys import hashkey
from fastapi import FastAPI
import asyncio

app = FastAPI()

# TTL Cache: Items expire after N seconds
ttl_cache = TTLCache(maxsize=1000, ttl=300)

# LRU Cache: Evicts least recently used items
lru_cache = LRUCache(maxsize=500)

# Async-safe caching
import asyncio
from threading import RLock

cache_lock = RLock()

@app.get("/cached-data/{key}")
async def get_cached_data(key: str):
    cache_key = hashkey(key)
    
    with cache_lock:
        if cache_key in ttl_cache:
            return {"data": ttl_cache[cache_key], "cached": True}
    
    # Expensive operation
    await asyncio.sleep(0.1)
    result = f"data_for_{key}"
    
    with cache_lock:
        ttl_cache[cache_key] = result
    
    return {"data": result, "cached": False}
```

**functools.lru_cache for Settings**:
```python
from functools import lru_cache
from pydantic import BaseSettings

class Settings(BaseSettings):
    database_url: str
    redis_url: str
    secret_key: str

@lru_cache()
def get_settings():
    """Cached settings - only loaded from env once."""
    return Settings()

@app.get("/config/")
async def get_config(settings: Settings = Depends(get_settings)):
    return {"db": settings.database_url}
```

### Cache Invalidation

```python
from fastapi import FastAPI, Depends
import aioredis

app = FastAPI()

class CacheManager:
    def __init__(self, redis):
        self.redis = redis
    
    async def invalidate(self, *keys: str):
        """Invalidate specific cache keys."""
        if keys:
            await self.redis.delete(*keys)
    
    async def invalidate_pattern(self, pattern: str):
        """Invalidate all keys matching pattern."""
        keys = await self.redis.keys(pattern)
        if keys:
            await self.redis.delete(*keys)
    
    async def set(self, key: str, value, ttl: int = 300):
        await self.redis.setex(key, ttl, json.dumps(value))
    
    async def get(self, key: str):
        data = await self.redis.get(key)
        return json.loads(data) if data else None

@app.put("/users/{user_id}")
async def update_user(user_id: int, data: dict, redis=Depends(get_redis)):
    cache = CacheManager(redis)
    
    # Update database (simulated)
    updated = {"id": user_id, **data}
    
    # Invalidate related caches
    await cache.invalidate(
        f"user:{user_id}",
        f"user_profile:{user_id}",
        "users:list"
    )
    
    # Also clear any search results
    await cache.invalidate_pattern("search:*")
    
    return updated

@app.delete("/categories/{category_id}")
async def delete_category(category_id: int, redis=Depends(get_redis)):
    cache = CacheManager(redis)
    
    # Invalidate all items in this category
    await cache.invalidate_pattern(f"category:{category_id}:*")
    await cache.invalidate(f"category:{category_id}")
    
    return {"deleted": category_id}
```

### Cache-Control Headers

```python
from fastapi import FastAPI, Response

app = FastAPI()

@app.get("/static-data/")
async def get_static_data(response: Response):
    """Cacheable for 1 hour by browsers and CDNs."""
    response.headers["Cache-Control"] = "public, max-age=3600"
    return {"data": "rarely_changes"}

@app.get("/user-data/")
async def get_user_data(response: Response):
    """Private - only browser can cache, not CDN."""
    response.headers["Cache-Control"] = "private, max-age=300"
    return {"data": "user_specific"}

@app.get("/real-time/")
async def get_realtime_data(response: Response):
    """Never cache."""
    response.headers["Cache-Control"] = "no-store, no-cache, must-revalidate"
    return {"data": "always_fresh"}

@app.get("/products/")
async def get_products(response: Response):
    """Cache for 10 minutes, allow stale for 1 hour while revalidating."""
    response.headers["Cache-Control"] = (
        "public, max-age=600, stale-while-revalidate=3600"
    )
    return {"products": []}
```

### ETags

```python
import hashlib
import json
from fastapi import FastAPI, Request, Response

app = FastAPI()

def generate_etag(data) -> str:
    """Generate ETag from data."""
    content = json.dumps(data, sort_keys=True)
    return hashlib.md5(content.encode()).hexdigest()

@app.get("/products/{product_id}")
async def get_product(product_id: int, request: Request, response: Response):
    # Get product data
    product = {"id": product_id, "name": "Widget", "price": 9.99}
    
    # Generate ETag
    etag = generate_etag(product)
    
    # Check if client has current version
    if request.headers.get("If-None-Match") == etag:
        return Response(status_code=304)  # Not Modified
    
    # Set ETag header
    response.headers["ETag"] = etag
    response.headers["Cache-Control"] = "public, max-age=300"
    
    return product
```

---

## 16.4 Database Optimization

### Query Optimization

```python
from sqlalchemy import select, func
from sqlalchemy.ext.asyncio import AsyncSession

# ❌ SLOW: Loading all records then filtering in Python
@app.get("/items/bad/")
async def get_items_bad(db: AsyncSession = Depends(get_db)):
    all_items = await db.execute(select(Item))
    items = all_items.scalars().all()
    return [i for i in items if i.price > 10]  # Python filtering!

# ✅ FAST: Filter in database
@app.get("/items/good/")
async def get_items_good(db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        select(Item)
        .where(Item.price > 10)     # Database filtering
        .order_by(Item.price)       # Database sorting
        .limit(100)                 # Database limiting
    )
    return result.scalars().all()

# ✅ FAST: Aggregations in database
@app.get("/stats/")
async def get_stats(db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        select(
            func.count(Item.id).label("total"),
            func.avg(Item.price).label("avg_price"),
            func.max(Item.price).label("max_price")
        )
    )
    return dict(result.first())

# ✅ Only select needed columns
@app.get("/item-names/")
async def get_item_names(db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        select(Item.id, Item.name)  # Only get what you need
    )
    return [{"id": r.id, "name": r.name} for r in result]
```

### Eager Loading vs Lazy Loading

```python
from sqlalchemy.orm import selectinload, joinedload, subqueryload

# ❌ N+1 Problem: Lazy loading
@app.get("/users-lazy/")
async def get_users_lazy(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User))
    users = result.scalars().all()
    
    # This triggers N additional queries - one per user!
    return [
        {
            "id": u.id,
            "name": u.name,
            "posts": u.posts  # Lazy load = extra query per user
        }
        for u in users
    ]

# ✅ Eager loading with selectinload (for one-to-many)
@app.get("/users-eager/")
async def get_users_eager(db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        select(User).options(selectinload(User.posts))
    )
    users = result.scalars().all()
    
    # posts already loaded - no extra queries
    return [{"id": u.id, "name": u.name, "posts": u.posts} for u in users]

# ✅ joinedload for many-to-one (load with JOIN)
@app.get("/posts/")
async def get_posts(db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        select(Post).options(joinedload(Post.author))
    )
    return result.scalars().all()

# ✅ Load multiple relationships
@app.get("/users-full/")
async def get_users_full(db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        select(User).options(
            selectinload(User.posts).selectinload(Post.comments),
            selectinload(User.profile)
        )
    )
    return result.scalars().all()
```

### Database Indexing

```python
from sqlalchemy import Column, Integer, String, Index
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class Product(Base):
    __tablename__ = "products"
    
    id = Column(Integer, primary_key=True)  # Auto-indexed
    name = Column(String, index=True)        # Simple index
    sku = Column(String, unique=True)        # Unique index
    category_id = Column(Integer, index=True) # Foreign key index
    price = Column(Float)
    description = Column(Text)
    
    # Composite index for common query patterns
    __table_args__ = (
        Index("ix_product_category_price", "category_id", "price"),
        Index("ix_product_name_search", "name"),
    )

# In Alembic migration:
"""
def upgrade():
    op.create_index(
        "ix_orders_user_created",
        "orders",
        ["user_id", "created_at"]
    )
    op.create_index(
        "ix_products_search",
        "products",
        ["name", "category_id", "price"]
    )
"""

# Index usage example - these queries will use indexes
@app.get("/products/")
async def search_products(
    category_id: int,
    min_price: float,
    db: AsyncSession = Depends(get_db)
):
    # Uses ix_product_category_price composite index
    result = await db.execute(
        select(Product)
        .where(Product.category_id == category_id)
        .where(Product.price >= min_price)
        .order_by(Product.price)
    )
    return result.scalars().all()
```

### Connection Pooling

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

# Optimized connection pool configuration
engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/db",
    
    # Pool configuration
    pool_size=20,           # Maintain 20 connections
    max_overflow=10,        # Allow 10 extra connections at peak
    pool_timeout=30,        # Wait max 30s for connection
    pool_recycle=3600,      # Recycle connections after 1 hour
    pool_pre_ping=True,     # Verify connections before use
    
    # Performance
    echo=False,             # Disable SQL logging in production
)

AsyncSessionLocal = sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False  # Don't expire objects after commit
)

# Health check for connection pool
@app.get("/db/health")
async def db_health():
    pool = engine.pool
    return {
        "pool_size": pool.size(),
        "checked_in": pool.checkedin(),
        "checked_out": pool.checkedout(),
        "overflow": pool.overflow()
    }
```

### N+1 Query Problem

```python
# ❌ N+1 Problem - 1 query for orders + N queries for users
@app.get("/orders-n1/")
async def get_orders_n1(db: AsyncSession = Depends(get_db)):
    orders = (await db.execute(select(Order))).scalars().all()
    
    result = []
    for order in orders:
        # This fires a separate query for EACH order!
        user = await db.get(User, order.user_id)
        result.append({
            "order_id": order.id,
            "user": user.name  # Extra query per order
        })
    
    return result  # 1 + N queries!

# ✅ Solved: 2 queries total
@app.get("/orders-solved/")
async def get_orders_solved(db: AsyncSession = Depends(get_db)):
    # Load orders with users in 2 queries
    result = await db.execute(
        select(Order).options(selectinload(Order.user))
    )
    orders = result.scalars().all()
    
    return [
        {"order_id": o.id, "user": o.user.name}
        for o in orders
    ]  # Always 2 queries

# ✅ Solved: JOIN (1 query)
@app.get("/orders-join/")
async def get_orders_join(db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        select(Order, User)
        .join(User, Order.user_id == User.id)
    )
    
    return [
        {"order_id": o.id, "user": u.name}
        for o, u in result
    ]  # 1 query
```

---

## 16.5 Response Optimization

### Response Compression (GZip)

```python
from fastapi import FastAPI
from fastapi.middleware.gzip import GZipMiddleware

app = FastAPI()

# Automatic compression for responses > 1000 bytes
app.add_middleware(GZipMiddleware, minimum_size=1000)

@app.get("/large-dataset/")
async def large_dataset():
    # This will be automatically compressed
    return {"items": [{"id": i, "name": f"Item {i}"} for i in range(10000)]}
```

### Pagination

```python
from fastapi import FastAPI, Query, Depends
from pydantic import BaseModel
from typing import List, Generic, TypeVar

app = FastAPI()

T = TypeVar("T")

class PaginatedResponse(BaseModel, Generic[T]):
    items: List[T]
    total: int
    page: int
    pages: int
    has_next: bool
    has_prev: bool

class PaginationParams:
    def __init__(
        self,
        page: int = Query(1, ge=1),
        page_size: int = Query(20, ge=1, le=100)
    ):
        self.page = page
        self.page_size = page_size
        self.offset = (page - 1) * page_size

@app.get("/items/", response_model=PaginatedResponse)
async def list_items(
    pagination: PaginationParams = Depends(),
    db: AsyncSession = Depends(get_db)
):
    # Count total
    total = (await db.execute(select(func.count(Item.id)))).scalar()
    
    # Get page
    items = (await db.execute(
        select(Item)
        .offset(pagination.offset)
        .limit(pagination.page_size)
    )).scalars().all()
    
    total_pages = (total + pagination.page_size - 1) // pagination.page_size
    
    return PaginatedResponse(
        items=items,
        total=total,
        page=pagination.page,
        pages=total_pages,
        has_next=pagination.page < total_pages,
        has_prev=pagination.page > 1
    )
```

**Cursor-Based Pagination** (better for large datasets):
```python
@app.get("/items/cursor/")
async def list_items_cursor(
    cursor: int = Query(None, description="Last item ID from previous page"),
    limit: int = Query(20, ge=1, le=100),
    db: AsyncSession = Depends(get_db)
):
    query = select(Item).order_by(Item.id).limit(limit + 1)
    
    if cursor:
        query = query.where(Item.id > cursor)
    
    items = (await db.execute(query)).scalars().all()
    
    has_more = len(items) > limit
    if has_more:
        items = items[:-1]
    
    next_cursor = items[-1].id if has_more and items else None
    
    return {
        "items": items,
        "next_cursor": next_cursor,
        "has_more": has_more
    }
```

### Field Filtering

```python
from fastapi import FastAPI, Query
from typing import Optional, Set

app = FastAPI()

@app.get("/users/{user_id}")
async def get_user(
    user_id: int,
    fields: Optional[str] = Query(None, description="Comma-separated fields"),
    db: AsyncSession = Depends(get_db)
):
    user = await db.get(User, user_id)
    
    if not user:
        raise HTTPException(status_code=404)
    
    user_dict = {
        "id": user.id,
        "username": user.username,
        "email": user.email,
        "created_at": user.created_at,
        "profile": user.profile,
        "settings": user.settings
    }
    
    # Filter fields if requested
    if fields:
        requested = set(fields.split(","))
        user_dict = {k: v for k, v in user_dict.items() if k in requested}
    
    return user_dict

# URL: /users/1?fields=id,username,email
# Returns only id, username, email
```

### Response Streaming

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import asyncio
import json

app = FastAPI()

async def generate_large_dataset():
    """Stream large dataset without loading into memory."""
    yield "["
    
    for i in range(100000):
        if i > 0:
            yield ","
        
        item = {"id": i, "name": f"Item {i}", "value": i * 2}
        yield json.dumps(item)
        
        # Yield control periodically
        if i % 1000 == 0:
            await asyncio.sleep(0)  # Allow other tasks to run
    
    yield "]"

@app.get("/stream/large-dataset/")
async def stream_dataset():
    return StreamingResponse(
        generate_large_dataset(),
        media_type="application/json"
    )

# File streaming
@app.get("/stream/file/{filename}")
async def stream_file(filename: str):
    async def file_generator():
        async with aiofiles.open(f"files/{filename}", "rb") as f:
            while chunk := await f.read(65536):  # 64KB chunks
                yield chunk
    
    return StreamingResponse(
        file_generator(),
        media_type="application/octet-stream",
        headers={"Content-Disposition": f"attachment; filename={filename}"}
    )
```

### Partial Responses

```python
from fastapi import FastAPI, Request, Response
from fastapi.responses import StreamingResponse
import aiofiles
import os

app = FastAPI()

@app.get("/video/{filename}")
async def stream_video(filename: str, request: Request):
    filepath = f"videos/{filename}"
    file_size = os.path.getsize(filepath)
    
    # Handle Range request (for video seeking)
    range_header = request.headers.get("Range")
    
    if range_header:
        start, end = range_header.replace("bytes=", "").split("-")
        start = int(start)
        end = int(end) if end else file_size - 1
        content_length = end - start + 1
        
        async def ranged_file():
            async with aiofiles.open(filepath, "rb") as f:
                await f.seek(start)
                remaining = content_length
                while remaining > 0:
                    chunk_size = min(65536, remaining)
                    chunk = await f.read(chunk_size)
                    if not chunk:
                        break
                    remaining -= len(chunk)
                    yield chunk
        
        return StreamingResponse(
            ranged_file(),
            status_code=206,
            media_type="video/mp4",
            headers={
                "Content-Range": f"bytes {start}-{end}/{file_size}",
                "Content-Length": str(content_length),
                "Accept-Ranges": "bytes"
            }
        )
    
    # Full file response
    async def full_file():
        async with aiofiles.open(filepath, "rb") as f:
            while chunk := await f.read(65536):
                yield chunk
    
    return StreamingResponse(
        full_file(),
        media_type="video/mp4",
        headers={
            "Content-Length": str(file_size),
            "Accept-Ranges": "bytes"
        }
    )
```

---

## 16.6 Load Balancing

### Horizontal Scaling

**Running Multiple Uvicorn Workers**:
```bash
# Multiple workers with Gunicorn
pip install gunicorn uvicorn[standard]

# Start with 4 worker processes
gunicorn app.main:app \
    -w 4 \
    -k uvicorn.workers.UvicornWorker \
    --bind 0.0.0.0:8000

# Or with uvicorn (single process, async concurrency)
uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4
```

**Docker Deployment**:
```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Run with multiple workers
CMD ["gunicorn", "app.main:app", 
     "-w", "4", 
     "-k", "uvicorn.workers.UvicornWorker",
     "--bind", "0.0.0.0:8000",
     "--timeout", "120",
     "--keep-alive", "5"]
```

**docker-compose.yml for Scaling**:
```yaml
version: "3.8"

services:
  api:
    build: .
    deploy:
      replicas: 4          # 4 API instances
    environment:
      - DATABASE_URL=postgresql://user:pass@db/app
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
  
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - api
  
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: app
    volumes:
      - postgres_data:/var/lib/postgresql/data
  
  redis:
    image: redis:7-alpine

volumes:
  postgres_data:
```

### Load Balancer Configuration

**nginx.conf**:
```nginx
upstream fastapi_backend {
    # Default: round-robin load balancing
    server api:8000;
    server api:8001;
    server api:8002;
    server api:8003;
    
    # Or with weights
    # server api1:8000 weight=3;
    # server api2:8000 weight=1;
    
    # Health checks
    keepalive 32;
}

server {
    listen 80;
    server_name example.com;
    
    # Gzip compression at proxy level
    gzip on;
    gzip_types application/json text/plain;
    gzip_min_length 1000;
    
    location / {
        proxy_pass http://fastapi_backend;
        proxy_http_version 1.1;
        
        # Pass real client info
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket support
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
    
    # Cache static responses at nginx level
    location /static/ {
        proxy_pass http://fastapi_backend;
        proxy_cache_valid 200 1h;
        add_header X-Cache-Status $upstream_cache_status;
    }
}
```

### Session Management in Scaled Apps

```python
from fastapi import FastAPI, Depends, HTTPException
import aioredis
import jwt
import json
from datetime import datetime, timedelta

app = FastAPI()

SECRET_KEY = "your-secret-key"

# Shared Redis session store (works across all instances)
async def get_redis():
    return await aioredis.create_redis_pool("redis://redis:6379")

class SessionManager:
    def __init__(self, redis):
        self.redis = redis
    
    async def create_session(self, user_id: int, data: dict) -> str:
        """Create session stored in Redis (shared across instances)."""
        session_id = str(uuid.uuid4())
        session_data = {"user_id": user_id, **data}
        
        await self.redis.setex(
            f"session:{session_id}",
            3600,  # 1 hour TTL
            json.dumps(session_data)
        )
        
        return session_id
    
    async def get_session(self, session_id: str) -> dict:
        """Retrieve session from Redis - works on any instance."""
        data = await self.redis.get(f"session:{session_id}")
        if not data:
            return None
        return json.loads(data)
    
    async def delete_session(self, session_id: str):
        await self.redis.delete(f"session:{session_id}")

@app.post("/login/")
async def login(username: str, password: str, redis=Depends(get_redis)):
    # Validate credentials (simplified)
    if not validate_credentials(username, password):
        raise HTTPException(status_code=401)
    
    manager = SessionManager(redis)
    session_id = await manager.create_session(
        user_id=1,
        data={"username": username, "role": "user"}
    )
    
    return {"session_id": session_id}

@app.get("/profile/")
async def get_profile(session_id: str, redis=Depends(get_redis)):
    """Works regardless of which API instance handles this request."""
    manager = SessionManager(redis)
    session = await manager.get_session(session_id)
    
    if not session:
        raise HTTPException(status_code=401, detail="Session expired")
    
    return session
```

### Sticky Sessions

```nginx
# nginx.conf - Sticky sessions by IP
upstream fastapi_backend {
    ip_hash;  # Same IP always goes to same server
    
    server api1:8000;
    server api2:8000;
    server api3:8000;
}
```

```python
# JWT-based stateless sessions (preferred - no stickiness needed)
from fastapi import FastAPI, Depends
from fastapi.security import HTTPBearer
import jwt

app = FastAPI()
security = HTTPBearer()

def create_jwt_token(user_id: int, data: dict) -> str:
    """All state lives in the token - no server-side storage."""
    payload = {
        "sub": str(user_id),
        "exp": datetime.utcnow() + timedelta(hours=1),
        **data
    }
    return jwt.encode(payload, SECRET_KEY, algorithm="HS256")

def verify_jwt_token(credentials=Depends(security)) -> dict:
    """Verify token on ANY instance - no shared state needed."""
    try:
        return jwt.decode(
            credentials.credentials,
            SECRET_KEY,
            algorithms=["HS256"]
        )
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.get("/profile/")
async def get_profile(payload: dict = Depends(verify_jwt_token)):
    """Works on any server instance - JWT is self-contained."""
    return {"user_id": payload["sub"]}
```

---

## Complete Example: Optimized Product Catalog API

```python
from fastapi import FastAPI, Depends, Query, Request, Response
from fastapi.middleware.gzip import GZipMiddleware
from starlette.middleware.base import BaseHTTPMiddleware
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker, selectinload
from sqlalchemy import select, func
import aioredis
import json
import time
import hashlib

app = FastAPI(title="Optimized Product Catalog")

# ── Middleware ──────────────────────────────────────────────
app.add_middleware(GZipMiddleware, minimum_size=1000)

class TimingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        start = time.perf_counter()
        response = await call_next(request)
        response.headers["X-Response-Time"] = f"{(time.perf_counter()-start)*1000:.2f}ms"
        return response

app.add_middleware(TimingMiddleware)

# ── Database ────────────────────────────────────────────────
engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/catalog",
    pool_size=20, max_overflow=10, pool_pre_ping=True
)
AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_db():
    async with AsyncSessionLocal() as session:
        yield session

# ── Redis ───────────────────────────────────────────────────
redis_pool = None

@app.on_event("startup")
async def startup():
    global redis_pool
    redis_pool = await aioredis.create_redis_pool("redis://localhost")

@app.on_event("shutdown")
async def shutdown():
    redis_pool.close()
    await redis_pool.wait_closed()

async def get_redis():
    return redis_pool

# ── Product List with Caching + Pagination ──────────────────
@app.get("/products/")
async def list_products(
    page: int = Query(1, ge=1),
    size: int = Query(20, ge=1, le=100),
    category_id: int = Query(None),
    min_price: float = Query(None, ge=0),
    response: Response = None,
    db: AsyncSession = Depends(get_db),
    redis = Depends(get_redis)
):
    # Build cache key from all params
    cache_key = f"products:{page}:{size}:{category_id}:{min_price}"
    
    # Check cache
    cached = await redis.get(cache_key)
    if cached:
        response.headers["X-Cache"] = "HIT"
        return json.loads(cached)
    
    # Build optimized query
    query = select(Product).options(selectinload(Product.category))
    count_query = select(func.count(Product.id))
    
    if category_id:
        query = query.where(Product.category_id == category_id)
        count_query = count_query.where(Product.category_id == category_id)
    
    if min_price is not None:
        query = query.where(Product.price >= min_price)
        count_query = count_query.where(Product.price >= min_price)
    
    # Execute count and data in parallel
    total, products = await asyncio.gather(
        db.execute(count_query),
        db.execute(query.offset((page - 1) * size).limit(size))
    )
    
    total = total.scalar()
    products = products.scalars().all()
    
    result = {
        "items": [p.dict() for p in products],
        "total": total,
        "page": page,
        "pages": (total + size - 1) // size
    }
    
    # Cache for 5 minutes
    await redis.setex(cache_key, 300, json.dumps(result))
    
    response.headers["X-Cache"] = "MISS"
    response.headers["Cache-Control"] = "public, max-age=300"
    return result

# ── Single Product with ETag ────────────────────────────────
@app.get("/products/{product_id}")
async def get_product(
    product_id: int,
    request: Request,
    response: Response,
    db: AsyncSession = Depends(get_db),
    redis = Depends(get_redis)
):
    cache_key = f"product:{product_id}"
    
    cached = await redis.get(cache_key)
    if cached:
        product = json.loads(cached)
    else:
        result = await db.execute(
            select(Product)
            .options(selectinload(Product.category), selectinload(Product.reviews))
            .where(Product.id == product_id)
        )
        product_obj = result.scalar_one_or_none()
        
        if not product_obj:
            raise HTTPException(status_code=404)
        
        product = product_obj.dict()
        await redis.setex(cache_key, 300, json.dumps(product))
    
    # ETag support
    etag = hashlib.md5(json.dumps(product, sort_keys=True).encode()).hexdigest()
    
    if request.headers.get("If-None-Match") == etag:
        return Response(status_code=304)
    
    response.headers["ETag"] = etag
    response.headers["Cache-Control"] = "public, max-age=300"
    return product

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=8000,
        workers=4,
        loop="uvloop",     # Faster event loop
        http="httptools"   # Faster HTTP parser
    )
```

---

## Summary

You've learned about Performance & Optimization in FastAPI:

1. **Performance Basics**: Benchmarking with wrk/ab/locust, monitoring with Prometheus, profiling to find bottlenecks
2. **Async Best Practices**: Async vs sync handlers, async database/I/O operations, avoiding blocking calls, offloading CPU-bound work
3. **Caching**: In-memory TTL caches, Redis caching, cache invalidation strategies, Cache-Control headers, ETags
4. **Database Optimization**: Query optimization, eager vs lazy loading, strategic indexing, connection pooling, solving N+1 queries
5. **Response Optimization**: GZip compression, offset and cursor pagination, field filtering, response streaming, partial responses
6. **Load Balancing**: Horizontal scaling with Gunicorn, nginx load balancing, shared Redis sessions, stateless JWT tokens

**Quick Performance Wins**:
- Add `GZipMiddleware` for large responses
- Use `selectinload` to eliminate N+1 queries
- Cache expensive queries with Redis
- Add database indexes to frequently queried columns
- Use `async` database drivers (asyncpg, motor)
- Use cursor-based pagination for large datasets
- Keep sessions stateless (JWT) for seamless horizontal scaling