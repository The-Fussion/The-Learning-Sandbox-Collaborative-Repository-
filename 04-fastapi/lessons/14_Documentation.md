# Documentation in FastAPI

## 14.1 Automatic Documentation

### Swagger UI Customization

FastAPI automatically generates interactive API documentation using Swagger UI. You can customize its appearance and behavior.

```python
from fastapi import FastAPI

# Basic Swagger UI customization
app = FastAPI(
    title="My API",
    description="A comprehensive API for managing resources",
    version="1.0.0",
    docs_url="/docs",  # Swagger UI URL (default: /docs)
    redoc_url="/redoc",  # ReDoc URL (default: /redoc)
    openapi_url="/openapi.json",  # OpenAPI schema URL
)

@app.get("/")
async def root():
    return {"message": "Hello World"}
```

**Advanced Swagger UI Customization:**

```python
from fastapi import FastAPI
from fastapi.openapi.docs import get_swagger_ui_html
from fastapi.openapi.utils import get_openapi

app = FastAPI(docs_url=None, redoc_url=None)  # Disable default docs

@app.get("/docs", include_in_schema=False)
async def custom_swagger_ui_html():
    return get_swagger_ui_html(
        openapi_url=app.openapi_url,
        title=app.title + " - Swagger UI",
        oauth2_redirect_url=app.swagger_ui_oauth2_redirect_url,
        swagger_js_url="https://cdn.jsdelivr.net/npm/swagger-ui-dist@5/swagger-ui-bundle.js",
        swagger_css_url="https://cdn.jsdelivr.net/npm/swagger-ui-dist@5/swagger-ui.css",
        swagger_favicon_url="https://fastapi.tiangolo.com/img/favicon.png",
        swagger_ui_parameters={
            "deepLinking": True,
            "displayRequestDuration": True,
            "filter": True,
            "showExtensions": True,
            "showCommonExtensions": True,
            "syntaxHighlight.theme": "monokai",
        }
    )

# Custom API metadata
app = FastAPI(
    title="Product Management API",
    description="""
    ## Features
    
    This API allows you to manage products in your e-commerce store.
    
    * **Create products** - Add new products to inventory
    * **Read products** - Retrieve product information
    * **Update products** - Modify existing products
    * **Delete products** - Remove products from inventory
    
    ## Authentication
    
    This API uses OAuth2 with password flow for authentication.
    """,
    version="2.0.0",
    terms_of_service="https://example.com/terms/",
    contact={
        "name": "API Support",
        "url": "https://example.com/support",
        "email": "support@example.com",
    },
    license_info={
        "name": "MIT",
        "url": "https://opensource.org/licenses/MIT",
    },
)
```

### ReDoc Customization

ReDoc provides an alternative documentation interface with a different layout.

```python
from fastapi import FastAPI
from fastapi.openapi.docs import get_redoc_html

app = FastAPI(redoc_url=None)  # Disable default ReDoc

@app.get("/redoc", include_in_schema=False)
async def redoc_html():
    return get_redoc_html(
        openapi_url=app.openapi_url,
        title=app.title + " - ReDoc",
        redoc_js_url="https://cdn.jsdelivr.net/npm/redoc@next/bundles/redoc.standalone.js",
        redoc_favicon_url="https://fastapi.tiangolo.com/img/favicon.png",
        with_google_fonts=True,
    )

# Hide docs in production
import os

app = FastAPI(
    docs_url="/docs" if os.getenv("ENVIRONMENT") == "development" else None,
    redoc_url="/redoc" if os.getenv("ENVIRONMENT") == "development" else None,
)
```

### OpenAPI Schema

Access and customize the OpenAPI schema that powers the documentation.

```python
from fastapi import FastAPI
from fastapi.openapi.utils import get_openapi

app = FastAPI()

def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema
    
    openapi_schema = get_openapi(
        title="Custom API",
        version="2.5.0",
        description="This is a custom OpenAPI schema",
        routes=app.routes,
    )
    
    # Add custom fields
    openapi_schema["info"]["x-logo"] = {
        "url": "https://fastapi.tiangolo.com/img/logo-margin/logo-teal.png"
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

@app.get("/")
async def root():
    return {"message": "Hello World"}
```

### Metadata and Tags

Organize your API endpoints using tags and add metadata.

