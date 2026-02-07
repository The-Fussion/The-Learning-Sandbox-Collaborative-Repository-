# Data Validation with Pydantic

## 5.1 Pydantic Basics

### BaseModel Fundamentals

Pydantic's `BaseModel` is the foundation for data validation and serialization in FastAPI.

```python
from pydantic import BaseModel

class User(BaseModel):
    id: int
    username: str
    email: str
    is_active: bool

# Create instance
user = User(id=1, username="john", email="john@example.com", is_active=True)
print(user.username)  # "john"
print(user.model_dump())  # Convert to dict
print(user.model_dump_json())  # Convert to JSON string
```

### Field Types

Pydantic supports all Python built-in types with automatic validation.

```python
from datetime import datetime, date
from decimal import Decimal
from typing import List

class Product(BaseModel):
    name: str
    price: float
    quantity: int
    is_available: bool
    tags: List[str]
    created_at: datetime
    expiry_date: date
    precise_price: Decimal
```

### Optional Fields

Use `Optional` or the union syntax for fields that can be `None`.

```python
from typing import Optional

class User(BaseModel):
    username: str
    email: str
    phone: Optional[str] = None  # Pydantic v1 & v2
    address: str | None = None   # Pydantic v2 (Python 3.10+)

# Both fields are optional
user1 = User(username="john", email="john@example.com")
user2 = User(username="jane", email="jane@example.com", phone="123-456-7890")
```

### Default Values

Set default values for fields that aren't required.

```python
class Settings(BaseModel):
    app_name: str = "MyApp"
    debug: bool = False
    max_connections: int = 100
    timeout: float = 30.0

# Use defaults
settings1 = Settings()
print(settings1.debug)  # False

# Override defaults
settings2 = Settings(debug=True, timeout=60.0)
```

### Field Validation

Pydantic automatically validates data types and constraints.

```python
class Person(BaseModel):
    name: str
    age: int
    email: str

# Valid
person = Person(name="John", age=30, email="john@example.com")

# Invalid - raises ValidationError
try:
    invalid_person = Person(name="John", age="thirty", email="john@example.com")
except Exception as e:
    print(e)  # age must be an integer
```

### Field Metadata with Field()

Use `Field()` to add constraints and metadata to fields.

```python
from pydantic import BaseModel, Field

class Product(BaseModel):
    name: str = Field(..., min_length=3, max_length=50)
    price: float = Field(..., gt=0, description="Price must be positive")
    quantity: int = Field(default=0, ge=0, le=10000)
    sku: str = Field(..., pattern=r'^[A-Z]{3}-\d{4}$')
    description: str = Field(default="", max_length=500)

# Valid
product = Product(
    name="Widget",
    price=29.99,
    quantity=100,
    sku="WDG-0001"
)

# Invalid - price must be positive
try:
    invalid = Product(name="Test", price=-10, sku="ABC-1234")
except Exception as e:
    print(e)
```

### Field Aliases

Use aliases to map different field names.

```python
from pydantic import Field

class User(BaseModel):
    username: str = Field(..., alias="user_name")
    email: str = Field(..., alias="email_address")

# Create using aliases
user = User(user_name="john", email_address="john@example.com")
print(user.username)  # "john"

# Serialization with aliases
print(user.model_dump(by_alias=True))
# {'user_name': 'john', 'email_address': 'john@example.com'}
```

### Computed Fields

Create fields that are computed from other fields (Pydantic v2).

```python
from pydantic import computed_field

class Rectangle(BaseModel):
    width: float
    height: float
    
    @computed_field
    @property
    def area(self) -> float:
        return self.width * self.height

rect = Rectangle(width=10, height=5)
print(rect.area)  # 50.0
print(rect.model_dump())  # {'width': 10.0, 'height': 5.0, 'area': 50.0}
```

## 5.2 Advanced Pydantic

### Custom Validators (@validator) - Pydantic v1

Use `@validator` decorator to create custom validation logic.

```python
from pydantic import BaseModel, validator

class User(BaseModel):
    username: str
    email: str
    password: str
    
    @validator('username')
    def username_alphanumeric(cls, v):
        if not v.isalnum():
            raise ValueError('Username must be alphanumeric')
        return v
    
    @validator('password')
    def password_strength(cls, v):
        if len(v) < 8:
            raise ValueError('Password must be at least 8 characters')
        if not any(c.isupper() for c in v):
            raise ValueError('Password must contain uppercase letter')
        if not any(c.isdigit() for c in v):
            raise ValueError('Password must contain digit')
        return v
```

