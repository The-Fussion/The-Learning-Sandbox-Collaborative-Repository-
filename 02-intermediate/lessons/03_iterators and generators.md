# Python Iterators and Generators: A Deep Dive

## Table of Contents
1. [Understanding Iteration](#understanding-iteration)
2. [Iterators](#iterators)
3. [Building Custom Iterators](#building-custom-iterators)
4. [Generators](#generators)
5. [Generator Functions with `yield`](#generator-functions-with-yield)
6. [Generator Expressions](#generator-expressions)
7. [Lazy Evaluation and Memory Efficiency](#lazy-evaluation-and-memory-efficiency)
8. [Advanced Generator Techniques](#advanced-generator-techniques)
9. [Practical Examples](#practical-examples)
10. [Iterators vs Generators](#iterators-vs-generators)

---

## Understanding Iteration

Iteration is the process of taking elements from a collection one at a time. In Python, iteration is fundamental to how loops work.

### The Iteration Protocol

Python's iteration protocol consists of two methods:

- `__iter__()`: Returns an iterator object
- `__next__()`: Returns the next item or raises `StopIteration` when exhausted

```python
# How a for loop actually works under the hood
numbers = [1, 2, 3]

# What Python does internally:
iterator = iter(numbers)  # Calls __iter__()

while True:
    try:
        item = next(iterator)  # Calls __next__()
        print(item)
    except StopIteration:
        break

# Output:
# 1
# 2
# 3
```

---

## Iterators

An **iterator** is an object that implements the iterator protocol (`__iter__` and `__next__`).

### Basic Iterator Concepts

```python
# Creating an iterator from a list
my_list = [1, 2, 3, 4, 5]
my_iterator = iter(my_list)

print(next(my_iterator))  # 1
print(next(my_iterator))  # 2
print(next(my_iterator))  # 3

# Iterators are exhaustible
numbers = [1, 2, 3]
it = iter(numbers)

for num in it:
    print(num)  # 1, 2, 3

for num in it:
    print(num)  # Nothing prints - iterator is exhausted
```

### Iterable vs Iterator

- **Iterable**: An object that can return an iterator (has `__iter__()`)
- **Iterator**: An object that produces values one at a time (has both `__iter__()` and `__next__()`)

```python
# List is iterable but not an iterator
my_list = [1, 2, 3]
print(hasattr(my_list, '__iter__'))  # True
print(hasattr(my_list, '__next__'))  # False

# Iterator has both methods
my_iter = iter(my_list)
print(hasattr(my_iter, '__iter__'))  # True
print(hasattr(my_iter, '__next__'))  # True
```

---

## Building Custom Iterators

### Simple Counter Iterator

```python
class Counter:
    def __init__(self, start, end):
        self.current = start
        self.end = end
    
    def __iter__(self):
        return self  # The iterator returns itself
    
    def __next__(self):
        if self.current > self.end:
            raise StopIteration
        
        value = self.current
        self.current += 1
        return value

# Usage
counter = Counter(1, 5)
for num in counter:
    print(num)

# Output: 1 2 3 4 5
```

### Reverse Iterator

```python
class ReverseIterator:
    def __init__(self, data):
        self.data = data
        self.index = len(data)
    
    def __iter__(self):
        return self
    
    def __next__(self):
        if self.index == 0:
            raise StopIteration
        
        self.index -= 1
        return self.data[self.index]

# Usage
rev = ReverseIterator([1, 2, 3, 4, 5])
for item in rev:
    print(item)

# Output: 5 4 3 2 1
```

### Fibonacci Iterator

```python
class Fibonacci:
    def __init__(self, max_count):
        self.max_count = max_count
        self.count = 0
        self.a, self.b = 0, 1
    
    def __iter__(self):
        return self
    
    def __next__(self):
        if self.count >= self.max_count:
            raise StopIteration
        
        self.count += 1
        result = self.a
        self.a, self.b = self.b, self.a + self.b
        return result

# Usage
fib = Fibonacci(10)
print(list(fib))
# Output: [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

### Iterator That Can Be Reused

```python
class ReusableRange:
    def __init__(self, start, end):
        self.start = start
        self.end = end
    
    def __iter__(self):
        # Return a new iterator each time
        return ReusableRangeIterator(self.start, self.end)

class ReusableRangeIterator:
    def __init__(self, start, end):
        self.current = start
        self.end = end
    
    def __iter__(self):
        return self
    
    def __next__(self):
        if self.current >= self.end:
            raise StopIteration
        
        value = self.current
        self.current += 1
        return value

# Usage
r = ReusableRange(1, 4)

# Can iterate multiple times
for num in r:
    print(num, end=' ')  # 1 2 3

print()

for num in r:
    print(num, end=' ')  # 1 2 3 (works again!)
```

---

## Generators

**Generators** are a simpler way to create iterators using functions. They use the `yield` keyword instead of `return`.

### Why Use Generators?

- Simpler syntax than creating iterator classes
- Automatically implement the iterator protocol
- Memory efficient (generate values on-the-fly)
- State is maintained between calls

---

## Generator Functions with `yield`

### Basic Generator

```python
def simple_generator():
    yield 1
    yield 2
    yield 3

gen = simple_generator()
print(next(gen))  # 1
print(next(gen))  # 2
print(next(gen))  # 3
# next(gen) would raise StopIteration

# Or use in a loop
for value in simple_generator():
    print(value)
```

### How `yield` Works

When a function contains `yield`:
1. Calling the function returns a generator object (doesn't execute the function)
2. Calling `next()` executes until the next `yield`
3. The function's state is frozen between `yield` statements
4. Execution resumes from where it left off on the next `next()` call

```python
def countdown(n):
    print("Starting countdown")
    while n > 0:
        print(f"About to yield {n}")
        yield n
        print(f"Resumed after yielding {n}")
        n -= 1
    print("Countdown finished")

gen = countdown(3)
print("Generator created")
print(next(gen))
print(next(gen))
print(next(gen))

# Output:
# Generator created
# Starting countdown
# About to yield 3
# 3
# Resumed after yielding 3
# About to yield 2
# 2
# Resumed after yielding 2
# About to yield 1
# 1
# Resumed after yielding 1
# Countdown finished
```

### Fibonacci Generator

```python
def fibonacci(n):
    a, b = 0, 1
    count = 0
    
    while count < n:
        yield a
        a, b = b, a + b
        count += 1

# Usage
for num in fibonacci(10):
    print(num, end=' ')
# Output: 0 1 1 2 3 5 8 13 21 34
```

### Infinite Generators

```python
def infinite_sequence():
    num = 0
    while True:
        yield num
        num += 1

# Use with caution - it's infinite!
gen = infinite_sequence()
print(next(gen))  # 0
print(next(gen))  # 1
print(next(gen))  # 2

# Better to limit it
def first_n(generator, n):
    for i, value in enumerate(generator):
        if i >= n:
            break
        yield value

for num in first_n(infinite_sequence(), 5):
    print(num, end=' ')
# Output: 0 1 2 3 4
```

### Generator with Parameters

```python
def count_up_to(max_value, step=1):
    current = 0
    while current <= max_value:
        yield current
        current += step

# Usage
for num in count_up_to(10, 2):
    print(num, end=' ')
# Output: 0 2 4 6 8 10
```

---

## Generator Expressions

Generator expressions are similar to list comprehensions but use parentheses instead of brackets and create generators instead of lists.

### Syntax Comparison

```python
# List comprehension - creates entire list in memory
list_comp = [x**2 for x in range(10)]
print(type(list_comp))  # <class 'list'>
print(list_comp)  # [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]

# Generator expression - creates generator object
gen_exp = (x**2 for x in range(10))
print(type(gen_exp))  # <class 'generator'>
print(gen_exp)  # <generator object at 0x...>

# Consume the generator
print(list(gen_exp))  # [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
```

### Memory Efficiency

```python
import sys

# List comprehension
list_comp = [x for x in range(1000000)]
print(f"List size: {sys.getsizeof(list_comp)} bytes")

# Generator expression
gen_exp = (x for x in range(1000000))
print(f"Generator size: {sys.getsizeof(gen_exp)} bytes")

# Output:
# List size: 8448728 bytes
# Generator size: 128 bytes (much smaller!)
```

### Practical Examples

```python
# Sum of squares
total = sum(x**2 for x in range(100))
print(total)  # 328350

# Filter with condition
even_squares = (x**2 for x in range(20) if x % 2 == 0)
print(list(even_squares))  # [0, 4, 16, 36, 64, 100, 144, 196, 256, 324]

# Chaining operations
data = (x for x in range(10))
doubled = (x * 2 for x in data)
filtered = (x for x in doubled if x > 5)
print(list(filtered))  # [6, 8, 10, 12, 14, 16, 18]
```

---

## Lazy Evaluation and Memory Efficiency

Generators use **lazy evaluation** - values are computed only when needed, not all at once.

### Memory Comparison

```python
# Non-lazy: Creates entire list in memory
def get_squares_list(n):
    return [x**2 for x in range(n)]

# Lazy: Generates values on demand
def get_squares_generator(n):
    for x in range(n):
        yield x**2

# For large n, generator uses constant memory
large_n = 1000000

# This creates a huge list
# squares_list = get_squares_list(large_n)  # Uses ~8MB

# This uses minimal memory
squares_gen = get_squares_generator(large_n)  # Uses ~128 bytes

# Process one at a time
for i, square in enumerate(squares_gen):
    if i >= 5:
        break
    print(square)
```

### Processing Large Files

```python
def read_large_file(file_path):
    """Generator for reading large files line by line"""
    with open(file_path, 'r') as file:
        for line in file:
            yield line.strip()

# Memory efficient - only one line in memory at a time
def process_file(file_path):
    for line in read_large_file(file_path):
        # Process each line
        if line.startswith('ERROR'):
            print(line)

# vs loading entire file (bad for large files):
# with open(file_path) as f:
#     lines = f.readlines()  # Entire file in memory!
```

### Pipeline Pattern

```python
def read_data():
    """Simulate reading data"""
    for i in range(100):
        yield i

def filter_even(numbers):
    """Filter even numbers"""
    for num in numbers:
        if num % 2 == 0:
            yield num

def square(numbers):
    """Square the numbers"""
    for num in numbers:
        yield num ** 2

def take(n, iterable):
    """Take first n items"""
    for i, item in enumerate(iterable):
        if i >= n:
            break
        yield item

# Chain generators together
pipeline = take(5, square(filter_even(read_data())))
print(list(pipeline))
# Output: [0, 4, 16, 36, 64]

# Only computes what's needed!
```

---

## Advanced Generator Techniques

### `yield from` (Delegating to Sub-generators)

```python
def generator1():
    yield 1
    yield 2

def generator2():
    yield 3
    yield 4

def combined():
    # Without yield from
    for value in generator1():
        yield value
    for value in generator2():
        yield value

def combined_better():
    # With yield from (cleaner)
    yield from generator1()
    yield from generator2()

print(list(combined_better()))
# Output: [1, 2, 3, 4]
```

### Flattening Nested Structures

```python
def flatten(nested_list):
    """Recursively flatten a nested list"""
    for item in nested_list:
        if isinstance(item, list):
            yield from flatten(item)
        else:
            yield item

nested = [1, [2, 3, [4, 5]], 6, [7, [8, 9]]]
print(list(flatten(nested)))
# Output: [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

### Generator Methods: `send()`, `throw()`, `close()`

```python
def echo_generator():
    value = None
    while True:
        # Receive value via send()
        received = yield value
        if received is not None:
            value = received

gen = echo_generator()
next(gen)  # Prime the generator

print(gen.send(10))  # 10
print(gen.send(20))  # 20
print(gen.send(30))  # 30

gen.close()  # Close the generator
```

### Two-Way Communication

```python
def running_average():
    total = 0
    count = 0
    average = None
    
    while True:
        value = yield average
        if value is not None:
            total += value
            count += 1
            average = total / count

avg = running_average()
next(avg)  # Prime it

print(avg.send(10))  # 10.0
print(avg.send(20))  # 15.0
print(avg.send(30))  # 20.0
```

### Generator Exception Handling

```python
def resilient_generator():
    try:
        for i in range(10):
            yield i
    except GeneratorExit:
        print("Generator is being closed!")
    except Exception as e:
        print(f"Caught exception: {e}")
        yield -1  # Recovery value

gen = resilient_generator()
print(next(gen))  # 0
print(next(gen))  # 1

gen.throw(ValueError("Something went wrong"))
# Output:
# Caught exception: Something went wrong
# Returns: -1
```

---

## Practical Examples

### Custom Range with Step

```python
def custom_range(start, end, step=1):
    current = start
    while current < end:
        yield current
        current += step

for num in custom_range(0, 10, 2):
    print(num, end=' ')
# Output: 0 2 4 6 8
```

### Reading CSV Files

```python
def read_csv_rows(filename):
    """Generator for reading CSV files row by row"""
    with open(filename, 'r') as file:
        # Skip header
        next(file)
        
        for line in file:
            # Parse CSV line
            row = line.strip().split(',')
            yield row

# Memory efficient processing
for row in read_csv_rows('data.csv'):
    # Process one row at a time
    print(row)
```

### Batch Processing

```python
def batch_generator(iterable, batch_size):
    """Yield items in batches"""
    batch = []
    for item in iterable:
        batch.append(item)
        if len(batch) == batch_size:
            yield batch
            batch = []
    
    # Yield remaining items
    if batch:
        yield batch

# Usage
data = range(25)
for batch in batch_generator(data, 10):
    print(batch)

# Output:
# [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
# [10, 11, 12, 13, 14, 15, 16, 17, 18, 19]
# [20, 21, 22, 23, 24]
```

### Sliding Window

```python
def sliding_window(iterable, window_size):
    """Generate sliding windows over an iterable"""
    from collections import deque
    
    window = deque(maxlen=window_size)
    
    for item in iterable:
        window.append(item)
        if len(window) == window_size:
            yield list(window)

# Usage
for window in sliding_window([1, 2, 3, 4, 5, 6], 3):
    print(window)

# Output:
# [1, 2, 3]
# [2, 3, 4]
# [3, 4, 5]
# [4, 5, 6]
```

### Infinite Cycle

```python
def cycle(iterable):
    """Cycle through an iterable indefinitely"""
    while True:
        for item in iterable:
            yield item

# Usage with limit
counter = 0
for item in cycle(['A', 'B', 'C']):
    print(item, end=' ')
    counter += 1
    if counter >= 10:
        break

# Output: A B C A B C A B C A
```

---

## Iterators vs Generators

| Feature | Iterator (Class) | Generator (Function) |
|---------|-----------------|---------------------|
| Syntax | Requires class with `__iter__` and `__next__` | Uses `yield` keyword |
| Complexity | More verbose | More concise |
| State Management | Manual (instance variables) | Automatic |
| Memory | Can be optimized | Automatically efficient |
| Reusability | Can be designed to be reusable | Single-use by default |
| Performance | Slightly faster | Minimal overhead |
| Use Case | Complex iteration logic | Simple to moderate logic |

### When to Use Each

**Use Iterators (Classes) when:**
- You need complex state management
- You want reusable iteration
- You need fine-grained control
- You're building a data structure

**Use Generators (Functions) when:**
- You want simple, clean code
- Memory efficiency is important
- You're processing large datasets
- You're creating one-time iterables

---

## Key Takeaways

- **Iterators** implement `__iter__()` and `__next__()` for custom iteration
- **Generators** are functions that use `yield` to produce values lazily
- **Generator expressions** are memory-efficient alternatives to list comprehensions
- **Lazy evaluation** means values are computed only when needed, saving memory
- Generators are excellent for processing large datasets and infinite sequences
- Use `yield from` to delegate to sub-generators
- Generators can communicate bidirectionally using `send()`, `throw()`, and `close()`
- Pipeline patterns with generators enable elegant data processing
- Choose generators for simplicity and memory efficiency, iterators for complex control

Mastering iterators and generators is essential for writing efficient, Pythonic code that handles data elegantly!