```python
from fastapi import FastAPI

# Define tags metadata
tags_metadata = [
    {
        "name": "users",
        "description": "Operations with users. Manage user accounts and profiles.",
    },
    {
        "name": "items",
        "description": "Manage items in the inventory.",
        "externalDocs": {
            "description": "Items external docs",
            "url": "https://example.com/docs/items",
        },
    },
    {
        "name": "admin",
        "description": "Administrative operations. **Requires admin privileges.**",
    },
]

app = FastAPI(
    title="E-Commerce API",
    description="API for managing an e-commerce platform",
    version="1.0.0",
    openapi_tags=tags_metadata,
)

# Use tags in endpoints
@app.get("/users/", tags=["users"])
async def read_users():
    return [{"username": "john"}, {"username": "jane"}]

@app.post("/users/", tags=["users"])
async def create_user(username: str, email: str):
    return {"username": username, "email": email}

@app.get("/items/", tags=["items"])
async def read_items():
    return [{"name": "Widget"}, {"name": "Gadget"}]

@app.delete("/admin/users/{user_id}", tags=["admin"])
async def delete_user(user_id: int):
    return {"message": f"User {user_id} deleted"}

# Multiple tags
@app.get("/admin/items/", tags=["admin", "items"])
async def admin_items():
    return {"message": "Admin items view"}
```

### Operation Descriptions

Add detailed descriptions to your endpoints.

```python
from fastapi import FastAPI, Query, Path, Body
from pydantic import BaseModel, Field

app = FastAPI()

class Item(BaseModel):
    name: str = Field(..., description="The name of the item")
    description: str | None = Field(None, description="A detailed description of the item")
    price: float = Field(..., gt=0, description="The price must be greater than zero")
    tax: float | None = Field(None, description="Optional tax amount")

@app.post(
    "/items/",
    summary="Create an item",
    description="Create a new item with all the required information: name, price, and optional description and tax.",
    response_description="The created item with generated ID",
)
async def create_item(item: Item):
    """
    Create an item with the following information:
    
    - **name**: The name of the item (required)
    - **description**: A long description (optional)
    - **price**: The price of the item (required, must be greater than 0)
    - **tax**: Tax amount (optional)
    
    This endpoint will return the created item with a generated ID.
    """
    return {"item_id": 1, **item.dict()}

@app.get(
    "/items/{item_id}",
    summary="Get an item",
    description="Retrieve a specific item by its ID",
    response_description="The requested item",
)
async def read_item(
    item_id: int = Path(..., description="The ID of the item to retrieve", ge=1),
    q: str | None = Query(None, description="Optional search query"),
):
    """
    Get an item by ID.
    
    - **item_id**: The unique identifier of the item
    - **q**: Optional query parameter for filtering
    """
    return {"item_id": item_id, "q": q}

# Deprecate endpoints
@app.get("/old-items/", deprecated=True)
async def read_old_items():
    """This endpoint is deprecated. Use /items/ instead."""
    return [{"name": "Old Widget"}]
```

### Example Values

Provide example values for request and response models.

```python
from pydantic import BaseModel, Field
from typing import List

class User(BaseModel):
    id: int = Field(..., description="Unique user identifier", example=123)
    username: str = Field(..., description="Username", example="johndoe")
    email: str = Field(..., description="Email address", example="john@example.com")
    full_name: str | None = Field(None, description="Full name", example="John Doe")
    
    class Config:
        json_schema_extra = {
            "examples": [
                {
                    "id": 1,
                    "username": "alice",
                    "email": "alice@example.com",
                    "full_name": "Alice Wonderland"
                },
                {
                    "id": 2,
                    "username": "bob",
                    "email": "bob@example.com",
                    "full_name": "Bob Builder"
                }
            ]
        }

class Product(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None
    tags: List[str] = []
    
    model_config = {
        "json_schema_extra": {
            "examples": [
                {
                    "name": "Premium Widget",
                    "description": "A high-quality widget with advanced features",
                    "price": 99.99,
                    "tax": 9.99,
                    "tags": ["premium", "featured", "electronics"]
                }
            ]
        }
    }

@app.post("/users/")
async def create_user(user: User):
    return user

@app.post("/products/")
async def create_product(
    product: Product = Body(
        ...,
        examples={
            "normal": {
                "summary": "A normal product",
                "description": "A standard product with typical values",
                "value": {
                    "name": "Widget",
                    "description": "A useful widget",
                    "price": 19.99,
                    "tax": 1.99,
                    "tags": ["standard"]
                }
            },
            "premium": {
                "summary": "A premium product",
                "description": "A high-end product with premium features",
                "value": {
                    "name": "Premium Gadget",
                    "description": "Top-of-the-line gadget",
                    "price": 299.99,
                    "tax": 29.99,
                    "tags": ["premium", "featured"]
                }
            },
            "sale": {
                "summary": "A sale product",
                "description": "A product currently on sale",
                "value": {
                    "name": "Sale Item",
                    "description": "Limited time offer",
                    "price": 9.99,
                    "tax": 0.99,
                    "tags": ["sale", "clearance"]
                }
            }
        }
    )
):
    return product
```