### Field Validators - Pydantic v2

Pydantic v2 uses `@field_validator` for field-level validation.

```python
from pydantic import BaseModel, field_validator

class User(BaseModel):
    username: str
    email: str
    age: int
    
    @field_validator('username')
    @classmethod
    def username_alphanumeric(cls, v):
        if not v.isalnum():
            raise ValueError('Username must be alphanumeric')
        return v
    
    @field_validator('age')
    @classmethod
    def age_must_be_adult(cls, v):
        if v < 18:
            raise ValueError('User must be 18 or older')
        return v
```

### Root Validators - Pydantic v1

Validate entire model or multiple fields together.

```python
from pydantic import BaseModel, root_validator

class UserRegistration(BaseModel):
    password: str
    password_confirm: str
    email: str
    
    @root_validator
    def passwords_match(cls, values):
        password = values.get('password')
        password_confirm = values.get('password_confirm')
        if password != password_confirm:
            raise ValueError('Passwords do not match')
        return values
```

### Model Validators - Pydantic v2

Pydantic v2 uses `@model_validator` for model-level validation.

```python
from pydantic import BaseModel, model_validator

class UserRegistration(BaseModel):
    password: str
    password_confirm: str
    username: str
    
    @model_validator(mode='after')
    def check_passwords_match(self):
        if self.password != self.password_confirm:
            raise ValueError('Passwords do not match')
        return self

class DateRange(BaseModel):
    start_date: str
    end_date: str
    
    @model_validator(mode='after')
    def check_dates(self):
        if self.start_date > self.end_date:
            raise ValueError('start_date must be before end_date')
        return self
```

### Pre and Post Validators

Control when validation occurs (before or after standard validation).

```python
# Pydantic v1
from pydantic import BaseModel, validator

class User(BaseModel):
    email: str
    
    @validator('email', pre=True)
    def lowercase_email(cls, v):
        if isinstance(v, str):
            return v.lower()
        return v

# Pydantic v2
from pydantic import field_validator

class User(BaseModel):
    email: str
    
    @field_validator('email', mode='before')
    @classmethod
    def lowercase_email(cls, v):
        if isinstance(v, str):
            return v.lower()
        return v
```

### Validator Dependencies

Validators can reference other fields.

```python
from pydantic import BaseModel, field_validator

class User(BaseModel):
    username: str
    email: str
    confirm_email: str
    
    @field_validator('confirm_email')
    @classmethod
    def emails_match(cls, v, info):
        if 'email' in info.data and v != info.data['email']:
            raise ValueError('Emails do not match')
        return v
```

### Reusing Validators

Create reusable validators for common validation logic.

```python
from pydantic import BaseModel, field_validator

def validate_positive(v: float) -> float:
    if v <= 0:
        raise ValueError('Must be positive')
    return v

class Product(BaseModel):
    price: float
    weight: float
    
    _validate_price = field_validator('price')(lambda cls, v: validate_positive(v))
    _validate_weight = field_validator('weight')(lambda cls, v: validate_positive(v))
```

## 5.3 Complex Data Types

### Nested Models

Models can contain other models for complex structures.

```python
class Address(BaseModel):
    street: str
    city: str
    country: str
    postal_code: str

class Company(BaseModel):
    name: str
    registration_number: str

class User(BaseModel):
    username: str
    email: str
    address: Address
    company: Company | None = None

# Create nested instance
user = User(
    username="john",
    email="john@example.com",
    address={
        "street": "123 Main St",
        "city": "New York",
        "country": "USA",
        "postal_code": "10001"
    }
)
```

### List Fields with Constraints

Apply constraints to list fields and their elements.

```python
from typing import List
from pydantic import Field

class Classroom(BaseModel):
    teacher: str
    students: List[str] = Field(..., min_length=1, max_length=30)
    grades: List[int] = Field(default_factory=list)
    
    @field_validator('grades')
    @classmethod
    def validate_grades(cls, v):
        for grade in v:
            if not 0 <= grade <= 100:
                raise ValueError('Grades must be between 0 and 100')
        return v

classroom = Classroom(
    teacher="Ms. Smith",
    students=["Alice", "Bob", "Charlie"],
    grades=[85, 90, 78]
)
```

