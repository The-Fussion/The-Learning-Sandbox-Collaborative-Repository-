# RESTful API Design in Python

## Table of Contents
1. [Introduction to REST](#introduction-to-rest)
2. [REST Principles and Constraints](#rest-principles-and-constraints)
3. [HTTP Methods and CRUD Operations](#http-methods-and-crud-operations)
4. [Resource Modeling](#resource-modeling)
5. [URL Structure and Naming Conventions](#url-structure-and-naming-conventions)
6. [Status Codes](#status-codes)
7. [Request and Response Formats](#request-and-response-formats)
8. [API Versioning](#api-versioning)
9. [Pagination, Filtering, and Sorting](#pagination-filtering-and-sorting)
10. [Error Handling](#error-handling)
11. [Authentication and Authorization](#authentication-and-authorization)
12. [HATEOAS](#hateoas)
13. [Building REST APIs with Flask](#building-rest-apis-with-flask)
14. [Building REST APIs with FastAPI](#building-rest-apis-with-fastapi)
15. [Building REST APIs with Django REST Framework](#building-rest-apis-with-django-rest-framework)
16. [API Documentation](#api-documentation)
17. [Best Practices](#best-practices)

---

## Introduction to REST

### What is REST?

REST (Representational State Transfer) is an architectural style for designing networked applications. It relies on a stateless, client-server protocol (usually HTTP) and treats server-side data as resources that can be created, read, updated, or deleted.

### Key Concepts

- **Resource**: Any information that can be named (users, posts, products, etc.)
- **Representation**: How a resource is represented (JSON, XML, etc.)
- **URI**: Uniform Resource Identifier - unique address for a resource
- **Stateless**: Each request contains all information needed to process it
- **Client-Server**: Separation of concerns between UI and data storage

### Why REST?

- **Scalability**: Stateless nature allows easy horizontal scaling
- **Flexibility**: Client and server can evolve independently
- **Platform Independence**: Works with any programming language
- **Simplicity**: Uses standard HTTP methods
- **Cacheability**: Responses can be cached for better performance

---

## REST Principles and Constraints

### 1. Client-Server Architecture

Separation between client (user interface) and server (data storage).

```
Client (React, Mobile App) ←→ REST API ←→ Server (Database)
```

### 2. Statelessness

Each request from client to server must contain all information needed to understand and process the request. The server doesn't store client context between requests.

```python
# Good: Stateless
# Each request includes authentication token
GET /api/users/123
Headers: Authorization: Bearer <token>

# Bad: Stateful
# Relies on server-side session
GET /api/users/current
# Expects server to remember who's logged in
```

### 3. Cacheability

Responses must define themselves as cacheable or non-cacheable.

```python
# Response headers
Cache-Control: max-age=3600, public
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
Last-Modified: Wed, 21 Oct 2025 07:28:00 GMT
```

### 4. Layered System

Client cannot tell whether it's connected directly to the end server or an intermediary (load balancer, cache, etc.).

### 5. Uniform Interface

Consistent interface between components. This includes:
- Resource identification through URIs
- Resource manipulation through representations
- Self-descriptive messages
- HATEOAS (Hypermedia as the Engine of Application State)

### 6. Code on Demand (Optional)

Servers can temporarily extend client functionality by transferring executable code.

---

## HTTP Methods and CRUD Operations

### Standard HTTP Methods

| HTTP Method | CRUD Operation | Description | Idempotent | Safe |
|-------------|----------------|-------------|------------|------|
| GET | Read | Retrieve resource(s) | Yes | Yes |
| POST | Create | Create new resource | No | No |
| PUT | Update | Replace entire resource | Yes | No |
| PATCH | Update | Partial update of resource | No | No |
| DELETE | Delete | Remove resource | Yes | No |
| HEAD | - | Get headers only | Yes | Yes |
| OPTIONS | - | Get supported methods | Yes | Yes |

### Idempotent vs Non-Idempotent

**Idempotent**: Making the same request multiple times has the same effect as making it once.

```python
# Idempotent: Can call multiple times safely
PUT /api/users/123
{
  "name": "John Doe",
  "email": "john@example.com"
}

# Non-idempotent: Creates new resource each time
POST /api/users
{
  "name": "John Doe",
  "email": "john@example.com"
}
```

### Safe Methods

**Safe**: Don't modify resources, only retrieve data.

```python
# Safe: Only reads data
GET /api/users/123

# Not safe: Modifies data
DELETE /api/users/123
```

### Examples of HTTP Methods

```python
# GET - Retrieve resources
GET /api/users           # Get all users
GET /api/users/123       # Get specific user
GET /api/users/123/posts # Get user's posts

# POST - Create new resource
POST /api/users
{
  "name": "Alice",
  "email": "alice@example.com"
}

# PUT - Replace entire resource
PUT /api/users/123
{
  "name": "Alice Smith",
  "email": "alice.smith@example.com",
  "bio": "Developer"
}

# PATCH - Partial update
PATCH /api/users/123
{
  "bio": "Senior Developer"
}

# DELETE - Remove resource
DELETE /api/users/123

# HEAD - Get headers without body
HEAD /api/users/123

# OPTIONS - Get allowed methods
OPTIONS /api/users
```

---

## Resource Modeling

### Resource Identification

Resources should be nouns, not verbs, and represent entities in your domain.

```python
# Good: Resources as nouns
/users
/posts
/comments
/products
/orders

# Bad: Actions as verbs
/getUsers
/createPost
/deleteComment
```

### Resource Hierarchies

Model relationships through URL structure.

```python
# Collection and individual resources
GET    /users              # Get all users
POST   /users              # Create user
GET    /users/123          # Get specific user
PUT    /users/123          # Update user
DELETE /users/123          # Delete user

# Nested resources (relationships)
GET    /users/123/posts           # Get user's posts
POST   /users/123/posts           # Create post for user
GET    /users/123/posts/456       # Get specific post by user
PUT    /users/123/posts/456       # Update post
DELETE /users/123/posts/456       # Delete post

# Deeper nesting (use sparingly)
GET    /users/123/posts/456/comments       # Get post's comments
POST   /users/123/posts/456/comments       # Create comment on post
```

### Singleton vs Collection Resources

```python
# Collection resource (multiple items)
GET /api/users
Response: [
  {"id": 1, "name": "Alice"},
  {"id": 2, "name": "Bob"}
]

# Singleton resource (single item)
GET /api/users/1
Response: {"id": 1, "name": "Alice", "email": "alice@example.com"}

# Singleton sub-resource
GET /api/users/1/profile
Response: {"bio": "Developer", "location": "NYC"}
```

### Resource Relationships

```python
# One-to-Many
GET /api/authors/123/books        # Books by author
GET /api/categories/5/products    # Products in category

# Many-to-Many
GET /api/students/123/courses     # Courses for student
GET /api/courses/456/students     # Students in course

# Alternative: Top-level with filters
GET /api/books?author_id=123
GET /api/products?category_id=5
```

---

## URL Structure and Naming Conventions

### URL Design Principles

```python
# 1. Use nouns, not verbs
# Good
GET    /api/users
POST   /api/users

# Bad
GET    /api/getUsers
POST   /api/createUser

# 2. Use plural nouns for collections
# Good
GET /api/users
GET /api/products

# Bad
GET /api/user
GET /api/product

# 3. Use lowercase and hyphens
# Good
GET /api/blog-posts
GET /api/user-profiles

# Bad
GET /api/blogPosts
GET /api/user_profiles

# 4. Keep URLs simple and intuitive
# Good
GET /api/users/123/orders

# Bad
GET /api/getUserOrdersById/123

# 5. Avoid deep nesting (max 2-3 levels)
# Good
GET /api/posts/123/comments

# Questionable (too deep)
GET /api/users/123/posts/456/comments/789/replies

# Better alternative
GET /api/comments/789/replies
```

### Query Parameters

Use query parameters for filtering, sorting, and pagination.

```python
# Filtering
GET /api/users?status=active
GET /api/products?category=electronics&min_price=100

# Sorting
GET /api/users?sort=created_at&order=desc
GET /api/products?sort=-price  # - prefix for descending

# Pagination
GET /api/users?page=2&per_page=20
GET /api/users?limit=20&offset=40

# Search
GET /api/users?search=john
GET /api/products?q=laptop

# Selecting fields
GET /api/users?fields=id,name,email
GET /api/products?include=category,manufacturer

# Combining parameters
GET /api/products?category=electronics&min_price=100&sort=-price&page=1&per_page=20
```

### Complete URL Structure

```
https://api.example.com/v1/users/123/posts?status=published&sort=-created_at&page=1
│      │              │   │  │     │   │                                        │
│      │              │   │  │     │   └─ Query parameters                     │
│      │              │   │  │     └─ Resource ID                              │
│      │              │   │  └─ Collection                                     │
│      │              │   └─ Parent resource ID                                │
│      │              └─ Version                                                │
│      └─ Subdomain                                                            │
└─ Protocol                                                                    │
```

---

## Status Codes

### Common HTTP Status Codes

#### 2xx Success

```python
# 200 OK - Request succeeded
GET /api/users/123
Response: 200 OK
{"id": 123, "name": "Alice"}

# 201 Created - Resource created successfully
POST /api/users
Response: 201 Created
Location: /api/users/124
{"id": 124, "name": "Bob"}

# 202 Accepted - Request accepted for processing (async)
POST /api/reports/generate
Response: 202 Accepted
{"job_id": "abc123", "status": "processing"}

# 204 No Content - Success but no content to return
DELETE /api/users/123
Response: 204 No Content
```

#### 3xx Redirection

```python
# 301 Moved Permanently
GET /api/old-endpoint
Response: 301 Moved Permanently
Location: /api/v2/new-endpoint

# 304 Not Modified - Cached version is still valid
GET /api/users/123
If-None-Match: "abc123"
Response: 304 Not Modified
```

#### 4xx Client Errors

```python
# 400 Bad Request - Invalid request format
POST /api/users
{"email": "invalid-email"}
Response: 400 Bad Request
{
  "error": "Invalid email format",
  "field": "email"
}

# 401 Unauthorized - Authentication required
GET /api/users/me
Response: 401 Unauthorized
{
  "error": "Authentication required"
}

# 403 Forbidden - Authenticated but not authorized
DELETE /api/users/456
Response: 403 Forbidden
{
  "error": "You don't have permission to delete this user"
}

# 404 Not Found - Resource doesn't exist
GET /api/users/999
Response: 404 Not Found
{
  "error": "User not found"
}

# 405 Method Not Allowed - HTTP method not supported
DELETE /api/users
Response: 405 Method Not Allowed
{
  "error": "DELETE method not allowed on this resource",
  "allowed_methods": ["GET", "POST"]
}

# 409 Conflict - Request conflicts with current state
POST /api/users
{"email": "existing@example.com"}
Response: 409 Conflict
{
  "error": "User with this email already exists"
}

# 422 Unprocessable Entity - Validation failed
POST /api/users
{
  "name": "",
  "email": "invalid"
}
Response: 422 Unprocessable Entity
{
  "errors": {
    "name": ["Name is required"],
    "email": ["Invalid email format"]
  }
}

# 429 Too Many Requests - Rate limit exceeded
GET /api/users
Response: 429 Too Many Requests
Retry-After: 60
{
  "error": "Rate limit exceeded. Try again in 60 seconds"
}
```

#### 5xx Server Errors

```python
# 500 Internal Server Error - Server error
GET /api/users
Response: 500 Internal Server Error
{
  "error": "An unexpected error occurred"
}

# 503 Service Unavailable - Server temporarily unavailable
GET /api/users
Response: 503 Service Unavailable
Retry-After: 300
{
  "error": "Service temporarily unavailable"
}
```

---

## Request and Response Formats

### JSON Format (Most Common)

```python
# Request
POST /api/users
Content-Type: application/json

{
  "name": "Alice",
  "email": "alice@example.com",
  "age": 30,
  "interests": ["coding", "music"],
  "address": {
    "city": "New York",
    "country": "USA"
  }
}

# Response
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/users/123

{
  "id": 123,
  "name": "Alice",
  "email": "alice@example.com",
  "age": 30,
  "interests": ["coding", "music"],
  "address": {
    "city": "New York",
    "country": "USA"
  },
  "created_at": "2025-01-31T10:30:00Z",
  "updated_at": "2025-01-31T10:30:00Z"
}
```

### XML Format (Less Common)

```python
# Request
POST /api/users
Content-Type: application/xml

<?xml version="1.0" encoding="UTF-8"?>
<user>
  <name>Alice</name>
  <email>alice@example.com</email>
  <age>30</age>
</user>

# Response
HTTP/1.1 201 Created
Content-Type: application/xml

<?xml version="1.0" encoding="UTF-8"?>
<user>
  <id>123</id>
  <name>Alice</name>
  <email>alice@example.com</email>
  <age>30</age>
  <created_at>2025-01-31T10:30:00Z</created_at>
</user>
```

### Content Negotiation

```python
# Client specifies preferred format
GET /api/users/123
Accept: application/json

# Server responds with requested format
HTTP/1.1 200 OK
Content-Type: application/json

# Alternative formats
Accept: application/xml
Accept: application/json, application/xml;q=0.9
Accept: */*
```

### Response Structure

```python
# Simple response
{
  "id": 123,
  "name": "Alice",
  "email": "alice@example.com"
}

# Response with metadata
{
  "data": {
    "id": 123,
    "name": "Alice",
    "email": "alice@example.com"
  },
  "meta": {
    "version": "1.0",
    "timestamp": "2025-01-31T10:30:00Z"
  }
}

# Collection response with pagination
{
  "data": [
    {"id": 1, "name": "Alice"},
    {"id": 2, "name": "Bob"}
  ],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total_pages": 5,
    "total_items": 100
  },
  "links": {
    "self": "/api/users?page=1",
    "next": "/api/users?page=2",
    "last": "/api/users?page=5"
  }
}

# Error response
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": {
      "email": ["Invalid email format"],
      "age": ["Must be at least 18"]
    }
  },
  "timestamp": "2025-01-31T10:30:00Z",
  "path": "/api/users"
}
```

---

## API Versioning

### Why Version Your API?

- Allow breaking changes without affecting existing clients
- Support multiple client versions simultaneously
- Enable gradual migration

### Versioning Strategies

#### 1. URL Path Versioning (Most Common)

```python
# Version in URL path
GET /api/v1/users
GET /api/v2/users

# Pros: Clear, easy to understand, easy to route
# Cons: URL changes, need to duplicate code
```

#### 2. Query Parameter Versioning

```python
# Version in query parameter
GET /api/users?version=1
GET /api/users?version=2

# Pros: Clean URLs for default version
# Cons: Easy to miss, complicates routing
```

#### 3. Header Versioning

```python
# Version in custom header
GET /api/users
X-API-Version: 1

GET /api/users
X-API-Version: 2

# Pros: Clean URLs, doesn't affect routing
# Cons: Not visible in URL, harder to test
```

#### 4. Accept Header Versioning

```python
# Version in Accept header
GET /api/users
Accept: application/vnd.myapi.v1+json

GET /api/users
Accept: application/vnd.myapi.v2+json

# Pros: Follows REST principles, content negotiation
# Cons: More complex, harder to test
```

### Versioning Best Practices

```python
# 1. Version from the start
/api/v1/users  # Even for first version

# 2. Support at least 2 versions
/api/v1/users  # Old version
/api/v2/users  # Current version

# 3. Deprecate old versions gracefully
GET /api/v1/users
Response Headers:
Warning: 299 - "API v1 is deprecated. Please migrate to v2 by 2025-12-31"
Sunset: Sat, 31 Dec 2025 23:59:59 GMT

# 4. Document version differences
"""
API v1 -> v2 Changes:
- User.full_name split into first_name and last_name
- Added pagination to all list endpoints
- Changed date format from DD/MM/YYYY to ISO 8601
"""

# 5. Major versions only
v1, v2, v3  # Not v1.1, v1.2
```

---

## Pagination, Filtering, and Sorting

### Pagination

#### Offset-Based Pagination

```python
# Request
GET /api/users?page=2&per_page=20

# Response
{
  "data": [...],
  "pagination": {
    "page": 2,
    "per_page": 20,
    "total_pages": 10,
    "total_items": 200,
    "has_next": true,
    "has_prev": true
  },
  "links": {
    "first": "/api/users?page=1&per_page=20",
    "prev": "/api/users?page=1&per_page=20",
    "self": "/api/users?page=2&per_page=20",
    "next": "/api/users?page=3&per_page=20",
    "last": "/api/users?page=10&per_page=20"
  }
}
```

#### Cursor-Based Pagination

```python
# Request
GET /api/users?cursor=eyJpZCI6MTAwfQ&limit=20

# Response
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6MTIwfQ",
    "prev_cursor": "eyJpZCI6ODB9",
    "has_more": true
  },
  "links": {
    "next": "/api/users?cursor=eyJpZCI6MTIwfQ&limit=20"
  }
}
```

#### Limit-Offset Pagination

```python
# Request
GET /api/users?limit=20&offset=40

# Response
{
  "data": [...],
  "pagination": {
    "limit": 20,
    "offset": 40,
    "total": 200
  }
}
```

### Filtering

```python
# Single filter
GET /api/users?status=active

# Multiple filters
GET /api/users?status=active&role=admin&verified=true

# Range filters
GET /api/products?min_price=100&max_price=500
GET /api/users?created_after=2025-01-01&created_before=2025-12-31

# List filters
GET /api/users?id=1,2,3,4,5
GET /api/products?category=electronics,computers

# Pattern matching
GET /api/users?name_contains=john
GET /api/users?email_endswith=@example.com

# Complex filters (JSON in query param)
GET /api/users?filter={"and":[{"status":"active"},{"or":[{"role":"admin"},{"role":"moderator"}]}]}
```

### Sorting

```python
# Single field ascending
GET /api/users?sort=created_at
GET /api/users?sort=name&order=asc

# Single field descending
GET /api/users?sort=created_at&order=desc
GET /api/users?sort=-created_at  # - prefix for descending

# Multiple fields
GET /api/users?sort=last_name,first_name
GET /api/users?sort=-created_at,name

# Response includes sort information
{
  "data": [...],
  "meta": {
    "sort": [
      {"field": "created_at", "order": "desc"},
      {"field": "name", "order": "asc"}
    ]
  }
}
```

### Field Selection

```python
# Select specific fields
GET /api/users?fields=id,name,email

# Exclude fields
GET /api/users?exclude=password_hash,secret_key

# Nested field selection
GET /api/users?fields=id,name,profile(bio,avatar)

# Response
{
  "data": [
    {
      "id": 1,
      "name": "Alice",
      "email": "alice@example.com"
    }
  ]
}
```

### Searching

```python
# Simple search
GET /api/users?q=john

# Search in specific fields
GET /api/users?search=john&search_fields=name,email

# Full-text search
GET /api/articles?search=python programming&search_type=fulltext

# Response with search metadata
{
  "data": [...],
  "meta": {
    "search": {
      "query": "john",
      "fields": ["name", "email"],
      "total_results": 15
    }
  }
}
```

---

## Error Handling

### Error Response Structure

```python
# Standard error response
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "The requested user does not exist",
    "details": {
      "user_id": 123
    }
  },
  "timestamp": "2025-01-31T10:30:00Z",
  "path": "/api/users/123",
  "request_id": "abc-123-def-456"
}

# Validation error response
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "errors": [
      {
        "field": "email",
        "message": "Invalid email format",
        "code": "INVALID_FORMAT"
      },
      {
        "field": "age",
        "message": "Must be at least 18",
        "code": "MIN_VALUE"
      }
    ]
  },
  "timestamp": "2025-01-31T10:30:00Z",
  "path": "/api/users"
}
```

### Error Code Conventions

```python
# Use consistent error codes
ERROR_CODES = {
    # Resource errors
    "RESOURCE_NOT_FOUND": "The requested resource does not exist",
    "RESOURCE_ALREADY_EXISTS": "Resource already exists",
    "RESOURCE_CONFLICT": "Resource state conflict",
    
    # Validation errors
    "VALIDATION_ERROR": "Request validation failed",
    "INVALID_FORMAT": "Invalid format",
    "REQUIRED_FIELD": "Required field missing",
    
    # Authentication/Authorization
    "UNAUTHORIZED": "Authentication required",
    "FORBIDDEN": "Insufficient permissions",
    "INVALID_TOKEN": "Invalid or expired token",
    
    # Rate limiting
    "RATE_LIMIT_EXCEEDED": "Too many requests",
    
    # Server errors
    "INTERNAL_ERROR": "Internal server error",
    "SERVICE_UNAVAILABLE": "Service temporarily unavailable"
}
```

### HTTP Status Code Mapping

```python
def get_status_code(error_code: str) -> int:
    """Map error codes to HTTP status codes."""
    mapping = {
        # 4xx Client Errors
        "VALIDATION_ERROR": 422,
        "INVALID_FORMAT": 422,
        "REQUIRED_FIELD": 422,
        "RESOURCE_NOT_FOUND": 404,
        "RESOURCE_ALREADY_EXISTS": 409,
        "UNAUTHORIZED": 401,
        "FORBIDDEN": 403,
        "INVALID_TOKEN": 401,
        "RATE_LIMIT_EXCEEDED": 429,
        
        # 5xx Server Errors
        "INTERNAL_ERROR": 500,
        "SERVICE_UNAVAILABLE": 503,
    }
    return mapping.get(error_code, 500)
```

---

## Authentication and Authorization

### Authentication Methods

#### 1. API Keys

```python
# API key in header
GET /api/users
X-API-Key: abc123def456

# API key in query parameter (less secure)
GET /api/users?api_key=abc123def456
```

#### 2. Bearer Token (JWT)

```python
# Token in Authorization header
GET /api/users
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# Token flow
# 1. Login to get token
POST /api/auth/login
{
  "email": "user@example.com",
  "password": "password123"
}

Response:
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer",
  "expires_in": 3600
}

# 2. Use token in subsequent requests
GET /api/users/me
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

#### 3. OAuth 2.0

```python
# OAuth flow
# 1. Authorization request
GET /oauth/authorize?
    response_type=code&
    client_id=CLIENT_ID&
    redirect_uri=CALLBACK_URL&
    scope=read write

# 2. Exchange code for token
POST /oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=AUTH_CODE&
client_id=CLIENT_ID&
client_secret=CLIENT_SECRET&
redirect_uri=CALLBACK_URL

# 3. Use access token
GET /api/users
Authorization: Bearer ACCESS_TOKEN
```

### Authorization Patterns

```python
# Role-Based Access Control (RBAC)
GET /api/admin/users
Authorization: Bearer <token>
# Token contains: {"role": "admin"}

# Permission-Based
GET /api/posts/123
Authorization: Bearer <token>
# Token contains: {"permissions": ["posts:read", "posts:update"]}

# Attribute-Based Access Control (ABAC)
GET /api/users/123/data
# Checks: Is requester the user OR has admin role OR is in same organization
```

---

## HATEOAS

HATEOAS (Hypermedia as the Engine of Application State) includes links in responses to guide clients to related resources.

```python
# Response with HATEOAS links
GET /api/users/123

{
  "id": 123,
  "name": "Alice",
  "email": "alice@example.com",
  "_links": {
    "self": {
      "href": "/api/users/123"
    },
    "posts": {
      "href": "/api/users/123/posts"
    },
    "followers": {
      "href": "/api/users/123/followers"
    },
    "following": {
      "href": "/api/users/123/following"
    },
    "update": {
      "href": "/api/users/123",
      "method": "PUT"
    },
    "delete": {
      "href": "/api/users/123",
      "method": "DELETE"
    }
  }
}

# Collection with HATEOAS
GET /api/users?page=2

{
  "data": [
    {
      "id": 1,
      "name": "Alice",
      "_links": {
        "self": {"href": "/api/users/1"}
      }
    }
  ],
  "_links": {
    "self": {"href": "/api/users?page=2"},
    "first": {"href": "/api/users?page=1"},
    "prev": {"href": "/api/users?page=1"},
    "next": {"href": "/api/users?page=3"},
    "last": {"href": "/api/users?page=10"}
  }
}
```

---

## Building REST APIs with Flask

### Basic Flask REST API

```python
from flask import Flask, jsonify, request, abort
from functools import wraps
import jwt
from datetime import datetime, timedelta

app = Flask(__name__)
app.config['SECRET_KEY'] = 'your-secret-key'

# Mock database
users = {
    1: {"id": 1, "name": "Alice", "email": "alice@example.com"},
    2: {"id": 2, "name": "Bob", "email": "bob@example.com"}
}
next_user_id = 3

# Error handlers
@app.errorhandler(404)
def not_found(error):
    return jsonify({
        "error": {
            "code": "RESOURCE_NOT_FOUND",
            "message": "The requested resource was not found"
        }
    }), 404

@app.errorhandler(400)
def bad_request(error):
    return jsonify({
        "error": {
            "code": "BAD_REQUEST",
            "message": str(error)
        }
    }), 400

# Authentication decorator
def token_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        token = request.headers.get('Authorization')
        
        if not token:
            return jsonify({
                "error": {
                    "code": "UNAUTHORIZED",
                    "message": "Token is missing"
                }
            }), 401
        
        try:
            # Remove 'Bearer ' prefix
            token = token.split()[1] if token.startswith('Bearer ') else token
            data = jwt.decode(token, app.config['SECRET_KEY'], algorithms=["HS256"])
            current_user_id = data['user_id']
        except:
            return jsonify({
                "error": {
                    "code": "UNAUTHORIZED",
                    "message": "Token is invalid"
                }
            }), 401
        
        return f(current_user_id, *args, **kwargs)
    
    return decorated

# Routes
@app.route('/api/v1/users', methods=['GET'])
def get_users():
    """Get all users."""
    # Pagination
    page = request.args.get('page', 1, type=int)
    per_page = request.args.get('per_page', 10, type=int)
    
    # Filter
    status = request.args.get('status')
    
    user_list = list(users.values())
    
    # Simple pagination
    start = (page - 1) * per_page
    end = start + per_page
    paginated_users = user_list[start:end]
    
    return jsonify({
        "data": paginated_users,
        "pagination": {
            "page": page,
            "per_page": per_page,
            "total": len(user_list),
            "total_pages": (len(user_list) + per_page - 1) // per_page
        }
    }), 200

@app.route('/api/v1/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    """Get a specific user."""
    user = users.get(user_id)
    
    if not user:
        abort(404)
    
    return jsonify(user), 200

@app.route('/api/v1/users', methods=['POST'])
def create_user():
    """Create a new user."""
    global next_user_id
    
    if not request.json:
        abort(400, description="Request must be JSON")
    
    # Validate required fields
    required_fields = ['name', 'email']
    for field in required_fields:
        if field not in request.json:
            return jsonify({
                "error": {
                    "code": "VALIDATION_ERROR",
                    "message": f"Missing required field: {field}"
                }
            }), 422
    
    # Create user
    user = {
        "id": next_user_id,
        "name": request.json['name'],
        "email": request.json['email'],
        "created_at": datetime.utcnow().isoformat()
    }
    
    users[next_user_id] = user
    next_user_id += 1
    
    return jsonify(user), 201

@app.route('/api/v1/users/<int:user_id>', methods=['PUT'])
@token_required
def update_user(current_user_id, user_id):
    """Update a user (full replacement)."""
    if user_id not in users:
        abort(404)
    
    if not request.json:
        abort(400, description="Request must be JSON")
    
    # Check authorization
    if current_user_id != user_id:
        return jsonify({
            "error": {
                "code": "FORBIDDEN",
                "message": "You can only update your own profile"
            }
        }), 403
    
    # Update user
    users[user_id].update({
        "name": request.json.get('name', users[user_id]['name']),
        "email": request.json.get('email', users[user_id]['email']),
        "updated_at": datetime.utcnow().isoformat()
    })
    
    return jsonify(users[user_id]), 200

@app.route('/api/v1/users/<int:user_id>', methods=['PATCH'])
@token_required
def patch_user(current_user_id, user_id):
    """Partially update a user."""
    if user_id not in users:
        abort(404)
    
    if not request.json:
        abort(400, description="Request must be JSON")
    
    # Update only provided fields
    for key, value in request.json.items():
        if key in ['name', 'email']:
            users[user_id][key] = value
    
    users[user_id]['updated_at'] = datetime.utcnow().isoformat()
    
    return jsonify(users[user_id]), 200

@app.route('/api/v1/users/<int:user_id>', methods=['DELETE'])
@token_required
def delete_user(current_user_id, user_id):
    """Delete a user."""
    if user_id not in users:
        abort(404)
    
    # Check authorization
    if current_user_id != user_id:
        return jsonify({
            "error": {
                "code": "FORBIDDEN",
                "message": "You can only delete your own account"
            }
        }), 403
    
    del users[user_id]
    return '', 204

# Authentication endpoint
@app.route('/api/v1/auth/login', methods=['POST'])
def login():
    """Login to get JWT token."""
    if not request.json or 'email' not in request.json:
        abort(400)
    
    email = request.json['email']
    
    # Find user by email
    user = next((u for u in users.values() if u['email'] == email), None)
    
    if not user:
        return jsonify({
            "error": {
                "code": "INVALID_CREDENTIALS",
                "message": "Invalid email or password"
            }
        }), 401
    
    # Generate token
    token = jwt.encode({
        'user_id': user['id'],
        'exp': datetime.utcnow() + timedelta(hours=24)
    }, app.config['SECRET_KEY'], algorithm="HS256")
    
    return jsonify({
        "access_token": token,
        "token_type": "bearer",
        "expires_in": 86400
    }), 200

# Health check
@app.route('/api/v1/health', methods=['GET'])
def health_check():
    """API health check."""
    return jsonify({
        "status": "healthy",
        "version": "1.0.0",
        "timestamp": datetime.utcnow().isoformat()
    }), 200

if __name__ == '__main__':
    app.run(debug=True, port=5000)
```

### Flask-RESTful Extension

```python
from flask import Flask
from flask_restful import Resource, Api, reqparse, fields, marshal_with, abort

app = Flask(__name__)
api = Api(app)

# Resource fields for serialization
user_fields = {
    'id': fields.Integer,
    'name': fields.String,
    'email': fields.String,
    'created_at': fields.DateTime(dt_format='iso8601')
}

# Request parser for validation
user_parser = reqparse.RequestParser()
user_parser.add_argument('name', type=str, required=True, help='Name is required')
user_parser.add_argument('email', type=str, required=True, help='Email is required')

# Mock database
users = {}
next_id = 1

class UserList(Resource):
    """User collection resource."""
    
    @marshal_with(user_fields)
    def get(self):
        """Get all users."""
        return list(users.values())
    
    @marshal_with(user_fields)
    def post(self):
        """Create a new user."""
        global next_id
        args = user_parser.parse_args()
        
        user = {
            'id': next_id,
            'name': args['name'],
            'email': args['email'],
            'created_at': datetime.utcnow()
        }
        
        users[next_id] = user
        next_id += 1
        
        return user, 201

class UserResource(Resource):
    """Individual user resource."""
    
    @marshal_with(user_fields)
    def get(self, user_id):
        """Get a specific user."""
        if user_id not in users:
            abort(404, message=f"User {user_id} not found")
        return users[user_id]
    
    @marshal_with(user_fields)
    def put(self, user_id):
        """Update a user."""
        if user_id not in users:
            abort(404, message=f"User {user_id} not found")
        
        args = user_parser.parse_args()
        users[user_id].update(args)
        return users[user_id]
    
    def delete(self, user_id):
        """Delete a user."""
        if user_id not in users:
            abort(404, message=f"User {user_id} not found")
        
        del users[user_id]
        return '', 204

# Register resources
api.add_resource(UserList, '/api/v1/users')
api.add_resource(UserResource, '/api/v1/users/<int:user_id>')

if __name__ == '__main__':
    app.run(debug=True)
```

---

## Building REST APIs with FastAPI

FastAPI is a modern, fast framework for building APIs with automatic documentation.

```python
from fastapi import FastAPI, HTTPException, Depends, status, Query, Path
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from pydantic import BaseModel, EmailStr, Field
from typing import List, Optional
from datetime import datetime
import jwt

app = FastAPI(
    title="My API",
    description="A RESTful API built with FastAPI",
    version="1.0.0"
)

security = HTTPBearer()

# Pydantic models for request/response validation
class UserBase(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    email: EmailStr

class UserCreate(UserBase):
    password: str = Field(..., min_length=8)

class UserUpdate(BaseModel):
    name: Optional[str] = Field(None, min_length=1, max_length=100)
    email: Optional[EmailStr] = None

class User(UserBase):
    id: int
    created_at: datetime
    updated_at: Optional[datetime] = None
    
    class Config:
        from_attributes = True

class PaginatedUsers(BaseModel):
    data: List[User]
    total: int
    page: int
    per_page: int
    total_pages: int

class Token(BaseModel):
    access_token: str
    token_type: str
    expires_in: int

class ErrorResponse(BaseModel):
    error: dict

# Mock database
users_db = {}
next_user_id = 1

# Dependency for authentication
def get_current_user(credentials: HTTPAuthorizationCredentials = Depends(security)):
    """Verify JWT token and return current user."""
    token = credentials.credentials
    
    try:
        payload = jwt.decode(token, "secret-key", algorithms=["HS256"])
        user_id = payload.get("user_id")
        
        if user_id not in users_db:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid token"
            )
        
        return users_db[user_id]
    except jwt.ExpiredSignatureError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token has expired"
        )
    except jwt.InvalidTokenError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid token"
        )

# Routes
@app.get("/api/v1/health")
async def health_check():
    """Health check endpoint."""
    return {
        "status": "healthy",
        "version": "1.0.0",
        "timestamp": datetime.utcnow().isoformat()
    }

@app.get(
    "/api/v1/users",
    response_model=PaginatedUsers,
    summary="Get all users",
    description="Retrieve a paginated list of all users"
)
async def get_users(
    page: int = Query(1, ge=1, description="Page number"),
    per_page: int = Query(10, ge=1, le=100, description="Items per page"),
    status: Optional[str] = Query(None, description="Filter by status")
):
    """Get all users with pagination."""
    user_list = list(users_db.values())
    
    # Calculate pagination
    start = (page - 1) * per_page
    end = start + per_page
    paginated_users = user_list[start:end]
    
    return {
        "data": paginated_users,
        "total": len(user_list),
        "page": page,
        "per_page": per_page,
        "total_pages": (len(user_list) + per_page - 1) // per_page
    }

@app.get(
    "/api/v1/users/{user_id}",
    response_model=User,
    summary="Get a user",
    responses={
        404: {"model": ErrorResponse, "description": "User not found"}
    }
)
async def get_user(
    user_id: int = Path(..., ge=1, description="The ID of the user to retrieve")
):
    """Get a specific user by ID."""
    if user_id not in users_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail={
                "error": {
                    "code": "RESOURCE_NOT_FOUND",
                    "message": f"User {user_id} not found"
                }
            }
        )
    
    return users_db[user_id]

@app.post(
    "/api/v1/users",
    response_model=User,
    status_code=status.HTTP_201_CREATED,
    summary="Create a user",
    responses={
        201: {"description": "User created successfully"},
        422: {"model": ErrorResponse, "description": "Validation error"}
    }
)
async def create_user(user: UserCreate):
    """Create a new user."""
    global next_user_id
    
    # Check if email already exists
    if any(u['email'] == user.email for u in users_db.values()):
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail={
                "error": {
                    "code": "RESOURCE_ALREADY_EXISTS",
                    "message": "User with this email already exists"
                }
            }
        )
    
    new_user = {
        "id": next_user_id,
        "name": user.name,
        "email": user.email,
        "created_at": datetime.utcnow(),
        "updated_at": None
    }
    
    users_db[next_user_id] = new_user
    next_user_id += 1
    
    return new_user

@app.put(
    "/api/v1/users/{user_id}",
    response_model=User,
    summary="Update a user",
    dependencies=[Depends(get_current_user)]
)
async def update_user(
    user_id: int,
    user_update: UserUpdate,
    current_user: dict = Depends(get_current_user)
):
    """Update a user (authenticated)."""
    if user_id not in users_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found"
        )
    
    # Check authorization
    if current_user['id'] != user_id:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="You can only update your own profile"
        )
    
    # Update user
    update_data = user_update.dict(exclude_unset=True)
    users_db[user_id].update(update_data)
    users_db[user_id]['updated_at'] = datetime.utcnow()
    
    return users_db[user_id]

@app.patch(
    "/api/v1/users/{user_id}",
    response_model=User,
    summary="Partially update a user"
)
async def patch_user(user_id: int, user_update: UserUpdate):
    """Partially update a user."""
    if user_id not in users_db:
        raise HTTPException(status_code=404, detail="User not found")
    
    update_data = user_update.dict(exclude_unset=True)
    users_db[user_id].update(update_data)
    users_db[user_id]['updated_at'] = datetime.utcnow()
    
    return users_db[user_id]

@app.delete(
    "/api/v1/users/{user_id}",
    status_code=status.HTTP_204_NO_CONTENT,
    summary="Delete a user"
)
async def delete_user(
    user_id: int,
    current_user: dict = Depends(get_current_user)
):
    """Delete a user."""
    if user_id not in users_db:
        raise HTTPException(status_code=404, detail="User not found")
    
    # Check authorization
    if current_user['id'] != user_id:
        raise HTTPException(
            status_code=403,
            detail="You can only delete your own account"
        )
    
    del users_db[user_id]
    return None

@app.post("/api/v1/auth/login", response_model=Token)
async def login(email: EmailStr, password: str):
    """Login to get access token."""
    # Find user by email
    user = next((u for u in users_db.values() if u['email'] == email), None)
    
    if not user:
        raise HTTPException(
            status_code=401,
            detail="Invalid credentials"
        )
    
    # Generate token
    token = jwt.encode({
        'user_id': user['id'],
        'exp': datetime.utcnow().timestamp() + 86400
    }, "secret-key", algorithm="HS256")
    
    return {
        "access_token": token,
        "token_type": "bearer",
        "expires_in": 86400
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## Building REST APIs with Django REST Framework

```python
# models.py
from django.db import models
from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    bio = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

class Post(models.Model):
    author = models.ForeignKey(User, on_delete=models.CASCADE, related_name='posts')
    title = models.CharField(max_length=200)
    content = models.TextField()
    published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        ordering = ['-created_at']

# serializers.py
from rest_framework import serializers
from .models import User, Post

class UserSerializer(serializers.ModelSerializer):
    post_count = serializers.SerializerMethodField()
    
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'bio', 'post_count', 'created_at']
        read_only_fields = ['id', 'created_at']
    
    def get_post_count(self, obj):
        return obj.posts.count()

class UserCreateSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True, min_length=8)
    
    class Meta:
        model = User
        fields = ['username', 'email', 'password', 'bio']
    
    def create(self, validated_data):
        user = User.objects.create_user(**validated_data)
        return user

class PostSerializer(serializers.ModelSerializer):
    author = UserSerializer(read_only=True)
    author_id = serializers.IntegerField(write_only=True)
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'published', 'author', 'author_id', 
                  'created_at', 'updated_at']
        read_only_fields = ['id', 'created_at', 'updated_at']

# views.py
from rest_framework import viewsets, status, filters
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated, IsAuthenticatedOrReadOnly
from django_filters.rest_framework import DjangoFilterBackend
from .models import User, Post
from .serializers import UserSerializer, UserCreateSerializer, PostSerializer

class UserViewSet(viewsets.ModelViewSet):
    """
    ViewSet for User CRUD operations.
    
    list: Get all users
    create: Create a new user
    retrieve: Get a specific user
    update: Update a user
    partial_update: Partially update a user
    destroy: Delete a user
    """
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    filter_backends = [filters.SearchFilter, filters.OrderingFilter, DjangoFilterBackend]
    search_fields = ['username', 'email']
    ordering_fields = ['created_at', 'username']
    filterset_fields = ['is_active']
    
    def get_serializer_class(self):
        if self.action == 'create':
            return UserCreateSerializer
        return UserSerializer
    
    def get_permissions(self):
        if self.action in ['update', 'partial_update', 'destroy']:
            return [IsAuthenticated()]
        return super().get_permissions()
    
    @action(detail=True, methods=['get'])
    def posts(self, request, pk=None):
        """Get all posts by a user."""
        user = self.get_object()
        posts = user.posts.all()
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data)
    
    @action(detail=False, methods=['get'], permission_classes=[IsAuthenticated])
    def me(self, request):
        """Get current user."""
        serializer = self.get_serializer(request.user)
        return Response(serializer.data)

class PostViewSet(viewsets.ModelViewSet):
    """ViewSet for Post CRUD operations."""
    queryset = Post.objects.all()
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    filter_backends = [filters.SearchFilter, filters.OrderingFilter, DjangoFilterBackend]
    search_fields = ['title', 'content']
    ordering_fields = ['created_at', 'title']
    filterset_fields = ['published', 'author']
    
    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
    
    @action(detail=False, methods=['get'])
    def published(self, request):
        """Get only published posts."""
        posts = self.queryset.filter(published=True)
        serializer = self.get_serializer(posts, many=True)
        return Response(serializer.data)

# urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import UserViewSet, PostViewSet

router = DefaultRouter()
router.register(r'users', UserViewSet)
router.register(r'posts', PostViewSet)

urlpatterns = [
    path('api/v1/', include(router.urls)),
]

# Pagination in settings.py
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    ],
}
```

---

## API Documentation

### OpenAPI/Swagger with FastAPI

FastAPI automatically generates OpenAPI documentation.

```python
from fastapi import FastAPI
from fastapi.openapi.utils import get_openapi

app = FastAPI()

def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema
    
    openapi_schema = get_openapi(
        title="My API",
        version="1.0.0",
        description="A comprehensive REST API",
        routes=app.routes,
    )
    
    # Customize schema
    openapi_schema["info"]["x-logo"] = {
        "url": "https://example.com/logo.png"
    }
    
    app.openapi_schema = openapi_schema
    return app.openapi_schema

app.openapi = custom_openapi

# Access docs at:
# http://localhost:8000/docs (Swagger UI)
# http://localhost:8000/redoc (ReDoc)
```

### API Blueprint Documentation

```markdown
# API Documentation

## Base URL
```
https://api.example.com/v1
```

## Authentication
All authenticated endpoints require a Bearer token in the Authorization header:
```
Authorization: Bearer <your_token>
```

## Endpoints

### Users

#### List Users
```http
GET /users
```

**Query Parameters:**
- `page` (integer, optional): Page number (default: 1)
- `per_page` (integer, optional): Items per page (default: 20)
- `status` (string, optional): Filter by status

**Response:** `200 OK`
```json
{
  "data": [
    {
      "id": 1,
      "name": "Alice",
      "email": "alice@example.com"
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total": 100
  }
}
```

#### Create User
```http
POST /users
```

**Request Body:**
```json
{
  "name": "Alice",
  "email": "alice@example.com",
  "password": "secure_password"
}
```

**Response:** `201 Created`
```json
{
  "id": 1,
  "name": "Alice",
  "email": "alice@example.com",
  "created_at": "2025-01-31T10:30:00Z"
}
```

## Error Responses

All error responses follow this format:
```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {}
  }
}
```
```

---

## Best Practices

### 1. Use Proper HTTP Methods

```python
# Good: RESTful design
GET    /api/users          # Get all users
POST   /api/users          # Create user
GET    /api/users/123      # Get user
PUT    /api/users/123      # Update user (full)
PATCH  /api/users/123      # Update user (partial)
DELETE /api/users/123      # Delete user

# Bad: Non-RESTful
GET    /api/getUsers
POST   /api/createUser
POST   /api/updateUser
POST   /api/deleteUser
```

### 2. Return Appropriate Status Codes

```python
# Create: 201 Created
@app.post("/users")
def create_user(user: User):
    # ... create user ...
    return user, 201

# Delete: 204 No Content
@app.delete("/users/{id}")
def delete_user(id: int):
    # ... delete user ...
    return None, 204

# Not found: 404
@app.get("/users/{id}")
def get_user(id: int):
    if not user_exists(id):
        raise HTTPException(status_code=404, detail="User not found")
```

### 3. Use Consistent Naming

```python
# Good: Consistent, plural nouns
/api/users
/api/posts
/api/comments

# Bad: Inconsistent
/api/user
/api/getAllPosts
/api/comment
```

### 4. Version Your API

```python
# Include version in URL
/api/v1/users
/api/v2/users

# Or use headers
GET /api/users
Accept: application/vnd.myapi.v1+json
```

### 5. Implement Pagination

```python
# Always paginate large collections
@app.get("/users")
def get_users(page: int = 1, per_page: int = 20):
    # Return paginated results
    pass
```

### 6. Provide Filtering and Sorting

```python
# Allow flexible querying
GET /api/products?category=electronics&sort=-price&min_price=100
```

### 7. Use HTTPS

```python
# Always use HTTPS in production
# Redirect HTTP to HTTPS
@app.middleware("http")
async def redirect_to_https(request, call_next):
    if request.url.scheme == "http":
        url = request.url.replace(scheme="https")
        return RedirectResponse(url)
    return await call_next(request)
```

### 8. Implement Rate Limiting

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.get("/api/users")
@limiter.limit("100/hour")
def get_users():
    pass
```

### 9. Use Caching

```python
from fastapi_cache import FastAPICache
from fastapi_cache.decorator import cache

@app.get("/users/{id}")
@cache(expire=60)  # Cache for 60 seconds
def get_user(id: int):
    pass
```

### 10. Handle Errors Consistently

```python
# Use consistent error format
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": {
      "email": ["Invalid format"]
    }
  }
}
```

### 11. Document Your API

```python
# Use FastAPI's automatic documentation
@app.get(
    "/users/{id}",
    summary="Get a user",
    description="Retrieve a user by their ID",
    response_description="The requested user"
)
def get_user(id: int):
    pass
```

### 12. Use ETags for Caching

```python
from hashlib import md5

@app.get("/users/{id}")
def get_user(id: int, if_none_match: str = Header(None)):
    user = get_user_from_db(id)
    user_json = json.dumps(user)
    etag = md5(user_json.encode()).hexdigest()
    
    if if_none_match == etag:
        return Response(status_code=304)  # Not Modified
    
    return Response(
        content=user_json,
        headers={"ETag": etag}
    )
```

---

## Summary

This comprehensive guide covered:

1. **REST Principles**: Understanding the architectural constraints
2. **HTTP Methods**: Proper usage of GET, POST, PUT, PATCH, DELETE
3. **Resource Modeling**: How to structure your API resources
4. **URL Design**: Best practices for URL structure and naming
5. **Status Codes**: Using appropriate HTTP status codes
6. **Request/Response Formats**: JSON, XML, and content negotiation
7. **API Versioning**: Different strategies and best practices
8. **Pagination & Filtering**: Handling large datasets
9. **Error Handling**: Consistent error responses
10. **Authentication**: Various methods (API keys, JWT, OAuth)
11. **HATEOAS**: Hypermedia-driven APIs
12. **Implementation**: Examples with Flask, FastAPI, and Django REST Framework
13. **Documentation**: OpenAPI/Swagger integration
14. **Best Practices**: Production-ready API design patterns

Remember: A well-designed REST API is intuitive, consistent, well-documented, and follows HTTP standards!