## 14.2 API Documentation Best Practices

### Endpoint Descriptions

Write clear, comprehensive descriptions for all endpoints.

```python
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel

app = FastAPI()

class UserCreate(BaseModel):
    username: str
    email: str
    password: str

class UserResponse(BaseModel):
    id: int
    username: str
    email: str

@app.post(
    "/users/register",
    response_model=UserResponse,
    status_code=status.HTTP_201_CREATED,
    summary="Register a new user",
    description="""
    ## Register a New User
    
    This endpoint allows you to create a new user account.
    
    ### Requirements
    - Username must be unique
    - Email must be valid and unique
    - Password must be at least 8 characters
    
    ### Process
    1. Validates the input data
    2. Checks for existing username/email
    3. Hashes the password securely
    4. Creates the user in the database
    5. Returns the created user (without password)
    
    ### Notes
    - The password is never returned in responses
    - User activation email is sent automatically
    - Rate limiting applies: 5 registrations per hour per IP
    """,
    response_description="The newly created user object",
    tags=["Authentication"],
    responses={
        201: {
            "description": "User successfully created",
            "content": {
                "application/json": {
                    "example": {
                        "id": 1,
                        "username": "johndoe",
                        "email": "john@example.com"
                    }
                }
            }
        },
        400: {
            "description": "Invalid input data",
            "content": {
                "application/json": {
                    "example": {
                        "detail": "Username already exists"
                    }
                }
            }
        },
        422: {
            "description": "Validation error",
            "content": {
                "application/json": {
                    "example": {
                        "detail": [
                            {
                                "loc": ["body", "email"],
                                "msg": "Invalid email format",
                                "type": "value_error"
                            }
                        ]
                    }
                }
            }
        }
    }
)
async def register_user(user: UserCreate):
    """
    Register a new user in the system.
    
    This creates a new user account with the provided credentials.
    """
    # Implementation here
    return UserResponse(id=1, username=user.username, email=user.email)
```

### Request/Response Examples

Provide comprehensive examples for requests and responses.

```python
from typing import List
from pydantic import BaseModel, Field

class OrderItem(BaseModel):
    product_id: int = Field(..., description="Product identifier", example=101)
    quantity: int = Field(..., ge=1, description="Quantity to order", example=2)
    price: float = Field(..., gt=0, description="Unit price", example=29.99)

class Order(BaseModel):
    customer_id: int = Field(..., description="Customer identifier", example=1)
    items: List[OrderItem] = Field(..., description="List of items in the order")
    notes: str | None = Field(None, description="Optional order notes", example="Please gift wrap")
    
    model_config = {
        "json_schema_extra": {
            "examples": [
                {
                    "customer_id": 1,
                    "items": [
                        {"product_id": 101, "quantity": 2, "price": 29.99},
                        {"product_id": 102, "quantity": 1, "price": 49.99}
                    ],
                    "notes": "Deliver between 9 AM - 5 PM"
                }
            ]
        }
    }

class OrderResponse(BaseModel):
    order_id: int
    customer_id: int
    total: float
    status: str
    
    model_config = {
        "json_schema_extra": {
            "examples": [
                {
                    "order_id": 12345,
                    "customer_id": 1,
                    "total": 109.97,
                    "status": "pending"
                }
            ]
        }
    }

@app.post(
    "/orders/",
    response_model=OrderResponse,
    summary="Create a new order",
    responses={
        200: {
            "description": "Order created successfully",
            "content": {
                "application/json": {
                    "examples": {
                        "success": {
                            "summary": "Successful order",
                            "value": {
                                "order_id": 12345,
                                "customer_id": 1,
                                "total": 109.97,
                                "status": "pending"
                            }
                        }
                    }
                }
            }
        },
        400: {
            "description": "Invalid order data",
            "content": {
                "application/json": {
                    "examples": {
                        "empty_cart": {
                            "summary": "Empty cart",
                            "value": {"detail": "Order must contain at least one item"}
                        },
                        "invalid_product": {
                            "summary": "Invalid product",
                            "value": {"detail": "Product 999 not found"}
                        }
                    }
                }
            }
        }
    }
)
async def create_order(order: Order):
    """Create a new order for a customer."""
    total = sum(item.quantity * item.price for item in order.items)
    return OrderResponse(
        order_id=12345,
        customer_id=order.customer_id,
        total=total,
        status="pending"
    )
```

### Error Documentation

Document all possible error responses.

