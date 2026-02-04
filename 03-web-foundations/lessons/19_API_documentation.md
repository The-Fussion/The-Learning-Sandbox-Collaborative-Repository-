# API Documentation in Python

## Table of Contents
1. [Introduction](#introduction)
2. [OpenAPI Specification](#openapi-specification)
3. [FastAPI - Auto-Generated Documentation](#fastapi)
4. [Flask with Flask-RESTX](#flask-with-flask-restx)
5. [Django REST Framework](#django-rest-framework)
6. [Advanced Documentation Techniques](#advanced-documentation-techniques)
7. [Best Practices](#best-practices)
8. [Testing Documentation](#testing-documentation)

---

## Introduction

API documentation is crucial for developers who consume your API. Good documentation should be:
- **Accurate**: Reflects the actual API behavior
- **Complete**: Covers all endpoints, parameters, and responses
- **Interactive**: Allows testing directly from the docs
- **Up-to-date**: Stays synchronized with code changes
- **Clear**: Easy to understand with examples

### Why OpenAPI/Swagger?

**OpenAPI Specification (OAS)** is an industry-standard for describing REST APIs. Benefits include:
- Machine-readable format (JSON/YAML)
- Auto-generated interactive documentation
- Client SDK generation
- API testing tools
- Contract validation

### Popular Tools

- **Swagger UI**: Interactive API documentation
- **ReDoc**: Alternative documentation UI
- **FastAPI**: Python framework with built-in OpenAPI support
- **Flask-RESTX**: Flask extension for API documentation
- **drf-spectacular**: Django REST Framework OpenAPI generator

---

## OpenAPI Specification

### Basic OpenAPI Structure

```yaml
openapi: 3.0.0
info:
  title: My API
  description: API for managing resources
  version: 1.0.0
  contact:
    name: API Support
    email: support@example.com
  license:
    name: MIT

servers:
  - url: https://api.example.com/v1
    description: Production server
  - url: https://staging-api.example.com/v1
    description: Staging server

paths:
  /users:
    get:
      summary: List all users
      description: Returns a list of all users in the system
      tags:
        - Users
      parameters:
        - name: limit
          in: query
          description: Maximum number of users to return
          required: false
          schema:
            type: integer
            default: 10
            minimum: 1
            maximum: 100
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/User'

components:
  schemas:
    User:
      type: object
      required:
        - id
        - email
      properties:
        id:
          type: integer
          example: 1
        email:
          type: string
          format: email
          example: user@example.com
        name:
          type: string
          example: John Doe
  
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

security:
  - BearerAuth: []
```

---

## FastAPI - Auto-Generated Documentation

FastAPI is a modern Python web framework that automatically generates OpenAPI documentation from your code.

### Installation

```bash
pip install fastapi uvicorn[standard]
pip install python-multipart  # For form data
pip install email-validator   # For email validation
```

### Basic FastAPI Application

```python
# main.py
from fastapi import FastAPI, HTTPException, Query, Path, Body, Header
from pydantic import BaseModel, Field, EmailStr
from typing import Optional, List
from datetime import datetime
from enum import Enum

# Create FastAPI app with metadata
app = FastAPI(
    title="User Management API",
    description="""
    ## User Management System
    
    This API allows you to manage users in the system.
    
    ### Features:
    * **Create users** - Register new users
    * **Read users** - Retrieve user information
    * **Update users** - Modify user details
    * **Delete users** - Remove users from system
    
    ### Authentication
    Most endpoints require Bearer token authentication.
    """,
    version="1.0.0",
    terms_of_service="http://example.com/terms/",
    contact={
        "name": "API Support",
        "url": "http://example.com/support",
        "email": "support@example.com",
    },
    license_info={
        "name": "MIT",
        "url": "https://opensource.org/licenses/MIT",
    },
    # Customize documentation URLs
    docs_url="/docs",          # Swagger UI
    redoc_url="/redoc",        # ReDoc
    openapi_url="/openapi.json"  # OpenAPI schema
)

# Enums for documentation
class UserRole(str, Enum):
    """User role in the system"""
    admin = "admin"
    user = "user"
    guest = "guest"

class SortOrder(str, Enum):
    """Sort order for listings"""
    asc = "asc"
    desc = "desc"

# Pydantic models for request/response validation and documentation
class UserBase(BaseModel):
    """Base user model with common fields"""
    email: EmailStr = Field(
        ...,
        description="User's email address",
        example="john.doe@example.com"
    )
    username: str = Field(
        ...,
        min_length=3,
        max_length=50,
        description="Unique username",
        example="johndoe"
    )
    full_name: Optional[str] = Field(
        None,
        max_length=100,
        description="User's full name",
        example="John Doe"
    )
    role: UserRole = Field(
        default=UserRole.user,
        description="User's role in the system"
    )

class UserCreate(UserBase):
    """Model for creating a new user"""
    password: str = Field(
        ...,
        min_length=8,
        description="User password (minimum 8 characters)",
        example="SecurePass123!"
    )

class UserUpdate(BaseModel):
    """Model for updating user information"""
    email: Optional[EmailStr] = Field(None, description="New email address")
    username: Optional[str] = Field(None, min_length=3, max_length=50)
    full_name: Optional[str] = Field(None, max_length=100)
    role: Optional[UserRole] = None
    is_active: Optional[bool] = Field(None, description="User account status")

class User(UserBase):
    """Complete user model returned in responses"""
    id: int = Field(..., description="Unique user identifier", example=1)
    is_active: bool = Field(default=True, description="Whether user account is active")
    created_at: datetime = Field(..., description="Account creation timestamp")
    updated_at: Optional[datetime] = Field(None, description="Last update timestamp")
    
    class Config:
        json_schema_extra = {
            "example": {
                "id": 1,
                "email": "john.doe@example.com",
                "username": "johndoe",
                "full_name": "John Doe",
                "role": "user",
                "is_active": True,
                "created_at": "2024-01-15T10:30:00Z",
                "updated_at": "2024-01-20T14:45:00Z"
            }
        }

class UserList(BaseModel):
    """Paginated list of users"""
    total: int = Field(..., description="Total number of users")
    page: int = Field(..., description="Current page number")
    page_size: int = Field(..., description="Number of items per page")
    users: List[User] = Field(..., description="List of users")

class ErrorResponse(BaseModel):
    """Standard error response"""
    detail: str = Field(..., description="Error message")
    error_code: Optional[str] = Field(None, description="Application-specific error code")

# Tags for grouping endpoints
tags_metadata = [
    {
        "name": "Users",
        "description": "Operations with users. Login, registration, and user management.",
    },
    {
        "name": "Authentication",
        "description": "Authentication and authorization endpoints.",
    },
    {
        "name": "Health",
        "description": "Health check and status endpoints.",
    },
]

app = FastAPI(
    title="User Management API",
    description="API for managing users",
    version="1.0.0",
    openapi_tags=tags_metadata
)

# Simulated database
fake_users_db = {}
user_id_counter = 1

# Endpoints with detailed documentation

@app.get(
    "/",
    tags=["Health"],
    summary="Root endpoint",
    response_description="Welcome message"
)
async def root():
    """
    # Welcome Endpoint
    
    Returns a welcome message for the API.
    
    This is the entry point to verify the API is running.
    """
    return {"message": "Welcome to User Management API"}

@app.get(
    "/health",
    tags=["Health"],
    summary="Health check",
    response_description="Service health status"
)
async def health_check():
    """
    ## Health Check
    
    Returns the current health status of the service.
    
    ### Response
    - **status**: Service status (ok, degraded, down)
    - **timestamp**: Current server time
    """
    return {
        "status": "ok",
        "timestamp": datetime.utcnow().isoformat()
    }

@app.post(
    "/users/",
    response_model=User,
    status_code=201,
    tags=["Users"],
    summary="Create a new user",
    response_description="The created user",
    responses={
        201: {
            "description": "User created successfully",
            "content": {
                "application/json": {
                    "example": {
                        "id": 1,
                        "email": "john@example.com",
                        "username": "johndoe",
                        "full_name": "John Doe",
                        "role": "user",
                        "is_active": True,
                        "created_at": "2024-01-15T10:30:00Z",
                        "updated_at": None
                    }
                }
            }
        },
        400: {
            "description": "Invalid input data",
            "model": ErrorResponse
        },
        409: {
            "description": "User already exists",
            "model": ErrorResponse
        }
    }
)
async def create_user(
    user: UserCreate = Body(
        ...,
        description="User data for registration",
        example={
            "email": "john@example.com",
            "username": "johndoe",
            "full_name": "John Doe",
            "role": "user",
            "password": "SecurePass123!"
        }
    )
):
    """
    ## Create New User
    
    Register a new user in the system.
    
    ### Parameters
    - **email**: Valid email address (unique)
    - **username**: Unique username (3-50 characters)
    - **full_name**: User's full name (optional)
    - **role**: User role (admin, user, guest)
    - **password**: Strong password (minimum 8 characters)
    
    ### Returns
    - **201**: User created successfully
    - **400**: Invalid input data
    - **409**: User with email or username already exists
    
    ### Example Request
    ```json
    {
        "email": "john@example.com",
        "username": "johndoe",
        "full_name": "John Doe",
        "role": "user",
        "password": "SecurePass123!"
    }
    ```
    """
    global user_id_counter
    
    # Check if user exists
    for existing_user in fake_users_db.values():
        if existing_user["email"] == user.email:
            raise HTTPException(
                status_code=409,
                detail=f"User with email {user.email} already exists"
            )
        if existing_user["username"] == user.username:
            raise HTTPException(
                status_code=409,
                detail=f"Username {user.username} is already taken"
            )
    
    # Create user
    user_dict = user.dict()
    user_dict.pop("password")  # Don't store plain password
    user_dict.update({
        "id": user_id_counter,
        "is_active": True,
        "created_at": datetime.utcnow(),
        "updated_at": None
    })
    
    fake_users_db[user_id_counter] = user_dict
    user_id_counter += 1
    
    return user_dict

@app.get(
    "/users/",
    response_model=UserList,
    tags=["Users"],
    summary="List all users",
    response_description="Paginated list of users"
)
async def list_users(
    page: int = Query(
        1,
        ge=1,
        description="Page number",
        example=1
    ),
    page_size: int = Query(
        10,
        ge=1,
        le=100,
        description="Number of items per page",
        example=10
    ),
    role: Optional[UserRole] = Query(
        None,
        description="Filter by user role"
    ),
    is_active: Optional[bool] = Query(
        None,
        description="Filter by active status"
    ),
    sort_by: str = Query(
        "created_at",
        description="Field to sort by",
        enum=["created_at", "email", "username"]
    ),
    sort_order: SortOrder = Query(
        SortOrder.desc,
        description="Sort order"
    )
):
    """
    ## List Users
    
    Retrieve a paginated list of users with optional filtering and sorting.
    
    ### Query Parameters
    - **page**: Page number (default: 1)
    - **page_size**: Items per page (1-100, default: 10)
    - **role**: Filter by role (admin, user, guest)
    - **is_active**: Filter by active status
    - **sort_by**: Sort field (created_at, email, username)
    - **sort_order**: Sort order (asc, desc)
    
    ### Returns
    Paginated list of users with metadata:
    - **total**: Total number of users matching filters
    - **page**: Current page number
    - **page_size**: Items per page
    - **users**: Array of user objects
    """
    # Filter users
    filtered_users = list(fake_users_db.values())
    
    if role:
        filtered_users = [u for u in filtered_users if u["role"] == role]
    
    if is_active is not None:
        filtered_users = [u for u in filtered_users if u["is_active"] == is_active]
    
    # Sort users
    reverse = sort_order == SortOrder.desc
    filtered_users.sort(key=lambda x: x.get(sort_by, ""), reverse=reverse)
    
    # Paginate
    total = len(filtered_users)
    start = (page - 1) * page_size
    end = start + page_size
    paginated_users = filtered_users[start:end]
    
    return {
        "total": total,
        "page": page,
        "page_size": page_size,
        "users": paginated_users
    }

@app.get(
    "/users/{user_id}",
    response_model=User,
    tags=["Users"],
    summary="Get user by ID",
    response_description="User details",
    responses={
        200: {"description": "User found"},
        404: {
            "description": "User not found",
            "model": ErrorResponse
        }
    }
)
async def get_user(
    user_id: int = Path(
        ...,
        ge=1,
        description="The ID of the user to retrieve",
        example=1
    )
):
    """
    ## Get User by ID
    
    Retrieve detailed information about a specific user.
    
    ### Path Parameters
    - **user_id**: Unique user identifier (must be positive integer)
    
    ### Returns
    - **200**: User details
    - **404**: User not found
    """
    if user_id not in fake_users_db:
        raise HTTPException(
            status_code=404,
            detail=f"User with ID {user_id} not found"
        )
    
    return fake_users_db[user_id]

@app.put(
    "/users/{user_id}",
    response_model=User,
    tags=["Users"],
    summary="Update user",
    response_description="Updated user details"
)
async def update_user(
    user_id: int = Path(..., ge=1, description="User ID to update"),
    user_update: UserUpdate = Body(..., description="Fields to update")
):
    """
    ## Update User
    
    Update user information. Only provided fields will be updated.
    
    ### Parameters
    - **user_id**: ID of user to update
    - **user_update**: Object containing fields to update
    
    ### Updatable Fields
    - email
    - username
    - full_name
    - role
    - is_active
    
    ### Returns
    Updated user object
    """
    if user_id not in fake_users_db:
        raise HTTPException(status_code=404, detail="User not found")
    
    stored_user = fake_users_db[user_id]
    update_data = user_update.dict(exclude_unset=True)
    
    # Update fields
    for field, value in update_data.items():
        stored_user[field] = value
    
    stored_user["updated_at"] = datetime.utcnow()
    
    return stored_user

@app.delete(
    "/users/{user_id}",
    status_code=204,
    tags=["Users"],
    summary="Delete user",
    response_description="User deleted successfully",
    responses={
        204: {"description": "User deleted"},
        404: {"description": "User not found"}
    }
)
async def delete_user(
    user_id: int = Path(..., ge=1, description="User ID to delete")
):
    """
    ## Delete User
    
    Permanently delete a user from the system.
    
    ### Parameters
    - **user_id**: ID of user to delete
    
    ### Returns
    - **204**: User deleted successfully
    - **404**: User not found
    
    ### Warning
    This action cannot be undone!
    """
    if user_id not in fake_users_db:
        raise HTTPException(status_code=404, detail="User not found")
    
    del fake_users_db[user_id]
    return None

# Custom header example
@app.get(
    "/users/{user_id}/profile",
    tags=["Users"],
    summary="Get user profile with custom headers"
)
async def get_user_profile(
    user_id: int = Path(..., ge=1),
    x_api_key: Optional[str] = Header(
        None,
        description="API key for authentication",
        example="your-api-key-here"
    ),
    user_agent: Optional[str] = Header(
        None,
        alias="User-Agent",
        description="Client user agent"
    )
):
    """
    ## Get User Profile
    
    Get user profile with custom header authentication.
    
    ### Headers
    - **X-API-Key**: API key for authentication (optional)
    - **User-Agent**: Client user agent (auto-included)
    """
    if user_id not in fake_users_db:
        raise HTTPException(status_code=404, detail="User not found")
    
    return {
        "user": fake_users_db[user_id],
        "requested_by": user_agent
    }

# File upload example
from fastapi import File, UploadFile

@app.post(
    "/users/{user_id}/avatar",
    tags=["Users"],
    summary="Upload user avatar",
    response_description="Avatar uploaded successfully"
)
async def upload_avatar(
    user_id: int = Path(..., ge=1),
    file: UploadFile = File(
        ...,
        description="Avatar image file (JPEG, PNG)",
        example="avatar.jpg"
    )
):
    """
    ## Upload User Avatar
    
    Upload a profile picture for a user.
    
    ### Parameters
    - **user_id**: User ID
    - **file**: Image file (max 5MB, JPEG/PNG only)
    
    ### Supported Formats
    - JPEG (.jpg, .jpeg)
    - PNG (.png)
    
    ### Returns
    Upload confirmation with file details
    """
    if user_id not in fake_users_db:
        raise HTTPException(status_code=404, detail="User not found")
    
    # Validate file type
    if file.content_type not in ["image/jpeg", "image/png"]:
        raise HTTPException(
            status_code=400,
            detail="Only JPEG and PNG images are supported"
        )
    
    # In production, save file to storage
    contents = await file.read()
    
    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": len(contents),
        "user_id": user_id
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Running FastAPI

```bash
# Run with uvicorn
uvicorn main:app --reload

# Access documentation
# Swagger UI: http://localhost:8000/docs
# ReDoc: http://localhost:8000/redoc
# OpenAPI JSON: http://localhost:8000/openapi.json
```

### Advanced FastAPI Documentation Features

#### 1. Dependencies Documentation

```python
from fastapi import Depends, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Security(security)
) -> dict:
    """
    Validate JWT token and return current user.
    
    ### Authentication
    Requires Bearer token in Authorization header.
    """
    # Validate token (simplified)
    token = credentials.credentials
    # In production: decode and validate JWT
    return {"user_id": 1, "username": "current_user"}

@app.get(
    "/users/me",
    tags=["Users"],
    summary="Get current user",
    dependencies=[Depends(get_current_user)]
)
async def get_me(current_user: dict = Depends(get_current_user)):
    """
    Get currently authenticated user information.
    
    ### Authentication Required
    This endpoint requires a valid Bearer token.
    """
    return current_user
```

#### 2. Response Models with Multiple Status Codes

```python
from typing import Union

class SuccessResponse(BaseModel):
    success: bool = True
    data: dict

class ErrorResponse(BaseModel):
    success: bool = False
    error: str

@app.post(
    "/users/batch",
    tags=["Users"],
    responses={
        200: {
            "model": SuccessResponse,
            "description": "All users created"
        },
        207: {
            "model": dict,
            "description": "Partial success"
        },
        400: {
            "model": ErrorResponse,
            "description": "Invalid request"
        }
    }
)
async def create_batch_users(users: List[UserCreate]):
    """Create multiple users in a single request."""
    # Implementation
    pass
```

#### 3. Custom OpenAPI Schema

```python
from fastapi.openapi.utils import get_openapi

def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema
    
    openapi_schema = get_openapi(
        title="Custom User API",
        version="2.0.0",
        description="This is a custom OpenAPI schema",
        routes=app.routes,
    )
    
    # Add custom fields
    openapi_schema["info"]["x-logo"] = {
        "url": "https://example.com/logo.png"
    }
    
    # Add security schemes
    openapi_schema["components"]["securitySchemes"] = {
        "Bearer": {
            "type": "http",
            "scheme": "bearer",
            "bearerFormat": "JWT",
        }
    }
    
    app.openapi_schema = openapi_schema
    return app.openapi_schema

app.openapi = custom_openapi
```

---

## Flask with Flask-RESTX

Flask-RESTX is a Flask extension that adds support for building REST APIs with Swagger documentation.

### Installation

```bash
pip install flask flask-restx
```

### Basic Flask-RESTX Application

```python
# app.py
from flask import Flask, request
from flask_restx import Api, Resource, fields, Namespace
from werkzeug.exceptions import NotFound, BadRequest
from datetime import datetime

# Initialize Flask app
app = Flask(__name__)

# Initialize API with documentation
api = Api(
    app,
    version='1.0',
    title='User Management API',
    description='''
    A complete API for managing users
    
    ## Features
    * User CRUD operations
    * Role-based access
    * Input validation
    ''',
    doc='/docs',
    prefix='/api/v1'
)

# Create namespace
users_ns = Namespace('users', description='User operations')
api.add_namespace(users_ns)

# Define models for documentation
user_model = api.model('User', {
    'id': fields.Integer(
        required=True,
        description='User identifier',
        example=1
    ),
    'email': fields.String(
        required=True,
        description='User email address',
        example='john@example.com'
    ),
    'username': fields.String(
        required=True,
        description='Unique username',
        example='johndoe'
    ),
    'full_name': fields.String(
        description='Full name',
        example='John Doe'
    ),
    'role': fields.String(
        required=True,
        description='User role',
        enum=['admin', 'user', 'guest'],
        example='user'
    ),
    'is_active': fields.Boolean(
        description='Account status',
        default=True
    ),
    'created_at': fields.DateTime(
        description='Creation timestamp'
    )
})

user_input_model = api.model('UserInput', {
    'email': fields.String(
        required=True,
        description='User email',
        example='john@example.com'
    ),
    'username': fields.String(
        required=True,
        description='Username',
        min_length=3,
        max_length=50,
        example='johndoe'
    ),
    'full_name': fields.String(
        description='Full name',
        example='John Doe'
    ),
    'password': fields.String(
        required=True,
        description='User password',
        min_length=8,
        example='SecurePass123!'
    ),
    'role': fields.String(
        description='User role',
        enum=['admin', 'user', 'guest'],
        default='user'
    )
})

user_update_model = api.model('UserUpdate', {
    'email': fields.String(description='New email'),
    'full_name': fields.String(description='New full name'),
    'role': fields.String(
        description='New role',
        enum=['admin', 'user', 'guest']
    ),
    'is_active': fields.Boolean(description='Account status')
})

pagination_model = api.model('Pagination', {
    'page': fields.Integer(description='Current page', example=1),
    'per_page': fields.Integer(description='Items per page', example=10),
    'total': fields.Integer(description='Total items', example=100),
    'pages': fields.Integer(description='Total pages', example=10)
})

user_list_model = api.model('UserList', {
    'users': fields.List(fields.Nested(user_model)),
    'pagination': fields.Nested(pagination_model)
})

error_model = api.model('Error', {
    'message': fields.String(
        required=True,
        description='Error message',
        example='User not found'
    ),
    'error_code': fields.String(
        description='Error code',
        example='USER_NOT_FOUND'
    )
})

# Mock database
fake_db = {}
user_id_counter = 1

# Query parameter parser
user_query_parser = api.parser()
user_query_parser.add_argument(
    'page',
    type=int,
    default=1,
    help='Page number',
    location='args'
)
user_query_parser.add_argument(
    'per_page',
    type=int,
    default=10,
    help='Items per page',
    location='args'
)
user_query_parser.add_argument(
    'role',
    type=str,
    help='Filter by role',
    choices=['admin', 'user', 'guest'],
    location='args'
)
user_query_parser.add_argument(
    'is_active',
    type=bool,
    help='Filter by active status',
    location='args'
)

# Resources
@users_ns.route('/')
class UserList(Resource):
    """User collection endpoint"""
    
    @users_ns.doc('list_users')
    @users_ns.expect(user_query_parser)
    @users_ns.marshal_with(user_list_model)
    @users_ns.response(200, 'Success')
    def get(self):
        """
        List all users with pagination and filtering
        
        Returns a paginated list of users. Use query parameters to filter results.
        """
        args = user_query_parser.parse_args()
        page = args['page']
        per_page = args['per_page']
        role_filter = args.get('role')
        active_filter = args.get('is_active')
        
        # Filter users
        users = list(fake_db.values())
        
        if role_filter:
            users = [u for u in users if u['role'] == role_filter]
        
        if active_filter is not None:
            users = [u for u in users if u['is_active'] == active_filter]
        
        # Paginate
        total = len(users)
        start = (page - 1) * per_page
        end = start + per_page
        paginated = users[start:end]
        
        return {
            'users': paginated,
            'pagination': {
                'page': page,
                'per_page': per_page,
                'total': total,
                'pages': (total + per_page - 1) // per_page
            }
        }
    
    @users_ns.doc('create_user')
    @users_ns.expect(user_input_model, validate=True)
    @users_ns.marshal_with(user_model, code=201)
    @users_ns.response(201, 'User created')
    @users_ns.response(400, 'Validation error', error_model)
    @users_ns.response(409, 'User already exists', error_model)
    def post(self):
        """
        Create a new user
        
        Register a new user in the system with the provided information.
        Email and username must be unique.
        """
        global user_id_counter
        
        data = request.json
        
        # Check duplicates
        for user in fake_db.values():
            if user['email'] == data['email']:
                api.abort(409, 'User with this email already exists')
            if user['username'] == data['username']:
                api.abort(409, 'Username already taken')
        
        # Create user
        user = {
            'id': user_id_counter,
            'email': data['email'],
            'username': data['username'],
            'full_name': data.get('full_name'),
            'role': data.get('role', 'user'),
            'is_active': True,
            'created_at': datetime.utcnow()
        }
        
        fake_db[user_id_counter] = user
        user_id_counter += 1
        
        return user, 201

@users_ns.route('/<int:user_id>')
@users_ns.param('user_id', 'The user identifier')
class UserResource(Resource):
    """Single user endpoint"""
    
    @users_ns.doc('get_user')
    @users_ns.marshal_with(user_model)
    @users_ns.response(200, 'Success')
    @users_ns.response(404, 'User not found', error_model)
    def get(self, user_id):
        """
        Get user by ID
        
        Retrieve detailed information about a specific user.
        """
        if user_id not in fake_db:
            api.abort(404, f'User {user_id} not found')
        
        return fake_db[user_id]
    
    @users_ns.doc('update_user')
    @users_ns.expect(user_update_model, validate=True)
    @users_ns.marshal_with(user_model)
    @users_ns.response(200, 'User updated')
    @users_ns.response(404, 'User not found', error_model)
    def put(self, user_id):
        """
        Update user
        
        Update user information. Only provided fields will be updated.
        """
        if user_id not in fake_db:
            api.abort(404, f'User {user_id} not found')
        
        user = fake_db[user_id]
        data = request.json
        
        # Update fields
        for key in ['email', 'full_name', 'role', 'is_active']:
            if key in data:
                user[key] = data[key]
        
        return user
    
    @users_ns.doc('delete_user')
    @users_ns.response(204, 'User deleted')
    @users_ns.response(404, 'User not found', error_model)
    def delete(self, user_id):
        """
        Delete user
        
        Permanently remove a user from the system.
        This action cannot be undone!
        """
        if user_id not in fake_db:
            api.abort(404, f'User {user_id} not found')
        
        del fake_db[user_id]
        return '', 204

# Authorization example
authorizations = {
    'Bearer': {
        'type': 'apiKey',
        'in': 'header',
        'name': 'Authorization',
        'description': 'JWT token. Format: Bearer <token>'
    }
}

api_with_auth = Api(
    app,
    authorizations=authorizations,
    security='Bearer'
)

if __name__ == '__main__':
    app.run(debug=True)
```

---

## Django REST Framework

Django REST Framework (DRF) with drf-spectacular for OpenAPI documentation.

### Installation

```bash
pip install djangorestframework drf-spectacular
```

### Django Settings

```python
# settings.py

INSTALLED_APPS = [
    # ... other apps
    'rest_framework',
    'drf_spectacular',
]

REST_FRAMEWORK = {
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10,
}

SPECTACULAR_SETTINGS = {
    'TITLE': 'User Management API',
    'DESCRIPTION': 'API for managing users in the system',
    'VERSION': '1.0.0',
    'SERVE_INCLUDE_SCHEMA': False,
    'COMPONENT_SPLIT_REQUEST': True,
    'SCHEMA_PATH_PREFIX': '/api/v1',
    
    # UI settings
    'SWAGGER_UI_SETTINGS': {
        'deepLinking': True,
        'persistAuthorization': True,
        'displayOperationId': True,
    },
    
    # Authentication
    'SERVE_AUTHENTICATION': ['rest_framework.authentication.SessionAuthentication'],
    'SERVE_PERMISSIONS': ['rest_framework.permissions.AllowAny'],
}
```

### Models

```python
# models.py
from django.db import models
from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    """Extended user model"""
    
    ROLE_CHOICES = [
        ('admin', 'Administrator'),
        ('user', 'Regular User'),
        ('guest', 'Guest'),
    ]
    
    role = models.CharField(
        max_length=10,
        choices=ROLE_CHOICES,
        default='user'
    )
    full_name = models.CharField(max_length=100, blank=True)
    is_active = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        ordering = ['-created_at']
```

### Serializers with Documentation

```python
# serializers.py
from rest_framework import serializers
from .models import User
from drf_spectacular.utils import extend_schema_field

class UserSerializer(serializers.ModelSerializer):
    """User serializer for API responses"""
    
    class Meta:
        model = User
        fields = [
            'id', 'email', 'username', 'full_name',
            'role', 'is_active', 'created_at', 'updated_at'
        ]
        read_only_fields = ['id', 'created_at', 'updated_at']
        
        # OpenAPI documentation
        extra_kwargs = {
            'email': {
                'help_text': 'User email address',
                'example': 'john@example.com'
            },
            'username': {
                'help_text': 'Unique username',
                'example': 'johndoe',
                'min_length': 3,
                'max_length': 50
            },
            'full_name': {
                'help_text': 'User full name',
                'example': 'John Doe'
            },
            'role': {
                'help_text': 'User role in system'
            }
        }

class UserCreateSerializer(serializers.ModelSerializer):
    """Serializer for creating users"""
    
    password = serializers.CharField(
        write_only=True,
        min_length=8,
        help_text='User password (minimum 8 characters)',
        style={'input_type': 'password'}
    )
    
    class Meta:
        model = User
        fields = ['email', 'username', 'password', 'full_name', 'role']
    
    def create(self, validated_data):
        user = User.objects.create_user(**validated_data)
        return user

class UserUpdateSerializer(serializers.ModelSerializer):
    """Serializer for updating users"""
    
    class Meta:
        model = User
        fields = ['email', 'full_name', 'role', 'is_active']
```

### Views with Documentation

```python
# views.py
from rest_framework import viewsets, status, filters
from rest_framework.decorators import action
from rest_framework.response import Response
from django_filters.rest_framework import DjangoFilterBackend
from drf_spectacular.utils import (
    extend_schema,
    extend_schema_view,
    OpenApiParameter,
    OpenApiExample,
    OpenApiResponse
)

from .models import User
from .serializers import (
    UserSerializer,
    UserCreateSerializer,
    UserUpdateSerializer
)

@extend_schema_view(
    list=extend_schema(
        summary="List all users",
        description="Retrieve a paginated list of users with optional filtering",
        parameters=[
            OpenApiParameter(
                name='role',
                type=str,
                enum=['admin', 'user', 'guest'],
                description='Filter by user role'
            ),
            OpenApiParameter(
                name='is_active',
                type=bool,
                description='Filter by active status'
            ),
            OpenApiParameter(
                name='search',
                type=str,
                description='Search in username, email, and full name'
            ),
        ],
        responses={
            200: UserSerializer(many=True),
        }
    ),
    create=extend_schema(
        summary="Create a new user",
        description="Register a new user in the system",
        request=UserCreateSerializer,
        responses={
            201: UserSerializer,
            400: OpenApiResponse(description='Invalid input'),
            409: OpenApiResponse(description='User already exists')
        },
        examples=[
            OpenApiExample(
                'Create User Example',
                value={
                    'email': 'john@example.com',
                    'username': 'johndoe',
                    'password': 'SecurePass123!',
                    'full_name': 'John Doe',
                    'role': 'user'
                },
                request_only=True
            )
        ]
    ),
    retrieve=extend_schema(
        summary="Get user by ID",
        description="Retrieve detailed information about a specific user",
        responses={
            200: UserSerializer,
            404: OpenApiResponse(description='User not found')
        }
    ),
    update=extend_schema(
        summary="Update user",
        description="Update user information",
        request=UserUpdateSerializer,
        responses={
            200: UserSerializer,
            400: OpenApiResponse(description='Invalid input'),
            404: OpenApiResponse(description='User not found')
        }
    ),
    partial_update=extend_schema(
        summary="Partially update user",
        description="Update specific fields of a user",
        request=UserUpdateSerializer,
        responses={
            200: UserSerializer,
            404: OpenApiResponse(description='User not found')
        }
    ),
    destroy=extend_schema(
        summary="Delete user",
        description="Permanently remove a user from the system",
        responses={
            204: OpenApiResponse(description='User deleted'),
            404: OpenApiResponse(description='User not found')
        }
    )
)
class UserViewSet(viewsets.ModelViewSet):
    """
    ViewSet for managing users.
    
    Provides CRUD operations for user management with filtering,
    searching, and pagination capabilities.
    """
    queryset = User.objects.all()
    filter_backends = [DjangoFilterBackend, filters.SearchFilter]
    filterset_fields = ['role', 'is_active']
    search_fields = ['username', 'email', 'full_name']
    
    def get_serializer_class(self):
        if self.action == 'create':
            return UserCreateSerializer
        elif self.action in ['update', 'partial_update']:
            return UserUpdateSerializer
        return UserSerializer
    
    @extend_schema(
        summary="Get current user",
        description="Retrieve information about the currently authenticated user",
        responses={200: UserSerializer}
    )
    @action(detail=False, methods=['get'])
    def me(self, request):
        """Get current authenticated user"""
        serializer = self.get_serializer(request.user)
        return Response(serializer.data)
    
    @extend_schema(
        summary="Deactivate user",
        description="Deactivate a user account without deleting it",
        request=None,
        responses={
            200: UserSerializer,
            404: OpenApiResponse(description='User not found')
        }
    )
    @action(detail=True, methods=['post'])
    def deactivate(self, request, pk=None):
        """Deactivate user account"""
        user = self.get_object()
        user.is_active = False
        user.save()
        serializer = self.get_serializer(user)
        return Response(serializer.data)
```

### URLs

```python
# urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from drf_spectacular.views import (
    SpectacularAPIView,
    SpectacularSwaggerView,
    SpectacularRedocView
)
from .views import UserViewSet

router = DefaultRouter()
router.register(r'users', UserViewSet)

urlpatterns = [
    path('api/v1/', include(router.urls)),
    
    # OpenAPI schema
    path('api/schema/', SpectacularAPIView.as_view(), name='schema'),
    
    # Swagger UI
    path('api/docs/', SpectacularSwaggerView.as_view(url_name='schema'), name='swagger-ui'),
    
    # ReDoc
    path('api/redoc/', SpectacularRedocView.as_view(url_name='schema'), name='redoc'),
]
```

---

## Advanced Documentation Techniques

### 1. Versioning

```python
# FastAPI versioning
from fastapi import APIRouter

v1_router = APIRouter(prefix="/api/v1", tags=["v1"])
v2_router = APIRouter(prefix="/api/v2", tags=["v2"])

@v1_router.get("/users")
async def get_users_v1():
    """V1: Returns simple user list"""
    return {"version": "1.0", "users": []}

@v2_router.get("/users")
async def get_users_v2():
    """V2: Returns enhanced user list with pagination"""
    return {"version": "2.0", "users": [], "pagination": {}}

app.include_router(v1_router)
app.include_router(v2_router)
```

### 2. Response Examples

```python
from fastapi.responses import JSONResponse

@app.get(
    "/users/{user_id}",
    responses={
        200: {
            "description": "Successful Response",
            "content": {
                "application/json": {
                    "examples": {
                        "normal": {
                            "summary": "Normal user",
                            "value": {
                                "id": 1,
                                "email": "user@example.com",
                                "role": "user"
                            }
                        },
                        "admin": {
                            "summary": "Admin user",
                            "value": {
                                "id": 2,
                                "email": "admin@example.com",
                                "role": "admin"
                            }
                        }
                    }
                }
            }
        }
    }
)
async def get_user(user_id: int):
    """Get user with multiple response examples"""
    pass
```

### 3. Deprecation

```python
@app.get(
    "/old-endpoint",
    deprecated=True,
    summary="Old endpoint (deprecated)",
    description="This endpoint is deprecated. Use /new-endpoint instead."
)
async def old_endpoint():
    """Deprecated endpoint - will be removed in v2.0"""
    return {"message": "Use /new-endpoint"}
```

### 4. External Documentation

```python
@app.get(
    "/complex-operation",
    summary="Complex operation",
    description="See external docs for details",
    externalDocs={
        "description": "Detailed documentation",
        "url": "https://docs.example.com/complex-operation"
    }
)
async def complex_operation():
    """Operation with external documentation link"""
    pass
```

---

## Best Practices

### 1. Clear Descriptions

```python
# ✅ GOOD: Detailed, clear description
@app.post(
    "/users/",
    summary="Create a new user",
    description="""
    Create a new user account with the following validations:
    
    - Email must be unique and valid format
    - Username: 3-50 characters, alphanumeric
    - Password: Minimum 8 characters
    - Role: One of [admin, user, guest]
    
    Returns the created user object without password.
    """
)

# ❌ BAD: Vague description
@app.post("/users/", summary="Create user")
```

### 2. Comprehensive Examples

```python
# ✅ GOOD: Realistic examples
class User(BaseModel):
    email: EmailStr = Field(example="john.doe@company.com")
    age: int = Field(ge=18, le=120, example=30)
    
    class Config:
        json_schema_extra = {
            "example": {
                "email": "john.doe@company.com",
                "age": 30,
                "full_name": "John Doe",
                "role": "user"
            }
        }

# ❌ BAD: No examples or unrealistic ones
class User(BaseModel):
    email: EmailStr
    age: int
```

### 3. Proper Status Codes

```python
# ✅ GOOD: Document all possible responses
@app.post(
    "/users/",
    status_code=201,
    responses={
        201: {"description": "User created successfully"},
        400: {"description": "Invalid input data"},
        409: {"description": "User already exists"},
        422: {"description": "Validation error"}
    }
)

# ❌ BAD: Only default response
@app.post("/users/")
```

### 4. Group Related Endpoints

```python
# ✅ GOOD: Use tags for organization
@app.get("/users/", tags=["Users"])
@app.post("/users/", tags=["Users"])
@app.get("/users/{id}", tags=["Users"])

@app.post("/auth/login", tags=["Authentication"])
@app.post("/auth/logout", tags=["Authentication"])

# ❌ BAD: No organization
@app.get("/users/")
@app.post("/auth/login")
```

### 5. Document Security

```python
from fastapi.security import HTTPBearer, OAuth2PasswordBearer

security = HTTPBearer()
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

# ✅ GOOD: Clear security documentation
@app.get(
    "/users/me",
    tags=["Users"],
    summary="Get current user",
    description="Requires valid JWT token in Authorization header",
    dependencies=[Depends(security)]
)

# Include in OpenAPI
app = FastAPI(
    openapi_tags=[{
        "name": "Users",
        "description": "User operations. **Authentication required.**"
    }]
)
```

### 6. Validate Input Data

```python
# ✅ GOOD: Comprehensive validation
class UserCreate(BaseModel):
    email: EmailStr
    username: str = Field(
        ...,
        min_length=3,
        max_length=50,
        pattern="^[a-zA-Z0-9_-]+$",
        description="Alphanumeric username with dashes and underscores"
    )
    age: int = Field(..., ge=18, le=120)

# ❌ BAD: No validation
class UserCreate(BaseModel):
    email: str
    username: str
    age: int
```

### 7. Pagination Documentation

```python
# ✅ GOOD: Clear pagination
@app.get(
    "/users/",
    summary="List users with pagination"
)
async def list_users(
    page: int = Query(1, ge=1, description="Page number"),
    page_size: int = Query(10, ge=1, le=100, description="Items per page")
) -> UserList:
    """
    Returns paginated user list.
    
    ### Pagination
    - Default page size: 10
    - Maximum page size: 100
    - Pages are 1-indexed
    """
    pass
```

---

## Testing Documentation

### 1. Validate OpenAPI Schema

```python
# test_openapi.py
import pytest
from fastapi.testclient import TestClient
from jsonschema import validate
from main import app

client = TestClient(app)

def test_openapi_schema():
    """Test that OpenAPI schema is valid"""
    response = client.get("/openapi.json")
    assert response.status_code == 200
    
    schema = response.json()
    assert schema["openapi"] == "3.0.2"
    assert "paths" in schema
    assert "components" in schema

def test_all_endpoints_documented():
    """Ensure all endpoints have documentation"""
    response = client.get("/openapi.json")
    schema = response.json()
    
    for path, methods in schema["paths"].items():
        for method, details in methods.items():
            assert "summary" in details, f"{method.upper()} {path} missing summary"
            assert "description" in details, f"{method.upper()} {path} missing description"
```

### 2. Test Examples

```python
def test_example_requests():
    """Test that example requests work"""
    response = client.get("/openapi.json")
    schema = response.json()
    
    # Get example from schema
    create_user_schema = schema["paths"]["/users/"]["post"]
    example = create_user_schema["requestBody"]["content"]["application/json"]["example"]
    
    # Test with example
    response = client.post("/users/", json=example)
    assert response.status_code in [200, 201]
```

### 3. Documentation Coverage

```python
def test_response_models():
    """Ensure all endpoints have response models"""
    response = client.get("/openapi.json")
    schema = response.json()
    
    for path, methods in schema["paths"].items():
        for method, details in methods.items():
            assert "responses" in details
            assert "200" in details["responses"] or "201" in details["responses"]
```

---

## Conclusion

Good API documentation is essential for API adoption and developer experience. Key takeaways:

1. **Use frameworks with built-in documentation** (FastAPI, Flask-RESTX, DRF)
2. **Follow OpenAPI specification** for standardization
3. **Provide clear descriptions and examples** for all endpoints
4. **Document all possible responses** including error cases
5. **Keep documentation in sync with code** through auto-generation
6. **Make documentation interactive** with Swagger UI/ReDoc
7. **Test your documentation** to ensure accuracy

### Framework Recommendations

- **FastAPI**: Best for new projects, excellent auto-documentation
- **Flask-RESTX**: Good for Flask applications, flexible
- **DRF + drf-spectacular**: Ideal for Django projects

Choose based on your existing stack and project requirements. All three provide excellent documentation capabilities when used correctly.