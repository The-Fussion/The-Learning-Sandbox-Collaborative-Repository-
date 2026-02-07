# FastAPI Request Handling

## 3.1 Request Body

### Pydantic Models for Request Body

FastAPI uses Pydantic models to define and validate request bodies with automatic type checking and serialization.

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class User(BaseModel):
    username: str
    email: str
    age: int

@app.post("/users")
async def create_user(user: User):
    return {"message": f"User {user.username} created"}
```

### Nested Models

Pydantic models can contain other models for complex data structures.

```python
class Address(BaseModel):
    street: str
    city: str
    country: str

class UserWithAddress(BaseModel):
    username: str
    email: str
    address: Address

@app.post("/users/complete")
async def create_user_with_address(user: UserWithAddress):
    return {"city": user.address.city}
```

### List and Set Fields

Use Python's type hints for collections.

```python
from typing import List, Set

class Product(BaseModel):
    name: str
    tags: List[str]
    categories: Set[str]

@app.post("/products")
async def create_product(product: Product):
    return product
```

### Dict Fields

Dictionaries can be used for flexible key-value data.

```python
from typing import Dict

class Settings(BaseModel):
    name: str
    config: Dict[str, str]

@app.post("/settings")
async def update_settings(settings: Settings):
    return settings.config
```

### Optional Fields and Default Values

```python
from typing import Optional

class Item(BaseModel):
    name: str
    description: Optional[str] = None
    price: float
    tax: float = 0.0
    in_stock: bool = True

@app.post("/items")
async def create_item(item: Item):
    return item
```

### Multiple Body Parameters

You can accept multiple Pydantic models in a single endpoint.

```python
class User(BaseModel):
    username: str

class Item(BaseModel):
    name: str

@app.post("/purchase")
async def purchase(user: User, item: Item):
    return {"buyer": user.username, "item": item.name}
```

### Singular Values in Body

Use `Body()` to embed singular values in the request body.

```python
from fastapi import Body

@app.post("/items/{item_id}")
async def update_item(
    item_id: int,
    item: Item,
    importance: int = Body(...)
):
    return {"item_id": item_id, "item": item, "importance": importance}
```

## 3.2 Headers & Cookies

### Reading Request Headers

Access headers using the `Header()` parameter.

```python
from fastapi import Header
from typing import Optional

@app.get("/items")
async def read_items(user_agent: Optional[str] = Header(None)):
    return {"User-Agent": user_agent}
```

### Custom Header Parameters

Headers are automatically converted from HTTP format (kebab-case) to Python format (snake_case).

```python
@app.get("/auth")
async def check_auth(x_token: Optional[str] = Header(None)):
    return {"token": x_token}
```

### Header Validation

Add validation constraints to headers.

```python
from fastapi import Header, HTTPException

@app.get("/secure")
async def secure_endpoint(
    authorization: str = Header(..., min_length=10)
):
    if not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Invalid token")
    return {"status": "authorized"}
```

### Reading Cookies

Use the `Cookie()` parameter to access cookies.

```python
from fastapi import Cookie

@app.get("/profile")
async def get_profile(session_id: Optional[str] = Cookie(None)):
    return {"session_id": session_id}
```

### Setting Response Cookies

Use the `Response` object to set cookies.

```python
from fastapi import Response

@app.post("/login")
async def login(response: Response):
    response.set_cookie(key="session_id", value="abc123")
    return {"message": "Logged in"}
```

### Secure Cookies

Configure cookies with security flags for production use.

```python
@app.post("/secure-login")
async def secure_login(response: Response):
    response.set_cookie(
        key="session_id",
        value="abc123",
        httponly=True,
        secure=True,
        samesite="lax",
        max_age=3600
    )
    return {"message": "Secure login successful"}
```

## 3.3 Form Data

### Form Fields

Handle form data using `Form()` for `application/x-www-form-urlencoded` submissions.

```python
from fastapi import Form

