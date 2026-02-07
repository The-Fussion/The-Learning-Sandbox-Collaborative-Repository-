# FastAPI Response Handling

## 4.1 Response Models

### response_model Parameter

The `response_model` parameter defines the schema for your API response, enabling automatic validation and documentation.

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class UserIn(BaseModel):
    username: str
    password: str
    email: str

class UserOut(BaseModel):
    username: str
    email: str

@app.post("/users", response_model=UserOut)
async def create_user(user: UserIn):
    # Password won't be included in response
    return user
```

### Response Model Validation

FastAPI validates the response data against the response model, ensuring type safety.

```python
class Item(BaseModel):
    name: str
    price: float
    is_available: bool

@app.get("/items/{item_id}", response_model=Item)
async def get_item(item_id: int):
    # Response must match Item model
    return {
        "name": "Widget",
        "price": 29.99,
        "is_available": True
    }
```

### response_model_exclude_unset

Exclude fields that were not explicitly set, returning only provided values.

```python
class ItemResponse(BaseModel):
    name: str
    description: str | None = None
    price: float
    tax: float | None = None

@app.get("/items/{item_id}", response_model=ItemResponse, response_model_exclude_unset=True)
async def get_item(item_id: int):
    # Only returns name and price if tax/description not set
    return {"name": "Widget", "price": 29.99}
```

### response_model_exclude_none

Exclude all fields with `None` values from the response.

```python
@app.get("/items/{item_id}", response_model=ItemResponse, response_model_exclude_none=True)
async def get_item(item_id: int):
    return {
        "name": "Widget",
        "description": None,  # This will be excluded
        "price": 29.99,
        "tax": None  # This will be excluded
    }
```

### response_model_include and exclude

Specify exactly which fields to include or exclude from the response.

```python
class UserComplete(BaseModel):
    username: str
    email: str
    full_name: str
    is_active: bool
    created_at: str

# Include only specific fields
@app.get("/users/{user_id}/basic", response_model=UserComplete, response_model_include={"username", "email"})
async def get_user_basic(user_id: int):
    return {
        "username": "johndoe",
        "email": "john@example.com",
        "full_name": "John Doe",
        "is_active": True,
        "created_at": "2024-01-01"
    }

# Exclude specific fields
@app.get("/users/{user_id}/public", response_model=UserComplete, response_model_exclude={"created_at", "is_active"})
async def get_user_public(user_id: int):
    return {
        "username": "johndoe",
        "email": "john@example.com",
        "full_name": "John Doe",
        "is_active": True,
        "created_at": "2024-01-01"
    }
```

### Union Response Models

Return different model types based on conditions using Union.

```python
from typing import Union

class ErrorResponse(BaseModel):
    error: str
    code: int

class SuccessResponse(BaseModel):
    message: str
    data: dict

@app.get("/data/{item_id}", response_model=Union[SuccessResponse, ErrorResponse])
async def get_data(item_id: int):
    if item_id < 0:
        return ErrorResponse(error="Invalid ID", code=400)
    return SuccessResponse(message="Success", data={"item_id": item_id})
```

### List Response Models

Return lists of model instances.

```python
from typing import List

class Product(BaseModel):
    id: int
    name: str
    price: float

@app.get("/products", response_model=List[Product])
async def get_products():
    return [
        {"id": 1, "name": "Widget", "price": 29.99},
        {"id": 2, "name": "Gadget", "price": 49.99},
        {"id": 3, "name": "Tool", "price": 19.99}
    ]
```

### Response Model Encoding

Customize how models are encoded in responses.

```python
from datetime import datetime
from pydantic import BaseModel, Field

class Event(BaseModel):
    title: str
    created_at: datetime = Field(default_factory=datetime.now)
    
    class Config:
        json_encoders = {
            datetime: lambda v: v.isoformat()
        }

@app.get("/events/{event_id}", response_model=Event)
async def get_event(event_id: int):
    return Event(title="Conference")
