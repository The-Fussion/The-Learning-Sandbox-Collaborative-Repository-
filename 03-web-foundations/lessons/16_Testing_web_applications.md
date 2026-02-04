# Testing Web Applications in Python

## Table of Contents
1. [Introduction to Testing](#introduction-to-testing)
2. [Testing Fundamentals](#testing-fundamentals)
3. [Unit Testing](#unit-testing)
4. [Integration Testing](#integration-testing)
5. [Testing Flask Applications](#testing-flask-applications)
6. [Testing FastAPI Applications](#testing-fastapi-applications)
7. [Testing Django Applications](#testing-django-applications)
8. [Database Testing](#database-testing)
9. [API Testing](#api-testing)
10. [End-to-End Testing with Selenium](#end-to-end-testing-with-selenium)
11. [Mocking and Fixtures](#mocking-and-fixtures)
12. [Test Coverage](#test-coverage)
13. [Performance Testing](#performance-testing)
14. [Security Testing](#security-testing)
15. [Continuous Integration](#continuous-integration)
16. [Best Practices](#best-practices)

---

## Introduction to Testing

### Why Test?

```python
"""
Benefits of Testing:

1. Quality Assurance
   - Catch bugs before production
   - Ensure features work as expected
   - Maintain code quality

2. Refactoring Confidence
   - Safely modify code
   - Detect regressions
   - Facilitate refactoring

3. Documentation
   - Tests serve as examples
   - Demonstrate expected behavior
   - Living documentation

4. Design Improvement
   - Test-driven development (TDD)
   - Better architecture
   - Loose coupling

5. Faster Development
   - Less debugging time
   - Faster feedback loop
   - Reduced maintenance costs

6. Confidence
   - Deploy with confidence
   - Fearless releases
   - Better sleep at night
"""
```

### Testing Pyramid

```python
"""
Testing Pyramid (from bottom to top):

                    /\
                   /  \
                  / E2E \          ← Few, slow, expensive
                 /______\
                /        \
               /   API    \        ← Some, medium speed
              /____________\
             /              \
            /   INTEGRATION  \    ← More, faster
           /___________________\
          /                     \
         /      UNIT TESTS       \  ← Many, fast, cheap
        /_________________________\

Unit Tests (70%):
- Test individual functions/methods
- Fast execution
- Easy to write and maintain
- High coverage

Integration Tests (20%):
- Test component interactions
- Database, API, services
- Medium execution time
- Test real integrations

End-to-End Tests (10%):
- Test complete user flows
- Browser automation
- Slow execution
- Most realistic
"""
```

---

## Testing Fundamentals

### Test Structure (AAA Pattern)

```python
import pytest

def test_user_registration():
    """
    AAA Pattern:
    - Arrange: Set up test data and conditions
    - Act: Execute the code being tested
    - Assert: Verify the results
    """
    
    # Arrange
    username = "testuser"
    email = "test@example.com"
    password = "SecurePassword123!"
    
    # Act
    user = register_user(username, email, password)
    
    # Assert
    assert user.username == username
    assert user.email == email
    assert user.is_active is True
```

### Test Naming Conventions

```python
"""
Good Test Names:

✅ test_user_can_register_with_valid_email
✅ test_login_fails_with_incorrect_password
✅ test_order_total_includes_tax
✅ test_admin_can_delete_any_post
✅ test_api_returns_404_for_nonexistent_resource

Bad Test Names:

❌ test_user
❌ test1
❌ test_function
❌ it_works

Pattern: test_<what>_<when>_<expected_result>
"""
```

---

## Unit Testing

### Using unittest

```python
# test_calculator.py

import unittest

class Calculator:
    """Simple calculator class."""
    
    def add(self, a, b):
        return a + b
    
    def subtract(self, a, b):
        return a - b
    
    def multiply(self, a, b):
        return a * b
    
    def divide(self, a, b):
        if b == 0:
            raise ValueError("Cannot divide by zero")
        return a / b

class TestCalculator(unittest.TestCase):
    """Test calculator functionality."""
    
    def setUp(self):
        """Set up test fixtures."""
        self.calc = Calculator()
    
    def tearDown(self):
        """Clean up after tests."""
        self.calc = None
    
    def test_add(self):
        """Test addition."""
        result = self.calc.add(2, 3)
        self.assertEqual(result, 5)
    
    def test_subtract(self):
        """Test subtraction."""
        result = self.calc.subtract(5, 3)
        self.assertEqual(result, 2)
    
    def test_multiply(self):
        """Test multiplication."""
        result = self.calc.multiply(3, 4)
        self.assertEqual(result, 12)
    
    def test_divide(self):
        """Test division."""
        result = self.calc.divide(10, 2)
        self.assertEqual(result, 5)
    
    def test_divide_by_zero(self):
        """Test division by zero raises error."""
        with self.assertRaises(ValueError):
            self.calc.divide(10, 0)
    
    def test_add_negative_numbers(self):
        """Test adding negative numbers."""
        result = self.calc.add(-5, -3)
        self.assertEqual(result, -8)
    
    def test_multiple_operations(self):
        """Test multiple operations."""
        result = self.calc.add(2, 3)
        result = self.calc.multiply(result, 2)
        self.assertEqual(result, 10)

if __name__ == '__main__':
    unittest.main()
```

### Using pytest

```python
# test_calculator_pytest.py

import pytest

class Calculator:
    def add(self, a, b):
        return a + b
    
    def subtract(self, a, b):
        return a - b
    
    def multiply(self, a, b):
        return a * b
    
    def divide(self, a, b):
        if b == 0:
            raise ValueError("Cannot divide by zero")
        return a / b

# Fixtures
@pytest.fixture
def calculator():
    """Create calculator instance."""
    return Calculator()

# Basic tests
def test_add(calculator):
    """Test addition."""
    assert calculator.add(2, 3) == 5

def test_subtract(calculator):
    """Test subtraction."""
    assert calculator.subtract(5, 3) == 2

def test_multiply(calculator):
    """Test multiplication."""
    assert calculator.multiply(3, 4) == 12

def test_divide(calculator):
    """Test division."""
    assert calculator.divide(10, 2) == 5

def test_divide_by_zero(calculator):
    """Test division by zero."""
    with pytest.raises(ValueError, match="Cannot divide by zero"):
        calculator.divide(10, 0)

# Parametrized tests
@pytest.mark.parametrize("a,b,expected", [
    (2, 3, 5),
    (0, 0, 0),
    (-1, 1, 0),
    (100, 200, 300),
    (-5, -3, -8)
])
def test_add_parametrized(calculator, a, b, expected):
    """Test addition with multiple inputs."""
    assert calculator.add(a, b) == expected

@pytest.mark.parametrize("a,b,expected", [
    (10, 2, 5),
    (20, 4, 5),
    (100, 10, 10),
    (1, 1, 1)
])
def test_divide_parametrized(calculator, a, b, expected):
    """Test division with multiple inputs."""
    assert calculator.divide(a, b) == expected

# Test classes
class TestCalculatorAdvanced:
    """Advanced calculator tests."""
    
    @pytest.fixture(autouse=True)
    def setup(self):
        """Set up for each test."""
        self.calc = Calculator()
    
    def test_chain_operations(self):
        """Test chaining operations."""
        result = self.calc.add(5, 3)
        result = self.calc.multiply(result, 2)
        result = self.calc.subtract(result, 4)
        assert result == 12
    
    @pytest.mark.skip(reason="Not implemented yet")
    def test_power(self):
        """Test power operation."""
        pass
    
    @pytest.mark.xfail(reason="Known bug")
    def test_known_bug(self):
        """Test known bug."""
        assert self.calc.divide(1, 3) == 0.33
```

### Running Tests

```bash
# unittest
python -m unittest test_calculator.py
python -m unittest discover

# pytest
pytest
pytest test_calculator_pytest.py
pytest -v  # verbose
pytest -k "test_add"  # run tests matching pattern
pytest -m "slow"  # run tests with marker
pytest --maxfail=1  # stop after first failure
pytest -x  # stop on first failure
```

---

## Integration Testing

### Testing Database Operations

```python
# test_user_service.py

import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.models import Base, User
from app.services import UserService

@pytest.fixture(scope='function')
def db_session():
    """Create test database session."""
    # Create in-memory SQLite database
    engine = create_engine('sqlite:///:memory:')
    Base.metadata.create_all(engine)
    
    Session = sessionmaker(bind=engine)
    session = Session()
    
    yield session
    
    session.close()
    Base.metadata.drop_all(engine)

@pytest.fixture
def user_service(db_session):
    """Create user service instance."""
    return UserService(db_session)

def test_create_user(user_service, db_session):
    """Test creating a user."""
    # Arrange
    username = "testuser"
    email = "test@example.com"
    
    # Act
    user = user_service.create_user(username, email)
    
    # Assert
    assert user.id is not None
    assert user.username == username
    assert user.email == email
    
    # Verify in database
    db_user = db_session.query(User).filter_by(username=username).first()
    assert db_user is not None
    assert db_user.email == email

def test_get_user_by_id(user_service, db_session):
    """Test retrieving user by ID."""
    # Create user
    user = user_service.create_user("testuser", "test@example.com")
    
    # Retrieve user
    found_user = user_service.get_user_by_id(user.id)
    
    assert found_user is not None
    assert found_user.id == user.id
    assert found_user.username == user.username

def test_update_user(user_service):
    """Test updating user."""
    # Create user
    user = user_service.create_user("testuser", "test@example.com")
    
    # Update user
    updated_user = user_service.update_user(
        user.id,
        username="newusername",
        email="new@example.com"
    )
    
    assert updated_user.username == "newusername"
    assert updated_user.email == "new@example.com"

def test_delete_user(user_service, db_session):
    """Test deleting user."""
    # Create user
    user = user_service.create_user("testuser", "test@example.com")
    user_id = user.id
    
    # Delete user
    user_service.delete_user(user_id)
    
    # Verify deletion
    deleted_user = db_session.query(User).filter_by(id=user_id).first()
    assert deleted_user is None

def test_get_all_users(user_service):
    """Test getting all users."""
    # Create multiple users
    user_service.create_user("user1", "user1@example.com")
    user_service.create_user("user2", "user2@example.com")
    user_service.create_user("user3", "user3@example.com")
    
    # Get all users
    users = user_service.get_all_users()
    
    assert len(users) == 3
    assert all(isinstance(u, User) for u in users)
```

---

## Testing Flask Applications

### Basic Flask Testing

```python
# app.py
from flask import Flask, jsonify, request

app = Flask(__name__)

users = {}
next_id = 1

@app.route('/users', methods=['GET'])
def get_users():
    return jsonify(list(users.values()))

@app.route('/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    user = users.get(user_id)
    if not user:
        return jsonify({'error': 'User not found'}), 404
    return jsonify(user)

@app.route('/users', methods=['POST'])
def create_user():
    global next_id
    data = request.get_json()
    
    if not data or 'username' not in data:
        return jsonify({'error': 'Username required'}), 400
    
    user = {
        'id': next_id,
        'username': data['username'],
        'email': data.get('email')
    }
    
    users[next_id] = user
    next_id += 1
    
    return jsonify(user), 201

@app.route('/users/<int:user_id>', methods=['PUT'])
def update_user(user_id):
    user = users.get(user_id)
    if not user:
        return jsonify({'error': 'User not found'}), 404
    
    data = request.get_json()
    user.update(data)
    
    return jsonify(user)

@app.route('/users/<int:user_id>', methods=['DELETE'])
def delete_user(user_id):
    if user_id not in users:
        return jsonify({'error': 'User not found'}), 404
    
    del users[user_id]
    return '', 204
```

```python
# test_app.py

import pytest
from app import app

@pytest.fixture
def client():
    """Create test client."""
    app.config['TESTING'] = True
    with app.test_client() as client:
        yield client

@pytest.fixture
def sample_user():
    """Sample user data."""
    return {
        'username': 'testuser',
        'email': 'test@example.com'
    }

def test_get_users_empty(client):
    """Test getting users when empty."""
    response = client.get('/users')
    
    assert response.status_code == 200
    assert response.json == []

def test_create_user(client, sample_user):
    """Test creating a user."""
    response = client.post('/users', json=sample_user)
    
    assert response.status_code == 201
    assert response.json['username'] == sample_user['username']
    assert response.json['email'] == sample_user['email']
    assert 'id' in response.json

def test_create_user_without_username(client):
    """Test creating user without username."""
    response = client.post('/users', json={'email': 'test@example.com'})
    
    assert response.status_code == 400
    assert 'error' in response.json

def test_get_user(client, sample_user):
    """Test getting a specific user."""
    # Create user first
    create_response = client.post('/users', json=sample_user)
    user_id = create_response.json['id']
    
    # Get user
    response = client.get(f'/users/{user_id}')
    
    assert response.status_code == 200
    assert response.json['id'] == user_id
    assert response.json['username'] == sample_user['username']

def test_get_nonexistent_user(client):
    """Test getting nonexistent user."""
    response = client.get('/users/999')
    
    assert response.status_code == 404
    assert 'error' in response.json

def test_update_user(client, sample_user):
    """Test updating user."""
    # Create user
    create_response = client.post('/users', json=sample_user)
    user_id = create_response.json['id']
    
    # Update user
    update_data = {'username': 'updateduser'}
    response = client.put(f'/users/{user_id}', json=update_data)
    
    assert response.status_code == 200
    assert response.json['username'] == 'updateduser'

def test_delete_user(client, sample_user):
    """Test deleting user."""
    # Create user
    create_response = client.post('/users', json=sample_user)
    user_id = create_response.json['id']
    
    # Delete user
    response = client.delete(f'/users/{user_id}')
    
    assert response.status_code == 204
    
    # Verify deletion
    get_response = client.get(f'/users/{user_id}')
    assert get_response.status_code == 404

def test_user_workflow(client):
    """Test complete user workflow."""
    # Create user
    user_data = {'username': 'workflow', 'email': 'workflow@example.com'}
    create_response = client.post('/users', json=user_data)
    assert create_response.status_code == 201
    user_id = create_response.json['id']
    
    # Get user
    get_response = client.get(f'/users/{user_id}')
    assert get_response.status_code == 200
    
    # Update user
    update_response = client.put(
        f'/users/{user_id}',
        json={'username': 'updated'}
    )
    assert update_response.status_code == 200
    
    # Delete user
    delete_response = client.delete(f'/users/{user_id}')
    assert delete_response.status_code == 204
```

### Testing with Database

```python
# conftest.py

import pytest
from app import create_app, db
from app.models import User

@pytest.fixture(scope='session')
def app():
    """Create application for testing."""
    app = create_app('testing')
    
    with app.app_context():
        db.create_all()
        yield app
        db.drop_all()

@pytest.fixture
def client(app):
    """Create test client."""
    return app.test_client()

@pytest.fixture
def runner(app):
    """Create CLI runner."""
    return app.test_cli_runner()

@pytest.fixture(autouse=True)
def db_session(app):
    """Create database session for each test."""
    with app.app_context():
        # Clean database before each test
        db.session.remove()
        db.drop_all()
        db.create_all()
        
        yield db
        
        # Clean up after test
        db.session.remove()

@pytest.fixture
def sample_user(db_session):
    """Create sample user in database."""
    user = User(username='testuser', email='test@example.com')
    user.set_password('password123')
    db_session.session.add(user)
    db_session.session.commit()
    return user
```

---

## Testing FastAPI Applications

### Basic FastAPI Testing

```python
# main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Optional

app = FastAPI()

class User(BaseModel):
    id: Optional[int] = None
    username: str
    email: str

users_db = {}
next_id = 1

@app.get("/users", response_model=List[User])
async def get_users():
    return list(users_db.values())

@app.get("/users/{user_id}", response_model=User)
async def get_user(user_id: int):
    user = users_db.get(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

@app.post("/users", response_model=User, status_code=201)
async def create_user(user: User):
    global next_id
    user.id = next_id
    users_db[next_id] = user
    next_id += 1
    return user

@app.put("/users/{user_id}", response_model=User)
async def update_user(user_id: int, user: User):
    if user_id not in users_db:
        raise HTTPException(status_code=404, detail="User not found")
    user.id = user_id
    users_db[user_id] = user
    return user

@app.delete("/users/{user_id}", status_code=204)
async def delete_user(user_id: int):
    if user_id not in users_db:
        raise HTTPException(status_code=404, detail="User not found")
    del users_db[user_id]
```

```python
# test_main.py

import pytest
from fastapi.testclient import TestClient
from main import app, users_db, next_id

@pytest.fixture
def client():
    """Create test client."""
    return TestClient(app)

@pytest.fixture(autouse=True)
def reset_db():
    """Reset database before each test."""
    global next_id
    users_db.clear()
    next_id = 1

@pytest.fixture
def sample_user():
    """Sample user data."""
    return {
        "username": "testuser",
        "email": "test@example.com"
    }

def test_get_users_empty(client):
    """Test getting empty user list."""
    response = client.get("/users")
    
    assert response.status_code == 200
    assert response.json() == []

def test_create_user(client, sample_user):
    """Test creating a user."""
    response = client.post("/users", json=sample_user)
    
    assert response.status_code == 201
    data = response.json()
    assert data["username"] == sample_user["username"]
    assert data["email"] == sample_user["email"]
    assert "id" in data

def test_create_user_validation(client):
    """Test user validation."""
    # Missing email
    response = client.post("/users", json={"username": "test"})
    assert response.status_code == 422
    
    # Invalid email format
    response = client.post("/users", json={
        "username": "test",
        "email": "invalid-email"
    })
    assert response.status_code == 422

def test_get_user(client, sample_user):
    """Test getting a user."""
    # Create user
    create_response = client.post("/users", json=sample_user)
    user_id = create_response.json()["id"]
    
    # Get user
    response = client.get(f"/users/{user_id}")
    
    assert response.status_code == 200
    assert response.json()["id"] == user_id

def test_get_nonexistent_user(client):
    """Test getting nonexistent user."""
    response = client.get("/users/999")
    
    assert response.status_code == 404
    assert response.json()["detail"] == "User not found"

def test_update_user(client, sample_user):
    """Test updating user."""
    # Create user
    create_response = client.post("/users", json=sample_user)
    user_id = create_response.json()["id"]
    
    # Update user
    update_data = {
        "username": "updateduser",
        "email": "updated@example.com"
    }
    response = client.put(f"/users/{user_id}", json=update_data)
    
    assert response.status_code == 200
    assert response.json()["username"] == "updateduser"

def test_delete_user(client, sample_user):
    """Test deleting user."""
    # Create user
    create_response = client.post("/users", json=sample_user)
    user_id = create_response.json()["id"]
    
    # Delete user
    response = client.delete(f"/users/{user_id}")
    
    assert response.status_code == 204
    
    # Verify deletion
    get_response = client.get(f"/users/{user_id}")
    assert get_response.status_code == 404

@pytest.mark.asyncio
async def test_async_operations():
    """Test async operations."""
    from main import get_users
    
    users = await get_users()
    assert isinstance(users, list)
```

### Testing with Database

```python
# test_database.py

import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.main import app, get_db
from app.database import Base
from app.models import User

# Test database
SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"

engine = create_engine(
    SQLALCHEMY_DATABASE_URL,
    connect_args={"check_same_thread": False}
)
TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

@pytest.fixture(scope="function")
def db():
    """Create test database."""
    Base.metadata.create_all(bind=engine)
    db = TestingSessionLocal()
    try:
        yield db
    finally:
        db.close()
        Base.metadata.drop_all(bind=engine)

@pytest.fixture
def client(db):
    """Create test client with database dependency override."""
    def override_get_db():
        try:
            yield db
        finally:
            pass
    
    app.dependency_overrides[get_db] = override_get_db
    
    with TestClient(app) as client:
        yield client
    
    app.dependency_overrides.clear()

def test_create_user_with_db(client):
    """Test creating user with database."""
    response = client.post(
        "/users",
        json={"username": "test", "email": "test@example.com"}
    )
    
    assert response.status_code == 201
    assert response.json()["username"] == "test"

def test_get_users_with_db(client, db):
    """Test getting users from database."""
    # Add users to database
    user1 = User(username="user1", email="user1@example.com")
    user2 = User(username="user2", email="user2@example.com")
    db.add(user1)
    db.add(user2)
    db.commit()
    
    # Get users
    response = client.get("/users")
    
    assert response.status_code == 200
    assert len(response.json()) == 2
```

---

## Testing Django Applications

### Django Test Case

```python
# tests/test_models.py

from django.test import TestCase
from django.contrib.auth.models import User
from app.models import Post

class PostModelTest(TestCase):
    """Test Post model."""
    
    @classmethod
    def setUpTestData(cls):
        """Set up data for the whole TestCase."""
        cls.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        cls.post = Post.objects.create(
            title='Test Post',
            content='Test content',
            author=cls.user
        )
    
    def test_post_creation(self):
        """Test post creation."""
        self.assertEqual(self.post.title, 'Test Post')
        self.assertEqual(self.post.content, 'Test content')
        self.assertEqual(self.post.author, self.user)
    
    def test_post_str(self):
        """Test post string representation."""
        self.assertEqual(str(self.post), 'Test Post')
    
    def test_post_absolute_url(self):
        """Test get_absolute_url method."""
        self.assertEqual(
            self.post.get_absolute_url(),
            f'/posts/{self.post.id}/'
        )
    
    def test_post_published(self):
        """Test published property."""
        self.assertTrue(self.post.published)
```

```python
# tests/test_views.py

from django.test import TestCase, Client
from django.contrib.auth.models import User
from django.urls import reverse
from app.models import Post

class PostViewsTest(TestCase):
    """Test Post views."""
    
    def setUp(self):
        """Set up for each test."""
        self.client = Client()
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        self.post = Post.objects.create(
            title='Test Post',
            content='Test content',
            author=self.user
        )
    
    def test_post_list_view(self):
        """Test post list view."""
        response = self.client.get(reverse('post_list'))
        
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, 'Test Post')
        self.assertTemplateUsed(response, 'posts/list.html')
    
    def test_post_detail_view(self):
        """Test post detail view."""
        response = self.client.get(
            reverse('post_detail', args=[self.post.id])
        )
        
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, 'Test Post')
        self.assertContains(response, 'Test content')
    
    def test_post_create_view_anonymous(self):
        """Test post create view for anonymous user."""
        response = self.client.get(reverse('post_create'))
        
        # Should redirect to login
        self.assertEqual(response.status_code, 302)
    
    def test_post_create_view_authenticated(self):
        """Test post create view for authenticated user."""
        self.client.login(username='testuser', password='testpass123')
        response = self.client.get(reverse('post_create'))
        
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed(response, 'posts/create.html')
    
    def test_post_create_post(self):
        """Test creating post via POST."""
        self.client.login(username='testuser', password='testpass123')
        
        response = self.client.post(reverse('post_create'), {
            'title': 'New Post',
            'content': 'New content'
        })
        
        # Should redirect to detail page
        self.assertEqual(response.status_code, 302)
        
        # Verify post created
        self.assertTrue(
            Post.objects.filter(title='New Post').exists()
        )
    
    def test_post_update_view(self):
        """Test post update view."""
        self.client.login(username='testuser', password='testpass123')
        
        response = self.client.post(
            reverse('post_update', args=[self.post.id]),
            {
                'title': 'Updated Title',
                'content': 'Updated content'
            }
        )
        
        # Verify update
        self.post.refresh_from_db()
        self.assertEqual(self.post.title, 'Updated Title')
    
    def test_post_delete_view(self):
        """Test post delete view."""
        self.client.login(username='testuser', password='testpass123')
        
        response = self.client.post(
            reverse('post_delete', args=[self.post.id])
        )
        
        # Verify deletion
        self.assertFalse(
            Post.objects.filter(id=self.post.id).exists()
        )
```

### Django REST Framework Testing

```python
# tests/test_api.py

from rest_framework.test import APITestCase, APIClient
from rest_framework import status
from django.contrib.auth.models import User
from app.models import Post

class PostAPITest(APITestCase):
    """Test Post API."""
    
    def setUp(self):
        """Set up for each test."""
        self.client = APIClient()
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        self.post = Post.objects.create(
            title='Test Post',
            content='Test content',
            author=self.user
        )
    
    def test_get_posts(self):
        """Test getting all posts."""
        response = self.client.get('/api/posts/')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data), 1)
    
    def test_get_post_detail(self):
        """Test getting post detail."""
        response = self.client.get(f'/api/posts/{self.post.id}/')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data['title'], 'Test Post')
    
    def test_create_post_authenticated(self):
        """Test creating post when authenticated."""
        self.client.force_authenticate(user=self.user)
        
        data = {
            'title': 'New Post',
            'content': 'New content'
        }
        response = self.client.post('/api/posts/', data)
        
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(Post.objects.count(), 2)
    
    def test_create_post_unauthenticated(self):
        """Test creating post when not authenticated."""
        data = {
            'title': 'New Post',
            'content': 'New content'
        }
        response = self.client.post('/api/posts/', data)
        
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)
    
    def test_update_post(self):
        """Test updating post."""
        self.client.force_authenticate(user=self.user)
        
        data = {
            'title': 'Updated Title',
            'content': 'Updated content'
        }
        response = self.client.put(f'/api/posts/{self.post.id}/', data)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.post.refresh_from_db()
        self.assertEqual(self.post.title, 'Updated Title')
    
    def test_delete_post(self):
        """Test deleting post."""
        self.client.force_authenticate(user=self.user)
        
        response = self.client.delete(f'/api/posts/{self.post.id}/')
        
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)
        self.assertEqual(Post.objects.count(), 0)
    
    def test_permissions(self):
        """Test post permissions."""
        # Create another user
        other_user = User.objects.create_user(
            username='otheruser',
            password='testpass123'
        )
        
        # Try to update someone else's post
        self.client.force_authenticate(user=other_user)
        
        data = {'title': 'Hacked', 'content': 'Hacked'}
        response = self.client.put(f'/api/posts/{self.post.id}/', data)
        
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)
```

---

## Database Testing

### Testing Transactions

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.models import Base, User, Order

@pytest.fixture(scope='function')
def db_session():
    """Create test database with transaction rollback."""
    engine = create_engine('sqlite:///:memory:')
    Base.metadata.create_all(engine)
    
    Session = sessionmaker(bind=engine)
    session = Session()
    
    # Start transaction
    connection = engine.connect()
    transaction = connection.begin()
    
    yield session
    
    # Rollback transaction
    session.close()
    transaction.rollback()
    connection.close()

def test_create_user_with_orders(db_session):
    """Test creating user with related orders."""
    # Create user
    user = User(username='testuser', email='test@example.com')
    db_session.add(user)
    db_session.flush()  # Get user.id without committing
    
    # Create orders
    order1 = Order(user_id=user.id, total=100.00)
    order2 = Order(user_id=user.id, total=200.00)
    db_session.add_all([order1, order2])
    db_session.commit()
    
    # Verify
    assert len(user.orders) == 2
    assert sum(o.total for o in user.orders) == 300.00

def test_transaction_rollback(db_session):
    """Test transaction rollback on error."""
    user = User(username='testuser', email='test@example.com')
    db_session.add(user)
    db_session.commit()
    
    try:
        # This should fail
        duplicate = User(username='testuser', email='test@example.com')
        db_session.add(duplicate)
        db_session.commit()
    except:
        db_session.rollback()
    
    # Verify only one user exists
    assert db_session.query(User).count() == 1
```

---

## API Testing

### Testing REST API Endpoints

```python
import pytest
import requests
from unittest.mock import Mock, patch

@pytest.fixture
def api_client():
    """Create API client."""
    return requests.Session()

def test_api_get_users(api_client):
    """Test getting users from API."""
    response = api_client.get('http://localhost:5000/api/users')
    
    assert response.status_code == 200
    assert isinstance(response.json(), list)

def test_api_create_user(api_client):
    """Test creating user via API."""
    user_data = {
        'username': 'testuser',
        'email': 'test@example.com'
    }
    
    response = api_client.post(
        'http://localhost:5000/api/users',
        json=user_data
    )
    
    assert response.status_code == 201
    assert response.json()['username'] == user_data['username']

def test_api_authentication(api_client):
    """Test API authentication."""
    # Without authentication
    response = api_client.get('http://localhost:5000/api/protected')
    assert response.status_code == 401
    
    # With authentication
    api_client.headers.update({'Authorization': 'Bearer valid_token'})
    response = api_client.get('http://localhost:5000/api/protected')
    assert response.status_code == 200

@patch('requests.Session.get')
def test_api_with_mock(mock_get):
    """Test API with mocked response."""
    # Mock response
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = [
        {'id': 1, 'username': 'user1'},
        {'id': 2, 'username': 'user2'}
    ]
    mock_get.return_value = mock_response
    
    # Test
    session = requests.Session()
    response = session.get('http://localhost:5000/api/users')
    
    assert response.status_code == 200
    assert len(response.json()) == 2
```

---

## End-to-End Testing with Selenium

### Basic Selenium Testing

```python
# test_e2e.py

import pytest
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.common.keys import Keys

@pytest.fixture(scope='module')
def browser():
    """Create browser instance."""
    # Use Chrome
    options = webdriver.ChromeOptions()
    options.add_argument('--headless')  # Run headless
    options.add_argument('--no-sandbox')
    options.add_argument('--disable-dev-shm-usage')
    
    driver = webdriver.Chrome(options=options)
    driver.implicitly_wait(10)
    
    yield driver
    
    driver.quit()

def test_homepage(browser):
    """Test homepage loads correctly."""
    browser.get('http://localhost:5000')
    
    assert 'Welcome' in browser.title
    
    # Find element
    heading = browser.find_element(By.TAG_NAME, 'h1')
    assert heading.text == 'Welcome to My App'

def test_user_registration(browser):
    """Test user registration flow."""
    browser.get('http://localhost:5000/register')
    
    # Fill form
    username_input = browser.find_element(By.ID, 'username')
    username_input.send_keys('testuser')
    
    email_input = browser.find_element(By.ID, 'email')
    email_input.send_keys('test@example.com')
    
    password_input = browser.find_element(By.ID, 'password')
    password_input.send_keys('SecurePassword123!')
    
    # Submit form
    submit_button = browser.find_element(By.ID, 'submit')
    submit_button.click()
    
    # Wait for redirect
    WebDriverWait(browser, 10).until(
        EC.url_contains('/dashboard')
    )
    
    # Verify success
    success_message = browser.find_element(By.CLASS_NAME, 'success')
    assert 'Registration successful' in success_message.text

def test_login(browser):
    """Test user login."""
    browser.get('http://localhost:5000/login')
    
    # Fill login form
    username_input = browser.find_element(By.ID, 'username')
    username_input.send_keys('testuser')
    
    password_input = browser.find_element(By.ID, 'password')
    password_input.send_keys('SecurePassword123!')
    password_input.send_keys(Keys.RETURN)
    
    # Wait for dashboard
    WebDriverWait(browser, 10).until(
        EC.presence_of_element_located((By.ID, 'dashboard'))
    )
    
    assert 'Dashboard' in browser.title

def test_create_post(browser):
    """Test creating a post."""
    # Assume already logged in
    browser.get('http://localhost:5000/posts/create')
    
    # Fill form
    title_input = browser.find_element(By.ID, 'title')
    title_input.send_keys('Test Post')
    
    content_textarea = browser.find_element(By.ID, 'content')
    content_textarea.send_keys('This is test content.')
    
    # Submit
    browser.find_element(By.ID, 'submit').click()
    
    # Verify post created
    WebDriverWait(browser, 10).until(
        EC.presence_of_element_located((By.CLASS_NAME, 'post'))
    )
    
    post_title = browser.find_element(By.CLASS_NAME, 'post-title')
    assert post_title.text == 'Test Post'

def test_search_functionality(browser):
    """Test search functionality."""
    browser.get('http://localhost:5000')
    
    # Find search box
    search_box = browser.find_element(By.ID, 'search')
    search_box.send_keys('test query')
    search_box.send_keys(Keys.RETURN)
    
    # Wait for results
    WebDriverWait(browser, 10).until(
        EC.presence_of_element_located((By.CLASS_NAME, 'search-results'))
    )
    
    results = browser.find_elements(By.CLASS_NAME, 'result')
    assert len(results) > 0

def test_javascript_interaction(browser):
    """Test JavaScript interactions."""
    browser.get('http://localhost:5000/interactive')
    
    # Click button
    button = browser.find_element(By.ID, 'load-more')
    button.click()
    
    # Wait for AJAX request to complete
    WebDriverWait(browser, 10).until(
        EC.text_to_be_present_in_element(
            (By.ID, 'items-count'),
            '20 items'
        )
    )
    
    # Verify items loaded
    items = browser.find_elements(By.CLASS_NAME, 'item')
    assert len(items) == 20
```

### Page Object Pattern

```python
# pages/login_page.py

from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

class LoginPage:
    """Login page object."""
    
    def __init__(self, driver):
        self.driver = driver
        self.wait = WebDriverWait(driver, 10)
    
    # Locators
    USERNAME_INPUT = (By.ID, 'username')
    PASSWORD_INPUT = (By.ID, 'password')
    LOGIN_BUTTON = (By.ID, 'login')
    ERROR_MESSAGE = (By.CLASS_NAME, 'error')
    
    def open(self):
        """Open login page."""
        self.driver.get('http://localhost:5000/login')
    
    def login(self, username, password):
        """Perform login."""
        username_input = self.driver.find_element(*self.USERNAME_INPUT)
        username_input.send_keys(username)
        
        password_input = self.driver.find_element(*self.PASSWORD_INPUT)
        password_input.send_keys(password)
        
        login_button = self.driver.find_element(*self.LOGIN_BUTTON)
        login_button.click()
    
    def get_error_message(self):
        """Get error message."""
        error = self.wait.until(
            EC.presence_of_element_located(self.ERROR_MESSAGE)
        )
        return error.text
    
    def is_logged_in(self):
        """Check if logged in."""
        self.wait.until(
            EC.url_contains('/dashboard')
        )
        return '/dashboard' in self.driver.current_url

# test_login_page.py

from pages.login_page import LoginPage

def test_successful_login(browser):
    """Test successful login."""
    login_page = LoginPage(browser)
    login_page.open()
    login_page.login('testuser', 'password123')
    
    assert login_page.is_logged_in()

def test_failed_login(browser):
    """Test failed login."""
    login_page = LoginPage(browser)
    login_page.open()
    login_page.login('baduser', 'badpass')
    
    error = login_page.get_error_message()
    assert 'Invalid credentials' in error
```

---

## Mocking and Fixtures

### Using unittest.mock

```python
from unittest.mock import Mock, patch, MagicMock
import pytest

# Mock objects
def test_mock_basic():
    """Test basic mocking."""
    mock = Mock()
    mock.method.return_value = 42
    
    assert mock.method() == 42
    mock.method.assert_called_once()

# Mock function
@patch('app.services.send_email')
def test_user_registration_sends_email(mock_send_email):
    """Test registration sends email."""
    from app.services import register_user
    
    user = register_user('test@example.com', 'password')
    
    # Verify email was sent
    mock_send_email.assert_called_once()
    mock_send_email.assert_called_with(
        to='test@example.com',
        subject='Welcome'
    )

# Mock external API
@patch('requests.get')
def test_fetch_weather(mock_get):
    """Test fetching weather data."""
    # Setup mock response
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {
        'temperature': 25,
        'condition': 'sunny'
    }
    mock_get.return_value = mock_response
    
    # Test function
    from app.weather import get_weather
    weather = get_weather('New York')
    
    assert weather['temperature'] == 25
    assert weather['condition'] == 'sunny'

# Mock class
@patch('app.services.UserService')
def test_with_mock_class(MockUserService):
    """Test with mocked class."""
    # Configure mock
    mock_service = MockUserService.return_value
    mock_service.get_user.return_value = {
        'id': 1,
        'username': 'testuser'
    }
    
    # Test
    service = MockUserService()
    user = service.get_user(1)
    
    assert user['username'] == 'testuser'
    mock_service.get_user.assert_called_once_with(1)

# Mock database
@patch('app.db.session')
def test_database_operation(mock_session):
    """Test database operation."""
    from app.models import User
    from app.services import create_user
    
    # Mock commit
    mock_session.commit = Mock()
    
    # Test
    user = create_user('testuser', 'test@example.com')
    
    # Verify session.add was called
    mock_session.add.assert_called_once()
    mock_session.commit.assert_called_once()
```

### Pytest Fixtures

```python
# conftest.py

import pytest
from app import create_app, db
from app.models import User

@pytest.fixture(scope='session')
def app():
    """Create application."""
    app = create_app('testing')
    return app

@pytest.fixture(scope='function')
def client(app):
    """Create test client."""
    return app.test_client()

@pytest.fixture(scope='function')
def db_session(app):
    """Create database session."""
    with app.app_context():
        db.create_all()
        yield db
        db.session.remove()
        db.drop_all()

@pytest.fixture
def user(db_session):
    """Create test user."""
    user = User(username='testuser', email='test@example.com')
    user.set_password('password123')
    db_session.session.add(user)
    db_session.session.commit()
    return user

@pytest.fixture
def authenticated_client(client, user):
    """Create authenticated client."""
    client.post('/login', data={
        'username': user.username,
        'password': 'password123'
    })
    return client

@pytest.fixture
def mock_email_service():
    """Mock email service."""
    with patch('app.services.send_email') as mock:
        yield mock

# test_example.py

def test_with_fixtures(client, user, mock_email_service):
    """Test using multiple fixtures."""
    response = client.post('/reset-password', data={
        'email': user.email
    })
    
    assert response.status_code == 200
    mock_email_service.assert_called_once()
```

---

## Test Coverage

### Using pytest-cov

```bash
# Install pytest-cov
pip install pytest-cov

# Run tests with coverage
pytest --cov=app tests/

# Generate HTML report
pytest --cov=app --cov-report=html tests/

# Generate XML report (for CI)
pytest --cov=app --cov-report=xml tests/

# Show missing lines
pytest --cov=app --cov-report=term-missing tests/

# Set minimum coverage threshold
pytest --cov=app --cov-fail-under=80 tests/
```

```python
# pytest.ini

[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = 
    --cov=app
    --cov-report=html
    --cov-report=term-missing
    --cov-fail-under=80
    -v
```

### Coverage Configuration

```ini
# .coveragerc

[run]
source = app
omit = 
    */tests/*
    */venv/*
    */migrations/*
    */__init__.py

[report]
exclude_lines =
    pragma: no cover
    def __repr__
    raise AssertionError
    raise NotImplementedError
    if __name__ == .__main__.:
    if TYPE_CHECKING:
    @abstractmethod

[html]
directory = htmlcov
```

---

## Performance Testing

### Using Locust

```bash
# Install Locust
pip install locust
```

```python
# locustfile.py

from locust import HttpUser, task, between

class WebsiteUser(HttpUser):
    """Simulate website user."""
    
    wait_time = between(1, 3)  # Wait 1-3 seconds between tasks
    
    def on_start(self):
        """Login when user starts."""
        self.client.post('/login', {
            'username': 'testuser',
            'password': 'password123'
        })
    
    @task(3)  # Weight of 3
    def view_homepage(self):
        """View homepage."""
        self.client.get('/')
    
    @task(2)
    def view_posts(self):
        """View posts."""
        self.client.get('/posts')
    
    @task(1)
    def create_post(self):
        """Create post."""
        self.client.post('/posts', {
            'title': 'Test Post',
            'content': 'Test content'
        })
    
    @task
    def view_profile(self):
        """View profile."""
        self.client.get('/profile')

# Run with:
# locust -f locustfile.py --host=http://localhost:5000
```

### Load Testing with pytest-benchmark

```bash
pip install pytest-benchmark
```

```python
import pytest

def complex_calculation(n):
    """Some complex calculation."""
    result = 0
    for i in range(n):
        result += i ** 2
    return result

def test_performance(benchmark):
    """Test performance of function."""
    result = benchmark(complex_calculation, 1000)
    assert result > 0

@pytest.mark.parametrize('n', [100, 1000, 10000])
def test_scalability(benchmark, n):
    """Test scalability with different inputs."""
    benchmark(complex_calculation, n)
```

---

## Security Testing

### Testing Authentication

```python
def test_login_required(client):
    """Test endpoint requires login."""
    response = client.get('/dashboard')
    assert response.status_code == 302  # Redirect to login

def test_sql_injection(client):
    """Test SQL injection prevention."""
    response = client.post('/login', data={
        'username': "admin' OR '1'='1",
        'password': 'anything'
    })
    assert response.status_code == 401

def test_xss_prevention(client):
    """Test XSS prevention."""
    response = client.post('/posts', data={
        'title': '<script>alert("XSS")</script>',
        'content': 'Content'
    })
    
    # Get the post
    posts_response = client.get('/posts')
    assert b'<script>' not in posts_response.data
    assert b'&lt;script&gt;' in posts_response.data

def test_csrf_protection(client):
    """Test CSRF protection."""
    # Request without CSRF token should fail
    response = client.post('/transfer', data={
        'to': 'attacker',
        'amount': 1000
    })
    assert response.status_code == 403

def test_password_hashing(db_session):
    """Test passwords are hashed."""
    from app.models import User
    
    user = User(username='test', email='test@example.com')
    user.set_password('password123')
    
    # Password should be hashed
    assert user.password_hash != 'password123'
    
    # Verify password works
    assert user.check_password('password123')
    assert not user.check_password('wrongpassword')
```

---

## Continuous Integration

### GitHub Actions

```yaml
# .github/workflows/tests.yml

name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        python-version: [3.9, 3.10, 3.11]
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v4
      with:
        python-version: ${{ matrix.python-version }}
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install -r requirements-test.txt
    
    - name: Run tests
      run: |
        pytest --cov=app --cov-report=xml
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
```

---

## Best Practices

### Testing Best Practices

```python
"""
✅ Testing Best Practices:

1. Test Organization
   ✅ One test file per module
   ✅ Group related tests in classes
   ✅ Clear, descriptive test names
   ✅ Follow AAA pattern (Arrange, Act, Assert)

2. Test Coverage
   ✅ Aim for 80%+ coverage
   ✅ Test edge cases
   ✅ Test error conditions
   ✅ Test happy path and sad path

3. Test Independence
   ✅ Tests should not depend on each other
   ✅ Each test should clean up after itself
   ✅ Use fixtures for setup/teardown
   ✅ Reset state between tests

4. Test Speed
   ✅ Keep unit tests fast
   ✅ Use mocks for external dependencies
   ✅ Separate slow tests (integration, E2E)
   ✅ Run fast tests frequently

5. Test Maintenance
   ✅ Keep tests simple
   ✅ Don't test implementation details
   ✅ Refactor tests when refactoring code
   ✅ Remove obsolete tests

6. CI/CD
   ✅ Run tests on every commit
   ✅ Fail build if tests fail
   ✅ Monitor test coverage
   ✅ Run different test types separately

7. Documentation
   ✅ Document test purpose
   ✅ Use clear assertions
   ✅ Comment complex test logic
   ✅ Provide setup instructions
"""
```

---

## Summary

This comprehensive guide covered:

1. **Introduction**: Testing pyramid and benefits
2. **Fundamentals**: AAA pattern, naming conventions
3. **Unit Testing**: unittest and pytest
4. **Integration Testing**: Database and service testing
5. **Flask Testing**: Test client, fixtures, database
6. **FastAPI Testing**: TestClient, async tests
7. **Django Testing**: TestCase, API testing
8. **Database Testing**: Transactions, fixtures
9. **API Testing**: REST endpoints, authentication
10. **E2E Testing**: Selenium, Page Object Pattern
11. **Mocking**: unittest.mock, pytest fixtures
12. **Coverage**: pytest-cov, configuration
13. **Performance**: Locust, benchmarking
14. **Security**: Authentication, XSS, CSRF
15. **CI/CD**: GitHub Actions
16. **Best Practices**: Organization, speed, maintenance

Key Takeaways:
- Write tests at multiple levels (unit, integration, E2E)
- Follow the testing pyramid
- Use appropriate testing tools
- Mock external dependencies
- Aim for high test coverage
- Keep tests fast and independent
- Integrate testing into CI/CD
- Test security vulnerabilities
- Document your tests
- Refactor tests with code

Testing gives confidence to deploy and refactor!