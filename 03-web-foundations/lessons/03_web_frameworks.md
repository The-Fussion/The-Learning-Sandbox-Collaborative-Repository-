# Web Frameworks with Python

## Table of Contents
1. [Introduction to Web Frameworks](#introduction-to-web-frameworks)
2. [Flask - Lightweight Framework](#flask---lightweight-framework)
3. [Django - Full-Featured Framework](#django---full-featured-framework)
4. [FastAPI - Modern Async Framework](#fastapi---modern-async-framework)
5. [Other Notable Frameworks](#other-notable-frameworks)
6. [Framework Comparison](#framework-comparison)
7. [Choosing the Right Framework](#choosing-the-right-framework)
8. [Building a Complete Application](#building-a-complete-application)

---

## Introduction to Web Frameworks

A **web framework** is a software framework designed to support the development of web applications, web services, and web APIs. It provides a standard way to build and deploy web applications.

### Why Use a Web Framework?

1. **Rapid Development**: Pre-built components save time
2. **Best Practices**: Enforces proven patterns and conventions
3. **Security**: Built-in protection against common vulnerabilities
4. **Maintainability**: Structured code is easier to maintain
5. **Community Support**: Libraries, plugins, and documentation
6. **Scalability**: Tools for growing applications

### Common Framework Components

- **Routing**: URL-to-function mapping
- **Request/Response Handling**: Processing HTTP requests
- **Templates**: Dynamic HTML generation
- **Database ORM**: Object-Relational Mapping
- **Forms**: Input validation and processing
- **Authentication**: User login and permissions
- **Session Management**: Stateful user interactions
- **Security Features**: CSRF protection, XSS prevention
- **Testing Tools**: Unit and integration testing support

### Framework Categories

**Micro Frameworks** (Flask, Bottle)
- Minimal core functionality
- Maximum flexibility
- Add features as needed
- Ideal for small to medium projects

**Full-Stack Frameworks** (Django, Pyramid)
- Batteries included
- Comprehensive feature set
- Opinionated structure
- Ideal for large, complex projects

**Async Frameworks** (FastAPI, Starlette, Sanic)
- Native async/await support
- High performance
- Modern Python features
- Ideal for real-time applications

---

## Flask - Lightweight Framework

Flask is a micro web framework written in Python. It's designed to be simple and extensible.

### Installation

```bash
pip install flask
```

### Hello World Application

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello_world():
    return '<h1>Hello, World!</h1>'

if __name__ == '__main__':
    app.run(debug=True)
```

### Routing and URL Building

```python
from flask import Flask, url_for, redirect

app = Flask(__name__)

# Basic route
@app.route('/')
def index():
    return 'Index Page'

# Route with variable
@app.route('/user/<username>')
def show_user_profile(username):
    return f'User: {username}'

# Route with type converter
@app.route('/post/<int:post_id>')
def show_post(post_id):
    return f'Post {post_id}'

# Multiple routes for same function
@app.route('/about')
@app.route('/info')
def about():
    return 'About Page'

# HTTP methods
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        return 'Processing login...'
    return 'Login form'

# URL building
@app.route('/build-url')
def build_url():
    # Generate URL for 'show_user_profile' function
    url = url_for('show_user_profile', username='john')
    return f'URL: {url}'

# Redirect
@app.route('/old-page')
def old_page():
    return redirect(url_for('index'))

if __name__ == '__main__':
    app.run(debug=True)
```

### Request and Response

```python
from flask import Flask, request, jsonify, make_response

app = Flask(__name__)

@app.route('/process', methods=['POST'])
def process_data():
    # Access form data
    username = request.form.get('username')
    
    # Access JSON data
    data = request.get_json()
    
    # Access query parameters
    page = request.args.get('page', 1, type=int)
    
    # Access headers
    user_agent = request.headers.get('User-Agent')
    
    # Access cookies
    session_id = request.cookies.get('session_id')
    
    # Access files
    if 'file' in request.files:
        file = request.files['file']
        file.save(f'uploads/{file.filename}')
    
    # Return JSON response
    return jsonify({
        'status': 'success',
        'data': data
    })

@app.route('/custom-response')
def custom_response():
    # Create custom response
    response = make_response('Custom content', 200)
    response.headers['X-Custom-Header'] = 'Value'
    response.set_cookie('user_id', '123', max_age=3600)
    return response

@app.route('/error')
def error():
    # Return error response
    return jsonify({'error': 'Something went wrong'}), 400

if __name__ == '__main__':
    app.run(debug=True)
```

### Templates with Jinja2

```python
from flask import Flask, render_template

app = Flask(__name__)

@app.route('/hello/<name>')
def hello(name):
    return render_template('hello.html', name=name)

@app.route('/users')
def users():
    users_list = [
        {'id': 1, 'name': 'Alice', 'email': 'alice@example.com'},
        {'id': 2, 'name': 'Bob', 'email': 'bob@example.com'},
        {'id': 3, 'name': 'Charlie', 'email': 'charlie@example.com'}
    ]
    return render_template('users.html', users=users_list)

if __name__ == '__main__':
    app.run(debug=True)
```

**templates/hello.html:**
```html
<!DOCTYPE html>
<html>
<head>
    <title>Hello {{ name }}</title>
</head>
<body>
    <h1>Hello, {{ name|capitalize }}!</h1>
    
    {% if name == 'admin' %}
        <p>Welcome, administrator!</p>
    {% else %}
        <p>Welcome, user!</p>
    {% endif %}
</body>
</html>
```

**templates/users.html:**
```html
<!DOCTYPE html>
<html>
<head>
    <title>Users</title>
</head>
<body>
    <h1>User List</h1>
    
    {% if users %}
        <ul>
        {% for user in users %}
            <li>{{ user.name }} - {{ user.email }}</li>
        {% endfor %}
        </ul>
    {% else %}
        <p>No users found.</p>
    {% endif %}
</body>
</html>
```

### Database Integration (SQLAlchemy)

```python
from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///app.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

db = SQLAlchemy(app)

# Define models
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    posts = db.relationship('Post', backref='author', lazy=True)
    
    def to_dict(self):
        return {
            'id': self.id,
            'username': self.username,
            'email': self.email,
            'created_at': self.created_at.isoformat()
        }

class Post(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200), nullable=False)
    content = db.Column(db.Text, nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    
    def to_dict(self):
        return {
            'id': self.id,
            'title': self.title,
            'content': self.content,
            'created_at': self.created_at.isoformat(),
            'author': self.author.username
        }

# Create tables
with app.app_context():
    db.create_all()

# CRUD endpoints
@app.route('/api/users', methods=['GET'])
def get_users():
    users = User.query.all()
    return jsonify([user.to_dict() for user in users])

@app.route('/api/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    user = User.query.get_or_404(user_id)
    return jsonify(user.to_dict())

@app.route('/api/users', methods=['POST'])
def create_user():
    data = request.get_json()
    
    user = User(
        username=data['username'],
        email=data['email']
    )
    db.session.add(user)
    db.session.commit()
    
    return jsonify(user.to_dict()), 201

@app.route('/api/users/<int:user_id>', methods=['PUT'])
def update_user(user_id):
    user = User.query.get_or_404(user_id)
    data = request.get_json()
    
    user.username = data.get('username', user.username)
    user.email = data.get('email', user.email)
    db.session.commit()
    
    return jsonify(user.to_dict())

@app.route('/api/users/<int:user_id>', methods=['DELETE'])
def delete_user(user_id):
    user = User.query.get_or_404(user_id)
    db.session.delete(user)
    db.session.commit()
    
    return '', 204

@app.route('/api/posts', methods=['GET'])
def get_posts():
    posts = Post.query.all()
    return jsonify([post.to_dict() for post in posts])

@app.route('/api/posts', methods=['POST'])
def create_post():
    data = request.get_json()
    
    post = Post(
        title=data['title'],
        content=data['content'],
        user_id=data['user_id']
    )
    db.session.add(post)
    db.session.commit()
    
    return jsonify(post.to_dict()), 201

if __name__ == '__main__':
    app.run(debug=True)
```

### Flask Blueprints (Modular Applications)

```python
# app.py
from flask import Flask
from blueprints.users import users_bp
from blueprints.posts import posts_bp

app = Flask(__name__)

# Register blueprints
app.register_blueprint(users_bp, url_prefix='/users')
app.register_blueprint(posts_bp, url_prefix='/posts')

@app.route('/')
def index():
    return 'Main Application'

if __name__ == '__main__':
    app.run(debug=True)
```

**blueprints/users.py:**
```python
from flask import Blueprint, jsonify

users_bp = Blueprint('users', __name__)

@users_bp.route('/')
def list_users():
    return jsonify({'users': ['Alice', 'Bob', 'Charlie']})

@users_bp.route('/<int:user_id>')
def get_user(user_id):
    return jsonify({'id': user_id, 'name': 'User'})

@users_bp.route('/profile')
def profile():
    return 'User Profile'
```

**blueprints/posts.py:**
```python
from flask import Blueprint, jsonify

posts_bp = Blueprint('posts', __name__)

@posts_bp.route('/')
def list_posts():
    return jsonify({'posts': ['Post 1', 'Post 2']})

@posts_bp.route('/<int:post_id>')
def get_post(post_id):
    return jsonify({'id': post_id, 'title': 'Post Title'})
```

### Error Handling and Logging

```python
from flask import Flask, jsonify
import logging
from logging.handlers import RotatingFileHandler

app = Flask(__name__)

# Configure logging
if not app.debug:
    file_handler = RotatingFileHandler('app.log', maxBytes=10240, backupCount=10)
    file_handler.setFormatter(logging.Formatter(
        '%(asctime)s %(levelname)s: %(message)s [in %(pathname)s:%(lineno)d]'
    ))
    file_handler.setLevel(logging.INFO)
    app.logger.addHandler(file_handler)
    app.logger.setLevel(logging.INFO)
    app.logger.info('Application startup')

# Custom error handlers
@app.errorhandler(404)
def not_found(error):
    app.logger.warning(f'404 error: {error}')
    return jsonify({'error': 'Resource not found'}), 404

@app.errorhandler(500)
def internal_error(error):
    app.logger.error(f'500 error: {error}')
    return jsonify({'error': 'Internal server error'}), 500

@app.errorhandler(Exception)
def handle_exception(error):
    app.logger.exception('Unhandled exception')
    return jsonify({'error': 'An unexpected error occurred'}), 500

# Route that logs
@app.route('/log-test')
def log_test():
    app.logger.debug('Debug message')
    app.logger.info('Info message')
    app.logger.warning('Warning message')
    app.logger.error('Error message')
    return 'Check logs'

if __name__ == '__main__':
    app.run(debug=True)
```

### Flask Extensions

```python
# Popular Flask extensions

# Flask-Login: User session management
from flask_login import LoginManager, login_user, logout_user, login_required

# Flask-WTF: Forms with CSRF protection
from flask_wtf import FlaskForm
from wtforms import StringField, PasswordField, SubmitField
from wtforms.validators import DataRequired, Email

# Flask-Mail: Email sending
from flask_mail import Mail, Message

# Flask-Migrate: Database migrations
from flask_migrate import Migrate

# Flask-CORS: Cross-Origin Resource Sharing
from flask_cors import CORS

# Flask-JWT-Extended: JWT authentication
from flask_jwt_extended import JWTManager, create_access_token

# Flask-Caching: Response caching
from flask_caching import Cache

# Flask-Limiter: Rate limiting
from flask_limiter import Limiter

app = Flask(__name__)

# Configure extensions
CORS(app)
cache = Cache(app, config={'CACHE_TYPE': 'simple'})
limiter = Limiter(app, key_func=lambda: request.remote_addr)
```

---

## Django - Full-Featured Framework

Django is a high-level Python web framework that encourages rapid development and clean, pragmatic design.

### Installation

```bash
pip install django
```

### Creating a Django Project

```bash
# Create new project
django-admin startproject myproject

# Project structure
myproject/
    manage.py
    myproject/
        __init__.py
        settings.py
        urls.py
        asgi.py
        wsgi.py

# Create an app
cd myproject
python manage.py startapp myapp

# Run development server
python manage.py runserver
```

### Django Models (ORM)

```python
# myapp/models.py
from django.db import models
from django.contrib.auth.models import User

class Category(models.Model):
    name = models.CharField(max_length=100)
    slug = models.SlugField(unique=True)
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        verbose_name_plural = 'categories'
        ordering = ['name']
    
    def __str__(self):
        return self.name

class Post(models.Model):
    STATUS_CHOICES = [
        ('draft', 'Draft'),
        ('published', 'Published'),
    ]
    
    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    author = models.ForeignKey(User, on_delete=models.CASCADE, related_name='posts')
    category = models.ForeignKey(Category, on_delete=models.SET_NULL, null=True, related_name='posts')
    content = models.TextField()
    status = models.CharField(max_length=10, choices=STATUS_CHOICES, default='draft')
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    published_at = models.DateTimeField(null=True, blank=True)
    views = models.IntegerField(default=0)
    tags = models.ManyToManyField('Tag', related_name='posts', blank=True)
    
    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['-created_at']),
            models.Index(fields=['slug']),
        ]
    
    def __str__(self):
        return self.title

class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)
    
    def __str__(self):
        return self.name

class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='comments')
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    active = models.BooleanField(default=True)
    
    class Meta:
        ordering = ['created_at']
    
    def __str__(self):
        return f'Comment by {self.author.username} on {self.post.title}'
```

### Django Migrations

```bash
# Create migrations
python manage.py makemigrations

# Apply migrations
python manage.py migrate

# Show migrations
python manage.py showmigrations

# SQL for migration
python manage.py sqlmigrate myapp 0001

# Reverse migration
python manage.py migrate myapp 0001
```

### Django Views

```python
# myapp/views.py
from django.shortcuts import render, get_object_or_404, redirect
from django.http import JsonResponse, HttpResponse
from django.views import View
from django.views.generic import ListView, DetailView, CreateView, UpdateView, DeleteView
from django.contrib.auth.decorators import login_required
from django.contrib.auth.mixins import LoginRequiredMixin
from django.urls import reverse_lazy
from .models import Post, Category
from .forms import PostForm

# Function-based view
def post_list(request):
    posts = Post.objects.filter(status='published').select_related('author', 'category')
    return render(request, 'myapp/post_list.html', {'posts': posts})

def post_detail(request, slug):
    post = get_object_or_404(Post, slug=slug, status='published')
    post.views += 1
    post.save(update_fields=['views'])
    return render(request, 'myapp/post_detail.html', {'post': post})

@login_required
def post_create(request):
    if request.method == 'POST':
        form = PostForm(request.POST)
        if form.is_valid():
            post = form.save(commit=False)
            post.author = request.user
            post.save()
            form.save_m2m()  # Save many-to-many relationships
            return redirect('post_detail', slug=post.slug)
    else:
        form = PostForm()
    return render(request, 'myapp/post_form.html', {'form': form})

# Class-based views
class PostListView(ListView):
    model = Post
    template_name = 'myapp/post_list.html'
    context_object_name = 'posts'
    paginate_by = 10
    
    def get_queryset(self):
        return Post.objects.filter(status='published').select_related('author', 'category')

class PostDetailView(DetailView):
    model = Post
    template_name = 'myapp/post_detail.html'
    context_object_name = 'post'
    
    def get_queryset(self):
        return Post.objects.filter(status='published')
    
    def get_object(self):
        obj = super().get_object()
        obj.views += 1
        obj.save(update_fields=['views'])
        return obj

class PostCreateView(LoginRequiredMixin, CreateView):
    model = Post
    form_class = PostForm
    template_name = 'myapp/post_form.html'
    success_url = reverse_lazy('post_list')
    
    def form_valid(self, form):
        form.instance.author = self.request.user
        return super().form_valid(form)

class PostUpdateView(LoginRequiredMixin, UpdateView):
    model = Post
    form_class = PostForm
    template_name = 'myapp/post_form.html'
    
    def get_queryset(self):
        return Post.objects.filter(author=self.request.user)

class PostDeleteView(LoginRequiredMixin, DeleteView):
    model = Post
    success_url = reverse_lazy('post_list')
    
    def get_queryset(self):
        return Post.objects.filter(author=self.request.user)

# API view
def api_posts(request):
    if request.method == 'GET':
        posts = Post.objects.filter(status='published').values('id', 'title', 'slug', 'created_at')
        return JsonResponse(list(posts), safe=False)
    
    elif request.method == 'POST':
        # Handle POST request
        import json
        data = json.loads(request.body)
        post = Post.objects.create(
            title=data['title'],
            content=data['content'],
            author=request.user
        )
        return JsonResponse({'id': post.id, 'title': post.title}, status=201)
```

### Django URLs

```python
# myproject/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('myapp.urls')),
]

# myapp/urls.py
from django.urls import path
from . import views

app_name = 'myapp'

urlpatterns = [
    # Function-based views
    path('', views.post_list, name='post_list'),
    path('post/<slug:slug>/', views.post_detail, name='post_detail'),
    path('post/create/', views.post_create, name='post_create'),
    
    # Class-based views
    path('posts/', views.PostListView.as_view(), name='posts'),
    path('posts/<slug:slug>/', views.PostDetailView.as_view(), name='post'),
    path('posts/new/', views.PostCreateView.as_view(), name='post_new'),
    path('posts/<slug:slug>/edit/', views.PostUpdateView.as_view(), name='post_edit'),
    path('posts/<slug:slug>/delete/', views.PostDeleteView.as_view(), name='post_delete'),
    
    # API endpoints
    path('api/posts/', views.api_posts, name='api_posts'),
]
```

### Django Templates

```django
<!-- myapp/templates/myapp/base.html -->
<!DOCTYPE html>
<html>
<head>
    <title>{% block title %}My Site{% endblock %}</title>
    <link rel="stylesheet" href="{% static 'css/style.css' %}">
</head>
<body>
    <nav>
        <a href="{% url 'myapp:post_list' %}">Home</a>
        {% if user.is_authenticated %}
            <a href="{% url 'myapp:post_create' %}">New Post</a>
            <a href="{% url 'logout' %}">Logout</a>
        {% else %}
            <a href="{% url 'login' %}">Login</a>
        {% endif %}
    </nav>
    
    <main>
        {% if messages %}
            {% for message in messages %}
                <div class="alert alert-{{ message.tags }}">
                    {{ message }}
                </div>
            {% endfor %}
        {% endif %}
        
        {% block content %}{% endblock %}
    </main>
    
    <footer>
        <p>&copy; 2026 My Site</p>
    </footer>
</body>
</html>

<!-- myapp/templates/myapp/post_list.html -->
{% extends 'myapp/base.html' %}

{% block title %}Posts{% endblock %}

{% block content %}
    <h1>All Posts</h1>
    
    {% for post in posts %}
        <article>
            <h2><a href="{% url 'myapp:post_detail' post.slug %}">{{ post.title }}</a></h2>
            <p>By {{ post.author.username }} on {{ post.created_at|date:"F d, Y" }}</p>
            <p>{{ post.content|truncatewords:30 }}</p>
            <p>Views: {{ post.views }}</p>
        </article>
    {% empty %}
        <p>No posts yet.</p>
    {% endfor %}
    
    {% if is_paginated %}
        <div class="pagination">
            {% if page_obj.has_previous %}
                <a href="?page={{ page_obj.previous_page_number }}">Previous</a>
            {% endif %}
            
            <span>Page {{ page_obj.number }} of {{ page_obj.paginator.num_pages }}</span>
            
            {% if page_obj.has_next %}
                <a href="?page={{ page_obj.next_page_number }}">Next</a>
            {% endif %}
        </div>
    {% endif %}
{% endblock %}
```

### Django Forms

```python
# myapp/forms.py
from django import forms
from .models import Post, Comment

class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = ['title', 'content', 'category', 'tags', 'status']
        widgets = {
            'content': forms.Textarea(attrs={'rows': 10}),
            'tags': forms.CheckboxSelectMultiple(),
        }
    
    def clean_title(self):
        title = self.cleaned_data['title']
        if len(title) < 5:
            raise forms.ValidationError('Title must be at least 5 characters long')
        return title

class CommentForm(forms.ModelForm):
    class Meta:
        model = Comment
        fields = ['content']
        widgets = {
            'content': forms.Textarea(attrs={'rows': 3, 'placeholder': 'Write a comment...'})
        }

class SearchForm(forms.Form):
    query = forms.CharField(
        max_length=100,
        required=False,
        widget=forms.TextInput(attrs={'placeholder': 'Search posts...'})
    )
    category = forms.ModelChoiceField(
        queryset=Category.objects.all(),
        required=False,
        empty_label='All Categories'
    )
```

### Django Admin

```python
# myapp/admin.py
from django.contrib import admin
from .models import Post, Category, Tag, Comment

@admin.register(Category)
class CategoryAdmin(admin.ModelAdmin):
    list_display = ['name', 'slug', 'created_at']
    prepopulated_fields = {'slug': ('name',)}

@admin.register(Tag)
class TagAdmin(admin.ModelAdmin):
    list_display = ['name']

class CommentInline(admin.TabularInline):
    model = Comment
    extra = 0
    fields = ['author', 'content', 'active', 'created_at']
    readonly_fields = ['created_at']

@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ['title', 'author', 'category', 'status', 'created_at', 'views']
    list_filter = ['status', 'category', 'created_at']
    search_fields = ['title', 'content']
    prepopulated_fields = {'slug': ('title',)}
    date_hierarchy = 'created_at'
    ordering = ['-created_at']
    filter_horizontal = ['tags']
    inlines = [CommentInline]
    
    fieldsets = (
        ('Basic Information', {
            'fields': ('title', 'slug', 'author', 'category')
        }),
        ('Content', {
            'fields': ('content', 'tags')
        }),
        ('Publishing', {
            'fields': ('status', 'published_at')
        }),
        ('Statistics', {
            'fields': ('views',),
            'classes': ('collapse',)
        }),
    )
    
    def get_queryset(self, request):
        qs = super().get_queryset(request)
        return qs.select_related('author', 'category')

@admin.register(Comment)
class CommentAdmin(admin.ModelAdmin):
    list_display = ['author', 'post', 'created_at', 'active']
    list_filter = ['active', 'created_at']
    search_fields = ['author__username', 'content']
    actions = ['approve_comments', 'disapprove_comments']
    
    def approve_comments(self, request, queryset):
        queryset.update(active=True)
    approve_comments.short_description = 'Approve selected comments'
    
    def disapprove_comments(self, request, queryset):
        queryset.update(active=False)
    disapprove_comments.short_description = 'Disapprove selected comments'
```

### Django REST Framework

```python
# Install: pip install djangorestframework

# settings.py
INSTALLED_APPS = [
    ...
    'rest_framework',
]

REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10,
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.TokenAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticatedOrReadOnly',
    ],
}

# myapp/serializers.py
from rest_framework import serializers
from .models import Post, Category, Comment

class CategorySerializer(serializers.ModelSerializer):
    class Meta:
        model = Category
        fields = ['id', 'name', 'slug']

class CommentSerializer(serializers.ModelSerializer):
    author = serializers.StringRelatedField()
    
    class Meta:
        model = Comment
        fields = ['id', 'author', 'content', 'created_at']

class PostSerializer(serializers.ModelSerializer):
    author = serializers.StringRelatedField()
    category = CategorySerializer()
    comments = CommentSerializer(many=True, read_only=True)
    
    class Meta:
        model = Post
        fields = ['id', 'title', 'slug', 'author', 'category', 'content', 
                  'status', 'created_at', 'views', 'comments']
        read_only_fields = ['slug', 'views', 'created_at']

class PostCreateSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['title', 'content', 'category', 'status']

# myapp/api_views.py
from rest_framework import viewsets, filters, status
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticatedOrReadOnly, IsAuthenticated
from django_filters.rest_framework import DjangoFilterBackend
from .models import Post, Category
from .serializers import PostSerializer, PostCreateSerializer, CategorySerializer

class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.filter(status='published').select_related('author', 'category')
    serializer_class = PostSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_fields = ['category', 'status']
    search_fields = ['title', 'content']
    ordering_fields = ['created_at', 'views']
    lookup_field = 'slug'
    
    def get_serializer_class(self):
        if self.action == 'create':
            return PostCreateSerializer
        return PostSerializer
    
    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
    
    @action(detail=True, methods=['post'])
    def increment_views(self, request, slug=None):
        post = self.get_object()
        post.views += 1
        post.save(update_fields=['views'])
        return Response({'views': post.views})
    
    @action(detail=False, methods=['get'])
    def popular(self, request):
        popular_posts = self.queryset.order_by('-views')[:10]
        serializer = self.get_serializer(popular_posts, many=True)
        return Response(serializer.data)

class CategoryViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Category.objects.all()
    serializer_class = CategorySerializer
    lookup_field = 'slug'

# myapp/urls.py (API)
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .api_views import PostViewSet, CategoryViewSet

router = DefaultRouter()
router.register(r'posts', PostViewSet)
router.register(r'categories', CategoryViewSet)

urlpatterns = [
    path('api/', include(router.urls)),
]
```

---

## FastAPI - Modern Async Framework

FastAPI is a modern, fast web framework for building APIs with Python 3.7+ based on standard Python type hints.

### Installation

```bash
pip install fastapi uvicorn[standard]
```

### Hello World Application

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}

@app.get("/items/{item_id}")
async def read_item(item_id: int, q: str = None):
    return {"item_id": item_id, "q": q}

# Run with: uvicorn main:app --reload
```

### Request and Response Models (Pydantic)

```python
from fastapi import FastAPI, HTTPException, status, Query, Path, Body
from pydantic import BaseModel, Field, EmailStr
from typing import Optional, List
from datetime import datetime

app = FastAPI(title="My API", version="1.0.0")

# Pydantic models for request/response
class UserBase(BaseModel):
    username: str = Field(..., min_length=3, max_length=50)
    email: EmailStr
    full_name: Optional[str] = None

class UserCreate(UserBase):
    password: str = Field(..., min_length=8)

class UserResponse(UserBase):
    id: int
    created_at: datetime
    is_active: bool = True
    
    class Config:
        orm_mode = True  # Allow ORM objects

class PostBase(BaseModel):
    title: str = Field(..., min_length=5, max_length=200)
    content: str
    published: bool = False

class PostCreate(PostBase):
    pass

class PostResponse(PostBase):
    id: int
    created_at: datetime
    author_id: int
    views: int = 0
    
    class Config:
        orm_mode = True

# In-memory database (for demo)
users_db = {}
posts_db = {}
user_id_counter = 1
post_id_counter = 1

# Endpoints
@app.post("/users/", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(user: UserCreate):
    global user_id_counter
    
    # Check if user exists
    for existing_user in users_db.values():
        if existing_user['email'] == user.email:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail="Email already registered"
            )
    
    user_dict = user.dict()
    user_dict['id'] = user_id_counter
    user_dict['created_at'] = datetime.now()
    user_dict['is_active'] = True
    del user_dict['password']  # Don't store plain password
    
    users_db[user_id_counter] = user_dict
    user_id_counter += 1
    
    return user_dict

@app.get("/users/", response_model=List[UserResponse])
async def list_users(
    skip: int = Query(0, ge=0),
    limit: int = Query(10, ge=1, le=100)
):
    users = list(users_db.values())
    return users[skip : skip + limit]

@app.get("/users/{user_id}", response_model=UserResponse)
async def get_user(user_id: int = Path(..., gt=0)):
    if user_id not in users_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found"
        )
    return users_db[user_id]

@app.put("/users/{user_id}", response_model=UserResponse)
async def update_user(
    user_id: int,
    user: UserBase
):
    if user_id not in users_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found"
        )
    
    user_dict = users_db[user_id]
    user_dict.update(user.dict(exclude_unset=True))
    
    return user_dict

@app.delete("/users/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(user_id: int):
    if user_id not in users_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found"
        )
    
    del users_db[user_id]
    return None

# Posts endpoints
@app.post("/posts/", response_model=PostResponse, status_code=status.HTTP_201_CREATED)
async def create_post(
    post: PostCreate,
    author_id: int = Body(..., embed=True)
):
    global post_id_counter
    
    if author_id not in users_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Author not found"
        )
    
    post_dict = post.dict()
    post_dict['id'] = post_id_counter
    post_dict['created_at'] = datetime.now()
    post_dict['author_id'] = author_id
    post_dict['views'] = 0
    
    posts_db[post_id_counter] = post_dict
    post_id_counter += 1
    
    return post_dict

@app.get("/posts/", response_model=List[PostResponse])
async def list_posts(
    published: Optional[bool] = None,
    skip: int = 0,
    limit: int = 10
):
    posts = list(posts_db.values())
    
    if published is not None:
        posts = [p for p in posts if p['published'] == published]
    
    return posts[skip : skip + limit]

@app.get("/posts/{post_id}", response_model=PostResponse)
async def get_post(post_id: int):
    if post_id not in posts_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Post not found"
        )
    
    # Increment views
    posts_db[post_id]['views'] += 1
    
    return posts_db[post_id]
```

### Database Integration (SQLAlchemy + Async)

```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship
from sqlalchemy import select, String, Text, Integer, DateTime, Boolean, ForeignKey
from datetime import datetime
from typing import List, Optional
from pydantic import BaseModel

# Database setup
DATABASE_URL = "sqlite+aiosqlite:///./test.db"

engine = create_async_engine(DATABASE_URL, echo=True)
AsyncSessionLocal = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

class Base(DeclarativeBase):
    pass

# Models
class User(Base):
    __tablename__ = "users"
    
    id: Mapped[int] = mapped_column(primary_key=True, index=True)
    username: Mapped[str] = mapped_column(String(50), unique=True, index=True)
    email: Mapped[str] = mapped_column(String(100), unique=True, index=True)
    hashed_password: Mapped[str] = mapped_column(String(255))
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
    
    posts: Mapped[List["Post"]] = relationship(back_populates="author")

class Post(Base):
    __tablename__ = "posts"
    
    id: Mapped[int] = mapped_column(primary_key=True, index=True)
    title: Mapped[str] = mapped_column(String(200))
    content: Mapped[str] = mapped_column(Text)
    published: Mapped[bool] = mapped_column(Boolean, default=False)
    views: Mapped[int] = mapped_column(Integer, default=0)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    
    author: Mapped["User"] = relationship(back_populates="posts")

# Pydantic schemas
class UserCreate(BaseModel):
    username: str
    email: str
    password: str

class UserOut(BaseModel):
    id: int
    username: str
    email: str
    is_active: bool
    created_at: datetime
    
    class Config:
        from_attributes = True

class PostCreate(BaseModel):
    title: str
    content: str
    published: bool = False

class PostOut(BaseModel):
    id: int
    title: str
    content: str
    published: bool
    views: int
    created_at: datetime
    author: UserOut
    
    class Config:
        from_attributes = True

# FastAPI app
app = FastAPI()

# Dependency
async def get_db():
    async with AsyncSessionLocal() as session:
        yield session

# Create tables
@app.on_event("startup")
async def startup():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

# Endpoints
@app.post("/users/", response_model=UserOut)
async def create_user(user: UserCreate, db: AsyncSession = Depends(get_db)):
    # Hash password (use proper hashing in production)
    hashed_password = f"hashed_{user.password}"
    
    db_user = User(
        username=user.username,
        email=user.email,
        hashed_password=hashed_password
    )
    db.add(db_user)
    await db.commit()
    await db.refresh(db_user)
    return db_user

@app.get("/users/", response_model=List[UserOut])
async def list_users(skip: int = 0, limit: int = 10, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).offset(skip).limit(limit))
    users = result.scalars().all()
    return users

@app.get("/users/{user_id}", response_model=UserOut)
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).where(User.id == user_id))
    user = result.scalar_one_or_none()
    
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    
    return user

@app.post("/posts/", response_model=PostOut)
async def create_post(post: PostCreate, author_id: int, db: AsyncSession = Depends(get_db)):
    db_post = Post(**post.dict(), author_id=author_id)
    db.add(db_post)
    await db.commit()
    await db.refresh(db_post)
    return db_post

@app.get("/posts/", response_model=List[PostOut])
async def list_posts(skip: int = 0, limit: int = 10, db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        select(Post).offset(skip).limit(limit)
    )
    posts = result.scalars().all()
    return posts
```

### Authentication with JWT

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jose import JWTError, jwt
from passlib.context import CryptContext
from datetime import datetime, timedelta
from pydantic import BaseModel
from typing import Optional

# pip install python-jose[cryptography] passlib[bcrypt]

app = FastAPI()

# Security configuration
SECRET_KEY = "your-secret-key-keep-it-secret"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

# Fake users database
fake_users_db = {
    "john": {
        "username": "john",
        "email": "john@example.com",
        "hashed_password": "$2b$12$EixZaYVK1fsbw1ZfbX3OXePaWxn96p36WQoeG6Lruj3vjPGga31lW",  # "secret"
        "disabled": False,
    }
}

# Models
class Token(BaseModel):
    access_token: str
    token_type: str

class TokenData(BaseModel):
    username: Optional[str] = None

class User(BaseModel):
    username: str
    email: Optional[str] = None
    disabled: Optional[bool] = None

class UserInDB(User):
    hashed_password: str

# Utility functions
def verify_password(plain_password, hashed_password):
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password):
    return pwd_context.hash(password)

def get_user(db, username: str):
    if username in db:
        user_dict = db[username]
        return UserInDB(**user_dict)

def authenticate_user(fake_db, username: str, password: str):
    user = get_user(fake_db, username)
    if not user:
        return False
    if not verify_password(password, user.hashed_password):
        return False
    return user

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None):
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=15)
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

