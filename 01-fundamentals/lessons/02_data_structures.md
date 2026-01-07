# Data Structures in Python: A Comprehensive Guide for Beginners

## Table of Contents
1. [Introduction to Data Structures](#introduction-to-data-structures)
2. [Lists: The Versatile Workhorse](#lists-the-versatile-workhorse)
3. [Tuples: Immutable Sequences](#tuples-immutable-sequences)
4. [Dictionaries: Key-Value Powerhouses](#dictionaries-key-value-powerhouses)
5. [Sets: Unique Collections](#sets-unique-collections)
6. [Strings: Immutable Character Sequences](#strings-immutable-character-sequences)
7. [Advanced: Stacks and Queues](#advanced-stacks-and-queues)
8. [Choosing the Right Data Structure](#choosing-the-right-data-structure)
9. [Performance Considerations](#performance-considerations)
10. [Practice Exercises](#practice-exercises)

---

## Introduction to Data Structures

Data structures are specialized formats for organizing, storing, and managing data in a way that enables efficient access and modification. Think of them as different types of containers, each designed for specific purposes—just like you wouldn't store water in a basket, you wouldn't use certain data structures for specific tasks.

### Why Data Structures Matter

Before diving into the technical details, understand this: choosing the right data structure is like choosing the right tool for a job. Use a hammer when you need a hammer, not a screwdriver. The right data structure can make your code faster, cleaner, and more elegant.

---

## Lists: The Versatile Workhorse

Lists are ordered, mutable collections that can hold items of any type. They're the most flexible data structure in Python.

### Creating Lists
```python
# Empty list
empty_list = []

# List with items
fruits = ["apple", "banana", "cherry"]

# Mixed data types
mixed = [1, "hello", 3.14, True]

# Nested lists
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
```

### Accessing Elements
```python
fruits = ["apple", "banana", "cherry", "date"]

# Indexing (starts at 0)
print(fruits[0])  # "apple"
print(fruits[-1])  # "date" (negative indexing from end)

# Slicing
print(fruits[1:3])  # ["banana", "cherry"]
print(fruits[:2])   # ["apple", "banana"]
print(fruits[2:])   # ["cherry", "date"]
```

### Key Concepts: Indexing

**Positive Indexing**: Starts from 0 at the beginning
```
["apple", "banana", "cherry", "date"]
   0        1         2        3
```

**Negative Indexing**: Starts from -1 at the end
```
["apple", "banana", "cherry", "date"]
   -4       -3        -2       -1
```

### Modifying Lists
```python
fruits = ["apple", "banana", "cherry"]

# Adding elements
fruits.append("date")           # Add to end
fruits.insert(1, "blueberry")   # Insert at index
fruits.extend(["elderberry", "fig"])  # Add multiple items

# Removing elements
fruits.remove("banana")  # Remove by value
popped = fruits.pop()    # Remove and return last item
popped_index = fruits.pop(0)  # Remove and return item at index
del fruits[1]            # Delete by index

# Modifying elements
fruits[0] = "apricot"
```

### Common List Operations
```python
numbers = [3, 1, 4, 1, 5, 9, 2, 6]

# Length
print(len(numbers))  # 8

# Sorting
numbers.sort()           # Sort in place
sorted_nums = sorted(numbers)  # Return new sorted list

# Reversing
numbers.reverse()        # Reverse in place
reversed_nums = numbers[::-1]  # Return new reversed list

# Finding items
index = numbers.index(5)  # Find index of value
count = numbers.count(1)  # Count occurrences

# Checking membership
if 4 in numbers:
    print("Found!")
```

### List Methods Reference

| Method | Description | Example |
|--------|-------------|---------|
| `append(x)` | Add item to end | `fruits.append("orange")` |
| `insert(i, x)` | Insert item at index | `fruits.insert(0, "kiwi")` |
| `remove(x)` | Remove first occurrence | `fruits.remove("apple")` |
| `pop([i])` | Remove and return item | `fruits.pop()` |
| `clear()` | Remove all items | `fruits.clear()` |
| `index(x)` | Find index of value | `fruits.index("banana")` |
| `count(x)` | Count occurrences | `fruits.count("apple")` |
| `sort()` | Sort list in place | `numbers.sort()` |
| `reverse()` | Reverse list in place | `fruits.reverse()` |
| `copy()` | Create shallow copy | `new_list = fruits.copy()` |
| `extend(iterable)` | Add multiple items | `fruits.extend(["grape", "melon"])` |

### List Comprehensions

A powerful way to create lists concisely.

**Basic Syntax**:
```python
[expression for item in iterable if condition]
```

**Examples**:
```python
# Create list of squares
squares = [x**2 for x in range(10)]
# Result: [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]

# Filter even numbers
evens = [x for x in range(20) if x % 2 == 0]
# Result: [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]

# Transform strings
names = ["alice", "bob", "charlie"]
capitalized = [name.upper() for name in names]
# Result: ["ALICE", "BOB", "CHARLIE"]

# Nested comprehensions
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flattened = [num for row in matrix for num in row]
# Result: [1, 2, 3, 4, 5, 6, 7, 8, 9]

# Conditional expression
numbers = [1, 2, 3, 4, 5, 6]
labels = ["even" if x % 2 == 0 else "odd" for x in numbers]
# Result: ["odd", "even", "odd", "even", "odd", "even"]
```

### Advanced List Operations

#### List Slicing with Step
```python
numbers = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

# Every second element
evens = numbers[::2]  # [0, 2, 4, 6, 8]

# Every second element starting from index 1
odds = numbers[1::2]  # [1, 3, 5, 7, 9]

# Reverse using negative step
reversed_nums = numbers[::-1]  # [9, 8, 7, 6, 5, 4, 3, 2, 1, 0]

# Every third element
thirds = numbers[::3]  # [0, 3, 6, 9]
```

#### Copying Lists
```python
original = [1, 2, 3, 4, 5]

# Shallow copy methods
copy1 = original[:]
copy2 = original.copy()
copy3 = list(original)

# Deep copy (for nested lists)
import copy
nested = [[1, 2], [3, 4]]
deep_copy = copy.deepcopy(nested)
```

#### List Unpacking
```python
# Basic unpacking
a, b, c = [1, 2, 3]

# Using * for remaining elements
first, *rest = [1, 2, 3, 4, 5]
# first = 1, rest = [2, 3, 4, 5]

*start, last = [1, 2, 3, 4, 5]
# start = [1, 2, 3, 4], last = 5

first, *middle, last = [1, 2, 3, 4, 5]
# first = 1, middle = [2, 3, 4], last = 5
```

---

## Tuples: Immutable Sequences

Tuples are like lists but immutable—once created, they cannot be modified. Use them for data that shouldn't change.

### Creating Tuples
```python
# Empty tuple
empty = ()

# Single element (note the comma)
single = (5,)

# Multiple elements
coordinates = (10, 20)
person = ("Alice", 25, "Engineer")

# Without parentheses (tuple packing)
point = 3, 4

# Using tuple() constructor
from_list = tuple([1, 2, 3, 4, 5])
```

### Accessing Tuple Elements
```python
person = ("Bob", 30, "Designer", "New York")

# Indexing
name = person[0]  # "Bob"
city = person[-1]  # "New York"

# Slicing
info = person[1:3]  # (30, "Designer")

# Unpacking
name, age, job, city = person

# Partial unpacking with *
first, *middle, last = (1, 2, 3, 4, 5)
# first = 1, middle = [2, 3, 4], last = 5
```

### Tuple Methods

Tuples have only two built-in methods because they're immutable:
```python
numbers = (1, 2, 3, 2, 4, 2, 5)

# Count occurrences
count = numbers.count(2)  # 3

# Find index of first occurrence
index = numbers.index(4)  # 4

# Find index with start and end
index = numbers.index(2, 2)  # Find 2 starting from index 2
```

### Why Use Tuples?

#### 1. Return Multiple Values from Functions
```python
def get_coordinates():
    x = 10
    y = 20
    return x, y  # Returns a tuple

# Unpack the returned tuple
latitude, longitude = get_coordinates()
```

#### 2. Dictionary Keys (Lists Can't Be Keys)
```python
# Tuples as dictionary keys
locations = {
    (0, 0): "origin",
    (1, 0): "right",
    (0, 1): "up",
    (1, 1): "diagonal"
}

# Access using tuple key
print(locations[(0, 0)])  # "origin"
```

#### 3. Data Integrity - Prevent Accidental Modification
```python
# Constants that should never change
RGB_RED = (255, 0, 0)
RGB_GREEN = (0, 255, 0)
RGB_BLUE = (0, 0, 255)

# Configuration that should remain constant
DATABASE_CONFIG = ("localhost", 5432, "mydb", "user")
```

#### 4. Memory Efficiency
```python
# Tuples use less memory than lists
import sys

list_data = [1, 2, 3, 4, 5]
tuple_data = (1, 2, 3, 4, 5)

print(sys.getsizeof(list_data))   # More bytes
print(sys.getsizeof(tuple_data))  # Fewer bytes
```

### Named Tuples

For more readable code, use named tuples from the `collections` module:
```python
from collections import namedtuple

# Define a named tuple
Point = namedtuple('Point', ['x', 'y'])
Person = namedtuple('Person', ['name', 'age', 'city'])

# Create instances
p = Point(10, 20)
person = Person('Alice', 25, 'Lagos')

# Access by name
print(p.x)  # 10
print(person.name)  # Alice

# Access by index (still works)
print(p[0])  # 10

# Convert to dictionary
person_dict = person._asdict()
```

### Tuple Operations
```python
# Concatenation
tuple1 = (1, 2, 3)
tuple2 = (4, 5, 6)
combined = tuple1 + tuple2  # (1, 2, 3, 4, 5, 6)

# Repetition
repeated = (1, 2) * 3  # (1, 2, 1, 2, 1, 2)

# Membership
if 2 in (1, 2, 3):
    print("Found!")

# Length
length = len((1, 2, 3, 4, 5))  # 5

# Min and max
numbers = (5, 2, 8, 1, 9)
minimum = min(numbers)  # 1
maximum = max(numbers)  # 9

# Sum
total = sum(numbers)  # 25
```

---

## Dictionaries: Key-Value Powerhouses

Dictionaries store data as key-value pairs, providing fast lookup by key. Think of them as real-world dictionaries where you look up a word (key) to find its definition (value).

### Creating Dictionaries
```python
# Empty dictionary
empty = {}
empty = dict()

# With initial data
person = {
    "name": "Alice",
    "age": 25,
    "city": "Lagos"
}

# Using dict() constructor
person = dict(name="Alice", age=25, city="Lagos")

# From list of tuples
pairs = [("a", 1), ("b", 2), ("c", 3)]
letters = dict(pairs)

# Using dict comprehension
squares = {x: x**2 for x in range(5)}
# Result: {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}

# Using zip to combine lists
keys = ["name", "age", "city"]
values = ["Bob", 30, "Abuja"]
person = dict(zip(keys, values))
```

### Accessing Dictionary Elements
```python
person = {"name": "Bob", "age": 30, "city": "Abuja"}

# Access by key
name = person["name"]  # "Bob"
# KeyError if key doesn't exist

# Safe access with get()
age = person.get("age")  # 30
country = person.get("country", "Nigeria")  # "Nigeria" (default value)

# Check if key exists
if "email" in person:
    print(person["email"])

# Check if key doesn't exist
if "phone" not in person:
    print("Phone number not found")
```

### Modifying Dictionaries
```python
person = {"name": "Charlie", "age": 35}

# Add new key-value pair
person["city"] = "Port Harcourt"

# Update existing value
person["age"] = 36

# Update multiple items
person.update({"job": "Engineer", "salary": 100000})
person.update([("country", "Nigeria"), ("state", "Rivers")])

# Remove items
age = person.pop("age")  # Remove and return value
age = person.pop("age", None)  # Return None if key doesn't exist

last_item = person.popitem()  # Remove and return last key-value pair

del person["city"]  # Delete by key

person.clear()  # Remove all items
```

### Dictionary Methods and Operations
```python
person = {"name": "Diana", "age": 28, "city": "Kano"}

# Get all keys
keys = person.keys()  # dict_keys(['name', 'age', 'city'])
keys_list = list(person.keys())  # Convert to list

# Get all values
values = person.values()  # dict_values(['Diana', 28, 'Kano'])
values_list = list(person.values())

# Get all key-value pairs
items = person.items()  # dict_items([('name', 'Diana'), ('age', 28), ('city', 'Kano')])

# Iterate over keys
for key in person:
    print(key, person[key])

# Iterate over values
for value in person.values():
    print(value)

# Iterate over key-value pairs
for key, value in person.items():
    print(f"{key}: {value}")

# Copy dictionary
person_copy = person.copy()

# Get with default if key doesn't exist
email = person.setdefault("email", "default@email.com")
# If "email" exists, return its value
# If not, set it to "default@email.com" and return that
```

### Dictionary Methods Reference

| Method | Description | Example |
|--------|-------------|---------|
| `get(key, default)` | Get value or default | `age = person.get("age", 0)` |
| `keys()` | Get all keys | `keys = person.keys()` |
| `values()` | Get all values | `values = person.values()` |
| `items()` | Get all key-value pairs | `items = person.items()` |
| `pop(key, default)` | Remove and return value | `age = person.pop("age")` |
| `popitem()` | Remove and return last item | `item = person.popitem()` |
| `update(dict)` | Update with another dict | `person.update({"age": 30})` |
| `clear()` | Remove all items | `person.clear()` |
| `copy()` | Create shallow copy | `new = person.copy()` |
| `setdefault(key, default)` | Get or set default | `email = person.setdefault("email", "")` |
| `fromkeys(seq, value)` | Create dict from sequence | `dict.fromkeys(['a','b'], 0)` |

### Dictionary Comprehensions
```python
# Basic dictionary comprehension
squares = {x: x**2 for x in range(6)}
# Result: {0: 0, 1: 1, 2: 4, 3: 9, 4: 16, 5: 25}

# Transform dictionary
prices = {"apple": 0.50, "banana": 0.30, "cherry": 1.00}
doubled = {fruit: price * 2 for fruit, price in prices.items()}

# Filter dictionary
expensive = {fruit: price for fruit, price in prices.items() if price > 0.40}

# Swap keys and values
original = {"a": 1, "b": 2, "c": 3}
swapped = {value: key for key, value in original.items()}

# Conditional values
numbers = [1, 2, 3, 4, 5]
parity = {num: "even" if num % 2 == 0 else "odd" for num in numbers}
```

### Nested Dictionaries
```python
# Company database
company = {
    "employees": {
        "001": {
            "name": "Alice",
            "role": "CEO",
            "salary": 200000
        },
        "002": {
            "name": "Bob",
            "role": "CTO",
            "salary": 180000
        }
    },
    "revenue": {
        "2023": 1000000,
        "2024": 1500000
    },
    "departments": {
        "engineering": ["001", "002"],
        "sales": ["003", "004"]
    }
}

# Access nested values
ceo_name = company["employees"]["001"]["name"]
revenue_2024 = company["revenue"]["2024"]

# Safely access nested values
def get_nested(dictionary, *keys, default=None):
    for key in keys:
        if isinstance(dictionary, dict):
            dictionary = dictionary.get(key, default)
        else:
            return default
    return dictionary

# Usage
cto_salary = get_nested(company, "employees", "002", "salary")
```

### Advanced Dictionary Patterns

#### Merging Dictionaries
```python
# Python 3.9+
dict1 = {"a": 1, "b": 2}
dict2 = {"b": 3, "c": 4}

# Using | operator
merged = dict1 | dict2  # {"a": 1, "b": 3, "c": 4}

# Using |= operator (in-place)
dict1 |= dict2

# Python 3.5+
# Using ** unpacking
merged = {**dict1, **dict2}

# Using update()
dict1.update(dict2)
```

#### Default Dictionaries
```python
from collections import defaultdict

# Regular dict
word_count = {}
for word in ["apple", "banana", "apple", "cherry"]:
    if word not in word_count:
        word_count[word] = 0
    word_count[word] += 1

# With defaultdict
word_count = defaultdict(int)
for word in ["apple", "banana", "apple", "cherry"]:
    word_count[word] += 1

# defaultdict with lists
groups = defaultdict(list)
groups["fruits"].append("apple")
groups["fruits"].append("banana")
```

#### Counter Dictionaries
```python
from collections import Counter

# Count elements
words = ["apple", "banana", "apple", "cherry", "banana", "apple"]
word_count = Counter(words)
# Counter({'apple': 3, 'banana': 2, 'cherry': 1})

# Most common elements
most_common = word_count.most_common(2)
# [('apple', 3), ('banana', 2)]

# Arithmetic with counters
counter1 = Counter(["a", "b", "c", "a"])
counter2 = Counter(["a", "c", "d", "c"])

combined = counter1 + counter2  # Add counts
difference = counter1 - counter2  # Subtract counts
```

---

## Sets: Unique Collections

Sets are unordered collections of unique elements. They're perfect for membership testing and eliminating duplicates.

### Creating Sets
```python
# Empty set (must use set(), not {})
empty = set()

# With initial data
fruits = {"apple", "banana", "cherry"}

# From list (removes duplicates automatically)
numbers = set([1, 2, 2, 3, 3, 3, 4])  # {1, 2, 3, 4}

# From string (unique characters)
letters = set("hello")  # {"h", "e", "l", "o"}

# Set comprehension
evens = {x for x in range(10) if x % 2 == 0}
# {0, 2, 4, 6, 8}

# Using set() constructor
from_tuple = set((1, 2, 3))
```

### Set Operations

#### Mathematical Set Operations
```python
a = {1, 2, 3, 4, 5}
b = {4, 5, 6, 7, 8}

# Union (all elements from both sets)
union = a | b  # {1, 2, 3, 4, 5, 6, 7, 8}
union = a.union(b)

# Intersection (elements in both sets)
intersection = a & b  # {4, 5}
intersection = a.intersection(b)

# Difference (elements in a but not in b)
difference = a - b  # {1, 2, 3}
difference = a.difference(b)

# Symmetric difference (elements in either set but not both)
sym_diff = a ^ b  # {1, 2, 3, 6, 7, 8}
sym_diff = a.symmetric_difference(b)

# Multiple sets
set1 = {1, 2, 3}
set2 = {2, 3, 4}
set3 = {3, 4, 5}

# Union of multiple sets
all_union = set1 | set2 | set3  # {1, 2, 3, 4, 5}
all_union = set1.union(set2, set3)

# Intersection of multiple sets
all_intersection = set1 & set2 & set3  # {3}
all_intersection = set1.intersection(set2, set3)
```

### Modifying Sets
```python
fruits = {"apple", "banana"}

# Adding elements
fruits.add("cherry")
# Only adds if not already present

# Adding multiple elements
fruits.update(["date", "elderberry"])
fruits.update({"fig", "grape"})

# Removing elements
fruits.remove("banana")  # Raises KeyError if not found
fruits.discard("fig")    # No error if not found
popped = fruits.pop()    # Remove and return arbitrary element

# Clear all elements
fruits.clear()

# In-place operations
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}

a |= b   # a = a.union(b)
a &= b   # a = a.intersection(b)
a -= b   # a = a.difference(b)
a ^= b   # a = a.symmetric_difference(b)
```

### Set Methods Reference

| Method | Description | Example |
|--------|-------------|---------|
| `add(elem)` | Add element | `s.add(5)` |
| `remove(elem)` | Remove element (error if missing) | `s.remove(5)` |
| `discard(elem)` | Remove element (no error) | `s.discard(5)` |
| `pop()` | Remove arbitrary element | `elem = s.pop()` |
| `clear()` | Remove all elements | `s.clear()` |
| `update(iterable)` | Add multiple elements | `s.update([1, 2, 3])` |
| `union(other)` | Return union | `s1.union(s2)` |
| `intersection(other)` | Return intersection | `s1.intersection(s2)` |
| `difference(other)` | Return difference | `s1.difference(s2)` |
| `symmetric_difference(other)` | Return symmetric diff | `s1.symmetric_difference(s2)` |
| `copy()` | Create shallow copy | `new_s = s.copy()` |

### Set Relationships
```python
a = {1, 2, 3, 4}
b = {2, 3}
c = {5, 6, 7}

# Subset (all elements of b are in a)
is_subset = b.issubset(a)  # True
is_subset = b <= a         # True

# Proper subset (subset but not equal)
is_proper_subset = b < a   # True

# Superset (a contains all elements of b)
is_superset = a.issuperset(b)  # True
is_superset = a >= b           # True

# Proper superset
is_proper_superset = a > b  # True

# Disjoint (no common elements)
is_disjoint = a.isdisjoint(c)  # True

# Equal sets
d = {1, 2, 3, 4}
are_equal = a == d  # True
```

### Practical Use Cases for Sets

#### Remove Duplicates
```python
# Remove duplicates from list
numbers = [1, 2, 2, 3, 3, 3, 4, 4, 4, 4]
unique = list(set(numbers))
# [1, 2, 3, 4] (order not guaranteed)

# Preserve order while removing duplicates
def remove_duplicates_ordered(items):
    seen = set()
    result = []
    for item in items:
        if item not in seen:
            seen.add(item)
            result.append(item)
    return result

numbers = [1, 2, 2, 3, 3, 3, 4, 4, 4, 4]
unique_ordered = remove_duplicates_ordered(numbers)
# [1, 2, 3, 4] (order preserved)
```

#### Find Unique Characters
```python
# Find unique characters in a string
text = "hello world"
unique_chars = set(text)
# {' ', 'd', 'e', 'h', 'l', 'o', 'r', 'w'}

# Count unique characters
unique_count = len(set(text))
```

#### Check for Common Elements
```python
list1 = [1, 2, 3, 4, 5]
list2 = [4, 5, 6, 7, 8]

# Check if lists have common elements
has_common = bool(set(list1) & set(list2))  # True

# Get common elements
common = list(set(list1) & set(list2))  # [4, 5]
```

#### Set-Based Filtering
```python
# Filter list based on another list
allowed_ids = {1, 3, 5, 7, 9}
all_ids = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

filtered = [id for id in all_ids if id in allowed_ids]
# [1, 3, 5, 7, 9]
```

#### Find Missing Elements
```python
# Find missing numbers in sequence
complete = set(range(1, 11))  # {1, 2, 3, ..., 10}
actual = {1, 2, 4, 5, 7, 8, 10}

missing = complete - actual  # {3, 6, 9}
```

### Frozen Sets

Immutable sets that can be used as dictionary keys or set elements:
```python
# Create frozen set
frozen = frozenset([1, 2, 3, 4, 5])

# Can't modify
# frozen.add(6)  # AttributeError

# Can be used as dictionary key
lookup = {
    frozenset([1, 2]): "pair1",
    frozenset([3, 4]): "pair2"
}

# Can be element of another set
set_of_sets = {frozenset([1, 2]), frozenset([3, 4])}
```

---

## Strings: Immutable Character Sequences

While strings are technically a data type, they behave like sequences and have powerful built-in methods.

### Creating and Accessing Strings
```python
# Different ways to create strings
single = 'Hello'
double = "World"
triple_single = '''Multi
line
string'''
triple_double = """Another
multi-line
string"""

# Raw strings (ignore escape characters)
path = r"C:\Users\name\folder"

# f-strings (formatted string literals)
name = "Alice"
age = 25
message = f"My name is {name} and I'm {age} years old"

# Indexing
text = "Python"
first = text[0]      # "P"
last = text[-1]      # "n"

# Slicing
substring = text[1:4]    # "yth"
start = text[:3]         # "Pyt"
end = text[3:]           # "hon"
reverse = text[::-1]     # "nohtyP"
```

### String Methods

#### Case Conversion
```python
text = "Hello, World!"

upper = text.upper()          # "HELLO, WORLD!"
lower = text.lower()          # "hello, world!"
title = text.title()          # "Hello, World!"
capitalize = text.capitalize() # "Hello, world!"
swapcase = text.swapcase()    # "hELLO, wORLD!"

# Case checking
is_upper = text.isupper()     # False
is_lower = text.islower()     # False
is_title = text.istitle()     # True
```

#### Whitespace Handling
```python
text = "  Hello, World!  "

stripped = text.strip()      # "Hello, World!"
lstripped = text.lstrip()    # "Hello, World!  "
rstripped = text.rstrip()    # "  Hello, World!"

# Strip specific characters
text = "***Hello***"
stripped = text.strip("*")   # "Hello"
```

#### Searching and Finding
```python
text = "Hello, World! Welcome to Python programming."

# Find substring (returns index or -1)
index = text.find("World")       # 7
index = text.find("xyz")         # -1

# Find from specific position
index = text.find("o", 10)       # 16 (first 'o' after index 10)

# Index (like find but raises error if not found)
index = text.index("World")      # 7
# index = text.index("xyz")      # ValueError

# Count occurrences
count = text.count("o")          # 5
count = text.count("Python")     # 1

# Check start and end
starts = text.startswith("Hello")    # True
ends = text.endswith("programming.") # True

# Check if substring exists
if "Python" in text:
    print("Found!")
```

#### String Checking Methods
```python
# Alphanumeric checking
"abc123".isalnum()    # True
"abc 123".isalnum()   # False (has space)

# Alphabetic checking
"abc".isalpha()       # True
"abc123".isalpha()    # False

# Digit checking
"123".isdigit()       # True
"12.3".isdigit()      # False

# Decimal checking
"123".isdecimal()     # True
"12.3".isdecimal()    # False

# Numeric checking
"123".isnumeric()     # True
"½".isnumeric()       # True

# Space checking
"   ".isspace()       # True
" a ".isspace()       # False

# Identifier checking
"variable_name".isidentifier()  # True
"2variable".isidentifier()      # False

# Printable checking
"Hello!".isprintable()  # True
"Hello\n".isprintable() # False
```

### String Manipulation

#### Replacing and Translating
```python
text = "Hello, World!"

# Replace substring
new_text = text.replace("World", "Python")  # "Hello, Python!"
new_text = text.replace("l", "L")           # "HeLLo, WorLd!"

# Replace with limit
text = "one two one three one"
new_text = text.replace("one", "1", 2)  # "1 two 1 three one"

# Translate using translation table
translation = str.maketrans("aeiou", "12345")
translated = "hello".translate(translation)  # "h2ll4"

# Remove characters
remove_vowels = str.maketrans("", "", "aeiou")
no_vowels = "hello world".translate(remove_vowels)  # "hll wrld"
```

#### Splitting and Joining
```python
# Split by delimiter
text = "apple,banana,cherry"
fruits = text.split(",")  # ["apple", "banana", "cherry"]

# Split by whitespace (default)
text = "hello world python"
words = text.split()  # ["hello", "world", "python"]

# Split with max splits
text = "a-b-c-d-e"
parts = text.split("-", 2)  # ["a", "b", "c-d-e"]

# Split from right
text = "path/to/file.txt"
parts = text.rsplit("/", 1)  # ["path/to", "file.txt"]

# Split by lines
text = "line1\nline2\nline3"
lines = text.splitlines()  # ["line1", "line2", "line3"]

# Join strings
words = ["apple", "banana", "cherry"]
result = ", ".join(words)  # "apple, banana, cherry"
result = "-".join(words)   # "apple-banana-cherry"
result = "".join(words)    # "applebananacherry"

# Join with numbers (convert to strings first)
numbers = [1, 2, 3, 4, 5]
result = "-".join(map(str, numbers))  # "1-2-3-4-5"
```

#### Alignment and Padding
```python
text = "Python"

# Left align
left = text.ljust(10)       # "Python    "
left = text.ljust(10, "*")  # "Python****"

# Right align
right = text.rjust(10)       # "    Python"
right = text.rjust(10, "*")  # "****Python"

# Center
center = text.center(10)       # "  Python  "
center = text.center(10, "*")  # "**Python**"

# Zero padding
number = "42"
padded = number.zfill(5)  # "00042"
```

### String Formatting

#### Old Style (% formatting)
```python
name = "Alice"
age = 25

# Basic formatting
message = "My name is %s and I'm %d years old" % (name, age)

# With dictionaries
message = "My name is %(name)s and I'm %(age)d years old" % {"name": name, "age": age}
```

#### str.format() Method
```python
name = "Bob"
age = 30

# Positional arguments
message = "My name is {} and I'm {} years old".format(name, age)

# Named arguments
message = "My name is {name} and I'm {age} years old".format(name=name, age=age)

# Mixed
message = "My name is {0} and I'm {1} years old. {0} is a cool name!".format(name, age)

# Formatting numbers
pi = 3.14159
formatted = "Pi is approximately {:.2f}".format(pi)  # "Pi is approximately 3.14"

# Padding and alignment
"{:>10}".format("right")   # "     right"
"{:<10}".format("left")    # "left      "
"{:^10}".format("center")  # "  center  "
"{:*^10}".format("center") # "**center**"
```

#### f-strings (Python 3.6+)
```python
name = "Charlie"
age = 35

# Basic
message = f"My name is {name} and I'm {age} years old"

# Expressions
message = f"Next year I'll be {age + 1}"

# Formatting
pi = 3.14159
message = f"Pi is approximately {pi:.2f}"

# Alignment
message = f"{'right':>10}"   # "     right"
message = f"{'left':<10}"    # "left      "
message = f"{'center':^10}"  # "  center  "

# Debugging (Python 3.8+)
x = 10
print(f"{x=}")  # "x=10"

# Multiple lines
name = "Diana"
age = 28
message = (
    f"Name: {name}\n"
    f"Age: {age}\n"
    f"Status: Active"
)

# Calling methods
text = "hello"
message = f"Uppercase: {text.upper()}"
```

### String Encoding and Decoding
```python
# Encode string to bytes
text = "Hello, World!"
encoded = text.encode("utf-8")  # b'Hello, World!'

# Decode bytes to string
decoded = encoded.decode("utf-8")  # "Hello, World!"

# Different encodings
text = "Café"
utf8 = text.encode("utf-8")      # b'Caf\xc3\xa9'
latin1 = text.encode("latin-1")  # b'Caf\xe9'
```

### String Constants
```python
import string

# ASCII letters
print(string.ascii_lowercase)  # "abcdefghijklmnopqrstuvwxyz"
print(string.ascii_uppercase)  # "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
print(string.ascii_letters)    # "abcd...ABCD..."

# Digits
print(string.digits)           # "0123456789"

# Punctuation
print(string.punctuation)      # "!"#$%&'()*+,-./:;<=>?@[\]^_`{|}~"

# Whitespace
print(string.whitespace)       # " \t\n\r\x0b\x0c"
```

### Advanced String Operations

#### String Partitioning
```python
text = "key=value"

# Partition (split at first occurrence)
parts = text.partition("=")  # ("key", "=", "value")

# Rpartition (split at last occurrence)
path = "path/to/file.txt"
parts = path.rpartition("/")  # ("path/to", "/", "file.txt")
```

#### Removing Prefixes and Suffixes (Python 3.9+)
```python
text = "HelloWorld"

# Remove prefix
without_prefix = text.removeprefix("Hello")  # "World"

# Remove suffix
without_suffix = text.removesuffix("World")  # "Hello"
```

---

## Advanced: Stacks and Queues

These are abstract data types that can be implemented using Python's built-in data structures.

### Stack (LIFO - Last In, First Out)

Think of a stack of plates—you add and remove from the top.

#### Using List as Stack
```python
# Create empty stack
stack = []

# Push (add to top)
stack.append(1)
stack.append(2)
stack.append(3)
# Stack: [1, 2, 3]

# Pop (remove from top)
top = stack.pop()  # 3
# Stack: [1, 2]

# Peek (look at top without removing)
if stack:
    top = stack[-1]  # 2

# Check if empty
is_empty = len(stack) == 0

# Size
size = len(stack)
```

#### Stack Implementation Class
```python
class Stack:
    def __init__(self):
        self.items = []
    
    def push(self, item):
        """Add item to top of stack"""
        self.items.append(item)
    
    def pop(self):
        """Remove and return top item"""
        if not self.is_empty():
            return self.items.pop()
        raise IndexError("Stack is empty")
    
    def peek(self):
        """Return top item without removing"""
        if not self.is_empty():
            return self.items[-1]
        raise IndexError("Stack is empty")
    
    def is_empty(self):
        """Check if stack is empty"""
        return len(self.items) == 0
    
    def size(self):
        """Return number of items"""
        return len(self.items)
    
    def __str__(self):
        return str(self.items)

# Usage
stack = Stack()
stack.push(1)
stack.push(2)
stack.push(3)
print(stack)        # [1, 2, 3]
print(stack.pop())  # 3
print(stack.peek()) # 2
```

#### Stack Use Cases
```python
# 1. Balanced Parentheses Checker
def is_balanced(expression):
    stack = []
    opening = "({["
    closing = ")}]"
    pairs = {"(": ")", "{": "}", "[": "]"}
    
    for char in expression:
        if char in opening:
            stack.append(char)
        elif char in closing:
            if not stack:
                return False
            if pairs[stack.pop()] != char:
                return False
    
    return len(stack) == 0

print(is_balanced("({[]})"))  # True
print(is_balanced("({[}])"))  # False

# 2. Reverse String
def reverse_string(text):
    stack = []
    for char in text:
        stack.append(char)
    
    reversed_text = ""
    while stack:
        reversed_text += stack.pop()
    
    return reversed_text

print(reverse_string("Hello"))  # "olleH"

# 3. Undo Functionality
class TextEditor:
    def __init__(self):
        self.text = ""
        self.history = []
    
    def write(self, text):
        self.history.append(self.text)
        self.text += text
    
    def undo(self):
        if self.history:
            self.text = self.history.pop()
    
    def show(self):
        return self.text

editor = TextEditor()
editor.write("Hello")
editor.write(" World")
print(editor.show())  # "Hello World"
editor.undo()
print(editor.show())  # "Hello"
```

### Queue (FIFO - First In, First Out)

Think of a line at a store—first person in line is served first.

#### Using collections.deque
```python
from collections import deque

# Create empty queue
queue = deque()

# Enqueue (add to back)
queue.append(1)
queue.append(2)
queue.append(3)
# Queue: deque([1, 2, 3])

# Dequeue (remove from front)
first = queue.popleft()  # 1
# Queue: deque([2, 3])

# Peek at front
if queue:
    front = queue[0]  # 2

# Check if empty
is_empty = len(queue) == 0

# Size
size = len(queue)
```

#### Queue Implementation Class
```python
from collections import deque

class Queue:
    def __init__(self):
        self.items = deque()
    
    def enqueue(self, item):
        """Add item to back of queue"""
        self.items.append(item)
    
    def dequeue(self):
        """Remove and return front item"""
        if not self.is_empty():
            return self.items.popleft()
        raise IndexError("Queue is empty")
    
    def front(self):
        """Return front item without removing"""
        if not self.is_empty():
            return self.items[0]
        raise IndexError("Queue is empty")
    
    def is_empty(self):
        """Check if queue is empty"""
        return len(self.items) == 0
    
    def size(self):
        """Return number of items"""
        return len(self.items)
    
    def __str__(self):
        return str(list(self.items))

# Usage
queue = Queue()
queue.enqueue(1)
queue.enqueue(2)
queue.enqueue(3)
print(queue)           # [1, 2, 3]
print(queue.dequeue()) # 1
print(queue.front())   # 2
```

#### Queue Use Cases
```python
# 1. Print Queue Simulation
class PrintQueue:
    def __init__(self):
        self.queue = deque()
    
    def add_document(self, document):
        self.queue.append(document)
        print(f"Added: {document}")
    
    def print_next(self):
        if self.queue:
            doc = self.queue.popleft()
            print(f"Printing: {doc}")
        else:
            print("Queue is empty")
    
    def show_queue(self):
        print("Current queue:", list(self.queue))

printer = PrintQueue()
printer.add_document("Document1.pdf")
printer.add_document("Document2.pdf")
printer.show_queue()
printer.print_next()
printer.show_queue()

# 2. Breadth-First Search (BFS)
def bfs_tree(tree, start):
    """Traverse tree level by level"""
    queue = deque([start])
    visited = set([start])
    result = []
    
    while queue:
        node = queue.popleft()
        result.append(node)
        
        # Add neighbors to queue
        for neighbor in tree.get(node, []):
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
    
    return result

tree = {
    1: [2, 3],
    2: [4, 5],
    3: [6, 7],
    4: [], 5: [], 6: [], 7: []
}
print(bfs_tree(tree, 1))  # [1, 2, 3, 4, 5, 6, 7]
```

### Priority Queue

A queue where elements have priorities, and higher priority elements are dequeued first.
```python
import heapq

class PriorityQueue:
    def __init__(self):
        self.heap = []
        self.counter = 0
    
    def enqueue(self, item, priority):
        """Add item with priority (lower number = higher priority)"""
        heapq.heappush(self.heap, (priority, self.counter, item))
        self.counter += 1
    
    def dequeue(self):
        """Remove and return highest priority item"""
        if self.heap:
            return heapq.heappop(self.heap)[2]
        raise IndexError("Queue is empty")
    
    def is_empty(self):
        return len(self.heap) == 0

# Usage
pq = PriorityQueue()
pq.enqueue("Low priority task", 3)
pq.enqueue("High priority task", 1)
pq.enqueue("Medium priority task", 2)

print(pq.dequeue())  # "High priority task"
print(pq.dequeue())  # "Medium priority task"
print(pq.dequeue())  # "Low priority task"
```

### Deque (Double-Ended Queue)

Can add/remove from both ends efficiently.
```python
from collections import deque

dq = deque()

# Add to both ends
dq.append(1)        # Add to right: deque([1])
dq.appendleft(0)    # Add to left: deque([0, 1])
dq.extend([2, 3])   # Add multiple to right: deque([0, 1, 2, 3])
dq.extendleft([-2, -1])  # Add multiple to left: deque([-1, -2, 0, 1, 2, 3])

# Remove from both ends
right = dq.pop()      # Remove from right: 3
left = dq.popleft()   # Remove from left: -1

# Rotate
dq = deque([1, 2, 3, 4, 5])
dq.rotate(2)   # deque([4, 5, 1, 2, 3])
dq.rotate(-1)  # deque([5, 1, 2, 3, 4])

# Max length deque (circular buffer)
dq = deque(maxlen=3)
dq.extend([1, 2, 3])  # deque([1, 2, 3], maxlen=3)
dq.append(4)          # deque([2, 3, 4], maxlen=3)
```

---

## Choosing the Right Data Structure

Here's a comprehensive decision guide:

### Quick Decision Tree

**Need to store items?**
- Sequential order matters + allow duplicates → **List**
- Sequential order matters + no modifications → **Tuple**
- Key-value pairs needed → **Dictionary**
- Only unique items needed → **Set**
- Text data → **String**

**Need specific behavior?**
- Last in, first out (LIFO) → **Stack** (use list)
- First in, first out (FIFO) → **Queue** (use deque)
- Access from both ends → **Deque**
- Priority-based access → **Priority Queue**

### Detailed Comparison

| Feature | List | Tuple | Dictionary | Set | String |
|---------|------|-------|-----------|-----|--------|
| **Mutable** | Yes | No | Yes | Yes | No |
| **Ordered** | Yes | Yes | Yes (3.7+) | No | Yes |
| **Duplicates** | Yes | Yes | Keys: No, Values: Yes | No | Yes |
| **Indexed** | Yes | Yes | By key | No | Yes |
| **Use Case** | General purpose | Immutable data | Key-value pairs | Unique items | Text |

### Performance Characteristics

#### List Operations

| Operation | Time Complexity | Description |
|-----------|----------------|-------------|
| Access by index | O(1) | Direct access |
| Append | O(1) | Add to end |
| Insert at beginning | O(n) | Shift all elements |
| Insert in middle | O(n) | Shift elements |
| Delete from end | O(1) | Remove last |
| Delete from beginning | O(n) | Shift all elements |
| Search | O(n) | Linear search |
| Sort | O(n log n) | Timsort algorithm |

#### Dictionary Operations

| Operation | Time Complexity | Description |
|-----------|----------------|-------------|
| Access by key | O(1) | Hash table lookup |
| Insert | O(1) | Hash table insert |
| Delete | O(1) | Hash table delete |
| Search by value | O(n) | Must check all values |
| Iteration | O(n) | Visit all items |

#### Set Operations

| Operation | Time Complexity | Description |
|-----------|----------------|-------------|
| Add | O(1) | Hash table insert |
| Remove | O(1) | Hash table delete |
| Membership test | O(1) | Hash table lookup |
| Union | O(len(s) + len(t)) | Combine sets |
| Intersection | O(min(len(s), len(t))) | Find common |
| Difference | O(len(s)) | Find unique |

### Use Case Scenarios

#### When to Use Lists
```python
# Sequential data that changes
shopping_cart = ["apple", "banana", "milk"]
shopping_cart.append("bread")

# Maintaining order is important
rankings = ["first", "second", "third"]

# Need indexing
students = ["Alice", "Bob", "Charlie"]
first_student = students[0]

# Stacks and simple queues
stack = []
stack.append(item)  # push
item = stack.pop()  # pop
```

#### When to Use Tuples
```python
# Immutable coordinates or points
location = (40.7128, -74.0060)  # New York City

# Function return values
def get_user_info():
    return ("Alice", 25, "Engineer")

# Dictionary keys
locations = {
    (0, 0): "origin",
    (1, 1): "diagonal"
}

# Unpacking values
name, age, job = get_user_info()
```

#### When to Use Dictionaries
```python
# Mapping relationships
phone_book = {
    "Alice": "123-4567",
    "Bob": "234-5678"
}

# Fast lookups
student_grades = {
    "Alice": 95,
    "Bob": 87,
    "Charlie": 92
}

# Grouping related data
person = {
    "name": "Alice",
    "age": 25,
    "city": "Lagos",
    "skills": ["Python", "JavaScript"]
}

# Caching and memoization
cache = {}
def expensive_function(x):
    if x in cache:
        return cache[x]
    result = x ** 2  # expensive calculation
    cache[x] = result
    return result
```

#### When to Use Sets
```python
# Remove duplicates
numbers = [1, 2, 2, 3, 3, 3, 4]
unique = set(numbers)

# Fast membership testing
allowed_users = {"alice", "bob", "charlie"}
if user in allowed_users:
    grant_access()

# Mathematical set operations
set_a = {1, 2, 3, 4}
set_b = {3, 4, 5, 6}
common = set_a & set_b
unique_to_a = set_a - set_b

# Tracking visited items
visited = set()
for item in items:
    if item not in visited:
        process(item)
        visited.add(item)
```

#### When to Use Strings
```python
# Text data
message = "Hello, World!"

# Immutable text that needs processing
text = "Python Programming"
upper_text = text.upper()
words = text.split()

# Template strings
template = "Hello, {name}! Welcome to {place}."
message = template.format(name="Alice", place="Python")
```

---

## Performance Considerations

### Time Complexity Cheat Sheet

#### Common Operations

**O(1) - Constant Time** (Best):
- Dictionary: get, set, delete by key
- Set: add, remove, membership test
- List: access by index, append, pop from end
- Deque: append/pop from either end

**O(log n) - Logarithmic Time** (Good):
- Binary search in sorted list
- Priority queue operations (heap)

**O(n) - Linear Time** (Acceptable):
- List: search, insert/delete at beginning
- String: search substring
- Dictionary/Set: iteration
- Most comprehensions

**O(n log n) - Linearithmic Time** (Fair):
- Sorting lists: `sort()`, `sorted()`

**O(n²) - Quadratic Time** (Avoid if possible):
- Nested loops over same data
- Bubble sort, insertion sort

### Space Complexity

| Data Structure | Space Complexity | Notes |
|---------------|------------------|-------|
| List | O(n) | Plus overhead for capacity |
| Tuple | O(n) | More compact than list |
| Dictionary | O(n) | Hash table overhead |
| Set | O(n) | Hash table overhead |
| String | O(n) | Immutable, no extra capacity |

### Memory Usage Comparison
```python
import sys

# Compare memory usage
list_data = [1, 2, 3, 4, 5]
tuple_data = (1, 2, 3, 4, 5)
set_data = {1, 2, 3, 4, 5}
dict_data = {1: 'a', 2: 'b', 3: 'c', 4: 'd', 5: 'e'}

print(f"List:  {sys.getsizeof(list_data)} bytes")
print(f"Tuple: {sys.getsizeof(tuple_data)} bytes")
print(f"Set:   {sys.getsizeof(set_data)} bytes")
print(f"Dict:  {sys.getsizeof(dict_data)} bytes")

# Typical output:
# List:  120 bytes
# Tuple: 80 bytes
# Set:   224 bytes
# Dict:  232 bytes
```

### Optimization Tips

#### Use Appropriate Data Structures
```python
# Bad: Using list for membership testing
users = ["alice", "bob", "charlie", "diana"]
if "bob" in users:  # O(n) operation
    print("Found!")

# Good: Using set for membership testing
users = {"alice", "bob", "charlie", "diana"}
if "bob" in users:  # O(1) operation
    print("Found!")
```

#### Avoid Repeated Operations
```python
# Bad: Repeated dictionary lookups
result = data["key1"] + data["key2"] + data["key3"]

# Good: Store in variable
key1_value = data["key1"]
result = key1_value + data["key2"] + data["key3"]
```

#### Use Built-in Functions
```python
# Bad: Manual loop
total = 0
for num in numbers:
    total += num

# Good: Built-in function
total = sum(numbers)
```

#### List Comprehensions vs Loops
```python
# Slower: Append in loop
squares = []
for x in range(1000):
    squares.append(x**2)

# Faster: List comprehension
squares = [x**2 for x in range(1000)]
```

#### String Concatenation
```python
# Bad: Repeated concatenation (creates new string each time)
result = ""
for word in words:
    result += word + " "

# Good: Join method
result = " ".join(words)
```

---

## Practice Exercises

### Exercise 1: List Manipulation

**Task**: Create a list of numbers from 1 to 10, remove all even numbers, then square the remaining numbers.

**Solution**:
```python
# Create list
numbers = list(range(1, 11))

# Remove even numbers
odds = [x for x in numbers if x % 2 != 0]

# Square the numbers
squared = [x**2 for x in odds]

print(squared)  # [1, 9, 25, 49, 81]

# One-liner solution
squared = [x**2 for x in range(1, 11) if x % 2 != 0]
```

### Exercise 2: Dictionary Operations

**Task**: Create a dictionary of student grades, calculate the average grade, and find the student with the highest grade.

**Solution**:
```python
# Create dictionary
grades = {
    "Alice": 95,
    "Bob": 87,
    "Charlie": 92,
    "Diana": 88,
    "Eve": 90
}

# Calculate average
average = sum(grades.values()) / len(grades)
print(f"Average grade: {average:.2f}")

# Find student with highest grade
top_student = max(grades, key=grades.get)
top_grade = grades[top_student]
print(f"Top student: {top_student} with grade {top_grade}")

# Alternative: Using items()
top_student, top_grade = max(grades.items(), key=lambda x: x[1])
```

### Exercise 3: Set Operations

**Task**: Given two lists of numbers, find common elements and elements unique to each list.

**Solution**:
```python
# Two lists
list1 = [1, 2, 3, 4, 5, 6]
list2 = [4, 5, 6, 7, 8, 9]

# Convert to sets
set1 = set(list1)
set2 = set(list2)

# Common elements
common = set1 & set2
print(f"Common: {common}")  # {4, 5, 6}

# Unique to list1
unique_to_1 = set1 - set2
print(f"Unique to list1: {unique_to_1}")  # {1, 2, 3}

# Unique to list2
unique_to_2 = set2 - set1
print(f"Unique to list2: {unique_to_2}")  # {7, 8, 9}

# All unique elements (symmetric difference)
all_unique = set1 ^ set2
print(f"All unique: {all_unique}")  # {1, 2, 3, 7, 8, 9}
```

### Exercise 4: String Processing

**Task**: Count word frequency in a sentence and return a dictionary of word counts.

**Solution**:
```python
def word_frequency(sentence):
    # Convert to lowercase and split
    words = sentence.lower().split()
    
    # Remove punctuation
    import string
    words = [word.strip(string.punctuation) for word in words]
    
    # Count frequencies
    frequency = {}
    for word in words:
        frequency[word] = frequency.get(word, 0) + 1
    
    return frequency

sentence = "Hello world! Hello Python. Python is great."
result = word_frequency(sentence)
print(result)
# {'hello': 2, 'world': 1, 'python': 2, 'is': 1, 'great': 1}

# Using Counter
from collections import Counter
words = sentence.lower().split()
words = [word.strip(string.punctuation) for word in words]
result = Counter(words)
```

### Exercise 5: Stack Implementation

**Task**: Implement a function that checks if brackets in an expression are balanced.

**Solution**:
```python
def is_balanced(expression):
    stack = []
    pairs = {'(': ')', '{': '}', '[': ']'}
    opening = set(pairs.keys())
    closing = set(pairs.values())
    
    for char in expression:
        if char in opening:
            stack.append(char)
        elif char in closing:
            if not stack or pairs[stack.pop()] != char:
                return False
    
    return len(stack) == 0

# Test cases
print(is_balanced("()"))           # True
print(is_balanced("()[]{}"))       # True
print(is_balanced("([{}])"))       # True
print(is_balanced("([)]"))         # False
print(is_balanced("((()"))         # False
```

### Exercise 6: Nested Dictionary

**Task**: Create a nested dictionary representing a company structure and extract all employee names.

**Solution**:
```python
company = {
    "Engineering": {
        "Frontend": ["Alice", "Bob"],
        "Backend": ["Charlie", "Diana"]
    },
    "Sales": {
        "Enterprise": ["Eve", "Frank"],
        "SMB": ["Grace"]
    }
}

def get_all_employees(structure):
    employees = []
    for department in structure.values():
        for team in department.values():
            employees.extend(team)
    return employees

all_employees = get_all_employees(company)
print(all_employees)
# ['Alice', 'Bob', 'Charlie', 'Diana', 'Eve', 'Frank', 'Grace']

# Using list comprehension
all_employees = [
    employee 
    for department in company.values() 
    for team in department.values() 
    for employee in team
]
```

### Exercise 7: Tuple Unpacking

**Task**: Parse a CSV-style string and extract data using tuple unpacking.

**Solution**:
```python
def parse_csv_line(line):
    # Parse CSV line
    values = line.split(',')
    name, age, city, salary = values
    
    return {
        'name': name.strip(),
        'age': int(age.strip()),
        'city': city.strip(),
        'salary': float(salary.strip())
    }

csv_line = "Alice, 25, Lagos, 75000.50"
data = parse_csv_line(csv_line)
print(data)
# {'name': 'Alice', 'age': 25, 'city': 'Lagos', 'salary': 75000.5}

# Parse multiple lines
csv_data = """Alice, 25, Lagos, 75000
Bob, 30, Abuja, 85000
Charlie, 28, Port Harcourt, 72000"""

employees = []
for line in csv_data.split('\n'):
    employees.append(parse_csv_line(line))
```

### Exercise 8: Set-Based Deduplication

**Task**: Remove duplicate dictionaries from a list based on a specific key.

**Solution**:
```python
def deduplicate_by_key(items, key):
    seen = set()
    result = []
    
    for item in items:
        value = item[key]
        if value not in seen:
            seen.add(value)
            result.append(item)
    
    return result

users = [
    {"id": 1, "name": "Alice"},
    {"id": 2, "name": "Bob"},
    {"id": 1, "name": "Alice Duplicate"},
    {"id": 3, "name": "Charlie"},
    {"id": 2, "name": "Bob Duplicate"}
]

unique_users = deduplicate_by_key(users, "id")
print(unique_users)
# [{'id': 1, 'name': 'Alice'}, {'id': 2, 'name': 'Bob'}, {'id': 3, 'name': 'Charlie'}]
```

### Exercise 9: Queue Simulation

**Task**: Simulate a customer service queue where customers are served in order.

**Solution**:
```python
from collections import deque

class CustomerService:
    def __init__(self):
        self.queue = deque()
        self.ticket_number = 1
    
    def add_customer(self, name):
        ticket = self.ticket_number
        self.queue.append((ticket, name))
        self.ticket_number += 1
        print(f"{name} received ticket #{ticket}")
        return ticket
    
    def serve_next(self):
        if self.queue:
            ticket, name = self.queue.popleft()
            print(f"Now serving: {name} (ticket #{ticket})")
            return ticket, name
        else:
            print("No customers in queue")
            return None
    
    def show_queue(self):
        if self.queue:
            print("Current queue:")
            for ticket, name in self.queue:
                print(f"  Ticket #{ticket}: {name}")
        else:
            print("Queue is empty")
    
    def queue_length(self):
        return len(self.queue)

# Usage
cs = CustomerService()
cs.add_customer("Alice")
cs.add_customer("Bob")
cs.add_customer("Charlie")
cs.show_queue()
cs.serve_next()
cs.serve_next()
cs.show_queue()
```

### Exercise 10: Complex Data Transformation

**Task**: Transform a list of tuples into a nested dictionary structure.

**Solution**:
```python
# Input: List of (department, team, employee, salary) tuples
data = [
    ("Engineering", "Frontend", "Alice", 75000),
    ("Engineering", "Frontend", "Bob", 72000),
    ("Engineering", "Backend", "Charlie", 80000),
    ("Sales", "Enterprise", "Diana", 85000),
    ("Sales", "SMB", "Eve", 65000),
]

def transform_to_structure(data):
    result = {}
    
    for dept, team, employee, salary in data:
        # Create department if doesn't exist
        if dept not in result:
            result[dept] = {}
        
        # Create team if doesn't exist
        if team not in result[dept]:
            result[dept][team] = []
        
        # Add employee
        result[dept][team].append({
            "name": employee,
            "salary": salary
        })
    
    return result

company_structure = transform_to_structure(data)
print(company_structure)

# Calculate average salary by department
def avg_salary_by_dept(structure):
    averages = {}
    for dept, teams in structure.items():
        total = 0
        count = 0
        for employees in teams.values():
            for emp in employees:
                total += emp["salary"]
                count += 1
        averages[dept] = total / count if count > 0 else 0
    return averages

print(avg_salary_by_dept(company_structure))
```

---

## Key Takeaways

### Essential Principles

1. **Choose Based on Requirements**: Don't default to lists for everything. Each data structure has specific strengths.

2. **Immutability Matters**: Use tuples when data shouldn't change. This prevents bugs and can improve performance.

3. **Hash Tables Are Powerful**: Dictionaries and sets provide O(1) lookup. Use them when speed matters.

4. **Order When Needed**: Lists and tuples maintain order. Sets and dictionaries (pre-3.7) don't guarantee order.

5. **Readability First**: Clear code is better than clever code. Choose the data structure that makes your intent obvious.

### Performance Guidelines

- Use **sets** for membership testing
- Use **dictionaries** for key-value lookups
- Use **lists** when order and mutability matter
- Use **tuples** for immutable sequences
- Use **deque** instead of list for queue operations
- Use **list comprehensions** instead of loops when possible

### Common Pitfalls to Avoid

1. **Mutable Default Arguments**
```python
# Bad
def add_item(item, items=[]):
    items.append(item)
    return items

# Good
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

2. **Modifying List While Iterating**
```python
# Bad
for item in items:
    if condition:
        items.remove(item)

# Good
items = [item for item in items if not condition]
```

3. **String Concatenation in Loops**
```python
# Bad
result = ""
for s in strings:
    result += s

# Good
result = "".join(strings)
```

### Next Steps

1. **Practice Regularly**: Implement these data structures in real projects
2. **Study Algorithms**: Learn how algorithms use these structures
3. **Profile Your Code**: Use tools like `timeit` to measure performance
4. **Read Python Docs**: The official documentation has excellent examples
5. **Solve Problems**: Use platforms like LeetCode, HackerRank to practice

---

## Summary

Data structures are the foundation of effective programming. Master these concepts:

- **Lists**: Flexible, ordered, mutable collections
- **Tuples**: Immutable sequences for fixed data
- **Dictionaries**: Fast key-value lookups
- **Sets**: Unique elements, fast membership testing
- **Strings**: Immutable text with rich methods
- **Stacks**: LIFO for undo, parsing, recursion
- **Queues**: FIFO for task processing, BFS

The secret to mastery isn't just knowing these structures exist—it's understanding when and why to use each one. Build this intuition through practice, and you'll write cleaner, faster, more elegant code.

Start simple, practice deliberately, and remember: every great system is built on these foundations. Now go build something remarkable.

---

**End of Lesson**