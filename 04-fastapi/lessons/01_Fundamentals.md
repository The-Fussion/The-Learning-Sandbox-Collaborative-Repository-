# FastAPI Fundamentals

## 1.1 Introduction to FastAPI

### What is FastAPI and Why Use It?

FastAPI is a modern, high-performance web framework for building APIs with Python 3.8+ based on standard Python type hints. Created by Sebastián Ramírez in 2018, it has quickly become one of the most popular frameworks for API development.

**Key Features**:
- **High Performance**: One of the fastest Python frameworks available, on par with NodeJS and Go
- **Type Safety**: Built on Python type hints with automatic validation via Pydantic
- **Automatic Documentation**: Interactive API docs (Swagger UI and ReDoc) generated automatically
- **Developer Experience**: Excellent autocomplete support, fewer bugs, reduced development time
- **Production Ready**: Used by companies like Microsoft, Uber, and Netflix

**Why Choose FastAPI**:
- Reduces development time by up to 40%
- Minimizes human errors through automatic validation
- Native async/await support for high concurrency
- Standards-based (OpenAPI, JSON Schema)

### FastAPI vs Flask vs Django

| Aspect | FastAPI | Flask | Django |
|--------|---------|-------|--------|
| Performance | Very High (async) | Moderate (sync) | Moderate (sync) |
| Type Hints | Built-in, required | Optional | Optional |
| Documentation | Automatic | Manual/Extensions | Manual/DRF |
| Validation | Automatic (Pydantic) | Manual/Extensions | Manual/Serializers |
| Async Support | Native | Limited | Partial (3.0+) |
| Learning Curve | Moderate | Easy | Steep |
| Use Case | Modern APIs | Simple apps | Full-stack web apps |

**When to choose FastAPI**:
- Building modern REST APIs
- Microservices architecture
- High-performance requirements
- Real-time applications (WebSockets)
- Data science API deployments

**When to choose Flask**:
- Legacy projects
- Simple synchronous applications
- Team familiar with Flask

**When to choose Django**:
- Full-stack web applications
- Admin panels and CMS
- Rapid prototyping with ORM

### ASGI vs WSGI Understanding

**WSGI (Web Server Gateway Interface)**:
- Traditional Python web server standard (2003)
- Synchronous only - one request per thread
- Used by Flask, Django (traditional)
- Blocking I/O operations

**ASGI (Asynchronous Server Gateway Interface)**:
- Modern async-capable successor to WSGI
- Supports both sync and async code
- Multiple concurrent requests per worker
- Non-blocking I/O operations
- Supports WebSockets, HTTP/2
- Used by FastAPI, Starlette, modern Django

**Why ASGI Matters for FastAPI**:
```python
# WSGI (blocking)
def fetch_data():
    time.sleep(2)  # Blocks entire worker
    return "data"

# ASGI (non-blocking)
async def fetch_data():
    await asyncio.sleep(2)  # Other requests can be processed
    return "data"
```

ASGI enables FastAPI to handle thousands of concurrent connections efficiently, making it ideal for:
- Real-time applications
- Microservices with many I/O operations
- High-traffic applications
- WebSocket connections

### When to Choose FastAPI

**Ideal Use Cases**:
1. RESTful APIs with automatic documentation
2. Microservices architecture
3. Real-time applications (WebSockets)
4. Machine Learning model serving
5. High-concurrency scenarios
6. GraphQL APIs
7. Rapid prototyping with type safety

**When FastAPI Might Not Be Best**:
1. Full-stack web apps with server-side rendering → Django
2. Team deeply experienced with Flask and async isn't needed → Flask
3. Legacy WSGI-only infrastructure
4. Simple static sites without API needs
5. Absolute minimum dependencies → Starlette

### FastAPI Ecosystem and Community

**Core Dependencies**:
- Starlette - ASGI framework foundation
- Pydantic - Data validation using type hints
- Uvicorn - Lightning-fast ASGI server