async def get_current_user(token: str = Depends(oauth2_scheme)):
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
        token_data = TokenData(username=username)
    except JWTError:
        raise credentials_exception
    user = get_user(fake_users_db, username=token_data.username)
    if user is None:
        raise credentials_exception
    return user

async def get_current_active_user(current_user: User = Depends(get_current_user)):
    if current_user.disabled:
        raise HTTPException(status_code=400, detail="Inactive user")
    return current_user

# Endpoints
@app.post("/token", response_model=Token)
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    user = authenticate_user(fake_users_db, form_data.username, form_data.password)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    access_token_expires = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = create_access_token(
        data={"sub": user.username}, expires_delta=access_token_expires
    )
    return {"access_token": access_token, "token_type": "bearer"}

@app.get("/users/me", response_model=User)
async def read_users_me(current_user: User = Depends(get_current_active_user)):
    return current_user

@app.get("/protected")
async def protected_route(current_user: User = Depends(get_current_active_user)):
    return {"message": f"Hello {current_user.username}, this is a protected route!"}
```

### Background Tasks

```python
from fastapi import FastAPI, BackgroundTasks
from pydantic import BaseModel, EmailStr
import time

app = FastAPI()

class EmailSchema(BaseModel):
    email: EmailStr
    subject: str
    body: str

