# Python Testing: A Detailed Guide

## Table of Contents
1. [Introduction to Testing](#introduction-to-testing)
2. [Unit Testing with unittest](#unit-testing-with-unittest)
3. [Testing with pytest](#testing-with-pytest)
4. [Test-Driven Development (TDD)](#test-driven-development-tdd)
5. [Test Fixtures and Setup](#test-fixtures-and-setup)
6. [Mocking and Patching](#mocking-and-patching)
7. [Test Coverage](#test-coverage)
8. [Parameterized Tests](#parameterized-tests)
9. [Testing Exceptions](#testing-exceptions)
10. [Best Practices](#best-practices)
11. [Integration and End-to-End Testing](#integration-and-end-to-end-testing)

---

## Introduction to Testing

Testing ensures your code works correctly and continues to work as you make changes.

### Types of Tests

**Unit Tests**: Test individual functions or methods in isolation
**Integration Tests**: Test how multiple components work together
**End-to-End Tests**: Test the entire application workflow
**Regression Tests**: Ensure bugs don't reappear after fixes

### Why Test?

- **Catch bugs early** before they reach production
- **Document behavior** through test examples
- **Enable refactoring** with confidence
- **Improve code design** by making code testable
- **Save time** in the long run

---

## Unit Testing with unittest

Python's built-in testing framework.

### Basic unittest Structure

```python
import unittest

def add(a, b):
    return a + b

def subtract(a, b):
    return a - b

class TestMathOperations(unittest.TestCase):
    
    def test_add_positive_numbers(self):
        result = add(5, 3)
        self.assertEqual(result, 8)
    
    def test_add_negative_numbers(self):
        result = add(-5, -3)
        self.assertEqual(result, -8)
    
    def test_subtract(self):
        result = subtract(10, 3)
        self.assertEqual(result, 7)

# Run tests
if __name__ == '__main__':
    unittest.main()
```

### Common Assertions

```python
import unittest

class TestAssertions(unittest.TestCase):
    
    def test_equality(self):
        self.assertEqual(5 + 3, 8)
        self.assertNotEqual(5 + 3, 9)
    
    def test_truth(self):
        self.assertTrue(5 > 3)
        self.assertFalse(5 < 3)
    
    def test_membership(self):
        self.assertIn('a', 'apple')
        self.assertNotIn('z', 'apple')
    
    def test_none(self):
        value = None
        self.assertIsNone(value)
        self.assertIsNotNone("text")
    
    def test_instance(self):
        self.assertIsInstance(5, int)
        self.assertNotIsInstance("5", int)
    
    def test_comparison(self):
        self.assertGreater(10, 5)
        self.assertLess(5, 10)
        self.assertGreaterEqual(10, 10)
        self.assertLessEqual(5, 5)
    
    def test_almost_equal(self):
        # For floating point comparisons
        self.assertAlmostEqual(0.1 + 0.2, 0.3)
    
    def test_lists(self):
        self.assertListEqual([1, 2, 3], [1, 2, 3])
        self.assertCountEqual([1, 2, 3], [3, 2, 1])  # Order doesn't matter

if __name__ == '__main__':
    unittest.main()
```

### setUp and tearDown

```python
import unittest

class Calculator:
    def __init__(self):
        self.result = 0
    
    def add(self, x):
        self.result += x
        return self.result
    
    def clear(self):
        self.result = 0

class TestCalculator(unittest.TestCase):
    
    def setUp(self):
        """Called before each test method"""
        self.calc = Calculator()
        print("setUp: Creating calculator")
    
    def tearDown(self):
        """Called after each test method"""
        print("tearDown: Cleaning up")
        self.calc = None
    
    def test_add(self):
        self.assertEqual(self.calc.add(5), 5)
        self.assertEqual(self.calc.add(3), 8)
    
    def test_clear(self):
        self.calc.add(10)
        self.calc.clear()
        self.assertEqual(self.calc.result, 0)

if __name__ == '__main__':
    unittest.main()
```

### Class-level Setup

```python
import unittest

class TestWithClassSetup(unittest.TestCase):
    
    @classmethod
    def setUpClass(cls):
        """Called once before all tests in the class"""
        print("\nsetUpClass: Expensive setup")
        cls.shared_resource = "Shared data"
    
    @classmethod
    def tearDownClass(cls):
        """Called once after all tests in the class"""
        print("tearDownClass: Cleanup")
        cls.shared_resource = None
    
    def test_one(self):
        print(f"test_one: Using {self.shared_resource}")
        self.assertIsNotNone(self.shared_resource)
    
    def test_two(self):
        print(f"test_two: Using {self.shared_resource}")
        self.assertIsNotNone(self.shared_resource)

if __name__ == '__main__':
    unittest.main()
```

---

## Testing with pytest

pytest is a more modern and popular testing framework with cleaner syntax.

### Installing pytest

```bash
pip install pytest
```

### Basic pytest Structure

```python
# test_math.py

def add(a, b):
    return a + b

def subtract(a, b):
    return a - b

# Test functions (no class needed)
def test_add_positive():
    assert add(5, 3) == 8

def test_add_negative():
    assert add(-5, -3) == -8

def test_subtract():
    assert subtract(10, 3) == 7

# Run with: pytest test_math.py
```

### pytest Assertions

```python
# test_assertions.py

def test_equality():
    assert 5 + 3 == 8
    assert 5 + 3 != 9

def test_membership():
    assert 'a' in 'apple'
    assert 'z' not in 'apple'

def test_none():
    value = None
    assert value is None
    assert "text" is not None

def test_boolean():
    assert True
    assert not False

def test_comparisons():
    assert 10 > 5
    assert 5 < 10
    assert 10 >= 10
    assert 5 <= 5

def test_collections():
    assert [1, 2, 3] == [1, 2, 3]
    assert {'a': 1} == {'a': 1}
```

### pytest Fixtures

Fixtures provide a way to set up test dependencies.

```python
# test_fixtures.py
import pytest

class Database:
    def __init__(self):
        self.data = {}
    
    def save(self, key, value):
        self.data[key] = value
    
    def get(self, key):
        return self.data.get(key)

# Fixture
@pytest.fixture
def database():
    """Provides a fresh database for each test"""
    db = Database()
    yield db  # Test runs here
    # Cleanup (if needed)
    db.data.clear()

def test_save(database):
    database.save('key1', 'value1')
    assert database.get('key1') == 'value1'

def test_get_missing_key(database):
    assert database.get('nonexistent') is None

# Fixture with scope
@pytest.fixture(scope='module')
def expensive_resource():
    """Created once per module, shared across tests"""
    print("\nSetting up expensive resource")
    resource = "Shared data"
    yield resource
    print("\nTearing down expensive resource")

def test_with_expensive_resource(expensive_resource):
    assert expensive_resource == "Shared data"
```

### Fixture Scopes

```python
import pytest

@pytest.fixture(scope='function')  # Default: new instance per test
def function_scope():
    return "function"

@pytest.fixture(scope='class')  # Shared within test class
def class_scope():
    return "class"

@pytest.fixture(scope='module')  # Shared within module
def module_scope():
    return "module"

@pytest.fixture(scope='session')  # Shared across entire test session
def session_scope():
    return "session"
```

---

## Test-Driven Development (TDD)

TDD is a development approach where you write tests before writing code.

### TDD Cycle: Red-Green-Refactor

**Red**: Write a failing test
**Green**: Write minimal code to pass the test
**Refactor**: Improve the code while keeping tests passing

### TDD Example

```python
# Step 1: Write failing test (RED)
import pytest

def test_calculate_total_empty_cart():
    """Test for empty shopping cart"""
    cart = ShoppingCart()
    assert cart.calculate_total() == 0

# This fails because ShoppingCart doesn't exist yet

# Step 2: Write minimal code to pass (GREEN)
class ShoppingCart:
    def __init__(self):
        self.items = []
    
    def calculate_total(self):
        return 0

# Test passes!

# Step 3: Add more tests (RED again)
def test_calculate_total_single_item():
    cart = ShoppingCart()
    cart.add_item('Apple', 1.5)
    assert cart.calculate_total() == 1.5

# Step 4: Implement feature (GREEN)
class ShoppingCart:
    def __init__(self):
        self.items = []
    
    def add_item(self, name, price):
        self.items.append({'name': name, 'price': price})
    
    def calculate_total(self):
        return sum(item['price'] for item in self.items)

# Step 5: Add more tests
def test_calculate_total_multiple_items():
    cart = ShoppingCart()
    cart.add_item('Apple', 1.5)
    cart.add_item('Banana', 0.75)
    cart.add_item('Orange', 2.0)
    assert cart.calculate_total() == 4.25

# Step 6: Refactor if needed (keeping tests green)
class ShoppingCart:
    def __init__(self):
        self.items = []
    
    def add_item(self, name, price, quantity=1):
        self.items.append({
            'name': name,
            'price': price,
            'quantity': quantity
        })
    
    def calculate_total(self):
        return sum(
            item['price'] * item.get('quantity', 1) 
            for item in self.items
        )

def test_calculate_total_with_quantities():
    cart = ShoppingCart()
    cart.add_item('Apple', 1.5, quantity=3)
    assert cart.calculate_total() == 4.5
```

---

## Test Fixtures and Setup

### pytest conftest.py

Share fixtures across multiple test files.

```python
# conftest.py
import pytest

@pytest.fixture
def sample_data():
    """Available to all test files in this directory"""
    return {
        'users': ['Alice', 'Bob', 'Charlie'],
        'scores': [95, 87, 92]
    }

@pytest.fixture
def database():
    """Database fixture available to all tests"""
    db = Database()
    db.connect()
    yield db
    db.disconnect()
```

```python
# test_users.py
def test_user_count(sample_data):
    assert len(sample_data['users']) == 3

def test_first_user(sample_data):
    assert sample_data['users'][0] == 'Alice'
```

### Fixture Dependencies

```python
import pytest

@pytest.fixture
def database():
    db = Database()
    db.connect()
    yield db
    db.disconnect()

@pytest.fixture
def user(database):
    """Fixture that depends on database fixture"""
    user = User(database)
    user.create('test_user')
    yield user
    user.delete()

def test_user_operations(user):
    assert user.name == 'test_user'
    user.update_name('new_name')
    assert user.name == 'new_name'
```

---

## Mocking and Patching

Mocking replaces real objects with fake ones for testing.

### unittest.mock

```python
from unittest.mock import Mock, patch
import pytest

# Code to test
import requests

def get_user_data(user_id):
    response = requests.get(f'https://api.example.com/users/{user_id}')
    return response.json()

# Test with mock
def test_get_user_data():
    # Create a mock response
    mock_response = Mock()
    mock_response.json.return_value = {
        'id': 1,
        'name': 'Alice',
        'email': 'alice@example.com'
    }
    
    # Patch requests.get
    with patch('requests.get', return_value=mock_response):
        result = get_user_data(1)
        assert result['name'] == 'Alice'
        assert result['email'] == 'alice@example.com'
```

### Mock Return Values

```python
from unittest.mock import Mock

# Simple mock
mock_func = Mock(return_value=42)
assert mock_func() == 42

# Different returns for multiple calls
mock_func = Mock(side_effect=[1, 2, 3])
assert mock_func() == 1
assert mock_func() == 2
assert mock_func() == 3

# Mock that raises exception
mock_func = Mock(side_effect=ValueError("Invalid input"))
with pytest.raises(ValueError):
    mock_func()
```

### Patching Methods

```python
from unittest.mock import patch

class EmailService:
    def send_email(self, to, subject, body):
        # Actually sends email
        pass

class UserRegistration:
    def __init__(self, email_service):
        self.email_service = email_service
    
    def register(self, email):
        # Register user...
        self.email_service.send_email(
            to=email,
            subject="Welcome!",
            body="Thanks for registering"
        )
        return True

# Test without actually sending emails
def test_user_registration():
    email_service = EmailService()
    
    with patch.object(email_service, 'send_email') as mock_send:
        registration = UserRegistration(email_service)
        result = registration.register('user@example.com')
        
        # Verify email was "sent"
        assert result is True
        mock_send.assert_called_once_with(
            to='user@example.com',
            subject="Welcome!",
            body="Thanks for registering"
        )
```

### pytest-mock

```python
# Install: pip install pytest-mock

def test_with_mocker(mocker):
    """Using pytest-mock plugin"""
    mock_get = mocker.patch('requests.get')
    mock_get.return_value.json.return_value = {'data': 'test'}
    
    result = get_user_data(1)
    assert result['data'] == 'test'
    mock_get.assert_called_once()
```

### Mock Assertions

```python
from unittest.mock import Mock

mock = Mock()

# Call the mock
mock('arg1', 'arg2', key='value')
mock('arg3')

# Assertions
mock.assert_called()  # Was called at least once
mock.assert_called_once()  # Was called exactly once (fails here)
mock.assert_called_with('arg3')  # Last call had these args
mock.assert_any_call('arg1', 'arg2', key='value')  # Was called with these args

# Check call count
assert mock.call_count == 2

# Check all calls
assert len(mock.call_args_list) == 2
```

---

## Test Coverage

Test coverage measures how much of your code is tested.

### Using coverage.py

```bash
# Install
pip install coverage

# Run tests with coverage
coverage run -m pytest

# Generate report
coverage report

# Generate HTML report
coverage html
# Open htmlcov/index.html in browser
```

### Coverage Report Example

```python
# calculator.py
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

# test_calculator.py
import pytest
from calculator import Calculator

def test_add():
    calc = Calculator()
    assert calc.add(5, 3) == 8

def test_subtract():
    calc = Calculator()
    assert calc.subtract(10, 3) == 7

# Running coverage will show multiply and divide are not tested
```

### pytest-cov

```bash
# Install
pip install pytest-cov

# Run with coverage
pytest --cov=mymodule tests/

# Generate HTML report
pytest --cov=mymodule --cov-report=html tests/
```

### Coverage Configuration

```ini
# .coveragerc
[run]
source = myapp
omit = 
    */tests/*
    */venv/*
    */__pycache__/*

[report]
exclude_lines =
    pragma: no cover
    def __repr__
    raise AssertionError
    raise NotImplementedError
    if __name__ == .__main__.:
```

---

## Parameterized Tests

Run the same test with different inputs.

### pytest.mark.parametrize

```python
import pytest

def is_palindrome(text):
    return text == text[::-1]

@pytest.mark.parametrize("text,expected", [
    ("racecar", True),
    ("hello", False),
    ("madam", True),
    ("python", False),
    ("A", True),
    ("", True),
])
def test_is_palindrome(text, expected):
    assert is_palindrome(text) == expected

# Runs 6 separate tests
```

### Multiple Parameters

```python
import pytest

def add(a, b):
    return a + b

@pytest.mark.parametrize("a,b,expected", [
    (1, 2, 3),
    (5, 5, 10),
    (-1, 1, 0),
    (0, 0, 0),
    (100, 200, 300),
])
def test_add(a, b, expected):
    assert add(a, b) == expected
```

### Parameterizing Fixtures

```python
import pytest

@pytest.fixture(params=['sqlite', 'postgres', 'mysql'])
def database(request):
    """Test with different database backends"""
    db_type = request.param
    # Setup database based on type
    yield db_type
    # Cleanup

def test_database_operations(database):
    """This test runs 3 times, once for each database"""
    print(f"Testing with {database}")
    assert database in ['sqlite', 'postgres', 'mysql']
```

### unittest Subtests

```python
import unittest

class TestMath(unittest.TestCase):
    
    def test_add_multiple_cases(self):
        test_cases = [
            (1, 2, 3),
            (5, 5, 10),
            (-1, 1, 0),
            (0, 0, 0),
        ]
        
        for a, b, expected in test_cases:
            with self.subTest(a=a, b=b):
                self.assertEqual(a + b, expected)
```

---

## Testing Exceptions

### pytest.raises

```python
import pytest

def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b

def test_divide_by_zero():
    with pytest.raises(ValueError):
        divide(10, 0)

def test_divide_by_zero_with_message():
    with pytest.raises(ValueError, match="Cannot divide by zero"):
        divide(10, 0)

def test_divide_by_zero_detailed():
    with pytest.raises(ValueError) as exc_info:
        divide(10, 0)
    
    assert "divide by zero" in str(exc_info.value)
```

### unittest assertRaises

```python
import unittest

class TestExceptions(unittest.TestCase):
    
    def test_divide_by_zero(self):
        with self.assertRaises(ValueError):
            divide(10, 0)
    
    def test_divide_by_zero_with_message(self):
        with self.assertRaisesRegex(ValueError, "Cannot divide"):
            divide(10, 0)
```

---

## Best Practices

### 1. Test Organization

```python
# Good structure
tests/
    __init__.py
    conftest.py          # Shared fixtures
    unit/
        test_models.py
        test_utils.py
    integration/
        test_api.py
        test_database.py
    e2e/
        test_workflows.py
```

### 2. Test Naming

```python
# Good: Descriptive names
def test_user_registration_with_valid_email():
    pass

def test_user_registration_with_invalid_email_raises_error():
    pass

# Bad: Unclear names
def test_user():
    pass

def test_1():
    pass
```

### 3. AAA Pattern (Arrange-Act-Assert)

```python
def test_shopping_cart_total():
    # Arrange: Set up test data
    cart = ShoppingCart()
    cart.add_item('Apple', 1.5)
    cart.add_item('Banana', 0.75)
    
    # Act: Perform the action
    total = cart.calculate_total()
    
    # Assert: Verify the result
    assert total == 2.25
```

### 4. One Assertion Per Test (When Possible)

```python
# Good: Focused tests
def test_user_name():
    user = User('Alice', 'alice@example.com')
    assert user.name == 'Alice'

def test_user_email():
    user = User('Alice', 'alice@example.com')
    assert user.email == 'alice@example.com'

# Acceptable: Related assertions
def test_user_creation():
    user = User('Alice', 'alice@example.com')
    assert user.name == 'Alice'
    assert user.email == 'alice@example.com'
    assert user.is_active is True
```

### 5. Test Independence

```python
# Bad: Tests depend on each other
class TestBad(unittest.TestCase):
    user = None
    
    def test_create_user(self):
        self.user = User('Alice')  # Bad: modifies class state
    
    def test_user_name(self):
        assert self.user.name == 'Alice'  # Bad: depends on previous test

# Good: Each test is independent
class TestGood(unittest.TestCase):
    
    def test_create_user(self):
        user = User('Alice')
        assert user.name == 'Alice'
    
    def test_user_name(self):
        user = User('Alice')
        assert user.name == 'Alice'
```

---

## Integration and End-to-End Testing

### Integration Test Example

```python
import pytest

class Database:
    def save(self, data):
        # Save to actual database
        pass

class UserService:
    def __init__(self, database):
        self.database = database
    
    def create_user(self, name, email):
        user_data = {'name': name, 'email': email}
        self.database.save(user_data)
        return user_data

# Integration test (tests Database + UserService together)
@pytest.fixture
def test_database():
    db = Database()
    # Setup test database
    yield db
    # Cleanup test database

def test_user_creation_integration(test_database):
    service = UserService(test_database)
    user = service.create_user('Alice', 'alice@example.com')
    
    assert user['name'] == 'Alice'
    assert user['email'] == 'alice@example.com'
    # Verify user actually saved to database
    # saved_user = test_database.query('users', name='Alice')
    # assert saved_user is not None
```

### Marks for Different Test Types

```python
import pytest

@pytest.mark.unit
def test_calculation():
    assert 2 + 2 == 4

@pytest.mark.integration
def test_database_integration():
    # Tests database operations
    pass

@pytest.mark.slow
def test_complex_operation():
    # Test that takes a long time
    pass

# Run only unit tests:
# pytest -m unit

# Run only integration tests:
# pytest -m integration

# Skip slow tests:
# pytest -m "not slow"
```

---

## Key Takeaways

- **Write tests** to catch bugs early and enable confident refactoring
- **Use unittest** for built-in testing or **pytest** for cleaner syntax
- **Follow TDD** (Red-Green-Refactor) for better code design
- **Use fixtures** to set up test dependencies cleanly
- **Mock external dependencies** to isolate units under test
- **Measure coverage** but aim for meaningful tests, not just high percentages
- **Parameterize tests** to avoid repetition
- **Test exceptions** to ensure errors are handled correctly
- **Follow best practices**: AAA pattern, descriptive names, test independence
- **Organize tests** into unit, integration, and e2e categories
- **Aim for 80%+ coverage** on critical code paths

Testing is an essential skill that improves code quality, reduces bugs, and makes development faster and more confident!