**Popular Extensions**:
- SQLAlchemy / Tortoise ORM - Database integration
- FastAPI-Users - Authentication and user management
- FastAPI-Cache - Caching utilities
- Celery - Background task processing
- HTTPX - Async HTTP client

**Community Resources**:
- Official Documentation: https://fastapi.tiangolo.com
- GitHub: 75,000+ stars, very active
- Discord and Gitter communities
- Used by major companies worldwide

---

## 1.2 Python Prerequisites

### Type Hints and Type Annotations

Type hints enable FastAPI's automatic validation and documentation.

**Basic Type Hints**:
```python
# Variables
name: str = "Alice"
age: int = 30
price: float = 19.99
is_active: bool = True

# Functions
def greet(name: str) -> str:
    return f"Hello, {name}!"

def add(a: int, b: int) -> int:
    return a + b
```

**Complex Types**:
```python
from typing import List, Dict, Set, Tuple, Optional, Union

names: List[str] = ["Alice", "Bob"]
user_data: Dict[str, int] = {"age": 30}
coordinates: Tuple[float, float] = (10.5, 20.3)
middle_name: Optional[str] = None  # Can be str or None
identifier: Union[int, str] = "user_123"  # Can be int or str
```

**Advanced Type Hints**:
```python
from typing import Callable, TypeVar, Generic, Literal

# Function types
Processor = Callable[[int, int], int]

# Literal values
Mode = Literal["read", "write", "append"]

# Type aliases
UserId = int
UserMap = Dict[UserId, str]
```

### Python 3.7+ Features

**f-strings (Python 3.6+)**:
```python
name = "Alice"
age = 30
message = f"Hello, {name}. You are {age} years old."

# Expressions in f-strings
price = 19.99
total = f"Total: ${price * 1.08:.2f}"  # Total: $21.59

# Debug formatting (Python 3.8+)
x = 10
print(f"{x=}")  # Output: x=10
```

**Dataclasses (Python 3.7+)**:
```python
from dataclasses import dataclass, field
from typing import List

@dataclass
class User:
    name: str
    age: int
    emails: List[str] = field(default_factory=list)
    active: bool = True

user = User(name="Alice", age=30)
# User(name='Alice', age=30, emails=[], active=True)
```

**Dictionary Merge (Python 3.9+)**:
```python
base = {"host": "localhost", "port": 8000}
user = {"port": 9000, "debug": True}
config = base | user  # {'host': 'localhost', 'port': 9000, 'debug': True}
```

**Type Union Operator (Python 3.10+)**:
```python
# Old way
from typing import Union
def process(value: Union[int, str]) -> str:
    pass

# Python 3.10+
def process(value: int | str) -> str:
    pass
```

### Async/Await Fundamentals

**Synchronous vs Asynchronous**:
```python
import time
import asyncio

# Synchronous (blocking)
def sync_task():
    time.sleep(2)  # Blocks for 2 seconds
    return "done"

# Asynchronous (non-blocking)
async def async_task():
    await asyncio.sleep(2)  # Doesn't block
    return "done"
```

**Basic Async Patterns**:
```python
import asyncio

async def fetch_data(id: int):
    await asyncio.sleep(1)
    return f"Data {id}"

async def main():
    # Sequential
    result1 = await fetch_data(1)
    result2 = await fetch_data(2)
    # Takes 2 seconds total
    
    # Concurrent
    results = await asyncio.gather(
        fetch_data(1),
        fetch_data(2),
        fetch_data(3)
    )
    # Takes 1 second total (runs concurrently)

asyncio.run(main())
```

**When to Use Async**:
- ✅ HTTP requests to external APIs
- ✅ Database queries (with async drivers)
- ✅ File I/O (with async libraries)
- ✅ WebSockets
- ❌ CPU-intensive computations
- ❌ Sync-only libraries

### Decorators Deep Dive

