Troubleshooting, Debugging, Migration & Integration in FastAPI
22.1 Common Issues
Async/Await Errors
The most frequent source of bugs in FastAPI applications.
pythonfrom fastapi import FastAPI
import asyncio, time

app = FastAPI()

# ----------------------------------------------------------------
# ISSUE 1: Calling a coroutine without await
# ----------------------------------------------------------------

# ✗ BAD — returns a coroutine object, never executes
@app.get("/bad-coroutine")
async def bad_coroutine():
    result = some_async_function()   # Missing await!
    return {"result": result}        # result = <coroutine object ...>

# ✓ GOOD
@app.get("/good-coroutine")
async def good_coroutine():
    result = await some_async_function()
    return {"result": result}


# ----------------------------------------------------------------
# ISSUE 2: Blocking the event loop in an async function
# ----------------------------------------------------------------

# ✗ BAD — time.sleep blocks the ENTIRE event loop
@app.get("/blocking")
async def blocking_endpoint():
    time.sleep(5)                    # Blocks every other request!
    return {"status": "done"}

# ✓ GOOD — use asyncio.sleep or run_in_executor
@app.get("/non-blocking")
async def non_blocking_endpoint():
    await asyncio.sleep(5)           # Yields control to the event loop
    return {"status": "done"}

# ✓ GOOD — CPU-bound or legacy blocking code
@app.get("/thread-pool")
async def thread_pool_endpoint():
    loop   = asyncio.get_event_loop()
    result = await loop.run_in_executor(None, blocking_cpu_task)
    return {"result": result}


# ----------------------------------------------------------------
# ISSUE 3: Using sync DB driver inside async function
# ----------------------------------------------------------------

# ✗ BAD — psycopg2 (sync) blocks the event loop
@app.get("/sync-db")
async def sync_db():
    conn   = psycopg2.connect(DATABASE_URL)   # Blocking!
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users")
    return cursor.fetchall()

# ✓ GOOD — asyncpg or SQLAlchemy async
@app.get("/async-db")
async def async_db(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User))
    return result.scalars().all()


# ----------------------------------------------------------------
# ISSUE 4: Mixing sync and async improperly
# ----------------------------------------------------------------

# ✗ BAD — calling async function from sync context
def sync_function():
    result = await some_async_function()   # SyntaxError!
    return result

# ✓ GOOD — use asyncio.run() or make the function async
def sync_wrapper():
    return asyncio.run(some_async_function())

# ✓ BEST — keep async functions async
async def async_function():
    return await some_async_function()


# ----------------------------------------------------------------
# ISSUE 5: RuntimeError — event loop is closed
# ----------------------------------------------------------------

# ✗ BAD — creating new event loop in thread
import threading

def run_in_thread():
    loop   = asyncio.new_event_loop()
    result = loop.run_until_complete(some_async_function())
    loop.close()                     # May cause issues with shared resources
    return result

# ✓ GOOD — use asyncio.run_coroutine_threadsafe
def run_from_thread(coro, loop):
    future = asyncio.run_coroutine_threadsafe(coro, loop)
    return future.result(timeout=10)
Dependency Injection Issues
pythonfrom fastapi import FastAPI, Depends, HTTPException

app = FastAPI()

# ----------------------------------------------------------------
# ISSUE 1: Dependency not called (missing parentheses)
# ----------------------------------------------------------------

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# ✗ BAD — passes the function, not its result
@app.get("/users")
async def get_users(db = Depends(get_db)):   # ✓ correct
    pass

@app.get("/users-bad")
async def get_users_bad(db = get_db):        # ✗ wrong — db is the function itself
    pass


# ----------------------------------------------------------------
# ISSUE 2: Dependency that raises but doesn't clean up
# ----------------------------------------------------------------

# ✗ BAD — resource leak if exception occurs before yield
def bad_dependency():
    db = SessionLocal()
    if not db:
        raise HTTPException(500, "DB unavailable")   # db never closed!
    yield db

# ✓ GOOD — always clean up with try/finally
def good_dependency():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


# ----------------------------------------------------------------
# ISSUE 3: Async dependency used in sync route
# ----------------------------------------------------------------

async def async_dep():
    return await fetch_something()