### Dict Fields

Use dictionaries for flexible key-value data.

```python
from typing import Dict

class Configuration(BaseModel):
    app_name: str
    settings: Dict[str, str]
    feature_flags: Dict[str, bool]
    limits: Dict[str, int]

config = Configuration(
    app_name="MyApp",
    settings={"theme": "dark", "language": "en"},
    feature_flags={"new_ui": True, "beta_features": False},
    limits={"max_users": 100, "max_requests": 1000}
)
```

### Set Fields

Use sets for unique collections.

```python
from typing import Set

class Article(BaseModel):
    title: str
    content: str
    tags: Set[str]
    categories: Set[str]

article = Article(
    title="Python Tips",
    content="...",
    tags={"python", "programming", "tips"},
    categories={"tutorial", "beginner"}
)
```

### Tuple Fields

Use tuples for fixed-length sequences.

```python
from typing import Tuple

class Location(BaseModel):
    name: str
    coordinates: Tuple[float, float]  # (latitude, longitude)
    rgb_color: Tuple[int, int, int]

location = Location(
    name="Central Park",
    coordinates=(40.785091, -73.968285),
    rgb_color=(34, 139, 34)
)
```

### Union Types

Allow multiple types for a field.

```python
from typing import Union

class Response(BaseModel):
    success: bool
    data: Union[str, int, dict, list]
    error: Union[str, None] = None

# Different data types
response1 = Response(success=True, data="Success message")
response2 = Response(success=True, data={"user_id": 123})
response3 = Response(success=False, data=[], error="Not found")
```

### Literal Types

Restrict values to specific literals.

```python
from typing import Literal

class User(BaseModel):
    username: str
    role: Literal["admin", "user", "guest"]
    status: Literal["active", "inactive", "suspended"]

# Valid
user = User(username="john", role="admin", status="active")

# Invalid - raises ValidationError
try:
    invalid = User(username="jane", role="superadmin", status="active")
except Exception as e:
    print(e)  # role must be one of: admin, user, guest
```

### Enum Types

Use Python enums for predefined choices.

```python
from enum import Enum

class UserRole(str, Enum):
    ADMIN = "admin"
    USER = "user"
    GUEST = "guest"

class Status(str, Enum):
    ACTIVE = "active"
    INACTIVE = "inactive"
    SUSPENDED = "suspended"

class User(BaseModel):
    username: str
    role: UserRole
    status: Status

user = User(
    username="john",
    role=UserRole.ADMIN,
    status=Status.ACTIVE
)

# Also accepts string values
user2 = User(username="jane", role="user", status="active")
```

## 5.4 Pydantic Configuration

### Config Class

Configure model behavior using the `Config` class (Pydantic v1) or `model_config` (Pydantic v2).

```python
# Pydantic v1
class User(BaseModel):
    username: str
    email: str
    
    class Config:
        str_strip_whitespace = True
        str_min_length = 1
        validate_assignment = True

# Pydantic v2
from pydantic import ConfigDict

class User(BaseModel):
    model_config = ConfigDict(
        str_strip_whitespace=True,
        validate_assignment=True
    )
    
    username: str
    email: str
```

### orm_mode / from_attributes

Allow creation from ORM objects (Pydantic v1: `orm_mode`, v2: `from_attributes`).

```python
# Pydantic v1
class UserSchema(BaseModel):
    id: int
    username: str
    email: str
    
    class Config:
        orm_mode = True

# Pydantic v2
from pydantic import ConfigDict

class UserSchema(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    
    id: int
    username: str
    email: str

# Mock ORM object
class UserORM:
    def __init__(self):
        self.id = 1
        self.username = "john"
        self.email = "john@example.com"

orm_user = UserORM()
user = UserSchema.model_validate(orm_user)
```

### Schema Customization

Customize JSON schema generation.

```python
from pydantic import Field

class Product(BaseModel):
    model_config = ConfigDict(
        json_schema_extra={
            "examples": [
                {
                    "name": "Widget",
                    "price": 29.99,
                    "sku": "WDG-001"
                }
            ]
        }
    )
    
    name: str = Field(..., title="Product Name", description="Name of the product")
    price: float = Field(..., title="Price", description="Product price in USD", gt=0)
    sku: str = Field(..., title="SKU", description="Stock Keeping Unit")
```

