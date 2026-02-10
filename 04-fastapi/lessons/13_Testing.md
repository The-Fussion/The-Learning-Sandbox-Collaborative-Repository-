# Testing

## 13.1 Testing Basics

### Why Test FastAPI Apps?

Testing ensures your API works correctly and continues to work as you make changes.

**Benefits of Testing**:
- **Catch Bugs Early**: Find issues before users do
- **Confidence in Changes**: Refactor without fear
- **Documentation**: Tests show how code should work
- **Regression Prevention**: Ensure old bugs don't return
- **Better Design**: Testable code is often better structured
- **Faster Development**: Less debugging in production

**What to Test**:
- ✅ Path operations (endpoints)
- ✅ Request validation
- ✅ Response formats
- ✅ Error handling
- ✅ Authentication/authorization
- ✅ Database operations
- ✅ Business logic
- ✅ Edge cases

### Testing Tools (pytest, httpx)

**pytest** - Python testing framework:
```bash
pip install pytest
```

**httpx** - Async HTTP client for testing:
```bash
pip install httpx
```

**FastAPI TestClient** - Built on httpx:
```bash
pip install fastapi[all]  # Includes TestClient
```

**Additional Tools**:
```bash
pip install pytest-cov      # Coverage reporting
pip install pytest-asyncio  # Async test support
pip install faker          # Generate test data
pip install factory-boy    # Test fixtures
```

### TestClient

FastAPI provides `TestClient` for testing your application without running a server.

**Basic Usage**:
```python
from fastapi import FastAPI
from fastapi.testclient import TestClient

app = FastAPI()

@app.get("/")
async def read_root():
    return {"message": "Hello World"}

# Create test client
client = TestClient(app)

def test_read_root():
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == {"message": "Hello World"}
```

**How TestClient Works**:
- Wraps your FastAPI app
- Simulates HTTP requests
- No need to run server
- Synchronous by default (converts async to sync)
- Returns standard `httpx.Response` objects

**TestClient Features**:
```python
from fastapi.testclient import TestClient

client = TestClient(app)

# GET request
response = client.get("/items/1")

# POST request with JSON
response = client.post("/items/", json={"name": "Item"})

# With headers
response = client.get("/", headers={"Authorization": "Bearer token"})

# With query parameters
response = client.get("/items/", params={"skip": 0, "limit": 10})

# With cookies
response = client.get("/", cookies={"session": "abc123"})

# File upload
files = {"file": ("test.txt", b"content", "text/plain")}
response = client.post("/upload/", files=files)
```

### Test Structure

**Basic Test Structure**:
```python
# test_main.py
from fastapi import FastAPI
from fastapi.testclient import TestClient

app = FastAPI()

@app.get("/items/{item_id}")
async def read_item(item_id: int):
    return {"item_id": item_id}

client = TestClient(app)

def test_read_item():
    # Arrange (setup)
    item_id = 1
    
    # Act (execute)
    response = client.get(f"/items/{item_id}")
    
    # Assert (verify)
    assert response.status_code == 200
    assert response.json() == {"item_id": item_id}
```

**AAA Pattern**:
- **Arrange**: Set up test data and conditions
- **Act**: Execute the code being tested
- **Assert**: Verify the results

**Test Naming Conventions**:
```python
# Good test names - describe what they test
def test_get_item_returns_item_data():
    pass

def test_create_item_with_valid_data_returns_201():
    pass

def test_delete_nonexistent_item_returns_404():
    pass

def test_unauthorized_access_returns_401():
    pass
```

### Test Organization

**Project Structure**:
```
my_project/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── models.py
│   ├── schemas.py
│   └── routers/
│       ├── __init__.py
│       ├── items.py
│       └── users.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py          # Shared fixtures
│   ├── test_main.py
│   ├── test_items.py
│   └── test_users.py
├── pytest.ini               # pytest configuration
└── requirements-test.txt    # Test dependencies
```

**pytest.ini**:
```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = 
    -v
    --strict-markers
    --cov=app
    --cov-report=html
    --cov-report=term
```