**Basic Decorator**:
```python
from functools import wraps
from time import time

def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time()
        result = func(*args, **kwargs)
        end = time()
        print(f"{func.__name__} took {end - start:.2f}s")
        return result
    return wrapper

@timer
def slow_function():
    time.sleep(2)
    return "Done"
```

**Decorators with Arguments**:
```python
def repeat(times: int):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(times=3)
def greet(name: str):
    print(f"Hello, {name}!")
```

**FastAPI Route Decorators**:
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/{item_id}")  # Decorator defines HTTP method and path
async def read_item(item_id: int):
    return {"item_id": item_id}
```

### Context Managers

**Basic Context Manager**:
```python
# File handling
with open("data.txt", "r") as file:
    content = file.read()
    # File automatically closed

# Multiple contexts
with open("in.txt", "r") as infile, open("out.txt", "w") as outfile:
    outfile.write(infile.read())
```

**Creating Context Managers**:
```python
from contextlib import contextmanager

@contextmanager
def database_connection():
    # Setup
    conn = "Database connected"
    print("Opening connection")
    try:
        yield conn  # Code in 'with' block runs here
    finally:
        # Cleanup
        print("Closing connection")

with database_connection() as conn:
    print(f"Using {conn}")
```

**Async Context Managers**:
```python
class AsyncDatabase:
    async def __aenter__(self):
        print("Connecting...")
        await asyncio.sleep(1)
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        print("Closing...")
        await asyncio.sleep(1)

async def main():
    async with AsyncDatabase() as db:
        print("Using database")
```

### Generators and Iterators

**Basic Generator**:
```python
def count_up_to(max_count: int):
    count = 1
    while count <= max_count:
        yield count
        count += 1

for num in count_up_to(5):
    print(num)  # 1, 2, 3, 4, 5
```

**Generator Expressions**:
```python
# Memory efficient - values generated on demand
squares = (x**2 for x in range(10))
print(next(squares))  # 0
print(next(squares))  # 1
```

**Async Generators**:
```python
async def async_counter(max_count: int):
    count = 0
    while count < max_count:
        await asyncio.sleep(0.1)
        count += 1
        yield count

async def main():
    async for num in async_counter(5):
        print(num)
```

**Generators in FastAPI**:
```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

def generate_data():
    for i in range(1000):
        yield f"Line {i}\n"

@app.get("/stream")
def stream_data():
    return StreamingResponse(generate_data(), media_type="text/plain")
```

---

## 1.3 Installation & Setup

### Installing FastAPI

**Basic Installation**:
```bash
# FastAPI only
pip install fastapi

# FastAPI with all dependencies
pip install "fastapi[all]"

# Minimal production install (recommended)
pip install fastapi
pip install "uvicorn[standard]"
```

**Verify Installation**:
```bash
python -c "import fastapi; print(fastapi.__version__)"
uvicorn --version
```

### Installing Uvicorn (ASGI Server)

**Installation Options**:
```bash
# Standard (recommended - includes performance optimizations)
pip install "uvicorn[standard]"

# Minimal
pip install uvicorn
```

**Running Uvicorn**:
```bash
# Basic
uvicorn main:app

# Development (auto-reload)
uvicorn main:app --reload

# Custom host/port
uvicorn main:app --host 0.0.0.0 --port 8000

# Production (multiple workers)
uvicorn main:app --workers 4
```

### Virtual Environments

**Using venv (Built-in)**:
```bash
# Create
python -m venv venv

# Activate
source venv/bin/activate  # macOS/Linux
venv\Scripts\activate     # Windows

# Install dependencies
pip install fastapi "uvicorn[standard]"

# Save dependencies
pip freeze > requirements.txt

# Deactivate
deactivate
```

**Using Conda**:
```bash
# Create environment
conda create -n fastapi-env python=3.11

# Activate
conda activate fastapi-env

# Install
pip install fastapi "uvicorn[standard]"

