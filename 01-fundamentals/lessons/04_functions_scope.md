# Function Scope in Python: A Complete Guide

Function scope determines where variables can be accessed in your code. Understanding scope is crucial for writing clean, bug-free Python programs.

## The Four Levels of Scope (LEGB Rule)

Python follows the LEGB rule when looking up variables:

**L**ocal → **E**nclosing → **G**lobal → **B**uilt-in

### 1. Local Scope

Variables defined inside a function exist only within that function.
```python
def calculate_profit():
    revenue = 10000  # Local variable
    costs = 6000
    profit = revenue - costs
    return profit

print(calculate_profit())  # Works: 4000
print(revenue)  # Error: revenue doesn't exist outside the function
```

### 2. Enclosing Scope

When you have nested functions, the inner function can access variables from the outer function.
```python
def outer_function():
    company = "Oversubscribed"  # Enclosing scope
    
    def inner_function():
        print(f"Working at {company}")  # Accesses enclosing scope
    
    inner_function()

outer_function()  # Output: Working at Oversubscribed
```

### 3. Global Scope

Variables defined at the top level of a script are global and accessible anywhere.
```python
platform = "OnSwift"  # Global variable

def show_platform():
    print(platform)  # Accesses global variable

show_platform()  # Output: OnSwift
```

### 4. Built-in Scope

Python's built-in functions and keywords like `print()`, `len()`, `range()` are always available.
```python
def count_clients(clients):
    return len(clients)  # len() is built-in

clients = ["Coach A", "Coach B", "Coach C"]
print(count_clients(clients))  # Output: 3
```

## Modifying Variables Across Scopes

### The `global` Keyword

Use `global` to modify a global variable from inside a function.
```python
revenue = 0

def add_client_payment(amount):
    global revenue  # Declare we're using the global variable
    revenue += amount

add_client_payment(5000)
add_client_payment(3000)
print(revenue)  # Output: 8000
```

**Without `global`, you create a new local variable:**
```python
revenue = 0

def add_payment(amount):
    revenue = amount  # Creates LOCAL variable, doesn't modify global
    
add_payment(5000)
print(revenue)  # Output: 0 (unchanged!)
```

### The `nonlocal` Keyword

Use `nonlocal` to modify a variable in an enclosing (but not global) scope.
```python
def create_counter():
    count = 0
    
    def increment():
        nonlocal count  # Refers to count in enclosing scope
        count += 1
        return count
    
    return increment

counter = create_counter()
print(counter())  # 1
print(counter())  # 2
print(counter())  # 3
```

## Common Scope Pitfalls

### Pitfall 1: Shadowing Variables

Local variables can "shadow" global ones, leading to confusion.
```python
name = "OnSwift"

def update_name():
    name = "Oversubscribed"  # Creates new local variable
    print(name)  # Prints: Oversubscribed

update_name()
print(name)  # Prints: OnSwift (unchanged)
```

### Pitfall 2: Loop Variables Leaking

In Python, loop variables remain accessible after the loop (unlike some languages).
```python
for i in range(3):
    pass

print(i)  # Works: 2 (last value from loop)
```

### Pitfall 3: Mutable Default Arguments

Default arguments are evaluated once at function definition, not each call.
```python
def add_client(client, client_list=[]):  # Dangerous!
    client_list.append(client)
    return client_list

print(add_client("Coach A"))  # ['Coach A']
print(add_client("Coach B"))  # ['Coach A', 'Coach B'] - Unexpected!
```

**Solution: Use `None` as default**
```python
def add_client(client, client_list=None):
    if client_list is None:
        client_list = []
    client_list.append(client)
    return client_list

print(add_client("Coach A"))  # ['Coach A']
print(add_client("Coach B"))  # ['Coach B'] - Correct!
```

## Best Practices

**Minimize global variables.** They make code harder to test and debug. Prefer passing values as function parameters.

**Use descriptive names.** Avoid reusing variable names across different scopes.

**Be explicit.** If you need to modify a global or enclosing variable, use `global` or `nonlocal` to make your intention clear.

**Prefer function parameters and return values** over modifying external state.
```python
# Instead of this:
total = 0
def add_to_total(value):
    global total
    total += value

# Do this:
def calculate_total(current_total, value):
    return current_total + value

total = 0
total = calculate_total(total, 100)
```

## Practical Example: Client Management System

Here's how scope works in a real-world scenario:
```python
class ClientManager:
    total_clients = 0  # Class variable (similar to global for the class)
    
    def __init__(self):
        self.clients = []  # Instance variable
    
    def add_client(self, name, package):
        client = {  # Local variables
            "name": name,
            "package": package
        }
        self.clients.append(client)
        ClientManager.total_clients += 1
        
        def send_welcome():  # Nested function
            return f"Welcome {name}!"  # Accesses enclosing scope
        
        return send_welcome()

manager = ClientManager()
print(manager.add_client("Dr. Charles", "Premium"))
print(f"Total clients: {ClientManager.total_clients}")
```

Understanding scope gives you precise control over where your data lives and how it's accessed. Master this, and you'll write cleaner, more maintainable Python code.