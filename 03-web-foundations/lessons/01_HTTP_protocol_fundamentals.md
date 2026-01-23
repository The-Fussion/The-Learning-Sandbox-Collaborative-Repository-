# HTTP Protocol Fundamentals with Python

## Table of Contents
1. [Introduction to HTTP](#introduction-to-http)
2. [Request/Response Cycle](#requestresponse-cycle)
3. [HTTP Methods](#http-methods)
4. [HTTP Status Codes](#http-status-codes)
5. [HTTP Headers](#http-headers)
6. [Cookies](#cookies)
7. [Python Libraries for HTTP](#python-libraries-for-http)
8. [Practical Examples](#practical-examples)

---

## Introduction to HTTP

**HTTP (HyperText Transfer Protocol)** is the foundation of data communication on the World Wide Web. It's an application-layer protocol that defines how messages are formatted and transmitted between clients (browsers, applications) and servers.

### Key Characteristics:
- **Stateless**: Each request is independent; the server doesn't retain information between requests
- **Client-Server Model**: Clients initiate requests, servers respond
- **Text-Based Protocol**: Messages are human-readable (though data can be binary)
- **Request-Response**: Communication happens in pairs

---

## Request/Response Cycle

The HTTP request/response cycle is the fundamental pattern of web communication:

```
CLIENT                          SERVER
  |                               |
  |  1. HTTP Request              |
  |------------------------------>|
  |                               |
  |                          2. Process
  |                               |
  |  3. HTTP Response             |
  |<------------------------------|
  |                               |
```

### HTTP Request Structure

```
[METHOD] [PATH] HTTP/[VERSION]
[HEADERS]

[BODY]
```

**Example:**
```
GET /api/users HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Accept: application/json

```

### HTTP Response Structure

```
HTTP/[VERSION] [STATUS_CODE] [STATUS_MESSAGE]
[HEADERS]

[BODY]
```

**Example:**
```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 85

{"id": 1, "name": "John Doe", "email": "john@example.com"}
```

---

## HTTP Methods

HTTP methods (also called verbs) indicate the desired action to be performed on a resource.

### Common HTTP Methods

| Method | Purpose | Safe | Idempotent | Has Body |
|--------|---------|------|------------|----------|
| GET | Retrieve data | Yes | Yes | No |
| POST | Create new resource | No | No | Yes |
| PUT | Update/replace resource | No | Yes | Yes |
| PATCH | Partial update | No | No | Yes |
| DELETE | Remove resource | No | Yes | Optional |
| HEAD | Get headers only | Yes | Yes | No |
| OPTIONS | Get allowed methods | Yes | Yes | No |

**Safe**: Doesn't modify server state  
**Idempotent**: Multiple identical requests have the same effect as one

### Python Examples

```python
import requests

base_url = "https://jsonplaceholder.typicode.com"

# GET - Retrieve data
response = requests.get(f"{base_url}/posts/1")
print(response.json())

# POST - Create new resource
new_post = {
    "title": "My New Post",
    "body": "This is the content",
    "userId": 1
}
response = requests.post(f"{base_url}/posts", json=new_post)
print(response.status_code)  # 201 Created
print(response.json())

# PUT - Replace entire resource
updated_post = {
    "id": 1,
    "title": "Updated Title",
    "body": "Updated content",
    "userId": 1
}
response = requests.put(f"{base_url}/posts/1", json=updated_post)
print(response.json())

# PATCH - Partial update
partial_update = {"title": "Only Title Changed"}
response = requests.patch(f"{base_url}/posts/1", json=partial_update)
print(response.json())

# DELETE - Remove resource
response = requests.delete(f"{base_url}/posts/1")
print(response.status_code)  # 200 OK

# HEAD - Get headers without body
response = requests.head(f"{base_url}/posts/1")
print(response.headers)
print(len(response.content))  # 0 - no body

# OPTIONS - Check allowed methods
response = requests.options(f"{base_url}/posts")
print(response.headers.get('Allow'))
```

---

## HTTP Status Codes

Status codes indicate the result of the HTTP request. They're grouped into five categories:

### 1xx: Informational
- **100 Continue**: Server received request headers, client should send body
- **101 Switching Protocols**: Server is switching protocols (e.g., to WebSocket)

### 2xx: Success
- **200 OK**: Request succeeded
- **201 Created**: New resource created successfully
- **202 Accepted**: Request accepted but processing not complete
- **204 No Content**: Success but no content to return
- **206 Partial Content**: Partial content delivered (range request)

### 3xx: Redirection
- **301 Moved Permanently**: Resource permanently moved to new URL
- **302 Found**: Temporary redirect
- **304 Not Modified**: Cached version is still valid
- **307 Temporary Redirect**: Like 302 but method must not change
- **308 Permanent Redirect**: Like 301 but method must not change

### 4xx: Client Errors
- **400 Bad Request**: Invalid request syntax
- **401 Unauthorized**: Authentication required
- **403 Forbidden**: Server refuses to fulfill request
- **404 Not Found**: Resource doesn't exist
- **405 Method Not Allowed**: HTTP method not supported for resource
- **408 Request Timeout**: Server timed out waiting for request
- **409 Conflict**: Request conflicts with server state
- **429 Too Many Requests**: Rate limit exceeded

### 5xx: Server Errors
- **500 Internal Server Error**: Generic server error
- **501 Not Implemented**: Server doesn't support functionality
- **502 Bad Gateway**: Invalid response from upstream server
- **503 Service Unavailable**: Server temporarily unavailable
- **504 Gateway Timeout**: Upstream server timed out

### Python Examples

```python
import requests

# Handling different status codes
def make_request(url):
    try:
        response = requests.get(url, timeout=5)
        
        # Check status code
        if response.status_code == 200:
            print("✓ Success!")
            return response.json()
        
        elif response.status_code == 404:
            print("✗ Resource not found")
        
        elif response.status_code == 401:
            print("✗ Authentication required")
        
        elif response.status_code == 403:
            print("✗ Access forbidden")
        
        elif response.status_code >= 500:
            print("✗ Server error")
        
        # Raise exception for bad status codes
        response.raise_for_status()
        
    except requests.exceptions.HTTPError as e:
        print(f"HTTP Error: {e}")
    except requests.exceptions.ConnectionError:
        print("Connection Error")
    except requests.exceptions.Timeout:
        print("Request timed out")
    except requests.exceptions.RequestException as e:
        print(f"Error: {e}")

# Test with various URLs
make_request("https://jsonplaceholder.typicode.com/posts/1")
make_request("https://jsonplaceholder.typicode.com/posts/99999")
```

---

## HTTP Headers

Headers provide metadata about the request or response. They consist of name-value pairs.

### Common Request Headers

| Header | Purpose | Example |
|--------|---------|---------|
| `Host` | Target server domain | `Host: example.com` |
| `User-Agent` | Client application info | `User-Agent: Mozilla/5.0` |
| `Accept` | Content types client can handle | `Accept: application/json` |
| `Content-Type` | Type of request body | `Content-Type: application/json` |
| `Authorization` | Authentication credentials | `Authorization: Bearer token123` |
| `Cookie` | Cookies sent to server | `Cookie: sessionId=abc123` |
| `Referer` | Previous page URL | `Referer: https://google.com` |
| `Accept-Encoding` | Supported compressions | `Accept-Encoding: gzip, deflate` |

### Common Response Headers

| Header | Purpose | Example |
|--------|---------|---------|
| `Content-Type` | Type of response body | `Content-Type: text/html` |
| `Content-Length` | Size of response body | `Content-Length: 1234` |
| `Set-Cookie` | Set cookies on client | `Set-Cookie: id=a3fWa; Expires=...` |
| `Cache-Control` | Caching directives | `Cache-Control: max-age=3600` |
| `Location` | Redirect URL | `Location: /new-page` |
| `Server` | Server software info | `Server: nginx/1.18.0` |
| `Access-Control-Allow-Origin` | CORS policy | `Access-Control-Allow-Origin: *` |

### Python Examples

```python
import requests
import json

url = "https://httpbin.org/anything"

# Setting custom headers
headers = {
    "User-Agent": "MyPythonApp/1.0",
    "Accept": "application/json",
    "Authorization": "Bearer my-secret-token",
    "X-Custom-Header": "CustomValue"
}

response = requests.get(url, headers=headers)
print("Request headers sent:")
print(json.dumps(response.request.headers.__dict__['_store'], indent=2))

# Reading response headers
print("\nResponse headers received:")
for key, value in response.headers.items():
    print(f"{key}: {value}")

# Accessing specific headers
print(f"\nContent-Type: {response.headers.get('Content-Type')}")
print(f"Server: {response.headers.get('Server')}")

# Headers are case-insensitive
print(f"content-type: {response.headers.get('content-type')}")

# Working with JSON and Content-Type
json_data = {"name": "Alice", "age": 30}
headers = {"Content-Type": "application/json"}

response = requests.post(
    "https://httpbin.org/post",
    data=json.dumps(json_data),
    headers=headers
)
print(response.json())

# requests.post with json parameter automatically sets Content-Type
response = requests.post(
    "https://httpbin.org/post",
    json=json_data  # Automatically sets Content-Type: application/json
)
print(response.json())
```

---

## Cookies

Cookies are small pieces of data stored on the client side and sent with every request to the same domain. They're used for session management, personalization, and tracking.

### Cookie Attributes

- **Name=Value**: The cookie data
- **Domain**: Which domain can access the cookie
- **Path**: Which paths can access the cookie
- **Expires/Max-Age**: When the cookie expires
- **Secure**: Only send over HTTPS
- **HttpOnly**: Not accessible via JavaScript
- **SameSite**: Cross-site request restrictions

### Python Examples

```python
import requests

# Session object automatically handles cookies
session = requests.Session()

# Login and receive cookies
login_url = "https://httpbin.org/cookies/set/username/john"
response = session.get(login_url)

# Cookies are automatically stored in session
print("Cookies in session:")
print(session.cookies.get_dict())

# Subsequent requests automatically include cookies
response = session.get("https://httpbin.org/cookies")
print(response.json())

# Manually setting cookies
cookies = {
    "session_id": "abc123",
    "preferences": "dark_mode"
}
response = requests.get("https://httpbin.org/cookies", cookies=cookies)
print(response.json())

# Working with cookie jar
from http.cookiejar import CookieJar

jar = CookieJar()
response = requests.get("https://httpbin.org/cookies/set/token/xyz789", cookies=jar)

# Access individual cookies
for cookie in jar:
    print(f"Name: {cookie.name}")
    print(f"Value: {cookie.value}")
    print(f"Domain: {cookie.domain}")
    print(f"Path: {cookie.path}")

# Creating a cookie object
from requests.cookies import create_cookie

cookie = create_cookie(
    name="my_cookie",
    value="my_value",
    domain="httpbin.org",
    path="/",
)
jar.set_cookie(cookie)

# Clear cookies
session.cookies.clear()
```

---

## Python Libraries for HTTP

### 1. requests (Recommended for most use cases)

The most popular and user-friendly HTTP library.

```python
import requests

# Simple GET request
response = requests.get("https://api.github.com")

# POST with JSON data
data = {"key": "value"}
response = requests.post("https://httpbin.org/post", json=data)

# Custom headers, timeout, authentication
response = requests.get(
    "https://api.github.com/user",
    headers={"Authorization": "token YOUR_TOKEN"},
    timeout=10,
    auth=("username", "password")
)
```

### 2. urllib (Built-in, no installation needed)

Standard library module for URL handling.

```python
import urllib.request
import urllib.parse
import json

# GET request
with urllib.request.urlopen("https://api.github.com") as response:
    data = json.loads(response.read())
    print(data)

# POST request
url = "https://httpbin.org/post"
data = urllib.parse.urlencode({"key": "value"}).encode()
req = urllib.request.Request(url, data=data, method="POST")

with urllib.request.urlopen(req) as response:
    print(response.read().decode())

# With headers
headers = {"User-Agent": "Python/urllib"}
req = urllib.request.Request(url, headers=headers)
with urllib.request.urlopen(req) as response:
    print(response.status)
```

### 3. httpx (Modern async alternative)

Supports both synchronous and asynchronous requests.

```python
import httpx

# Synchronous
response = httpx.get("https://api.github.com")
print(response.json())

# Asynchronous
import asyncio

async def fetch_data():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.github.com")
        return response.json()

data = asyncio.run(fetch_data())
```

---

## Practical Examples

### Example 1: Building a Simple API Client

```python
import requests
from typing import Dict, List, Optional

class APIClient:
    def __init__(self, base_url: str, api_key: Optional[str] = None):
        self.base_url = base_url.rstrip('/')
        self.session = requests.Session()
        
        if api_key:
            self.session.headers.update({
                "Authorization": f"Bearer {api_key}"
            })
    
    def get(self, endpoint: str, params: Optional[Dict] = None) -> Dict:
        """Make a GET request"""
        url = f"{self.base_url}/{endpoint.lstrip('/')}"
        response = self.session.get(url, params=params)
        response.raise_for_status()
        return response.json()
    
    def post(self, endpoint: str, data: Dict) -> Dict:
        """Make a POST request"""
        url = f"{self.base_url}/{endpoint.lstrip('/')}"
        response = self.session.post(url, json=data)
        response.raise_for_status()
        return response.json()
    
    def put(self, endpoint: str, data: Dict) -> Dict:
        """Make a PUT request"""
        url = f"{self.base_url}/{endpoint.lstrip('/')}"
        response = self.session.put(url, json=data)
        response.raise_for_status()
        return response.json()
    
    def delete(self, endpoint: str) -> bool:
        """Make a DELETE request"""
        url = f"{self.base_url}/{endpoint.lstrip('/')}"
        response = self.session.delete(url)
        response.raise_for_status()
        return response.status_code == 204 or response.status_code == 200

# Usage
client = APIClient("https://jsonplaceholder.typicode.com")

# Get all posts
posts = client.get("/posts")
print(f"Total posts: {len(posts)}")

# Get specific post
post = client.get("/posts/1")
print(f"Post title: {post['title']}")

# Create new post
new_post = client.post("/posts", {
    "title": "My Post",
    "body": "Content here",
    "userId": 1
})
print(f"Created post with ID: {new_post['id']}")

# Update post
updated = client.put("/posts/1", {
    "id": 1,
    "title": "Updated Title",
    "body": "Updated body",
    "userId": 1
})

# Delete post
deleted = client.delete("/posts/1")
print(f"Deleted: {deleted}")
```

### Example 2: Handling Authentication

```python
import requests
from requests.auth import HTTPBasicAuth

# Basic Authentication
response = requests.get(
    "https://httpbin.org/basic-auth/user/pass",
    auth=HTTPBasicAuth("user", "pass")
)
print(response.json())

# Simplified basic auth
response = requests.get(
    "https://httpbin.org/basic-auth/user/pass",
    auth=("user", "pass")
)

# Bearer Token Authentication
headers = {
    "Authorization": "Bearer YOUR_ACCESS_TOKEN"
}
response = requests.get("https://api.example.com/data", headers=headers)

# API Key Authentication (in query parameter)
params = {"api_key": "YOUR_API_KEY"}
response = requests.get("https://api.example.com/data", params=params)

# API Key in Header
headers = {"X-API-Key": "YOUR_API_KEY"}
response = requests.get("https://api.example.com/data", headers=headers)
```

### Example 3: File Upload/Download

```python
import requests

# Upload a file
files = {"file": open("document.pdf", "rb")}
response = requests.post("https://httpbin.org/post", files=files)
print(response.json())

# Upload with additional form data
files = {"file": open("image.jpg", "rb")}
data = {"description": "Profile picture"}
response = requests.post(
    "https://httpbin.org/post",
    files=files,
    data=data
)

# Download a file
url = "https://www.python.org/static/img/python-logo.png"
response = requests.get(url)

with open("python-logo.png", "wb") as f:
    f.write(response.content)

# Download large file in chunks
url = "https://example.com/large-file.zip"
with requests.get(url, stream=True) as response:
    response.raise_for_status()
    with open("large-file.zip", "wb") as f:
        for chunk in response.iter_content(chunk_size=8192):
            f.write(chunk)
```

### Example 4: Error Handling and Retries

```python
import requests
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_session_with_retries():
    """Create a session with automatic retries"""
    session = requests.Session()
    
    retry_strategy = Retry(
        total=3,  # Total number of retries
        backoff_factor=1,  # Wait 1, 2, 4 seconds between retries
        status_forcelist=[429, 500, 502, 503, 504],
        method_whitelist=["HEAD", "GET", "OPTIONS", "POST"]
    )
    
    adapter = HTTPAdapter(max_retries=retry_strategy)
    session.mount("http://", adapter)
    session.mount("https://", adapter)
    
    return session

# Usage
session = create_session_with_retries()

try:
    response = session.get("https://api.example.com/data", timeout=5)
    response.raise_for_status()
    data = response.json()
except requests.exceptions.HTTPError as e:
    print(f"HTTP Error: {e}")
except requests.exceptions.ConnectionError:
    print("Failed to connect to the server")
except requests.exceptions.Timeout:
    print("Request timed out")
except requests.exceptions.RequestException as e:
    print(f"An error occurred: {e}")
```

### Example 5: Rate Limiting

```python
import requests
import time
from datetime import datetime, timedelta

class RateLimitedClient:
    def __init__(self, requests_per_minute: int = 60):
        self.requests_per_minute = requests_per_minute
        self.request_times = []
    
    def _wait_if_needed(self):
        """Wait if rate limit would be exceeded"""
        now = datetime.now()
        
        # Remove requests older than 1 minute
        self.request_times = [
            t for t in self.request_times 
            if now - t < timedelta(minutes=1)
        ]
        
        # If at limit, wait
        if len(self.request_times) >= self.requests_per_minute:
            sleep_time = 60 - (now - self.request_times[0]).seconds
            print(f"Rate limit reached. Waiting {sleep_time} seconds...")
            time.sleep(sleep_time)
            self.request_times = []
        
        self.request_times.append(now)
    
    def get(self, url: str, **kwargs):
        """Make a rate-limited GET request"""
        self._wait_if_needed()
        return requests.get(url, **kwargs)

# Usage
client = RateLimitedClient(requests_per_minute=10)

for i in range(15):
    response = client.get("https://api.github.com")
    print(f"Request {i+1}: {response.status_code}")
```

---

## Best Practices

1. **Always handle exceptions** when making HTTP requests
2. **Set timeouts** to prevent hanging requests
3. **Use sessions** for multiple requests to the same host (connection pooling)
4. **Close resources** properly or use context managers
5. **Respect rate limits** and implement backoff strategies
6. **Validate and sanitize** user input before including in requests
7. **Use HTTPS** for sensitive data
8. **Don't log sensitive information** (passwords, tokens, API keys)
9. **Handle redirects carefully** (requests follows them by default)
10. **Set appropriate User-Agent** headers

---

## Summary

HTTP is the backbone of web communication. Understanding its fundamental concepts—request/response cycles, methods, status codes, headers, and cookies—is essential for building web applications. Python's `requests` library provides an elegant and powerful way to work with HTTP, making it easy to build robust API clients and web scrapers.

Key takeaways:
- HTTP methods define the action (GET, POST, PUT, DELETE)
- Status codes indicate the result (2xx success, 4xx client error, 5xx server error)
- Headers provide metadata about requests and responses
- Cookies enable stateful communication over stateless HTTP
- Always implement proper error handling and respect server resources