# ✗ BAD — async dependency in sync route handler
@app.get("/sync-route")
def sync_route(data = Depends(async_dep)):   # FastAPI will warn
    return data

# ✓ GOOD — match async dep with async route
@app.get("/async-route")
async def async_route(data = Depends(async_dep)):
    return data


# ----------------------------------------------------------------
# ISSUE 4: Circular dependencies
# ----------------------------------------------------------------

# ✗ BAD — A depends on B, B depends on A
def dependency_a(b = Depends(lambda: dependency_b())):
    return "a"

def dependency_b(a = Depends(lambda: dependency_a())):
    return "b"

# ✓ GOOD — extract shared logic into a base dependency
def shared_dep():
    return "shared"

def dependency_a(shared = Depends(shared_dep)):
    return f"a+{shared}"

def dependency_b(shared = Depends(shared_dep)):
    return f"b+{shared}"
CORS Problems
pythonfrom fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# ----------------------------------------------------------------
# ISSUE 1: CORS not configured at all
# ----------------------------------------------------------------
# Browser error: "Access to fetch at 'http://api.com' from origin
# 'http://localhost:3000' has been blocked by CORS policy"

# ✓ FIX — add CORSMiddleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "https://myapp.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


# ----------------------------------------------------------------
# ISSUE 2: Wildcard + credentials conflict
# ----------------------------------------------------------------

# ✗ BAD — browsers reject wildcard when credentials are sent
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,   # Error: Cannot use wildcard with credentials
)

# ✓ GOOD — list specific origins when using credentials
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://myapp.com"],
    allow_credentials=True,
)


# ----------------------------------------------------------------
# ISSUE 3: Middleware added after router (order matters)
# ----------------------------------------------------------------

# ✗ BAD — routers registered before middleware
app.include_router(users_router)   # Routes added first
app.add_middleware(CORSMiddleware, allow_origins=["*"])  # Too late!

# ✓ GOOD — middleware always before routers
app.add_middleware(CORSMiddleware, allow_origins=["*"])
app.include_router(users_router)


# ----------------------------------------------------------------
# ISSUE 4: Preflight OPTIONS request failing
# ----------------------------------------------------------------

# ✓ FIX — ensure OPTIONS method is allowed (it's included in "*")
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://myapp.com"],
    allow_methods=["GET", "POST", "PUT", "DELETE", "OPTIONS"],  # Explicit
    allow_headers=["Content-Type", "Authorization"],
)

# Debug: log CORS headers on responses
@app.middleware("http")
async def debug_cors(request: Request, call_next):
    response = await call_next(request)
    print("CORS headers:", {k: v for k, v in response.headers.items() if "access-control" in k})
    return response
Database Connection Issues
pythonfrom sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.exc import OperationalError, TimeoutError

# ----------------------------------------------------------------
# ISSUE 1: Connection string format
# ----------------------------------------------------------------

# ✗ BAD — sync driver for async engine
DATABASE_URL = "postgresql://user:pass@localhost/db"        # psycopg2 (sync)

# ✓ GOOD — async driver
DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/db"  # asyncpg (async)
DATABASE_URL = "sqlite+aiosqlite:///./app.db"               # aiosqlite for SQLite


# ----------------------------------------------------------------
# ISSUE 2: Pool exhaustion
# ----------------------------------------------------------------

# ✗ BAD — default pool too small for high concurrency
engine = create_async_engine(DATABASE_URL)

# ✓ GOOD — tune pool for expected load
engine = create_async_engine(
    DATABASE_URL,
    pool_size=20,           # Persistent connections
    max_overflow=10,        # Burst connections
    pool_timeout=30,        # Wait max 30s for a connection
    pool_recycle=3600,      # Recycle connections hourly
    pool_pre_ping=True,     # Test connection before use
)


# ----------------------------------------------------------------
# ISSUE 3: Sessions not closed — connection leak
# ----------------------------------------------------------------

# ✗ BAD — session never closed on error
@app.get("/users-leaky")
async def leaky():
    db     = AsyncSession(engine)
    result = await db.execute(select(User))   # If this raises, db is never closed
    return result.scalars().all()

# ✓ GOOD — use context manager or dependency
async def get_db():
    async with AsyncSession(engine) as session:
        yield session           # Automatically closed after request