def send_email(email: str, subject: str, body: str):
    """Simulate sending email (slow operation)"""
    time.sleep(5)  # Simulate delay
    print(f"Email sent to {email}")
    print(f"Subject: {subject}")
    print(f"Body: {body}")

@app.post("/send-email/")
async def send_email_endpoint(
    email_data: EmailSchema,
    background_tasks: BackgroundTasks
):
    # Add task to background
    background_tasks.add_task(
        send_email,
        email_data.email,
        email_data.subject,
        email_data.body
    )
    
    return {"message": "Email will be sent in the background"}

@app.post("/send-notification/")
async def send_notification(
    user_id: int,
    message: str,
    background_tasks: BackgroundTasks
):
    # Multiple background tasks
    background_tasks.add_task(log_notification, user_id, message)
    background_tasks.add_task(update_user_stats, user_id)
    
    return {"message": "Notification scheduled"}

def log_notification(user_id: int, message: str):
    time.sleep(2)
    print(f"Notification logged for user {user_id}: {message}")

def update_user_stats(user_id: int):
    time.sleep(1)
    print(f"Stats updated for user {user_id}")
```

### WebSocket Support

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import List

app = FastAPI()

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

@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: int):
    await manager.connect(websocket)
    try:
        while True:
            data = await websocket.receive_text()
            await manager.send_personal_message(f"You wrote: {data}", websocket)
            await manager.broadcast(f"Client #{client_id} says: {data}")
    except WebSocketDisconnect:
        manager.disconnect(websocket)
        await manager.broadcast(f"Client #{client_id} left the chat")
```

