# Caching Strategies in Python

## Table of Contents
1. [Introduction to Caching](#introduction-to-caching)
2. [Types of Caching](#types-of-caching)
3. [Redis Caching](#redis-caching)
4. [Memcached](#memcached)
5. [Application-Level Caching](#application-level-caching)
6. [Database Query Caching](#database-query-caching)
7. [HTTP Caching](#http-caching)
8. [CDN Caching](#cdn-caching)
9. [Cache Invalidation](#cache-invalidation)
10. [Caching Patterns](#caching-patterns)
11. [Query Optimization](#query-optimization)
12. [Performance Monitoring](#performance-monitoring)
13. [Cache Strategies](#cache-strategies)
14. [Best Practices](#best-practices)

---

## Introduction to Caching

### What is Caching?

```python
"""
Caching is the process of storing copies of data in a temporary storage
location (cache) to enable faster access on subsequent requests.

Benefits:
1. Reduced Latency - Faster response times
2. Reduced Load - Less work for servers/databases
3. Cost Savings - Lower infrastructure costs
4. Better UX - Improved user experience
5. Scalability - Handle more requests

When to Use Caching:
✅ Expensive computations
✅ Frequent database queries
✅ API responses
✅ Static content
✅ Session data
✅ Configuration data

When NOT to Use Caching:
❌ Highly dynamic data
❌ Personal/sensitive data (use with care)
❌ Data that must be real-time
❌ Small, fast operations
"""
```

### Cache Hit vs Cache Miss

```python
"""
Cache Hit: Data found in cache
┌──────────┐      ┌─────────┐
│ Request  │─────>│  Cache  │
└──────────┘      └────┬────┘
                       │ HIT!
                       ▼
                  ┌─────────┐
                  │ Return  │
                  │  Data   │
                  └─────────┘

Cache Miss: Data NOT in cache
┌──────────┐      ┌─────────┐
│ Request  │─────>│  Cache  │
└──────────┘      └────┬────┘
                       │ MISS
                       ▼
                  ┌─────────┐
                  │Database │
                  └────┬────┘
                       │
                       ▼
                  ┌─────────┐
                  │  Cache  │ ← Store in cache
                  └────┬────┘
                       │
                       ▼
                  ┌─────────┐
                  │ Return  │
                  │  Data   │
                  └─────────┘

Cache Hit Ratio = Hits / (Hits + Misses)
Goal: Maximize cache hit ratio
"""
```

---

## Types of Caching

### Caching Layers

```python
"""
1. Client-Side Caching
   - Browser cache
   - HTTP caching headers
   - Service workers
   - LocalStorage/SessionStorage

2. CDN Caching
   - Edge locations
   - Static assets
   - Geographic distribution

3. Application Caching
   - In-memory cache
   - Redis/Memcached
   - Query results
   - Computed values

4. Database Caching
   - Query cache
   - Buffer pool
   - Materialized views

5. CPU Caching
   - L1, L2, L3 cache
   - Hardware level

Cache Hierarchy (from fastest to slowest):
CPU Cache → Memory Cache → Disk Cache → Network
"""
```

---

## Redis Caching

### Basic Redis Setup

```python
# Install redis
# pip install redis

import redis
import json
from functools import wraps
from typing import Any, Optional
import hashlib

# Connect to Redis
redis_client = redis.Redis(
    host='localhost',
    port=6379,
    db=0,
    decode_responses=True  # Automatically decode responses to strings
)

# Test connection
try:
    redis_client.ping()
    print("Connected to Redis!")
except redis.ConnectionError:
    print("Failed to connect to Redis")

# Basic operations
def redis_basic_operations():
    """Demonstrate basic Redis operations."""
    
    # String operations
    redis_client.set('key', 'value')
    value = redis_client.get('key')
    print(f"Value: {value}")
    
    # Set with expiration (TTL in seconds)
    redis_client.setex('temp_key', 300, 'temporary_value')  # 5 minutes
    
    # Check TTL
    ttl = redis_client.ttl('temp_key')
    print(f"TTL: {ttl} seconds")
    
    # Delete key
    redis_client.delete('key')
    
    # Check existence
    exists = redis_client.exists('key')
    print(f"Exists: {exists}")
    
    # Increment/Decrement
    redis_client.set('counter', 0)
    redis_client.incr('counter')  # Increment by 1
    redis_client.incrby('counter', 5)  # Increment by 5
    redis_client.decr('counter')  # Decrement by 1
    
    # Hash operations
    redis_client.hset('user:1', 'name', 'John Doe')
    redis_client.hset('user:1', 'email', 'john@example.com')
    user = redis_client.hgetall('user:1')
    print(f"User: {user}")
    
    # List operations
    redis_client.lpush('queue', 'item1', 'item2', 'item3')
    item = redis_client.rpop('queue')
    print(f"Popped: {item}")
    
    # Set operations
    redis_client.sadd('tags', 'python', 'redis', 'caching')
    members = redis_client.smembers('tags')
    print(f"Tags: {members}")
    
    # Sorted set operations
    redis_client.zadd('leaderboard', {'player1': 100, 'player2': 200})
    top_players = redis_client.zrevrange('leaderboard', 0, 9, withscores=True)
    print(f"Top players: {top_players}")
```

### Redis Cache Decorator

```python
import time
from functools import wraps
import json
import hashlib

class RedisCache:
    """Redis cache manager."""
    
    def __init__(self, redis_client, default_ttl=300):
        self.redis_client = redis_client
        self.default_ttl = default_ttl
    
    def _generate_key(self, prefix, *args, **kwargs):
        """Generate cache key from function arguments."""
        # Create a string from args and kwargs
        key_data = f"{prefix}:{args}:{sorted(kwargs.items())}"
        # Hash it to create a consistent key
        key_hash = hashlib.md5(key_data.encode()).hexdigest()
        return f"{prefix}:{key_hash}"
    
    def cache(self, ttl=None, prefix='cache'):
        """Decorator to cache function results."""
        def decorator(func):
            @wraps(func)
            def wrapper(*args, **kwargs):
                # Generate cache key
                cache_key = self._generate_key(prefix, *args, **kwargs)
                
                # Try to get from cache
                cached_value = self.redis_client.get(cache_key)
                if cached_value is not None:
                    print(f"Cache HIT: {cache_key}")
                    return json.loads(cached_value)
                
                # Cache miss - execute function
                print(f"Cache MISS: {cache_key}")
                result = func(*args, **kwargs)
                
                # Store in cache
                cache_ttl = ttl or self.default_ttl
                self.redis_client.setex(
                    cache_key,
                    cache_ttl,
                    json.dumps(result)
                )
                
                return result
            
            return wrapper
        return decorator
    
    def invalidate(self, pattern):
        """Invalidate cache keys matching pattern."""
        keys = self.redis_client.keys(pattern)
        if keys:
            self.redis_client.delete(*keys)
            print(f"Invalidated {len(keys)} keys")
    
    def get(self, key):
        """Get value from cache."""
        value = self.redis_client.get(key)
        return json.loads(value) if value else None
    
    def set(self, key, value, ttl=None):
        """Set value in cache."""
        cache_ttl = ttl or self.default_ttl
        self.redis_client.setex(key, cache_ttl, json.dumps(value))
    
    def delete(self, key):
        """Delete key from cache."""
        self.redis_client.delete(key)

# Usage example
cache = RedisCache(redis_client, default_ttl=300)

@cache.cache(ttl=600, prefix='user')
def get_user(user_id):
    """Get user from database (simulated)."""
    print(f"Fetching user {user_id} from database...")
    time.sleep(2)  # Simulate slow database query
    return {
        'id': user_id,
        'name': f'User {user_id}',
        'email': f'user{user_id}@example.com'
    }

# First call - cache miss
user = get_user(1)  # Takes 2 seconds
print(user)

# Second call - cache hit
user = get_user(1)  # Instant
print(user)

# Invalidate cache
cache.invalidate('user:*')
```

### Flask-Caching with Redis

```python
from flask import Flask
from flask_caching import Cache

app = Flask(__name__)

# Configure Redis cache
app.config['CACHE_TYPE'] = 'RedisCache'
app.config['CACHE_REDIS_URL'] = 'redis://localhost:6379/0'
app.config['CACHE_DEFAULT_TIMEOUT'] = 300

cache = Cache(app)

@app.route('/users/<int:user_id>')
@cache.cached(timeout=600, key_prefix='user_%s')
def get_user(user_id):
    """Get user with caching."""
    # Simulate database query
    user = fetch_user_from_db(user_id)
    return jsonify(user)

@app.route('/posts')
@cache.cached(timeout=300, query_string=True)
def get_posts():
    """Cache includes query parameters."""
    page = request.args.get('page', 1, type=int)
    posts = fetch_posts(page)
    return jsonify(posts)

@app.route('/expensive-operation')
@cache.memoize(timeout=3600)
def expensive_operation(param1, param2):
    """Cache based on function arguments."""
    result = perform_expensive_calculation(param1, param2)
    return jsonify(result)

# Manual cache operations
@app.route('/clear-cache')
def clear_cache():
    """Clear all cache."""
    cache.clear()
    return jsonify({'message': 'Cache cleared'})

@app.route('/delete-cache/<key>')
def delete_cache_key(key):
    """Delete specific cache key."""
    cache.delete(key)
    return jsonify({'message': f'Deleted {key}'})

# Cache with custom key
@app.route('/custom-cache')
def custom_cache_example():
    """Cache with custom key."""
    cache_key = f"custom:{request.args.get('id')}"
    
    # Try to get from cache
    result = cache.get(cache_key)
    if result is not None:
        return jsonify(result)
    
    # Cache miss - compute result
    result = expensive_computation()
    
    # Store in cache
    cache.set(cache_key, result, timeout=600)
    
    return jsonify(result)
```

### FastAPI with Redis

```python
from fastapi import FastAPI, Depends
from redis import asyncio as aioredis
import json
from typing import Optional

app = FastAPI()

# Redis connection pool
redis_pool = None

async def get_redis():
    """Get Redis connection."""
    global redis_pool
    if redis_pool is None:
        redis_pool = await aioredis.from_url(
            "redis://localhost:6379",
            encoding="utf-8",
            decode_responses=True
        )
    return redis_pool

@app.on_event("shutdown")
async def shutdown():
    """Close Redis connection on shutdown."""
    if redis_pool:
        await redis_pool.close()

@app.get("/users/{user_id}")
async def get_user(user_id: int, redis=Depends(get_redis)):
    """Get user with caching."""
    cache_key = f"user:{user_id}"
    
    # Try cache
    cached_user = await redis.get(cache_key)
    if cached_user:
        return json.loads(cached_user)
    
    # Fetch from database
    user = await fetch_user_from_db(user_id)
    
    # Store in cache
    await redis.setex(
        cache_key,
        600,  # 10 minutes
        json.dumps(user)
    )
    
    return user

# Async cache decorator
from functools import wraps

def async_cache(ttl=300, prefix='cache'):
    """Async cache decorator."""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, redis=None, **kwargs):
            if redis is None:
                return await func(*args, **kwargs)
            
            # Generate cache key
            cache_key = f"{prefix}:{args}:{kwargs}"
            
            # Try cache
            cached = await redis.get(cache_key)
            if cached:
                return json.loads(cached)
            
            # Execute function
            result = await func(*args, **kwargs)
            
            # Store in cache
            await redis.setex(cache_key, ttl, json.dumps(result))
            
            return result
        
        return wrapper
    return decorator

@async_cache(ttl=600, prefix='expensive')
async def expensive_async_operation(param1, param2, redis=None):
    """Expensive operation with caching."""
    # Simulate expensive operation
    await asyncio.sleep(2)
    return {'result': param1 + param2}
```

---

## Memcached

### Basic Memcached Setup

```python
# Install pymemcache
# pip install pymemcache

from pymemcache.client import base
from pymemcache import serde
import time

# Create client with serialization
memcache_client = base.Client(
    ('localhost', 11211),
    serde=serde.pickle_serde  # Use pickle for serialization
)

# Basic operations
def memcached_basic_operations():
    """Demonstrate basic Memcached operations."""
    
    # Set value
    memcache_client.set('key', 'value')
    
    # Get value
    value = memcache_client.get('key')
    print(f"Value: {value}")
    
    # Set with expiration (seconds)
    memcache_client.set('temp_key', 'temp_value', expire=300)
    
    # Add (only if doesn't exist)
    memcache_client.add('new_key', 'new_value')
    
    # Replace (only if exists)
    memcache_client.replace('key', 'updated_value')
    
    # Delete
    memcache_client.delete('key')
    
    # Increment/Decrement
    memcache_client.set('counter', 0)
    memcache_client.incr('counter', 1)
    memcache_client.decr('counter', 1)
    
    # Multiple get
    values = memcache_client.get_many(['key1', 'key2', 'key3'])
    print(f"Values: {values}")
    
    # Flush all
    memcache_client.flush_all()

# Memcached cache decorator
class MemcachedCache:
    """Memcached cache manager."""
    
    def __init__(self, client, default_ttl=300):
        self.client = client
        self.default_ttl = default_ttl
    
    def cache(self, ttl=None, prefix='cache'):
        """Cache decorator."""
        def decorator(func):
            @wraps(func)
            def wrapper(*args, **kwargs):
                # Generate key
                key_data = f"{prefix}:{func.__name__}:{args}:{kwargs}"
                cache_key = hashlib.md5(key_data.encode()).hexdigest()
                
                # Try cache
                cached = self.client.get(cache_key)
                if cached is not None:
                    print(f"Cache HIT: {cache_key}")
                    return cached
                
                # Execute function
                print(f"Cache MISS: {cache_key}")
                result = func(*args, **kwargs)
                
                # Store in cache
                cache_ttl = ttl or self.default_ttl
                self.client.set(cache_key, result, expire=cache_ttl)
                
                return result
            
            return wrapper
        return decorator

# Usage
cache = MemcachedCache(memcache_client)

@cache.cache(ttl=600)
def get_data(id):
    """Get data with caching."""
    time.sleep(2)
    return {'id': id, 'data': f'Data for {id}'}
```

### Flask with Memcached

```python
from flask import Flask
from flask_caching import Cache

app = Flask(__name__)

# Configure Memcached
app.config['CACHE_TYPE'] = 'MemcachedCache'
app.config['CACHE_MEMCACHED_SERVERS'] = ['localhost:11211']
app.config['CACHE_DEFAULT_TIMEOUT'] = 300

cache = Cache(app)

@app.route('/data/<int:id>')
@cache.cached(timeout=600)
def get_cached_data(id):
    """Endpoint with Memcached caching."""
    data = fetch_data(id)
    return jsonify(data)
```

---

## Application-Level Caching

### In-Memory Cache (LRU Cache)

```python
from functools import lru_cache
from collections import OrderedDict
from threading import Lock

# Simple LRU cache using functools
@lru_cache(maxsize=128)
def fibonacci(n):
    """Calculate Fibonacci with caching."""
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

# View cache info
print(fibonacci.cache_info())

# Clear cache
fibonacci.cache_clear()

# Custom LRU Cache
class LRUCache:
    """Thread-safe LRU cache implementation."""
    
    def __init__(self, capacity=128):
        self.capacity = capacity
        self.cache = OrderedDict()
        self.lock = Lock()
    
    def get(self, key):
        """Get item from cache."""
        with self.lock:
            if key not in self.cache:
                return None
            # Move to end (most recently used)
            self.cache.move_to_end(key)
            return self.cache[key]
    
    def set(self, key, value):
        """Set item in cache."""
        with self.lock:
            if key in self.cache:
                # Update and move to end
                self.cache.move_to_end(key)
            self.cache[key] = value
            
            # Remove oldest if over capacity
            if len(self.cache) > self.capacity:
                self.cache.popitem(last=False)
    
    def delete(self, key):
        """Delete item from cache."""
        with self.lock:
            self.cache.pop(key, None)
    
    def clear(self):
        """Clear entire cache."""
        with self.lock:
            self.cache.clear()
    
    def size(self):
        """Get cache size."""
        return len(self.cache)

# Usage
cache = LRUCache(capacity=100)

def get_user(user_id):
    """Get user with LRU cache."""
    # Try cache
    cached_user = cache.get(f'user:{user_id}')
    if cached_user:
        return cached_user
    
    # Fetch from database
    user = fetch_user_from_db(user_id)
    
    # Store in cache
    cache.set(f'user:{user_id}', user)
    
    return user
```

### Time-based Expiration Cache

```python
import time
from threading import Lock
from typing import Any, Optional

class TTLCache:
    """Cache with time-to-live expiration."""
    
    def __init__(self, default_ttl=300):
        self.cache = {}
        self.default_ttl = default_ttl
        self.lock = Lock()
    
    def get(self, key):
        """Get value from cache if not expired."""
        with self.lock:
            if key not in self.cache:
                return None
            
            value, expiry = self.cache[key]
            
            # Check if expired
            if time.time() > expiry:
                del self.cache[key]
                return None
            
            return value
    
    def set(self, key, value, ttl=None):
        """Set value with TTL."""
        with self.lock:
            cache_ttl = ttl or self.default_ttl
            expiry = time.time() + cache_ttl
            self.cache[key] = (value, expiry)
    
    def delete(self, key):
        """Delete key from cache."""
        with self.lock:
            self.cache.pop(key, None)
    
    def clear(self):
        """Clear entire cache."""
        with self.lock:
            self.cache.clear()
    
    def cleanup_expired(self):
        """Remove all expired entries."""
        with self.lock:
            current_time = time.time()
            expired_keys = [
                key for key, (_, expiry) in self.cache.items()
                if current_time > expiry
            ]
            for key in expired_keys:
                del self.cache[key]
            return len(expired_keys)

# Usage
cache = TTLCache(default_ttl=300)

# Set with default TTL
cache.set('key1', 'value1')

# Set with custom TTL
cache.set('key2', 'value2', ttl=60)

# Get value
value = cache.get('key1')

# Cleanup expired entries
expired_count = cache.cleanup_expired()
```

---

## Database Query Caching

### SQLAlchemy Query Caching

```python
from sqlalchemy import create_engine, Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from functools import wraps
import hashlib
import json

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'
    
    id = Column(Integer, primary_key=True)
    username = Column(String(50))
    email = Column(String(100))

engine = create_engine('sqlite:///test.db')
Session = sessionmaker(bind=engine)

class QueryCache:
    """Cache for database queries."""
    
    def __init__(self, cache_backend):
        self.cache = cache_backend
    
    def _generate_key(self, query_str, params):
        """Generate cache key from query and parameters."""
        key_data = f"{query_str}:{json.dumps(params, sort_keys=True)}"
        return f"query:{hashlib.md5(key_data.encode()).hexdigest()}"
    
    def get_or_execute(self, query, ttl=300):
        """Get query result from cache or execute."""
        # Generate cache key
        query_str = str(query)
        params = query.statement.compile().params
        cache_key = self._generate_key(query_str, params)
        
        # Try cache
        cached_result = self.cache.get(cache_key)
        if cached_result:
            return cached_result
        
        # Execute query
        result = query.all()
        
        # Convert to dict for caching
        result_dicts = [
            {c.name: getattr(r, c.name) for c in r.__table__.columns}
            for r in result
        ]
        
        # Store in cache
        self.cache.set(cache_key, result_dicts, ttl=ttl)
        
        return result

# Usage
query_cache = QueryCache(cache)

session = Session()
query = session.query(User).filter(User.username.like('%john%'))

# Get from cache or execute
users = query_cache.get_or_execute(query, ttl=600)
```

### Django Query Caching

```python
from django.core.cache import cache
from django.db.models import QuerySet
import hashlib

def cache_queryset(timeout=300, prefix='qs'):
    """Decorator to cache Django queryset results."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Generate cache key
            cache_key = f"{prefix}:{func.__name__}:{args}:{kwargs}"
            cache_key_hash = hashlib.md5(cache_key.encode()).hexdigest()
            
            # Try cache
            cached_result = cache.get(cache_key_hash)
            if cached_result is not None:
                return cached_result
            
            # Execute function
            result = func(*args, **kwargs)
            
            # Convert queryset to list for caching
            if isinstance(result, QuerySet):
                result_list = list(result)
                cache.set(cache_key_hash, result_list, timeout)
                return result_list
            
            cache.set(cache_key_hash, result, timeout)
            return result
        
        return wrapper
    return decorator

# Usage
from django.contrib.auth.models import User

@cache_queryset(timeout=600)
def get_active_users():
    """Get active users with caching."""
    return User.objects.filter(is_active=True)

# Invalidate cache when user is updated
from django.db.models.signals import post_save
from django.dispatch import receiver

@receiver(post_save, sender=User)
def invalidate_user_cache(sender, instance, **kwargs):
    """Invalidate user cache on save."""
    cache.delete_pattern('qs:get_active_users:*')
```

---

## HTTP Caching

### HTTP Cache Headers

```python
from flask import Flask, make_response
from datetime import datetime, timedelta

app = Flask(__name__)

@app.route('/static-resource')
def static_resource():
    """Serve static resource with caching."""
    response = make_response(render_template('resource.html'))
    
    # Cache for 1 year
    response.headers['Cache-Control'] = 'public, max-age=31536000, immutable'
    
    # Set Expires header
    expires = datetime.utcnow() + timedelta(days=365)
    response.headers['Expires'] = expires.strftime('%a, %d %b %Y %H:%M:%S GMT')
    
    return response

@app.route('/dynamic-resource')
def dynamic_resource():
    """Serve dynamic resource with validation."""
    response = make_response(render_template('dynamic.html'))
    
    # Allow caching but validate
    response.headers['Cache-Control'] = 'public, max-age=300, must-revalidate'
    
    # Set ETag for validation
    content = response.get_data()
    etag = hashlib.md5(content).hexdigest()
    response.headers['ETag'] = f'"{etag}"'
    
    # Check If-None-Match header
    if request.headers.get('If-None-Match') == f'"{etag}"':
        return '', 304  # Not Modified
    
    return response

@app.route('/no-cache')
def no_cache_resource():
    """Serve resource that shouldn't be cached."""
    response = make_response(render_template('private.html'))
    
    # Prevent caching
    response.headers['Cache-Control'] = 'no-cache, no-store, must-revalidate'
    response.headers['Pragma'] = 'no-cache'
    response.headers['Expires'] = '0'
    
    return response

@app.route('/conditional-cache')
def conditional_cache():
    """Conditional caching with Last-Modified."""
    last_modified = get_last_modified_time()
    
    # Check If-Modified-Since
    if_modified_since = request.headers.get('If-Modified-Since')
    if if_modified_since:
        if_modified_datetime = datetime.strptime(
            if_modified_since,
            '%a, %d %b %Y %H:%M:%S GMT'
        )
        if last_modified <= if_modified_datetime:
            return '', 304
    
    response = make_response(get_content())
    response.headers['Last-Modified'] = last_modified.strftime(
        '%a, %d %b %Y %H:%M:%S GMT'
    )
    response.headers['Cache-Control'] = 'public, max-age=3600'
    
    return response
```

### FastAPI HTTP Caching

```python
from fastapi import FastAPI, Response, Request
from fastapi.responses import JSONResponse
from datetime import datetime, timedelta
import hashlib

app = FastAPI()

@app.get("/cached-resource")
async def cached_resource(response: Response):
    """Resource with cache headers."""
    data = {"message": "This is cached"}
    
    # Set cache headers
    response.headers["Cache-Control"] = "public, max-age=3600"
    response.headers["Expires"] = (
        datetime.utcnow() + timedelta(hours=1)
    ).strftime('%a, %d %b %Y %H:%M:%S GMT')
    
    return data

@app.get("/etag-resource")
async def etag_resource(request: Request, response: Response):
    """Resource with ETag validation."""
    content = {"message": "Content with ETag"}
    content_str = json.dumps(content, sort_keys=True)
    etag = hashlib.md5(content_str.encode()).hexdigest()
    
    # Check If-None-Match
    if_none_match = request.headers.get("if-none-match")
    if if_none_match == f'"{etag}"':
        return Response(status_code=304)
    
    # Set ETag
    response.headers["ETag"] = f'"{etag}"'
    response.headers["Cache-Control"] = "public, max-age=3600"
    
    return content

@app.middleware("http")
async def add_cache_headers(request: Request, call_next):
    """Middleware to add cache headers."""
    response = await call_next(request)
    
    # Add cache headers for static files
    if request.url.path.startswith("/static"):
        response.headers["Cache-Control"] = "public, max-age=31536000"
    
    return response
```

---

## CDN Caching

### CDN Cache Control

```python
"""
CDN Caching Best Practices:

1. Static Assets (Images, CSS, JS)
   Cache-Control: public, max-age=31536000, immutable
   - Cache for 1 year
   - Use versioned filenames (main.v1.2.3.js)

2. API Responses
   Cache-Control: public, max-age=300, s-maxage=600
   - Browser cache: 5 minutes
   - CDN cache: 10 minutes

3. Dynamic Content
   Cache-Control: public, max-age=60, s-maxage=0
   - Short browser cache
   - No CDN cache

4. Private Data
   Cache-Control: private, max-age=0, no-cache
   - No CDN caching
   - No shared caching

CDN Cache Headers:
- Cache-Control: Controls caching behavior
- s-maxage: CDN-specific max age (overrides max-age)
- Vary: Specifies which headers affect cache
- Surrogate-Control: CDN-specific (Fastly, Cloudflare)
"""
```

```python
from flask import Flask, Response

app = Flask(__name__)

@app.route('/api/public-data')
def public_api_data():
    """Public API with CDN caching."""
    data = get_public_data()
    response = Response(json.dumps(data), mimetype='application/json')
    
    # Browser: 5 min, CDN: 10 min
    response.headers['Cache-Control'] = 'public, max-age=300, s-maxage=600'
    
    # Vary by Accept header
    response.headers['Vary'] = 'Accept, Accept-Encoding'
    
    return response

@app.route('/api/user-data')
def user_api_data():
    """User-specific data - no CDN caching."""
    data = get_user_data(current_user.id)
    response = Response(json.dumps(data), mimetype='application/json')
    
    # Browser only, no CDN
    response.headers['Cache-Control'] = 'private, max-age=300'
    
    return response

# Cloudflare-specific cache control
@app.route('/cloudflare-cached')
def cloudflare_cached():
    """Resource with Cloudflare-specific caching."""
    data = get_data()
    response = Response(json.dumps(data), mimetype='application/json')
    
    # Cloudflare edge cache: 1 day
    response.headers['Cache-Control'] = 'public, max-age=3600'
    response.headers['CDN-Cache-Control'] = 'max-age=86400'
    
    return response
```

---

## Cache Invalidation

### Cache Invalidation Strategies

```python
"""
Cache Invalidation Methods:

1. Time-based (TTL)
   - Set expiration time
   - Automatic cleanup
   - Simple but may serve stale data

2. Event-based
   - Invalidate on data changes
   - Immediate consistency
   - More complex

3. Manual
   - Explicit invalidation
   - Full control
   - Requires careful management

4. Lazy (Read-through)
   - Check validity on read
   - Recompute if stale
   - Good for expensive operations

Cache Invalidation is Hard:
"There are only two hard things in Computer Science:
cache invalidation and naming things." - Phil Karlton
"""
```

```python
from datetime import datetime

class SmartCache:
    """Cache with multiple invalidation strategies."""
    
    def __init__(self, redis_client):
        self.redis = redis_client
    
    def set_with_tags(self, key, value, tags=None, ttl=300):
        """Set cache value with tags for invalidation."""
        # Store main value
        self.redis.setex(key, ttl, json.dumps(value))
        
        # Store tags mapping
        if tags:
            for tag in tags:
                tag_key = f"tag:{tag}"
                self.redis.sadd(tag_key, key)
                self.redis.expire(tag_key, ttl)
    
    def invalidate_by_tag(self, tag):
        """Invalidate all cache entries with tag."""
        tag_key = f"tag:{tag}"
        
        # Get all keys with this tag
        keys = self.redis.smembers(tag_key)
        
        if keys:
            # Delete all keys
            self.redis.delete(*keys)
            # Delete tag set
            self.redis.delete(tag_key)
            
            return len(keys)
        
        return 0
    
    def set_with_version(self, key, value, version, ttl=300):
        """Set cache with version for invalidation."""
        versioned_key = f"{key}:v{version}"
        self.redis.setex(versioned_key, ttl, json.dumps(value))
        
        # Store latest version
        self.redis.set(f"{key}:version", version)
    
    def get_with_version(self, key):
        """Get cache checking version."""
        # Get current version
        version = self.redis.get(f"{key}:version")
        if not version:
            return None
        
        # Get versioned value
        versioned_key = f"{key}:v{version}"
        value = self.redis.get(versioned_key)
        
        return json.loads(value) if value else None
    
    def increment_version(self, key):
        """Increment version (invalidates cache)."""
        self.redis.incr(f"{key}:version")

# Usage
smart_cache = SmartCache(redis_client)

# Cache with tags
smart_cache.set_with_tags(
    'user:1',
    {'id': 1, 'name': 'John'},
    tags=['users', 'user:1'],
    ttl=600
)

# Invalidate all user caches
smart_cache.invalidate_by_tag('users')

# Version-based caching
smart_cache.set_with_version('config', {'theme': 'dark'}, version=1)

# Invalidate by incrementing version
smart_cache.increment_version('config')
```

### Write-through Cache

```python
class WriteThroughCache:
    """Write-through cache pattern."""
    
    def __init__(self, cache, database):
        self.cache = cache
        self.database = database
    
    def get(self, key):
        """Get from cache, fallback to database."""
        # Try cache
        value = self.cache.get(key)
        if value is not None:
            return value
        
        # Cache miss - get from database
        value = self.database.get(key)
        if value is not None:
            # Store in cache
            self.cache.set(key, value)
        
        return value
    
    def set(self, key, value):
        """Write to both cache and database."""
        # Write to database first
        self.database.set(key, value)
        
        # Then update cache
        self.cache.set(key, value)
    
    def delete(self, key):
        """Delete from both cache and database."""
        # Delete from database
        self.database.delete(key)
        
        # Delete from cache
        self.cache.delete(key)

# Write-behind (lazy write) cache
from queue import Queue
from threading import Thread

class WriteBehindCache:
    """Write-behind cache pattern."""
    
    def __init__(self, cache, database):
        self.cache = cache
        self.database = database
        self.write_queue = Queue()
        self.writer_thread = Thread(target=self._process_writes, daemon=True)
        self.writer_thread.start()
    
    def get(self, key):
        """Get from cache."""
        value = self.cache.get(key)
        if value is None:
            value = self.database.get(key)
            if value is not None:
                self.cache.set(key, value)
        return value
    
    def set(self, key, value):
        """Write to cache immediately, queue database write."""
        # Immediate cache update
        self.cache.set(key, value)
        
        # Queue database write
        self.write_queue.put(('set', key, value))
    
    def delete(self, key):
        """Delete from cache, queue database deletion."""
        self.cache.delete(key)
        self.write_queue.put(('delete', key, None))
    
    def _process_writes(self):
        """Background thread to process writes."""
        while True:
            operation, key, value = self.write_queue.get()
            
            try:
                if operation == 'set':
                    self.database.set(key, value)
                elif operation == 'delete':
                    self.database.delete(key)
            except Exception as e:
                print(f"Write error: {e}")
            finally:
                self.write_queue.task_done()
```

---

## Caching Patterns

### Cache-Aside (Lazy Loading)

```python
def cache_aside_get(key):
    """Cache-aside pattern."""
    # 1. Try cache
    value = cache.get(key)
    if value is not None:
        return value
    
    # 2. Cache miss - load from database
    value = database.get(key)
    
    # 3. Store in cache
    if value is not None:
        cache.set(key, value, ttl=600)
    
    return value

def cache_aside_update(key, new_value):
    """Update with cache-aside."""
    # 1. Update database
    database.set(key, new_value)
    
    # 2. Invalidate cache
    cache.delete(key)
    
    # Next read will populate cache
```

### Read-Through Cache

```python
class ReadThroughCache:
    """Read-through cache pattern."""
    
    def __init__(self, cache, database):
        self.cache = cache
        self.database = database
    
    def get(self, key, loader=None):
        """Get with automatic loading."""
        # Try cache
        value = self.cache.get(key)
        if value is not None:
            return value
        
        # Load from source
        if loader:
            value = loader(key)
        else:
            value = self.database.get(key)
        
        # Store in cache
        if value is not None:
            self.cache.set(key, value)
        
        return value

# Usage
read_through = ReadThroughCache(cache, database)

user = read_through.get(
    'user:1',
    loader=lambda key: fetch_user_from_api(key)
)
```

### Refresh-Ahead

```python
import threading
import time

class RefreshAheadCache:
    """Refresh-ahead caching pattern."""
    
    def __init__(self, cache, loader, ttl=300, refresh_threshold=0.8):
        self.cache = cache
        self.loader = loader
        self.ttl = ttl
        self.refresh_threshold = refresh_threshold
    
    def get(self, key):
        """Get with automatic refresh."""
        # Get from cache with TTL info
        cached = self.cache.get(key)
        
        if cached is None:
            # Cache miss - load and cache
            value = self.loader(key)
            self.cache.set(key, value, ttl=self.ttl)
            return value
        
        # Check if refresh needed
        ttl = self.cache.ttl(key)
        if ttl < (self.ttl * self.refresh_threshold):
            # Trigger background refresh
            threading.Thread(
                target=self._refresh,
                args=(key,),
                daemon=True
            ).start()
        
        return cached
    
    def _refresh(self, key):
        """Background refresh."""
        try:
            value = self.loader(key)
            self.cache.set(key, value, ttl=self.ttl)
        except Exception as e:
            print(f"Refresh error for {key}: {e}")
```

---

## Query Optimization

### Database Indexing

```python
"""
Query Optimization Strategies:

1. Indexing
   - Create indexes on frequently queried columns
   - Composite indexes for multi-column queries
   - Avoid over-indexing (slows writes)

2. Query Structure
   - Use SELECT specific columns, not SELECT *
   - Avoid N+1 queries (use joins/prefetch)
   - Use LIMIT for pagination
   - Use EXISTS instead of COUNT when checking existence

3. Database-Level Caching
   - Query cache (MySQL)
   - Prepared statement cache
   - Buffer pool

4. Application-Level Optimization
   - Batch operations
   - Connection pooling
   - Read replicas for read-heavy loads
"""
```

```python
from sqlalchemy import Index, create_engine
from sqlalchemy.orm import sessionmaker, joinedload

# Define indexes
class User(Base):
    __tablename__ = 'users'
    
    id = Column(Integer, primary_key=True)
    email = Column(String(100), unique=True)
    username = Column(String(50))
    created_at = Column(DateTime)
    
    # Single column index
    __table_args__ = (
        Index('idx_email', 'email'),
        Index('idx_username', 'username'),
        Index('idx_created_at', 'created_at'),
        # Composite index
        Index('idx_username_created', 'username', 'created_at'),
    )

# Optimized queries
def get_users_optimized(session):
    """Optimized user query."""
    # ✅ Good: Select specific columns
    users = session.query(User.id, User.username).all()
    
    # ❌ Bad: Select all columns
    # users = session.query(User).all()
    
    return users

def get_user_with_posts(session, user_id):
    """Avoid N+1 queries."""
    # ✅ Good: Use joinedload (one query)
    user = session.query(User)\
        .options(joinedload(User.posts))\
        .filter(User.id == user_id)\
        .first()
    
    # ❌ Bad: N+1 queries
    # user = session.query(User).filter(User.id == user_id).first()
    # posts = user.posts  # Triggers separate query for each user
    
    return user

def paginate_users(session, page=1, per_page=20):
    """Paginated query."""
    offset = (page - 1) * per_page
    
    users = session.query(User)\
        .order_by(User.created_at.desc())\
        .limit(per_page)\
        .offset(offset)\
        .all()
    
    return users

def user_exists(session, email):
    """Check existence efficiently."""
    # ✅ Good: Use EXISTS
    exists = session.query(
        session.query(User).filter(User.email == email).exists()
    ).scalar()
    
    # ❌ Bad: Count all rows
    # count = session.query(User).filter(User.email == email).count()
    # exists = count > 0
    
    return exists
```

### Django Query Optimization

```python
from django.db import models
from django.db.models import Prefetch, Count, Q

class User(models.Model):
    username = models.CharField(max_length=50, db_index=True)
    email = models.EmailField(unique=True)
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)
    
    class Meta:
        indexes = [
            models.Index(fields=['username', 'created_at']),
        ]

# Optimized queries
def get_users_with_posts():
    """Optimize with select_related and prefetch_related."""
    # ✅ Good: Prefetch related objects
    users = User.objects.prefetch_related('posts').all()
    
    # For each user, posts are already loaded (no extra queries)
    for user in users:
        posts = user.posts.all()
    
    return users

def get_active_users_with_counts():
    """Aggregate in database."""
    users = User.objects.filter(is_active=True)\
        .annotate(post_count=Count('posts'))\
        .select_related('profile')
    
    return users

def search_users(query):
    """Efficient search with Q objects."""
    users = User.objects.filter(
        Q(username__icontains=query) | Q(email__icontains=query)
    ).only('id', 'username', 'email')  # Only load specific fields
    
    return users

def get_users_batch(ids):
    """Batch query."""
    users = User.objects.filter(id__in=ids)
    return users
```

---

## Performance Monitoring

### Cache Performance Metrics

```python
import time
from collections import defaultdict

class CacheMonitor:
    """Monitor cache performance."""
    
    def __init__(self):
        self.hits = 0
        self.misses = 0
        self.sets = 0
        self.deletes = 0
        self.errors = 0
        self.total_time = 0
        self.operation_times = defaultdict(list)
    
    def record_hit(self, duration=0):
        """Record cache hit."""
        self.hits += 1
        self.total_time += duration
        self.operation_times['hit'].append(duration)
    
    def record_miss(self, duration=0):
        """Record cache miss."""
        self.misses += 1
        self.total_time += duration
        self.operation_times['miss'].append(duration)
    
    def record_set(self, duration=0):
        """Record cache set."""
        self.sets += 1
        self.total_time += duration
        self.operation_times['set'].append(duration)
    
    def record_delete(self, duration=0):
        """Record cache delete."""
        self.deletes += 1
        self.operation_times['delete'].append(duration)
    
    def record_error(self):
        """Record error."""
        self.errors += 1
    
    @property
    def hit_rate(self):
        """Calculate hit rate."""
        total = self.hits + self.misses
        return (self.hits / total * 100) if total > 0 else 0
    
    @property
    def avg_time(self):
        """Calculate average operation time."""
        total_ops = self.hits + self.misses + self.sets + self.deletes
        return (self.total_time / total_ops) if total_ops > 0 else 0
    
    def get_stats(self):
        """Get cache statistics."""
        return {
            'hits': self.hits,
            'misses': self.misses,
            'sets': self.sets,
            'deletes': self.deletes,
            'errors': self.errors,
            'hit_rate': f"{self.hit_rate:.2f}%",
            'avg_time': f"{self.avg_time*1000:.2f}ms",
            'total_operations': self.hits + self.misses + self.sets + self.deletes
        }

# Monitored cache wrapper
class MonitoredCache:
    """Cache with performance monitoring."""
    
    def __init__(self, cache_backend):
        self.cache = cache_backend
        self.monitor = CacheMonitor()
    
    def get(self, key):
        """Get with monitoring."""
        start = time.time()
        try:
            value = self.cache.get(key)
            duration = time.time() - start
            
            if value is not None:
                self.monitor.record_hit(duration)
            else:
                self.monitor.record_miss(duration)
            
            return value
        except Exception as e:
            self.monitor.record_error()
            raise
    
    def set(self, key, value, ttl=None):
        """Set with monitoring."""
        start = time.time()
        try:
            self.cache.set(key, value, ttl)
            duration = time.time() - start
            self.monitor.record_set(duration)
        except Exception as e:
            self.monitor.record_error()
            raise
    
    def get_stats(self):
        """Get cache stats."""
        return self.monitor.get_stats()

# Usage
monitored_cache = MonitoredCache(redis_client)

# Operations are automatically monitored
monitored_cache.set('key1', 'value1')
monitored_cache.get('key1')  # Hit
monitored_cache.get('key2')  # Miss

# Get statistics
stats = monitored_cache.get_stats()
print(stats)
```

---

## Cache Strategies

### Multi-Level Caching

```python
class MultiLevelCache:
    """Multi-level cache (L1: Memory, L2: Redis)."""
    
    def __init__(self, l1_cache, l2_cache):
        self.l1 = l1_cache  # Fast, small (in-memory)
        self.l2 = l2_cache  # Slower, large (Redis)
    
    def get(self, key):
        """Get from multi-level cache."""
        # Try L1 first
        value = self.l1.get(key)
        if value is not None:
            print("L1 cache hit")
            return value
        
        # Try L2
        value = self.l2.get(key)
        if value is not None:
            print("L2 cache hit")
            # Promote to L1
            self.l1.set(key, value)
            return value
        
        print("Cache miss")
        return None
    
    def set(self, key, value, ttl=None):
        """Set in both cache levels."""
        # Set in both L1 and L2
        self.l1.set(key, value, ttl)
        self.l2.set(key, value, ttl)
    
    def delete(self, key):
        """Delete from both levels."""
        self.l1.delete(key)
        self.l2.delete(key)

# Usage
l1_cache = LRUCache(capacity=100)  # Small, fast
l2_cache = RedisCache(redis_client)  # Large, slower

multi_cache = MultiLevelCache(l1_cache, l2_cache)

# Get with fallback through cache levels
value = multi_cache.get('expensive_key')
```

---

## Best Practices

### Caching Best Practices

```python
"""
✅ Caching Best Practices:

1. Cache Appropriate Data
   ✅ Expensive computations
   ✅ Frequent database queries
   ✅ External API calls
   ✅ Static content
   ❌ Highly dynamic data
   ❌ User-specific data (be careful)
   ❌ Sensitive data

2. Set Appropriate TTLs
   ✅ Longer TTL for static data
   ✅ Shorter TTL for dynamic data
   ✅ Consider data change frequency
   ✅ Balance freshness vs performance

3. Cache Keys
   ✅ Use descriptive, consistent naming
   ✅ Include version in key
   ✅ Use namespaces
   ✅ Hash long keys
   
   Example: user:123:profile:v2

4. Invalidation Strategy
   ✅ Choose appropriate strategy
   ✅ Implement proper invalidation
   ✅ Handle race conditions
   ✅ Monitor stale data

5. Error Handling
   ✅ Handle cache failures gracefully
   ✅ Don't fail if cache is down
   ✅ Log cache errors
   ✅ Use circuit breakers

6. Monitoring
   ✅ Track hit/miss ratio
   ✅ Monitor cache size
   ✅ Track eviction rate
   ✅ Alert on anomalies

7. Security
   ✅ Don't cache sensitive data
   ✅ Use appropriate access controls
   ✅ Encrypt cached data if needed
   ✅ Consider cache timing attacks

8. Testing
   ✅ Test with and without cache
   ✅ Test cache invalidation
   ✅ Test cache failures
   ✅ Load test with cache
"""
```

### Complete Caching Example

```python
from flask import Flask, jsonify
from flask_caching import Cache
from functools import wraps
import time

app = Flask(__name__)

# Configure cache
app.config['CACHE_TYPE'] = 'RedisCache'
app.config['CACHE_REDIS_URL'] = 'redis://localhost:6379/0'
app.config['CACHE_DEFAULT_TIMEOUT'] = 300

cache = Cache(app)

# Cache monitoring
cache_stats = CacheMonitor()

def cached_with_monitoring(timeout=300, key_prefix='view'):
    """Cache decorator with monitoring."""
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            cache_key = f"{key_prefix}:{request.url}"
            
            # Try cache
            start = time.time()
            cached_response = cache.get(cache_key)
            
            if cached_response:
                cache_stats.record_hit(time.time() - start)
                return cached_response
            
            cache_stats.record_miss(time.time() - start)
            
            # Execute function
            response = f(*args, **kwargs)
            
            # Store in cache
            cache.set(cache_key, response, timeout=timeout)
            
            return response
        
        return decorated_function
    return decorator

@app.route('/users/<int:user_id>')
@cached_with_monitoring(timeout=600, key_prefix='user')
def get_user(user_id):
    """Get user with caching and monitoring."""
    user = fetch_user_from_db(user_id)
    return jsonify(user)

@app.route('/cache/stats')
def cache_statistics():
    """Get cache statistics."""
    return jsonify(cache_stats.get_stats())

@app.route('/cache/clear')
def clear_cache():
    """Clear all cache."""
    cache.clear()
    return jsonify({'message': 'Cache cleared'})

if __name__ == '__main__':
    app.run(debug=True)
```

---

## Summary

This comprehensive guide covered:

1. **Introduction**: Caching benefits and cache hit/miss
2. **Types**: Client, CDN, application, database caching
3. **Redis**: Setup, decorators, Flask/FastAPI integration
4. **Memcached**: Basic operations and Flask integration
5. **Application Caching**: LRU cache, TTL cache
6. **Database Caching**: SQLAlchemy and Django query caching
7. **HTTP Caching**: Cache headers, ETag, Last-Modified
8. **CDN Caching**: Cache-Control, s-maxage
9. **Invalidation**: Time-based, event-based, manual
10. **Patterns**: Cache-aside, read-through, write-through
11. **Query Optimization**: Indexing, efficient queries
12. **Monitoring**: Cache statistics and metrics
13. **Strategies**: Multi-level caching
14. **Best Practices**: What to cache, TTLs, security

Key Takeaways:
- Cache expensive operations and frequent queries
- Use appropriate TTLs based on data volatility
- Implement proper cache invalidation
- Monitor cache performance (hit rate)
- Use multi-level caching for optimal performance
- Handle cache failures gracefully
- Don't cache sensitive data without encryption
- Test cache invalidation thoroughly
- Use Redis for distributed caching
- Optimize database queries first

Caching is essential for performance and scalability!