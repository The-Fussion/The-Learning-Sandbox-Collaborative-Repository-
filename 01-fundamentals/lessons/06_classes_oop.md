# Object-Oriented Programming with Python Classes: A Complete Guide

Object-Oriented Programming (OOP) is a programming paradigm that organizes code around objects rather than functions and logic. It's the foundation of building scalable, maintainable software systems—the kind that power billion-dollar companies.

## Why OOP Matters

OOP allows you to:

- Model real-world entities and relationships
- Organize complex systems into manageable pieces
- Reuse code efficiently
- Build scalable applications
- Create clean, maintainable codebases

Think of OOP as creating blueprints for your business logic. Just as you'd have a blueprint for onboarding clients, you create classes that define how clients behave in your system.

## What is a Class?

A **class** is a blueprint for creating objects. It defines attributes (data) and methods (behaviors) that objects created from the class will have.

An **object** is an instance of a class—a specific realization of that blueprint.
```python
# Class: The blueprint
class Client:
    pass

# Objects: Specific instances
client1 = Client()
client2 = Client()
```

## Creating Your First Class

### Basic Class Structure
```python
class Client:
    """A class representing a client"""
    
    def __init__(self, name, email):
        """Constructor method - runs when object is created"""
        self.name = name
        self.email = email
    
    def greet(self):
        """Instance method"""
        return f"Hello, I'm {self.name}"

# Creating objects
client1 = Client("Dr. Sabine Charles", "sabine@example.com")
client2 = Client("Coach Martinez", "martinez@example.com")

print(client1.name)      # Dr. Sabine Charles
print(client1.greet())   # Hello, I'm Dr. Sabine Charles
print(client2.greet())   # Hello, I'm Coach Martinez
```

### Understanding `self`

`self` refers to the instance of the class itself. It's how methods access the object's attributes and other methods.
```python
class BrandPackage:
    def __init__(self, name, price):
        self.name = name      # Instance attribute
        self.price = price    # Instance attribute
    
    def display_info(self):
        # self allows access to instance attributes
        return f"{self.name}: ${self.price}"

package = BrandPackage("Premium Branding", 5000)
print(package.display_info())  # Premium Branding: $5000
```

## Instance Attributes vs Class Attributes

### Instance Attributes

Unique to each object, defined in `__init__`.
```python
class Client:
    def __init__(self, name, package):
        self.name = name          # Instance attribute
        self.package = package    # Instance attribute

client1 = Client("Dr. Charles", "Premium")
client2 = Client("Coach Smith", "Basic")

print(client1.name)    # Dr. Charles
print(client2.name)    # Coach Smith (different!)
```

### Class Attributes

Shared across all instances of the class.
```python
class Client:
    # Class attribute - shared by all instances
    total_clients = 0
    agency_name = "Oversubscribed"
    
    def __init__(self, name):
        self.name = name                    # Instance attribute
        Client.total_clients += 1           # Modify class attribute

client1 = Client("Dr. Charles")
client2 = Client("Coach Smith")
client3 = Client("Sarah Johnson")

print(Client.total_clients)        # 3
print(client1.total_clients)       # 3 (accessible from instance)
print(Client.agency_name)          # Oversubscribed
```

## Methods in Classes

### Instance Methods

Most common type—operate on instance data.
```python
class BrandStrategy:
    def __init__(self, client_name, industry):
        self.client_name = client_name
        self.industry = industry
        self.deliverables = []
    
    def add_deliverable(self, deliverable):
        """Instance method - modifies instance data"""
        self.deliverables.append(deliverable)
    
    def get_summary(self):
        """Instance method - reads instance data"""
        return f"{self.client_name} ({self.industry}): {len(self.deliverables)} deliverables"

strategy = BrandStrategy("Dr. Charles", "Coaching")
strategy.add_deliverable("Brand Guide")
strategy.add_deliverable("Content Calendar")
print(strategy.get_summary())  # Dr. Charles (Coaching): 2 deliverables
```

### Class Methods

Operate on class-level data, use `@classmethod` decorator. First parameter is `cls` (the class itself).
```python
class Client:
    total_clients = 0
    
    def __init__(self, name):
        self.name = name
        Client.total_clients += 1
    
    @classmethod
    def get_total_clients(cls):
        """Class method - operates on class data"""
        return cls.total_clients
    
    @classmethod
    def create_premium_client(cls, name):
        """Class method as alternative constructor"""
        client = cls(name)
        client.package = "Premium"
        return client

# Using class methods
client1 = Client("John")
client2 = Client.create_premium_client("Dr. Charles")

print(Client.get_total_clients())    # 2
print(client2.package)               # Premium
```