@app.get("/users")
async def get_users(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User))
    return result.scalars().all()


# ----------------------------------------------------------------
# ISSUE 4: Transaction not committed / rolled back
# ----------------------------------------------------------------

# ✗ BAD — changes never persisted
@app.post("/users")
async def create_user(user_in: UserCreate, db: AsyncSession = Depends(get_db)):
    user = User(**user_in.dict())
    db.add(user)
    # Missing: await db.commit()
    return user

# ✓ GOOD — commit on success, rollback on error
@app.post("/users")
async def create_user(user_in: UserCreate, db: AsyncSession = Depends(get_db)):
    try:
        user = User(**user_in.dict())
        db.add(user)
        await db.commit()
        await db.refresh(user)
        return user
    except Exception:
        await db.rollback()
        raise


# ----------------------------------------------------------------
# ISSUE 5: Database not reachable on startup
# ----------------------------------------------------------------

@app.on_event("startup")
async def verify_db():
    try:
        async with engine.connect() as conn:
            await conn.execute(text("SELECT 1"))
        print("✅ Database connection OK")
    except OperationalError as e:
        print(f"❌ Database connection failed: {e}")
        raise SystemExit(1)
Import Errors
python# ----------------------------------------------------------------
# ISSUE 1: Circular imports
# ----------------------------------------------------------------

# ✗ BAD — app.py imports from users.py, users.py imports from app.py
# app.py
from routers.users import router   # users.py imports app → circular!

# ✓ GOOD — use a shared module
# deps.py  (shared dependencies, no app import)
from database import get_db
from models import User

# users.py
from deps import get_db, User       # No circular reference

# app.py
from routers import users
app.include_router(users.router)


# ----------------------------------------------------------------
# ISSUE 2: Model imported before Base defined
# ----------------------------------------------------------------

# ✗ BAD — importing model before Base is initialised
from models.user import User        # user.py needs Base from database.py
from database import Base           # Too late!

# ✓ GOOD — import Base first, then models
from database import Base, engine   # Base defined here
from models.user import User        # Now safe to import


# ----------------------------------------------------------------
# ISSUE 3: Missing __init__.py in packages
# ----------------------------------------------------------------
# ✓ Structure every Python package with __init__.py:
# app/
#   __init__.py
#   main.py
#   routers/
#     __init__.py
#     users.py
#     posts.py
#   models/
#     __init__.py
#     user.py


# ----------------------------------------------------------------
# ISSUE 4: Wrong Python path / virtual environment
# ----------------------------------------------------------------

# Diagnose:
import sys
print(sys.executable)   # Should point to your venv
print(sys.path)         # Check PYTHONPATH

# Fix — always activate venv before running
# source venv/bin/activate      (Linux/Mac)
# venv\Scripts\activate         (Windows)
# pip install -r requirements.txt

22.2 Debugging Techniques
Debug Mode
pythonimport uvicorn
from fastapi import FastAPI

app = FastAPI(debug=True)   # Enables detailed tracebacks in responses

# Run with hot-reload in development
if __name__ == "__main__":
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=8000,
        reload=True,           # Hot-reload on file change
        reload_dirs=["app"],   # Watch only app directory
        log_level="debug",     # Verbose logging
    )

# Environment-aware debug flag
import os
from config import settings

app = FastAPI(
    debug=settings.ENVIRONMENT == "development",
    docs_url="/docs"  if settings.ENVIRONMENT == "development" else None,
    redoc_url="/redoc" if settings.ENVIRONMENT == "development" else None,
)
Logging Strategies
pythonimport logging
import structlog
from fastapi import FastAPI, Request
import time, uuid

app = FastAPI()

# ── Basic logging setup ────────────────────────────────────────
logging.basicConfig(
    level=logging.DEBUG,
    format="%(asctime)s [%(levelname)s] %(name)s — %(message)s",
)
logger = logging.getLogger("myapp")

# ── Structured logging with structlog ─────────────────────────
structlog.configure(
    processors=[
        structlog.stdlib.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ],
    logger_factory=structlog.stdlib.LoggerFactory(),
)
slog = structlog.get_logger()