```python
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel

app = FastAPI()

class ErrorResponse(BaseModel):
    detail: str
    error_code: str | None = None
    
    model_config = {
        "json_schema_extra": {
            "examples": [
                {
                    "detail": "Resource not found",
                    "error_code": "RESOURCE_NOT_FOUND"
                }
            ]
        }
    }

@app.get(
    "/users/{user_id}",
    summary="Get user by ID",
    responses={
        200: {
            "description": "User found",
            "content": {
                "application/json": {
                    "example": {
                        "id": 1,
                        "username": "johndoe",
                        "email": "john@example.com"
                    }
                }
            }
        },
        404: {
            "description": "User not found",
            "model": ErrorResponse,
            "content": {
                "application/json": {
                    "example": {
                        "detail": "User with ID 999 not found",
                        "error_code": "USER_NOT_FOUND"
                    }
                }
            }
        },
        401: {
            "description": "Not authenticated",
            "model": ErrorResponse,
            "content": {
                "application/json": {
                    "example": {
                        "detail": "Not authenticated",
                        "error_code": "NOT_AUTHENTICATED"
                    }
                }
            }
        },
        403: {
            "description": "Not authorized to access this resource",
            "model": ErrorResponse,
            "content": {
                "application/json": {
                    "example": {
                        "detail": "Insufficient permissions",
                        "error_code": "FORBIDDEN"
                    }
                }
            }
        },
        500: {
            "description": "Internal server error",
            "model": ErrorResponse,
            "content": {
                "application/json": {
                    "example": {
                        "detail": "An unexpected error occurred",
                        "error_code": "INTERNAL_ERROR"
                    }
                }
            }
        }
    }
)
async def get_user(user_id: int):
    """
    Retrieve a user by their ID.
    
    ## Possible Errors
    
    - **404**: User not found - The requested user ID doesn't exist
    - **401**: Not authenticated - Valid authentication required
    - **403**: Forbidden - User lacks permission to view this user
    - **500**: Internal error - Something went wrong on the server
    """
    if user_id == 999:
        raise HTTPException(
            status_code=404,
            detail="User with ID 999 not found"
        )
    
    return {"id": user_id, "username": "johndoe", "email": "john@example.com"}
```

### Authentication Documentation

Document authentication requirements clearly.

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm

app = FastAPI(
    title="Secure API",
    description="""
    # Authentication
    
    This API uses OAuth2 with Password flow for authentication.
    
    ## How to Authenticate
    
    1. **Get Access Token**
       - Send POST request to `/token` with username and password
       - Receive an access token in the response
    
    2. **Use Access Token**
       - Include token in Authorization header: `Authorization: Bearer <token>`
       - Token is valid for 30 minutes
    
    3. **Refresh Token**
       - When token expires, request a new one using `/token`
    
    ## Example
    
    ```bash
    # Get token
    curl -X POST "http://api.example.com/token" \\
      -H "Content-Type: application/x-www-form-urlencoded" \\
      -d "username=johndoe&password=secret"
    
    # Use token
    curl -X GET "http://api.example.com/users/me" \\
      -H "Authorization: Bearer <your_token>"
    ```
    """,
)

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

@app.post(
    "/token",
    summary="Login to get access token",
    description="""
    ## Login Endpoint
    
    Authenticate with username and password to receive an access token.
    
    ### Request Body
    - **username**: Your username
    - **password**: Your password
    
    ### Response
    - **access_token**: JWT token for authentication
    - **token_type**: Always "bearer"
    
    ### Token Lifetime
    - Access tokens expire after 30 minutes
    - Request a new token when expired
    """,
    tags=["Authentication"],
    response_description="Access token and token type",
)
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    """
    OAuth2 compatible token login.
    
    Returns an access token that can be used for subsequent requests.
    """
    # Simplified - implement actual authentication
    if form_data.username != "johndoe" or form_data.password != "secret":
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    
    return {"access_token": "fake-token", "token_type": "bearer"}

@app.get(
    "/users/me",
    summary="Get current user",
    description="""
    Get information about the currently authenticated user.
    
    **Authentication Required**: This endpoint requires a valid access token.
    """,
    tags=["Users"],
)
async def read_users_me(token: str = Depends(oauth2_scheme)):
    """Get current user information based on the provided access token."""
    return {"username": "johndoe", "email": "john@example.com"}
```

### Versioning Documentation

Document API versioning strategy.

```python
from fastapi import FastAPI, APIRouter

# Version 1
v1_router = APIRouter(prefix="/v1", tags=["V1"])

@v1_router.get(
    "/users/",
    summary="Get users (V1 - Deprecated)",
    deprecated=True,
    description="""
    **DEPRECATED**: This endpoint is deprecated and will be removed in version 3.0.
    
    Please use `/v2/users/` instead.
    
    ## Migration Guide
    The V2 endpoint includes additional fields:
    - `created_at`: User creation timestamp
    - `updated_at`: Last update timestamp
    - `is_active`: Account status
    """,
)
async def get_users_v1():
    return [{"id": 1, "username": "john"}]

