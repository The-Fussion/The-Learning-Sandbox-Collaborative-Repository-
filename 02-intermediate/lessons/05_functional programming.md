# Python Functional Programming: A Detailed Guide

## Table of Contents
1. [Introduction to Functional Programming](#introduction-to-functional-programming)
2. [Map, Filter, and Reduce](#map-filter-and-reduce)
3. [Lambda Functions](#lambda-functions)
4. [Closures](#closures)
5. [Partial Functions](#partial-functions)
6. [The `functools` Module](#the-functools-module)
7. [The `itertools` Module](#the-itertools-module)
8. [Writing Declarative Code](#writing-declarative-code)
9. [Function Composition](#function-composition)
10. [Practical Examples](#practical-examples)

---

## Introduction to Functional Programming

Functional programming (FP) is a programming paradigm that treats computation as the evaluation of mathematical functions and avoids changing state and mutable data.

### Key Principles

**Pure Functions**: Functions that always return the same output for the same input and have no side effects.

```python
# Pure function
def add(a, b):
    return a + b

# Impure function (has side effects)
total = 0
def add_to_total(x):
    global total
    total += x  # Modifies external state
    return total
```

**Immutability**: Data cannot be changed after creation.

```python
# Immutable approach
def add_item(items, new_item):
    return items + [new_item]  # Returns new list

original = [1, 2, 3]
new_list = add_item(original, 4)
print(original)  # [1, 2, 3] - unchanged
print(new_list)  # [1, 2, 3, 4] - new list
```

**First-Class Functions**: Functions can be assigned to variables, passed as arguments, and returned from other functions.

```python
def greet(name):
    return f"Hello, {name}!"

# Assign to variable
say_hello = greet
print(say_hello("Alice"))  # Hello, Alice!

# Pass as argument
def execute_function(func, value):
    return func(value)

result = execute_function(greet, "Bob")
print(result)  # Hello, Bob!
```

---

## Map, Filter, and Reduce

### Map

`map()` applies a function to every item in an iterable.

```python
# Basic map
numbers = [1, 2, 3, 4, 5]
squared = map(lambda x: x ** 2, numbers)
print(list(squared))  # [1, 4, 9, 16, 25]

# Map with named function
def double(x):
    return x * 2

doubled = map(double, numbers)
print(list(doubled))  # [2, 4, 6, 8, 10]

# Map with multiple iterables
list1 = [1, 2, 3]
list2 = [4, 5, 6]
result = map(lambda x, y: x + y, list1, list2)
print(list(result))  # [5, 7, 9]
```

### Filter

`filter()` filters items based on a condition (function returns True/False).

```python
# Basic filter
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
even = filter(lambda x: x % 2 == 0, numbers)
print(list(even))  # [2, 4, 6, 8, 10]

# Filter with named function
def is_positive(x):
    return x > 0

numbers = [-2, -1, 0, 1, 2, 3]
positive = filter(is_positive, numbers)
print(list(positive))  # [1, 2, 3]

# Filter strings by length
words = ["hi", "hello", "hey", "goodbye"]
long_words = filter(lambda w: len(w) > 3, words)
print(list(long_words))  # ['hello', 'goodbye']
```

### Reduce

`reduce()` applies a function cumulatively to items, reducing them to a single value.

```python
from functools import reduce

# Sum of numbers
numbers = [1, 2, 3, 4, 5]
total = reduce(lambda x, y: x + y, numbers)
print(total)  # 15

# Product of numbers
product = reduce(lambda x, y: x * y, numbers)
print(product)  # 120

# Find maximum
numbers = [3, 7, 2, 9, 1]
maximum = reduce(lambda x, y: x if x > y else y, numbers)
print(maximum)  # 9

# With initial value
numbers = [1, 2, 3]
result = reduce(lambda x, y: x + y, numbers, 10)
print(result)  # 16 (10 + 1 + 2 + 3)
```

### Chaining Map, Filter, and Reduce

```python
from functools import reduce

numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# Get sum of squares of even numbers
result = reduce(
    lambda x, y: x + y,
    map(
        lambda x: x ** 2,
        filter(lambda x: x % 2 == 0, numbers)
    )
)
print(result)  # 220 (4 + 16 + 36 + 64 + 100)

# More readable version
even_numbers = filter(lambda x: x % 2 == 0, numbers)
squared = map(lambda x: x ** 2, even_numbers)
total = reduce(lambda x, y: x + y, squared)
print(total)  # 220
```

---

## Lambda Functions

Lambda functions are anonymous, single-expression functions.

### Basic Lambda Syntax

```python
# Regular function
def add(x, y):
    return x + y

# Lambda equivalent
add_lambda = lambda x, y: x + y

print(add(5, 3))        # 8
print(add_lambda(5, 3)) # 8
```

### Lambda in Sorting

```python
# Sort by second element
pairs = [(1, 'one'), (3, 'three'), (2, 'two')]
sorted_pairs = sorted(pairs, key=lambda x: x[0])
print(sorted_pairs)  # [(1, 'one'), (2, 'two'), (3, 'three')]

# Sort strings by length
words = ["python", "is", "awesome"]
sorted_words = sorted(words, key=lambda w: len(w))
print(sorted_words)  # ['is', 'python', 'awesome']

# Sort dictionaries
students = [
    {'name': 'Alice', 'grade': 85},
    {'name': 'Bob', 'grade': 92},
    {'name': 'Charlie', 'grade': 78}
]
sorted_students = sorted(students, key=lambda s: s['grade'], reverse=True)
print(sorted_students)
# [{'name': 'Bob', 'grade': 92}, {'name': 'Alice', 'grade': 85}, {'name': 'Charlie', 'grade': 78}]
```

### When to Use Lambda

```python
# Good: Simple, one-time use
numbers = [1, 2, 3, 4, 5]
squared = map(lambda x: x ** 2, numbers)

# Bad: Complex logic (use regular function instead)
# Don't do this:
result = map(lambda x: x ** 2 if x > 0 else -x ** 2 if x < 0 else 0, numbers)

# Better: Named function
def complex_operation(x):
    if x > 0:
        return x ** 2
    elif x < 0:
        return -x ** 2
    else:
        return 0

result = map(complex_operation, numbers)
```

---

## Closures

A closure is a function that remembers values from its enclosing scope even after that scope has finished executing.

### Basic Closure

```python
def outer_function(x):
    def inner_function(y):
        return x + y  # inner_function "closes over" x
    return inner_function

add_five = outer_function(5)
add_ten = outer_function(10)

print(add_five(3))   # 8 (5 + 3)
print(add_ten(3))    # 13 (10 + 3)
```

### Closure as Function Factory

```python
def make_multiplier(n):
    def multiplier(x):
        return x * n
    return multiplier

times_two = make_multiplier(2)
times_three = make_multiplier(3)

print(times_two(5))    # 10
print(times_three(5))  # 15
```

### Closure with State

```python
def counter():
    count = 0
    
    def increment():
        nonlocal count  # Access outer variable
        count += 1
        return count
    
    return increment

counter1 = counter()
counter2 = counter()

print(counter1())  # 1
print(counter1())  # 2
print(counter1())  # 3

print(counter2())  # 1 (separate state)
print(counter2())  # 2
```

### Practical Closure Example

```python
def make_logger(prefix):
    def log(message):
        print(f"[{prefix}] {message}")
    return log

error_logger = make_logger("ERROR")
info_logger = make_logger("INFO")

error_logger("Something went wrong")  # [ERROR] Something went wrong
info_logger("Process started")        # [INFO] Process started
```

---

## Partial Functions

Partial functions allow you to fix some arguments of a function and create a new function.

### Using `functools.partial`

```python
from functools import partial

def power(base, exponent):
    return base ** exponent

# Create specialized functions
square = partial(power, exponent=2)
cube = partial(power, exponent=3)

print(square(5))  # 25
print(cube(5))    # 125
```

### Practical Examples

```python
from functools import partial

# Converting units
def convert_temperature(temp, from_unit, to_unit):
    if from_unit == "C" and to_unit == "F":
        return (temp * 9/5) + 32
    elif from_unit == "F" and to_unit == "C":
        return (temp - 32) * 5/9
    return temp

celsius_to_fahrenheit = partial(convert_temperature, from_unit="C", to_unit="F")
fahrenheit_to_celsius = partial(convert_temperature, from_unit="F", to_unit="C")

print(celsius_to_fahrenheit(0))    # 32.0
print(celsius_to_fahrenheit(100))  # 212.0
print(fahrenheit_to_celsius(32))   # 0.0
```

### Partial with Multiple Arguments

```python
from functools import partial

def greet(greeting, name, punctuation):
    return f"{greeting}, {name}{punctuation}"

# Fix greeting
hello = partial(greet, "Hello")
print(hello("Alice", "!"))  # Hello, Alice!

# Fix greeting and punctuation
friendly_hello = partial(greet, "Hello", punctuation="!")
print(friendly_hello("Bob"))  # Hello, Bob!
```

---

## The `functools` Module

### `functools.reduce`

Already covered above. Reduces an iterable to a single value.

### `functools.partial`

Already covered above. Creates partial functions.

### `functools.wraps`

Preserves function metadata when creating decorators.

```python
from functools import wraps

def my_decorator(func):
    @wraps(func)  # Preserves original function's metadata
    def wrapper(*args, **kwargs):
        print("Before function")
        result = func(*args, **kwargs)
        print("After function")
        return result
    return wrapper

@my_decorator
def greet(name):
    """Greets a person"""
    return f"Hello, {name}!"

print(greet.__name__)  # greet (not 'wrapper')
print(greet.__doc__)   # Greets a person
```

### `functools.lru_cache`

Memoization decorator for caching function results.

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

# First call computes
print(fibonacci(10))  # 55

# Subsequent calls use cache (much faster)
print(fibonacci(10))  # 55 (from cache)
print(fibonacci(20))  # 6765

# Check cache info
print(fibonacci.cache_info())
# CacheInfo(hits=18, misses=21, maxsize=128, currsize=21)
```

### `functools.cached_property`

Caches property value after first access.

```python
from functools import cached_property
import time

class DataProcessor:
    def __init__(self, data):
        self.data = data
    
    @cached_property
    def processed_data(self):
        print("Processing data (expensive operation)...")
        time.sleep(2)  # Simulate expensive operation
        return [x * 2 for x in self.data]

processor = DataProcessor([1, 2, 3, 4, 5])

# First access (takes 2 seconds)
print(processor.processed_data)  # [2, 4, 6, 8, 10]

# Subsequent accesses (instant)
print(processor.processed_data)  # [2, 4, 6, 8, 10] (from cache)
```

### `functools.singledispatch`

Creates generic functions with type-specific implementations.

```python
from functools import singledispatch

@singledispatch
def process(arg):
    print(f"Processing generic type: {arg}")

@process.register(int)
def _(arg):
    print(f"Processing integer: {arg * 2}")

@process.register(str)
def _(arg):
    print(f"Processing string: {arg.upper()}")

@process.register(list)
def _(arg):
    print(f"Processing list: {len(arg)} items")

process(42)           # Processing integer: 84
process("hello")      # Processing string: HELLO
process([1, 2, 3])    # Processing list: 3 items
process(3.14)         # Processing generic type: 3.14
```

---

## The `itertools` Module

`itertools` provides fast, memory-efficient tools for working with iterators.

### Infinite Iterators

```python
from itertools import count, cycle, repeat

# count: infinite counter
counter = count(start=10, step=2)
print(next(counter))  # 10
print(next(counter))  # 12
print(next(counter))  # 14

# cycle: infinite cycle through iterable
colors = cycle(['red', 'green', 'blue'])
print(next(colors))  # red
print(next(colors))  # green
print(next(colors))  # blue
print(next(colors))  # red (cycles back)

# repeat: repeat value n times (or infinitely)
repeated = repeat('Hello', 3)
print(list(repeated))  # ['Hello', 'Hello', 'Hello']
```

### Combinatoric Iterators

```python
from itertools import product, permutations, combinations, combinations_with_replacement

# product: Cartesian product
result = product([1, 2], ['a', 'b'])
print(list(result))  # [(1, 'a'), (1, 'b'), (2, 'a'), (2, 'b')]

# permutations: all orderings
result = permutations([1, 2, 3], 2)
print(list(result))  # [(1, 2), (1, 3), (2, 1), (2, 3), (3, 1), (3, 2)]

# combinations: unordered selections
result = combinations([1, 2, 3, 4], 2)
print(list(result))  # [(1, 2), (1, 3), (1, 4), (2, 3), (2, 4), (3, 4)]

# combinations_with_replacement: combinations allowing repeats
result = combinations_with_replacement([1, 2, 3], 2)
print(list(result))  # [(1, 1), (1, 2), (1, 3), (2, 2), (2, 3), (3, 3)]
```

### Terminating Iterators

```python
from itertools import chain, islice, accumulate, groupby, compress, dropwhile, takewhile

# chain: combine multiple iterables
result = chain([1, 2, 3], ['a', 'b'], [4, 5])
print(list(result))  # [1, 2, 3, 'a', 'b', 4, 5]

# islice: slice an iterator
numbers = range(100)
result = islice(numbers, 5, 10)  # Elements from index 5 to 9
print(list(result))  # [5, 6, 7, 8, 9]

# accumulate: running totals
result = accumulate([1, 2, 3, 4, 5])
print(list(result))  # [1, 3, 6, 10, 15]

# accumulate with custom function
result = accumulate([1, 2, 3, 4, 5], lambda x, y: x * y)
print(list(result))  # [1, 2, 6, 24, 120]

# groupby: group consecutive elements
data = [1, 1, 2, 2, 2, 3, 3, 1, 1]
result = groupby(data)
for key, group in result:
    print(f"{key}: {list(group)}")
# 1: [1, 1]
# 2: [2, 2, 2]
# 3: [3, 3]
# 1: [1, 1]

# compress: filter by boolean mask
data = ['a', 'b', 'c', 'd', 'e']
selectors = [1, 0, 1, 0, 1]
result = compress(data, selectors)
print(list(result))  # ['a', 'c', 'e']

# dropwhile: drop elements while predicate is true
result = dropwhile(lambda x: x < 5, [1, 3, 5, 7, 2, 4])
print(list(result))  # [5, 7, 2, 4]

# takewhile: take elements while predicate is true
result = takewhile(lambda x: x < 5, [1, 3, 5, 7, 2, 4])
print(list(result))  # [1, 3]
```

### Practical Examples

```python
from itertools import chain, groupby, islice

# Flatten nested lists
nested = [[1, 2], [3, 4], [5, 6]]
flattened = chain.from_iterable(nested)
print(list(flattened))  # [1, 2, 3, 4, 5, 6]

# Group and count
words = ['apple', 'apricot', 'banana', 'blueberry', 'cherry']
sorted_words = sorted(words, key=lambda w: w[0])
for letter, group in groupby(sorted_words, key=lambda w: w[0]):
    print(f"{letter}: {list(group)}")
# a: ['apple', 'apricot']
# b: ['banana', 'blueberry']
# c: ['cherry']

# Pagination
def paginate(iterable, page_size):
    iterator = iter(iterable)
    while True:
        page = list(islice(iterator, page_size))
        if not page:
            break
        yield page

data = range(25)
for page_num, page in enumerate(paginate(data, 10), 1):
    print(f"Page {page_num}: {page}")
# Page 1: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
# Page 2: [10, 11, 12, 13, 14, 15, 16, 17, 18, 19]
# Page 3: [20, 21, 22, 23, 24]
```

---

## Writing Declarative Code

Declarative code focuses on **what** to do rather than **how** to do it.

### Imperative vs Declarative

```python
# Imperative (how to do it)
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
result = []
for num in numbers:
    if num % 2 == 0:
        result.append(num ** 2)
print(result)

# Declarative (what to do)
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
result = [num ** 2 for num in numbers if num % 2 == 0]
print(result)

# Functional (even more declarative)
from functools import reduce
result = list(map(lambda x: x ** 2, filter(lambda x: x % 2 == 0, numbers)))
print(result)
```

### Pipeline Pattern

```python
from functools import reduce

def pipe(*functions):
    """Create a pipeline of functions"""
    def pipeline(value):
        return reduce(lambda v, f: f(v), functions, value)
    return pipeline

# Define transformations
def double(x):
    return x * 2

def add_ten(x):
    return x + 10

def square(x):
    return x ** 2

# Create pipeline
transform = pipe(double, add_ten, square)

print(transform(5))  # ((5 * 2) + 10) ** 2 = 400
```

### Method Chaining

```python
class DataPipeline:
    def __init__(self, data):
        self.data = data
    
    def filter(self, predicate):
        self.data = list(filter(predicate, self.data))
        return self
    
    def map(self, transformer):
        self.data = list(map(transformer, self.data))
        return self
    
    def reduce(self, reducer, initial=None):
        from functools import reduce
        if initial is None:
            return reduce(reducer, self.data)
        return reduce(reducer, self.data, initial)
    
    def get(self):
        return self.data

# Usage
result = (DataPipeline([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])
    .filter(lambda x: x % 2 == 0)
    .map(lambda x: x ** 2)
    .get())

print(result)  # [4, 16, 36, 64, 100]
```

---

## Function Composition

Combining simple functions to create more complex ones.

### Manual Composition

```python
def compose(f, g):
    """Compose two functions: (f ∘ g)(x) = f(g(x))"""
    return lambda x: f(g(x))

def add_five(x):
    return x + 5

def double(x):
    return x * 2

# Compose functions
add_five_then_double = compose(double, add_five)
print(add_five_then_double(10))  # (10 + 5) * 2 = 30

double_then_add_five = compose(add_five, double)
print(double_then_add_five(10))  # (10 * 2) + 5 = 25
```

### Multiple Function Composition

```python
from functools import reduce

def compose(*functions):
    """Compose multiple functions"""
    return reduce(lambda f, g: lambda x: f(g(x)), functions, lambda x: x)

def add_ten(x):
    return x + 10

def double(x):
    return x * 2

def square(x):
    return x ** 2

# Compose from right to left
transform = compose(square, double, add_ten)
print(transform(5))  # ((5 + 10) * 2) ** 2 = 900
```

---

## Practical Examples

### Data Processing Pipeline

```python
from functools import reduce
from itertools import groupby

# Sample data
sales = [
    {'product': 'A', 'quantity': 10, 'price': 5},
    {'product': 'B', 'quantity': 5, 'price': 10},
    {'product': 'A', 'quantity': 15, 'price': 5},
    {'product': 'C', 'quantity': 8, 'price': 7},
    {'product': 'B', 'quantity': 3, 'price': 10},
]

# Calculate total revenue per product
def calculate_revenue(sale):
    return {**sale, 'revenue': sale['quantity'] * sale['price']}

def group_by_product(sales):
    sorted_sales = sorted(sales, key=lambda x: x['product'])
    return groupby(sorted_sales, key=lambda x: x['product'])

# Process data
sales_with_revenue = list(map(calculate_revenue, sales))
grouped = group_by_product(sales_with_revenue)

for product, group in grouped:
    total_revenue = reduce(
        lambda acc, sale: acc + sale['revenue'],
        group,
        0
    )
    print(f"Product {product}: ${total_revenue}")

# Output:
# Product A: $125
# Product B: $80
# Product C: $56
```

### Text Processing

```python
from functools import reduce

text = """
Python is awesome.
Functional programming is powerful.
Python makes coding fun!
"""

# Process text functionally
words = (text.lower()
    .replace('.', '')
    .replace('!', '')
    .split())

# Count word frequency
word_count = reduce(
    lambda counts, word: {**counts, word: counts.get(word, 0) + 1},
    words,
    {}
)

# Get top 3 most common words
top_words = sorted(word_count.items(), key=lambda x: x[1], reverse=True)[:3]
print(top_words)
# [('python', 2), ('is', 2), ('awesome', 1)]
```

### Validation Pipeline

```python
from functools import reduce

def validate_email(email):
    """Validation functions"""
    validators = [
        lambda e: (e, None) if '@' in e else (e, "Missing @"),
        lambda e: (e[0], None) if '.' in e[0].split('@')[1] else (e[0], "Missing domain extension"),
        lambda e: (e[0], None) if len(e[0]) > 5 else (e[0], "Email too short"),
    ]
    
    result = reduce(
        lambda acc, validator: validator(acc) if acc[1] is None else acc,
        validators,
        (email, None)
    )
    
    return result[1] is None, result[1]

# Test
emails = ["user@example.com", "invalid", "a@b.c", "user@domain"]

for email in emails:
    is_valid, error = validate_email(email)
    print(f"{email}: {'Valid' if is_valid else f'Invalid - {error}'}")

# Output:
# user@example.com: Valid
# invalid: Invalid - Missing @
# a@b.c: Invalid - Email too short
# user@domain: Invalid - Missing domain extension
```

---

## Key Takeaways

- **Map, filter, reduce** are fundamental functional operations for transforming data
- **Lambda functions** provide concise syntax for simple, one-time functions
- **Closures** enable functions to remember their enclosing scope
- **Partial functions** create specialized versions of existing functions
- **functools** provides powerful utilities like `lru_cache`, `wraps`, and `singledispatch`
- **itertools** offers memory-efficient tools for iteration and combinatorics
- **Declarative code** focuses on what to do rather than how to do it
- **Function composition** combines simple functions to build complex behaviors
- Functional programming promotes **immutability**, **pure functions**, and **code reusability**
- Use functional patterns when appropriate, but don't force them everywhere

Mastering functional programming concepts makes your Python code more elegant, testable, and maintainable!