**conftest.py** (Shared Fixtures):
```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from app.main import app

@pytest.fixture
def client():
    """Provide test client."""
    return TestClient(app)

@pytest.fixture
def sample_item():
    """Provide sample item data."""
    return {
        "name": "Test Item",
        "price": 10.99,
        "description": "A test item"
    }
```

**Running Tests**:
```bash
# Run all tests
pytest

# Run specific file
pytest tests/test_items.py

# Run specific test
pytest tests/test_items.py::test_create_item

# Run with coverage
pytest --cov=app

# Run with verbose output
pytest -v

# Run and stop at first failure
pytest -x

# Run only failed tests from last run
pytest --lf
```

---

## 13.2 Unit Testing

### Testing Path Operations

**Testing GET Endpoints**:
```python
# app/main.py
from fastapi import FastAPI, HTTPException

app = FastAPI()

items = {}

@app.get("/items/{item_id}")
async def read_item(item_id: int):
    if item_id not in items:
        raise HTTPException(status_code=404, detail="Item not found")
    return items[item_id]

# tests/test_items.py
from fastapi.testclient import TestClient
from app.main import app, items

client = TestClient(app)

def test_read_existing_item():
    # Setup
    items[1] = {"name": "Test Item"}
    
    # Test
    response = client.get("/items/1")
    
    # Verify
    assert response.status_code == 200
    assert response.json() == {"name": "Test Item"}

def test_read_nonexistent_item():
    response = client.get("/items/999")
    
    assert response.status_code == 404
    assert response.json() == {"detail": "Item not found"}
```

**Testing POST Endpoints**:
```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float

@app.post("/items/", status_code=201)
async def create_item(item: Item):
    item_id = len(items) + 1
    items[item_id] = item.dict()
    return {"id": item_id, **item.dict()}

def test_create_item():
    item_data = {"name": "New Item", "price": 19.99}
    
    response = client.post("/items/", json=item_data)
    
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "New Item"
    assert data["price"] == 19.99
    assert "id" in data

def test_create_item_invalid_data():
    # Missing required field
    response = client.post("/items/", json={"name": "Item"})
    
    assert response.status_code == 422  # Validation error
```

**Testing PUT/PATCH Endpoints**:
```python
@app.put("/items/{item_id}")
async def update_item(item_id: int, item: Item):
    if item_id not in items:
        raise HTTPException(status_code=404, detail="Item not found")
    items[item_id] = item.dict()
    return items[item_id]

def test_update_item():
    items[1] = {"name": "Old Name", "price": 10.0}
    
    response = client.put(
        "/items/1",
        json={"name": "New Name", "price": 15.0}
    )
    
    assert response.status_code == 200
    assert response.json()["name"] == "New Name"
```

**Testing DELETE Endpoints**:
```python
@app.delete("/items/{item_id}")
async def delete_item(item_id: int):
    if item_id not in items:
        raise HTTPException(status_code=404, detail="Item not found")
    del items[item_id]
    return {"message": "Item deleted"}

def test_delete_item():
    items[1] = {"name": "Item to delete"}
    
    response = client.delete("/items/1")
    
    assert response.status_code == 200
    assert 1 not in items
```

### Testing Dependencies

**Testing with Dependencies**:
```python
from fastapi import Depends, HTTPException

def get_current_user(token: str = Header(...)):
    if token != "valid-token":
        raise HTTPException(status_code=401, detail="Invalid token")
    return {"username": "testuser"}

@app.get("/users/me")
async def read_current_user(user: dict = Depends(get_current_user)):
    return user

def test_read_current_user_with_valid_token():
    response = client.get(
        "/users/me",
        headers={"token": "valid-token"}
    )
    
    assert response.status_code == 200
    assert response.json() == {"username": "testuser"}

def test_read_current_user_without_token():
    response = client.get("/users/me")
    
    assert response.status_code == 422  # Missing header
```

### Testing Utilities