### Static Methods

Don't access instance or class data, use `@staticmethod` decorator. They're utility functions that belong to the class logically.
```python
class EmailValidator:
    @staticmethod
    def is_valid_email(email):
        """Static method - doesn't need instance or class data"""
        return '@' in email and '.' in email.split('@')[1]
    
    @staticmethod
    def format_email(email):
        """Static method - utility function"""
        return email.lower().strip()

# Using static methods (no instance needed)
print(EmailValidator.is_valid_email("sabine@example.com"))  # True
print(EmailValidator.format_email("  JOHN@EXAMPLE.COM  "))  # john@example.com
```

## The Four Pillars of OOP

### 1. Encapsulation

Bundling data and methods that operate on that data within a single unit (class). Hide internal details and expose only what's necessary.
```python
class BankAccount:
    def __init__(self, owner, balance):
        self.owner = owner
        self.__balance = balance  # Private attribute (convention: __ prefix)
    
    def deposit(self, amount):
        """Public method to modify private data"""
        if amount > 0:
            self.__balance += amount
            return True
        return False
    
    def withdraw(self, amount):
        """Public method with validation"""
        if 0 < amount <= self.__balance:
            self.__balance -= amount
            return True
        return False
    
    def get_balance(self):
        """Controlled access to private data"""
        return self.__balance

account = BankAccount("OnSwift", 10000)
account.deposit(5000)
print(account.get_balance())     # 15000

# Cannot access private attribute directly (Python mangles the name)
# print(account.__balance)       # AttributeError
```

**Access Modifiers Convention:**
```python
class Example:
    def __init__(self):
        self.public = "Accessible anywhere"
        self._protected = "Should not be accessed outside class (convention)"
        self.__private = "Name mangled to prevent easy access"
```

### 2. Inheritance

Create new classes based on existing ones, inheriting their attributes and methods.
```python
# Parent/Base class
class Service:
    def __init__(self, name, price):
        self.name = name
        self.price = price
    
    def get_info(self):
        return f"{self.name}: ${self.price}"

# Child/Derived class
class BrandingService(Service):
    def __init__(self, name, price, deliverables):
        super().__init__(name, price)  # Call parent constructor
        self.deliverables = deliverables
    
    def get_info(self):
        # Override parent method
        base_info = super().get_info()
        return f"{base_info} ({len(self.deliverables)} deliverables)"

class ConsultingService(Service):
    def __init__(self, name, price, hours):
        super().__init__(name, price)
        self.hours = hours
    
    def hourly_rate(self):
        return self.price / self.hours

# Using inheritance
branding = BrandingService("Premium Brand Package", 5000, ["Logo", "Brand Guide", "Templates"])
consulting = ConsultingService("Strategy Session", 2000, 4)

print(branding.get_info())         # Premium Brand Package: $5000 (3 deliverables)
print(consulting.hourly_rate())    # 500.0
```

**Multiple Inheritance:**
```python
class Marketer:
    def create_campaign(self):
        return "Campaign created"

class Designer:
    def create_design(self):
        return "Design created"

class BrandStrategist(Marketer, Designer):
    def create_brand(self):
        campaign = self.create_campaign()
        design = self.create_design()
        return f"{campaign} and {design}"

strategist = BrandStrategist()
print(strategist.create_brand())  # Campaign created and Design created
```

### 3. Polymorphism

The ability of different classes to be used through the same interface. Same method name, different implementations.
```python
class PaymentProcessor:
    def process(self, amount):
        raise NotImplementedError("Subclass must implement process()")

class StripeProcessor(PaymentProcessor):
    def process(self, amount):
        return f"Processing ${amount} via Stripe"

class PayPalProcessor(PaymentProcessor):
    def process(self, amount):
        return f"Processing ${amount} via PayPal"

class BankTransferProcessor(PaymentProcessor):
    def process(self, amount):
        return f"Processing ${amount} via Bank Transfer"

# Polymorphism in action
def handle_payment(processor, amount):
    # Same method call, different behavior based on object type
    print(processor.process(amount))

# All work with the same interface
stripe = StripeProcessor()
paypal = PayPalProcessor()
bank = BankTransferProcessor()

handle_payment(stripe, 5000)   # Processing $5000 via Stripe
handle_payment(paypal, 3000)   # Processing $3000 via PayPal
handle_payment(bank, 10000)    # Processing $10000 via Bank Transfer
```