# Export
conda env export > environment.yml
```

**Using Poetry**:
```bash
# Install Poetry
curl -sSL https://install.python-poetry.org | python3 -

# Initialize project
poetry init

# Add dependencies
poetry add fastapi "uvicorn[standard]"

# Install
poetry install

# Run
poetry run uvicorn main:app --reload
```

### IDE Configuration

**VSCode Settings** (`.vscode/settings.json`):
```json
{
  "python.defaultInterpreterPath": "${workspaceFolder}/venv/bin/python",
  "python.linting.enabled": true,
  "python.linting.ruffEnabled": true,
  "python.formatting.provider": "black",
  "editor.formatOnSave": true,
  "[python]": {
    "editor.defaultFormatter": "ms-python.black-formatter",
    "editor.tabSize": 4
  },
  "python.testing.pytestEnabled": true
}
```

**VSCode Launch Configuration** (`.vscode/launch.json`):
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "FastAPI",
      "type": "python",
      "request": "launch",
      "module": "uvicorn",
      "args": [
        "main:app",
        "--reload",
        "--host", "0.0.0.0",
        "--port", "8000"
      ],
      "jinja": true
    }
  ]
}
```

**PyCharm Configuration**:
1. Settings → Project → Python Interpreter → Add virtualenv
2. Run → Edit Configurations → Add Python
3. Set Script path to uvicorn
4. Parameters: `main:app --reload`

### Project Structure Best Practices

**Small Project**:
```
my-project/
├── venv/
├── main.py
├── requirements.txt
├── .env
├── .gitignore
└── README.md
```

**Medium Project**:
```
my-project/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── config.py
│   ├── models.py
│   └── routers/
│       ├── __init__.py
│       ├── users.py
│       └── items.py
├── tests/
│   ├── __init__.py
│   └── test_main.py
├── venv/
├── requirements.txt
├── .env
└── README.md
```

**Large Project (Production)**:
```
my-project/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── api/
│   │   └── v1/
│   │       └── endpoints/
│   ├── core/
│   │   ├── config.py
│   │   └── security.py
│   ├── models/
│   ├── schemas/
│   ├── crud/
│   └── db/
├── tests/
├── alembic/
├── .vscode/
├── requirements.txt
├── pytest.ini
├── Dockerfile
└── docker-compose.yml
```

**Essential Files**:

`.gitignore`:
```gitignore
__pycache__/
*.py[cod]
venv/
.env
.pytest_cache/
*.db
```

`requirements.txt`:
```txt
fastapi==0.109.0
uvicorn[standard]==0.27.0
pydantic==2.5.0
pydantic-settings==2.1.0
```

`.env.example`:
```env
PROJECT_NAME="My API"
DEBUG=True
DATABASE_URL=postgresql://user:pass@localhost/db
SECRET_KEY=your-secret-key
```

---

## 1.4 First FastAPI Application

### Hello World Application

**Minimal Application** (`main.py`):
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"message": "Hello World"}
```

**Enhanced Application**:
```python
from fastapi import FastAPI

app = FastAPI(
    title="My First API",
    description="Learning FastAPI",
    version="1.0.0"
)

@app.get("/")
def read_root():
    """Root endpoint - returns welcome message."""
    return {"message": "Hello World"}

@app.get("/hello/{name}")
def greet(name: str):
    """Greet a user by name."""
    return {"message": f"Hello, {name}!"}

@app.get("/items/{item_id}")
def read_item(item_id: int, q: str = None):
    """Get item by ID with optional query."""
    return {"item_id": item_id, "q": q}
```

### Running with Uvicorn

**Command Line**:
```bash
# Basic run
uvicorn main:app

# With auto-reload (development)
uvicorn main:app --reload

# Custom host and port
uvicorn main:app --host 0.0.0.0 --port 8080

# All options
uvicorn main:app \
  --reload \
  --host 0.0.0.0 \
  --port 8000 \
  --log-level info
