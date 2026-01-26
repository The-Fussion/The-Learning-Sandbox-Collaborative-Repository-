# Routing and URL Handling with Python

## Table of Contents
1. [Introduction to Routing](#introduction-to-routing)
2. [URL Structure and Components](#url-structure-and-components)
3. [Flask Routing](#flask-routing)
4. [Django Routing](#django-routing)
5. [FastAPI Routing](#fastapi-routing)
6. [RESTful URL Design](#restful-url-design)
7. [Advanced Routing Patterns](#advanced-routing-patterns)
8. [URL Building and Reversing](#url-building-and-reversing)
9. [Best Practices](#best-practices)

---

## Introduction to Routing

**Routing** is the mechanism that maps URLs (Uniform Resource Locators) to specific functions or views in your web application. It's the bridge between what users type in their browser and the code that executes.

### What is a Route?

A route consists of:
- **URL Pattern**: The path the user requests (e.g., `/users/123`)
- **HTTP Method**: The type of request (GET, POST, PUT, DELETE, etc.)
- **Handler Function**: The function that processes the request
- **Route Name** (optional): An identifier for URL generation

### Why Routing Matters

1. **Clean URLs**: User-friendly and SEO-optimized
2. **Organization**: Logical structure for your application
3. **RESTful Design**: Standard API conventions
4. **Flexibility**: Dynamic content based on URL parameters
5. **Security**: Control access to different parts of your app

---

## URL Structure and Components

### Anatomy of a URL

```
https://example.com:8000/users/profile?page=2&sort=name#bio
\___/   \_________/ \__/ \____________/ \_______________/ \__/
  |          |       |         |                |           |
scheme     host     port     path            query       fragment
```

**Components:**
- **Scheme**: Protocol (http, https)
- **Host**: Domain name (example.com)
- **Port**: Server port (8000, optional)
- **Path**: Resource location (/users/profile)
- **Query String**: Parameters (?page=2&sort=name)
- **Fragment**: Page section (#bio)

### Path Parameters vs Query Parameters

**Path Parameters** (part of the URL path):
```
/users/123/posts/456
       ^^^       ^^^
    user_id   post_id
```

**Query Parameters** (after the `?`):
```
/search?q=python&page=2&limit=10
         ^^^^^^^^^^^^^^^^^^^^^^^^
         query parameters
```

**When to use which:**
- **Path Parameters**: For resource identification (required)
- **Query Parameters**: For filtering, sorting, pagination (optional)

---

## Flask Routing

### Basic Routes

```python
from flask import Flask

app = Flask(__name__)

# Simple route
@app.route('/')
def index():
    return 'Home Page'

# Route with static path
@app.route('/about')
def about():
    return 'About Page'

# Multiple routes to same function
@app.route('/info')
@app.route('/information')
def info():
    return 'Information Page'

if __name__ == '__main__':
    app.run(debug=True)
```

### HTTP Methods

```python
from flask import Flask, request

app = Flask(__name__)

# GET request (default)
@app.route('/items')
def get_items():
    return 'List of items'

# Specify methods explicitly
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form.get('username')
        return f'Logging in user: {username}'
    return 'Login form'

# Multiple methods
@app.route('/data', methods=['GET', 'POST', 'PUT', 'DELETE'])
def handle_data():
    if request.method == 'GET':
        return 'Getting data'
    elif request.method == 'POST':
        return 'Creating data'
    elif request.method == 'PUT':
        return 'Updating data'
    elif request.method == 'DELETE':
        return 'Deleting data'
```

### URL Parameters (Path Variables)

```python
from flask import Flask

app = Flask(__name__)

# String parameter (default)
@app.route('/user/<username>')
def show_user_profile(username):
    return f'User: {username}'

# Integer parameter
@app.route('/post/<int:post_id>')
def show_post(post_id):
    return f'Post ID: {post_id} (type: {type(post_id).__name__})'

# Float parameter
@app.route('/price/<float:price>')
def show_price(price):
    return f'Price: ${price:.2f}'

# Path parameter (accepts slashes)
@app.route('/path/<path:subpath>')
def show_subpath(subpath):
    return f'Subpath: {subpath}'

# UUID parameter
from uuid import UUID
@app.route('/item/<uuid:item_id>')
def show_item(item_id):
    return f'Item UUID: {item_id}'

# Multiple parameters
@app.route('/user/<username>/post/<int:post_id>')
def show_user_post(username, post_id):
    return f'Post {post_id} by {username}'

# Parameter with default value
@app.route('/page/')
@app.route('/page/<int:page_num>')
def show_page(page_num=1):
    return f'Page: {page_num}'
```

**Flask Converters:**
- `string`: Default, accepts any text without slashes
- `int`: Accepts integers
- `float`: Accepts floating point values
- `path`: Like string but accepts slashes
- `uuid`: Accepts UUID strings

### Query Parameters

```python
from flask import Flask, request

app = Flask(__name__)

@app.route('/search')
def search():
    # Get single parameter
    query = request.args.get('q', default='', type=str)
    
    # Get with default value
    page = request.args.get('page', default=1, type=int)
    limit = request.args.get('limit', default=10, type=int)
    
    # Get multiple values for same parameter
    tags = request.args.getlist('tag')
    
    # Get all parameters
    all_params = request.args.to_dict()
    
    return {
        'query': query,
        'page': page,
        'limit': limit,
        'tags': tags,
        'all_params': all_params
    }

# Example: /search?q=python&page=2&limit=20&tag=web&tag=backend
```

### Custom URL Converters

```python
from flask import Flask
from werkzeug.routing import BaseConverter

app = Flask(__name__)

# Custom converter for slug format
class SlugConverter(BaseConverter):
    regex = r'[a-z0-9]+(?:-[a-z0-9]+)*'

# Custom converter for year validation
class YearConverter(BaseConverter):
    regex = r'\d{4}'
    
    def to_python(self, value):
        return int(value)
    
    def to_url(self, value):
        return str(value)

# Register converters
app.url_map.converters['slug'] = SlugConverter
app.url_map.converters['year'] = YearConverter

# Use custom converters
@app.route('/article/<slug:article_slug>')
def show_article(article_slug):
    return f'Article slug: {article_slug}'

@app.route('/archive/<year:year>')
def show_archive(year):
    return f'Archive for year: {year}'

# Example: /article/my-first-post
# Example: /archive/2024
```

### Trailing Slashes

```python
from flask import Flask

app = Flask(__name__)

# With trailing slash - redirects /about to /about/
@app.route('/about/')
def about():
    return 'About Page'

# Without trailing slash - /projects/ returns 404
@app.route('/projects')
def projects():
    return 'Projects Page'

# Strict slashes disabled (accepts both)
@app.route('/flexible', strict_slashes=False)
def flexible():
    return 'Works with or without trailing slash'
```

### Route Registration and Blueprints

```python
from flask import Flask, Blueprint

app = Flask(__name__)

# Create blueprint
api = Blueprint('api', __name__, url_prefix='/api/v1')

@api.route('/users')
def get_users():
    return {'users': ['Alice', 'Bob']}

@api.route('/posts')
def get_posts():
    return {'posts': ['Post 1', 'Post 2']}

# Register blueprint
app.register_blueprint(api)

# Another blueprint with different prefix
admin = Blueprint('admin', __name__, url_prefix='/admin')

@admin.route('/dashboard')
def dashboard():
    return 'Admin Dashboard'

app.register_blueprint(admin)

# URLs will be:
# /api/v1/users
# /api/v1/posts
# /admin/dashboard
```

### Error Handlers as Routes

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/protected')
def protected():
    return 'Protected content'

# Custom error pages
@app.errorhandler(404)
def not_found(error):
    return jsonify({'error': 'Resource not found'}), 404

@app.errorhandler(500)
def internal_error(error):
    return jsonify({'error': 'Internal server error'}), 500

@app.errorhandler(403)
def forbidden(error):
    return jsonify({'error': 'Access forbidden'}), 403
```

---

## Django Routing

### URL Configuration (urls.py)

```python
# myproject/urls.py (main URL configuration)
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('myapp.urls')),
    path('api/', include('api.urls')),
]
```

### Basic Path Routes

```python
# myapp/urls.py
from django.urls import path
from . import views

app_name = 'myapp'  # Namespace for URL reversing

urlpatterns = [
    # Simple path
    path('', views.index, name='index'),
    
    # Static path
    path('about/', views.about, name='about'),
    
    # Path with parameters
    path('user/<str:username>/', views.user_profile, name='user_profile'),
    path('post/<int:post_id>/', views.post_detail, name='post_detail'),
    path('article/<slug:slug>/', views.article_detail, name='article_detail'),
    
    # Multiple parameters
    path('blog/<int:year>/<int:month>/', views.blog_archive, name='blog_archive'),
    path('category/<slug:category>/post/<int:id>/', views.category_post, name='category_post'),
]

# myapp/views.py
from django.shortcuts import render, get_object_or_404
from django.http import HttpResponse

def index(request):
    return HttpResponse('Home Page')

def user_profile(request, username):
    return HttpResponse(f'User: {username}')

def post_detail(request, post_id):
    return HttpResponse(f'Post ID: {post_id}')

def blog_archive(request, year, month):
    return HttpResponse(f'Archive: {year}/{month}')
```

### Django Path Converters

**Built-in converters:**
- `str`: Matches any non-empty string (excluding `/`)
- `int`: Matches zero or any positive integer
- `slug`: Matches slug format (letters, numbers, hyphens, underscores)
- `uuid`: Matches formatted UUID
- `path`: Matches any non-empty string (including `/`)

```python
from django.urls import path
from . import views

urlpatterns = [
    path('str/<str:text>/', views.string_view),           # /str/hello/
    path('int/<int:number>/', views.int_view),            # /int/123/
    path('slug/<slug:slug>/', views.slug_view),           # /slug/my-article/
    path('uuid/<uuid:id>/', views.uuid_view),             # /uuid/075194d3-.../
    path('path/<path:filepath>/', views.path_view),       # /path/docs/guide.pdf/
]
```

### Regular Expression Routes (re_path)

```python
from django.urls import re_path
from . import views

urlpatterns = [
    # Year (4 digits)
    re_path(r'^archive/(?P<year>[0-9]{4})/$', views.year_archive),
    
    # Year and month
    re_path(r'^archive/(?P<year>[0-9]{4})/(?P<month>[0-9]{2})/$', views.month_archive),
    
    # Optional parameter
    re_path(r'^page/(?:(?P<page>[0-9]+)/)?$', views.page_view),
    
    # Custom pattern
    re_path(r'^post/(?P<slug>[\w-]+)/$', views.post_detail),
    
    # Multiple formats
    re_path(r'^download/(?P<filename>[\w-]+\.(?:pdf|doc|txt))$', views.download),
]

# views.py
def year_archive(request, year):
    return HttpResponse(f'Archive for {year}')

def month_archive(request, year, month):
    return HttpResponse(f'Archive for {year}/{month}')

def page_view(request, page=1):
    return HttpResponse(f'Page {page}')
```

### Query Parameters in Django

```python
# views.py
from django.shortcuts import render

def search_view(request):
    # Get query parameters
    query = request.GET.get('q', '')
    page = request.GET.get('page', 1)
    
    # Get multiple values
    tags = request.GET.getlist('tag')
    
    # Get all parameters
    all_params = request.GET.dict()
    
    context = {
        'query': query,
        'page': page,
        'tags': tags,
        'params': all_params
    }
    return render(request, 'search.html', context)

# URL: /search/?q=django&page=2&tag=python&tag=web
```

### Class-Based Views Routing

```python
# urls.py
from django.urls import path
from . import views

urlpatterns = [
    # Function-based view
    path('posts/', views.post_list, name='post_list'),
    
    # Class-based view
    path('articles/', views.ArticleListView.as_view(), name='article_list'),
    path('articles/<int:pk>/', views.ArticleDetailView.as_view(), name='article_detail'),
    path('articles/create/', views.ArticleCreateView.as_view(), name='article_create'),
    path('articles/<int:pk>/update/', views.ArticleUpdateView.as_view(), name='article_update'),
    path('articles/<int:pk>/delete/', views.ArticleDeleteView.as_view(), name='article_delete'),
]

# views.py
from django.views.generic import ListView, DetailView, CreateView, UpdateView, DeleteView
from django.urls import reverse_lazy
from .models import Article

class ArticleListView(ListView):
    model = Article
    template_name = 'article_list.html'
    context_object_name = 'articles'
    paginate_by = 10

class ArticleDetailView(DetailView):
    model = Article
    template_name = 'article_detail.html'

class ArticleCreateView(CreateView):
    model = Article
    fields = ['title', 'content']
    template_name = 'article_form.html'
    success_url = reverse_lazy('article_list')

class ArticleUpdateView(UpdateView):
    model = Article
    fields = ['title', 'content']
    template_name = 'article_form.html'
    success_url = reverse_lazy('article_list')

class ArticleDeleteView(DeleteView):
    model = Article
    template_name = 'article_confirm_delete.html'
    success_url = reverse_lazy('article_list')
```

### Including Other URL Configs

```python
# myproject/urls.py
from django.urls import path, include

urlpatterns = [
    path('blog/', include('blog.urls')),
    path('shop/', include('shop.urls')),
    path('api/', include([
        path('v1/', include('api.v1.urls')),
        path('v2/', include('api.v2.urls')),
    ])),
]

# blog/urls.py
from django.urls import path
from . import views

app_name = 'blog'

urlpatterns = [
    path('', views.post_list, name='list'),
    path('<slug:slug>/', views.post_detail, name='detail'),
]

# URLs become:
# /blog/
# /blog/my-post/
# /api/v1/...
# /api/v2/...
```

### Custom Path Converters

```python
# myapp/converters.py
class FourDigitYearConverter:
    regex = '[0-9]{4}'
    
    def to_python(self, value):
        return int(value)
    
    def to_url(self, value):
        return '%04d' % value

class ISBNConverter:
    regex = '(?:97[89])?[0-9]{10}'
    
    def to_python(self, value):
        return value
    
    def to_url(self, value):
        return value

# urls.py
from django.urls import path, register_converter
from .converters import FourDigitYearConverter, ISBNConverter
from . import views

# Register custom converters
register_converter(FourDigitYearConverter, 'yyyy')
register_converter(ISBNConverter, 'isbn')

urlpatterns = [
    path('archive/<yyyy:year>/', views.year_archive),
    path('book/<isbn:isbn>/', views.book_detail),
]
```

---

## FastAPI Routing

### Basic Routes

```python
from fastapi import FastAPI

app = FastAPI()

# Simple route
@app.get("/")
async def root():
    return {"message": "Hello World"}

# Different HTTP methods
@app.get("/items")
async def read_items():
    return {"items": ["item1", "item2"]}

@app.post("/items")
async def create_item():
    return {"message": "Item created"}

@app.put("/items/{item_id}")
async def update_item(item_id: int):
    return {"message": f"Item {item_id} updated"}

@app.delete("/items/{item_id}")
async def delete_item(item_id: int):
    return {"message": f"Item {item_id} deleted"}

@app.patch("/items/{item_id}")
async def partial_update_item(item_id: int):
    return {"message": f"Item {item_id} partially updated"}
```

### Path Parameters with Type Hints

```python
from fastapi import FastAPI, Path
from typing import Optional
from enum import Enum

app = FastAPI()

# Integer parameter
@app.get("/items/{item_id}")
async def read_item(item_id: int):
    return {"item_id": item_id}

# String parameter
@app.get("/users/{username}")
async def read_user(username: str):
    return {"username": username}

# Float parameter
@app.get("/prices/{price}")
async def read_price(price: float):
    return {"price": price}

# Boolean parameter
@app.get("/active/{is_active}")
async def read_active(is_active: bool):
    return {"is_active": is_active}

# Multiple parameters
@app.get("/users/{user_id}/posts/{post_id}")
async def read_user_post(user_id: int, post_id: int):
    return {"user_id": user_id, "post_id": post_id}

# Path with validation
@app.get("/items/{item_id}")
async def read_item_validated(
    item_id: int = Path(..., title="Item ID", ge=1, le=1000)
):
    return {"item_id": item_id}

# UUID parameter
from uuid import UUID

@app.get("/items/{item_id}")
async def read_item_uuid(item_id: UUID):
    return {"item_id": item_id}
```

### Enum Path Parameters

```python
from fastapi import FastAPI
from enum import Enum

app = FastAPI()

class ModelName(str, Enum):
    alexnet = "alexnet"
    resnet = "resnet"
    lenet = "lenet"

@app.get("/models/{model_name}")
async def get_model(model_name: ModelName):
    if model_name == ModelName.alexnet:
        return {"model_name": model_name, "message": "Deep Learning FTW!"}
    
    if model_name.value == "lenet":
        return {"model_name": model_name, "message": "LeCNN all the images"}
    
    return {"model_name": model_name, "message": "Have some residuals"}

# URLs: /models/alexnet, /models/resnet, /models/lenet
# Other values return 422 Unprocessable Entity
```

### Query Parameters

```python
from fastapi import FastAPI, Query
from typing import Optional, List

app = FastAPI()

# Basic query parameters
@app.get("/items/")
async def read_items(skip: int = 0, limit: int = 10):
    return {"skip": skip, "limit": limit}

# Optional query parameter
@app.get("/items/")
async def read_items_optional(q: Optional[str] = None):
    if q:
        return {"q": q}
    return {"message": "No query parameter"}

# Query parameter with validation
@app.get("/items/")
async def read_items_validated(
    q: Optional[str] = Query(None, min_length=3, max_length=50),
    skip: int = Query(0, ge=0),
    limit: int = Query(10, ge=1, le=100)
):
    results = {"items": [{"item_id": "Foo"}, {"item_id": "Bar"}]}
    if q:
        results.update({"q": q})
    return results

# Multiple query parameters with same name
@app.get("/items/")
async def read_items_list(tags: List[str] = Query(None)):
    return {"tags": tags}

# URL: /items/?tags=python&tags=fastapi&tags=web
```

### Request Body with Path and Query Parameters

```python
from fastapi import FastAPI
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

class Item(BaseModel):
    name: str
    description: Optional[str] = None
    price: float
    tax: Optional[float] = None

# Path + Query + Body
@app.put("/items/{item_id}")
async def update_item(
    item_id: int,              # Path parameter
    item: Item,                # Request body
    q: Optional[str] = None,   # Query parameter
    short: bool = False        # Query parameter
):
    result = {"item_id": item_id, **item.dict()}
    if q:
        result.update({"q": q})
    if not short:
        result.update({
            "description": "This is an amazing item with a long description"
        })
    return result
```

### APIRouter for Organization

```python
# main.py
from fastapi import FastAPI
from routers import users, posts

app = FastAPI()

app.include_router(users.router)
app.include_router(posts.router)
app.include_router(
    users.router,
    prefix="/api/v2",
    tags=["users-v2"]
)

# routers/users.py
from fastapi import APIRouter

router = APIRouter(
    prefix="/users",
    tags=["users"],
    responses={404: {"description": "Not found"}},
)

@router.get("/")
async def read_users():
    return [{"username": "Alice"}, {"username": "Bob"}]

@router.get("/{user_id}")
async def read_user(user_id: int):
    return {"user_id": user_id}

@router.post("/")
async def create_user():
    return {"message": "User created"}

# routers/posts.py
from fastapi import APIRouter

router = APIRouter(
    prefix="/posts",
    tags=["posts"],
)

@router.get("/")
async def read_posts():
    return [{"title": "Post 1"}, {"title": "Post 2"}]

@router.get("/{post_id}")
async def read_post(post_id: int):
    return {"post_id": post_id}

# URLs become:
# /users/
# /users/123
# /posts/
# /posts/456
# /api/v2/users/
```

### Dependencies in Routes

```python
from fastapi import FastAPI, Depends, HTTPException, Header
from typing import Optional

app = FastAPI()

# Dependency function
async def get_token_header(x_token: str = Header(...)):
    if x_token != "fake-super-secret-token":
        raise HTTPException(status_code=400, detail="X-Token header invalid")

async def get_query_token(token: str):
    if token != "jessica":
        raise HTTPException(status_code=400, detail="No Jessica token provided")

# Use dependencies in routes
@app.get("/items/", dependencies=[Depends(get_token_header), Depends(get_query_token)])
async def read_items():
    return [{"item": "Foo"}, {"item": "Bar"}]

# Dependency with return value
async def common_parameters(q: Optional[str] = None, skip: int = 0, limit: int = 100):
    return {"q": q, "skip": skip, "limit": limit}

@app.get("/items/")
async def read_items(commons: dict = Depends(common_parameters)):
    return commons

@app.get("/users/")
async def read_users(commons: dict = Depends(common_parameters)):
    return commons
```

---

## RESTful URL Design

### REST Principles

**REST (Representational State Transfer)** uses standard HTTP methods and clean URLs to create intuitive APIs.

### Resource-Oriented URLs

```python
# GOOD: Resource-oriented
GET    /users              # List all users
POST   /users              # Create new user
GET    /users/123          # Get user 123
PUT    /users/123          # Update user 123
DELETE /users/123          # Delete user 123

# Nested resources
GET    /users/123/posts    # Get posts by user 123
POST   /users/123/posts    # Create post for user 123
GET    /users/123/posts/456  # Get post 456 by user 123

# BAD: Action-oriented
POST   /createUser
POST   /getUser
POST   /deleteUser
```

### HTTP Methods Mapping

| HTTP Method | CRUD | Description | Idempotent | Safe |
|------------|------|-------------|-----------|------|
| GET | Read | Retrieve resource(s) | Yes | Yes |
| POST | Create | Create new resource | No | No |
| PUT | Update | Replace entire resource | Yes | No |
| PATCH | Update | Partial update | No | No |
| DELETE | Delete | Remove resource | Yes | No |

### RESTful API Example (Flask)

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

# In-memory database
users = {}
user_id_counter = 1

# Collection endpoints
@app.route('/api/users', methods=['GET'])
def get_users():
    """List all users"""
    return jsonify(list(users.values()))

@app.route('/api/users', methods=['POST'])
def create_user():
    """Create a new user"""
    global user_id_counter
    data = request.get_json()
    
    user = {
        'id': user_id_counter,
        'name': data['name'],
        'email': data['email']
    }
    users[user_id_counter] = user
    user_id_counter += 1
    
    return jsonify(user), 201

# Resource endpoints
@app.route('/api/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    """Get specific user"""
    user = users.get(user_id)
    if not user:
        return jsonify({'error': 'User not found'}), 404
    return jsonify(user)

@app.route('/api/users/<int:user_id>', methods=['PUT'])
def update_user(user_id):
    """Replace entire user"""
    if user_id not in users:
        return jsonify({'error': 'User not found'}), 404
    
    data = request.get_json()
    users[user_id] = {
        'id': user_id,
        'name': data['name'],
        'email': data['email']
    }
    return jsonify(users[user_id])

@app.route('/api/users/<int:user_id>', methods=['PATCH'])
def partial_update_user(user_id):
    """Partial update of user"""
    if user_id not in users:
        return jsonify({'error': 'User not found'}), 404
    
    data = request.get_json()
    user = users[user_id]
    
    if 'name' in data:
        user['name'] = data['name']
    if 'email' in data:
        user['email'] = data['email']
    
    return jsonify(user)

@app.route('/api/users/<int:user_id>', methods=['DELETE'])
def delete_user(user_id):
    """Delete user"""
    if user_id not in users:
        return jsonify({'error': 'User not found'}), 404
    
    del users[user_id]
    return '', 204

# Nested resource
@app.route('/api/users/<int:user_id>/posts', methods=['GET'])
def get_user_posts(user_id):
    """Get all posts by specific user"""
    if user_id not in users:
        return jsonify({'error': 'User not found'}), 404
    
    # Filter posts by user_id
    user_posts = [p for p in posts.values() if p['user_id'] == user_id]
    return jsonify(user_posts)
```

### URL Naming Conventions

```python
# Use nouns, not verbs
GET /api/users          # Good
GET /api/getUsers       # Bad

# Use plurals for collections
GET /api/users          # Good
GET /api/user           # Bad (unless singular resource)

# Use hyphens, not underscores
GET /api/blog-posts     # Good
GET /api/blog_posts     # Acceptable but less common

# Lowercase URLs
GET /api/users          # Good
GET /api/Users          # Bad

# No trailing slash (or be consistent)
GET /api/users          # Good
GET /api/users/         # Acceptable if consistent

# Versioning
GET /api/v1/users       # Good
GET /api/v2/users       # Next version

# Filtering, sorting, searching
GET /api/users?status=active&sort=created_at
GET /api/posts?category=tech&limit=10
GET /api/search?q=python
```

---

## Advanced Routing Patterns

### URL Versioning

```python
# Flask with blueprints
from flask import Flask, Blueprint

app = Flask(__name__)

# API v1
api_v1 = Blueprint('api_v1', __name__, url_prefix='/api/v1')

@api_v1.route('/users')
def get_users_v1():
    return {'version': 'v1', 'users': ['Alice', 'Bob']}

# API v2
api_v2 = Blueprint('api_v2', __name__, url_prefix='/api/v2')

@api_v2.route('/users')
def get_users_v2():
    return {'version': 'v2', 'users': [{'id': 1, 'name': 'Alice'}, {'id': 2, 'name': 'Bob'}]}

app.register_blueprint(api_v1)
app.register_blueprint(api_v2)

# FastAPI with routers
from fastapi import APIRouter

router_v1 = APIRouter(prefix="/api/v1", tags=["v1"])
router_v2 = APIRouter(prefix="/api/v2", tags=["v2"])

@router_v1.get("/users")
async def get_users_v1():
    return {"version": "v1", "users": ["Alice", "Bob"]}

@router_v2.get("/users")
async def get_users_v2():
    return {"version": "v2", "users": [{"id": 1, "name": "Alice"}]}
```

### Subdomain Routing (Flask)

```python
from flask import Flask

app = Flask(__name__)
app.config['SERVER_NAME'] = 'example.com:5000'

@app.route('/')
def index():
    return 'Main domain'

@app.route('/', subdomain='api')
def api_index():
    return 'API subdomain'

@app.route('/users', subdomain='api')
def api_users():
    return 'API users endpoint'

@app.route('/', subdomain='blog')
def blog_index():
    return 'Blog subdomain'

# Dynamic subdomain
@app.route('/', subdomain='<username>')
def user_subdomain(username):
    return f'Subdomain for user: {username}'

# URLs:
# http://example.com:5000/
# http://api.example.com:5000/
# http://api.example.com:5000/users
# http://blog.example.com:5000/
# http://john.example.com:5000/
```

### Conditional Routing

```python
from flask import Flask, request

app = Flask(__name__)

@app.route('/data')
def get_data():
    # Route based on Accept header
    if request.headers.get('Accept') == 'application/json':
        return {'data': 'JSON response'}
    elif request.headers.get('Accept') == 'application/xml':
        return '<data>XML response</data>'
    else:
        return 'Plain text response'

# Content negotiation
@app.route('/users')
def users():
    if 'application/json' in request.headers.get('Accept', ''):
        return {'users': ['Alice', 'Bob']}
    return 'Alice, Bob'
```

### Regex-Based Routing Patterns

```python
# Flask with custom converter
from werkzeug.routing import BaseConverter

class RegexConverter(BaseConverter):
    def __init__(self, url_map, *items):
        super(RegexConverter, self).__init__(url_map)
        self.regex = items[0]

app.url_map.converters['regex'] = RegexConverter

# Match phone numbers
@app.route('/phone/<regex("[0-9]{3}-[0-9]{3}-[0-9]{4}"):phone>')
def phone_route(phone):
    return f'Phone: {phone}'

# Match dates
@app.route('/date/<regex("\d{4}-\d{2}-\d{2}"):date>')
def date_route(date):
    return f'Date: {date}'

# Django regex patterns
from django.urls import re_path

urlpatterns = [
    # Match email
    re_path(r'^user/(?P<email>[\w\.-]+@[\w\.-]+\.\w+)/, views.user_by_email),
    
    # Match phone
    re_path(r'^phone/(?P<number>\d{3}-\d{3}-\d{4})/, views.phone_lookup),
    
    # Match hex color
    re_path(r'^color/(?P<hex>[0-9A-Fa-f]{6})/, views.color_detail),
]
```

### Route Priorities and Order

```python
from flask import Flask

app = Flask(__name__)

# Order matters! More specific routes first
@app.route('/users/new')
def new_user():
    return 'Create new user form'

@app.route('/users/<int:user_id>')
def get_user(user_id):
    return f'User {user_id}'

# If reversed, '/users/new' would match the second route
# with user_id = 'new' (which would fail type conversion)

# Static before dynamic
@app.route('/posts/featured')
def featured_posts():
    return 'Featured posts'

@app.route('/posts/<slug>')
def post_detail(slug):
    return f'Post: {slug}'
```

### Catch-All Routes

```python
from flask import Flask

app = Flask(__name__)

# Catch-all route (should be last)
@app.route('/', defaults={'path': ''})
@app.route('/<path:path>')
def catch_all(path):
    return f'You requested: /{path}'

# More specific example
@app.route('/docs/')
@app.route('/docs/<path:filename>')
def documentation(filename='index.html'):
    return f'Serving documentation: {filename}'

# FastAPI catch-all
from fastapi import FastAPI

app = FastAPI()

@app.get("/{full_path:path}")
async def catch_all(full_path: str):
    return {"path": full_path}
```

### Route Decorators and Middleware

```python
from flask import Flask, jsonify
from functools import wraps

app = Flask(__name__)

# Authentication decorator
def require_auth(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        auth = request.headers.get('Authorization')
        if not auth or auth != 'Bearer secret-token':
            return jsonify({'error': 'Unauthorized'}), 401
        return f(*args, **kwargs)
    return decorated_function

# Rate limiting decorator
from time import time
from collections import defaultdict

rate_limit_storage = defaultdict(list)

def rate_limit(max_requests=5, window=60):
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            client_id = request.remote_addr
            now = time()
            
            # Clean old requests
            rate_limit_storage[client_id] = [
                req_time for req_time in rate_limit_storage[client_id]
                if now - req_time < window
            ]
            
            # Check limit
            if len(rate_limit_storage[client_id]) >= max_requests:
                return jsonify({'error': 'Rate limit exceeded'}), 429
            
            rate_limit_storage[client_id].append(now)
            return f(*args, **kwargs)
        return decorated_function
    return decorator

# Apply decorators to routes
@app.route('/protected')
@require_auth
def protected_route():
    return {'message': 'This is protected'}

@app.route('/limited')
@rate_limit(max_requests=3, window=60)
def limited_route():
    return {'message': 'Rate limited endpoint'}

@app.route('/secure')
@require_auth
@rate_limit(max_requests=10, window=60)
def secure_route():
    return {'message': 'Secure and rate limited'}
```

---

## URL Building and Reversing

### Flask URL Building

```python
from flask import Flask, url_for, redirect

app = Flask(__name__)

@app.route('/')
def index():
    return 'Home'

@app.route('/user/<username>')
def user_profile(username):
    return f'User: {username}'

@app.route('/post/<int:post_id>')
def post_detail(post_id):
    return f'Post: {post_id}'

@app.route('/build-urls')
def build_urls():
    # Generate URLs
    home_url = url_for('index')                                    # '/'
    user_url = url_for('user_profile', username='john')            # '/user/john'
    post_url = url_for('post_detail', post_id=42)                  # '/post/42'
    
    # With query parameters
    search_url = url_for('index', q='python', page=2)              # '/?q=python&page=2'
    
    # External URLs (with _external=True)
    full_url = url_for('index', _external=True)                    # 'http://localhost:5000/'
    
    # Anchor links
    section_url = url_for('index', _anchor='section1')             # '/#section1'
    
    # With scheme
    https_url = url_for('index', _external=True, _scheme='https')  # 'https://localhost:5000/'
    
    return {
        'home': home_url,
        'user': user_url,
        'post': post_url,
        'search': search_url,
        'full': full_url
    }

@app.route('/redirect-example')
def redirect_example():
    # Redirect to another route
    return redirect(url_for('user_profile', username='admin'))

# Blueprint URL building
from flask import Blueprint

api = Blueprint('api', __name__, url_prefix='/api')

@api.route('/users')
def users():
    return 'Users'

app.register_blueprint(api)

@app.route('/api-urls')
def api_urls():
    # Reference blueprint routes
    users_url = url_for('api.users')  # '/api/users'
    return {'users': users_url}
```

### Django URL Reversing

```python
# urls.py
from django.urls import path
from . import views

app_name = 'blog'

urlpatterns = [
    path('', views.index, name='index'),
    path('post/<int:pk>/', views.post_detail, name='post_detail'),
    path('category/<slug:slug>/', views.category, name='category'),
]

# views.py
from django.shortcuts import render, redirect
from django.urls import reverse

def index(request):
    # Reverse URL by name
    post_url = reverse('blog:post_detail', kwargs={'pk': 1})
    # Result: '/post/1/'
    
    category_url = reverse('blog:category', args=['python'])
    # Result: '/category/python/'
    
    return render(request, 'index.html', {
        'post_url': post_url,
        'category_url': category_url
    })

def redirect_example(request):
    # Redirect using reverse
    return redirect(reverse('blog:post_detail', kwargs={'pk': 42}))

# In templates
"""
{% url 'blog:index' %}
{% url 'blog:post_detail' pk=1 %}
{% url 'blog:category' slug='python' %}

<!-- With query parameters -->
<a href="{% url 'blog:index' %}?page=2&sort=date">Page 2</a>
"""

# In models
from django.db import models
from django.urls import reverse

class Post(models.Model):
    title = models.CharField(max_length=200)
    slug = models.SlugField()
    
    def get_absolute_url(self):
        return reverse('blog:post_detail', kwargs={'pk': self.pk})
```

### FastAPI URL Building

```python
from fastapi import FastAPI, Request
from fastapi.responses import RedirectResponse

app = FastAPI()

@app.get("/", name="home")
async def home():
    return {"message": "Home"}

@app.get("/users/{user_id}", name="user_detail")
async def user_detail(user_id: int):
    return {"user_id": user_id}

@app.get("/build-url")
async def build_url(request: Request):
    # Get URL for route by name
    home_url = request.url_for("home")
    user_url = request.url_for("user_detail", user_id=123)
    
    return {
        "home": str(home_url),
        "user": str(user_url)
    }

@app.get("/redirect-example")
async def redirect_example(request: Request):
    url = request.url_for("user_detail", user_id=42)
    return RedirectResponse(url=url)

# Using APIRouter
from fastapi import APIRouter

router = APIRouter(prefix="/api", tags=["api"])

@router.get("/items/{item_id}", name="item_detail")
async def get_item(item_id: int):
    return {"item_id": item_id}

app.include_router(router)

@app.get("/api-url")
async def api_url(request: Request):
    # Reference router route
    item_url = request.url_for("item_detail", item_id=1)
    return {"item_url": str(item_url)}
```

---

## Best Practices

### 1. URL Design Principles

```python
# ✅ GOOD: Clear, descriptive, hierarchical
GET /api/v1/users
GET /api/v1/users/123
GET /api/v1/users/123/posts
GET /api/v1/users/123/posts/456

# ❌ BAD: Confusing, flat structure
GET /api/v1/getUserById
GET /api/v1/data
GET /api/v1/stuff/123/things/456

# ✅ GOOD: Consistent naming
GET /blog-posts
GET /blog-posts/my-first-post
GET /blog-posts/my-first-post/comments

# ❌ BAD: Inconsistent
GET /blogPosts
GET /blog_posts/my-first-post
GET /BlogPosts/MyFirstPost/Comments

# ✅ GOOD: Resource-oriented
POST /users              # Create user
GET /users/123           # Get user
PUT /users/123           # Update user
DELETE /users/123        # Delete user

# ❌ BAD: Action-oriented
POST /createUser
POST /getUser/123
POST /updateUser/123
POST /deleteUser/123
```

### 2. Use Appropriate HTTP Methods

```python
# Create
POST /api/posts
{
    "title": "New Post",
    "content": "Content here"
}

# Read (single)
GET /api/posts/123

# Read (collection)
GET /api/posts?page=1&limit=10

# Update (full replacement)
PUT /api/posts/123
{
    "title": "Updated Title",
    "content": "Updated Content",
    "published": true
}

# Update (partial)
PATCH /api/posts/123
{
    "title": "New Title Only"
}

# Delete
DELETE /api/posts/123

# List with filters
GET /api/posts?status=published&category=tech&sort=-created_at
```

### 3. Versioning Strategy

```python
# URL versioning (most common)
GET /api/v1/users
GET /api/v2/users

# Header versioning
GET /api/users
Headers: Accept: application/vnd.myapp.v1+json

# Query parameter versioning
GET /api/users?version=1

# Subdomain versioning
GET https://v1.api.example.com/users
GET https://v2.api.example.com/users
```

### 4. Handle Nested Resources Carefully

```python
# ✅ GOOD: Limited nesting (2-3 levels max)
GET /users/123/posts
GET /posts/456/comments

# ❌ BAD: Too deep nesting
GET /users/123/posts/456/comments/789/replies/012

# ✅ BETTER: Flatten when possible
GET /comments/789/replies
GET /replies?comment_id=789
```

### 5. Pagination, Filtering, and Sorting

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/api/posts')
def get_posts():
    # Pagination
    page = request.args.get('page', 1, type=int)
    per_page = request.args.get('per_page', 20, type=int)
    
    # Filtering
    status = request.args.get('status')  # ?status=published
    category = request.args.get('category')  # ?category=tech
    
    # Searching
    search = request.args.get('q')  # ?q=python
    
    # Sorting
    sort = request.args.get('sort', '-created_at')  # -created_at means descending
    
    # Build response with metadata
    return jsonify({
        'data': [],  # Your filtered, sorted, paginated data
        'meta': {
            'page': page,
            'per_page': per_page,
            'total_pages': 10,
            'total_items': 200
        },
        'links': {
            'self': f'/api/posts?page={page}',
            'next': f'/api/posts?page={page + 1}',
            'prev': f'/api/posts?page={page - 1}' if page > 1 else None
        }
    })

# Example URLs:
# /api/posts?page=2&per_page=10
# /api/posts?status=published&category=tech
# /api/posts?q=python&sort=-views
# /api/posts?status=published&sort=created_at&page=1
```

### 6. Error Handling

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.errorhandler(404)
def not_found(error):
    return jsonify({
        'error': 'Not Found',
        'message': 'The requested resource was not found',
        'status': 404
    }), 404

@app.errorhandler(400)
def bad_request(error):
    return jsonify({
        'error': 'Bad Request',
        'message': str(error),
        'status': 400
    }), 400

@app.errorhandler(500)
def internal_error(error):
    return jsonify({
        'error': 'Internal Server Error',
        'message': 'An unexpected error occurred',
        'status': 500
    }), 500

@app.route('/api/posts/<int:post_id>')
def get_post(post_id):
    post = get_post_from_db(post_id)
    
    if not post:
        return jsonify({
            'error': 'Not Found',
            'message': f'Post with id {post_id} not found'
        }), 404
    
    return jsonify(post)
```

### 7. Use Trailing Slashes Consistently

```python
# Choose one style and stick to it

# WITH trailing slashes (Flask default with strict_slashes=True)
@app.route('/users/')
@app.route('/posts/')

# WITHOUT trailing slashes
@app.route('/users', strict_slashes=False)
@app.route('/posts', strict_slashes=False)

# Django encourages trailing slashes
path('users/', views.users),
path('posts/', views.posts),
```

### 8. Security Considerations

```python
from flask import Flask, request, abort

app = Flask(__name__)

# Validate input parameters
@app.route('/api/users/<int:user_id>')
def get_user(user_id):
    # Type conversion provides basic validation
    if user_id < 1 or user_id > 1000000:
        abort(400, 'Invalid user ID')
    
    return {'user_id': user_id}

# Sanitize path parameters
import re

@app.route('/files/<path:filename>')
def serve_file(filename):
    # Prevent directory traversal
    if '..' in filename or filename.startswith('/'):
        abort(400, 'Invalid filename')
    
    # Only allow certain characters
    if not re.match(r'^[\w\-./]+, filename):
        abort(400, 'Invalid filename')
    
    return {'filename': filename}

# Rate limiting per route
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(
    app=app,
    key_func=get_remote_address,
    default_limits=["200 per day", "50 per hour"]
)

@app.route('/api/expensive-operation')
@limiter.limit("5 per minute")
def expensive_operation():
    return {'status': 'processing'}
```

### 9. Documentation

```python
# FastAPI automatic documentation
from fastapi import FastAPI, Path, Query
from typing import Optional

app = FastAPI(
    title="My API",
    description="API for managing resources",
    version="1.0.0"
)

@app.get(
    "/users/{user_id}",
    summary="Get user by ID",
    description="Retrieve a specific user by their unique identifier",
    response_description="The requested user",
    tags=["users"]
)
async def get_user(
    user_id: int = Path(..., description="The ID of the user to retrieve", ge=1),
    include_posts: bool = Query(False, description="Include user's posts in response")
):
    """
    Get a user by ID.
    
    - **user_id**: Unique identifier for the user
    - **include_posts**: Whether to include posts (optional)
    """
    return {"user_id": user_id, "include_posts": include_posts}

# Access automatic docs at:
# /docs (Swagger UI)
# /redoc (ReDoc)
```

### 10. Testing Routes

```python
# Flask testing
import unittest
from app import app

class TestRoutes(unittest.TestCase):
    def setUp(self):
        self.app = app.test_client()
        self.app.testing = True
    
    def test_home_route(self):
        response = self.app.get('/')
        self.assertEqual(response.status_code, 200)
    
    def test_user_route(self):
        response = self.app.get('/users/123')
        self.assertEqual(response.status_code, 200)
        data = response.get_json()
        self.assertEqual(data['user_id'], 123)
    
    def test_create_user(self):
        response = self.app.post('/users', 
            json={'name': 'John', 'email': 'john@example.com'})
        self.assertEqual(response.status_code, 201)
    
    def test_not_found(self):
        response = self.app.get('/nonexistent')
        self.assertEqual(response.status_code, 404)

# FastAPI testing
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_read_user():
    response = client.get("/users/1")
    assert response.status_code == 200
    assert response.json()["user_id"] == 1

def test_create_user():
    response = client.post("/users", json={"name": "Alice", "email": "alice@example.com"})
    assert response.status_code == 201

def test_query_parameters():
    response = client.get("/search?q=python&page=2")
    assert response.status_code == 200
    data = response.json()
    assert data["query"] == "python"
    assert data["page"] == 2
```

---

## Summary

### Key Takeaways

**1. Routing Fundamentals:**
- Routes map URLs to handler functions
- Combine URL patterns with HTTP methods
- Use path parameters for required values
- Use query parameters for optional values

**2. Framework Differences:**
- **Flask**: Decorator-based, simple and flexible
- **Django**: Configuration-based, powerful regex support
- **FastAPI**: Type-hint driven, automatic validation

**3. RESTful Design:**
- Use resource-oriented URLs
- Follow HTTP method conventions
- Keep URLs clean and predictable
- Version your APIs

**4. Best Practices:**
- Be consistent in naming conventions
- Limit URL nesting depth
- Validate all input parameters
- Document your routes
- Test thoroughly
- Consider security implications

**5. URL Building:**
- Always use URL building functions
- Never hardcode URLs
- Use named routes for flexibility
- Generate URLs programmatically

### Common Patterns

```python
# Collection and resource
GET    /api/items          # List collection
POST   /api/items          # Create item
GET    /api/items/:id      # Get single item
PUT    /api/items/:id      # Replace item
PATCH  /api/items/:id      # Update item
DELETE /api/items/:id      # Delete item

# Nested resources
GET    /api/users/:id/posts          # User's posts
POST   /api/users/:id/posts          # Create post for user
GET    /api/users/:id/posts/:post_id # Specific post

# Filtering and pagination
GET    /api/items?status=active&page=1&sort=-created_at

# Search
GET    /api/search?q=query

# Actions (when REST doesn't fit)
POST   /api/items/:id/publish
POST   /api/items/:id/archive
```

Routing is the foundation of web applications. Master these concepts, and you'll build intuitive, maintainable, and scalable web services!