### JSON Encoders

Define custom JSON encoders for specific types (Pydantic v1).

```python
from datetime import datetime
from decimal import Decimal

# Pydantic v1
class Transaction(BaseModel):
    id: int
    amount: Decimal
    created_at: datetime
    
    class Config:
        json_encoders = {
            datetime: lambda v: v.isoformat(),
            Decimal: lambda v: float(v)
        }
```

### Extra Fields Handling

Control how extra fields are handled.

```python
from pydantic import ConfigDict

# Forbid extra fields
class StrictUser(BaseModel):
    model_config = ConfigDict(extra='forbid')
    
    username: str
    email: str

# Allow extra fields
class FlexibleUser(BaseModel):
    model_config = ConfigDict(extra='allow')
    
    username: str
    email: str

# Ignore extra fields (default)
class IgnoreUser(BaseModel):
    model_config = ConfigDict(extra='ignore')
    
    username: str
    email: str

# Test
try:
    StrictUser(username="john", email="john@example.com", age=30)
except Exception as e:
    print(e)  # Extra inputs are not permitted

flexible = FlexibleUser(username="john", email="john@example.com", age=30)
print(flexible.model_dump())  # Includes 'age'
```

### Validation on Assignment

Validate data when fields are modified after creation.

```python
from pydantic import ConfigDict

class User(BaseModel):
    model_config = ConfigDict(validate_assignment=True)
    
    username: str = Field(..., min_length=3)
    email: str

user = User(username="john", email="john@example.com")

# This will raise ValidationError
try:
    user.username = "jo"  # Too short
except Exception as e:
    print(e)
```

### Model Serialization

Convert models to dictionaries and JSON with various options.

```python
class User(BaseModel):
    username: str
    email: str
    password: str
    age: int | None = None

user = User(username="john", email="john@example.com", password="secret123")

# To dict
print(user.model_dump())
# {'username': 'john', 'email': 'john@example.com', 'password': 'secret123', 'age': None}

# Exclude fields
print(user.model_dump(exclude={'password'}))
# {'username': 'john', 'email': 'john@example.com', 'age': None}

# Exclude unset
print(user.model_dump(exclude_unset=True))
# {'username': 'john', 'email': 'john@example.com', 'password': 'secret123'}

# Exclude None
print(user.model_dump(exclude_none=True))
# {'username': 'john', 'email': 'john@example.com', 'password': 'secret123'}

# To JSON
print(user.model_dump_json(exclude={'password'}))
```

### Model Parsing

Parse data from various formats.

```python
# From dict
user_data = {"username": "john", "email": "john@example.com", "password": "secret"}
user = User(**user_data)

# From JSON string
json_str = '{"username": "john", "email": "john@example.com", "password": "secret"}'
user = User.model_validate_json(json_str)

# From ORM object (with from_attributes=True)
class UserSchema(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    username: str
    email: str

class UserORM:
    username = "john"
    email = "john@example.com"

user = UserSchema.model_validate(UserORM())
```

## 5.5 Pydantic v2 Features

### Migration from v1 to v2

Key changes when migrating from Pydantic v1 to v2:

```python
# Pydantic v1
class UserV1(BaseModel):
    username: str
    
    class Config:
        orm_mode = True
        
    @validator('username')
    def validate_username(cls, v):
        return v.lower()
    
    @root_validator
    def validate_model(cls, values):
        return values

# Pydantic v2
from pydantic import ConfigDict, field_validator, model_validator

class UserV2(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    
    username: str
    
    @field_validator('username')
    @classmethod
    def validate_username(cls, v):
        return v.lower()
    
    @model_validator(mode='after')
    def validate_model(self):
        return self
```

### New Validation Decorators

Pydantic v2 introduces new decorator patterns.

```python
from pydantic import (
    BaseModel,
    field_validator,
    model_validator,
    ValidationInfo
)

class User(BaseModel):
    username: str
    email: str
    age: int
    
    # Field validator with mode
    @field_validator('username', mode='before')
    @classmethod
    def lowercase_username(cls, v):
        if isinstance(v, str):
            return v.lower()
        return v
    
    # Field validator with ValidationInfo
    @field_validator('email')
    @classmethod
    def validate_email(cls, v, info: ValidationInfo):
        if '@' not in v:
            raise ValueError('Invalid email')
        return v
    
    # Model validator (after all fields validated)
    @model_validator(mode='after')
    def validate_user(self):
        if self.age < 18 and 'admin' in self.username:
            raise ValueError('Admins must be 18+')
        return self
```