# Version 2
v2_router = APIRouter(prefix="/v2", tags=["V2"])

@v2_router.get(
    "/users/",
    summary="Get users (V2 - Current)",
    description="""
    Get a list of all users with extended information.
    
    ## Changes from V1
    - Added `created_at` field
    - Added `updated_at` field
    - Added `is_active` field
    - Improved response structure
    """,
)
async def get_users_v2():
    return [
        {
            "id": 1,
            "username": "john",
            "created_at": "2024-01-01T00:00:00Z",
            "updated_at": "2024-01-15T10:30:00Z",
            "is_active": True
        }
    ]

app = FastAPI(
    title="Versioned API",
    description="""
    # API Versioning
    
    This API uses URL path versioning.
    
    ## Current Version: 2.0
    
    ## Version History
    
    ### Version 2.0 (Current)
    - Enhanced user endpoints with timestamps
    - Added account status field
    - Improved error messages
    
    ### Version 1.0 (Deprecated)
    - Basic user management
    - Will be removed in version 3.0
    
    ## Migration
    
    If you're using V1 endpoints, please migrate to V2 before the deprecation date.
    See individual endpoint documentation for migration guides.
    """,
    version="2.0.0",
)

app.include_router(v1_router)
app.include_router(v2_router)
```

## 14.3 Custom Documentation

### Custom OpenAPI Schema

Customize the OpenAPI schema to add additional information.

```python
from fastapi import FastAPI
from fastapi.openapi.utils import get_openapi

app = FastAPI()

def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema
    
    openapi_schema = get_openapi(
        title="Custom API Documentation",
        version="1.0.0",
        description="API with custom OpenAPI schema",
        routes=app.routes,
    )
    
    # Add custom extensions
    openapi_schema["info"]["x-logo"] = {
        "url": "https://example.com/logo.png",
        "altText": "Company Logo"
    }
    
    # Add custom server information
    openapi_schema["servers"] = [
        {
            "url": "https://api.example.com",
            "description": "Production server"
        },
        {
            "url": "https://staging-api.example.com",
            "description": "Staging server"
        },
        {
            "url": "http://localhost:8000",
            "description": "Development server"
        }
    ]
    
    # Add custom security schemes
    openapi_schema["components"]["securitySchemes"] = {
        "BearerAuth": {
            "type": "http",
            "scheme": "bearer",
            "bearerFormat": "JWT",
            "description": "Enter your JWT token"
        },
        "ApiKeyAuth": {
            "type": "apiKey",
            "in": "header",
            "name": "X-API-Key",
            "description": "Enter your API key"
        }
    }
    
    # Add global security requirement
    openapi_schema["security"] = [
        {"BearerAuth": []}
    ]
    
    # Add custom vendor extensions
    openapi_schema["x-api-id"] = "api-12345"
    openapi_schema["x-audience"] = "external"
    
    app.openapi_schema = openapi_schema
    return app.openapi_schema

app.openapi = custom_openapi

@app.get("/items/")
async def read_items():
    return [{"name": "Item 1"}]
```

### Additional Documentation Pages

Add custom documentation pages alongside the auto-generated docs.

```python
from fastapi import FastAPI
from fastapi.responses import HTMLResponse

app = FastAPI()

@app.get("/docs/getting-started", include_in_schema=False)
async def getting_started():
    html_content = """
    <!DOCTYPE html>
    <html>
    <head>
        <title>Getting Started - API Documentation</title>
        <style>
            body {
                font-family: Arial, sans-serif;
                max-width: 800px;
                margin: 50px auto;
                padding: 20px;
                line-height: 1.6;
            }
            h1 { color: #009688; }
            h2 { color: #00796b; }
            code {
                background: #f4f4f4;
                padding: 2px 6px;
                border-radius: 3px;
            }
            pre {
                background: #f4f4f4;
                padding: 15px;
                border-radius: 5px;
                overflow-x: auto;
            }
        </style>
    </head>
    <body>
        <h1>Getting Started with Our API</h1>
        
        <h2>1. Authentication</h2>
        <p>To use the API, you need to authenticate:</p>
        <pre><code>curl -X POST "https://api.example.com/token" \\
  -H "Content-Type: application/x-www-form-urlencoded" \\
  -d "username=your_username&password=your_password"</code></pre>
        
        <h2>2. Making Your First Request</h2>
        <p>Use the token to make authenticated requests:</p>
        <pre><code>curl -X GET "https://api.example.com/users/me" \\
  -H "Authorization: Bearer YOUR_TOKEN"</code></pre>
        
        <h2>3. Error Handling</h2>
        <p>The API uses standard HTTP status codes:</p>
        <ul>
            <li><code>200</code> - Success</li>
            <li><code>400</code> - Bad Request</li>
            <li><code>401</code> - Unauthorized</li>
            <li><code>404</code> - Not Found</li>
            <li><code>500</code> - Server Error</li>
        </ul>
        
        <h2>4. Rate Limits</h2>
        <p>API requests are limited to:</p>
        <ul>
            <li>100 requests per minute for authenticated users</li>
            <li>10 requests per minute for unauthenticated requests</li>
        </ul>
        
        <h2>Need Help?</h2>
        <p>Contact us at <a href="mailto:support@example.com">support@example.com</a></p>
        
        <p><a href="/docs">Back to API Documentation</a></p>
    </body>
    </html>
    """
    return HTMLResponse(content=html_content)

