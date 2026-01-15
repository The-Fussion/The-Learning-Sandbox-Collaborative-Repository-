# Error Handling in Python: A Complete Guide

Error handling is the process of anticipating, detecting, and resolving errors that occur during program execution. Mastering error handling separates professional developers from amateurs—it's the difference between a program that crashes unexpectedly and one that gracefully handles problems.

## Why Error Handling Matters

Without proper error handling, your application crashes at the first sign of trouble. With it, you create resilient systems that can:

- Recover from unexpected situations
- Provide meaningful feedback to users
- Log issues for debugging
- Maintain data integrity
- Deliver premium user experiences

## Understanding Exceptions

An **exception** is an event that disrupts the normal flow of a program. When Python encounters an error, it "raises" an exception.
```python
# This raises a ZeroDivisionError
result = 10 / 0
```

If unhandled, exceptions terminate your program immediately.

## The Try-Except Block

The fundamental structure for handling errors in Python.

### Basic Syntax
```python
try:
    # Code that might raise an exception
    risky_operation()
except ExceptionType:
    # Code that runs if the exception occurs
    handle_error()
```

### Real Example
```python
def calculate_roi(revenue, investment):
    try:
        roi = ((revenue - investment) / investment) * 100
        return roi
    except ZeroDivisionError:
        return "Cannot calculate ROI with zero investment"

print(calculate_roi(10000, 5000))  # 100.0
print(calculate_roi(10000, 0))     # Cannot calculate ROI with zero investment
```

## Common Python Exceptions

### ZeroDivisionError

Occurs when dividing by zero.
```python
try:
    price_per_unit = total_price / quantity
except ZeroDivisionError:
    print("Quantity cannot be zero")
```

### ValueError

Raised when a function receives an argument of correct type but inappropriate value.
```python
try:
    age = int(input("Enter your age: "))
except ValueError:
    print("Please enter a valid number")
```

### KeyError

Occurs when accessing a dictionary key that doesn't exist.
```python
client_data = {"name": "Dr. Charles", "package": "Premium"}

try:
    email = client_data["email"]
except KeyError:
    email = "No email provided"
```

### FileNotFoundError

Raised when trying to open a file that doesn't exist.
```python
try:
    with open("client_list.txt", "r") as file:
        data = file.read()
except FileNotFoundError:
    print("Client list file not found. Creating new file...")
    with open("client_list.txt", "w") as file:
        file.write("")
```

### TypeError

Occurs when an operation is applied to an object of inappropriate type.
```python
try:
    result = "OnSwift" + 2026
except TypeError:
    result = "OnSwift " + str(2026)
```

### IndexError

Raised when trying to access an index that doesn't exist in a list.
```python
clients = ["Coach A", "Coach B", "Coach C"]

try:
    fourth_client = clients[3]
except IndexError:
    print("Client index out of range")
```

## Advanced Error Handling Techniques

### Catching Multiple Exceptions

**Method 1: Separate except blocks**
```python
try:
    revenue = int(input("Enter revenue: "))
    clients = int(input("Enter number of clients: "))
    revenue_per_client = revenue / clients
except ValueError:
    print("Please enter valid numbers")
except ZeroDivisionError:
    print("Number of clients cannot be zero")
```

**Method 2: Tuple of exceptions**
```python
try:
    process_payment(amount, client_id)
except (ValueError, TypeError) as e:
    print(f"Invalid payment data: {e}")
```

### The `else` Clause

Runs only if no exception was raised in the try block.
```python
def add_client(name, email):
    try:
        validate_email(email)
    except ValueError:
        print("Invalid email format")
    else:
        # Only executes if validation succeeded
        save_to_database(name, email)
        print(f"Client {name} added successfully")
```

### The `finally` Clause

Always executes, regardless of whether an exception occurred. Perfect for cleanup operations.
```python
def process_client_data(filename):
    file = None
    try:
        file = open(filename, 'r')
        data = file.read()
        process_data(data)
    except FileNotFoundError:
        print("File not found")
    except Exception as e:
        print(f"Error processing file: {e}")
    finally:
        # Always closes the file, even if an error occurred
        if file:
            file.close()
            print("File closed")
```