**Testing Helper Functions**:
```python
# app/utils.py
def calculate_discount(price: float, discount_percent: float) -> float:
    """Calculate discounted price."""
    return price * (1 - discount_percent / 100)

# tests/test_utils.py
from app.utils import calculate_discount

def test_calculate_discount():
    result = calculate_discount(100.0, 20.0)
    assert result == 80.0

def test_calculate_discount_zero():
    result = calculate_discount(100.0, 0.0)
    assert result == 100.0

def test_calculate_discount_full():
    result = calculate_discount(100.0, 100.0)
    assert result == 0.0
```

### Mocking and Patching

**Using unittest.mock**:
```python
from unittest.mock import Mock, patch
import pytest

# app/services.py
import httpx

async def fetch_external_data(url: str):
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        return response.json()

# tests/test_services.py
from app.services import fetch_external_data

@pytest.mark.asyncio
async def test_fetch_external_data():
    with patch('httpx.AsyncClient.get') as mock_get:
        # Setup mock
        mock_response = Mock()
        mock_response.json.return_value = {"data": "test"}
        mock_get.return_value = mock_response
        
        # Test
        result = await fetch_external_data("https://api.example.com")
        
        # Verify
        assert result == {"data": "test"}
        mock_get.assert_called_once()
```

**Mocking Database Calls**:
```python
from unittest.mock import AsyncMock

# app/database.py
async def get_item_from_db(item_id: int):
    # Actual database query
    pass

# tests/test_database.py
@pytest.mark.asyncio
async def test_get_item_from_db():
    with patch('app.database.get_item_from_db') as mock_get:
        mock_get.return_value = {"id": 1, "name": "Test"}
        
        result = await get_item_from_db(1)
        
        assert result["name"] == "Test"
```

### Dependency Overrides in Tests

**Overriding Dependencies**:
```python
from fastapi import FastAPI, Depends

app = FastAPI()

# Original dependency
def get_db():
    db = "production_database"
    return db

@app.get("/items/")
async def read_items(db: str = Depends(get_db)):
    return {"database": db}

# Test with override
from fastapi.testclient import TestClient

def get_test_db():
    return "test_database"

def test_read_items():
    # Override dependency
    app.dependency_overrides[get_db] = get_test_db
    
    client = TestClient(app)
    response = client.get("/items/")
    
    assert response.json() == {"database": "test_database"}
    
    # Clean up
    app.dependency_overrides.clear()
```

**Fixture for Dependency Override**:
```python
# tests/conftest.py
import pytest

@pytest.fixture
def client():
    from app.main import app
    from app.dependencies import get_db
    
    def override_get_db():
        return "test_database"
    
    app.dependency_overrides[get_db] = override_get_db
    
    client = TestClient(app)
    yield client
    
    app.dependency_overrides.clear()

# tests/test_items.py
def test_items_with_test_db(client):
    response = client.get("/items/")
    assert response.status_code == 200
```

---

## 13.3 Integration Testing

### Testing with Database

**Setup Test Database**:
```python
# tests/conftest.py
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.database import Base
from app.main import app
from app.dependencies import get_db

SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"

engine = create_engine(SQLALCHEMY_DATABASE_URL, connect_args={"check_same_thread": False})
TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

@pytest.fixture
def db():
    """Create fresh database for each test."""
    Base.metadata.create_all(bind=engine)
    db = TestingSessionLocal()
    try:
        yield db
    finally:
        db.close()
        Base.metadata.drop_all(bind=engine)

@pytest.fixture
def client(db):
    """Test client with database override."""
    def override_get_db():
        try:
            yield db
        finally:
            db.close()
    
    app.dependency_overrides[get_db] = override_get_db
    
    with TestClient(app) as client:
        yield client
    
    app.dependency_overrides.clear()
```