@app.post("/login")
async def login(username: str = Form(...), password: str = Form(...)):
    return {"username": username}
```

### Form Data Validation

Apply validation rules to form fields.

```python
from pydantic import EmailStr

@app.post("/register")
async def register(
    email: EmailStr = Form(...),
    password: str = Form(..., min_length=8),
    age: int = Form(..., ge=18)
):
    return {"email": email}
```

### Form Data with Pydantic

Note: Pydantic models don't directly work with `Form()`, so you need to define each field separately.

```python
@app.post("/user-form")
async def create_user_form(
    username: str = Form(...),
    email: str = Form(...),
    age: int = Form(...)
):
    # Manually create the model
    user = User(username=username, email=email, age=age)
    return user
```

### Combining Form and File Uploads

Mix form fields with file uploads in multipart requests.

```python
from fastapi import File, UploadFile

@app.post("/upload-profile")
async def upload_profile(
    username: str = Form(...),
    bio: str = Form(...),
    profile_pic: UploadFile = File(...)
):
    return {
        "username": username,
        "filename": profile_pic.filename
    }
```

## 3.4 File Uploads

### Single File Upload

Use `UploadFile` for efficient file handling with spooled upload.

```python
from fastapi import File, UploadFile

@app.post("/upload")
async def upload_file(file: UploadFile = File(...)):
    return {
        "filename": file.filename,
        "content_type": file.content_type
    }
```

### Multiple File Uploads

Accept multiple files using a `List` type hint.

```python
from typing import List

@app.post("/upload-multiple")
async def upload_multiple(files: List[UploadFile] = File(...)):
    return {
        "filenames": [file.filename for file in files]
    }
```

### File Validation

Validate file size and type before processing.

```python
from fastapi import HTTPException

@app.post("/upload-image")
async def upload_image(file: UploadFile = File(...)):
    # Check file type
    if file.content_type not in ["image/jpeg", "image/png"]:
        raise HTTPException(400, "Only JPEG and PNG allowed")
    
    # Check file size (read in chunks)
    max_size = 5 * 1024 * 1024  # 5MB
    size = 0
    chunk_size = 1024
    
    while chunk := await file.read(chunk_size):
        size += len(chunk)
        if size > max_size:
            raise HTTPException(400, "File too large")
    
    await file.seek(0)  # Reset file pointer
    return {"filename": file.filename}
```

### UploadFile vs File

`UploadFile` is preferred over `bytes = File()` for larger files as it uses spooled memory.

```python
# Less efficient for large files
@app.post("/upload-bytes")
async def upload_bytes(file: bytes = File(...)):
    return {"size": len(file)}

# More efficient - uses spooled upload
@app.post("/upload-file")
async def upload_file(file: UploadFile = File(...)):
    contents = await file.read()
    return {"size": len(contents)}
```

### Saving Uploaded Files

Save files to disk asynchronously.

```python
import aiofiles

@app.post("/save-file")
async def save_file(file: UploadFile = File(...)):
    file_location = f"uploads/{file.filename}"
    
    async with aiofiles.open(file_location, "wb") as f:
        content = await file.read()
        await f.write(content)
    
    return {"filename": file.filename, "location": file_location}
```

### File Metadata

Access detailed file information.

```python
@app.post("/file-info")
async def file_info(file: UploadFile = File(...)):
    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": file.size if hasattr(file, 'size') else "unknown"
    }
```

### Image Processing with Pillow

Process uploaded images using the Pillow library.

```python
from PIL import Image
import io

@app.post("/process-image")
async def process_image(file: UploadFile = File(...)):
    contents = await file.read()
    image = Image.open(io.BytesIO(contents))
    
    # Resize image
    image.thumbnail((800, 800))
    
    # Save processed image
    output = io.BytesIO()
    image.save(output, format="JPEG")
    
    return {
        "original_size": image.size,
        "format": image.format,
        "mode": image.mode
    }