### Performance Improvements

Pydantic v2 is significantly faster than v1 due to Rust core.

```python
from pydantic import BaseModel
import time

class User(BaseModel):
    username: str
    email: str
    age: int

# Benchmark
start = time.time()
for i in range(100000):
    user = User(username=f"user{i}", email=f"user{i}@example.com", age=25)
end = time.time()

print(f"Created 100,000 users in {end - start:.2f} seconds")
# Pydantic v2 is 5-50x faster than v1
```

### Computed Fields

Define computed properties that are included in serialization.

```python
from pydantic import BaseModel, computed_field

class User(BaseModel):
    first_name: str
    last_name: str
    birth_year: int
    
    @computed_field
    @property
    def full_name(self) -> str:
        return f"{self.first_name} {self.last_name}"
    
    @computed_field
    @property
    def age(self) -> int:
        from datetime import datetime
        current_year = datetime.now().year
        return current_year - self.birth_year

user = User(first_name="John", last_name="Doe", birth_year=1990)
print(user.full_name)  # "John Doe"
print(user.age)  # 34 (in 2024)
print(user.model_dump())  
# {'first_name': 'John', 'last_name': 'Doe', 'birth_year': 1990, 'full_name': 'John Doe', 'age': 34}
```

### Serialization Improvements

Enhanced control over serialization in Pydantic v2.

```python
from pydantic import BaseModel, field_serializer
from datetime import datetime

class Event(BaseModel):
    name: str
    created_at: datetime
    
    @field_serializer('created_at')
    def serialize_datetime(self, value: datetime, _info):
        return value.isoformat()

event = Event(name="Conference", created_at=datetime.now())
print(event.model_dump())
# created_at is serialized as ISO format string
```

### TypeAdapter

Validate data without creating a model class.

```python
from pydantic import TypeAdapter
from typing import List

# Validate a list of integers
int_list_adapter = TypeAdapter(List[int])
validated = int_list_adapter.validate_python([1, 2, 3, "4"])
print(validated)  # [1, 2, 3, 4] - "4" coerced to int

# Validate complex types
from typing import Dict

dict_adapter = TypeAdapter(Dict[str, int])
validated_dict = dict_adapter.validate_python({"a": 1, "b": "2"})
print(validated_dict)  # {'a': 1, 'b': 2}

# Validate JSON
json_data = '[1, 2, 3, 4]'
validated_json = int_list_adapter.validate_json(json_data)
print(validated_json)  # [1, 2, 3, 4]
```

---

## Practice Exercise

Create a comprehensive user management system using advanced Pydantic features:

1. Create a `User` model with nested `Address` and `Profile` models
2. Add custom validators for email, phone, and password
3. Use computed fields for full name and age
4. Implement proper v2 validation decorators
5. Configure the model to work with ORM objects
6. Add serialization customization

### Solution