### Middleware and CORS

```python
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.gzip import GZipMiddleware
import time

app = FastAPI()

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # In production, specify exact origins
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# GZip compression
app.add_middleware(GZipMiddleware, minimum_size=1000)

# Custom middleware
@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start_time = time.time()
    response = await call_next(request)
    process_time = time.time() - start_time
    response.headers["X-Process-Time"] = str(process_time)
    return response

@app.middleware("http")
async def log_requests(request: Request, call_next):
    print(f"Request: {request.method} {request.url}")
    response = await call_next(request)
    print(f"Response status: {response.status_code}")
    return response
```

---

## Other Notable Frameworks

### Bottle - Minimalist Framework

```python
from bottle import route, run, template, request, response

@route('/')
def index():
    return '<h1>Hello from Bottle!</h1>'

@route('/hello/<name>')
def hello(name):
    return template('<h1>Hello {{name}}!</h1>', name=name)

@route('/api/data', method='POST')
def api_data():
    data = request.json
    response.content_type = 'application/json'
    return {'received': data}

run(host='localhost', port=8080, debug=True)
```

### Tornado - Async Web Framework

```python
import tornado.ioloop
import tornado.web

class MainHandler(tornado.web.RequestHandler):
    def get(self):
        self.write("<h1>Hello from Tornado!</h1>")

class APIHandler(tornado.web.RequestHandler):
    def get(self):
        self.set_header("Content-Type", "application/json")
        self.write({"message": "API response"})
    
    def post(self):
        data = tornado.escape.json_decode(self.request.body)
        self.write({"received": data})

def make_app():
    return tornado.web.Application([
        (r"/", MainHandler),
        (r"/api", APIHandler),
    ])

if __name__ == "__main__":
    app = make_app()
    app.listen(8888)
    print("Server running on http://localhost:8888")
    tornado.ioloop.IOLoop.current().start()
```