**Testing Database Operations**:
```python
# tests/test_database_integration.py
from app.models import Item

def test_create_item_in_database(client, db):
    # Create item via API
    response = client.post(
        "/items/",
        json={"name": "Test Item", "price": 10.99}
    )
    
    assert response.status_code == 201
    
    # Verify in database
    item = db.query(Item).first()
    assert item is not None
    assert item.name == "Test Item"

def test_read_items_from_database(client, db):
    # Add items to database
    db.add(Item(name="Item 1", price=10.0))
    db.add(Item(name="Item 2", price=20.0))
    db.commit()
    
    # Read via API
    response = client.get("/items/")
    
    assert response.status_code == 200
    items = response.json()
    assert len(items) == 2
```

### Testing Authentication

**Testing Login**:
```python
def test_login_success(client):
    response = client.post(
        "/login",
        data={"username": "testuser", "password": "testpass"}
    )
    
    assert response.status_code == 200
    data = response.json()
    assert "access_token" in data
    assert data["token_type"] == "bearer"

def test_login_invalid_credentials(client):
    response = client.post(
        "/login",
        data={"username": "wrong", "password": "wrong"}
    )
    
    assert response.status_code == 401
```

**Testing Protected Endpoints**:
```python
def test_protected_endpoint_without_token(client):
    response = client.get("/protected")
    
    assert response.status_code == 401

def test_protected_endpoint_with_token(client):
    # Get token
    login_response = client.post(
        "/login",
        data={"username": "testuser", "password": "testpass"}
    )
    token = login_response.json()["access_token"]
    
    # Access protected endpoint
    response = client.get(
        "/protected",
        headers={"Authorization": f"Bearer {token}"}
    )
    
    assert response.status_code == 200
```

### Testing File Uploads

```python
from io import BytesIO

def test_upload_file(client):
    # Create test file
    file_content = b"Test file content"
    files = {
        "file": ("test.txt", BytesIO(file_content), "text/plain")
    }
    
    response = client.post("/upload/", files=files)
    
    assert response.status_code == 200
    assert response.json()["filename"] == "test.txt"

def test_upload_invalid_file_type(client):
    files = {
        "file": ("test.exe", BytesIO(b"content"), "application/x-msdownload")
    }
    
    response = client.post("/upload/", files=files)
    
    assert response.status_code == 400
```

### Testing WebSockets

```python
from fastapi import WebSocket

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    await websocket.send_text("Hello WebSocket")
    data = await websocket.receive_text()
    await websocket.send_text(f"Echo: {data}")
    await websocket.close()

def test_websocket():
    with client.websocket_connect("/ws") as websocket:
        # Receive welcome message
        data = websocket.receive_text()
        assert data == "Hello WebSocket"
        
        # Send message
        websocket.send_text("Test message")
        
        # Receive echo
        data = websocket.receive_text()
        assert data == "Echo: Test message"
```

### Testing Background Tasks

```python
from fastapi import BackgroundTasks

task_log = []

def log_task(message: str):
    task_log.append(message)

@app.post("/send-notification/")
async def send_notification(
    email: str,
    background_tasks: BackgroundTasks
):
    background_tasks.add_task(log_task, f"Email to {email}")
    return {"message": "Notification queued"}

def test_background_task(client):
    task_log.clear()
    
    response = client.post("/send-notification/?email=test@example.com")
    
    assert response.status_code == 200
    # Background tasks execute during test
    assert "Email to test@example.com" in task_log
```

---

## 13.4 Test Coverage

### pytest-cov

**Installation**:
```bash
pip install pytest-cov
```

**Running with Coverage**:
```bash
# Basic coverage
pytest --cov=app

# With HTML report
pytest --cov=app --cov-report=html

# With terminal report
pytest --cov=app --cov-report=term

# Show missing lines
pytest --cov=app --cov-report=term-missing

# Set minimum coverage
pytest --cov=app --cov-fail-under=80
```

**Configuration in pytest.ini**:
```ini
[pytest]
addopts = 
    --cov=app
    --cov-report=html
    --cov-report=term-missing
    --cov-fail-under=80
```

### Code Coverage Reports