### 4. Abstraction

Hiding complex implementation details and showing only essential features.
```python
from abc import ABC, abstractmethod

class PaymentGateway(ABC):
    """Abstract base class - cannot be instantiated"""
    
    @abstractmethod
    def connect(self):
        """Must be implemented by subclasses"""
        pass
    
    @abstractmethod
    def charge(self, amount):
        """Must be implemented by subclasses"""
        pass
    
    def log_transaction(self, amount):
        """Concrete method - inherited by all subclasses"""
        print(f"Transaction of ${amount} logged")

class StripeGateway(PaymentGateway):
    def connect(self):
        return "Connected to Stripe API"
    
    def charge(self, amount):
        self.log_transaction(amount)
        return f"Charged ${amount} via Stripe"

# Cannot instantiate abstract class
# gateway = PaymentGateway()  # TypeError

# Can instantiate concrete implementation
stripe = StripeGateway()
print(stripe.connect())      # Connected to Stripe API
print(stripe.charge(1000))   # Transaction of $1000 logged
                             # Charged $1000 via Stripe
```

## Special Methods (Magic Methods/Dunder Methods)

Methods with double underscores that give classes special behaviors.

### `__init__` - Constructor
```python
class Client:
    def __init__(self, name, email):
        """Called when object is created"""
        self.name = name
        self.email = email
```

### `__str__` and `__repr__`
```python
class Client:
    def __init__(self, name, package):
        self.name = name
        self.package = package
    
    def __str__(self):
        """User-friendly string representation"""
        return f"{self.name} ({self.package} package)"
    
    def __repr__(self):
        """Developer-friendly representation"""
        return f"Client(name='{self.name}', package='{self.package}')"

client = Client("Dr. Charles", "Premium")
print(str(client))    # Dr. Charles (Premium package)
print(repr(client))   # Client(name='Dr. Charles', package='Premium')
print(client)         # Dr. Charles (Premium package) (uses __str__)
```

### `__len__`
```python
class ClientPortfolio:
    def __init__(self):
        self.clients = []
    
    def add_client(self, client):
        self.clients.append(client)
    
    def __len__(self):
        """Enable len() function"""
        return len(self.clients)

portfolio = ClientPortfolio()
portfolio.add_client("Client 1")
portfolio.add_client("Client 2")
print(len(portfolio))  # 2
```

### `__getitem__` and `__setitem__`
```python
class ClientDatabase:
    def __init__(self):
        self.clients = {}
    
    def __getitem__(self, client_id):
        """Enable bracket notation for getting"""
        return self.clients[client_id]
    
    def __setitem__(self, client_id, client_data):
        """Enable bracket notation for setting"""
        self.clients[client_id] = client_data

db = ClientDatabase()
db[1] = {"name": "Dr. Charles", "package": "Premium"}
db[2] = {"name": "Coach Smith", "package": "Basic"}

print(db[1])  # {'name': 'Dr. Charles', 'package': 'Premium'}
```

### Comparison Methods
```python
class BrandPackage:
    def __init__(self, name, price):
        self.name = name
        self.price = price
    
    def __eq__(self, other):
        """Equal to (==)"""
        return self.price == other.price
    
    def __lt__(self, other):
        """Less than (<)"""
        return self.price < other.price
    
    def __le__(self, other):
        """Less than or equal (<=)"""
        return self.price <= other.price
    
    def __gt__(self, other):
        """Greater than (>)"""
        return self.price > other.price

basic = BrandPackage("Basic", 1000)
premium = BrandPackage("Premium", 5000)
enterprise = BrandPackage("Enterprise", 10000)

print(premium > basic)        # True
print(basic == premium)       # False
print(enterprise >= premium)  # True
```

### Arithmetic Operations
```python
class Revenue:
    def __init__(self, amount):
        self.amount = amount
    
    def __add__(self, other):
        """Addition (+)"""
        return Revenue(self.amount + other.amount)
    
    def __sub__(self, other):
        """Subtraction (-)"""
        return Revenue(self.amount - other.amount)
    
    def __str__(self):
        return f"${self.amount:,.2f}"

q1_revenue = Revenue(50000)
q2_revenue = Revenue(75000)
total = q1_revenue + q2_revenue

print(total)  # $125,000.00
```

## Properties and Getters/Setters

Properties provide controlled access to attributes.