# ── Request / response logging middleware ──────────────────────
@app.middleware("http")
async def log_requests(request: Request, call_next):
    request_id = str(uuid.uuid4())[:8]
    start      = time.perf_counter()

    slog.info("request_started",
              id=request_id, method=request.method, path=request.url.path,
              client=request.client.host if request.client else "unknown")

    response   = await call_next(request)
    duration   = (time.perf_counter() - start) * 1000

    level = "warning" if response.status_code >= 400 else "info"
    getattr(slog, level)("request_finished",
                         id=request_id, status=response.status_code,
                         duration_ms=round(duration, 2))

    response.headers["X-Request-ID"] = request_id
    return response

# ── Log SQL queries ────────────────────────────────────────────
engine = create_async_engine(DATABASE_URL, echo=True)  # echo=True logs all SQL

# ── Log slow queries ───────────────────────────────────────────
SLOW_QUERY_THRESHOLD = 0.5   # seconds

@event.listens_for(engine.sync_engine, "before_cursor_execute")
def before_execute(conn, cursor, statement, params, context, executemany):
    context._query_start = time.perf_counter()

@event.listens_for(engine.sync_engine, "after_cursor_execute")
def after_execute(conn, cursor, statement, params, context, executemany):
    duration = time.perf_counter() - context._query_start
    if duration > SLOW_QUERY_THRESHOLD:
        logger.warning(f"SLOW QUERY ({duration:.2f}s): {statement[:200]}")
Python Debugger (pdb)
python# ── Inline breakpoints ─────────────────────────────────────────
@app.get("/debug-pdb")
async def debug_with_pdb():
    data = fetch_some_data()

    import pdb; pdb.set_trace()   # Pauses here, opens interactive REPL

    return {"data": data}

# ── Modern breakpoint() — Python 3.7+ ─────────────────────────
@app.get("/debug-breakpoint")
async def debug_breakpoint():
    result = complex_calculation()
    breakpoint()                  # Same as pdb.set_trace()
    return result

# ── Async-compatible debugger (aiohttp-devtools / aiodebug) ───
import asyncio

asyncio.get_event_loop().set_debug(True)   # Enable async debug mode

# Catch slow coroutines (> 0.1 s)
import os
os.environ["PYTHONASYNCIODEBUG"] = "1"

# ── Post-mortem debugging on unhandled exceptions ──────────────
import sys, pdb, traceback

@app.exception_handler(Exception)
async def debug_exception_handler(request: Request, exc: Exception):
    if settings.ENVIRONMENT == "development":
        traceback.print_exc()
        pdb.post_mortem(sys.exc_info()[2])   # Drop into debugger on crash
    raise exc
IDE Debugging
json// .vscode/launch.json  (VS Code)
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "FastAPI Debug",
      "type": "debugpy",
      "request": "launch",
      "module": "uvicorn",
      "args": ["main:app", "--reload", "--port", "8000"],
      "jinja": true,
      "justMyCode": false,
      "env": {
        "ENVIRONMENT": "development",
        "DATABASE_URL": "postgresql+asyncpg://user:pass@localhost/dev"
      }
    },
    {
      "name": "FastAPI Tests",
      "type": "debugpy",
      "request": "launch",
      "module": "pytest",
      "args": ["tests/", "-v", "--no-header", "-x"],
      "console": "integratedTerminal"
    }
  ]
}
python# PyCharm — set breakpoints in the gutter, then:
# Run > Debug 'uvicorn main:app --reload'

# Remote debugging (attach to running process)
import debugpy
debugpy.listen(("0.0.0.0", 5678))
print("⏳ Waiting for debugger attach on port 5678...")
debugpy.wait_for_client()            # Block until IDE attaches
Request/Response Inspection
pythonfrom fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
import json

app = FastAPI()

# ── Inspect raw request ────────────────────────────────────────
@app.post("/inspect")
async def inspect_request(request: Request):
    body    = await request.body()
    form    = await request.form() if "form" in request.headers.get("content-type","") else {}

    return {
        "method":      request.method,
        "url":         str(request.url),
        "path_params": request.path_params,
        "query":       dict(request.query_params),
        "headers":     dict(request.headers),
        "client":      request.client.host if request.client else None,
        "body":        body.decode("utf-8", errors="replace"),
        "form":        dict(form),
    }

