# Python Type Hints and Annotations: A Detailed Guide

## Table of Contents
1. [Introduction to Type Hints](#introduction-to-type-hints)
2. [Basic Type Annotations](#basic-type-annotations)
3. [The `typing` Module](#the-typing-module)
4. [Function Annotations](#function-annotations)
5. [Collection Types](#collection-types)
6. [Optional and Union Types](#optional-and-union-types)
7. [Type Aliases](#type-aliases)
8. [Generic Types](#generic-types)
9. [Protocol and Abstract Types](#protocol-and-abstract-types)
10. [Static Type Checking with mypy](#static-type-checking-with-mypy)
11. [Advanced Type Hints](#advanced-type-hints)
12. [Best Practices](#best-practices)

---

## Introduction to Type Hints

Type hints (annotations) are optional metadata that specify the expected types of variables, function parameters, and return values.

### Why Use Type Hints?

- **Better documentation**: Makes code self-documenting
- **IDE support**: Enables autocomplete and better IntelliSense
- **Early error detection**: Catch type errors before runtime
- **Refactoring safety**: Easier to refactor with confidence
- **Team collaboration**: Clearer interfaces for team members

### Important Notes

- Type hints are **optional** and **don't affect runtime behavior**
- Python remains dynamically typed
- Type hints are checked by external tools (mypy, pyright, etc.)
- They're mainly for documentation and static analysis

```python
# Without type hints
def greet(name):
    return f"Hello, {name}!"

# With type hints
def greet(name: str) -> str:
    return f"Hello, {name}!"

# Both work exactly the same at runtime
print(greet("Alice"))  # Hello, Alice!
print(greet(123))      # Hello, 123! (no runtime error)
```

---

## Basic Type Annotations

### Variable Annotations

```python
# Basic types
name: str = "Alice"
age: int = 30
height: float = 5.6
is_student: bool = True

# Without initial value
username: str
count: int

# Later assignment
username = "bob123"
count = 42

# Multiple variables
x: int
y: int
x, y = 10, 20
```

### Built-in Types

```python
# Numeric types
integer: int = 42
floating: float = 3.14
complex_num: complex = 3 + 4j

# Text
text: str = "Hello"
byte_string: bytes = b"Hello"

# Boolean
flag: bool = True

# None type
result: None = None
```

---

## The `typing` Module

The `typing` module provides more complex type hints.

### Common typing Imports

```python
from typing import (
    List, Dict, Set, Tuple,
    Optional, Union, Any,
    Callable, Iterator, Iterable,
    TypeVar, Generic, Protocol
)
```

### Any Type

```python
from typing import Any

def process_data(data: Any) -> Any:
    """Accepts and returns any type"""
    return data

# Can pass anything
result = process_data(42)
result = process_data("text")
result = process_data([1, 2, 3])
```

---

## Function Annotations

### Basic Function Annotations

```python
def add(a: int, b: int) -> int:
    """Add two integers and return the result"""
    return a + b

def greet(name: str) -> str:
    """Greet a person by name"""
    return f"Hello, {name}!"

def print_message(message: str) -> None:
    """Print a message (returns nothing)"""
    print(message)
```

### Multiple Return Types

```python
from typing import Tuple

def get_user_info(user_id: int) -> Tuple[str, int, str]:
    """Returns (name, age, email)"""
    return ("Alice", 30, "alice@example.com")

# Unpack the tuple
name, age, email = get_user_info(1)
```

### Default Arguments

```python
def create_user(
    name: str,
    age: int = 18,
    active: bool = True
) -> dict:
    """Create a user with optional age and active status"""
    return {
        'name': name,
        'age': age,
        'active': active
    }

user1 = create_user("Alice")
user2 = create_user("Bob", 25)
user3 = create_user("Charlie", 30, False)
```

### *args and **kwargs

```python
from typing import Any

def sum_numbers(*args: int) -> int:
    """Sum variable number of integers"""
    return sum(args)

def print_info(**kwargs: Any) -> None:
    """Print keyword arguments"""
    for key, value in kwargs.items():
        print(f"{key}: {value}")

# Usage
total = sum_numbers(1, 2, 3, 4, 5)
print_info(name="Alice", age=30, city="NYC")
```

---

## Collection Types

### Lists

```python
from typing import List

# List of integers
numbers: List[int] = [1, 2, 3, 4, 5]

# List of strings
names: List[str] = ["Alice", "Bob", "Charlie"]

# Nested lists
matrix: List[List[int]] = [
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9]
]

# Function with list parameter
def process_numbers(numbers: List[int]) -> List[int]:
    """Double each number in the list"""
    return [n * 2 for n in numbers]
```

### Modern Syntax (Python 3.9+)

```python
# Python 3.9+ allows using built-in types directly
numbers: list[int] = [1, 2, 3]
names: list[str] = ["Alice", "Bob"]

def sum_list(numbers: list[int]) -> int:
    return sum(numbers)
```

### Dictionaries

```python
from typing import Dict

# Dictionary with string keys and int values
scores: Dict[str, int] = {
    "Alice": 95,
    "Bob": 87,
    "Charlie": 92
}

# Dictionary with int keys and string values
id_to_name: Dict[int, str] = {
    1: "Alice",
    2: "Bob",
    3: "Charlie"
}

# Nested dictionaries
config: Dict[str, Dict[str, str]] = {
    "database": {
        "host": "localhost",
        "port": "5432"
    },
    "cache": {
        "host": "redis",
        "port": "6379"
    }
}

# Python 3.9+
user_data: dict[str, int] = {"age": 30, "score": 95}
```

### Sets and Tuples

```python
from typing import Set, Tuple

# Set of strings
tags: Set[str] = {"python", "programming", "tutorial"}

# Tuple with fixed types
coordinates: Tuple[float, float] = (40.7128, -74.0060)

# Tuple with variable length (all same type)
from typing import Tuple as VariableTuple
numbers: VariableTuple[int, ...] = (1, 2, 3, 4, 5)

# Python 3.9+
unique_ids: set[int] = {1, 2, 3, 4, 5}
point: tuple[float, float] = (10.5, 20.3)
```

---

## Optional and Union Types

### Optional

```python
from typing import Optional

# Optional[str] means "str or None"
def find_user(user_id: int) -> Optional[str]:
    """Returns username or None if not found"""
    users = {1: "Alice", 2: "Bob"}
    return users.get(user_id)

# Variable that might be None
username: Optional[str] = None
username = "alice123"

# Equivalent to Union[str, None]
def get_config(key: str) -> Optional[dict]:
    """Returns config dict or None"""
    configs = {"debug": {"level": "info"}}
    return configs.get(key)
```

### Union Types

```python
from typing import Union

# Can be int or str
def process_id(id: Union[int, str]) -> str:
    """Process ID as integer or string"""
    return str(id)

# Can be list or dict
def process_data(data: Union[List[int], Dict[str, int]]) -> int:
    """Process list or dict of integers"""
    if isinstance(data, list):
        return sum(data)
    else:
        return sum(data.values())

# Python 3.10+ syntax (using | operator)
def display_value(value: int | str | float) -> str:
    """Display value as string"""
    return str(value)
```

### Literal Types

```python
from typing import Literal

# Only specific values allowed
def set_status(status: Literal["active", "inactive", "pending"]) -> None:
    """Set status to one of three specific values"""
    print(f"Status set to: {status}")

# Type checker will catch invalid values
set_status("active")    # OK
set_status("deleted")   # Type error

# Multiple literals
def get_direction(dir: Literal["up", "down", "left", "right"]) -> str:
    return f"Moving {dir}"
```

---

## Type Aliases

Type aliases create readable names for complex types.

### Simple Aliases

```python
from typing import List, Dict, Tuple

# Define aliases
UserId = int
Username = str
Coordinates = Tuple[float, float]
UserData = Dict[str, Union[str, int]]

# Use aliases
def get_user(user_id: UserId) -> Username:
    """Get username by user ID"""
    return "alice123"

def get_location(user_id: UserId) -> Coordinates:
    """Get user's coordinates"""
    return (40.7128, -74.0060)

def create_user(data: UserData) -> UserId:
    """Create user from data dictionary"""
    return 1
```

### Complex Aliases

```python
from typing import List, Dict, Tuple, Optional

# Complex nested type
JSON = Union[
    None,
    bool,
    int,
    float,
    str,
    List['JSON'],
    Dict[str, 'JSON']
]

def parse_json(data: str) -> JSON:
    """Parse JSON string into Python object"""
    import json
    return json.loads(data)

# Network response type
Response = Tuple[int, Dict[str, str], bytes]

def make_request(url: str) -> Response:
    """Returns (status_code, headers, body)"""
    return (200, {"Content-Type": "text/html"}, b"<html></html>")
```

### NewType

```python
from typing import NewType

# Create distinct types (for type checking only)
UserId = NewType('UserId', int)
ProductId = NewType('ProductId', int)

def get_user(user_id: UserId) -> str:
    return f"User {user_id}"

def get_product(product_id: ProductId) -> str:
    return f"Product {product_id}"

# Must explicitly create the new type
user_id = UserId(42)
product_id = ProductId(100)

get_user(user_id)        # OK
# get_user(product_id)   # Type error!
# get_user(42)           # Type error!
```

---

## Generic Types

Generics allow you to write flexible, reusable code with type safety.

### TypeVar

```python
from typing import TypeVar, List

T = TypeVar('T')

def first_element(items: List[T]) -> T:
    """Get the first element of a list"""
    return items[0]

# Works with any type
numbers = [1, 2, 3]
first_num = first_element(numbers)  # Type: int

names = ["Alice", "Bob"]
first_name = first_element(names)   # Type: str
```

### Generic Classes

```python
from typing import Generic, TypeVar

T = TypeVar('T')

class Stack(Generic[T]):
    """A generic stack implementation"""
    
    def __init__(self) -> None:
        self._items: List[T] = []
    
    def push(self, item: T) -> None:
        self._items.append(item)
    
    def pop(self) -> T:
        return self._items.pop()
    
    def peek(self) -> T:
        return self._items[-1]
    
    def is_empty(self) -> bool:
        return len(self._items) == 0

# Create type-specific stacks
int_stack: Stack[int] = Stack()
int_stack.push(1)
int_stack.push(2)

str_stack: Stack[str] = Stack()
str_stack.push("hello")
str_stack.push("world")
```

### Bounded TypeVar

```python
from typing import TypeVar

# T must be int or float
T = TypeVar('T', int, float)

def add_numbers(a: T, b: T) -> T:
    """Add two numbers (int or float only)"""
    return a + b

# OK
result1 = add_numbers(5, 3)      # int
result2 = add_numbers(5.5, 3.2)  # float

# Type error
# result3 = add_numbers("5", "3")  # str not allowed
```

### Multiple TypeVars

```python
from typing import TypeVar, Tuple

K = TypeVar('K')
V = TypeVar('V')

def get_first_item(items: Dict[K, V]) -> Tuple[K, V]:
    """Get first key-value pair from dictionary"""
    key = next(iter(items))
    return (key, items[key])

# Works with any dict types
int_str_dict = {1: "one", 2: "two"}
first = get_first_item(int_str_dict)  # Type: Tuple[int, str]
```

---

## Protocol and Abstract Types

### Callable

```python
from typing import Callable

# Function that takes (int, int) and returns int
BinaryOperation = Callable[[int, int], int]

def apply_operation(a: int, b: int, operation: BinaryOperation) -> int:
    """Apply a binary operation to two numbers"""
    return operation(a, b)

# Usage
def add(x: int, y: int) -> int:
    return x + y

def multiply(x: int, y: int) -> int:
    return x * y

result1 = apply_operation(5, 3, add)       # 8
result2 = apply_operation(5, 3, multiply)  # 15

# With lambda
result3 = apply_operation(5, 3, lambda x, y: x - y)  # 2
```

### Protocol (Structural Subtyping)

```python
from typing import Protocol

class Drawable(Protocol):
    """Protocol for drawable objects"""
    
    def draw(self) -> str:
        ...

class Circle:
    def draw(self) -> str:
        return "Drawing circle"

class Square:
    def draw(self) -> str:
        return "Drawing square"

def render(shape: Drawable) -> None:
    """Render any drawable shape"""
    print(shape.draw())

# Both work because they implement draw()
circle = Circle()
square = Square()

render(circle)  # OK
render(square)  # OK
```

### Runtime Checkable Protocol

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Sized(Protocol):
    """Protocol for objects with a length"""
    
    def __len__(self) -> int:
        ...

def print_size(obj: Sized) -> None:
    """Print the size of any sized object"""
    print(f"Size: {len(obj)}")

# Works with any object that has __len__
print_size([1, 2, 3])        # Size: 3
print_size("hello")          # Size: 5
print_size({"a": 1, "b": 2}) # Size: 2

# Runtime check
if isinstance([1, 2, 3], Sized):
    print("List is Sized")
```

---

## Static Type Checking with mypy

mypy is the most popular static type checker for Python.

### Installing mypy

```bash
pip install mypy
```

### Running mypy

```bash
# Check a single file
mypy script.py

# Check a directory
mypy src/

# Check with more strict settings
mypy --strict script.py
```

### Example Code

```python
# example.py
def greet(name: str) -> str:
    return f"Hello, {name}!"

def add(a: int, b: int) -> int:
    return a + b

# Correct usage
result1 = greet("Alice")
result2 = add(5, 3)

# Type errors (mypy will catch these)
result3 = greet(123)      # Error: Argument 1 has incompatible type "int"
result4 = add("5", "3")   # Error: Argument 1 has incompatible type "str"
```

### mypy Configuration

```ini
# mypy.ini or setup.cfg
[mypy]
python_version = 3.9
warn_return_any = True
warn_unused_configs = True
disallow_untyped_defs = True
disallow_incomplete_defs = True
check_untyped_defs = True
no_implicit_optional = True
warn_redundant_casts = True
warn_unused_ignores = True
strict_equality = True

# Ignore specific files
[mypy-tests.*]
ignore_errors = True
```

### Type Ignore Comments

```python
# Ignore type error on a specific line
result = some_untyped_library_function()  # type: ignore

# Ignore specific error
value = cast_to_int("123")  # type: ignore[arg-type]

# Reveal type (for debugging)
reveal_type(result)  # mypy will print the inferred type
```

---

## Advanced Type Hints

### Overload

```python
from typing import overload, Union

@overload
def process(value: int) -> int:
    ...

@overload
def process(value: str) -> str:
    ...

def process(value: Union[int, str]) -> Union[int, str]:
    """Process int or str, returning same type"""
    if isinstance(value, int):
        return value * 2
    else:
        return value.upper()

# mypy knows the exact return type
num = process(5)      # Type: int
text = process("hi")  # Type: str
```

### Final

```python
from typing import Final

# Constant that shouldn't be reassigned
MAX_SIZE: Final = 100
API_KEY: Final[str] = "secret123"

# MAX_SIZE = 200  # mypy error: Cannot assign to final name

class Config:
    # Final class attribute
    VERSION: Final = "1.0.0"
    
    def __init__(self):
        # Final instance attribute
        self.id: Final = 12345

from typing import final

@final
class BaseClass:
    """Cannot be subclassed"""
    pass

# class Derived(BaseClass):  # mypy error
#     pass
```

### ClassVar

```python
from typing import ClassVar

class Counter:
    # Class variable (shared by all instances)
    count: ClassVar[int] = 0
    
    def __init__(self, name: str):
        # Instance variable
        self.name: str = name
        Counter.count += 1

c1 = Counter("first")
c2 = Counter("second")
print(Counter.count)  # 2
```

### TypedDict

```python
from typing import TypedDict

class User(TypedDict):
    """Type-safe dictionary structure"""
    name: str
    age: int
    email: str

def create_user(name: str, age: int, email: str) -> User:
    """Create a user dictionary"""
    return {
        'name': name,
        'age': age,
        'email': email
    }

user: User = create_user("Alice", 30, "alice@example.com")

# Type error if missing keys
# invalid_user: User = {'name': 'Bob'}  # Missing 'age' and 'email'

# Optional fields
class OptionalUser(TypedDict, total=False):
    name: str
    age: int
    email: str  # All fields are optional

partial_user: OptionalUser = {'name': 'Charlie'}  # OK
```

### Self Type (Python 3.11+)

```python
from typing import Self

class Builder:
    def __init__(self) -> None:
        self.value = 0
    
    def add(self, x: int) -> Self:
        """Returns self for chaining"""
        self.value += x
        return self
    
    def multiply(self, x: int) -> Self:
        """Returns self for chaining"""
        self.value *= x
        return self

# Method chaining with correct type
result = Builder().add(5).multiply(3).add(2)
```

---

## Best Practices

### 1. Start Gradually

```python
# Don't try to annotate everything at once
# Start with function signatures

# Good starting point
def calculate_total(prices: list[float]) -> float:
    return sum(prices)

# Add more detail as needed
from typing import List

def calculate_total(prices: List[float]) -> float:
    return sum(prices)
```

### 2. Annotate Public APIs

```python
# Always annotate public functions/methods
class UserService:
    def get_user(self, user_id: int) -> Optional[User]:
        """Public method - annotated"""
        return self._fetch_from_db(user_id)
    
    def _fetch_from_db(self, user_id):
        """Private method - annotation optional"""
        # Internal implementation
        pass
```

### 3. Use Type Aliases for Readability

```python
# Bad: Repeated complex type
def process(data: Dict[str, List[Tuple[int, str]]]) -> List[str]:
    pass

def transform(data: Dict[str, List[Tuple[int, str]]]) -> Dict[str, int]:
    pass

# Good: Use type alias
UserData = Dict[str, List[Tuple[int, str]]]

def process(data: UserData) -> List[str]:
    pass

def transform(data: UserData) -> Dict[str, int]:
    pass
```

### 4. Prefer Specific Types Over Any

```python
from typing import Any

# Bad: Too vague
def process(data: Any) -> Any:
    return data

# Good: Specific types
def process(data: Union[int, str, List[int]]) -> str:
    return str(data)
```

### 5. Document Complex Types

```python
from typing import TypedDict, List

class ProductInfo(TypedDict):
    """
    Product information structure
    
    Fields:
        id: Unique product identifier
        name: Product name
        price: Price in USD
        tags: List of product tags
    """
    id: int
    name: str
    price: float
    tags: List[str]
```

### 6. Use Protocols for Duck Typing

```python
from typing import Protocol

# Instead of requiring specific class
class Drawable(Protocol):
    def draw(self) -> None:
        ...

# Any class with draw() method works
def render_all(items: List[Drawable]) -> None:
    for item in items:
        item.draw()
```

---

## Key Takeaways

- **Type hints** improve code documentation and enable better tooling
- **Basic types**: int, str, float, bool, None
- **Collections**: List, Dict, Set, Tuple (or list, dict, set, tuple in Python 3.9+)
- **Optional** and **Union** for multiple possible types
- **Type aliases** make complex types readable
- **Generics** with TypeVar enable flexible, reusable code
- **Protocols** enable structural subtyping (duck typing with types)
- **mypy** is the standard static type checker
- **Start gradually** - annotate public APIs first
- **Type hints don't affect runtime** - Python remains dynamically typed
- Use **--strict** mode in mypy for maximum type safety
- Type hints are **documentation** that tools can verify

Type hints make Python code more maintainable, easier to understand, and help catch bugs before runtime!