### Pyramid - Flexible Framework

```python
from wsgiref.simple_server import make_server
from pyramid.config import Configurator
from pyramid.response import Response
from pyramid.view import view_config

@view_config(route_name='home')
def home(request):
    return Response('<h1>Hello from Pyramid!</h1>')

@view_config(route_name='api', renderer='json')
def api(request):
    return {'message': 'API response', 'status': 'success'}

if __name__ == '__main__':
    with Configurator() as config:
        config.add_route('home', '/')
        config.add_route('api', '/api')
        config.scan()
        app = config.make_wsgi_app()
    
    server = make_server('0.0.0.0', 6543, app)
    print('Server running on http://localhost:6543')
    server.serve_forever()
```

### Sanic - Async Performance

```python
from sanic import Sanic, response

app = Sanic("MyApp")

@app.route("/")
async def index(request):
    return response.html('<h1>Hello from Sanic!</h1>')

@app.route("/api/data", methods=["GET", "POST"])
async def api_data(request):
    if request.method == "POST":
        data = request.json
        return response.json({"received": data})
    return response.json({"message": "Send POST request"})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000, debug=True)
```

---

## Framework Comparison

| Feature | Flask | Django | FastAPI | Bottle | Pyramid |
|---------|-------|--------|---------|--------|---------|
| **Type** | Micro | Full-stack | Async API | Micro | Flexible |
| **Learning Curve** | Easy | Moderate | Easy | Easy | Moderate |
| **Async Support** | Limited | Limited | Native | No | Limited |
| **ORM** | Extensions | Built-in | Extensions | None | Extensions |
| **Admin Panel** | Extensions | Built-in | No | No | Extensions |
| **API Development** | Good | Good | Excellent | Basic | Good |
| **Documentation** | Excellent | Excellent | Excellent | Good | Good |
| **Performance** | Good | Good | Excellent | Good | Good |
| **Flexibility** | High | Low | High | High | Very High |
| **Best For** | Small-Medium | Large apps | APIs | Tiny apps | Complex apps |

