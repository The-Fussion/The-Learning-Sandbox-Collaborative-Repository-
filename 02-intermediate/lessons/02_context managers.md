# Python Context Managers: A Detailed Guide

## Table of Contents
1. [What Are Context Managers?](#what-are-context-managers)
2. [The `with` Statement](#the-with-statement)
3. [Creating Context Managers with Classes](#creating-context-managers-with-classes)
4. [The `__enter__` and `__exit__` Methods](#the-enter-and-exit-methods)
5. [Exception Handling in Context Managers](#exception-handling-in-context-managers)
6. [Using `contextlib` Module](#using-contextlib-module)
7. [Practical Examples](#practical-examples)
8. [Advanced Patterns](#advanced-patterns)
9. [Common Use Cases](#common-use-cases)

---

## What Are Context Managers?

A context manager is an object that defines the runtime context to be established when executing a `with` statement. Context managers are primarily used for resource management, ensuring that resources are properly acquired and released, even if errors occur.

**Key Benefits:**
- Automatic setup and teardown of resources
- Exception-safe resource handling
- Cleaner, more readable code
- Prevents resource leaks (files, connections, locks, etc.)

---

## The `with` Statement

The `with` statement simplifies exception handling by encapsulating common preparation and cleanup tasks.

### Without Context Manager

```python
# Manual resource management
file = open('example.txt', 'w')
try:
    file.write('Hello, World!')
finally:
    file.close()  # Must remember to close
```

### With Context Manager

```python
# Automatic resource management
with open('example.txt', 'w') as file:
    file.write('Hello, World!')
# File is automatically closed here, even if an exception occurs
```

---

## Creating Context Managers with Classes

To create a context manager, define a class with `__enter__` and `__exit__` methods.

### Basic Structure

```python
class MyContextManager:
    def __enter__(self):
        # Setup code goes here
        print("Entering the context")
        return self  # This is what gets assigned to the 'as' variable
    
    def __exit__(self, exc_type, exc_value, traceback):
        # Cleanup code goes here
        print("Exiting the context")
        return False  # Don't suppress exceptions

# Usage
with MyContextManager() as manager:
    print("Inside the context")

# Output:
# Entering the context
# Inside the context
# Exiting the context
```

---

## The `__enter__` and `__exit__` Methods

### `__enter__(self)`

- Called when execution enters the `with` block
- Performs setup operations
- Returns the resource to be managed (assigned to the variable after `as`)
- Can return `self` or any other object

```python
class DatabaseConnection:
    def __enter__(self):
        print("Opening database connection")
        self.connection = "DB Connection Object"
        return self.connection  # This gets assigned to 'conn'

    def __exit__(self, exc_type, exc_value, traceback):
        print("Closing database connection")
        return False

with DatabaseConnection() as conn:
    print(f"Using {conn}")

# Output:
# Opening database connection
# Using DB Connection Object
# Closing database connection
```

### `__exit__(self, exc_type, exc_value, traceback)`

- Called when execution leaves the `with` block
- Performs cleanup operations
- Receives exception information if an exception occurred
- Parameters:
  - `exc_type`: Exception class (or `None` if no exception)
  - `exc_value`: Exception instance (or `None`)
  - `traceback`: Traceback object (or `None`)
- Return value:
  - `True`: Suppress the exception
  - `False` or `None`: Propagate the exception

---

## Exception Handling in Context Managers

### Propagating Exceptions

```python
class FileManager:
    def __init__(self, filename):
        self.filename = filename
    
    def __enter__(self):
        print(f"Opening {self.filename}")
        self.file = open(self.filename, 'w')
        return self.file
    
    def __exit__(self, exc_type, exc_value, traceback):
        print(f"Closing {self.filename}")
        self.file.close()
        
        if exc_type is not None:
            print(f"Exception occurred: {exc_type.__name__}: {exc_value}")
        
        return False  # Propagate the exception

try:
    with FileManager('test.txt') as f:
        f.write('Hello')
        raise ValueError("Something went wrong!")
except ValueError as e:
    print(f"Caught exception: {e}")

# Output:
# Opening test.txt
# Closing test.txt
# Exception occurred: ValueError: Something went wrong!
# Caught exception: Something went wrong!
```

### Suppressing Exceptions

```python
class ErrorSuppressor:
    def __enter__(self):
        print("Entering context")
        return self
    
    def __exit__(self, exc_type, exc_value, traceback):
        print("Exiting context")
        
        if exc_type is ValueError:
            print(f"Suppressing ValueError: {exc_value}")
            return True  # Suppress the exception
        
        return False  # Propagate other exceptions

with ErrorSuppressor():
    print("This will execute")
    raise ValueError("This error will be suppressed")
    print("This won't execute")

print("Execution continues normally")

# Output:
# Entering context
# This will execute
# Exiting context
# Suppressing ValueError: This error will be suppressed
# Execution continues normally
```

---

## Using `contextlib` Module

The `contextlib` module provides utilities for working with context managers, making them easier to create.

### Using `@contextmanager` Decorator

The `@contextmanager` decorator allows you to write a context manager using a generator function.

```python
from contextlib import contextmanager

@contextmanager
def my_context():
    # Setup code (before yield)
    print("Entering context")
    resource = "My Resource"
    
    try:
        yield resource  # This value is assigned to the 'as' variable
    finally:
        # Cleanup code (after yield)
        print("Exiting context")

with my_context() as res:
    print(f"Using {res}")

# Output:
# Entering context
# Using My Resource
# Exiting context
```

### File Manager Example

```python
from contextlib import contextmanager

@contextmanager
def file_manager(filename, mode):
    print(f"Opening {filename}")
    file = open(filename, mode)
    try:
        yield file
    finally:
        print(f"Closing {filename}")
        file.close()

with file_manager('test.txt', 'w') as f:
    f.write('Hello, World!')

# Output:
# Opening test.txt
# Closing test.txt
```

### Handling Exceptions with `@contextmanager`

```python
from contextlib import contextmanager

@contextmanager
def error_handler():
    print("Setup")
    try:
        yield
    except ValueError as e:
        print(f"Caught and handled: {e}")
        # Exception is suppressed (not re-raised)
    finally:
        print("Cleanup")

with error_handler():
    print("Doing work")
    raise ValueError("Something went wrong")

print("Continuing...")

# Output:
# Setup
# Doing work
# Caught and handled: Something went wrong
# Cleanup
# Continuing...
```

---

## Practical Examples

### Database Transaction Manager

```python
from contextlib import contextmanager

class Database:
    def __init__(self):
        self.connected = False
        self.in_transaction = False
    
    def connect(self):
        print("Connecting to database")
        self.connected = True
    
    def disconnect(self):
        print("Disconnecting from database")
        self.connected = False
    
    def begin_transaction(self):
        print("Beginning transaction")
        self.in_transaction = True
    
    def commit(self):
        print("Committing transaction")
        self.in_transaction = False
    
    def rollback(self):
        print("Rolling back transaction")
        self.in_transaction = False

@contextmanager
def database_transaction(db):
    db.connect()
    db.begin_transaction()
    
    try:
        yield db
        db.commit()
    except Exception as e:
        db.rollback()
        print(f"Transaction failed: {e}")
        raise
    finally:
        db.disconnect()

# Usage
db = Database()

with database_transaction(db):
    print("Executing database operations")
    # Simulated operations

# Output:
# Connecting to database
# Beginning transaction
# Executing database operations
# Committing transaction
# Disconnecting from database
```

### Timer Context Manager

```python
import time
from contextlib import contextmanager

@contextmanager
def timer(name):
    start_time = time.time()
    print(f"Starting {name}")
    
    yield
    
    end_time = time.time()
    elapsed = end_time - start_time
    print(f"{name} took {elapsed:.4f} seconds")

with timer("Data Processing"):
    time.sleep(1)
    print("Processing data...")

# Output:
# Starting Data Processing
# Processing data...
# Data Processing took 1.0001 seconds
```

### Temporary Directory Manager

```python
import os
import shutil
from contextlib import contextmanager

@contextmanager
def temporary_directory(prefix='tmp'):
    temp_dir = f"{prefix}_{os.getpid()}"
    
    print(f"Creating temporary directory: {temp_dir}")
    os.makedirs(temp_dir, exist_ok=True)
    
    try:
        yield temp_dir
    finally:
        print(f"Removing temporary directory: {temp_dir}")
        shutil.rmtree(temp_dir, ignore_errors=True)

with temporary_directory('work') as temp_dir:
    print(f"Working in {temp_dir}")
    # Create files in temp_dir
    with open(f"{temp_dir}/test.txt", 'w') as f:
        f.write("Temporary file")

# Output:
# Creating temporary directory: work_12345
# Working in work_12345
# Removing temporary directory: work_12345
```

### Lock Manager (Thread Synchronization)

```python
import threading
from contextlib import contextmanager

class SharedResource:
    def __init__(self):
        self.lock = threading.Lock()
        self.value = 0
    
    @contextmanager
    def acquire_lock(self):
        print(f"Thread {threading.current_thread().name} acquiring lock")
        self.lock.acquire()
        
        try:
            yield self
        finally:
            print(f"Thread {threading.current_thread().name} releasing lock")
            self.lock.release()

resource = SharedResource()

with resource.acquire_lock() as res:
    res.value += 1
    print(f"Value is now {res.value}")

# Output:
# Thread MainThread acquiring lock
# Value is now 1
# Thread MainThread releasing lock
```

---

## Advanced Patterns

### Nested Context Managers

```python
from contextlib import contextmanager

@contextmanager
def outer():
    print("Outer setup")
    yield "outer resource"
    print("Outer cleanup")

@contextmanager
def inner():
    print("Inner setup")
    yield "inner resource"
    print("Inner cleanup")

with outer() as o:
    with inner() as i:
        print(f"Using {o} and {i}")

# Output:
# Outer setup
# Inner setup
# Using outer resource and inner resource
# Inner cleanup
# Outer cleanup
```

### Multiple Context Managers (Python 3.1+)

```python
with open('input.txt', 'r') as infile, open('output.txt', 'w') as outfile:
    content = infile.read()
    outfile.write(content.upper())
```

### Using `contextlib.ExitStack`

Manage a dynamic number of context managers:

```python
from contextlib import ExitStack

filenames = ['file1.txt', 'file2.txt', 'file3.txt']

with ExitStack() as stack:
    files = [stack.enter_context(open(fname, 'w')) for fname in filenames]
    
    for i, f in enumerate(files):
        f.write(f"Content for file {i+1}\n")

# All files are automatically closed
```

### Reusable Context Manager

```python
class ReusableContextManager:
    def __init__(self, name):
        self.name = name
    
    def __enter__(self):
        print(f"Entering {self.name}")
        return self
    
    def __exit__(self, exc_type, exc_value, traceback):
        print(f"Exiting {self.name}")
        return False

manager = ReusableContextManager("MyManager")

# Can be reused multiple times
with manager:
    print("First use")

with manager:
    print("Second use")

# Output:
# Entering MyManager
# First use
# Exiting MyManager
# Entering MyManager
# Second use
# Exiting MyManager
```

---

## Common Use Cases

### File Operations
```python
with open('data.txt', 'r') as f:
    data = f.read()
```

### Database Connections
```python
with database.connection() as conn:
    conn.execute("SELECT * FROM users")
```

### Thread Locks
```python
with lock:
    # Critical section
    shared_resource.modify()
```

### Changing Directories
```python
@contextmanager
def change_dir(path):
    old_dir = os.getcwd()
    os.chdir(path)
    try:
        yield
    finally:
        os.chdir(old_dir)

with change_dir('/tmp'):
    # Work in /tmp
    pass
# Back to original directory
```

### Suppressing Exceptions
```python
from contextlib import suppress

with suppress(FileNotFoundError):
    os.remove('nonexistent_file.txt')
# FileNotFoundError is silently ignored
```

### Redirecting Output
```python
from contextlib import redirect_stdout
import io

f = io.StringIO()
with redirect_stdout(f):
    print("This goes to the StringIO object")

output = f.getvalue()
print(f"Captured: {output}")
```

---

## Key Takeaways

- Context managers ensure proper resource management with automatic setup and cleanup
- Use `__enter__` and `__exit__` methods for class-based context managers
- Use `@contextmanager` decorator with generators for simpler implementations
- The `with` statement guarantees cleanup code runs, even if exceptions occur
- `__exit__` can suppress exceptions by returning `True`
- Context managers are perfect for files, locks, connections, and any resource that needs cleanup
- Use `contextlib` utilities like `ExitStack`, `suppress`, and `redirect_stdout` for advanced patterns
- Context managers make code cleaner, safer, and more Pythonic

Context managers are essential for writing robust Python code that properly handles resources and prevents leaks!