@app.get("/docs/examples", include_in_schema=False)
async def examples():
    html_content = """
    <!DOCTYPE html>
    <html>
    <head>
        <title>API Examples</title>
        <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.7.0/styles/default.min.css">
        <script src="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.7.0/highlight.min.js"></script>
        <script>hljs.highlightAll();</script>
        <style>
            body { font-family: Arial, sans-serif; max-width: 900px; margin: 0 auto; padding: 20px; }
            h1 { color: #009688; }
            h2 { color: #00796b; margin-top: 30px; }
            pre { border-radius: 5px; }
        </style>
    </head>
    <body>
        <h1>API Usage Examples</h1>
        
        <h2>Python Example</h2>
        <pre><code class="language-python">import requests

# Authenticate
response = requests.post(
    "https://api.example.com/token",
    data={"username": "user", "password": "pass"}
)
token = response.json()["access_token"]

# Make authenticated request
headers = {"Authorization": f"Bearer {token}"}
users = requests.get("https://api.example.com/users", headers=headers)
print(users.json())</code></pre>
        
        <h2>JavaScript Example</h2>
        <pre><code class="language-javascript">// Authenticate
const tokenResponse = await fetch('https://api.example.com/token', {
  method: 'POST',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
  body: 'username=user&password=pass'
});
const { access_token } = await tokenResponse.json();

// Make authenticated request
const usersResponse = await fetch('https://api.example.com/users', {
  headers: { 'Authorization': `Bearer ${access_token}` }
});
const users = await usersResponse.json();
console.log(users);</code></pre>
        
        <h2>cURL Example</h2>
        <pre><code class="language-bash"># Authenticate and save token
TOKEN=$(curl -s -X POST "https://api.example.com/token" \\
  -H "Content-Type: application/x-www-form-urlencoded" \\
  -d "username=user&password=pass" | jq -r '.access_token')

# Use token
curl -X GET "https://api.example.com/users" \\
  -H "Authorization: Bearer $TOKEN"</code></pre>
        
        <p><a href="/docs">Back to API Documentation</a></p>
    </body>
    </html>
    """
    return HTMLResponse(content=html_content)
```

### External Documentation Links

Link to external documentation and resources.

```python
from fastapi import FastAPI

app = FastAPI(
    title="API with External Docs",
    description="""
    ## External Resources
    
    - [Getting Started Guide](https://docs.example.com/getting-started)
    - [API Reference](https://docs.example.com/api-reference)
    - [Tutorials](https://docs.example.com/tutorials)
    - [GitHub Repository](https://github.com/example/api)
    - [Community Forum](https://forum.example.com)
    """,
    docs_url="/api/docs",
    redoc_url="/api/redoc",
)

# Add external docs to specific tags
tags_metadata = [
    {
        "name": "users",
        "description": "User management endpoints",
        "externalDocs": {
            "description": "User Management Guide",
            "url": "https://docs.example.com/users",
        },
    },
    {
        "name": "products",
        "description": "Product management endpoints",
        "externalDocs": {
            "description": "Product API Guide",
            "url": "https://docs.example.com/products",
        },
    },
]

app = FastAPI(openapi_tags=tags_metadata)

@app.get("/users/", tags=["users"])
async def get_users():
    """
    Get all users.
    
    For more information, see the [User Management Guide](https://docs.example.com/users).
    """
    return [{"id": 1, "name": "John"}]

@app.get("/products/", tags=["products"])
async def get_products():
    """
    Get all products.
    
    See also:
    - [Product API Guide](https://docs.example.com/products)
    - [Product Examples](https://docs.example.com/product-examples)
    """
    return [{"id": 1, "name": "Widget"}]
```

### API Changelog

Maintain and display an API changelog.

```python
from fastapi import FastAPI
from fastapi.responses import HTMLResponse