**Terminal Report**:
```
---------- coverage: platform linux, python 3.10.0 -----------
Name                    Stmts   Miss  Cover   Missing
-----------------------------------------------------
app/__init__.py             0      0   100%
app/main.py                45      2    96%   23, 45
app/models.py              20      0   100%
app/routers/items.py       35      5    86%   12-16
app/utils.py               15      1    93%   42
-----------------------------------------------------
TOTAL                     115      8    93%
```

**HTML Report**:
```bash
pytest --cov=app --cov-report=html

# Opens htmlcov/index.html in browser
# Shows line-by-line coverage
# Highlights uncovered code in red
```

### Coverage Goals

**Coverage Targets**:
- **80%+**: Minimum acceptable coverage
- **90%+**: Good coverage
- **95%+**: Excellent coverage
- **100%**: Not always necessary or practical

**What to Focus On**:
1. **Critical paths**: User-facing features
2. **Business logic**: Core functionality
3. **Error handling**: Edge cases
4. **Security**: Authentication, authorization

**What to Exclude**:
```python
# .coveragerc
[run]
omit = 
    */tests/*
    */venv/*
    */__init__.py
    */migrations/*
    */config.py

[report]
exclude_lines =
    pragma: no cover
    def __repr__
    raise AssertionError
    raise NotImplementedError
    if __name__ == .__main__.:
    if TYPE_CHECKING:
```

### Testing Edge Cases

**Boundary Testing**:
```python
def test_pagination_boundaries(client):
    # Test minimum
    response = client.get("/items/?skip=0&limit=1")
    assert len(response.json()) <= 1
    
    # Test maximum
    response = client.get("/items/?skip=0&limit=100")
    assert len(response.json()) <= 100
    
    # Test negative (should fail validation)
    response = client.get("/items/?skip=-1")
    assert response.status_code == 422
```

**Null/Empty Testing**:
```python
def test_empty_string_handling(client):
    response = client.post("/items/", json={"name": "", "price": 10})
    assert response.status_code == 422

def test_null_handling(client):
    response = client.post("/items/", json={"name": None, "price": 10})
    assert response.status_code == 422
```

**Error Conditions**:
```python
def test_database_connection_error(client, monkeypatch):
    def mock_db_error():
        raise Exception("Database connection failed")
    
    monkeypatch.setattr("app.database.get_db", mock_db_error)
    
    response = client.get("/items/")
    assert response.status_code == 500
```

---

## 13.5 Advanced Testing

### Parameterized Tests

```python
import pytest

@pytest.mark.parametrize("item_id,expected_status", [
    (1, 200),
    (2, 200),
    (999, 404),
])
def test_read_item_various_ids(client, item_id, expected_status):
    response = client.get(f"/items/{item_id}")
    assert response.status_code == expected_status

@pytest.mark.parametrize("price,discount,expected", [
    (100, 10, 90),
    (100, 0, 100),
    (100, 100, 0),
    (50, 20, 40),
])
def test_calculate_discount(price, discount, expected):
    from app.utils import calculate_discount
    result = calculate_discount(price, discount)
    assert result == expected

@pytest.mark.parametrize("username,password,should_succeed", [
    ("alice", "correct", True),
    ("alice", "wrong", False),
    ("unknown", "password", False),
    ("", "", False),
])
def test_login_scenarios(client, username, password, should_succeed):
    response = client.post(
        "/login",
        data={"username": username, "password": password}
    )
    
    if should_succeed:
        assert response.status_code == 200
        assert "access_token" in response.json()
    else:
        assert response.status_code == 401
```

### Fixtures

**Simple Fixtures**:
```python
@pytest.fixture
def sample_item():
    return {"name": "Test Item", "price": 10.99}

@pytest.fixture
def sample_user():
    return {"username": "testuser", "email": "test@example.com"}

def test_create_item(client, sample_item):
    response = client.post("/items/", json=sample_item)
    assert response.status_code == 201
```