### Complete Try-Except-Else-Finally Example
```python
def calculate_and_save_metrics(revenue, costs, filename):
    try:
        profit = revenue - costs
        margin = (profit / revenue) * 100
    except ZeroDivisionError:
        print("Revenue cannot be zero")
        return None
    except TypeError:
        print("Revenue and costs must be numbers")
        return None
    else:
        # Only runs if calculation succeeded
        print(f"Profit margin: {margin:.2f}%")
        try:
            with open(filename, 'w') as f:
                f.write(f"Profit: {profit}\nMargin: {margin}%")
        except IOError:
            print("Could not save to file")
    finally:
        # Always runs
        print("Calculation attempt completed")
    
    return margin
```

## Raising Exceptions

You can raise exceptions intentionally to signal errors in your code.

### Basic Raise
```python
def set_client_package(package):
    valid_packages = ["Basic", "Premium", "Enterprise"]
    
    if package not in valid_packages:
        raise ValueError(f"Invalid package. Choose from: {valid_packages}")
    
    return package

# This will raise ValueError
set_client_package("Free")
```

### Raise with Custom Messages
```python
def process_payment(amount):
    if amount <= 0:
        raise ValueError("Payment amount must be positive")
    
    if amount > 100000:
        raise ValueError("Payment amount exceeds maximum limit")
    
    print(f"Processing ${amount}")
```

### Re-raising Exceptions
```python
def critical_operation():
    try:
        risky_function()
    except Exception as e:
        log_error(e)  # Log the error
        raise  # Re-raise the same exception
```

## Custom Exceptions

Create your own exception classes for specific use cases.
```python
class InsufficientFundsError(Exception):
    """Raised when account has insufficient funds"""
    pass

class InvalidPackageError(Exception):
    """Raised when an invalid package is selected"""
    def __init__(self, package, valid_packages):
        self.package = package
        self.valid_packages = valid_packages
        super().__init__(f"'{package}' is not valid. Choose from: {valid_packages}")

def upgrade_package(current_package, new_package):
    valid_packages = ["Basic", "Premium", "Enterprise"]
    
    if new_package not in valid_packages:
        raise InvalidPackageError(new_package, valid_packages)
    
    print(f"Upgraded from {current_package} to {new_package}")

try:
    upgrade_package("Basic", "Ultimate")
except InvalidPackageError as e:
    print(f"Error: {e}")
    print(f"You tried: {e.package}")
```

## Exception Hierarchy

Python's exceptions are organized in a hierarchy. Catching a parent exception catches all its children.
```python
# BaseException (top of hierarchy)
#   ├── Exception (most user-defined exceptions inherit from this)
#   │   ├── ArithmeticError
#   │   │   ├── ZeroDivisionError
#   │   │   └── OverflowError
#   │   ├── LookupError
#   │   │   ├── IndexError
#   │   │   └── KeyError
#   │   ├── ValueError
#   │   └── TypeError
#   └── KeyboardInterrupt
```

### Catching Parent Exceptions
```python
try:
    risky_operation()
except ArithmeticError:
    # Catches ZeroDivisionError, OverflowError, etc.
    print("Math error occurred")
```

### Generic Exception Handling
```python
try:
    complex_operation()
except Exception as e:
    # Catches almost all exceptions (but not KeyboardInterrupt, SystemExit)
    print(f"An error occurred: {e}")
```

**Warning:** Avoid using bare `except:` as it catches everything, including KeyboardInterrupt, making it hard to stop your program.
```python
# DON'T DO THIS
try:
    operation()
except:  # Too broad!
    pass

# DO THIS INSTEAD
try:
    operation()
except Exception as e:
    print(f"Error: {e}")
```

## Best Practices for Error Handling

### 1. Be Specific with Exceptions

Catch specific exceptions rather than generic ones.
```python
# Bad
try:
    client_id = int(client_input)
except:
    print("Error")

# Good
try:
    client_id = int(client_input)
except ValueError:
    print("Client ID must be a number")
```

### 2. Don't Silence Errors

Always handle errors meaningfully or let them propagate.
```python
# Bad - silently hides all errors
try:
    process_data()
except:
    pass

# Good - log and handle appropriately
try:
    process_data()
except DataProcessingError as e:
    logger.error(f"Data processing failed: {e}")
    notify_admin(e)
```

### 3. Use Finally for Cleanup
```python
def upload_client_data(data):
    connection = None
    try:
        connection = database.connect()
        connection.upload(data)
    except ConnectionError:
        print("Failed to connect to database")
    finally:
        if connection:
            connection.close()
```

