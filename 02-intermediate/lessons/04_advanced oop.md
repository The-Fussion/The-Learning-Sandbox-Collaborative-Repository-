# Python Advanced OOP: A Detailed Guide

## Table of Contents
1. [Inheritance Basics](#inheritance-basics)
2. [Method Resolution Order (MRO)](#method-resolution-order-mro)
3. [Multiple Inheritance](#multiple-inheritance)
4. [Abstract Base Classes (ABC)](#abstract-base-classes-abc)
5. [Dunder Methods (Magic Methods)](#dunder-methods-magic-methods)
6. [Composition vs Inheritance](#composition-vs-inheritance)
7. [Advanced Inheritance Patterns](#advanced-inheritance-patterns)
8. [Best Practices](#best-practices)

---

## Inheritance Basics

Inheritance allows a class to inherit attributes and methods from another class, promoting code reuse and establishing relationships between classes.

### Simple Inheritance

```python
class Animal:
    def __init__(self, name):
        self.name = name
    
    def speak(self):
        return "Some sound"
    
    def info(self):
        return f"I am {self.name}"

class Dog(Animal):
    def speak(self):  # Method overriding
        return "Woof!"
    
    def fetch(self):  # New method
        return f"{self.name} is fetching"

# Usage
dog = Dog("Buddy")
print(dog.speak())     # Woof! (overridden)
print(dog.info())      # I am Buddy (inherited)
print(dog.fetch())     # Buddy is fetching (new method)
```

### Using `super()`

```python
class Animal:
    def __init__(self, name, species):
        self.name = name
        self.species = species
    
    def describe(self):
        return f"{self.name} is a {self.species}"

class Dog(Animal):
    def __init__(self, name, breed):
        # Call parent constructor
        super().__init__(name, "Dog")
        self.breed = breed
    
    def describe(self):
        # Extend parent method
        base_description = super().describe()
        return f"{base_description}, breed: {self.breed}"

dog = Dog("Max", "Golden Retriever")
print(dog.describe())
# Output: Max is a Dog, breed: Golden Retriever
```

### Checking Inheritance

```python
class Animal:
    pass

class Dog(Animal):
    pass

class Cat(Animal):
    pass

dog = Dog()

# Check instance type
print(isinstance(dog, Dog))      # True
print(isinstance(dog, Animal))   # True
print(isinstance(dog, Cat))      # False

# Check class inheritance
print(issubclass(Dog, Animal))   # True
print(issubclass(Dog, Cat))      # False
```

---

## Method Resolution Order (MRO)

MRO determines the order in which Python searches for methods in a class hierarchy. Python uses the C3 Linearization algorithm.

### Understanding MRO

```python
class A:
    def method(self):
        print("A.method")

class B(A):
    def method(self):
        print("B.method")

class C(A):
    def method(self):
        print("C.method")

class D(B, C):
    pass

# Check MRO
print(D.__mro__)
# Output: (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)

# Or use mro() method
print(D.mro())

d = D()
d.method()  # Calls B.method (B comes before C in MRO)
```

### MRO with super()

```python
class A:
    def __init__(self):
        print("A.__init__")
        super().__init__()

class B(A):
    def __init__(self):
        print("B.__init__")
        super().__init__()

class C(A):
    def __init__(self):
        print("C.__init__")
        super().__init__()

class D(B, C):
    def __init__(self):
        print("D.__init__")
        super().__init__()

print("Creating D instance:")
d = D()

# Output:
# Creating D instance:
# D.__init__
# B.__init__
# C.__init__
# A.__init__

print("\nMRO:", [cls.__name__ for cls in D.__mro__])
# MRO: ['D', 'B', 'C', 'A', 'object']
```

### Diamond Problem

```python
class Base:
    def method(self):
        print("Base.method")

class Left(Base):
    def method(self):
        print("Left.method")
        super().method()

class Right(Base):
    def method(self):
        print("Right.method")
        super().method()

class Child(Left, Right):
    def method(self):
        print("Child.method")
        super().method()

child = Child()
child.method()

# Output:
# Child.method
# Left.method
# Right.method
# Base.method

# MRO ensures Base.method is called only once
print(Child.__mro__)
# (<class 'Child'>, <class 'Left'>, <class 'Right'>, <class 'Base'>, <class 'object'>)
```

---

## Multiple Inheritance

Python supports multiple inheritance, where a class can inherit from multiple parent classes.

### Basic Multiple Inheritance

```python
class Flyer:
    def fly(self):
        return "Flying high!"

class Swimmer:
    def swim(self):
        return "Swimming deep!"

class Duck(Flyer, Swimmer):
    def quack(self):
        return "Quack!"

duck = Duck()
print(duck.fly())    # Flying high!
print(duck.swim())   # Swimming deep!
print(duck.quack())  # Quack!
```

### Mixin Pattern

Mixins are classes that provide specific functionality to be inherited by other classes.

```python
class JSONMixin:
    """Mixin to add JSON serialization"""
    def to_json(self):
        import json
        return json.dumps(self.__dict__)

class XMLMixin:
    """Mixin to add XML serialization"""
    def to_xml(self):
        items = ''.join(f'<{k}>{v}</{k}>' for k, v in self.__dict__.items())
        return f'<object>{items}</object>'

class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

class SerializablePerson(Person, JSONMixin, XMLMixin):
    pass

person = SerializablePerson("Alice", 30)
print(person.to_json())
# Output: {"name": "Alice", "age": 30}

print(person.to_xml())
# Output: <object><name>Alice</name><age>30</age></object>
```

### Avoiding Multiple Inheritance Issues

```python
class LoggerMixin:
    def log(self, message):
        print(f"[{self.__class__.__name__}] {message}")

class ValidatorMixin:
    def validate(self):
        if not hasattr(self, 'name') or not self.name:
            raise ValueError("Name is required")
        return True

class User(LoggerMixin, ValidatorMixin):
    def __init__(self, name):
        self.name = name
    
    def save(self):
        self.validate()
        self.log(f"Saving user: {self.name}")

user = User("Alice")
user.save()
# Output: [User] Saving user: Alice

# Invalid user
try:
    invalid_user = User("")
    invalid_user.save()
except ValueError as e:
    print(f"Error: {e}")
# Output: Error: Name is required
```

---

## Abstract Base Classes (ABC)

Abstract Base Classes define interfaces and enforce that derived classes implement certain methods.

### Basic ABC

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self):
        """Calculate the area of the shape"""
        pass
    
    @abstractmethod
    def perimeter(self):
        """Calculate the perimeter of the shape"""
        pass
    
    def describe(self):
        """Non-abstract method with default implementation"""
        return f"I am a {self.__class__.__name__}"

# Cannot instantiate abstract class
# shape = Shape()  # TypeError: Can't instantiate abstract class

class Rectangle(Shape):
    def __init__(self, width, height):
        self.width = width
        self.height = height
    
    def area(self):
        return self.width * self.height
    
    def perimeter(self):
        return 2 * (self.width + self.height)

rect = Rectangle(5, 3)
print(rect.area())        # 15
print(rect.perimeter())   # 16
print(rect.describe())    # I am a Rectangle
```

### Abstract Properties

```python
from abc import ABC, abstractmethod

class Vehicle(ABC):
    @property
    @abstractmethod
    def max_speed(self):
        """Must be implemented by subclasses"""
        pass
    
    @abstractmethod
    def start_engine(self):
        pass

class Car(Vehicle):
    def __init__(self, speed):
        self._max_speed = speed
    
    @property
    def max_speed(self):
        return self._max_speed
    
    def start_engine(self):
        return "Car engine started"

car = Car(200)
print(car.max_speed)       # 200
print(car.start_engine())  # Car engine started
```

### Multiple Abstract Base Classes

```python
from abc import ABC, abstractmethod

class Drawable(ABC):
    @abstractmethod
    def draw(self):
        pass

class Movable(ABC):
    @abstractmethod
    def move(self, x, y):
        pass

class GameObject(Drawable, Movable):
    def __init__(self, name):
        self.name = name
        self.x = 0
        self.y = 0
    
    def draw(self):
        return f"Drawing {self.name} at ({self.x}, {self.y})"
    
    def move(self, x, y):
        self.x = x
        self.y = y
        return f"{self.name} moved to ({self.x}, {self.y})"

obj = GameObject("Player")
print(obj.draw())           # Drawing Player at (0, 0)
print(obj.move(10, 20))     # Player moved to (10, 20)
print(obj.draw())           # Drawing Player at (10, 20)
```

---

## Dunder Methods (Magic Methods)

Dunder (double underscore) methods allow you to define how objects behave with built-in operations.

### String Representation

```python
class Book:
    def __init__(self, title, author, pages):
        self.title = title
        self.author = author
        self.pages = pages
    
    def __str__(self):
        """String representation for users (used by print())"""
        return f"{self.title} by {self.author}"
    
    def __repr__(self):
        """String representation for developers (used by repr())"""
        return f"Book('{self.title}', '{self.author}', {self.pages})"

book = Book("1984", "George Orwell", 328)

print(str(book))   # 1984 by George Orwell
print(repr(book))  # Book('1984', 'George Orwell', 328)

# In interactive mode or print()
print(book)        # Uses __str__

# In a list or direct reference
print([book])      # Uses __repr__
```

### Comparison Methods

```python
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age
    
    def __eq__(self, other):
        """Equal to (==)"""
        if not isinstance(other, Person):
            return False
        return self.age == other.age
    
    def __lt__(self, other):
        """Less than (<)"""
        return self.age < other.age
    
    def __le__(self, other):
        """Less than or equal to (<=)"""
        return self.age <= other.age
    
    def __gt__(self, other):
        """Greater than (>)"""
        return self.age > other.age
    
    def __ge__(self, other):
        """Greater than or equal to (>=)"""
        return self.age >= other.age
    
    def __repr__(self):
        return f"Person('{self.name}', {self.age})"

p1 = Person("Alice", 30)
p2 = Person("Bob", 25)
p3 = Person("Charlie", 30)

print(p1 == p3)  # True (same age)
print(p1 > p2)   # True (30 > 25)
print(p2 < p1)   # True (25 < 30)

# Sorting
people = [p1, p2, p3]
print(sorted(people))
# [Person('Bob', 25), Person('Alice', 30), Person('Charlie', 30)]
```

### Container Methods

```python
class ShoppingCart:
    def __init__(self):
        self.items = []
    
    def __len__(self):
        """Length of container (len())"""
        return len(self.items)
    
    def __getitem__(self, index):
        """Get item by index (cart[index])"""
        return self.items[index]
    
    def __setitem__(self, index, value):
        """Set item by index (cart[index] = value)"""
        self.items[index] = value
    
    def __delitem__(self, index):
        """Delete item by index (del cart[index])"""
        del self.items[index]
    
    def __contains__(self, item):
        """Check membership (item in cart)"""
        return item in self.items
    
    def __iter__(self):
        """Make iterable (for item in cart)"""
        return iter(self.items)
    
    def add(self, item):
        self.items.append(item)

cart = ShoppingCart()
cart.add("Apple")
cart.add("Banana")
cart.add("Orange")

print(len(cart))           # 3
print(cart[0])             # Apple
print("Banana" in cart)    # True

for item in cart:
    print(item)
# Apple
# Banana
# Orange
```

### Arithmetic Methods

```python
class Vector:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    
    def __add__(self, other):
        """Addition (+)"""
        return Vector(self.x + other.x, self.y + other.y)
    
    def __sub__(self, other):
        """Subtraction (-)"""
        return Vector(self.x - other.x, self.y - other.y)
    
    def __mul__(self, scalar):
        """Multiplication (*)"""
        return Vector(self.x * scalar, self.y * scalar)
    
    def __truediv__(self, scalar):
        """Division (/)"""
        return Vector(self.x / scalar, self.y / scalar)
    
    def __abs__(self):
        """Absolute value (abs())"""
        return (self.x ** 2 + self.y ** 2) ** 0.5
    
    def __repr__(self):
        return f"Vector({self.x}, {self.y})"

v1 = Vector(3, 4)
v2 = Vector(1, 2)

print(v1 + v2)      # Vector(4, 6)
print(v1 - v2)      # Vector(2, 2)
print(v1 * 2)       # Vector(6, 8)
print(v1 / 2)       # Vector(1.5, 2.0)
print(abs(v1))      # 5.0
```

### Context Manager Methods

```python
class FileHandler:
    def __init__(self, filename, mode):
        self.filename = filename
        self.mode = mode
        self.file = None
    
    def __enter__(self):
        """Called when entering 'with' block"""
        print(f"Opening {self.filename}")
        self.file = open(self.filename, self.mode)
        return self.file
    
    def __exit__(self, exc_type, exc_value, traceback):
        """Called when exiting 'with' block"""
        print(f"Closing {self.filename}")
        if self.file:
            self.file.close()
        return False  # Don't suppress exceptions

with FileHandler('test.txt', 'w') as f:
    f.write("Hello, World!")

# Output:
# Opening test.txt
# Closing test.txt
```

### Callable Objects

```python
class Multiplier:
    def __init__(self, factor):
        self.factor = factor
    
    def __call__(self, x):
        """Make object callable like a function"""
        return x * self.factor

times_three = Multiplier(3)
print(times_three(10))  # 30
print(times_three(5))   # 15

# It's an object but acts like a function
print(callable(times_three))  # True
```

---

## Composition vs Inheritance

Composition means building complex objects by combining simpler objects, rather than inheriting from them.

### When to Use Inheritance

Use inheritance for **"is-a"** relationships:

```python
class Animal:
    def __init__(self, name):
        self.name = name
    
    def eat(self):
        return f"{self.name} is eating"

class Dog(Animal):  # Dog IS-A Animal
    def bark(self):
        return f"{self.name} says Woof!"

dog = Dog("Buddy")
print(dog.eat())   # Buddy is eating
print(dog.bark())  # Buddy says Woof!
```

### When to Use Composition

Use composition for **"has-a"** relationships:

```python
class Engine:
    def __init__(self, horsepower):
        self.horsepower = horsepower
    
    def start(self):
        return "Engine started"
    
    def stop(self):
        return "Engine stopped"

class Wheels:
    def __init__(self, count):
        self.count = count
    
    def rotate(self):
        return f"{self.count} wheels rotating"

class Car:  # Car HAS-A Engine and HAS-A Wheels
    def __init__(self, model, horsepower, wheel_count):
        self.model = model
        self.engine = Engine(horsepower)  # Composition
        self.wheels = Wheels(wheel_count)  # Composition
    
    def start(self):
        return f"{self.model}: {self.engine.start()}"
    
    def drive(self):
        return f"{self.model}: {self.wheels.rotate()}"

car = Car("Tesla Model 3", 300, 4)
print(car.start())  # Tesla Model 3: Engine started
print(car.drive())  # Tesla Model 3: 4 wheels rotating
```

### Composition Over Inheritance

```python
# Bad: Using inheritance for behavior
class LoggingList(list):  # Inheriting from list
    def append(self, item):
        print(f"Adding {item}")
        super().append(item)

# Better: Using composition
class LoggingList:
    def __init__(self):
        self._list = []  # Composition
    
    def append(self, item):
        print(f"Adding {item}")
        self._list.append(item)
    
    def __getitem__(self, index):
        return self._list[index]
    
    def __len__(self):
        return len(self._list)

log_list = LoggingList()
log_list.append(1)  # Adding 1
log_list.append(2)  # Adding 2
```

### Flexible Design with Composition

```python
class Logger:
    def log(self, message):
        print(f"LOG: {message}")

class EmailSender:
    def send(self, to, message):
        print(f"Sending email to {to}: {message}")

class SMSSender:
    def send(self, to, message):
        print(f"Sending SMS to {to}: {message}")

class NotificationService:
    def __init__(self, sender, logger=None):
        self.sender = sender  # Composition
        self.logger = logger  # Optional composition
    
    def notify(self, recipient, message):
        if self.logger:
            self.logger.log(f"Notifying {recipient}")
        self.sender.send(recipient, message)

# Can easily swap implementations
email_service = NotificationService(EmailSender(), Logger())
email_service.notify("user@example.com", "Hello!")

sms_service = NotificationService(SMSSender())
sms_service.notify("+1234567890", "Hi!")

# Output:
# LOG: Notifying user@example.com
# Sending email to user@example.com: Hello!
# Sending SMS to +1234567890: Hi!
```

---

## Advanced Inheritance Patterns

### Template Method Pattern

```python
from abc import ABC, abstractmethod

class DataProcessor(ABC):
    """Template method pattern"""
    
    def process(self):
        """Template method - defines algorithm structure"""
        data = self.read_data()
        processed = self.transform_data(data)
        self.save_data(processed)
    
    @abstractmethod
    def read_data(self):
        pass
    
    @abstractmethod
    def transform_data(self, data):
        pass
    
    @abstractmethod
    def save_data(self, data):
        pass

class CSVProcessor(DataProcessor):
    def read_data(self):
        print("Reading CSV file")
        return [1, 2, 3, 4, 5]
    
    def transform_data(self, data):
        print("Transforming CSV data")
        return [x * 2 for x in data]
    
    def save_data(self, data):
        print(f"Saving CSV data: {data}")

processor = CSVProcessor()
processor.process()

# Output:
# Reading CSV file
# Transforming CSV data
# Saving CSV data: [2, 4, 6, 8, 10]
```

### Polymorphism

```python
class Animal:
    def speak(self):
        pass

class Dog(Animal):
    def speak(self):
        return "Woof!"

class Cat(Animal):
    def speak(self):
        return "Meow!"

class Bird(Animal):
    def speak(self):
        return "Tweet!"

def make_animal_speak(animal):
    """Polymorphism - works with any Animal subclass"""
    print(animal.speak())

animals = [Dog(), Cat(), Bird()]
for animal in animals:
    make_animal_speak(animal)

# Output:
# Woof!
# Meow!
# Tweet!
```

---

## Best Practices

### 1. Favor Composition Over Inheritance

```python
# Instead of inheriting everything
# Use composition for flexibility
```

### 2. Keep Inheritance Hierarchies Shallow

```python
# Good: Shallow hierarchy (2-3 levels max)
class Animal:
    pass

class Dog(Animal):
    pass

# Avoid: Deep hierarchies that are hard to maintain
```

### 3. Use Abstract Base Classes for Interfaces

```python
from abc import ABC, abstractmethod

class PaymentProcessor(ABC):
    @abstractmethod
    def process_payment(self, amount):
        pass

# Forces all implementations to define the method
```

### 4. Follow the Liskov Substitution Principle

```python
# Subclasses should be substitutable for their base classes
# If S is a subtype of T, objects of type T can be replaced
# with objects of type S without breaking the program
```

### 5. Use `super()` for Cooperative Inheritance

```python
# Always use super() to ensure proper MRO traversal
class Child(Parent):
    def __init__(self):
        super().__init__()  # Good
        # Parent.__init__(self)  # Avoid this
```

---

## Key Takeaways

- **Inheritance** creates "is-a" relationships and enables code reuse
- **MRO** (Method Resolution Order) determines method lookup order using C3 linearization
- **Multiple inheritance** is powerful but should be used carefully, often through mixins
- **Abstract Base Classes** define interfaces and enforce contracts
- **Dunder methods** customize object behavior with built-in operations
- **Composition** creates "has-a" relationships and is often more flexible than inheritance
- **Favor composition over inheritance** for better maintainability and flexibility
- Use **polymorphism** to write generic code that works with multiple types
- Keep inheritance hierarchies **shallow** and **focused**
- Always use **`super()`** for proper method resolution

Mastering advanced OOP concepts enables you to design flexible, maintainable, and elegant Python applications!