```

## 3.5 Request Object

### Accessing Raw Request

Access the underlying Starlette `Request` object for low-level operations.

```python
from fastapi import Request

@app.get("/request-info")
async def request_info(request: Request):
    return {
        "method": request.method,
        "url": str(request.url),
        "headers": dict(request.headers)
    }
```

### Request Attributes

The `Request` object provides access to various request properties.

```python
@app.get("/details")
async def get_details(request: Request):
    return {
        "method": request.method,
        "url": str(request.url),
        "base_url": str(request.base_url),
        "headers": dict(request.headers),
        "query_params": dict(request.query_params),
        "path_params": request.path_params,
        "cookies": request.cookies
    }
```

### Request Body (Raw Bytes)

Read the raw request body as bytes.

```python
@app.post("/raw-body")
async def read_raw_body(request: Request):
    body = await request.body()
    return {
        "body_length": len(body),
        "content_type": request.headers.get("content-type")
    }
```

### Request Client Information

Access client connection details.

```python
@app.get("/client-info")
async def client_info(request: Request):
    return {
        "client_host": request.client.host if request.client else None,
        "client_port": request.client.port if request.client else None,
        "remote_addr": request.client.host if request.client else "unknown"
    }
```

### Request URL Components

Parse and access different parts of the request URL.

```python
@app.get("/url-components")
async def url_components(request: Request):
    return {
        "scheme": request.url.scheme,
        "netloc": request.url.netloc,
        "path": request.url.path,
        "query": request.url.query,
        "fragment": request.url.fragment,
        "hostname": request.url.hostname,
        "port": request.url.port,
        "full_url": str(request.url)
    }
```

---

## Practice Exercise

Create an endpoint that accepts a profile update with:
- Form data (name, bio)
- A profile picture upload
- Validates the image type and size
- Processes it with Pillow to create a thumbnail
- Returns both the request details and processed image metadata

### Solution

```python
from fastapi import FastAPI, Form, File, UploadFile, HTTPException, Request
from PIL import Image
import io

app = FastAPI()

@app.post("/update-profile")
async def update_profile(
    request: Request,
    name: str = Form(..., min_length=2),
    bio: str = Form(..., max_length=500),
    profile_pic: UploadFile = File(...)
):
    # Validate image type
    allowed_types = ["image/jpeg", "image/png", "image/webp"]
    if profile_pic.content_type not in allowed_types:
        raise HTTPException(400, "Only JPEG, PNG, and WebP images are allowed")
    
    # Read and validate file size
    contents = await profile_pic.read()
    max_size = 5 * 1024 * 1024  # 5MB
    if len(contents) > max_size:
        raise HTTPException(400, "File size exceeds 5MB limit")
    
    # Process image with Pillow
    try:
        image = Image.open(io.BytesIO(contents))
        original_size = image.size
        
        # Create thumbnail
        image.thumbnail((200, 200))
        thumbnail_size = image.size
        
        # Save thumbnail
        output = io.BytesIO()
        image.save(output, format=image.format or "JPEG")
        thumbnail_bytes = output.getvalue()
        
    except Exception as e:
        raise HTTPException(400, f"Error processing image: {str(e)}")
    
    # Return comprehensive response
    return {
        "profile_data": {
            "name": name,
            "bio": bio
        },
        "image_metadata": {
            "filename": profile_pic.filename,
            "content_type": profile_pic.content_type,
            "original_size": original_size,
            "thumbnail_size": thumbnail_size,
            "original_bytes": len(contents),
            "thumbnail_bytes": len(thumbnail_bytes),
            "format": image.format
        },
        "request_info": {
            "client": request.client.host if request.client else None,
            "user_agent": request.headers.get("user-agent")
        }
    }
```

---

**Additional Resources:**
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Pydantic Documentation](https://docs.pydantic.dev/)
- [Pillow Documentation](https://pillow.readthedocs.io/)