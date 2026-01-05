# 🐍 Python Fundamentals: Variables & Data Types

Welcome to your first step in mastering Python! Before we can build complex systems, we need to understand how Python stores and handles information.

## 1. What is a Variable?

Think of a variable as a labeled storage box.

- **The Box** holds the data.
- **The Label** is the variable name.
- **The Assignment Operator (=)** is how you put the data into the box.

```python
# 'user_name' is the label, "Alice" is the data
user_name = "Alice"
age = 25
```

## 2. Core Data Types

Python is "dynamically typed," meaning you don't have to tell Python what kind of data you are storing; it figures it out automatically!

### A. Integers (int)

Whole numbers without decimals. Used for counting, IDs, or age.

```python
score = 100
level = -5
```

### B. Floats (float)

Numbers with a decimal point. Used for prices, measurements, or scientific data.

```python
price = 19.99
pi = 3.14159
```

### C. Strings (str)

Text wrapped in single (`'`) or double (`"`) quotes.

```python
greeting = "Hello, World!"
language = 'Python'
```

**Pro Tip:** You can use f-strings to inject variables into text easily:

```python
print(f"User {user_name} is {age} years old.")
```

### D. Booleans (bool)

The simplest type. It can only be one of two values: `True` or `False`. Used for logic and decision making.

```python
is_logged_in = True
has_permission = False
```

## 3. Naming Rules (PEP 8)

To keep our community code clean and readable, please follow these naming conventions:

- **Snake Case:** Use lowercase letters and underscores (e.g., `user_account_balance`, not `userAccountBalance`).
- **Descriptive:** Names should explain what is inside. `x = 5` is bad; `retry_attempts = 5` is good.
- **Start with Letters:** Variables cannot start with a number (e.g., `1st_user` is invalid).

## 4. Type Conversion (Casting)

Sometimes you need to change one type into another. This is called **Casting**.

```python
# Converting a string to an integer
age_input = "25"
actual_age = int(age_input) 

# Converting an integer to a float
points = 10
decimal_points = float(points) # Result: 10.0
```

## 5. Practical Summary Table

| Data Type | Keyword | Example | Use Case |
|-----------|---------|---------|----------|
| Integer | `int` | `10`, `-2` | Counting items |
| Float | `float` | `10.5`, `0.0` | Math & Money |
| String | `str` | `"Dev"` | Names & Messages |
| Boolean | `bool` | `True` | Logic & Checks |