# ── Response inspection middleware ─────────────────────────────
@app.middleware("http")
async def inspect_responses(request: Request, call_next):
    response = await call_next(request)

    if settings.ENVIRONMENT == "development" and response.status_code >= 400:
        body = b""
        async for chunk in response.body_iterator:
            body += chunk
        print(f"⚠  {request.method} {request.url.path} → {response.status_code}")
        print(f"   Response body: {body.decode()[:500]}")

        return JSONResponse(
            status_code=response.status_code,
            content=json.loads(body),
            headers=dict(response.headers),
        )

    return response

# ── HTTPie / curl — quick CLI testing ─────────────────────────
"""
# Install HTTPie
pip install httpie

# Test endpoints
http GET localhost:8000/users
http POST localhost:8000/login username=john password=secret
http GET localhost:8000/users/1 Authorization:"Bearer TOKEN"

# With curl
curl -X POST http://localhost:8000/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=john&password=secret" | python -m json.tool
"""

22.3 Performance Issues
Identifying Slow Endpoints
pythonimport time, statistics
from collections import defaultdict
from fastapi import FastAPI, Request

app = FastAPI()

# ── Per-endpoint timing ────────────────────────────────────────
endpoint_times: defaultdict[str, list[float]] = defaultdict(list)

@app.middleware("http")
async def profile_endpoints(request: Request, call_next):
    start    = time.perf_counter()
    response = await call_next(request)
    duration = time.perf_counter() - start

    key = f"{request.method} {request.url.path}"
    endpoint_times[key].append(duration)

    if duration > 1.0:
        logger.warning(f"SLOW: {key} took {duration:.2f}s")

    response.headers["X-Response-Time"] = f"{duration*1000:.1f}ms"
    return response

@app.get("/debug/performance", include_in_schema=False)
async def perf_report():
    report = {}
    for endpoint, times in sorted(endpoint_times.items()):
        if times:
            report[endpoint] = {
                "calls":   len(times),
                "avg_ms":  round(statistics.mean(times) * 1000, 2),
                "max_ms":  round(max(times) * 1000, 2),
                "p95_ms":  round(sorted(times)[int(len(times) * 0.95)] * 1000, 2),
            }
    return report

# ── Profiling with cProfile ────────────────────────────────────
import cProfile, pstats, io

@app.get("/debug/profile/{n}", include_in_schema=False)
async def profile_route(n: int):
    pr  = cProfile.Profile()
    pr.enable()
    result = cpu_intensive_task(n)
    pr.disable()

    buf   = io.StringIO()
    stats = pstats.Stats(pr, stream=buf).sort_stats("cumulative")
    stats.print_stats(15)
    return {"profile": buf.getvalue(), "result": result}
Database Query Analysis
pythonfrom sqlalchemy import event, text
from sqlalchemy.engine import Engine
import time

# ── Query counter ──────────────────────────────────────────────
query_count = 0

@event.listens_for(Engine, "before_cursor_execute")
def count_queries(conn, cursor, statement, params, context, executemany):
    global query_count
    query_count += 1
    context._start = time.perf_counter()

@event.listens_for(Engine, "after_cursor_execute")
def log_query(conn, cursor, statement, params, context, executemany):
    duration = time.perf_counter() - context._start
    if duration > 0.1:
        logger.warning(f"SLOW QUERY ({duration:.3f}s):\n{statement[:300]}")

# ── Detect N+1 queries per request ────────────────────────────
request_query_counts: dict[str, int] = {}

@app.middleware("http")
async def count_db_queries(request: Request, call_next):
    global query_count
    before   = query_count
    response = await call_next(request)
    after    = query_count
    n        = after - before

    if n > 10:
        logger.warning(f"N+1 SUSPECT: {request.url.path} made {n} queries")
    response.headers["X-DB-Queries"] = str(n)
    return response

# ── EXPLAIN ANALYZE (PostgreSQL) ───────────────────────────────
@app.get("/debug/explain", include_in_schema=False)
async def explain_query(db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        text("EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT * FROM users WHERE email = :e"),
        {"e": "test@example.com"}
    )
    plan = result.scalar()
    return {"plan": plan}