app = FastAPI()

@app.get("/changelog", include_in_schema=False)
async def changelog():
    html_content = """
    <!DOCTYPE html>
    <html>
    <head>
        <title>API Changelog</title>
        <style>
            body {
                font-family: Arial, sans-serif;
                max-width: 800px;
                margin: 50px auto;
                padding: 20px;
                line-height: 1.6;
            }
            h1 { color: #009688; }
            h2 { 
                color: #00796b;
                border-bottom: 2px solid #009688;
                padding-bottom: 5px;
            }
            .version {
                background: #e0f2f1;
                padding: 10px;
                border-radius: 5px;
                margin: 20px 0;
            }
            .date {
                color: #666;
                font-size: 0.9em;
            }
            .badge {
                display: inline-block;
                padding: 3px 8px;
                border-radius: 3px;
                font-size: 0.85em;
                font-weight: bold;
                margin-right: 5px;
            }
            .added { background: #4caf50; color: white; }
            .changed { background: #ff9800; color: white; }
            .deprecated { background: #f44336; color: white; }
            .fixed { background: #2196f3; color: white; }
            .removed { background: #9e9e9e; color: white; }
        </style>
    </head>
    <body>
        <h1>API Changelog</h1>
        <p>Track all changes, improvements, and deprecations to our API.</p>
        
        <div class="version">
            <h2>Version 2.1.0 <span class="date">(2024-01-15)</span></h2>
            
            <h3><span class="badge added">ADDED</span></h3>
            <ul>
                <li>New <code>/users/{id}/preferences</code> endpoint for user preferences</li>
                <li>Support for batch operations in <code>/products</code></li>
                <li>Webhook notifications for order status changes</li>
            </ul>
            
            <h3><span class="badge changed">CHANGED</span></h3>
            <ul>
                <li>Improved performance of <code>/search</code> endpoint (30% faster)</li>
                <li>Updated rate limits: 100 → 150 requests per minute for authenticated users</li>
                <li>Enhanced error messages with more detailed information</li>
            </ul>
            
            <h3><span class="badge fixed">FIXED</span></h3>
            <ul>
                <li>Fixed pagination issue in <code>/products</code> endpoint</li>
                <li>Resolved timezone handling in date filters</li>
            </ul>
        </div>
        
        <div class="version">
            <h2>Version 2.0.0 <span class="date">(2024-01-01)</span></h2>
            
            <h3><span class="badge added">ADDED</span></h3>
            <ul>
                <li>New API versioning scheme (v1, v2)</li>
                <li>OAuth2 authentication support</li>
                <li>Comprehensive error codes</li>
            </ul>
            
            <h3><span class="badge changed">CHANGED</span></h3>
            <ul>
                <li>Renamed <code>user_name</code> to <code>username</code> in all endpoints</li>
                <li>Response format now includes metadata object</li>
            </ul>
            
            <h3><span class="badge deprecated">DEPRECATED</span></h3>
            <ul>
                <li><code>/v1/users</code> - Use <code>/v2/users</code> instead</li>
                <li>API key authentication - Migrate to OAuth2</li>
            </ul>
            
            <h3><span class="badge removed">REMOVED</span></h3>
            <ul>
                <li>Legacy <code>/auth/login</code> endpoint (use <code>/token</code>)</li>
            </ul>
        </div>
        
        <div class="version">
            <h2>Version 1.5.0 <span class="date">(2023-12-01)</span></h2>
            
            <h3><span class="badge added">ADDED</span></h3>
            <ul>
                <li>File upload support in <code>/attachments</code></li>
                <li>Export functionality for reports</li>
            </ul>
            
            <h3><span class="badge fixed">FIXED</span></h3>
            <ul>
                <li>Memory leak in long-running connections</li>
                <li>CORS configuration for new domains</li>
            </ul>
        </div>
        
        <p><a href="/docs">Back to API Documentation</a></p>
    </body>
    </html>
    """
    return HTMLResponse(content=html_content)

# Include changelog link in API description
app = FastAPI(
    title="Documented API",
    description="""
    Welcome to our API documentation.
    
    ## Resources
    - [API Changelog](/changelog) - See what's new
    - [Getting Started](/docs/getting-started)
    - [Examples](/docs/examples)
    """,
)
```

---

## Complete Documentation Example

```python
from fastapi import FastAPI, HTTPException, Depends, status
from fastapi.openapi.utils import get_openapi
from pydantic import BaseModel, Field
from typing import List