### Performance Comparison

Based on typical benchmarks:

1. **FastAPI** - Fastest (async, modern)
2. **Sanic** - Very fast (async)
3. **Flask** - Good (sync, lightweight)
4. **Bottle** - Good (sync, minimal)
5. **Django** - Good (full-featured, more overhead)

---

## Choosing the Right Framework

### Choose Flask if:
- Building a small to medium application
- Need flexibility and simplicity
- Want to learn web development fundamentals
- Prefer choosing your own tools and libraries
- Building a RESTful API with moderate complexity

### Choose Django if:
- Building a large, complex application
- Need admin interface out of the box
- Want built-in authentication and authorization
- Prefer convention over configuration
- Building a content management system
- Need rapid development with many features

### Choose FastAPI if:
- Building modern REST APIs
- Need automatic API documentation
- Want async/await support
- Require data validation with type hints
- Building microservices
- Need WebSocket support
- Want high performance

### Choose Bottle/Pyramid/Sanic if:
- **Bottle**: Prototyping or single-file applications
- **Pyramid**: Need maximum flexibility and scalability
- **Sanic**: Need async performance without FastAPI's features

---

## Building a Complete Application

### Todo App with Flask

```python
# app.py
from flask import Flask, render_template, request, redirect, url_for, flash
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime

app = Flask(__name__)
app.config['SECRET_KEY'] = 'your-secret-key'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///todos.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

db = SQLAlchemy(app)

# Model
class Todo(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200), nullable=False)
    description = db.Column(db.Text)
    completed = db.Column(db.Boolean, default=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    def __repr__(self):
        return f'<Todo {self.title}>'

# Create tables
with app.app_context():
    db.create_all()

# Routes
@app.route('/')
def index():
    todos = Todo.query.order_by(Todo.created_at.desc()).all()
    return render_template('index.html', todos=todos)

@app.route('/add', methods=['POST'])
def add():
    title = request.form.get('title')
    description = request.form.get('description')
    
    if title:
        todo = Todo(title=title, description=description)
        db.session.add(todo)
        db.session.commit()
        flash('Todo added successfully!', 'success')
    else:
        flash('Title is required!', 'error')
    
    return redirect(url_for('index'))

@app.route('/toggle/<int:id>')
def toggle(id):
    todo = Todo.query.get_or_404(id)
    todo.completed = not todo.completed
    db.session.commit()
    flash(f'Todo {"completed" if todo.completed else "uncompleted"}!', 'success')
    return redirect(url_for('index'))

@app.route('/delete/<int:id>')
def delete(id):
    todo = Todo.query.get_or_404(id)
    db.session.delete(todo)
    db.session.commit()
    flash('Todo deleted!', 'success')
    return redirect(url_for('index'))

@app.route('/edit/<int:id>', methods=['GET', 'POST'])
def edit(id):
    todo = Todo.query.get_or_404(id)
    
    if request.method == 'POST':
        todo.title = request.form.get('title')
        todo.description = request.form.get('description')
        db.session.commit()
        flash('Todo updated!', 'success')
        return redirect(url_for('index'))
    
    return render_template('edit.html', todo=todo)

if __name__ == '__main__':
    app.run(debug=True)
```

