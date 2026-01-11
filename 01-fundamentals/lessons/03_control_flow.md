# Control Flow in Python

## Introduction

Control flow is the order in which your code executes. In Python, control flow statements allow you to create programs that make decisions, repeat actions, and respond dynamically to data—the foundation of building intelligent, responsive applications.

---

## Table of Contents

1. [Conditional Statements](#1-conditional-statements)
2. [Loops](#2-loops)
3. [Loop Control Statements](#3-loop-control-statements)
4. [Match-Case Statement](#4-the-match-case-statement-python-310)
5. [Comprehensions](#5-comprehensions-elegant-control-flow)
6. [Real-World Examples](#6-real-world-examples)
7. [Key Takeaways](#key-takeaways)
8. [Practice Exercises](#practice-exercises)

---

## 1. Conditional Statements

Conditional statements let your program make decisions based on conditions.

### 1.1 The `if` Statement

The `if` statement executes code only when a condition is true.

**Syntax:**
```python
if condition:
    # code to execute if condition is True
```

**Example:**
```python
age = 25

if age >= 18:
    print("You are eligible to vote")
```

**Output:**
```
You are eligible to vote
```

---

### 1.2 The `if-else` Statement

Provides an alternative path when the condition is false.

**Syntax:**
```python
if condition:
    # code to execute if condition is True
else:
    # code to execute if condition is False
```

**Example:**
```python
temperature = 15

if temperature > 20:
    print("It's warm outside")
else:
    print("It's cold outside")
```

**Output:**
```
It's cold outside
```

---

### 1.3 The `if-elif-else` Statement

Handles multiple conditions in sequence.

**Syntax:**
```python
if condition1:
    # code if condition1 is True
elif condition2:
    # code if condition2 is True
elif condition3:
    # code if condition3 is True
else:
    # code if all conditions are False
```

**Example:**
```python
score = 85

if score >= 90:
    grade = "A"
elif score >= 80:
    grade = "B"
elif score >= 70:
    grade = "C"
elif score >= 60:
    grade = "D"
else:
    grade = "F"

print(f"Your grade is: {grade}")
```

**Output:**
```
Your grade is: B
```

---

### 1.4 Nested Conditionals

You can nest `if` statements inside others for complex decision-making.

**Example:**
```python
user_role = "admin"
is_authenticated = True

if is_authenticated:
    if user_role == "admin":
        print("Access granted to admin panel")
    else:
        print("Access granted to user dashboard")
else:
    print("Please log in")
```

**Output:**
```
Access granted to admin panel
```

---

### 1.5 Comparison Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `==` | Equal to | `x == y` |
| `!=` | Not equal to | `x != y` |
| `>` | Greater than | `x > y` |
| `<` | Less than | `x < y` |
| `>=` | Greater than or equal to | `x >= y` |
| `<=` | Less than or equal to | `x <= y` |

---

### 1.6 Logical Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `and` | Returns True if both statements are true | `x > 5 and x < 10` |
| `or` | Returns True if at least one statement is true | `x < 5 or x > 10` |
| `not` | Reverses the result | `not(x > 5)` |

**Example:**
```python
age = 25
has_license = True

if age >= 18 and has_license:
    print("You can drive")
elif age >= 18 and not has_license:
    print("You need to get a license first")
else:
    print("You are too young to drive")
```

**Output:**
```
You can drive
```

---

## 2. Loops

Loops allow you to repeat code multiple times, essential for processing collections and automating repetitive tasks.

### 2.1 The `for` Loop

Iterates over sequences (lists, tuples, strings, ranges).

**Syntax:**
```python
for variable in sequence:
    # code to execute for each item
```

#### 2.1.1 Iterating Over a List

**Example:**
```python
fruits = ["apple", "banana", "cherry"]
for fruit in fruits:
    print(fruit)
```

**Output:**
```
apple
banana
cherry
```

---

#### 2.1.2 Using `range()`

The `range()` function generates a sequence of numbers.

**Syntax:**
```python
range(start, stop, step)
```

**Example:**
```python
for i in range(5):
    print(f"Iteration {i}")
```

**Output:**
```
Iteration 0
Iteration 1
Iteration 2
Iteration 3
Iteration 4
```

**Example with start, stop, step:**
```python
for i in range(2, 10, 2):
    print(i)
```

**Output:**
```
2
4
6
8
```

---

#### 2.1.3 Iterating Over a String

**Example:**
```python
for char in "Python":
    print(char)
```

**Output:**
```
P
y
t
h
o
n
```

---

#### 2.1.4 Using `enumerate()`

Get both index and value while iterating.

**Example:**
```python
fruits = ["apple", "banana", "cherry"]
for index, fruit in enumerate(fruits):
    print(f"{index}: {fruit}")
```

**Output:**
```
0: apple
1: banana
2: cherry
```

**Starting index from 1:**
```python
for index, fruit in enumerate(fruits, start=1):
    print(f"{index}: {fruit}")
```

**Output:**
```
1: apple
2: banana
3: cherry
```

---

#### 2.1.5 Iterating Over Dictionaries

**Example:**
```python
user = {"name": "Daniel", "age": 28, "role": "CEO"}

# Iterate over keys
for key in user:
    print(key)

# Iterate over values
for value in user.values():
    print(value)

# Iterate over key-value pairs
for key, value in user.items():
    print(f"{key}: {value}")
```

**Output:**
```
name
age
role

Daniel
28
CEO

name: Daniel
age: 28
role: CEO
```

---

### 2.2 The `while` Loop

Repeats as long as a condition remains true.

**Syntax:**
```python
while condition:
    # code to execute while condition is True
```

**Example:**
```python
count = 0

while count < 5:
    print(f"Count is: {count}")
    count += 1
```

**Output:**
```
Count is: 0
Count is: 1
Count is: 2
Count is: 3
Count is: 4
```

---

#### 2.2.1 Practical Example: User Input Validation

**Example:**
```python
password = ""
while len(password) < 8:
    password = input("Enter a password (min 8 characters): ")

print("Password accepted!")
```

**Sample Interaction:**
```
Enter a password (min 8 characters): hi
Enter a password (min 8 characters): test
Enter a password (min 8 characters): secure123
Password accepted!
```

---

#### 2.2.2 Infinite Loops

Be careful to ensure your condition will eventually become `False`, or use `break` to exit.

**Example:**
```python
while True:
    user_input = input("Type 'quit' to exit: ")
    if user_input == "quit":
        break
    print(f"You typed: {user_input}")
```

---

### 2.3 Nested Loops

Loops can be nested inside other loops.

**Example:**
```python
for i in range(1, 4):
    for j in range(1, 4):
        print(f"i={i}, j={j}")
```

**Output:**
```
i=1, j=1
i=1, j=2
i=1, j=3
i=2, j=1
i=2, j=2
i=2, j=3
i=3, j=1
i=3, j=2
i=3, j=3
```

---

## 3. Loop Control Statements

These statements give you fine-grained control over loop execution.

### 3.1 `break`

Exits the loop immediately, regardless of the condition.

**Example:**
```python
for num in range(10):
    if num == 5:
        break
    print(num)
```

**Output:**
```
0
1
2
3
4
```

**Example with `while`:**
```python
count = 0
while True:
    print(count)
    count += 1
    if count >= 5:
        break
```

**Output:**
```
0
1
2
3
4
```

---

### 3.2 `continue`

Skips the current iteration and moves to the next one.

**Example:**
```python
for num in range(10):
    if num % 2 == 0:
        continue
    print(num)
```

**Output:**
```
1
3
5
7
9
```

**Example: Skip specific values**
```python
for i in range(1, 11):
    if i == 5 or i == 7:
        continue
    print(i)
```

**Output:**
```
1
2
3
4
6
8
9
10
```

---

### 3.3 `pass`

A placeholder that does nothing—useful when syntax requires a statement but you don't want any action.

**Example:**
```python
for item in ["special", "normal", "important"]:
    if item == "special":
        pass  # Handle this later
    else:
        print(f"Processing: {item}")
```

**Output:**
```
Processing: normal
Processing: important
```

**Common use case:**
```python
def future_function():
    pass  # TODO: Implement this later

class FutureClass:
    pass  # TODO: Add methods later
```

---

### 3.4 `else` Clause in Loops

Loops can have an `else` clause that executes when the loop completes normally (without `break`).

**Example with `for`:**
```python
for num in range(5):
    print(num)
else:
    print("Loop completed successfully")
```

**Output:**
```
0
1
2
3
4
Loop completed successfully
```

**Example with `break`:**
```python
for num in range(5):
    if num == 3:
        break
    print(num)
else:
    print("Loop completed successfully")
```

**Output:**
```
0
1
2
```
*(Note: `else` clause does not execute because loop was broken)*

---

## 4. The `match-case` Statement (Python 3.10+)

Python's structural pattern matching—similar to switch statements in other languages but more powerful.

**Syntax:**
```python
match subject:
    case pattern1:
        # code
    case pattern2:
        # code
    case _:
        # default case
```

### 4.1 Basic Example

**Example:**
```python
def http_status(code):
    match code:
        case 200:
            return "OK"
        case 404:
            return "Not Found"
        case 500:
            return "Internal Server Error"
        case _:
            return "Unknown Status"

print(http_status(404))
print(http_status(200))
print(http_status(999))
```

**Output:**
```
Not Found
OK
Unknown Status
```

---

### 4.2 Pattern Matching with Multiple Values

**Example:**
```python
def day_type(day):
    match day:
        case "Saturday" | "Sunday":
            return "Weekend"
        case "Monday" | "Tuesday" | "Wednesday" | "Thursday" | "Friday":
            return "Weekday"
        case _:
            return "Invalid day"

print(day_type("Saturday"))
print(day_type("Monday"))
```

**Output:**
```
Weekend
Weekday
```

---

### 4.3 Advanced Pattern Matching

**Example with sequence patterns:**
```python
def process_command(command):
    match command.split():
        case ["quit"]:
            return "Exiting..."
        case ["load", filename]:
            return f"Loading {filename}"
        case ["save", filename]:
            return f"Saving to {filename}"
        case ["delete", *files]:
            return f"Deleting {len(files)} files"
        case _:
            return "Unknown command"

print(process_command("load data.txt"))
print(process_command("delete file1.txt file2.txt file3.txt"))
```

**Output:**
```
Loading data.txt
Deleting 3 files
```

---

### 4.4 Pattern Matching with Dictionaries

**Example:**
```python
def process_user(user):
    match user:
        case {"role": "admin", "name": name}:
            return f"Admin access granted for {name}"
        case {"role": "user", "name": name}:
            return f"User access granted for {name}"
        case _:
            return "Access denied"

print(process_user({"role": "admin", "name": "Daniel"}))
print(process_user({"role": "user", "name": "John"}))
```

**Output:**
```
Admin access granted for Daniel
User access granted for John
```

---

## 5. Comprehensions (Elegant Control Flow)

Python's comprehensions combine loops and conditionals in a single, readable line.

### 5.1 List Comprehensions

**Syntax:**
```python
[expression for item in iterable if condition]
```

#### 5.1.1 Basic List Comprehension

**Traditional approach:**
```python
squares = []
for x in range(10):
    squares.append(x**2)
print(squares)
```

**List comprehension:**
```python
squares = [x**2 for x in range(10)]
print(squares)
```

**Output:**
```
[0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
```

---

#### 5.1.2 List Comprehension with Condition

**Example:**
```python
even_squares = [x**2 for x in range(10) if x % 2 == 0]
print(even_squares)
```

**Output:**
```
[0, 4, 16, 36, 64]
```

---

#### 5.1.3 List Comprehension with Multiple Conditions

**Example:**
```python
numbers = [x for x in range(20) if x % 2 == 0 if x % 3 == 0]
print(numbers)
```

**Output:**
```
[0, 6, 12, 18]
```

---

#### 5.1.4 List Comprehension with if-else

**Example:**
```python
labels = ["Even" if x % 2 == 0 else "Odd" for x in range(10)]
print(labels)
```

**Output:**
```
['Even', 'Odd', 'Even', 'Odd', 'Even', 'Odd', 'Even', 'Odd', 'Even', 'Odd']
```

---

#### 5.1.5 Nested List Comprehension

**Example:**
```python
matrix = [[i * j for j in range(1, 4)] for i in range(1, 4)]
print(matrix)
```

**Output:**
```
[[1, 2, 3], [2, 4, 6], [3, 6, 9]]
```

**Flattening a nested list:**
```python
nested = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flat = [num for row in nested for num in row]
print(flat)
```

**Output:**
```
[1, 2, 3, 4, 5, 6, 7, 8, 9]
```

---

### 5.2 Dictionary Comprehensions

**Syntax:**
```python
{key_expression: value_expression for item in iterable if condition}
```

**Example:**
```python
squares_dict = {x: x**2 for x in range(5)}
print(squares_dict)
```

**Output:**
```
{0: 0, 1: 1, 2: 4, 3: 9, 4: 16}
```

**Example with condition:**
```python
even_squares_dict = {x: x**2 for x in range(10) if x % 2 == 0}
print(even_squares_dict)
```

**Output:**
```
{0: 0, 2: 4, 4: 16, 6: 36, 8: 64}
```

**Example: Swapping keys and values**
```python
original = {"a": 1, "b": 2, "c": 3}
swapped = {value: key for key, value in original.items()}
print(swapped)
```

**Output:**
```
{1: 'a', 2: 'b', 3: 'c'}
```

---

### 5.3 Set Comprehensions

**Syntax:**
```python
{expression for item in iterable if condition}
```

**Example:**
```python
unique_lengths = {len(word) for word in ["apple", "banana", "cherry", "apple"]}
print(unique_lengths)
```

**Output:**
```
{5, 6}
```

**Example:**
```python
squared_set = {x**2 for x in range(-5, 6)}
print(squared_set)
```

**Output:**
```
{0, 1, 4, 9, 16, 25}
```

---

### 5.4 Generator Expressions

Similar to list comprehensions but use parentheses and generate values lazily (memory efficient).

**Syntax:**
```python
(expression for item in iterable if condition)
```

**Example:**
```python
gen = (x**2 for x in range(10))
print(type(gen))
print(list(gen))
```

**Output:**
```
<class 'generator'>
[0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
```

**Memory efficiency example:**
```python
# List comprehension - creates entire list in memory
large_list = [x**2 for x in range(1000000)]

# Generator expression - generates values on demand
large_gen = (x**2 for x in range(1000000))
```

---

## 6. Real-World Examples

### 6.1 User Authentication System

**Example:**
```python
def authenticate(username, password):
    """
    Authenticate user credentials
    """
    users = {
        "admin": "secure123",
        "daniel": "premium456",
        "user1": "password789"
    }
    
    if username not in users:
        return "Username not found"
    elif users[username] != password:
        return "Incorrect password"
    else:
        return "Login successful"

# Test the function
print(authenticate("daniel", "premium456"))
print(authenticate("daniel", "wrongpass"))
print(authenticate("unknown", "anypass"))
```

**Output:**
```
Login successful
Incorrect password
Username not found
```

---

### 6.2 Processing Business Data

**Example:**
```python
sales = [1200, 850, 2100, 430, 1800, 990]

# Categorize sales performance
for sale in sales:
    if sale >= 2000:
        category = "Excellent"
    elif sale >= 1000:
        category = "Good"
    else:
        category = "Needs Improvement"
    
    print(f"Sale: ${sale} - {category}")
```

**Output:**
```
Sale: $1200 - Good
Sale: $850 - Needs Improvement
Sale: $2100 - Excellent
Sale: $430 - Needs Improvement
Sale: $1800 - Good
Sale: $990 - Needs Improvement
```

---

### 6.3 Filtering Premium Clients

**Example:**
```python
clients = [
    {"name": "Tech Corp", "revenue": 50000, "tier": "enterprise"},
    {"name": "Startup Inc", "revenue": 5000, "tier": "basic"},
    {"name": "Global Solutions", "revenue": 100000, "tier": "enterprise"},
    {"name": "Medium Business", "revenue": 80000, "tier": "enterprise"},
]

# Get all enterprise clients with revenue > 75k
premium_clients = [
    client["name"] 
    for client in clients 
    if client["tier"] == "enterprise" and client["revenue"] > 75000
]

print("Premium Clients:")
for client in premium_clients:
    print(f"  - {client}")
```

**Output:**
```
Premium Clients:
  - Global Solutions
  - Medium Business
```

---

### 6.4 Grade Calculator

**Example:**
```python
def calculate_grade(scores):
    """
    Calculate average and letter grade from list of scores
    """
    if not scores:
        return None, "No scores provided"
    
    average = sum(scores) / len(scores)
    
    if average >= 90:
        letter = "A"
    elif average >= 80:
        letter = "B"
    elif average >= 70:
        letter = "C"
    elif average >= 60:
        letter = "D"
    else:
        letter = "F"
    
    return average, letter

# Test the function
test_scores = [85, 92, 78, 90, 88]
avg, grade = calculate_grade(test_scores)
print(f"Average: {avg:.2f}")
print(f"Letter Grade: {grade}")
```

**Output:**
```
Average: 86.60
Letter Grade: B
```

---

### 6.5 Command-Line Menu System

**Example:**
```python
def show_menu():
    """
    Display and process menu options
    """
    while True:
        print("\n=== Main Menu ===")
        print("1. View Profile")
        print("2. Edit Settings")
        print("3. View Reports")
        print("4. Exit")
        
        choice = input("Enter your choice (1-4): ")
        
        match choice:
            case "1":
                print("Displaying profile...")
            case "2":
                print("Opening settings...")
            case "3":
                print("Loading reports...")
            case "4":
                print("Goodbye!")
                break
            case _:
                print("Invalid choice. Please try again.")

# Uncomment to run
# show_menu()
```

---

### 6.6 Data Validation

**Example:**
```python
def validate_email(email):
    """
    Basic email validation
    """
    if "@" not in email:
        return False, "Email must contain @"
    
    if email.count("@") > 1:
        return False, "Email can only have one @"
    
    username, domain = email.split("@")
    
    if len(username) == 0:
        return False, "Username cannot be empty"
    
    if "." not in domain:
        return False, "Domain must contain a dot"
    
    return True, "Valid email"

# Test cases
emails = [
    "daniel@onswift.com",
    "invalidemail",
    "no@domain",
    "double@@email.com",
    "@nodomain.com"
]

for email in emails:
    is_valid, message = validate_email(email)
    print(f"{email}: {message}")
```

**Output:**
```
daniel@onswift.com: Valid email
invalidemail: Email must contain @
no@domain: Domain must contain a dot
double@@email.com: Email can only have one @
@nodomain.com: Username cannot be empty
```

---

### 6.7 Prime Number Finder

**Example:**
```python
def find_primes(n):
    """
    Find all prime numbers up to n
    """
    primes = []
    
    for num in range(2, n + 1):
        is_prime = True
        
        for i in range(2, int(num ** 0.5) + 1):
            if num % i == 0:
                is_prime = False
                break
        
        if is_prime:
            primes.append(num)
    
    return primes

# Find primes up to 50
result = find_primes(50)
print(f"Prime numbers up to 50: {result}")
```

**Output:**
```
Prime numbers up to 50: [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47]
```

---

### 6.8 FizzBuzz Challenge

**Example:**
```python
def fizzbuzz(n):
    """
    Classic FizzBuzz problem
    """
    for i in range(1, n + 1):
        if i % 3 == 0 and i % 5 == 0:
            print("FizzBuzz")
        elif i % 3 == 0:
            print("Fizz")
        elif i % 5 == 0:
            print("Buzz")
        else:
            print(i)

# Run FizzBuzz for 1-15
fizzbuzz(15)
```

**Output:**
```
1
2
Fizz
4
Buzz
Fizz
7
8
Fizz
Buzz
11
Fizz
13
14
FizzBuzz
```

---

## Key Takeaways

1. **Conditionals** (`if`, `elif`, `else`) enable decision-making in your code
2. **Loops** (`for`, `while`) automate repetitive tasks and process collections
3. **Control statements** (`break`, `continue`, `pass`) provide fine-grained control
4. **Pattern matching** (`match-case`) offers elegant handling of multiple conditions
5. **Comprehensions** combine loops and conditions into concise, readable expressions
6. **Comparison operators** (`==`, `!=`, `>`, `<`, `>=`, `<=`) allow value comparison
7. **Logical operators** (`and`, `or`, `not`) combine multiple conditions
8. **`enumerate()`** provides both index and value when iterating
9. **Generator expressions** offer memory-efficient iteration for large datasets
10. **Loop `else` clauses** execute when loops complete without breaking

Mastering control flow transforms you from writing linear scripts to building intelligent, responsive applications that adapt to any situation.

---

## Practice Exercises

### Exercise 1: Prime Number Finder
Write a program that finds all prime numbers between 1 and 100.

**Hint:** A prime number is only divisible by 1 and itself.

---

### Exercise 2: Grade Calculator
Create a grade calculator that:
- Takes multiple test scores as input
- Calculates the average
- Returns the letter grade (A, B, C, D, F)
- Handles edge cases (empty list, invalid scores)

---

### Exercise 3: Command-Line Menu
Build a simple command-line menu system that:
- Displays options to the user
- Processes user input
- Performs actions based on selection
- Loops until user chooses to exit

---

### Exercise 4: Data Filtering
Use list comprehension to:
- Filter a dataset of your choice (e.g., sales data, student records)
- Transform the filtered data
- Create a new data structure with the results

**Example dataset:**
```python
products = [
    {"name": "Laptop", "price": 1200, "category": "Electronics"},
    {"name": "Phone", "price": 800, "category": "Electronics"},
    {"name": "Desk", "price": 300, "category": "Furniture"},
    {"name": "Chair", "price": 150, "category": "Furniture"},
]
```

---

### Exercise 5: Password Validator
Create a password validation function that checks:
- Minimum length (8 characters)
- Contains at least one uppercase letter
- Contains at least one lowercase letter
- Contains at least one number
- Contains at least one special character

**Return:** `True` if valid, `False` with error message if invalid

---

### Exercise 6: Fibonacci Sequence
Generate the first n numbers in the Fibonacci sequence using:
- A `for` loop
- A `while` loop
- List comprehension (if possible)

**Fibonacci sequence:** Each number is the sum of the two preceding ones (0, 1, 1, 2, 3, 5, 8, 13...)

---

### Exercise 7: Shopping Cart
Build a shopping cart system that:
- Allows adding items with prices
- Calculates subtotal
- Applies discount based on total (10% if over $100, 15% if over $200)
- Calculates tax (8%)
- Shows final total

---

### Exercise 8: Pattern Printer
Create a program that prints various patterns using nested loops:

**Pattern 1 (Right Triangle):**
```
*
**
***
****
*****
```

**Pattern 2 (Pyramid):**
```
    *
   ***
  *****
 *******
*********
```

---

### Exercise 9: Word Frequency Counter
Write a program that:
- Takes a string of text
- Counts the frequency of each word
- Returns a dictionary with word frequencies
- Uses comprehensions where possible

---

### Exercise 10: Number Guessing Game
Create a number guessing game that:
- Generates a random number between 1 and 100
- Asks user to guess
- Provides hints (higher/lower)
- Counts number of attempts
- Uses a `while` loop with `break` when guessed correctly

---

## Next Steps

Move on to **Functions and Modules** to organize your control flow logic into reusable, scalable components. You'll learn how to:
- Define and call functions
- Work with parameters and return values
- Understand scope and namespaces
- Create and import modules
- Build modular, maintainable code

---

## Additional Resources

- [Python Official Documentation - Control Flow](https://docs.python.org/3/tutorial/controlflow.html)
- [Python Pattern Matching (PEP 636)](https://peps.python.org/pep-0636/)
- [Real Python - Control Flow](https://realpython.com/python-conditional-statements/)
- [Python Comprehensions Guide](https://realpython.com/list-comprehension-python/)

---

**Author:** Daniel Araromi  
**Company:** OnSwift | Oversubscribed  
**Last Updated:** January 2026

---