# Models with examples
class User(BaseModel):
    id: int = Field(..., description="Unique user identifier", example=1)
    username: str = Field(..., description="Username", min_length=3, max_length=50, example="johndoe")
    email: str = Field(..., description="Email address", example="john@example.com")
    is_active: bool = Field(True, description="Account status", example=True)
    
    model_config = {
        "json_schema_extra": {
            "examples": [
                {
                    "id": 1,
                    "username": "johndoe",
                    "email": "john@example.com",
                    "is_active": True
                }
            ]
        }
    }

class UserCreate(BaseModel):
    username: str = Field(..., min_length=3, max_length=50)
    email: str
    password: str = Field(..., min_length=8)

# Tags metadata
tags_metadata = [
    {
        "name": "users",
        "description": "Operations with users. **Authentication required for most operations.**",
        "externalDocs": {
            "description": "User guide",
            "url": "https://docs.example.com/users",
        },
    },
    {
        "name": "admin",
        "description": "Administrative operations. **Requires admin role.**",
    },
]

# Create app with full configuration
app = FastAPI(
    title="Well-Documented API",
    description="""
    ## Overview
    
    This API provides comprehensive user management functionality.
    
    ## Features
    
    * 🔐 **Secure Authentication** - OAuth2 with JWT tokens
    * 👥 **User Management** - Full CRUD operations
    * 📊 **Admin Dashboard** - Administrative controls
    * 🔔 **Real-time Notifications** - WebSocket support
    
    ## Authentication
    
    Use the `/token` endpoint to obtain an access token, then include it in the
    `Authorization` header as `Bearer <token>`.
    
    ## Rate Limiting
    
    - Authenticated: 100 requests/minute
    - Unauthenticated: 10 requests/minute
    
    ## Support
    
    For issues or questions, contact us at support@example.com
    """,
    version="2.1.0",
    terms_of_service="https://example.com/terms",
    contact={
        "name": "API Support Team",
        "url": "https://example.com/support",
        "email": "support@example.com",
    },
    license_info={
        "name": "MIT",
        "url": "https://opensource.org/licenses/MIT",
    },
    openapi_tags=tags_metadata,
)

# Custom OpenAPI schema
def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema
    
    openapi_schema = get_openapi(
        title=app.title,
        version=app.version,
        description=app.description,
        routes=app.routes,
    )
    
    openapi_schema["info"]["x-logo"] = {
        "url": "https://fastapi.tiangolo.com/img/logo-margin/logo-teal.png"
    }
    
    app.openapi_schema = openapi_schema
    return app.openapi_schema

app.openapi = custom_openapi

# Well-documented endpoints
@app.post(
    "/users/",
    response_model=User,
    status_code=status.HTTP_201_CREATED,
    tags=["users"],
    summary="Create a new user",
    description="Register a new user account with username, email, and password.",
    response_description="The newly created user",
    responses={
        201: {
            "description": "User successfully created",
            "content": {
                "application/json": {
                    "example": {
                        "id": 1,
                        "username": "johndoe",
                        "email": "john@example.com",
                        "is_active": True
                    }
                }
            }
        },
        400: {"description": "Invalid input or username/email already exists"},
        422: {"description": "Validation error"},
    }
)
async def create_user(user: UserCreate):
    """
    Create a new user with all the required information:
    
    - **username**: Unique username (3-50 characters)
    - **email**: Valid email address
    - **password**: Secure password (minimum 8 characters)
    
    Returns the created user object with generated ID.
    """
    return User(id=1, username=user.username, email=user.email, is_active=True)

@app.get(
    "/users/",
    response_model=List[User],
    tags=["users"],
    summary="List all users",
    description="Retrieve a list of all users with pagination support.",
)
async def list_users(
    skip: int = 0,
    limit: int = 100
):
    """Get a paginated list of users."""
    return [
        User(id=1, username="johndoe", email="john@example.com", is_active=True),
        User(id=2, username="janedoe", email="jane@example.com", is_active=True),
    ]

@app.get(
    "/users/{user_id}",
    response_model=User,
    tags=["users"],
    summary="Get a specific user",
    responses={
        200: {"description": "User found"},
        404: {"description": "User not found"},
    }
)
async def get_user(user_id: int):
    """Retrieve a user by their unique ID."""
    return User(id=user_id, username="johndoe", email="john@example.com", is_active=True)

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

**Key Takeaways:**
- Use descriptive titles, summaries, and descriptions for all endpoints
- Provide comprehensive examples for requests and responses
- Document all possible error responses
- Include authentication and versioning information
- Use tags to organize endpoints logically
- Add external documentation links where helpful
- Maintain a changelog for API changes
- Customize OpenAPI schema for branding and additional info

**Additional Resources:**
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [OpenAPI Specification](https://swagger.io/specification/)
- [Swagger UI](https://swagger.io/tools/swagger-ui/)
- [ReDoc](https://github.com/Redocly/redoc)