**Fixture Scopes**:
```python
@pytest.fixture(scope="function")  # Default - new for each test
def function_fixture():
    return "new for each test"

@pytest.fixture(scope="class")  # New for each test class
def class_fixture():
    return "shared across test class"

@pytest.fixture(scope="module")  # New for each test module
def module_fixture():
    return "shared across module"

@pytest.fixture(scope="session")  # Once per test session
def session_fixture():
    return "shared across all tests"
```

**Fixture with Setup/Teardown**:
```python
@pytest.fixture
def database():
    # Setup
    db = create_test_database()
    print("Database created")
    
    yield db  # Provide to test
    
    # Teardown
    db.close()
    print("Database closed")

def test_with_database(database):
    # database is set up and torn down automatically
    database.query("SELECT * FROM items")
```

**Fixture Dependencies**:
```python
@pytest.fixture
def database():
    return create_database()

@pytest.fixture
def user(database):
    return database.create_user("testuser")

@pytest.fixture
def authenticated_client(client, user):
    token = get_token(user)
    client.headers = {"Authorization": f"Bearer {token}"}
    return client

def test_protected_endpoint(authenticated_client):
    response = authenticated_client.get("/protected")
    assert response.status_code == 200
```

### Test Databases

**SQLite In-Memory Database**:
```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

@pytest.fixture
def test_db():
    # In-memory database
    engine = create_engine("sqlite:///:memory:")
    TestingSessionLocal = sessionmaker(bind=engine)
    
    Base.metadata.create_all(bind=engine)
    
    db = TestingSessionLocal()
    yield db
    
    db.close()
```

**PostgreSQL Test Database**:
```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

@pytest.fixture(scope="session")
def test_db_engine():
    # Create test database
    engine = create_engine("postgresql://user:pass@localhost/test_db")
    Base.metadata.create_all(bind=engine)
    
    yield engine
    
    Base.metadata.drop_all(bind=engine)
    engine.dispose()

@pytest.fixture
def test_db(test_db_engine):
    connection = test_db_engine.connect()
    transaction = connection.begin()
    
    Session = sessionmaker(bind=connection)
    session = Session()
    
    yield session
    
    session.close()
    transaction.rollback()
    connection.close()
```

### Async Testing

**pytest-asyncio**:
```bash
pip install pytest-asyncio
```

**Testing Async Functions**:
```python
import pytest

@pytest.mark.asyncio
async def test_async_function():
    result = await async_function()
    assert result == expected_value

@pytest.mark.asyncio
async def test_async_endpoint():
    from httpx import AsyncClient
    from app.main import app
    
    async with AsyncClient(app=app, base_url="http://test") as ac:
        response = await ac.get("/")
    
    assert response.status_code == 200
```

**Async Fixtures**:
```python
@pytest.fixture
async def async_client():
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac

@pytest.mark.asyncio
async def test_with_async_client(async_client):
    response = await async_client.get("/items/")
    assert response.status_code == 200
```

### Performance Testing

**Basic Performance Test**:
```python
import time

def test_response_time(client):
    start = time.time()
    response = client.get("/items/")
    duration = time.time() - start
    
    assert response.status_code == 200
    assert duration < 0.5  # Should respond within 500ms
```

**Concurrent Requests**:
```python
import concurrent.futures

def test_concurrent_requests(client):
    def make_request():
        return client.get("/items/")
    
    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
        futures = [executor.submit(make_request) for _ in range(100)]
        results = [f.result() for f in futures]
    
    # All should succeed
    assert all(r.status_code == 200 for r in results)
```

### Load Testing with Locust

**Installation**:
```bash
pip install locust
```

**locustfile.py**:
```python
from locust import HttpUser, task, between

class APIUser(HttpUser):
    wait_time = between(1, 3)  # Wait 1-3 seconds between tasks
    
    def on_start(self):
        """Run once when user starts."""
        # Login
        response = self.client.post("/login", json={
            "username": "testuser",
            "password": "testpass"
        })
        self.token = response.json()["access_token"]
    
    @task(3)  # Weight: 3x more likely than other tasks
    def read_items(self):
        """Test reading items list."""
        self.client.get(
            "/items/",
            headers={"Authorization": f"Bearer {self.token}"}
        )
    
    @task(2)
    def read_item(self):
        """Test reading single item."""
        self.client.get(
            "/items/1",
            headers={"Authorization": f"Bearer {self.token}"}
        )
    
    @task(1)
    def create_item(self):
        """Test creating item."""
        self.client.post(
            "/items/",
            json={"name": "Test", "price": 10.99},
            headers={"Authorization": f"Bearer {self.token}"}
        )
```