**templates/base.html:**
```html
<!DOCTYPE html>
<html>
<head>
    <title>Todo App</title>
    <style>
        body { font-family: Arial, sans-serif; max-width: 800px; margin: 0 auto; padding: 20px; }
        .todo { border: 1px solid #ddd; padding: 15px; margin: 10px 0; border-radius: 5px; }
        .todo.completed { opacity: 0.6; text-decoration: line-through; }
        .flash { padding: 10px; margin: 10px 0; border-radius: 5px; }
        .flash.success { background: #d4edda; color: #155724; }
        .flash.error { background: #f8d7da; color: #721c24; }
        button, input[type="submit"] { cursor: pointer; padding: 8px 15px; margin: 5px; }
    </style>
</head>
<body>
    {% with messages = get_flashed_messages(with_categories=true) %}
        {% if messages %}
            {% for category, message in messages %}
                <div class="flash {{ category }}">{{ message }}</div>
            {% endfor %}
        {% endif %}
    {% endwith %}
    
    {% block content %}{% endblock %}
</body>
</html>
```

**templates/index.html:**
```html
{% extends 'base.html' %}

{% block content %}
    <h1>Todo List</h1>
    
    <form method="POST" action="{{ url_for('add') }}">
        <input type="text" name="title" placeholder="Todo title" required>
        <textarea name="description" placeholder="Description"></textarea>
        <button type="submit">Add Todo</button>
    </form>
    
    <h2>My Todos</h2>
    {% for todo in todos %}
        <div class="todo {% if todo.completed %}completed{% endif %}">
            <h3>{{ todo.title }}</h3>
            <p>{{ todo.description }}</p>
            <small>Created: {{ todo.created_at.strftime('%Y-%m-%d %H:%M') }}</small>
            <div>
                <a href="{{ url_for('toggle', id=todo.id) }}">
                    <button>{% if todo.completed %}Undo{% else %}Complete{% endif %}</button>
                </a>
                <a href="{{ url_for('edit', id=todo.id) }}">
                    <button>Edit</button>
                </a>
                <a href="{{ url_for('delete', id=todo.id) }}" onclick="return confirm('Are you sure?')">
                    <button>Delete</button>
                </a>
            </div>
        </div>
    {% else %}
        <p>No todos yet. Add one above!</p>
    {% endfor %}
{% endblock %}
```

### Complete API with FastAPI

```python
# main.py - Complete RESTful API
from fastapi import FastAPI, HTTPException, Depends, status
from fastapi.middleware.cors import CORSMiddleware
from sqlalchemy import create_engine, Column, Integer, String, Boolean, DateTime
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, Session
from pydantic import BaseModel, Field
from typing import List, Optional
from datetime import datetime

# Database setup
SQLALCHEMY_DATABASE_URL = "sqlite:///./todos.db"
engine = create_engine(SQLALCHEMY_DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

# Database model
class TodoDB(Base):
    __tablename__ = "todos"
    
    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(200), nullable=False)
    description = Column(String, nullable=True)
    completed = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.utcnow)

Base.metadata.create_all(bind=engine)

# Pydantic schemas
class TodoBase(BaseModel):
    title: str = Field(..., min_length=1, max_length=200)
    description: Optional[str] = None
    completed: bool = False

class TodoCreate(TodoBase):
    pass

class TodoUpdate(BaseModel):
    title: Optional[str] = Field(None, min_length=1, max_length=200)
    description: Optional[str] = None
    completed: Optional[bool] = None

class TodoResponse(TodoBase):
    id: int
    created_at: datetime
    
    class Config:
        from_attributes = True

# FastAPI app
app = FastAPI(
    title="Todo API",
    description="A simple Todo API with FastAPI",
    version="1.0.0"
)

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Dependency
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# Endpoints
@app.get("/", tags=["Root"])
async def root():
    return {
        "message": "Welcome to Todo API",
        "docs": "/docs",
        "redoc": "/redoc"
    }

@app.post("/todos/", response_model=TodoResponse, status_code=status.HTTP_201_CREATED, tags=["Todos"])
async def create_todo(todo: TodoCreate, db: Session = Depends(get_db)):
    """Create a new todo item"""
    db_todo = TodoDB(**todo.dict())
    db.add(db_todo)
    db.commit()
    db.refresh(db_todo)
    return db_todo

@app.get("/todos/", response_model=List[TodoResponse], tags=["Todos"])
async def list_todos(
    skip: int = 0,
    limit: int = 100,
    completed: Optional[bool] = None,
    db: Session = Depends(get_db)
):
    """Get list of todos with optional filtering"""
    query = db.query(TodoDB)
    
    if completed is not None:
        query = query.filter(TodoDB.completed == completed)
    
    todos = query.offset(skip).limit(limit).all()
    return todos

@app.get("/todos/{todo_id}", response_model=TodoResponse, tags=["Todos"])
async def get_todo(todo_id: int, db: Session = Depends(get_db)):
    """Get a specific todo by ID"""
    todo = db.query(TodoDB).filter(TodoDB.id == todo_id).first()
    
    if todo is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Todo with id {todo_id} not found"
        )
    
    return todo

@app.put("/todos/{todo_id}", response_model=TodoResponse, tags=["Todos"])
async def update_todo(
    todo_id: int,
    todo_update: TodoUpdate,
    db: Session = Depends(get_db)
):
    """Update a todo item"""
    todo = db.query(TodoDB).filter(TodoDB.id == todo_id).first()
    
    if todo is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Todo with id {todo_id} not found"
        )
    
    update_data = todo_update.dict(exclude_unset=True)
    for field, value in update_data.items():
        setattr(todo, field, value)
    
    db.commit()
    db.refresh(todo)
    return todo

@app.delete("/todos/{todo_id}", status_code=status.HTTP_204_NO_CONTENT, tags=["Todos"])
async def delete_todo(todo_id: int, db: Session = Depends(get_db)):
    """Delete a todo item"""
    todo = db.query(TodoDB).filter(TodoDB.id == todo_id).first()
    
    if todo is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Todo with id {todo_id} not found"
        )
    
    db.delete(todo)
    db.commit()
    return None

@app.patch("/todos/{todo_id}/toggle", response_model=TodoResponse, tags=["Todos"])
async def toggle_todo(todo_id: int, db: Session = Depends(get_db)):
    """Toggle the completed status of a todo"""
    todo = db.query(TodoDB).filter(TodoDB.id == todo_id).first()
    
    if todo is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Todo with id {todo_id} not found"
        )
    
    todo.completed = not todo.completed
    db.commit()
    db.refresh(todo)
    return todo

@app.get("/todos/stats/summary", tags=["Statistics"])
async def get_statistics(db: Session = Depends(get_db)):
    """Get todo statistics"""
    total = db.query(TodoDB).count()
    completed = db.query(TodoDB).filter(TodoDB.completed == True).count()
    pending = total - completed
    
    return {
        "total": total,
        "completed": completed,
        "pending": pending,
        "completion_rate": (completed / total * 100) if total > 0 else 0
    }

# Run with: uvicorn main:app --reload
```