```python
from pydantic import (
    BaseModel,
    ConfigDict,
    Field,
    field_validator,
    model_validator,
    computed_field
)
from datetime import datetime
from typing import Literal
from enum import Enum
import re

# Enums
class UserRole(str, Enum):
    ADMIN = "admin"
    USER = "user"
    MODERATOR = "moderator"

# Nested Models
class Address(BaseModel):
    street: str = Field(..., min_length=5)
    city: str = Field(..., min_length=2)
    state: str = Field(..., min_length=2, max_length=2)
    zip_code: str = Field(..., pattern=r'^\d{5}$')
    country: str = "USA"

class Profile(BaseModel):
    bio: str | None = Field(None, max_length=500)
    website: str | None = None
    avatar_url: str | None = None
    
    @field_validator('website')
    @classmethod
    def validate_website(cls, v):
        if v and not v.startswith(('http://', 'https://')):
            raise ValueError('Website must start with http:// or https://')
        return v

# Main User Model
class User(BaseModel):
    model_config = ConfigDict(
        from_attributes=True,
        validate_assignment=True,
        str_strip_whitespace=True,
        json_schema_extra={
            "examples": [{
                "first_name": "John",
                "last_name": "Doe",
                "email": "john@example.com",
                "phone": "555-123-4567",
                "birth_year": 1990,
                "role": "user",
                "address": {
                    "street": "123 Main St",
                    "city": "New York",
                    "state": "NY",
                    "zip_code": "10001"
                }
            }]
        }
    )
    
    # Basic fields
    first_name: str = Field(..., min_length=2, max_length=50)
    last_name: str = Field(..., min_length=2, max_length=50)
    email: str
    phone: str | None = None
    password: str = Field(..., min_length=8)
    birth_year: int = Field(..., ge=1900, le=2024)
    role: UserRole = UserRole.USER
    is_active: bool = True
    
    # Nested models
    address: Address
    profile: Profile | None = None
    
    # Field validators
    @field_validator('email')
    @classmethod
    def validate_email(cls, v):
        email_regex = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        if not re.match(email_regex, v):
            raise ValueError('Invalid email format')
        return v.lower()
    
    @field_validator('phone')
    @classmethod
    def validate_phone(cls, v):
        if v:
            phone_regex = r'^\d{3}-\d{3}-\d{4}$'
            if not re.match(phone_regex, v):
                raise ValueError('Phone must be in format XXX-XXX-XXXX')
        return v
    
    @field_validator('password')
    @classmethod
    def validate_password(cls, v):
        if len(v) < 8:
            raise ValueError('Password must be at least 8 characters')
        if not any(c.isupper() for c in v):
            raise ValueError('Password must contain uppercase letter')
        if not any(c.islower() for c in v):
            raise ValueError('Password must contain lowercase letter')
        if not any(c.isdigit() for c in v):
            raise ValueError('Password must contain digit')
        if not any(c in '!@#$%^&*()_+-=[]{}|;:,.<>?' for c in v):
            raise ValueError('Password must contain special character')
        return v
    
    # Computed fields
    @computed_field
    @property
    def full_name(self) -> str:
        return f"{self.first_name} {self.last_name}"
    
    @computed_field
    @property
    def age(self) -> int:
        current_year = datetime.now().year
        return current_year - self.birth_year
    
    # Model validator
    @model_validator(mode='after')
    def validate_admin_requirements(self):
        if self.role == UserRole.ADMIN and self.age < 21:
            raise ValueError('Admin users must be at least 21 years old')
        return self
    
    def to_public(self):
        """Return public-safe version without sensitive data"""
        return self.model_dump(exclude={'password'})

# Usage Example
if __name__ == "__main__":
    # Create user
    user = User(
        first_name="John",
        last_name="Doe",
        email="JOHN@EXAMPLE.COM",  # Will be lowercased
        phone="555-123-4567",
        password="SecureP@ss123",
        birth_year=1990,
        role=UserRole.USER,
        address={
            "street": "123 Main Street",
            "city": "New York",
            "state": "NY",
            "zip_code": "10001"
        },
        profile={
            "bio": "Software developer",
            "website": "https://johndoe.com"
        }
    )
    
    print(f"Full Name: {user.full_name}")
    print(f"Age: {user.age}")
    print(f"Email: {user.email}")
    
    # Serialize
    print("\nFull data:")
    print(user.model_dump())
    
    print("\nPublic data (no password):")
    print(user.to_public())
    
    print("\nJSON output:")
    print(user.model_dump_json(exclude={'password'}, indent=2))
    
    # Test validation
    try:
        invalid_user = User(
            first_name="J",  # Too short
            last_name="Doe",
            email="invalid-email",
            password="weak",
            birth_year=1990,
            address={
                "street": "123 Main St",
                "city": "NYC",
                "state": "NY",
                "zip_code": "10001"
            }
        )
    except Exception as e:
        print(f"\nValidation errors:\n{e}")
```

---

**Key Takeaways:**
- Pydantic provides powerful data validation with minimal code
- Use `Field()` for constraints and metadata
- Validators allow custom validation logic
- Nested models enable complex data structures
- Pydantic v2 offers significant performance improvements
- Computed fields provide dynamic properties
- Configuration options control model behavior

**Additional Resources:**
- [Pydantic Documentation](https://docs.pydantic.dev/)
- [Pydantic v2 Migration Guide](https://docs.pydantic.dev/latest/migration/)
- [FastAPI with Pydantic](https://fastapi.tiangolo.com/tutorial/body/)