### Using `@property`
```python
class Client:
    def __init__(self, name, monthly_revenue):
        self.name = name
        self._monthly_revenue = monthly_revenue
    
    @property
    def monthly_revenue(self):
        """Getter - access like an attribute"""
        return self._monthly_revenue
    
    @monthly_revenue.setter
    def monthly_revenue(self, value):
        """Setter - with validation"""
        if value < 0:
            raise ValueError("Revenue cannot be negative")
        self._monthly_revenue = value
    
    @property
    def annual_revenue(self):
        """Computed property"""
        return self._monthly_revenue * 12

client = Client("Dr. Charles", 10000)

# Access like attributes, but with validation
print(client.monthly_revenue)    # 10000
client.monthly_revenue = 15000   # Uses setter
print(client.annual_revenue)     # 180000

# client.monthly_revenue = -5000  # ValueError
```

## Class Composition

Building complex classes by combining simpler ones (alternative to inheritance).
```python
class Address:
    def __init__(self, street, city, country):
        self.street = street
        self.city = city
        self.country = country
    
    def __str__(self):
        return f"{self.street}, {self.city}, {self.country}"

class ContactInfo:
    def __init__(self, email, phone):
        self.email = email
        self.phone = phone

class Client:
    def __init__(self, name, address, contact):
        self.name = name
        self.address = address      # Composition
        self.contact = contact      # Composition
    
    def get_full_info(self):
        return f"{self.name}\n{self.address}\n{self.contact.email}"

# Building through composition
address = Address("123 Innovation St", "Lagos", "Nigeria")
contact = ContactInfo("client@example.com", "+234-123-4567")
client = Client("Dr. Charles", address, contact)

print(client.get_full_info())
# Dr. Charles
# 123 Innovation St, Lagos, Nigeria
# client@example.com
```

## Real-World Example: Complete Client Management System
```python
from datetime import datetime
from enum import Enum

class PackageType(Enum):
    """Enum for package types"""
    BASIC = "Basic"
    PREMIUM = "Premium"
    ENTERPRISE = "Enterprise"

class Service:
    """Base class for all services"""
    service_count = 0
    
    def __init__(self, name, price):
        self.name = name
        self.price = price
        self.created_at = datetime.now()
        Service.service_count += 1
    
    def __str__(self):
        return f"{self.name}: ${self.price:,.2f}"
    
    @classmethod
    def get_service_count(cls):
        return cls.service_count

class BrandingService(Service):
    """Specialized service for branding"""
    def __init__(self, name, price, deliverables):
        super().__init__(name, price)
        self.deliverables = deliverables
    
    def add_deliverable(self, deliverable):
        self.deliverables.append(deliverable)
    
    def __len__(self):
        return len(self.deliverables)

class Client:
    """Client representation with full functionality"""
    total_clients = 0
    
    def __init__(self, name, email, package_type):
        self.name = name
        self.email = email
        self._package = package_type
        self.services = []
        self.created_at = datetime.now()
        Client.total_clients += 1
    
    @property
    def package(self):
        return self._package
    
    @package.setter
    def package(self, new_package):
        if not isinstance(new_package, PackageType):
            raise ValueError("Package must be a PackageType enum")
        print(f"Upgrading {self.name} from {self._package.value} to {new_package.value}")
        self._package = new_package
    
    def add_service(self, service):
        """Add service to client"""
        if not isinstance(service, Service):
            raise TypeError("Must be a Service instance")
        self.services.append(service)
    
    def get_total_investment(self):
        """Calculate total client investment"""
        return sum(service.price for service in self.services)
    
    def __str__(self):
        return f"{self.name} ({self._package.value})"
    
    def __repr__(self):
        return f"Client(name='{self.name}', email='{self.email}', package={self._package})"
    
    def __len__(self):
        """Number of services"""
        return len(self.services)
    
    @classmethod
    def create_premium_client(cls, name, email):
        """Factory method for premium clients"""
        return cls(name, email, PackageType.PREMIUM)
    
    @staticmethod
    def validate_email(email):
        """Validate email format"""
        return '@' in email and '.' in email.split('@')[1]

class ClientPortfolio:
    """Manages multiple clients"""
    def __init__(self, agency_name):
        self.agency_name = agency_name
        self.clients = []
    
    def add_client(self, client):
        """Add client with validation"""
        if not isinstance(client, Client):
            raise TypeError("Must be a Client instance")
        
        if not Client.validate_email(client.email):
            raise ValueError(f"Invalid email: {client.email}")
        
        self.clients.append(client)
    
    def get_total_revenue(self):
        """Calculate total revenue from all clients"""
        return sum(client.get_total_investment() for client in self.clients)
    
    def get_clients_by_package(self, package_type):
        """Filter clients by package"""
        return [c for c in self.clients if c.package == package_type]
    
    def __len__(self):
        return len(self.clients)
    
    def __getitem__(self, index):
        return self.clients[index]
    
    def __str__(self):
        return f"{self.agency_name}: {len(self.clients)} clients, ${self.get_total_revenue():,.2f} total revenue"

# Usage Example
if __name__ == "__main__":
    # Create portfolio
    portfolio = ClientPortfolio("Oversubscribed")
    
    # Create clients
    client1 = Client("Dr. Sabine Charles", "sabine@example.com", PackageType.PREMIUM)
    client2 = Client.create_premium_client("Coach Martinez", "martinez@example.com")
    client3 = Client("Sarah Johnson", "sarah@example.com", PackageType.BASIC)
    
    # Add services
    branding = BrandingService("Brand Identity Package", 5000, ["Logo", "Brand Guide"])
    branding.add_deliverable("Social Templates")
    
    consulting = Service("Strategy Consultation", 2000)
    
    client1.add_service(branding)
    client1.add_service(consulting)
    
    client2.add_service(Service("Content Calendar", 1500))
    
    # Add to portfolio
    portfolio.add_client(client1)
    portfolio.add_client(client2)
    portfolio.add_client(client3)
    
    # Display results
    print(portfolio)
    print(f"\nClient 1: {client1}")
    print(f"Total investment: ${client1.get_total_investment():,.2f}")
    print(f"Services: {len(client1)}")
    
    # Upgrade client
    client3.package = PackageType.PREMIUM
    
    # Statistics
    print(f"\nTotal clients: {Client.total_clients}")
    print(f"Total services created: {Service.get_service_count()}")
    print(f"Premium clients: {len(portfolio.get_clients_by_package(PackageType.PREMIUM))}")
```

