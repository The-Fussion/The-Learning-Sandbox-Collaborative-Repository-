# Python Decorators: A Detailed Guide

## Table of Contents
1. [Functions as First-Class Objects](#functions-as-first-class-objects)
2. [What Are Decorators?](#what-are-decorators)
3. [Basic Decorator Syntax](#basic-decorator-syntax)
4. [Decorators with Arguments](#decorators-with-arguments)
5. [Preserving Function Metadata](#preserving-function-metadata)
6. [Common Built-in Decorators](#common-built-in-decorators)
7. [Practical Examples](#practical-examples)
8. [Chaining Decorators](#chaining-decorators)
9. [Class Decorators](#class-decorators)

---

## Functions as First-Class Objects

Before understanding decorators, you need to grasp that in Python, functions are first-class objects. This means functions can be:

- Assigned to variables
- Passed as arguments to other functions
- Returned from other functions
- Stored in data structures

```python
# Assigning a function to a variable
def greet(name):
    return f"Hello, {name}!"

say_hello = greet
print(say_hello("Alice"))  # Output: Hello, Alice!

# Passing a function as an argument
def execute_function(func, value):
    return func(value)

result = execute_function(greet, "Bob")
print(result)  # Output: Hello, Bob!

# Returning a function from another function
def create_multiplier(factor):
    def multiply(x):
        return x * factor
    return multiply

times_three = create_multiplier(3)
print(times_three(10))  # Output: 30
```

---

## What Are Decorators?

A decorator is a function that takes another function as input and extends or modifies its behavior without permanently changing it. Decorators use the `@` syntax as syntactic sugar for wrapping functions.

Think of decorators as "wrappers" that add functionality before and/or after the original function executes.

---

## Basic Decorator Syntax

### The Long Way (Without @ Syntax)

```python
def my_decorator(func):
    def wrapper():
        print("Something before the function")
        func()
        print("Something after the function")
    return wrapper

def say_hello():
    print("Hello!")

# Manually wrapping the function
say_hello = my_decorator(say_hello)
say_hello()

# Output:
# Something before the function
# Hello!
# Something after the function
```

### The Decorator Syntax (With @)

```python
def my_decorator(func):
    def wrapper():
        print("Something before the function")
        func()
        print("Something after the function")
    return wrapper

@my_decorator
def say_hello():
    print("Hello!")

say_hello()

# Output:
# Something before the function
# Hello!
# Something after the function
```

The `@my_decorator` syntax is equivalent to `say_hello = my_decorator(say_hello)`.

---

## Decorators with Arguments

### Decorating Functions That Take Arguments

```python
def my_decorator(func):
    def