**Running Locust**:
```bash
# Start Locust web interface
locust -f locustfile.py

# Or headless mode
locust -f locustfile.py --headless -u 100 -r 10 --run-time 1m

# -u 100: 100 concurrent users
# -r 10: Spawn 10 users per second
# --run-time 1m: Run for 1 minute
```

**Advanced Locust Example**:
```python
from locust import HttpUser, task, between, events
import logging

class AdvancedAPIUser(HttpUser):
    wait_time = between(0.5, 2)
    
    def on_start(self):
        """Setup before tasks."""
        self.login()
    
    def login(self):
        """Authenticate user."""
        response = self.client.post("/login", json={
            "username": "user",
            "password": "pass"
        })
        
        if response.status_code == 200:
            self.token = response.json()["access_token"]
        else:
            logging.error("Login failed")
    
    @task
    def read_and_update(self):
        """Complex workflow."""
        # Read item
        with self.client.get(
            "/items/1",
            catch_response=True,
            headers={"Authorization": f"Bearer {self.token}"}
        ) as response:
            if response.status_code == 200:
                item = response.json()
                response.success()
            else:
                response.failure(f"Got status {response.status_code}")
                return
        
        # Update item
        item["price"] = item["price"] + 1
        
        with self.client.put(
            "/items/1",
            json=item,
            catch_response=True,
            headers={"Authorization": f"Bearer {self.token}"}
        ) as response:
            if response.status_code == 200:
                response.success()
            else:
                response.failure(f"Update failed: {response.status_code}")

@events.test_start.add_listener
def on_test_start(environment, **kwargs):
    """Run before test starts."""
    logging.info("Load test starting...")

@events.test_stop.add_listener
def on_test_stop(environment, **kwargs):
    """Run after test stops."""
    logging.info("Load test complete!")
```

---

## Complete Example: Comprehensive Test Suite

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.main import app
from app.database import Base, get_db
from app.models import User, Item

SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"
engine = create_engine(SQLALCHEMY_DATABASE_URL, connect_args={"check_same_thread": False})
TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

@pytest.fixture(scope="function")
def db():
    """Create fresh database for each test."""
    Base.metadata.create_all(bind=engine)
    db = TestingSessionLocal()
    try:
        yield db
    finally:
        db.close()
        Base.metadata.drop_all(bind=engine)

@pytest.fixture(scope="function")
def client(db):
    """Test client with database override."""
    def override_get_db():
        try:
            yield db
        finally:
            pass
    
    app.dependency_overrides[get_db] = override_get_db
    
    with TestClient(app) as test_client:
        yield test_client
    
    app.dependency_overrides.clear()

@pytest.fixture
def test_user(db):
    """Create test user."""
    user = User(username="testuser", email="test@example.com")
    user.set_password("testpass")
    db.add(user)
    db.commit()
    db.refresh(user)
    return user

@pytest.fixture
def auth_headers(client, test_user):
    """Get authentication headers."""
    response = client.post(
        "/login",
        data={"username": "testuser", "password": "testpass"}
    )
    token = response.json()["access_token"]
    return {"Authorization": f"Bearer {token}"}

# tests/test_items.py
import pytest