# ── Fix: eager loading to eliminate N+1 ───────────────────────
from sqlalchemy.orm import selectinload

# ✗ BAD — 1 query for posts + N queries for authors
posts = (await db.execute(select(Post))).scalars().all()
for p in posts:
    _ = p.author.username   # Triggers N lazy-load queries

# ✓ GOOD — 2 queries total regardless of N
posts = (await db.execute(
    select(Post).options(selectinload(Post.author))
)).scalars().all()
for p in posts:
    _ = p.author.username   # Already loaded
Memory Leaks
pythonimport gc, tracemalloc, linecache
from fastapi import FastAPI

app = FastAPI()

# ── Enable tracemalloc on startup ─────────────────────────────
@app.on_event("startup")
async def start_tracemalloc():
    tracemalloc.start(10)   # Keep 10 frames of traceback
    print("🔍 tracemalloc started")

# ── Memory snapshot endpoint ───────────────────────────────────
@app.get("/debug/memory", include_in_schema=False)
async def memory_report(limit: int = 20):
    snapshot = tracemalloc.take_snapshot()
    stats    = snapshot.statistics("lineno")[:limit]

    return {
        "top_allocations": [
            {
                "file":  stat.traceback[0].filename,
                "line":  stat.traceback[0].lineno,
                "size_kb": round(stat.size / 1024, 2),
                "count": stat.count,
            }
            for stat in stats
        ]
    }

# ── Common memory leak patterns ────────────────────────────────

# ✗ BAD — global list that grows forever
request_log = []   # Never cleared → OOM over time

@app.get("/bad-log")
async def bad_logging():
    request_log.append({"time": time.time(), "data": "x" * 10_000})
    return {"count": len(request_log)}

# ✓ GOOD — bounded collection
from collections import deque
request_log = deque(maxlen=1000)   # Auto-evicts oldest entries

# ✗ BAD — in-memory cache without TTL
cache = {}   # Grows without bound

# ✓ GOOD — TTL cache
from cachetools import TTLCache
cache = TTLCache(maxsize=1000, ttl=300)

# ── Force GC and check for uncollectable objects ───────────────
@app.get("/debug/gc", include_in_schema=False)
async def gc_report():
    gc.collect()
    return {
        "uncollectable": len(gc.garbage),
        "counts":        gc.get_count(),
        "thresholds":    gc.get_threshold(),
    }
Connection Pool Exhaustion
pythonfrom sqlalchemy.pool import QueuePool
from sqlalchemy import event

# ── Diagnose pool exhaustion ───────────────────────────────────
@app.get("/debug/pool", include_in_schema=False)
async def pool_status():
    pool = engine.pool
    return {
        "pool_size":    pool.size(),
        "checked_in":  pool.checkedin(),
        "checked_out": pool.checkedout(),
        "overflow":    pool.overflow(),
        "invalid":     pool.status(),
    }

# ── Detect connection leaks ────────────────────────────────────
@event.listens_for(engine.sync_engine, "checkout")
def on_checkout(dbapi_conn, conn_record, conn_proxy):
    conn_record._checkout_time = time.perf_counter()

@event.listens_for(engine.sync_engine, "checkin")
def on_checkin(dbapi_conn, conn_record):
    if hasattr(conn_record, "_checkout_time"):
        duration = time.perf_counter() - conn_record._checkout_time
        if duration > 30:
            logger.warning(f"Connection held for {duration:.1f}s — possible leak!")

# ── Tuning recommendations ────────────────────────────────────
"""
Symptoms of pool exhaustion:
  - TimeoutError: QueuePool limit of size X overflow Y reached
  - Requests hang indefinitely
  - DB connections spike

Fixes:
  1. Increase pool_size / max_overflow
  2. Reduce connection hold time (commit/close sooner)
  3. Use connection-per-request pattern (async context manager)
  4. Add pool_timeout to fail fast rather than hang
  5. Find and fix connection leaks (always use try/finally)
"""

engine = create_async_engine(
    DATABASE_URL,
    poolclass=QueuePool,
    pool_size=20,
    max_overflow=30,
    pool_timeout=10,       # Raise after 10s wait — don't hang
    pool_recycle=1800,
    pool_pre_ping=True,
)