```

## 4.2 Status Codes

### Standard HTTP Status Codes

Use standard HTTP status codes to indicate response status.

```python
from fastapi import status

@app.post("/users", status_code=status.HTTP_201_CREATED)
async def create_user(user: UserIn):
    return {"message": "User created"}

@app.delete("/users/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(user_id: int):
    return None

@app.get("/items", status_code=status.HTTP_200_OK)
async def get_items():
    return {"items": []}
```

### Custom Status Codes

Set custom status codes when needed.

```python
@app.post("/custom", status_code=299)
async def custom_endpoint():
    return {"message": "Custom status code"}
```

### status_code Parameter

The `status_code` parameter sets the default successful response code.

```python
@app.post("/items", status_code=201)
async def create_item(item: Item):
    return item

@app.put("/items/{item_id}", status_code=200)
async def update_item(item_id: int, item: Item):
    return item
```

### Multiple Status Code Responses

Document multiple possible status codes using `responses` parameter.

```python
from fastapi import Response

@app.get(
    "/items/{item_id}",
    status_code=200,
    responses={
        200: {"description": "Item found"},
        404: {"description": "Item not found"},
        500: {"description": "Internal server error"}
    }
)
async def get_item(item_id: int, response: Response):
    if item_id == 0:
        response.status_code = 404
        return {"error": "Not found"}
    return {"id": item_id, "name": "Item"}
```

### Status Code Enums

Use status code enums for better readability and IDE support.

```python
from fastapi import status

@app.post("/register", status_code=status.HTTP_201_CREATED)
async def register():
    return {"message": "Registered"}

@app.delete("/logout", status_code=status.HTTP_204_NO_CONTENT)
async def logout():
    return None

@app.get("/health", status_code=status.HTTP_200_OK)
async def health_check():
    return {"status": "healthy"}
```

## 4.3 Response Types

### JSONResponse (Default)

The default response type for FastAPI, returning JSON data.

```python
from fastapi.responses import JSONResponse

@app.get("/json")
async def get_json():
    return {"message": "This is JSON"}

# Explicit JSONResponse
@app.get("/explicit-json")
async def explicit_json():
    return JSONResponse(
        content={"message": "Explicit JSON"},
        status_code=200,
        headers={"X-Custom-Header": "Value"}
    )
```

### HTMLResponse

Return HTML content directly.

```python
from fastapi.responses import HTMLResponse

@app.get("/html", response_class=HTMLResponse)
async def get_html():
    html_content = """
    <!DOCTYPE html>
    <html>
        <head>
            <title>FastAPI</title>
        </head>
        <body>
            <h1>Hello from FastAPI!</h1>
        </body>
    </html>
    """
    return html_content

# With HTMLResponse directly
@app.get("/page")
async def get_page():
    return HTMLResponse(content="<h1>Hello World</h1>", status_code=200)
```

### PlainTextResponse

Return plain text responses.

```python
from fastapi.responses import PlainTextResponse

@app.get("/text", response_class=PlainTextResponse)
async def get_text():
    return "This is plain text"

@app.get("/file-content")
async def get_file_content():
    return PlainTextResponse("Line 1\nLine 2\nLine 3")
```

### FileResponse

Send files to the client.

```python
from fastapi.responses import FileResponse

@app.get("/download")
async def download_file():
    return FileResponse(
        path="files/document.pdf",
        filename="downloaded_document.pdf",
        media_type="application/pdf"
    )

@app.get("/image")
async def get_image():
    return FileResponse(
        path="images/photo.jpg",
        media_type="image/jpeg"
    )
```

### StreamingResponse

Stream large files or generated content.

```python
from fastapi.responses import StreamingResponse
import io

@app.get("/stream")
async def stream_data():
    def generate():
        for i in range(100):
            yield f"Line {i}\n"
    
    return StreamingResponse(
        generate(),
        media_type="text/plain"
    )

# Stream a file
@app.get("/stream-file")
async def stream_file():
    def iterfile():
        with open("large_file.txt", mode="rb") as file:
            yield from file
    
    return StreamingResponse(iterfile(), media_type="text/plain")

# Stream CSV
@app.get("/export-csv")
async def export_csv():
    def generate_csv():
        yield "Name,Age,Email\n"
        yield "John,30,john@example.com\n"
        yield "Jane,25,jane@example.com\n"
    
    return StreamingResponse(
        generate_csv(),
        media_type="text/csv",
        headers={"Content-Disposition": "attachment; filename=users.csv"}
    )
```

### RedirectResponse

Redirect to another URL.

```python
from fastapi.responses import RedirectResponse

@app.get("/redirect")
async def redirect():
    return RedirectResponse(url="/new-location")

@app.get("/old-page")
async def old_page():
    return RedirectResponse(
        url="/new-page",
        status_code=status.HTTP_301_MOVED_PERMANENTLY
    )

# Temporary redirect
@app.get("/temporary")
async def temporary_redirect():
    return RedirectResponse(
        url="/temp-location",
        status_code=status.HTTP_307_TEMPORARY_REDIRECT
    )
```

### Custom Response Classes

Create custom response classes for specific needs.

```python
from fastapi.responses import Response

class CustomXMLResponse(Response):
    media_type = "application/xml"

@app.get("/xml", response_class=CustomXMLResponse)
async def get_xml():
    xml_content = """<?xml version="1.0"?>
    <root>
        <message>Hello XML</message>
    </root>
    """
    return CustomXMLResponse(content=xml_content)
```

### Response Headers

Add custom headers to responses.

```python
from fastapi import Response

@app.get("/headers")
async def get_with_headers(response: Response):
    response.headers["X-Custom-Header"] = "Custom Value"
    response.headers["X-Process-Time"] = "0.002"
    return {"message": "Check the headers"}

# Using Response class directly
@app.get("/custom-headers")
async def custom_headers():
    return JSONResponse(
        content={"message": "Success"},
        headers={
            "X-Custom-Header": "Value",
            "Cache-Control": "no-cache"
        }
    )
```

### Response Cookies

Set cookies in responses.

```python
@app.post("/login")
async def login(response: Response):
    response.set_cookie(
        key="session_id",
        value="abc123xyz",
        max_age=3600,
        httponly=True,
        secure=True,
        samesite="lax"
    )
    return {"message": "Logged in successfully"}

@app.post("/logout")
async def logout(response: Response):
    response.delete_cookie(key="session_id")
    return {"message": "Logged out"}
```

## 4.4 Error Handling

### HTTPException

Raise HTTP exceptions to return error responses.

```python
from fastapi import HTTPException

@app.get("/items/{item_id}")
async def get_item(item_id: int):
    if item_id < 1:
        raise HTTPException(
            status_code=400,
            detail="Item ID must be positive"
        )
    if item_id > 1000:
        raise HTTPException(
            status_code=404,
            detail="Item not found"
        )
    return {"item_id": item_id}

# With custom headers
@app.get("/secure/{item_id}")
async def get_secure_item(item_id: int):
    if item_id == 0:
        raise HTTPException(
            status_code=401,
            detail="Not authenticated",
            headers={"WWW-Authenticate": "Bearer"}
        )
    return {"item_id": item_id}
```

### Custom Exception Handlers

Create custom handlers for specific exception types.

```python
from fastapi import Request
from fastapi.responses import JSONResponse

class ItemNotFoundException(Exception):
    def __init__(self, item_id: int):
        self.item_id = item_id

@app.exception_handler(ItemNotFoundException)
async def item_not_found_handler(request: Request, exc: ItemNotFoundException):
    return JSONResponse(
        status_code=404,
        content={
            "message": f"Item {exc.item_id} not found",
            "item_id": exc.item_id
        }
    )

@app.get("/items/{item_id}")
async def get_item(item_id: int):
    if item_id == 0:
        raise ItemNotFoundException(item_id=item_id)
    return {"item_id": item_id}
```

### Validation Error Handlers

Override the default validation error handler.

```python
from fastapi.exceptions import RequestValidationError
from fastapi.responses import PlainTextResponse

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    return PlainTextResponse(
        content=f"Validation error: {exc}",
        status_code=422
    )
```

### RequestValidationError

Handle Pydantic validation errors.

```python
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse

@app.exception_handler(RequestValidationError)
async def validation_error_handler(request: Request, exc: RequestValidationError):
    errors = []
    for error in exc.errors():
        errors.append({
            "field": " -> ".join(str(x) for x in error["loc"]),
            "message": error["msg"],
            "type": error["type"]
        })
    
    return JSONResponse(
        status_code=422,
        content={
            "detail": "Validation failed",
            "errors": errors
        }
    )
```

### Global Exception Handlers

Handle all unhandled exceptions globally.

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    return JSONResponse(
        status_code=500,
        content={
            "message": "Internal server error",
            "detail": str(exc)
        }
    )

# Handle specific exception types
@app.exception_handler(ValueError)
async def value_error_handler(request: Request, exc: ValueError):
    return JSONResponse(
        status_code=400,
        content={"message": "Invalid value", "detail": str(exc)}
    )

@app.exception_handler(KeyError)
async def key_error_handler(request: Request, exc: KeyError):
    return JSONResponse(
        status_code=404,
        content={"message": "Key not found", "key": str(exc)}
    )
```

### Error Response Models

Define Pydantic models for error responses.

```python
from pydantic import BaseModel
from typing import List, Optional

class ErrorDetail(BaseModel):
    field: str
    message: str
    type: str

class ErrorResponse(BaseModel):
    error: str
    status_code: int
    details: Optional[List[ErrorDetail]] = None

@app.exception_handler(RequestValidationError)
async def validation_handler(request: Request, exc: RequestValidationError):
    details = [
        ErrorDetail(
            field=" -> ".join(str(x) for x in error["loc"]),
            message=error["msg"],
            type=error["type"]
        )
        for error in exc.errors()
    ]
    
    error_response = ErrorResponse(
        error="Validation Error",
        status_code=422,
        details=details
    )
    
    return JSONResponse(
        status_code=422,
        content=error_response.dict()
    )
```

### Custom Error Responses

Create reusable error response functions.

```python
def create_error_response(
    status_code: int,
    message: str,
    details: Optional[dict] = None
):
    content = {
        "error": True,
        "message": message,
        "status_code": status_code
    }
    if details:
        content["details"] = details
    
    return JSONResponse(
        status_code=status_code,
        content=content
    )

@app.get("/items/{item_id}")
async def get_item(item_id: int):
    if item_id < 0:
        return create_error_response(
            status_code=400,
            message="Invalid item ID",
            details={"item_id": item_id, "constraint": "must be positive"}
        )
    
    if item_id > 1000:
        return create_error_response(
            status_code=404,
            message="Item not found"
        )
    
    return {"item_id": item_id, "name": "Sample Item"}
```

---

## Practice Exercise

Create a complete user management system with proper response handling:

1. Use response models to exclude sensitive data (passwords)
2. Return appropriate status codes (201 for creation, 204 for deletion)
3. Implement custom exception handling for "UserNotFound" and "DuplicateUser" errors
4. Add validation error handling with detailed error messages
5. Return different response types (JSON for API, HTML for web pages)

### Solution

```python
from fastapi import FastAPI, HTTPException, Request, status
from fastapi.responses import JSONResponse, HTMLResponse
from fastapi.exceptions import RequestValidationError
from pydantic import BaseModel, EmailStr
from typing import List, Optional

app = FastAPI()

# Models
class UserCreate(BaseModel):
    username: str
    email: EmailStr
    password: str

class UserResponse(BaseModel):
    id: int
    username: str
    email: str
    is_active: bool

class ErrorDetail(BaseModel):
    field: str
    message: str

class ErrorResponse(BaseModel):
    error: str
    details: Optional[List[ErrorDetail]] = None

# Custom Exceptions
class UserNotFoundException(Exception):
    def __init__(self, user_id: int):
        self.user_id = user_id

class DuplicateUserException(Exception):
    def __init__(self, username: str):
        self.username = username

# Exception Handlers
@app.exception_handler(UserNotFoundException)
async def user_not_found_handler(request: Request, exc: UserNotFoundException):
    return JSONResponse(
        status_code=404,
        content={
            "error": "User not found",
            "user_id": exc.user_id
        }
    )

@app.exception_handler(DuplicateUserException)
async def duplicate_user_handler(request: Request, exc: DuplicateUserException):
    return JSONResponse(
        status_code=409,
        content={
            "error": "User already exists",
            "username": exc.username
        }
    )

@app.exception_handler(RequestValidationError)
async def validation_handler(request: Request, exc: RequestValidationError):
    details = [
        ErrorDetail(
            field=" -> ".join(str(x) for x in error["loc"]),
            message=error["msg"]
        )
        for error in exc.errors()
    ]
    
    return JSONResponse(
        status_code=422,
        content=ErrorResponse(
            error="Validation failed",
            details=details
        ).dict()
    )

# In-memory database
users_db = {}
user_id_counter = 1

# Endpoints
@app.post(
    "/users",
    response_model=UserResponse,
    status_code=status.HTTP_201_CREATED,
    responses={
        201: {"description": "User created successfully"},
        409: {"description": "User already exists"},
        422: {"description": "Validation error"}
    }
)
async def create_user(user: UserCreate):
    global user_id_counter
    
    # Check for duplicate
    if any(u["username"] == user.username for u in users_db.values()):
        raise DuplicateUserException(username=user.username)
    
    # Create user
    user_id = user_id_counter
    users_db[user_id] = {
        "id": user_id,
        "username": user.username,
        "email": user.email,
        "password": user.password,  # In production, hash this!
        "is_active": True
    }
    user_id_counter += 1
    
    return users_db[user_id]

@app.get(
    "/users",
    response_model=List[UserResponse],
    response_model_exclude_none=True
)
async def get_users():
    return list(users_db.values())

@app.get(
    "/users/{user_id}",
    response_model=UserResponse,
    responses={
        200: {"description": "User found"},
        404: {"description": "User not found"}
    }
)
async def get_user(user_id: int):
    if user_id not in users_db:
        raise UserNotFoundException(user_id=user_id)
    return users_db[user_id]

@app.delete(
    "/users/{user_id}",
    status_code=status.HTTP_204_NO_CONTENT,
    responses={
        204: {"description": "User deleted successfully"},
        404: {"description": "User not found"}
    }
)
async def delete_user(user_id: int):
    if user_id not in users_db:
        raise UserNotFoundException(user_id=user_id)
    del users_db[user_id]
    return None

@app.get("/users-page", response_class=HTMLResponse)
async def get_users_page():
    users_html = "".join(
        f"<li>{user['username']} - {user['email']}</li>"
        for user in users_db.values()
    )
    
    html_content = f"""
    <!DOCTYPE html>
    <html>
        <head>
            <title>Users</title>
        </head>
        <body>
            <h1>Users List</h1>
            <ul>
                {users_html or "<li>No users found</li>"}
            </ul>
        </body>
    </html>
    """
    return html_content
```

---

**Key Takeaways:**
- Use `response_model` to control what data is returned
- Set appropriate status codes for different operations
- Implement custom exception handlers for better error messages
- Choose the right response type for your use case
- Always validate and handle errors gracefully

**Additional Resources:**
- [FastAPI Response Documentation](https://fastapi.tiangolo.com/advanced/response-directly/)
- [HTTP Status Codes Reference](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
- [Pydantic Models](https://docs.pydantic.dev/)