### 4. Provide Context in Error Messages
```python
def calculate_commission(sales, rate):
    if not 0 <= rate <= 1:
        raise ValueError(
            f"Commission rate must be between 0 and 1, got {rate}"
        )
    return sales * rate
```

### 5. Use Context Managers

Context managers automatically handle cleanup with `with` statements.
```python
# Automatically closes file, even if error occurs
with open('clients.txt', 'r') as file:
    data = file.read()
    process_data(data)
```

## Practical Example: Robust Client Onboarding System
```python
class ClientOnboardingError(Exception):
    """Base exception for client onboarding"""
    pass

class InvalidEmailError(ClientOnboardingError):
    """Invalid email format"""
    pass

class DuplicateClientError(ClientOnboardingError):
    """Client already exists"""
    pass

class ClientOnboardingSystem:
    def __init__(self):
        self.clients = []
    
    def validate_email(self, email):
        """Validate email format"""
        if '@' not in email or '.' not in email:
            raise InvalidEmailError(f"Invalid email format: {email}")
    
    def check_duplicate(self, email):
        """Check if client already exists"""
        if email in [client['email'] for client in self.clients]:
            raise DuplicateClientError(f"Client with email {email} already exists")
    
    def add_client(self, name, email, package):
        """Add new client with comprehensive error handling"""
        try:
            # Validation steps
            if not name or not email:
                raise ValueError("Name and email are required")
            
            self.validate_email(email)
            self.check_duplicate(email)
            
            # Validate package
            valid_packages = ["Basic", "Premium", "Enterprise"]
            if package not in valid_packages:
                raise ValueError(f"Package must be one of: {valid_packages}")
            
            # Add client
            client = {
                "name": name,
                "email": email,
                "package": package
            }
            self.clients.append(client)
            
        except InvalidEmailError as e:
            print(f"❌ Email Error: {e}")
            return False
        except DuplicateClientError as e:
            print(f"❌ Duplicate Error: {e}")
            return False
        except ValueError as e:
            print(f"❌ Validation Error: {e}")
            return False
        except Exception as e:
            print(f"❌ Unexpected Error: {e}")
            return False
        else:
            print(f"✅ Successfully onboarded {name} ({package} package)")
            return True
        finally:
            print(f"Total clients: {len(self.clients)}")

# Usage
system = ClientOnboardingSystem()

system.add_client("Dr. Sabine Charles", "sabine@example.com", "Premium")
system.add_client("", "invalid", "Free")  # Multiple errors
system.add_client("Dr. Sabine Charles", "sabine@example.com", "Premium")  # Duplicate
```

## Debugging with Exception Information

### Getting Exception Details
```python
import traceback
import sys

try:
    problematic_function()
except Exception as e:
    # Get exception type
    print(f"Exception type: {type(e).__name__}")
    
    # Get exception message
    print(f"Exception message: {str(e)}")
    
    # Get full traceback
    print("\nFull traceback:")
    traceback.print_exc()
    
    # Get exception info
    exc_type, exc_value, exc_traceback = sys.exc_info()
    print(f"Line number: {exc_traceback.tb_lineno}")
```

### Logging Exceptions
```python
import logging

logging.basicConfig(level=logging.ERROR, filename='errors.log')

try:
    process_payment(amount)
except Exception as e:
    logging.error(f"Payment processing failed: {e}", exc_info=True)
    raise
```

## Advanced Pattern: EAFP vs LBYL

Python follows **EAFP** (Easier to Ask for Forgiveness than Permission) rather than **LBYL** (Look Before You Leap).

### LBYL (Not Pythonic)
```python
# Check before acting
if 'email' in client_data:
    email = client_data['email']
else:
    email = None
```

### EAFP (Pythonic)
```python
# Try and handle if it fails
try:
    email = client_data['email']
except KeyError:
    email = None
```

EAFP is preferred because:
- It's more readable
- It handles race conditions better
- It's often faster (no redundant checks)

## Summary

Error handling transforms fragile code into robust systems. The key principles:

**Anticipate failures.** Think about what could go wrong.

**Be specific.** Catch specific exceptions, not generic ones.

**Provide context.** Error messages should be actionable.

**Clean up resources.** Use `finally` or context managers.

**Fail gracefully.** Handle errors in ways that maintain user trust.

**Log strategically.** Record errors for debugging without exposing sensitive data.

Master these patterns and you'll build Python applications that don't just work when everything goes right—they handle the unexpected with grace and professionalism.