## Best Practices for OOP in Python

### 1. Follow the Single Responsibility Principle

Each class should have one clear purpose.
```python
# Bad - too many responsibilities
class Client:
    def add_client(self):
        pass
    def send_email(self):
        pass
    def process_payment(self):
        pass
    def generate_report(self):
        pass

# Good - separate responsibilities
class Client:
    def __init__(self, name, email):
        self.name = name
        self.email = email

class EmailService:
    def send_email(self, client, message):
        pass

class PaymentProcessor:
    def process_payment(self, client, amount):
        pass

class ReportGenerator:
    def generate_report(self, clients):
        pass
```

### 2. Favor Composition Over Inheritance
```python
# Instead of deep inheritance hierarchies
class Animal:
    pass

class Mammal(Animal):
    pass

class Dog(Mammal):
    pass

# Use composition
class Walker:
    def walk(self):
        return "Walking"

class Swimmer:
    def swim(self):
        return "Swimming"

class Dog:
    def __init__(self):
        self.walker = Walker()
        self.swimmer = Swimmer()
```

### 3. Use Descriptive Names
```python
# Bad
class C:
    def p(self):
        pass

# Good
class Client:
    def process_payment(self):
        pass
```

### 4. Keep Classes Focused and Small

Aim for classes under 200-300 lines. If larger, consider splitting.

### 5. Use Type Hints
```python
from typing import List, Optional

class Client:
    def __init__(self, name: str, email: str) -> None:
        self.name = name
        self.email = email
        self.services: List[Service] = []
    
    def add_service(self, service: Service) -> None:
        self.services.append(service)
    
    def get_total_investment(self) -> float:
        return sum(s.price for s in self.services)
```

## Summary

Object-Oriented Programming with classes is the foundation of professional Python development. Master these concepts:

**Classes define blueprints.** Objects are instances of those blueprints.

**Encapsulation hides complexity.** Expose only what users need.

**Inheritance creates hierarchies.** Build on existing functionality.

**Polymorphism enables flexibility.** Same interface, different behaviors.

**Abstraction simplifies interfaces.** Hide implementation details.

**Special methods add magic.** Make your classes behave like built-in types.

**Composition builds complexity.** Combine simple objects into complex ones.

Start simple, refactor as you grow, and always prioritize readability and maintainability. Classes are tools for organizing complexity—use them to build systems that scale from your first client to your thousandth.