# AJAX and Asynchronous Requests in Python

## Table of Contents
1. [Introduction to AJAX](#introduction-to-ajax)
2. [How AJAX Works](#how-ajax-works)
3. [JSON Responses in Python](#json-responses-in-python)
4. [AJAX with Flask](#ajax-with-flask)
5. [AJAX with FastAPI](#ajax-with-fastapi)
6. [AJAX with Django](#ajax-with-django)
7. [Frontend AJAX Implementation](#frontend-ajax-implementation)
8. [Real-Time Updates with WebSockets](#real-time-updates-with-websockets)
9. [Server-Sent Events (SSE)](#server-sent-events-sse)
10. [Long Polling](#long-polling)
11. [Error Handling](#error-handling)
12. [Security Considerations](#security-considerations)
13. [Performance Optimization](#performance-optimization)
14. [Best Practices](#best-practices)

---

## Introduction to AJAX

### What is AJAX?

```python
"""
AJAX (Asynchronous JavaScript and XML) is a technique for creating
interactive web applications that can send and receive data from a
server asynchronously without interfering with the display and
behavior of the existing page.

Key Benefits:
1. Partial page updates (no full page reload)
2. Better user experience (faster, more responsive)
3. Reduced server load (only send necessary data)
4. Rich, interactive interfaces

Modern AJAX:
- Uses JSON instead of XML (lighter, easier to parse)
- Uses Fetch API or Axios instead of XMLHttpRequest
- Integrates with modern frameworks (React, Vue, Angular)
"""
```

### Traditional vs AJAX Requests

```python
"""
Traditional Request Flow:
1. User clicks link/submits form
2. Browser sends full HTTP request
3. Server processes and renders entire HTML page
4. Browser receives and displays new page
5. Page reloads completely

AJAX Request Flow:
1. User triggers action (click, input, etc.)
2. JavaScript sends HTTP request in background
3. Server processes and returns data (JSON)
4. JavaScript receives response
5. JavaScript updates specific page elements
6. No page reload
"""
```

### Use Cases

```python
"""
Common AJAX Use Cases:

1. Form Submission
   - Submit forms without page reload
   - Instant validation feedback
   - Progress indicators

2. Auto-complete/Search
   - Search-as-you-type
   - Live search results
   - Filter lists dynamically

3. Infinite Scroll
   - Load more content automatically
   - Pagination without page reload

4. Live Updates
   - Notifications
   - Chat messages
   - Real-time data (stock prices, sports scores)

5. Dynamic Content
   - Load content on demand
   - Tab switching without reload
   - Modal dialogs with server data

6. Interactive Features
   - Like/vote buttons
   - Shopping cart updates
   - Comments without reload
"""
```

---

## How AJAX Works

### Request-Response Cycle

```javascript
// Frontend: Send AJAX request
fetch('/api/users')
    .then(response => response.json())
    .then(data => {
        console.log(data);
        // Update UI with data
    })
    .catch(error => console.error('Error:', error));
```

```python
# Backend: Return JSON response
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/api/users')
def get_users():
    users = [
        {'id': 1, 'name': 'Alice'},
        {'id': 2, 'name': 'Bob'}
    ]
    return jsonify(users)
```

### HTTP Methods in AJAX

```javascript
// GET request - Retrieve data
fetch('/api/users/1')
    .then(response => response.json())
    .then(data => console.log(data));

// POST request - Create data
fetch('/api/users', {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({name: 'Charlie', email: 'charlie@example.com'})
})
    .then(response => response.json())
    .then(data => console.log(data));

// PUT request - Update data
fetch('/api/users/1', {
    method: 'PUT',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({name: 'Alice Updated'})
})
    .then(response => response.json())
    .then(data => console.log(data));

// DELETE request - Delete data
fetch('/api/users/1', {
    method: 'DELETE'
})
    .then(response => response.json())
    .then(data => console.log(data));
```

---

## JSON Responses in Python

### Creating JSON Responses

```python
from flask import Flask, jsonify, request
from datetime import datetime
import json

app = Flask(__name__)

# Simple JSON response
@app.route('/api/hello')
def hello():
    return jsonify({'message': 'Hello, World!'})

# JSON with multiple fields
@app.route('/api/user/<int:user_id>')
def get_user(user_id):
    user = {
        'id': user_id,
        'name': 'Alice',
        'email': 'alice@example.com',
        'created_at': datetime.utcnow().isoformat()
    }
    return jsonify(user)

# JSON array
@app.route('/api/users')
def get_users():
    users = [
        {'id': 1, 'name': 'Alice'},
        {'id': 2, 'name': 'Bob'},
        {'id': 3, 'name': 'Charlie'}
    ]
    return jsonify(users)

# Nested JSON
@app.route('/api/user/<int:user_id>/profile')
def get_user_profile(user_id):
    profile = {
        'user': {
            'id': user_id,
            'name': 'Alice',
            'email': 'alice@example.com'
        },
        'settings': {
            'theme': 'dark',
            'notifications': True
        },
        'stats': {
            'posts': 42,
            'followers': 1337,
            'following': 256
        }
    }
    return jsonify(profile)

# Custom status code
@app.route('/api/users', methods=['POST'])
def create_user():
    data = request.get_json()
    
    # Validate data
    if not data or 'name' not in data:
        return jsonify({'error': 'Name is required'}), 400
    
    # Create user
    user = {
        'id': 123,
        'name': data['name'],
        'email': data.get('email'),
        'created_at': datetime.utcnow().isoformat()
    }
    
    return jsonify(user), 201  # 201 Created

# Error response
@app.route('/api/error')
def error_example():
    return jsonify({
        'error': {
            'code': 'NOT_FOUND',
            'message': 'Resource not found',
            'details': {}
        }
    }), 404
```

### Custom JSON Encoder

```python
from flask import Flask, jsonify
from flask.json.provider import DefaultJSONProvider
from datetime import datetime, date
from decimal import Decimal
import uuid

class CustomJSONProvider(DefaultJSONProvider):
    """Custom JSON encoder for special types."""
    
    def default(self, obj):
        """Convert special types to JSON-serializable format."""
        if isinstance(obj, datetime):
            return obj.isoformat()
        if isinstance(obj, date):
            return obj.isoformat()
        if isinstance(obj, Decimal):
            return float(obj)
        if isinstance(obj, uuid.UUID):
            return str(obj)
        if isinstance(obj, set):
            return list(obj)
        
        # Let the base class handle other types
        return super().default(obj)

app = Flask(__name__)
app.json = CustomJSONProvider(app)

@app.route('/api/data')
def get_data():
    """Return data with special types."""
    data = {
        'id': uuid.uuid4(),
        'created_at': datetime.utcnow(),
        'date': date.today(),
        'price': Decimal('99.99'),
        'tags': {'python', 'flask', 'ajax'}
    }
    return jsonify(data)
```

---

## AJAX with Flask

### Basic Flask AJAX Example

```python
from flask import Flask, render_template, request, jsonify
import time

app = Flask(__name__)

# Mock database
users = [
    {'id': 1, 'name': 'Alice', 'email': 'alice@example.com'},
    {'id': 2, 'name': 'Bob', 'email': 'bob@example.com'},
    {'id': 3, 'name': 'Charlie', 'email': 'charlie@example.com'}
]

@app.route('/')
def index():
    """Main page."""
    return render_template('index.html')

# GET - Retrieve all users
@app.route('/api/users', methods=['GET'])
def get_users():
    """Get all users."""
    # Simulate database query delay
    time.sleep(0.1)
    
    # Optional filtering
    search = request.args.get('search', '')
    if search:
        filtered_users = [
            u for u in users 
            if search.lower() in u['name'].lower()
        ]
        return jsonify(filtered_users)
    
    return jsonify(users)

# GET - Retrieve single user
@app.route('/api/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    """Get user by ID."""
    user = next((u for u in users if u['id'] == user_id), None)
    
    if not user:
        return jsonify({'error': 'User not found'}), 404
    
    return jsonify(user)

# POST - Create user
@app.route('/api/users', methods=['POST'])
def create_user():
    """Create new user."""
    data = request.get_json()
    
    # Validate data
    if not data:
        return jsonify({'error': 'No data provided'}), 400
    
    if 'name' not in data:
        return jsonify({'error': 'Name is required'}), 400
    
    if 'email' not in data:
        return jsonify({'error': 'Email is required'}), 400
    
    # Create user
    new_user = {
        'id': max(u['id'] for u in users) + 1 if users else 1,
        'name': data['name'],
        'email': data['email']
    }
    
    users.append(new_user)
    
    return jsonify(new_user), 201

# PUT - Update user
@app.route('/api/users/<int:user_id>', methods=['PUT'])
def update_user(user_id):
    """Update user."""
    user = next((u for u in users if u['id'] == user_id), None)
    
    if not user:
        return jsonify({'error': 'User not found'}), 404
    
    data = request.get_json()
    
    # Update fields
    if 'name' in data:
        user['name'] = data['name']
    if 'email' in data:
        user['email'] = data['email']
    
    return jsonify(user)

# DELETE - Delete user
@app.route('/api/users/<int:user_id>', methods=['DELETE'])
def delete_user(user_id):
    """Delete user."""
    global users
    
    user = next((u for u in users if u['id'] == user_id), None)
    
    if not user:
        return jsonify({'error': 'User not found'}), 404
    
    users = [u for u in users if u['id'] != user_id]
    
    return jsonify({'message': 'User deleted successfully'})

if __name__ == '__main__':
    app.run(debug=True)
```

### Flask Template with AJAX

```html
<!-- templates/index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Flask AJAX Example</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 50px auto;
            padding: 20px;
        }
        .user-card {
            border: 1px solid #ddd;
            padding: 15px;
            margin: 10px 0;
            border-radius: 5px;
        }
        .loading {
            color: #666;
            font-style: italic;
        }
        .error {
            color: #d00;
            padding: 10px;
            background: #fee;
            border-radius: 3px;
        }
        button {
            padding: 10px 20px;
            margin: 5px;
            cursor: pointer;
        }
        input {
            padding: 8px;
            margin: 5px;
            width: 200px;
        }
    </style>
</head>
<body>
    <h1>User Management (AJAX)</h1>
    
    <!-- Search -->
    <div>
        <input type="text" id="searchInput" placeholder="Search users...">
        <button onclick="searchUsers()">Search</button>
        <button onclick="loadUsers()">Show All</button>
    </div>
    
    <!-- Create User Form -->
    <div style="margin: 20px 0; padding: 20px; border: 1px solid #ddd;">
        <h3>Add New User</h3>
        <input type="text" id="newName" placeholder="Name">
        <input type="email" id="newEmail" placeholder="Email">
        <button onclick="createUser()">Add User</button>
    </div>
    
    <!-- Users List -->
    <div id="usersContainer">
        <p class="loading">Loading users...</p>
    </div>
    
    <script>
        // Load users on page load
        document.addEventListener('DOMContentLoaded', function() {
            loadUsers();
        });
        
        // Load all users
        function loadUsers() {
            const container = document.getElementById('usersContainer');
            container.innerHTML = '<p class="loading">Loading users...</p>';
            
            fetch('/api/users')
                .then(response => {
                    if (!response.ok) {
                        throw new Error('Network response was not ok');
                    }
                    return response.json();
                })
                .then(users => {
                    displayUsers(users);
                })
                .catch(error => {
                    container.innerHTML = `<p class="error">Error: ${error.message}</p>`;
                });
        }
        
        // Search users
        function searchUsers() {
            const search = document.getElementById('searchInput').value;
            const container = document.getElementById('usersContainer');
            container.innerHTML = '<p class="loading">Searching...</p>';
            
            fetch(`/api/users?search=${encodeURIComponent(search)}`)
                .then(response => response.json())
                .then(users => {
                    displayUsers(users);
                })
                .catch(error => {
                    container.innerHTML = `<p class="error">Error: ${error.message}</p>`;
                });
        }
        
        // Display users
        function displayUsers(users) {
            const container = document.getElementById('usersContainer');
            
            if (users.length === 0) {
                container.innerHTML = '<p>No users found.</p>';
                return;
            }
            
            let html = '';
            users.forEach(user => {
                html += `
                    <div class="user-card">
                        <h3>${user.name}</h3>
                        <p>Email: ${user.email}</p>
                        <button onclick="editUser(${user.id})">Edit</button>
                        <button onclick="deleteUser(${user.id})">Delete</button>
                    </div>
                `;
            });
            
            container.innerHTML = html;
        }
        
        // Create user
        function createUser() {
            const name = document.getElementById('newName').value;
            const email = document.getElementById('newEmail').value;
            
            if (!name || !email) {
                alert('Please fill in all fields');
                return;
            }
            
            fetch('/api/users', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify({ name, email })
            })
                .then(response => {
                    if (!response.ok) {
                        return response.json().then(data => {
                            throw new Error(data.error);
                        });
                    }
                    return response.json();
                })
                .then(user => {
                    alert(`User ${user.name} created successfully!`);
                    document.getElementById('newName').value = '';
                    document.getElementById('newEmail').value = '';
                    loadUsers();
                })
                .catch(error => {
                    alert(`Error: ${error.message}`);
                });
        }
        
        // Edit user (simplified)
        function editUser(userId) {
            const newName = prompt('Enter new name:');
            if (!newName) return;
            
            fetch(`/api/users/${userId}`, {
                method: 'PUT',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify({ name: newName })
            })
                .then(response => response.json())
                .then(user => {
                    alert(`User updated: ${user.name}`);
                    loadUsers();
                })
                .catch(error => {
                    alert(`Error: ${error.message}`);
                });
        }
        
        // Delete user
        function deleteUser(userId) {
            if (!confirm('Are you sure you want to delete this user?')) {
                return;
            }
            
            fetch(`/api/users/${userId}`, {
                method: 'DELETE'
            })
                .then(response => response.json())
                .then(data => {
                    alert(data.message);
                    loadUsers();
                })
                .catch(error => {
                    alert(`Error: ${error.message}`);
                });
        }
    </script>
</body>
</html>
```

### Flask Autocomplete Example

```python
# Autocomplete endpoint
@app.route('/api/autocomplete')
def autocomplete():
    """Autocomplete search."""
    query = request.args.get('q', '').lower()
    
    if not query or len(query) < 2:
        return jsonify([])
    
    # Mock data - in production, query database
    suggestions = [
        'Python', 'JavaScript', 'Java', 'C++', 'Ruby',
        'PHP', 'Swift', 'Kotlin', 'Go', 'Rust'
    ]
    
    # Filter suggestions
    results = [s for s in suggestions if query in s.lower()]
    
    return jsonify(results[:5])  # Return top 5 matches
```

```html
<!-- Autocomplete input -->
<div>
    <input type="text" id="autocompleteInput" placeholder="Search...">
    <div id="suggestions"></div>
</div>

<script>
let debounceTimer;

document.getElementById('autocompleteInput').addEventListener('input', function(e) {
    const query = e.target.value;
    
    // Debounce - wait 300ms after user stops typing
    clearTimeout(debounceTimer);
    
    if (query.length < 2) {
        document.getElementById('suggestions').innerHTML = '';
        return;
    }
    
    debounceTimer = setTimeout(() => {
        fetch(`/api/autocomplete?q=${encodeURIComponent(query)}`)
            .then(response => response.json())
            .then(suggestions => {
                const suggestionsDiv = document.getElementById('suggestions');
                
                if (suggestions.length === 0) {
                    suggestionsDiv.innerHTML = '';
                    return;
                }
                
                let html = '<ul>';
                suggestions.forEach(suggestion => {
                    html += `<li onclick="selectSuggestion('${suggestion}')">${suggestion}</li>`;
                });
                html += '</ul>';
                
                suggestionsDiv.innerHTML = html;
            });
    }, 300);
});

function selectSuggestion(value) {
    document.getElementById('autocompleteInput').value = value;
    document.getElementById('suggestions').innerHTML = '';
}
</script>
```

---

## AJAX with FastAPI

### Basic FastAPI AJAX Example

```python
from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import HTMLResponse, JSONResponse
from fastapi.templating import Jinja2Templates
from pydantic import BaseModel, EmailStr
from typing import List, Optional
from datetime import datetime

app = FastAPI()
templates = Jinja2Templates(directory="templates")

# Pydantic models
class User(BaseModel):
    id: int
    name: str
    email: EmailStr
    created_at: Optional[datetime] = None

class UserCreate(BaseModel):
    name: str
    email: EmailStr

class UserUpdate(BaseModel):
    name: Optional[str] = None
    email: Optional[EmailStr] = None

# Mock database
users_db = [
    User(id=1, name='Alice', email='alice@example.com', created_at=datetime.utcnow()),
    User(id=2, name='Bob', email='bob@example.com', created_at=datetime.utcnow()),
]

@app.get("/", response_class=HTMLResponse)
async def index(request: Request):
    """Main page."""
    return templates.TemplateResponse("index.html", {"request": request})

# GET - Retrieve all users
@app.get("/api/users", response_model=List[User])
async def get_users(search: Optional[str] = None):
    """Get all users with optional search."""
    if search:
        return [u for u in users_db if search.lower() in u.name.lower()]
    return users_db

# GET - Retrieve single user
@app.get("/api/users/{user_id}", response_model=User)
async def get_user(user_id: int):
    """Get user by ID."""
    user = next((u for u in users_db if u.id == user_id), None)
    
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    
    return user

# POST - Create user
@app.post("/api/users", response_model=User, status_code=201)
async def create_user(user: UserCreate):
    """Create new user."""
    # Check if email already exists
    if any(u.email == user.email for u in users_db):
        raise HTTPException(status_code=400, detail="Email already exists")
    
    new_user = User(
        id=max(u.id for u in users_db) + 1 if users_db else 1,
        name=user.name,
        email=user.email,
        created_at=datetime.utcnow()
    )
    
    users_db.append(new_user)
    return new_user

# PUT - Update user
@app.put("/api/users/{user_id}", response_model=User)
async def update_user(user_id: int, user_update: UserUpdate):
    """Update user."""
    user = next((u for u in users_db if u.id == user_id), None)
    
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    
    # Update fields
    if user_update.name is not None:
        user.name = user_update.name
    if user_update.email is not None:
        user.email = user_update.email
    
    return user

# DELETE - Delete user
@app.delete("/api/users/{user_id}")
async def delete_user(user_id: int):
    """Delete user."""
    global users_db
    
    user = next((u for u in users_db if u.id == user_id), None)
    
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    
    users_db = [u for u in users_db if u.id != user_id]
    
    return {"message": "User deleted successfully"}

# Streaming response example
from fastapi.responses import StreamingResponse
import asyncio

@app.get("/api/stream")
async def stream_data():
    """Stream data to client."""
    async def generate():
        for i in range(10):
            yield f"data: Event {i}\n\n"
            await asyncio.sleep(1)
    
    return StreamingResponse(generate(), media_type="text/event-stream")

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### FastAPI with Async Database Operations

```python
from fastapi import FastAPI, Depends
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy import select

# Async database setup
DATABASE_URL = "sqlite+aiosqlite:///./test.db"

engine = create_async_engine(DATABASE_URL, echo=True)
async_session = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_db():
    """Dependency to get database session."""
    async with async_session() as session:
        yield session

@app.get("/api/users")
async def get_users_async(db: AsyncSession = Depends(get_db)):
    """Get users with async database query."""
    result = await db.execute(select(User))
    users = result.scalars().all()
    return users

@app.post("/api/users")
async def create_user_async(
    user: UserCreate,
    db: AsyncSession = Depends(get_db)
):
    """Create user with async database operation."""
    new_user = User(name=user.name, email=user.email)
    db.add(new_user)
    await db.commit()
    await db.refresh(new_user)
    return new_user
```

---

## AJAX with Django

### Django AJAX Views

```python
# views.py
from django.http import JsonResponse
from django.views.decorators.http import require_http_methods
from django.views.decorators.csrf import csrf_exempt
from django.shortcuts import render
import json

def index(request):
    """Main page."""
    return render(request, 'index.html')

# GET - Retrieve all users
def get_users(request):
    """Get all users."""
    from .models import User
    
    search = request.GET.get('search', '')
    
    users = User.objects.all()
    
    if search:
        users = users.filter(name__icontains=search)
    
    data = [
        {
            'id': user.id,
            'name': user.name,
            'email': user.email
        }
        for user in users
    ]
    
    return JsonResponse(data, safe=False)

# GET - Retrieve single user
def get_user(request, user_id):
    """Get user by ID."""
    from .models import User
    from django.shortcuts import get_object_or_404
    
    user = get_object_or_404(User, id=user_id)
    
    data = {
        'id': user.id,
        'name': user.name,
        'email': user.email
    }
    
    return JsonResponse(data)

# POST - Create user
@require_http_methods(["POST"])
def create_user(request):
    """Create new user."""
    from .models import User
    
    try:
        data = json.loads(request.body)
        
        # Validate
        if 'name' not in data or 'email' not in data:
            return JsonResponse(
                {'error': 'Name and email are required'},
                status=400
            )
        
        # Create user
        user = User.objects.create(
            name=data['name'],
            email=data['email']
        )
        
        return JsonResponse({
            'id': user.id,
            'name': user.name,
            'email': user.email
        }, status=201)
        
    except json.JSONDecodeError:
        return JsonResponse({'error': 'Invalid JSON'}, status=400)
    except Exception as e:
        return JsonResponse({'error': str(e)}, status=500)

# PUT - Update user
@require_http_methods(["PUT"])
def update_user(request, user_id):
    """Update user."""
    from .models import User
    from django.shortcuts import get_object_or_404
    
    try:
        user = get_object_or_404(User, id=user_id)
        data = json.loads(request.body)
        
        # Update fields
        if 'name' in data:
            user.name = data['name']
        if 'email' in data:
            user.email = data['email']
        
        user.save()
        
        return JsonResponse({
            'id': user.id,
            'name': user.name,
            'email': user.email
        })
        
    except json.JSONDecodeError:
        return JsonResponse({'error': 'Invalid JSON'}, status=400)

# DELETE - Delete user
@require_http_methods(["DELETE"])
def delete_user(request, user_id):
    """Delete user."""
    from .models import User
    from django.shortcuts import get_object_or_404
    
    user = get_object_or_404(User, id=user_id)
    user.delete()
    
    return JsonResponse({'message': 'User deleted successfully'})

# urls.py
from django.urls import path
from . import views

urlpatterns = [
    path('', views.index, name='index'),
    path('api/users/', views.get_users, name='get_users'),
    path('api/users/create/', views.create_user, name='create_user'),
    path('api/users/<int:user_id>/', views.get_user, name='get_user'),
    path('api/users/<int:user_id>/update/', views.update_user, name='update_user'),
    path('api/users/<int:user_id>/delete/', views.delete_user, name='delete_user'),
]
```

### Django Class-Based Views for AJAX

```python
# views.py
from django.http import JsonResponse
from django.views import View
from django.utils.decorators import method_decorator
from django.views.decorators.csrf import csrf_exempt
import json

@method_decorator(csrf_exempt, name='dispatch')
class UserAPIView(View):
    """Class-based view for user API."""
    
    def get(self, request, user_id=None):
        """Get user(s)."""
        from .models import User
        
        if user_id:
            # Get single user
            try:
                user = User.objects.get(id=user_id)
                return JsonResponse({
                    'id': user.id,
                    'name': user.name,
                    'email': user.email
                })
            except User.DoesNotExist:
                return JsonResponse({'error': 'User not found'}, status=404)
        else:
            # Get all users
            users = User.objects.all()
            data = [
                {'id': u.id, 'name': u.name, 'email': u.email}
                for u in users
            ]
            return JsonResponse(data, safe=False)
    
    def post(self, request):
        """Create user."""
        from .models import User
        
        try:
            data = json.loads(request.body)
            
            user = User.objects.create(
                name=data['name'],
                email=data['email']
            )
            
            return JsonResponse({
                'id': user.id,
                'name': user.name,
                'email': user.email
            }, status=201)
            
        except (KeyError, json.JSONDecodeError):
            return JsonResponse({'error': 'Invalid data'}, status=400)
    
    def put(self, request, user_id):
        """Update user."""
        from .models import User
        
        try:
            user = User.objects.get(id=user_id)
            data = json.loads(request.body)
            
            if 'name' in data:
                user.name = data['name']
            if 'email' in data:
                user.email = data['email']
            
            user.save()
            
            return JsonResponse({
                'id': user.id,
                'name': user.name,
                'email': user.email
            })
            
        except User.DoesNotExist:
            return JsonResponse({'error': 'User not found'}, status=404)
        except json.JSONDecodeError:
            return JsonResponse({'error': 'Invalid JSON'}, status=400)
    
    def delete(self, request, user_id):
        """Delete user."""
        from .models import User
        
        try:
            user = User.objects.get(id=user_id)
            user.delete()
            return JsonResponse({'message': 'User deleted'})
        except User.DoesNotExist:
            return JsonResponse({'error': 'User not found'}, status=404)
```

---

## Frontend AJAX Implementation

### Using Fetch API (Modern)

```javascript
// GET request
async function getUsers() {
    try {
        const response = await fetch('/api/users');
        
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        
        const users = await response.json();
        console.log(users);
        return users;
        
    } catch (error) {
        console.error('Error fetching users:', error);
        throw error;
    }
}

// POST request
async function createUser(userData) {
    try {
        const response = await fetch('/api/users', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                // Include CSRF token for Django
                'X-CSRFToken': getCookie('csrftoken')
            },
            body: JSON.stringify(userData)
        });
        
        if (!response.ok) {
            const error = await response.json();
            throw new Error(error.message || 'Failed to create user');
        }
        
        const newUser = await response.json();
        return newUser;
        
    } catch (error) {
        console.error('Error creating user:', error);
        throw error;
    }
}

// PUT request
async function updateUser(userId, updates) {
    const response = await fetch(`/api/users/${userId}`, {
        method: 'PUT',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify(updates)
    });
    
    return await response.json();
}

// DELETE request
async function deleteUser(userId) {
    const response = await fetch(`/api/users/${userId}`, {
        method: 'DELETE'
    });
    
    return await response.json();
}

// Get CSRF token (for Django)
function getCookie(name) {
    let cookieValue = null;
    if (document.cookie && document.cookie !== '') {
        const cookies = document.cookie.split(';');
        for (let i = 0; i < cookies.length; i++) {
            const cookie = cookies[i].trim();
            if (cookie.substring(0, name.length + 1) === (name + '=')) {
                cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
                break;
            }
        }
    }
    return cookieValue;
}
```

### Using Axios (Popular Library)

```html
<!-- Include Axios -->
<script src="https://cdn.jsdelivr.net/npm/axios/dist/axios.min.js"></script>

<script>
// Configure Axios defaults
axios.defaults.headers.common['X-Requested-With'] = 'XMLHttpRequest';
axios.defaults.headers.post['Content-Type'] = 'application/json';

// Add CSRF token for Django
const csrftoken = getCookie('csrftoken');
if (csrftoken) {
    axios.defaults.headers.common['X-CSRFToken'] = csrftoken;
}

// GET request
async function getUsers() {
    try {
        const response = await axios.get('/api/users');
        console.log(response.data);
        return response.data;
    } catch (error) {
        console.error('Error:', error.response.data);
        throw error;
    }
}

// POST request
async function createUser(userData) {
    try {
        const response = await axios.post('/api/users', userData);
        return response.data;
    } catch (error) {
        if (error.response) {
            // Server responded with error
            console.error('Error:', error.response.data);
        } else if (error.request) {
            // Request made but no response
            console.error('No response received');
        } else {
            // Other error
            console.error('Error:', error.message);
        }
        throw error;
    }
}

// PUT request
async function updateUser(userId, updates) {
    const response = await axios.put(`/api/users/${userId}`, updates);
    return response.data;
}

// DELETE request
async function deleteUser(userId) {
    const response = await axios.delete(`/api/users/${userId}`);
    return response.data;
}

// Interceptors for global error handling
axios.interceptors.response.use(
    response => response,
    error => {
        if (error.response.status === 401) {
            // Redirect to login
            window.location.href = '/login';
        }
        return Promise.reject(error);
    }
);
</script>
```

### jQuery AJAX (Legacy)

```html
<!-- Include jQuery -->
<script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>

<script>
// Setup CSRF token for Django
$.ajaxSetup({
    headers: {
        'X-CSRFToken': getCookie('csrftoken')
    }
});

// GET request
function getUsers() {
    $.ajax({
        url: '/api/users',
        type: 'GET',
        dataType: 'json',
        success: function(users) {
            console.log(users);
        },
        error: function(xhr, status, error) {
            console.error('Error:', error);
        }
    });
}

// POST request
function createUser(userData) {
    $.ajax({
        url: '/api/users',
        type: 'POST',
        contentType: 'application/json',
        data: JSON.stringify(userData),
        success: function(newUser) {
            console.log('User created:', newUser);
        },
        error: function(xhr, status, error) {
            console.error('Error:', error);
        }
    });
}

// Shorthand methods
$.get('/api/users', function(users) {
    console.log(users);
});

$.post('/api/users', userData, function(newUser) {
    console.log(newUser);
});
</script>
```

---

## Real-Time Updates with WebSockets

### Flask-SocketIO

```python
from flask import Flask, render_template
from flask_socketio import SocketIO, emit, join_room, leave_room
import time

app = Flask(__name__)
app.config['SECRET_KEY'] = 'secret!'
socketio = SocketIO(app, cors_allowed_origins="*")

# Connected users
users = {}

@app.route('/')
def index():
    return render_template('chat.html')

@socketio.on('connect')
def handle_connect():
    """Handle client connection."""
    print(f'Client connected: {request.sid}')
    emit('message', {'data': 'Connected to server'})

@socketio.on('disconnect')
def handle_disconnect():
    """Handle client disconnection."""
    print(f'Client disconnected: {request.sid}')
    
    # Remove from users
    if request.sid in users:
        username = users[request.sid]
        del users[request.sid]
        
        # Notify others
        emit('user_left', {
            'username': username,
            'user_count': len(users)
        }, broadcast=True)

@socketio.on('join')
def handle_join(data):
    """Handle user joining chat."""
    username = data['username']
    users[request.sid] = username
    
    emit('user_joined', {
        'username': username,
        'user_count': len(users)
    }, broadcast=True)

@socketio.on('message')
def handle_message(data):
    """Handle chat message."""
    username = users.get(request.sid, 'Anonymous')
    
    emit('message', {
        'username': username,
        'message': data['message'],
        'timestamp': time.time()
    }, broadcast=True)

@socketio.on('join_room')
def handle_join_room(data):
    """Handle joining a specific room."""
    room = data['room']
    join_room(room)
    
    emit('message', {
        'message': f'You joined room: {room}'
    }, room=room)

@socketio.on('leave_room')
def handle_leave_room(data):
    """Handle leaving a room."""
    room = data['room']
    leave_room(room)
    
    emit('message', {
        'message': f'You left room: {room}'
    }, room=room)

if __name__ == '__main__':
    socketio.run(app, debug=True)
```

```html
<!-- templates/chat.html -->
<!DOCTYPE html>
<html>
<head>
    <title>WebSocket Chat</title>
    <script src="https://cdn.socket.io/4.5.4/socket.io.min.js"></script>
</head>
<body>
    <h1>Real-Time Chat</h1>
    
    <div id="status">Disconnected</div>
    <div id="userCount">Users: 0</div>
    
    <div id="messages" style="height: 400px; overflow-y: scroll; border: 1px solid #ccc; padding: 10px;">
    </div>
    
    <div>
        <input type="text" id="username" placeholder="Username">
        <button onclick="joinChat()">Join</button>
    </div>
    
    <div>
        <input type="text" id="messageInput" placeholder="Type message...">
        <button onclick="sendMessage()">Send</button>
    </div>
    
    <script>
        // Connect to WebSocket server
        const socket = io.connect('http://localhost:5000');
        
        // Connection events
        socket.on('connect', function() {
            document.getElementById('status').textContent = 'Connected';
            console.log('Connected to server');
        });
        
        socket.on('disconnect', function() {
            document.getElementById('status').textContent = 'Disconnected';
            console.log('Disconnected from server');
        });
        
        // Receive messages
        socket.on('message', function(data) {
            const messagesDiv = document.getElementById('messages');
            const messageElement = document.createElement('div');
            
            if (data.username) {
                messageElement.textContent = `${data.username}: ${data.message}`;
            } else {
                messageElement.textContent = data.data || data.message;
            }
            
            messagesDiv.appendChild(messageElement);
            messagesDiv.scrollTop = messagesDiv.scrollHeight;
        });
        
        // User joined
        socket.on('user_joined', function(data) {
            document.getElementById('userCount').textContent = `Users: ${data.user_count}`;
            
            const messagesDiv = document.getElementById('messages');
            const messageElement = document.createElement('div');
            messageElement.textContent = `${data.username} joined the chat`;
            messageElement.style.color = 'green';
            messagesDiv.appendChild(messageElement);
        });
        
        // User left
        socket.on('user_left', function(data) {
            document.getElementById('userCount').textContent = `Users: ${data.user_count}`;
            
            const messagesDiv = document.getElementById('messages');
            const messageElement = document.createElement('div');
            messageElement.textContent = `${data.username} left the chat`;
            messageElement.style.color = 'red';
            messagesDiv.appendChild(messageElement);
        });
        
        // Join chat
        function joinChat() {
            const username = document.getElementById('username').value;
            if (username) {
                socket.emit('join', { username: username });
            }
        }
        
        // Send message
        function sendMessage() {
            const message = document.getElementById('messageInput').value;
            if (message) {
                socket.emit('message', { message: message });
                document.getElementById('messageInput').value = '';
            }
        }
        
        // Send on Enter key
        document.getElementById('messageInput').addEventListener('keypress', function(e) {
            if (e.key === 'Enter') {
                sendMessage();
            }
        });
    </script>
</body>
</html>
```

### FastAPI WebSockets

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from fastapi.responses import HTMLResponse
from typing import List
import json

app = FastAPI()

# Connection manager
class ConnectionManager:
    def __init__(self):
        self.active_connections: List[WebSocket] = []
    
    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)
    
    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)
    
    async def send_personal_message(self, message: str, websocket: WebSocket):
        await websocket.send_text(message)
    
    async def broadcast(self, message: str):
        for connection in self.active_connections:
            await connection.send_text(message)

manager = ConnectionManager()

@app.get("/")
async def get():
    return HTMLResponse(html)

@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: int):
    await manager.connect(websocket)
    
    try:
        # Send welcome message
        await manager.send_personal_message(
            f"Welcome! You are client #{client_id}",
            websocket
        )
        
        # Notify others
        await manager.broadcast(f"Client #{client_id} joined the chat")
        
        # Listen for messages
        while True:
            data = await websocket.receive_text()
            
            # Broadcast message
            await manager.broadcast(f"Client #{client_id}: {data}")
            
    except WebSocketDisconnect:
        manager.disconnect(websocket)
        await manager.broadcast(f"Client #{client_id} left the chat")

html = """
<!DOCTYPE html>
<html>
<head>
    <title>FastAPI WebSocket</title>
</head>
<body>
    <h1>WebSocket Chat</h1>
    <div id="messages"></div>
    <input type="text" id="messageInput" autocomplete="off">
    <button onclick="sendMessage()">Send</button>
    
    <script>
        const clientId = Date.now();
        const ws = new WebSocket(`ws://localhost:8000/ws/${clientId}`);
        
        ws.onmessage = function(event) {
            const messagesDiv = document.getElementById('messages');
            const message = document.createElement('div');
            message.textContent = event.data;
            messagesDiv.appendChild(message);
        };
        
        function sendMessage() {
            const input = document.getElementById('messageInput');
            ws.send(input.value);
            input.value = '';
        }
        
        document.getElementById('messageInput').addEventListener('keypress', function(e) {
            if (e.key === 'Enter') sendMessage();
        });
    </script>
</body>
</html>
"""
```

---

## Server-Sent Events (SSE)

### Flask SSE Example

```python
from flask import Flask, Response, render_template
import time
import json

app = Flask(__name__)

@app.route('/')
def index():
    return render_template('sse.html')

@app.route('/stream')
def stream():
    """Server-Sent Events stream."""
    def generate():
        count = 0
        while True:
            # Send event
            data = {
                'count': count,
                'timestamp': time.time()
            }
            
            yield f"data: {json.dumps(data)}\n\n"
            
            count += 1
            time.sleep(1)
    
    return Response(generate(), mimetype='text/event-stream')

@app.route('/notifications')
def notifications():
    """Send periodic notifications."""
    def generate():
        notifications = [
            "New message from Alice",
            "Bob liked your post",
            "You have a new follower",
            "Charlie commented on your photo"
        ]
        
        for notification in notifications:
            yield f"data: {notification}\n\n"
            time.sleep(2)
        
        # Send close event
        yield "event: close\ndata: Stream ended\n\n"
    
    return Response(generate(), mimetype='text/event-stream')
```

```html
<!-- templates/sse.html -->
<!DOCTYPE html>
<html>
<head>
    <title>Server-Sent Events</title>
</head>
<body>
    <h1>Server-Sent Events Demo</h1>
    
    <div>
        <h2>Live Counter</h2>
        <div id="counter">Waiting for data...</div>
    </div>
    
    <div>
        <h2>Notifications</h2>
        <div id="notifications"></div>
        <button onclick="startNotifications()">Start Notifications</button>
    </div>
    
    <script>
        // Live counter
        const counterSource = new EventSource('/stream');
        
        counterSource.onmessage = function(event) {
            const data = JSON.parse(event.data);
            document.getElementById('counter').textContent = 
                `Count: ${data.count} (${new Date(data.timestamp * 1000).toLocaleTimeString()})`;
        };
        
        counterSource.onerror = function(error) {
            console.error('EventSource error:', error);
            counterSource.close();
        };
        
        // Notifications
        let notificationSource;
        
        function startNotifications() {
            if (notificationSource) {
                notificationSource.close();
            }
            
            document.getElementById('notifications').innerHTML = '';
            
            notificationSource = new EventSource('/notifications');
            
            notificationSource.onmessage = function(event) {
                const notificationsDiv = document.getElementById('notifications');
                const notification = document.createElement('div');
                notification.textContent = event.data;
                notification.style.padding = '10px';
                notification.style.margin = '5px 0';
                notification.style.background = '#e3f2fd';
                notification.style.borderRadius = '5px';
                notificationsDiv.appendChild(notification);
            };
            
            notificationSource.addEventListener('close', function(event) {
                console.log('Stream closed:', event.data);
                notificationSource.close();
            });
        }
    </script>
</body>
</html>
```

### FastAPI SSE Example

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import asyncio
import json

app = FastAPI()

@app.get("/stream")
async def stream_data():
    """Stream data using Server-Sent Events."""
    async def generate():
        count = 0
        while True:
            data = {
                'count': count,
                'timestamp': time.time()
            }
            
            yield f"data: {json.dumps(data)}\n\n"
            
            count += 1
            await asyncio.sleep(1)
    
    return StreamingResponse(generate(), media_type="text/event-stream")
```

---

## Long Polling

### Flask Long Polling Example

```python
from flask import Flask, jsonify, request
import time
import threading

app = Flask(__name__)

# Shared state
messages = []
message_lock = threading.Lock()

@app.route('/api/poll')
def poll_messages():
    """Long polling endpoint."""
    last_id = request.args.get('last_id', 0, type=int)
    timeout = 30  # 30 seconds timeout
    start_time = time.time()
    
    while time.time() - start_time < timeout:
        with message_lock:
            # Check for new messages
            new_messages = [
                msg for msg in messages 
                if msg['id'] > last_id
            ]
            
            if new_messages:
                return jsonify(new_messages)
        
        # Wait before checking again
        time.sleep(0.5)
    
    # Timeout - return empty array
    return jsonify([])

@app.route('/api/send', methods=['POST'])
def send_message():
    """Send a new message."""
    data = request.get_json()
    
    with message_lock:
        message = {
            'id': len(messages) + 1,
            'text': data['text'],
            'timestamp': time.time()
        }
        messages.append(message)
    
    return jsonify(message), 201
```

```javascript
// Long polling client
let lastMessageId = 0;

async function pollMessages() {
    try {
        const response = await fetch(`/api/poll?last_id=${lastMessageId}`);
        const newMessages = await response.json();
        
        if (newMessages.length > 0) {
            // Update UI with new messages
            newMessages.forEach(msg => {
                displayMessage(msg);
                lastMessageId = Math.max(lastMessageId, msg.id);
            });
        }
        
        // Poll again
        pollMessages();
        
    } catch (error) {
        console.error('Polling error:', error);
        // Retry after delay
        setTimeout(pollMessages, 5000);
    }
}

// Start polling
pollMessages();
```

---

## Error Handling

### Frontend Error Handling

```javascript
// Comprehensive error handling
async function fetchWithErrorHandling(url, options = {}) {
    try {
        const response = await fetch(url, options);
        
        // Check if response is OK
        if (!response.ok) {
            // Try to parse error message
            let errorMessage = `HTTP error! status: ${response.status}`;
            
            try {
                const errorData = await response.json();
                errorMessage = errorData.error || errorData.message || errorMessage;
            } catch (e) {
                // Response is not JSON
            }
            
            throw new Error(errorMessage);
        }
        
        // Parse JSON response
        const data = await response.json();
        return data;
        
    } catch (error) {
        // Network error or other error
        console.error('Fetch error:', error);
        
        // Display user-friendly error
        if (error.message.includes('Failed to fetch')) {
            showError('Network error. Please check your connection.');
        } else {
            showError(error.message);
        }
        
        throw error;
    }
}

// Usage
try {
    const users = await fetchWithErrorHandling('/api/users');
    displayUsers(users);
} catch (error) {
    // Error already handled and displayed
}

// Global error handler
window.addEventListener('unhandledrejection', function(event) {
    console.error('Unhandled promise rejection:', event.reason);
    showError('An unexpected error occurred. Please try again.');
});

function showError(message) {
    const errorDiv = document.getElementById('error');
    errorDiv.textContent = message;
    errorDiv.style.display = 'block';
    
    // Hide after 5 seconds
    setTimeout(() => {
        errorDiv.style.display = 'none';
    }, 5000);
}
```

### Backend Error Handling

```python
from flask import Flask, jsonify
from functools import wraps

app = Flask(__name__)

class APIError(Exception):
    """Custom API exception."""
    def __init__(self, message, status_code=400, payload=None):
        super().__init__()
        self.message = message
        self.status_code = status_code
        self.payload = payload
    
    def to_dict(self):
        rv = dict(self.payload or ())
        rv['error'] = self.message
        return rv

@app.errorhandler(APIError)
def handle_api_error(error):
    """Handle API errors."""
    response = jsonify(error.to_dict())
    response.status_code = error.status_code
    return response

@app.errorhandler(404)
def not_found(error):
    """Handle 404 errors."""
    return jsonify({'error': 'Resource not found'}), 404

@app.errorhandler(500)
def internal_error(error):
    """Handle 500 errors."""
    # Log error
    app.logger.error(f'Internal error: {error}')
    
    # Don't expose internal error details in production
    if app.debug:
        return jsonify({'error': str(error)}), 500
    else:
        return jsonify({'error': 'Internal server error'}), 500

def validate_json(f):
    """Decorator to validate JSON input."""
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if not request.is_json:
            raise APIError('Content-Type must be application/json', 400)
        
        data = request.get_json()
        if not data:
            raise APIError('Empty JSON body', 400)
        
        return f(*args, **kwargs)
    
    return decorated_function

@app.route('/api/users', methods=['POST'])
@validate_json
def create_user():
    """Create user with validation."""
    data = request.get_json()
    
    # Validate required fields
    if 'name' not in data:
        raise APIError('Name is required', 422)
    
    if 'email' not in data:
        raise APIError('Email is required', 422)
    
    # Create user
    # ...
    
    return jsonify({'message': 'User created'}), 201
```

---

## Security Considerations

### CSRF Protection for AJAX

```python
# Flask with CSRF
from flask_wtf.csrf import CSRFProtect

app = Flask(__name__)
csrf = CSRFProtect(app)

# Exempt API endpoints if using token authentication
@app.route('/api/users', methods=['POST'])
@csrf.exempt
def api_create_user():
    # Verify API token instead
    pass
```

```javascript
// Include CSRF token in AJAX requests
function getCookie(name) {
    let cookieValue = null;
    if (document.cookie && document.cookie !== '') {
        const cookies = document.cookie.split(';');
        for (let i = 0; i < cookies.length; i++) {
            const cookie = cookies[i].trim();
            if (cookie.substring(0, name.length + 1) === (name + '=')) {
                cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
                break;
            }
        }
    }
    return cookieValue;
}

const csrftoken = getCookie('csrftoken');

fetch('/api/users', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-CSRFToken': csrftoken
    },
    body: JSON.stringify(userData)
});
```

### Rate Limiting AJAX Endpoints

```python
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(
    app=app,
    key_func=get_remote_address,
    default_limits=["200 per day", "50 per hour"]
)

@app.route('/api/search')
@limiter.limit("10 per minute")
def search():
    """Rate-limited search endpoint."""
    query = request.args.get('q')
    # Perform search
    return jsonify(results)
```

### Input Validation

```python
from pydantic import BaseModel, EmailStr, validator

class UserCreate(BaseModel):
    name: str
    email: EmailStr
    
    @validator('name')
    def name_must_not_be_empty(cls, v):
        if not v.strip():
            raise ValueError('Name cannot be empty')
        return v.strip()

@app.post("/api/users")
async def create_user(user: UserCreate):
    # Pydantic automatically validates
    return {"message": "User created", "user": user}
```

---

## Performance Optimization

### Debouncing Requests

```javascript
// Debounce function
function debounce(func, wait) {
    let timeout;
    
    return function executedFunction(...args) {
        const later = () => {
            clearTimeout(timeout);
            func(...args);
        };
        
        clearTimeout(timeout);
        timeout = setTimeout(later, wait);
    };
}

// Usage for search
const searchInput = document.getElementById('search');

const debouncedSearch = debounce(function(query) {
    fetch(`/api/search?q=${encodeURIComponent(query)}`)
        .then(response => response.json())
        .then(results => displayResults(results));
}, 300);  // Wait 300ms after user stops typing

searchInput.addEventListener('input', function(e) {
    debouncedSearch(e.target.value);
});
```

### Request Caching

```javascript
// Simple cache implementation
class RequestCache {
    constructor(ttl = 60000) {  // 60 seconds default
        this.cache = new Map();
        this.ttl = ttl;
    }
    
    get(key) {
        const item = this.cache.get(key);
        
        if (!item) return null;
        
        // Check if expired
        if (Date.now() > item.expiry) {
            this.cache.delete(key);
            return null;
        }
        
        return item.data;
    }
    
    set(key, data) {
        this.cache.set(key, {
            data: data,
            expiry: Date.now() + this.ttl
        });
    }
    
    clear() {
        this.cache.clear();
    }
}

// Usage
const cache = new RequestCache(60000);  // 60 second cache

async function getUsers() {
    const cacheKey = 'users';
    
    // Check cache first
    const cached = cache.get(cacheKey);
    if (cached) {
        console.log('Returning cached data');
        return cached;
    }
    
    // Fetch from server
    const response = await fetch('/api/users');
    const users = await response.json();
    
    // Cache result
    cache.set(cacheKey, users);
    
    return users;
}
```

### Batch Requests

```python
# Backend: Batch endpoint
@app.route('/api/batch', methods=['POST'])
def batch_requests():
    """Handle multiple requests in one call."""
    data = request.get_json()
    requests = data.get('requests', [])
    
    results = []
    
    for req in requests:
        try:
            # Route to appropriate handler
            if req['endpoint'] == '/api/users':
                result = get_users()
            elif req['endpoint'].startswith('/api/users/'):
                user_id = int(req['endpoint'].split('/')[-1])
                result = get_user(user_id)
            else:
                result = {'error': 'Unknown endpoint'}
            
            results.append({
                'success': True,
                'data': result
            })
        except Exception as e:
            results.append({
                'success': False,
                'error': str(e)
            })
    
    return jsonify(results)
```

```javascript
// Frontend: Batch requests
async function batchFetch(requests) {
    const response = await fetch('/api/batch', {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({ requests })
    });
    
    return await response.json();
}

// Usage
const results = await batchFetch([
    { endpoint: '/api/users' },
    { endpoint: '/api/users/1' },
    { endpoint: '/api/users/2' }
]);
```

---

## Best Practices

### AJAX Best Practices Checklist

```javascript
"""
✅ AJAX Best Practices:

1. Error Handling
   ✅ Handle network errors
   ✅ Handle HTTP errors (4xx, 5xx)
   ✅ Display user-friendly error messages
   ✅ Implement retry logic for failed requests

2. User Experience
   ✅ Show loading indicators
   ✅ Disable buttons during requests
   ✅ Provide feedback on success/failure
   ✅ Implement optimistic UI updates

3. Performance
   ✅ Debounce rapid requests
   ✅ Cache responses when appropriate
   ✅ Batch requests when possible
   ✅ Use pagination for large datasets

4. Security
   ✅ Include CSRF tokens
   ✅ Validate input on both client and server
   ✅ Use HTTPS
   ✅ Implement rate limiting
   ✅ Sanitize output

5. Code Quality
   ✅ Use async/await for cleaner code
   ✅ Handle promises properly
   ✅ Implement proper error boundaries
   ✅ Use TypeScript for type safety (optional)

6. Accessibility
   ✅ Announce dynamic content to screen readers
   ✅ Maintain keyboard navigation
   ✅ Provide alternative non-JS fallbacks
"""
```

### Complete AJAX Example with Best Practices

```javascript
// Modern AJAX implementation with all best practices

class AJAXManager {
    constructor(baseURL = '') {
        this.baseURL = baseURL;
        this.cache = new Map();
        this.pendingRequests = new Map();
    }
    
    // Main request method
    async request(url, options = {}) {
        const cacheKey = `${options.method || 'GET'}:${url}`;
        
        // Check cache for GET requests
        if (!options.method || options.method === 'GET') {
            const cached = this.cache.get(cacheKey);
            if (cached && Date.now() < cached.expiry) {
                return cached.data;
            }
        }
        
        // Prevent duplicate requests
        if (this.pendingRequests.has(cacheKey)) {
            return this.pendingRequests.get(cacheKey);
        }
        
        // Show loading
        this.showLoading();
        
        // Make request
        const requestPromise = this._fetch(url, options);
        this.pendingRequests.set(cacheKey, requestPromise);
        
        try {
            const data = await requestPromise;
            
            // Cache GET requests
            if (!options.method || options.method === 'GET') {
                this.cache.set(cacheKey, {
                    data,
                    expiry: Date.now() + 60000  // 60 seconds
                });
            }
            
            return data;
            
        } finally {
            this.pendingRequests.delete(cacheKey);
            this.hideLoading();
        }
    }
    
    async _fetch(url, options) {
        const fullURL = this.baseURL + url;
        
        // Add default headers
        const headers = {
            'Content-Type': 'application/json',
            ...options.headers
        };
        
        // Add CSRF token
        const csrfToken = this.getCSRFToken();
        if (csrfToken) {
            headers['X-CSRFToken'] = csrfToken;
        }
        
        try {
            const response = await fetch(fullURL, {
                ...options,
                headers
            });
            
            if (!response.ok) {
                const error = await this.parseError(response);
                throw new Error(error);
            }
            
            return await response.json();
            
        } catch (error) {
            this.handleError(error);
            throw error;
        }
    }
    
    async parseError(response) {
        try {
            const data = await response.json();
            return data.error || data.message || `HTTP ${response.status}`;
        } catch {
            return `HTTP ${response.status}: ${response.statusText}`;
        }
    }
    
    handleError(error) {
        console.error('AJAX Error:', error);
        
        // Show user-friendly error
        const message = error.message.includes('fetch')
            ? 'Network error. Please check your connection.'
            : error.message;
        
        this.showError(message);
    }
    
    getCSRFToken() {
        const cookies = document.cookie.split(';');
        for (let cookie of cookies) {
            const [name, value] = cookie.trim().split('=');
            if (name === 'csrftoken') {
                return decodeURIComponent(value);
            }
        }
        return null;
    }
    
    showLoading() {
        const loader = document.getElementById('loader');
        if (loader) loader.style.display = 'block';
    }
    
    hideLoading() {
        const loader = document.getElementById('loader');
        if (loader) loader.style.display = 'none';
    }
    
    showError(message) {
        const errorDiv = document.getElementById('error');
        if (errorDiv) {
            errorDiv.textContent = message;
            errorDiv.style.display = 'block';
            setTimeout(() => errorDiv.style.display = 'none', 5000);
        } else {
            alert(message);
        }
    }
    
    // Convenience methods
    get(url, options = {}) {
        return this.request(url, { ...options, method: 'GET' });
    }
    
    post(url, data, options = {}) {
        return this.request(url, {
            ...options,
            method: 'POST',
            body: JSON.stringify(data)
        });
    }
    
    put(url, data, options = {}) {
        return this.request(url, {
            ...options,
            method: 'PUT',
            body: JSON.stringify(data)
        });
    }
    
    delete(url, options = {}) {
        return this.request(url, { ...options, method: 'DELETE' });
    }
}

// Usage
const ajax = new AJAXManager('/api');

// GET request
const users = await ajax.get('/users');

// POST request
const newUser = await ajax.post('/users', {
    name: 'Alice',
    email: 'alice@example.com'
});

// PUT request
const updated = await ajax.put('/users/1', {
    name: 'Alice Updated'
});

// DELETE request
await ajax.delete('/users/1');
```

---

## Summary

This comprehensive guide covered:

1. **AJAX Basics**: Understanding asynchronous requests and benefits
2. **JSON Responses**: Creating and formatting JSON in Python
3. **Flask AJAX**: Complete examples with templates
4. **FastAPI AJAX**: Modern async implementation
5. **Django AJAX**: Class-based views and function-based views
6. **Frontend Implementation**: Fetch API, Axios, jQuery
7. **WebSockets**: Real-time bidirectional communication
8. **Server-Sent Events**: One-way server-to-client streaming
9. **Long Polling**: Alternative real-time technique
10. **Error Handling**: Comprehensive client and server-side
11. **Security**: CSRF protection, rate limiting, validation
12. **Performance**: Debouncing, caching, batching
13. **Best Practices**: Complete checklist and production-ready code

Key Takeaways:
- Use AJAX for better user experience (no page reloads)
- Always handle errors gracefully
- Implement proper loading states
- Protect against security vulnerabilities (CSRF, XSS)
- Optimize performance with debouncing and caching
- Use modern Fetch API or Axios over jQuery
- Choose WebSockets for real-time, SSE for updates, AJAX for requests
- Validate input on both client and server
- Provide user feedback for all actions

AJAX enables building modern, interactive web applications!