class TestItemsEndpoints:
    """Test items CRUD operations."""
    
    def test_create_item(self, client, auth_headers):
        """Test creating a new item."""
        response = client.post(
            "/items/",
            json={"name": "Test Item", "price": 10.99},
            headers=auth_headers
        )
        
        assert response.status_code == 201
        data = response.json()
        assert data["name"] == "Test Item"
        assert data["price"] == 10.99
    
    @pytest.mark.parametrize("name,price,expected_status", [
        ("Valid", 10.0, 201),
        ("", 10.0, 422),  # Empty name
        ("Valid", -5.0, 422),  # Negative price
        ("Valid", 0, 422),  # Zero price
    ])
    def test_create_item_validation(self, client, auth_headers, name, price, expected_status):
        """Test item validation."""
        response = client.post(
            "/items/",
            json={"name": name, "price": price},
            headers=auth_headers
        )
        assert response.status_code == expected_status
    
    def test_read_items(self, client, db, auth_headers):
        """Test reading items list."""
        # Create test items
        db.add(Item(name="Item 1", price=10.0))
        db.add(Item(name="Item 2", price=20.0))
        db.commit()
        
        response = client.get("/items/", headers=auth_headers)
        
        assert response.status_code == 200
        items = response.json()
        assert len(items) == 2
    
    def test_read_item(self, client, db, auth_headers):
        """Test reading single item."""
        item = Item(name="Test Item", price=15.0)
        db.add(item)
        db.commit()
        
        response = client.get(f"/items/{item.id}", headers=auth_headers)
        
        assert response.status_code == 200
        assert response.json()["name"] == "Test Item"
    
    def test_update_item(self, client, db, auth_headers):
        """Test updating item."""
        item = Item(name="Old Name", price=10.0)
        db.add(item)
        db.commit()
        
        response = client.put(
            f"/items/{item.id}",
            json={"name": "New Name", "price": 15.0},
            headers=auth_headers
        )
        
        assert response.status_code == 200
        assert response.json()["name"] == "New Name"
    
    def test_delete_item(self, client, db, auth_headers):
        """Test deleting item."""
        item = Item(name="To Delete", price=10.0)
        db.add(item)
        db.commit()
        item_id = item.id
        
        response = client.delete(f"/items/{item_id}", headers=auth_headers)
        
        assert response.status_code == 200
        
        # Verify deleted
        assert db.query(Item).filter_by(id=item_id).first() is None

# tests/test_auth.py
class TestAuthentication:
    """Test authentication endpoints."""
    
    def test_login_success(self, client, test_user):
        """Test successful login."""
        response = client.post(
            "/login",
            data={"username": "testuser", "password": "testpass"}
        )
        
        assert response.status_code == 200
        data = response.json()
        assert "access_token" in data
        assert data["token_type"] == "bearer"
    
    def test_login_wrong_password(self, client, test_user):
        """Test login with wrong password."""
        response = client.post(
            "/login",
            data={"username": "testuser", "password": "wrong"}
        )
        
        assert response.status_code == 401
    
    def test_protected_endpoint_without_auth(self, client):
        """Test accessing protected endpoint without token."""
        response = client.get("/items/")
        
        assert response.status_code == 401
    
    def test_protected_endpoint_with_auth(self, client, auth_headers):
        """Test accessing protected endpoint with token."""
        response = client.get("/items/", headers=auth_headers)
        
        assert response.status_code == 200

# Run tests:
# pytest -v --cov=app --cov-report=html
```

---

## Summary

You've learned about Testing in FastAPI:

1. **Testing Basics**: Why test, testing tools, TestClient, test structure, and organization
2. **Unit Testing**: Testing endpoints, dependencies, utilities, mocking, and dependency overrides
3. **Integration Testing**: Testing with databases, authentication, file uploads, WebSockets, and background tasks
4. **Test Coverage**: pytest-cov, coverage reports, goals, and edge cases
5. **Advanced Testing**: Parameterized tests, fixtures, test databases, async testing, and load testing

**Key Takeaways**:
- Use **TestClient** for testing without running a server
- **Override dependencies** for testing with test databases
- Aim for **80%+ coverage** of critical code
- Test **edge cases** and error conditions
- Use **fixtures** for reusable test setup
- **Parameterize tests** to reduce duplication
- Use **Locust** for load testing

Testing ensures your FastAPI application works correctly and continues to work as it grows!