```

**From Python Code**:
```python
from fastapi import FastAPI
import uvicorn

app = FastAPI()

@app.get("/")
def read_root():
    return {"message": "Hello World"}

if __name__ == "__main__":
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=8000,
        reload=True
    )
```

Then run: `python main.py`

### Hot Reload in Development

**Enable Hot Reload**:
```bash
uvicorn main:app --reload
```

**How It Works**:
1. Uvicorn watches your files for changes
2. When you save a file, it detects the change
3. Server automatically restarts with new code
4. No manual restart needed

**Example**:
```
INFO:     Will watch for changes in these directories: ['/path/to/project']
INFO:     Uvicorn running on http://127.0.0.1:8000
INFO:     Started reloader process [12345]

# After you modify and save main.py:
WARNING:  Detected file change in 'main.py'. Reloading...
INFO:     Shutting down
INFO:     Started server process [12346]
INFO:     Application startup complete.
```

**Important Notes**:
- Only use `--reload` in development
- Never use in production
- May be slower on Windows

### Application Lifecycle

**Modern Lifespan (Recommended)**:
```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

# Simulated resources
ml_models = {}

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: runs before application starts
    print("🚀 Application starting...")
    ml_models["sentiment"] = "Sentiment Model Loaded"
    # Initialize database, load models, etc.
    
    yield  # Application runs here
    
    # Shutdown: runs after application stops
    print("🛑 Application shutting down...")
    ml_models.clear()
    # Close connections, cleanup, etc.

app = FastAPI(lifespan=lifespan)

@app.get("/")
def read_root():
    return {"status": "running", "models": list(ml_models.keys())}
```

**Legacy Events (Still Supported)**:
```python
from fastapi import FastAPI

app = FastAPI()

@app.on_event("startup")
async def startup_event():
    print("Starting up...")
    # Initialize resources

@app.on_event("shutdown")
async def shutdown_event():
    print("Shutting down...")
    # Cleanup resources
```

### Automatic Interactive API Docs (Swagger UI)

FastAPI automatically generates interactive documentation at:
```
http://localhost:8000/docs
```

**Features**:
- List of all endpoints
- Interactive testing (try requests in browser)
- Request/response schemas
- Parameter descriptions
- Authentication testing

**Example with Rich Documentation**:
```python
from fastapi import FastAPI, Query
from pydantic import BaseModel, Field

app = FastAPI(
    title="User API",
    description="API for managing users",
    version="1.0.0"
)

class User(BaseModel):
    name: str = Field(..., example="Alice")
    email: str = Field(..., example="alice@example.com")
    age: int = Field(..., ge=0, le=120, example=30)

@app.post("/users/", tags=["users"], summary="Create a user")
async def create_user(user: User):
    """
    Create a new user with:
    - **name**: User's full name
    - **email**: Valid email address
    - **age**: User's age (0-120)
    """
    return user
```

### Alternative Documentation (ReDoc)

ReDoc provides an alternative documentation interface at:
```
http://localhost:8000/redoc
```

**ReDoc vs Swagger UI**:
| Feature | Swagger UI | ReDoc |
|---------|-----------|-------|
| Interactive Testing | ✅ Yes | ❌ No |
| Design | Standard | Modern, clean |
| Search | Basic | Advanced |
| Best For | Testing | Reading docs |

**Customizing Docs**:
```python
from fastapi import FastAPI

app = FastAPI(
    docs_url="/documentation",  # Change Swagger URL
    redoc_url="/docs",          # Change ReDoc URL
    # docs_url=None,            # Disable Swagger
    # redoc_url=None,           # Disable ReDoc
)
```

**Conditional Documentation**:
```python
import os