---

## Best Practices for Web Frameworks

### 1. Project Structure

**Flask Structure:**
```
myproject/
├── app/
│   ├── __init__.py
│   ├── models.py
│   ├── views.py
│   ├── forms.py
│   ├── templates/
│   └── static/
├── tests/
├── config.py
├── requirements.txt
└── run.py
```

**Django Structure:**
```
myproject/
├── myproject/
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── apps/
│   ├── users/
│   ├── posts/
│   └── api/
├── templates/
├── static/
├── manage.py
└── requirements.txt
```

**FastAPI Structure:**
```
myproject/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── models/
│   ├── schemas/
│   ├── routers/
│   ├── dependencies.py
│   └── database.py
├── tests/
├── alembic/
├── requirements.txt
└── .env
```

### 2. Configuration Management

```python
# config.py
import os
from dotenv import load_dotenv

load_dotenv()

class Config:
    SECRET_KEY = os.getenv('SECRET_KEY', 'dev-secret-key')
    SQLALCHEMY_DATABASE_URI = os.getenv('DATABASE_URL', 'sqlite:///app.db')
    SQLALCHEMY_TRACK_MODIFICATIONS = False

class DevelopmentConfig(Config):
    DEBUG = True

class ProductionConfig(Config):
    DEBUG = False

class TestingConfig(Config):
    TESTING = True
    SQLALCHEMY_DATABASE_URI = 'sqlite:///:memory:'

config = {
    'development': DevelopmentConfig,
    'production': ProductionConfig,
    'testing': TestingConfig,
    'default': DevelopmentConfig
}
```

### 3. Error Handling

```python
# Flask
@app.errorhandler(404)
def not_found(error):
    return jsonify({'error': 'Not found'}), 404

@app.errorhandler(500)
def internal_error(error):
    db.session.rollback()
    return jsonify({'error': 'Internal server error'}), 500

# FastAPI
from fastapi import Request
from fastapi.responses import JSONResponse

@app.exception_handler(ValueError)
async def value_error_handler(request: Request, exc: ValueError):
    return JSONResponse(
        status_code=400,
        content={"error": str(exc)}
    )
```

### 4. Testing

```python
# Flask testing
import unittest
from app import app, db

class TestApp(unittest.TestCase):
    def setUp(self):
        app.config['TESTING'] = True
        app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///:memory:'
        self.app = app.test_client()
        with app.app_context():
            db.create_all()
    
    def tearDown(self):
        with app.app_context():
            db.session.remove()
            db.drop_all()
    
    def test_index(self):
        response = self.app.get('/')
        self.assertEqual(response.status_code, 200)

# FastAPI testing
from fastapi.testclient import TestClient

client = TestClient(app)

def test_read_root():
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == {"message": "Hello World"}

def test_create_todo():
    response = client.post(
        "/todos/",
        json={"title": "Test Todo", "description": "Test"}
    )
    assert response.status_code == 201
    assert response.json()["title"] == "Test Todo"
```

### 5. Security Best Practices

- Use environment variables for sensitive data
- Implement CSRF protection
- Use parameterized queries (prevent SQL injection)
- Validate and sanitize all user input
- Use HTTPS in production
- Implement rate limiting
- Keep dependencies updated
- Use security headers
- Hash passwords properly (bcrypt, argon2)
- Implement proper authentication and authorization

---

## Summary

### Key Takeaways

**Flask:**
- Lightweight and flexible
- Great for learning and small to medium projects
- Large ecosystem of extensions
- Minimal boilerplate

**Django:**
- Full-featured and batteries-included
- Built-in admin interface and ORM
- Best for large applications
- Strong conventions and structure

**FastAPI:**
- Modern and high-performance
- Automatic API documentation
- Native async support
- Type hints and validation
- Perfect for APIs and microservices

### When to Use Each

**Start with Flask if:**
- You're learning web development
- Need a simple API or web app
- Want maximum flexibility

**Start with Django if:**
- Building a complex web application
- Need rapid development with many features
- Want a complete solution out of the box

**Start with FastAPI if:**
- Building REST APIs
- Need async capabilities
- Want modern Python features
- Need automatic documentation

### Final Recommendations

1. **Learn the fundamentals** with Flask first
2. **Understand HTTP and web concepts** before diving deep
3. **Choose based on project requirements**, not hype
4. **Follow framework conventions** for better maintainability
5. **Write tests** from the beginning
6. **Use version control** (Git) for all projects
7. **Deploy early and often** to catch production issues
8. **Read documentation** - all three frameworks have excellent docs
9. **Join communities** for support and learning
10. **Build real projects** to solidify your knowledge

Web frameworks are tools - master the concepts, and you can use any framework effectively!