app = FastAPI(
    docs_url="/docs" if os.getenv("ENV") == "dev" else None,
    redoc_url="/redoc" if os.getenv("ENV") == "dev" else None
)
```

### OpenAPI Schema Generation

FastAPI generates an OpenAPI 3.0+ schema automatically at:
```
http://localhost:8000/openapi.json
```

**What is OpenAPI?**
- Standard specification for describing REST APIs
- Defines endpoints, request/response formats, authentication
- Machine-readable, can generate client SDKs

**Example Schema** (for simple endpoint):
```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float

@app.post("/items/")
async def create_item(item: Item):
    return item
```

Generates:
```json
{
  "openapi": "3.1.0",
  "info": {"title": "FastAPI", "version": "0.1.0"},
  "paths": {
    "/items/": {
      "post": {
        "summary": "Create Item",
        "requestBody": {
          "content": {
            "application/json": {
              "schema": {"$ref": "#/components/schemas/Item"}
            }
          }
        }
      }
    }
  },
  "components": {
    "schemas": {
      "Item": {
        "properties": {
          "name": {"type": "string"},
          "price": {"type": "number"}
        },
        "required": ["name", "price"]
      }
    }
  }
}
```

**Using OpenAPI Schema**:
- Generate client SDKs (TypeScript, Python, Java)
- Import into Postman/Insomnia
- API contract testing
- Documentation generation

---

## Complete Example: Putting It All Together

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, Query
from pydantic import BaseModel, Field
from typing import List, Optional
import uvicorn

# Resources
items_db: List[dict] = []

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    print("🚀 Starting application...")
    items_db.append({"id": 1, "name": "Sample Item", "price": 10.99})
    yield
    # Shutdown
    print("🛑 Shutting down...")
    items_db.clear()

app = FastAPI(
    title="Complete FastAPI Example",
    description="Demonstrates all fundamental concepts",
    version="1.0.0",
    lifespan=lifespan
)

class Item(BaseModel):
    name: str = Field(..., min_length=1, example="Widget")
    price: float = Field(..., gt=0, example=19.99)
    description: Optional[str] = Field(None, example="A useful widget")

@app.get("/", tags=["root"])
async def root():
    """Root endpoint with API information."""
    return {
        "message": "Welcome to FastAPI",
        "docs": "/docs",
        "redoc": "/redoc"
    }

@app.get("/items/", tags=["items"], summary="List all items")
async def list_items(
    skip: int = Query(0, ge=0),
    limit: int = Query(10, ge=1, le=100)
):
    """Retrieve items with pagination."""
    return items_db[skip : skip + limit]

@app.post("/items/", tags=["items"], summary="Create item")
async def create_item(item: Item):
    """Create a new item."""
    new_item = {"id": len(items_db) + 1, **item.dict()}
    items_db.append(new_item)
    return new_item

@app.get("/items/{item_id}", tags=["items"])
async def read_item(item_id: int):
    """Get item by ID."""
    for item in items_db:
        if item["id"] == item_id:
            return item
    return {"error": "Item not found"}

if __name__ == "__main__":
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=8000,
        reload=True
    )
```

**To Run**:
```bash
python main.py
```

**Then Visit**:
- API: http://localhost:8000/
- Swagger UI: http://localhost:8000/docs
- ReDoc: http://localhost:8000/redoc
- OpenAPI: http://localhost:8000/openapi.json

---

## Summary

You've learned the fundamentals of FastAPI:

1. **Introduction**: What FastAPI is, when to use it, and how it compares to other frameworks
2. **Python Prerequisites**: Type hints, async/await, decorators, and modern Python features
3. **Installation & Setup**: Environment setup, project structure, and IDE configuration
4. **First Application**: Creating, running, and documenting your first FastAPI application

These fundamentals provide the foundation for building production-ready APIs with FastAPI. The automatic documentation, type safety, and async support make it an excellent choice for modern API development.

**Next Steps**:
- Explore path and query parameters
- Learn request body handling with Pydantic
- Implement authentication and authorization
- Connect to databases
- Deploy to production