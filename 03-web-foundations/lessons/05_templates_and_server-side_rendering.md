# Templates and Server-Side Rendering with Python

## Table of Contents
1. [Introduction to Templates](#introduction-to-templates)
2. [Template Engines Overview](#template-engines-overview)
3. [Jinja2 Template Engine](#jinja2-template-engine)
4. [Flask Templates](#flask-templates)
5. [Django Templates](#django-templates)
6. [Template Inheritance](#template-inheritance)
7. [Template Filters and Functions](#template-filters-and-functions)
8. [Context and Data Passing](#context-and-data-passing)
9. [Template Best Practices](#template-best-practices)
10. [Server-Side vs Client-Side Rendering](#server-side-vs-client-side-rendering)

---

## Introduction to Templates

**Templates** are files that contain static HTML markup combined with special syntax for dynamic content. They allow you to separate presentation (HTML) from business logic (Python code).

### Why Use Templates?

1. **Separation of Concerns**: Keep HTML separate from Python code
2. **Reusability**: Create reusable components and layouts
3. **Maintainability**: Easier to update design without touching logic
4. **DRY Principle**: Don't Repeat Yourself with template inheritance
5. **Designer-Friendly**: Non-programmers can work with templates
6. **Security**: Auto-escaping prevents XSS attacks

### Server-Side Rendering (SSR)

**SSR** means generating HTML on the server before sending it to the client.

**Advantages:**
- Better SEO (search engines see full content)
- Faster initial page load
- Works without JavaScript
- Better for accessibility
- Simpler state management

**Disadvantages:**
- More server resources
- Full page reloads
- Less interactive than SPAs
- Slower subsequent navigation

---

## Template Engines Overview

### Popular Python Template Engines

| Engine | Used By | Features | Syntax |
|--------|---------|----------|--------|
| **Jinja2** | Flask, Ansible | Fast, powerful, extensible | `{{ }}`, `{% %}` |
| **Django Templates** | Django | Integrated, secure | `{{ }}`, `{% %}` |
| **Mako** | Pyramid | Fast, Python expressions | `${ }`, `<% %>` |
| **Chameleon** | Pyramid | XML-based, fast | `tal:`, `metal:` |
| **Genshi** | Various | XML/HTML-based | Pythonic |

**We'll focus on Jinja2 and Django Templates as they're most widely used.**

---

## Jinja2 Template Engine

Jinja2 is a modern, designer-friendly template engine for Python. It's the default for Flask and can be used standalone.

### Installation

```bash
pip install jinja2
```

### Basic Jinja2 Syntax

```jinja2
{# This is a comment #}

{# Variables #}
{{ variable }}
{{ user.name }}
{{ items[0] }}
{{ dict['key'] }}

{# Expressions #}
{{ 2 + 2 }}
{{ "Hello " + name }}
{{ price * 1.1 }}

{# Control Structures #}
{% if condition %}
    <p>True</p>
{% elif other_condition %}
    <p>Maybe</p>
{% else %}
    <p>False</p>
{% endif %}

{% for item in items %}
    <li>{{ item }}</li>
{% endfor %}

{# Filters #}
{{ name|upper }}
{{ price|round(2) }}
{{ text|truncate(100) }}

{# Tests #}
{% if user is defined %}
{% if number is even %}
{% if value is none %}

{# Block definitions #}
{% block content %}
    Default content
{% endblock %}
```

### Standalone Jinja2 Example

```python
from jinja2 import Template

# Simple template
template = Template('Hello {{ name }}!')
output = template.render(name='World')
print(output)  # Hello World!

# Template with control structures
template = Template('''
<ul>
{% for item in items %}
    <li>{{ item }}</li>
{% endfor %}
</ul>
''')

output = template.render(items=['Apple', 'Banana', 'Cherry'])
print(output)

# Template with conditions
template = Template('''
{% if user.is_admin %}
    <p>Welcome, Admin {{ user.name }}!</p>
{% else %}
    <p>Welcome, {{ user.name }}!</p>
{% endif %}
''')

output = template.render(user={'name': 'John', 'is_admin': True})
print(output)
```

### Jinja2 Environment

```python
from jinja2 import Environment, FileSystemLoader, select_autoescape

# Create environment
env = Environment(
    loader=FileSystemLoader('templates'),
    autoescape=select_autoescape(['html', 'xml']),
    trim_blocks=True,
    lstrip_blocks=True
)

# Load and render template
template = env.get_template('page.html')
output = template.render(title='My Page', content='Hello World')

# Custom filters
def reverse_string(s):
    return s[::-1]

env.filters['reverse'] = reverse_string

# Usage in template: {{ "hello"|reverse }}
```

---

## Flask Templates

Flask uses Jinja2 as its template engine with some additional features.

### Basic Flask Template Setup

```python
# app.py
from flask import Flask, render_template

app = Flask(__name__)

@app.route('/')
def index():
    return render_template('index.html', 
                         title='Home Page',
                         user='John')

@app.route('/about')
def about():
    return render_template('about.html',
                         title='About Us')

if __name__ == '__main__':
    app.run(debug=True)
```

**Directory Structure:**
```
project/
├── app.py
├── templates/
│   ├── base.html
│   ├── index.html
│   └── about.html
└── static/
    ├── css/
    ├── js/
    └── images/
```

### Simple Template Example

```html
<!-- templates/index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ title }}</title>
</head>
<body>
    <h1>Welcome, {{ user }}!</h1>
    <p>This is the home page.</p>
</body>
</html>
```

### Variables and Expressions

```python
# app.py
@app.route('/demo')
def demo():
    return render_template('demo.html',
        name='Alice',
        age=25,
        hobbies=['Reading', 'Coding', 'Gaming'],
        user={
            'username': 'alice',
            'email': 'alice@example.com',
            'is_admin': True
        },
        products=[
            {'name': 'Laptop', 'price': 999.99},
            {'name': 'Mouse', 'price': 29.99},
            {'name': 'Keyboard', 'price': 79.99}
        ]
    )
```

```html
<!-- templates/demo.html -->
<!DOCTYPE html>
<html>
<head>
    <title>Demo</title>
</head>
<body>
    {# Simple variable #}
    <h1>Hello, {{ name }}!</h1>
    
    {# Expression #}
    <p>Age: {{ age }} (Next year: {{ age + 1 }})</p>
    
    {# List access #}
    <p>First hobby: {{ hobbies[0] }}</p>
    
    {# Dictionary access #}
    <p>Username: {{ user.username }}</p>
    <p>Email: {{ user['email'] }}</p>
    
    {# List iteration #}
    <h2>Hobbies:</h2>
    <ul>
    {% for hobby in hobbies %}
        <li>{{ hobby }}</li>
    {% endfor %}
    </ul>
    
    {# List of dictionaries #}
    <h2>Products:</h2>
    <table>
        <tr>
            <th>Product</th>
            <th>Price</th>
        </tr>
        {% for product in products %}
        <tr>
            <td>{{ product.name }}</td>
            <td>${{ product.price }}</td>
        </tr>
        {% endfor %}
    </table>
    
    {# Conditional #}
    {% if user.is_admin %}
        <p>⭐ Admin Access Granted</p>
    {% else %}
        <p>Regular User</p>
    {% endif %}
</body>
</html>
```

### Control Structures

```html
<!-- Conditionals -->
{% if user.is_authenticated %}
    <p>Welcome back, {{ user.name }}!</p>
{% elif user.is_guest %}
    <p>Welcome, Guest!</p>
{% else %}
    <p>Please log in</p>
{% endif %}

<!-- For loops -->
<ul>
{% for item in items %}
    <li>{{ loop.index }}: {{ item.name }}</li>
{% endfor %}
</ul>

<!-- Loop variables -->
{% for user in users %}
    <div class="user {% if loop.first %}first{% endif %} {% if loop.last %}last{% endif %}">
        <p>{{ loop.index }} of {{ loop.length }}: {{ user.name }}</p>
    </div>
{% endfor %}

<!-- Empty check -->
<ul>
{% for item in items %}
    <li>{{ item }}</li>
{% else %}
    <li>No items found</li>
{% endfor %}
</ul>

<!-- Loop filtering -->
{% for user in users if user.is_active %}
    <li>{{ user.name }}</li>
{% endfor %}

<!-- Nested loops -->
{% for category in categories %}
    <h3>{{ category.name }}</h3>
    <ul>
    {% for product in category.products %}
        <li>{{ product.name }}</li>
    {% endfor %}
    </ul>
{% endfor %}
```

### Loop Variables

```html
{% for item in items %}
    {{ loop.index }}      {# Current iteration (1-indexed) #}
    {{ loop.index0 }}     {# Current iteration (0-indexed) #}
    {{ loop.revindex }}   {# Iterations from end (1-indexed) #}
    {{ loop.revindex0 }}  {# Iterations from end (0-indexed) #}
    {{ loop.first }}      {# True if first iteration #}
    {{ loop.last }}       {# True if last iteration #}
    {{ loop.length }}     {# Total number of items #}
    {{ loop.cycle('odd', 'even') }}  {# Cycle through values #}
{% endfor %}
```

### Macros (Reusable Components)

```html
<!-- templates/macros.html -->
{% macro render_input(name, label, type='text', value='') %}
<div class="form-group">
    <label for="{{ name }}">{{ label }}</label>
    <input type="{{ type }}" 
           name="{{ name }}" 
           id="{{ name }}" 
           value="{{ value }}"
           class="form-control">
</div>
{% endmacro %}

{% macro render_button(text, type='button', class='btn-primary') %}
<button type="{{ type }}" class="btn {{ class }}">
    {{ text }}
</button>
{% endmacro %}

{% macro render_card(title, content, footer='') %}
<div class="card">
    <div class="card-header">
        <h3>{{ title }}</h3>
    </div>
    <div class="card-body">
        {{ content }}
    </div>
    {% if footer %}
    <div class="card-footer">
        {{ footer }}
    </div>
    {% endif %}
</div>
{% endmacro %}
```

```html
<!-- templates/form.html -->
{% from 'macros.html' import render_input, render_button %}

<form method="POST">
    {{ render_input('username', 'Username') }}
    {{ render_input('email', 'Email', type='email') }}
    {{ render_input('password', 'Password', type='password') }}
    {{ render_button('Submit', type='submit') }}
    {{ render_button('Cancel', class='btn-secondary') }}
</form>
```

### Including Templates

```html
<!-- templates/header.html -->
<header>
    <nav>
        <a href="/">Home</a>
        <a href="/about">About</a>
        <a href="/contact">Contact</a>
    </nav>
</header>

<!-- templates/footer.html -->
<footer>
    <p>&copy; 2024 My Website. All rights reserved.</p>
</footer>

<!-- templates/page.html -->
<!DOCTYPE html>
<html>
<head>
    <title>{{ title }}</title>
</head>
<body>
    {% include 'header.html' %}
    
    <main>
        <h1>{{ title }}</h1>
        {{ content }}
    </main>
    
    {% include 'footer.html' %}
</body>
</html>
```

### Static Files in Flask

```html
<!-- templates/page.html -->
<!DOCTYPE html>
<html>
<head>
    <title>{{ title }}</title>
    
    {# CSS files #}
    <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
    
    {# Favicon #}
    <link rel="icon" href="{{ url_for('static', filename='favicon.ico') }}">
</head>
<body>
    {# Images #}
    <img src="{{ url_for('static', filename='images/logo.png') }}" alt="Logo">
    
    {# JavaScript files #}
    <script src="{{ url_for('static', filename='js/script.js') }}"></script>
</body>
</html>
```

---

## Django Templates

Django has its own template engine, similar to Jinja2 but with some differences.

### Django Template Setup

```python
# settings.py
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]

# views.py
from django.shortcuts import render

def index(request):
    context = {
        'title': 'Home Page',
        'user': request.user,
        'items': ['Item 1', 'Item 2', 'Item 3']
    }
    return render(request, 'index.html', context)
```

### Django Template Syntax

```django
{# Comments #}

{# Variables #}
{{ variable }}
{{ user.username }}
{{ items.0 }}

{# Filters #}
{{ name|lower }}
{{ text|truncatewords:30 }}
{{ date|date:"Y-m-d" }}

{# Tags #}
{% if condition %}
    <p>True</p>
{% endif %}

{% for item in items %}
    <li>{{ item }}</li>
{% endfor %}

{# Template inheritance #}
{% extends "base.html" %}
{% block content %}
    Content here
{% endblock %}

{# Include #}
{% include "header.html" %}

{# Load custom tags #}
{% load static %}
{% load custom_tags %}

{# URL generation #}
{% url 'view_name' %}
{% url 'view_name' arg1 arg2 %}

{# Static files #}
{% load static %}
<img src="{% static 'images/logo.png' %}">

{# CSRF token #}
<form method="post">
    {% csrf_token %}
    <!-- form fields -->
</form>
```

### Django Template Example

```python
# views.py
from django.shortcuts import render

def blog_post(request, post_id):
    post = {
        'id': post_id,
        'title': 'My Blog Post',
        'author': 'John Doe',
        'content': 'This is the post content...',
        'created_at': '2024-01-15',
        'tags': ['python', 'django', 'web'],
        'comments': [
            {'author': 'Alice', 'text': 'Great post!'},
            {'author': 'Bob', 'text': 'Thanks for sharing!'}
        ]
    }
    return render(request, 'blog_post.html', {'post': post})
```

```django
<!-- templates/blog_post.html -->
{% extends "base.html" %}
{% load static %}

{% block title %}{{ post.title }}{% endblock %}

{% block content %}
<article class="blog-post">
    <h1>{{ post.title }}</h1>
    
    <div class="meta">
        <span>By {{ post.author }}</span>
        <span>{{ post.created_at|date:"F d, Y" }}</span>
    </div>
    
    <div class="content">
        {{ post.content|linebreaks }}
    </div>
    
    <div class="tags">
        {% for tag in post.tags %}
            <span class="tag">{{ tag }}</span>
        {% endfor %}
    </div>
    
    <section class="comments">
        <h2>Comments ({{ post.comments|length }})</h2>
        {% for comment in post.comments %}
            <div class="comment">
                <strong>{{ comment.author }}</strong>
                <p>{{ comment.text }}</p>
            </div>
        {% empty %}
            <p>No comments yet.</p>
        {% endfor %}
    </section>
</article>
{% endblock %}
```

### Django vs Jinja2 Differences

| Feature | Django | Jinja2 |
|---------|--------|--------|
| **Method calls** | `{{ user.get_name }}` | `{{ user.get_name() }}` |
| **Comparison** | `{% if a == b %}` | `{% if a == b %}` |
| **Logic** | Limited | More Python-like |
| **Custom tags** | Via template tags | Via extensions |
| **Auto-escape** | On by default | Configurable |
| **Performance** | Good | Faster |

---

## Template Inheritance

Template inheritance allows you to create a base layout and extend it in child templates.

### Base Template Pattern

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    
    <title>{% block title %}Default Title{% endblock %}</title>
    
    {% block meta %}{% endblock %}
    
    <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
    {% block styles %}{% endblock %}
</head>
<body>
    <header>
        <nav>
            <div class="logo">My Website</div>
            <ul>
                <li><a href="/">Home</a></li>
                <li><a href="/about">About</a></li>
                <li><a href="/blog">Blog</a></li>
                <li><a href="/contact">Contact</a></li>
            </ul>
        </nav>
    </header>
    
    <main>
        {% block content %}
            <p>Default content</p>
        {% endblock %}
    </main>
    
    <aside>
        {% block sidebar %}{% endblock %}
    </aside>
    
    <footer>
        {% block footer %}
            <p>&copy; 2024 My Website</p>
        {% endblock %}
    </footer>
    
    <script src="{{ url_for('static', filename='js/main.js') }}"></script>
    {% block scripts %}{% endblock %}
</body>
</html>
```

### Child Template

```html
<!-- templates/blog_list.html -->
{% extends "base.html" %}

{% block title %}Blog Posts{% endblock %}

{% block meta %}
    <meta name="description" content="List of blog posts">
{% endblock %}

{% block styles %}
    <link rel="stylesheet" href="{{ url_for('static', filename='css/blog.css') }}">
{% endblock %}

{% block content %}
    <h1>Blog Posts</h1>
    
    {% for post in posts %}
        <article class="post-preview">
            <h2><a href="/blog/{{ post.id }}">{{ post.title }}</a></h2>
            <p class="meta">{{ post.author }} - {{ post.date }}</p>
            <p>{{ post.excerpt }}</p>
        </article>
    {% endfor %}
    
    {% if posts|length == 0 %}
        <p>No blog posts yet.</p>
    {% endif %}
{% endblock %}

{% block sidebar %}
    <h3>Categories</h3>
    <ul>
        {% for category in categories %}
            <li><a href="/blog/category/{{ category.slug }}">{{ category.name }}</a></li>
        {% endfor %}
    </ul>
{% endblock %}

{% block scripts %}
    <script src="{{ url_for('static', filename='js/blog.js') }}"></script>
{% endblock %}
```

### Multi-Level Inheritance

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html>
<head>
    <title>{% block title %}{% endblock %}</title>
</head>
<body>
    {% block body %}{% endblock %}
</body>
</html>

<!-- templates/dashboard_base.html -->
{% extends "base.html" %}

{% block body %}
    <div class="dashboard">
        <aside class="sidebar">
            {% block sidebar %}{% endblock %}
        </aside>
        <main class="content">
            {% block content %}{% endblock %}
        </main>
    </div>
{% endblock %}

<!-- templates/dashboard_analytics.html -->
{% extends "dashboard_base.html" %}

{% block title %}Analytics Dashboard{% endblock %}

{% block sidebar %}
    <ul>
        <li><a href="/dashboard">Overview</a></li>
        <li><a href="/dashboard/analytics">Analytics</a></li>
        <li><a href="/dashboard/reports">Reports</a></li>
    </ul>
{% endblock %}

{% block content %}
    <h1>Analytics</h1>
    <!-- Analytics content -->
{% endblock %}
```

### Using super()

```html
<!-- templates/base.html -->
{% block scripts %}
    <script src="{{ url_for('static', filename='js/jquery.js') }}"></script>
    <script src="{{ url_for('static', filename='js/main.js') }}"></script>
{% endblock %}

<!-- templates/page.html -->
{% extends "base.html" %}

{% block scripts %}
    {{ super() }}  {# Include parent block content #}
    <script src="{{ url_for('static', filename='js/page-specific.js') }}"></script>
{% endblock %}

<!-- Rendered output will include all three scripts -->
```

---

## Template Filters and Functions

### Built-in Jinja2 Filters

```html
<!-- String filters -->
{{ "hello world"|upper }}           {# HELLO WORLD #}
{{ "HELLO WORLD"|lower }}           {# hello world #}
{{ "hello world"|title }}           {# Hello World #}
{{ "hello world"|capitalize }}      {# Hello world #}
{{ "  hello  "|trim }}              {# hello #}
{{ "hello"|center(20) }}            {# Centered text #}

<!-- Truncation -->
{{ long_text|truncate(100) }}       {# First 100 chars... #}
{{ long_text|truncate(100, True) }} {# Word-aware truncate #}
{{ items|truncate(5) }}             {# First 5 items #}

<!-- Default values -->
{{ variable|default('N/A') }}
{{ user.name|default('Anonymous') }}

<!-- Formatting -->
{{ price|round(2) }}                {# 19.99 #}
{{ number|int }}                    {# Convert to integer #}
{{ value|float }}                   {# Convert to float #}
{{ text|replace('old', 'new') }}

<!-- Lists and iteration -->
{{ items|length }}                  {# Number of items #}
{{ items|first }}                   {# First item #}
{{ items|last }}                    {# Last item #}
{{ items|join(', ') }}              {# Join with separator #}
{{ items|reverse }}                 {# Reverse list #}
{{ items|sort }}                    {# Sort list #}
{{ items|unique }}                  {# Remove duplicates #}

<!-- Safety and escaping -->
{{ html_content|safe }}             {# Don't escape HTML #}
{{ user_input|escape }}             {# Escape HTML #}
{{ url|urlencode }}                 {# URL encode #}

<!-- Formatting -->
{{ content|wordcount }}             {# Count words #}
{{ content|striptags }}             {# Remove HTML tags #}
```

### Django Template Filters

```django
<!-- String filters -->
{{ value|lower }}
{{ value|upper }}
{{ value|title }}
{{ value|capfirst }}
{{ value|slugify }}

<!-- Truncation -->
{{ value|truncatewords:10 }}
{{ value|truncatechars:50 }}

<!-- Lists -->
{{ value|length }}
{{ value|first }}
{{ value|last }}
{{ value|join:", " }}
{{ value|slice:":5" }}

<!-- Numbers -->
{{ value|add:5 }}
{{ value|floatformat:2 }}

<!-- Dates -->
{{ date|date:"Y-m-d" }}
{{ date|date:"F j, Y" }}
{{ date|time:"H:i" }}
{{ date|timesince }}
{{ date|timeuntil }}

<!-- Safety -->
{{ value|safe }}
{{ value|escape }}
{{ value|escapejs }}
{{ value|urlencode }}

<!-- Logic -->
{{ value|default:"N/A" }}
{{ value|default_if_none:"Unknown" }}
{{ value|yesno:"Yes,No,Maybe" }}

<!-- Django-specific -->
{{ value|linebreaks }}         {# Convert newlines to <p> and <br> #}
{{ value|linebreaksbr }}       {# Convert newlines to <br> #}
{{ value|pluralize }}          {# Add 's' if needed #}
{{ value|filesizeformat }}     {# Human-readable file size #}
```

### Custom Filters (Flask)

```python
# app.py
from flask import Flask
import re

app = Flask(__name__)

@app.template_filter('reverse')
def reverse_filter(s):
    """Reverse a string"""
    return s[::-1]

@app.template_filter('currency')
def currency_filter(value):
    """Format as currency"""
    return f"${value:,.2f}"

@app.template_filter('phone')
def phone_filter(value):
    """Format phone number"""
    value = re.sub(r'\D', '', str(value))
    return f"({value[:3]}) {value[3:6]}-{value[6:]}"

@app.template_filter('excerpt')
def excerpt_filter(text, length=100):
    """Create excerpt from text"""
    if len(text) <= length:
        return text
    return text[:length].rsplit(' ', 1)[0] + '...'

# Alternative registration method
def markdown_filter(text):
    """Convert markdown to HTML"""
    import markdown
    return markdown.markdown(text)

app.jinja_env.filters['markdown'] = markdown_filter
```

```html
<!-- Usage in templates -->
{{ "hello"|reverse }}                {# olleh #}
{{ 1234.56|currency }}               {# $1,234.56 #}
{{ "5551234567"|phone }}             {# (555) 123-4567 #}
{{ long_text|excerpt(50) }}
{{ markdown_content|markdown|safe }}
```

### Custom Filters (Django)

```python
# myapp/templatetags/custom_filters.py
from django import template
from django.utils.safestring import mark_safe
import markdown

register = template.Library()

@register.filter(name='currency')
def currency(value):
    """Format as currency"""
    return f"${value:,.2f}"

@register.filter
def markdown_to_html(text):
    """Convert markdown to HTML"""
    return mark_safe(markdown.markdown(text))

@register.filter
def split(value, separator=','):
    """Split string by separator"""
    return value.split(separator)
```

```django
<!-- templates/page.html -->
{% load custom_filters %}

{{ price|currency }}
{{ content|markdown_to_html }}
{{ tags|split:"|" }}
```

### Template Functions/Tags

```python
# Flask - Custom function
@app.context_processor
def utility_processor():
    def format_date(date):
        return date.strftime('%B %d, %Y')
    
    def is_mobile():
        user_agent = request.headers.get('User-Agent', '').lower()
        return 'mobile' in user_agent
    
    return dict(format_date=format_date, is_mobile=is_mobile)
```

```html
<!-- Usage -->
{{ format_date(post.created_at) }}

{% if is_mobile() %}
    <p>Mobile view</p>
{% endif %}
```

---

## Context and Data Passing

### Flask Context

```python
from flask import Flask, render_template, request, session

app = Flask(__name__)
app.secret_key = 'your-secret-key'

# Simple context
@app.route('/')
def index():
    return render_template('index.html',
        title='Home',
        user='John')

# Dictionary context
@app.route('/profile')
def profile():
    context = {
        'title': 'Profile',
        'user': {
            'name': 'Alice',
            'email': 'alice@example.com',
            'age': 25
        },
        'posts_count': 42
    }
    return render_template('profile.html', **context)

# Context processors (available in all templates)
@app.context_processor
def inject_user():
    return dict(current_user=get_current_user())

@app.context_processor
def inject_config():
    return dict(
        site_name='My Website',
        current_year=2024,
        debug=app.debug
    )

# Request context (automatically available)
@app.route('/debug')
def debug():
    return render_template('debug.html')
    # request, session, g are automatically available
```

```html
<!-- templates/debug.html -->
<p>URL: {{ request.url }}</p>
<p>Method: {{ request.method }}</p>
<p>User Agent: {{ request.headers.get('User-Agent') }}</p>
<p>Session ID: {{ session.get('id') }}</p>
<p>Config: {{ config.DEBUG }}</p>
```

### Django Context

```python
# views.py
from django.shortcuts import render
from django.contrib.auth.decorators import login_required

def index(request):
    context = {
        'title': 'Home',
        'user': request.user,
        'items': Item.objects.all()
    }
    return render(request, 'index.html', context)

# Context processors (settings.py)
# These are available in all templates automatically:
# - request (django.template.context_processors.request)
# - user (django.contrib.auth.context_processors.auth)
# - messages (django.contrib.messages.context_processors.messages)

# Custom context processor
# myapp/context_processors.py
def site_info(request):
    return {
        'site_name': 'My Django Site',
        'current_year': 2024,
        'social_links': {
            'twitter': 'https://twitter.com/mysite',
            'github': 'https://github.com/mysite'
        }
    }

# Add to settings.py
TEMPLATES = [{
    'OPTIONS': {
        'context_processors': [
            # ... default processors
            'myapp.context_processors.site_info',
        ],
    },
}]
```

```django
<!-- Now available in all templates -->
<h1>{{ site_name }}</h1>
<footer>&copy; {{ current_year }}</footer>
<a href="{{ social_links.twitter }}">Twitter</a>
```

### Passing Complex Data

```python
# Flask example
from datetime import datetime

@app.route('/dashboard')
def dashboard():
    return render_template('dashboard.html',
        stats={
            'users': 1234,
            'posts': 5678,
            'comments': 9012
        },
        recent_posts=[
            {
                'id': 1,
                'title': 'First Post',
                'author': 'John',
                'created_at': datetime(2024, 1, 15),
                'tags': ['python', 'flask']
            },
            {
                'id': 2,
                'title': 'Second Post',
                'author': 'Alice',
                'created_at': datetime(2024, 1, 16),
                'tags': ['django', 'web']
            }
        ],
        charts_data={
            'labels': ['Jan', 'Feb', 'Mar'],
            'values': [10, 20, 30]
        }
    )
```

```html
<!-- templates/dashboard.html -->
<h1>Dashboard</h1>

<div class="stats">
    <div class="stat">
        <h3>{{ stats.users }}</h3>
        <p>Users</p>
    </div>
    <div class="stat">
        <h3>{{ stats.posts }}</h3>
        <p>Posts</p>
    </div>
    <div class="stat">
        <h3>{{ stats.comments }}</h3>
        <p>Comments</p>
    </div>
</div>

<h2>Recent Posts</h2>
{% for post in recent_posts %}
    <article>
        <h3>{{ post.title }}</h3>
        <p>By {{ post.author }} on {{ post.created_at.strftime('%Y-%m-%d') }}</p>
        <div class="tags">
            {% for tag in post.tags %}
                <span class="tag">{{ tag }}</span>
            {% endfor %}
        </div>
    </article>
{% endfor %}

<script>
// Pass data to JavaScript
const chartData = {
    labels: {{ charts_data.labels|tojson }},
    values: {{ charts_data.values|tojson }}
};
</script>
```

---

## Template Best Practices

### 1. Separation of Concerns

```python
# ❌ BAD: Logic in templates
{% for user in all_users %}
    {% if user.is_active and user.posts.count() > 0 and user.created_at > some_date %}
        <!-- Complex logic in template -->
    {% endif %}
{% endfor %}

# ✅ GOOD: Logic in view
@app.route('/users')
def users():
    active_users = [
        user for user in User.query.all()
        if user.is_active and user.posts.count() > 0 and user.is_recent()
    ]
    return render_template('users.html', users=active_users)
```

### 2. DRY (Don't Repeat Yourself)

```html
<!-- ❌ BAD: Repeated code -->
<!-- page1.html -->
<header>
    <nav><!-- navigation --></nav>
</header>
<main><!-- content --></main>
<footer><!-- footer --></footer>

<!-- page2.html -->
<header>
    <nav><!-- same navigation --></nav>
</header>
<main><!-- different content --></main>
<footer><!-- same footer --></footer>

<!-- ✅ GOOD: Use inheritance -->
<!-- base.html -->
<header>{% include 'nav.html' %}</header>
<main>{% block content %}{% endblock %}</main>
<footer>{% include 'footer.html' %}</footer>

<!-- page1.html -->
{% extends 'base.html' %}
{% block content %}<!-- page 1 content -->{% endblock %}

<!-- page2.html -->
{% extends 'base.html' %}
{% block content %}<!-- page 2 content -->{% endblock %}
```

### 3. Security: Always Escape User Input

```html
<!-- ❌ DANGEROUS: Unescaped user input (XSS vulnerability) -->
<p>{{ user_comment|safe }}</p>

<!-- ✅ SAFE: Escaped by default -->
<p>{{ user_comment }}</p>

<!-- ✅ SAFE: Only use |safe for trusted content -->
<div>{{ admin_approved_html|safe }}</div>

<!-- Django CSRF protection -->
<form method="post">
    {% csrf_token %}
    <!-- form fields -->
</form>
```

### 4. Performance Optimization

```python
# ❌ BAD: N+1 query problem
@app.route('/posts')
def posts():
    posts = Post.query.all()
    return render_template('posts.html', posts=posts)
```

```html
<!-- This causes N queries for authors -->
{% for post in posts %}
    <p>Author: {{ post.author.name }}</p>  <!-- Separate query each time -->
{% endfor %}
```

```python
# ✅ GOOD: Eager loading
@app.route('/posts')
def posts():
    posts = Post.query.options(joinedload(Post.author)).all()
    return render_template('posts.html', posts=posts)
```

### 5. Organize Templates

```
templates/
├── base.html
├── components/
│   ├── navbar.html
│   ├── footer.html
│   ├── sidebar.html
│   └── pagination.html
├── macros/
│   ├── forms.html
│   └── cards.html
├── layouts/
│   ├── dashboard.html
│   └── landing.html
├── pages/
│   ├── home.html
│   ├── about.html
│   └── contact.html
├── blog/
│   ├── post_list.html
│   ├── post_detail.html
│   └── post_form.html
└── errors/
    ├── 404.html
    └── 500.html
```

### 6. Responsive Design

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    
    <title>{% block title %}Default Title{% endblock %} | My Site</title>
    
    {% block meta %}{% endblock %}
    
    <!-- Responsive CSS -->
    <link rel="stylesheet" href="{{ url_for('static', filename='css/responsive.css') }}">
    
    {% block styles %}{% endblock %}
</head>
<body>
    <!-- Mobile-first approach -->
    <nav class="mobile-nav">
        <!-- Mobile navigation -->
    </nav>
    
    <nav class="desktop-nav">
        <!-- Desktop navigation -->
    </nav>
    
    {% block content %}{% endblock %}
</body>
</html>
```

### 7. Accessibility

```html
<!-- Semantic HTML -->
<header role="banner">
    <nav role="navigation" aria-label="Main navigation">
        <ul>
            <li><a href="/">Home</a></li>
            <li><a href="/about">About</a></li>
        </ul>
    </nav>
</header>

<main role="main" id="main-content">
    <h1>{{ page_title }}</h1>
    {% block content %}{% endblock %}
</main>

<aside role="complementary" aria-label="Sidebar">
    {% block sidebar %}{% endblock %}
</aside>

<footer role="contentinfo">
    {% block footer %}{% endblock %}
</footer>

<!-- Form accessibility -->
<form>
    <label for="email">Email Address</label>
    <input type="email" 
           id="email" 
           name="email" 
           aria-required="true"
           aria-describedby="email-help">
    <span id="email-help">We'll never share your email.</span>
    
    <button type="submit" aria-label="Submit form">Submit</button>
</form>

<!-- Image alt text -->
<img src="{{ url_for('static', filename='images/logo.png') }}" 
     alt="Company logo - Home link">
```

### 8. Error Handling Templates

```python
# Flask error handlers
@app.errorhandler(404)
def page_not_found(error):
    return render_template('errors/404.html'), 404

@app.errorhandler(500)
def internal_error(error):
    db.session.rollback()  # Rollback any failed transactions
    return render_template('errors/500.html'), 500

@app.errorhandler(403)
def forbidden(error):
    return render_template('errors/403.html'), 403
```

```html
<!-- templates/errors/404.html -->
{% extends "base.html" %}

{% block title %}Page Not Found{% endblock %}

{% block content %}
<div class="error-page">
    <h1>404</h1>
    <h2>Page Not Found</h2>
    <p>The page you're looking for doesn't exist.</p>
    <a href="{{ url_for('index') }}" class="btn">Go Home</a>
</div>
{% endblock %}

<!-- templates/errors/500.html -->
{% extends "base.html" %}

{% block title %}Server Error{% endblock %}

{% block content %}
<div class="error-page">
    <h1>500</h1>
    <h2>Internal Server Error</h2>
    <p>Something went wrong on our end. We're working to fix it.</p>
    {% if config.DEBUG %}
        <pre>{{ error }}</pre>
    {% endif %}
</div>
{% endblock %}
```

### 9. Caching Templates

```python
# Flask with caching
from flask import Flask
from flask_caching import Cache

app = Flask(__name__)
cache = Cache(app, config={'CACHE_TYPE': 'simple'})

@app.route('/expensive')
@cache.cached(timeout=300)  # Cache for 5 minutes
def expensive_page():
    # Expensive computation
    data = compute_expensive_data()
    return render_template('expensive.html', data=data)

# Cache specific template fragments
@app.route('/partial-cache')
def partial_cache():
    return render_template('partial_cache.html')
```

```html
<!-- templates/partial_cache.html -->
{% extends "base.html" %}

{% block content %}
    <!-- This part changes frequently -->
    <p>Current time: {{ current_time }}</p>
    
    <!-- Cache this expensive part -->
    {% cache 300, 'sidebar' %}
    <aside>
        <!-- Expensive sidebar content -->
        {% for item in expensive_items %}
            <div>{{ item }}</div>
        {% endfor %}
    </aside>
    {% endcache %}
{% endblock %}
```

### 10. Internationalization (i18n)

```python
# Flask-Babel example
from flask import Flask
from flask_babel import Babel, gettext

app = Flask(__name__)
babel = Babel(app)

@app.route('/')
def index():
    return render_template('index.html')
```

```html
<!-- templates/index.html -->
{% extends "base.html" %}

{% block content %}
    <h1>{{ _('Welcome') }}</h1>
    <p>{{ _('Hello, World!') }}</p>
    
    <!-- Plural forms -->
    <p>{{ ngettext('%(num)d item', '%(num)d items', items|length) }}</p>
    
    <!-- Variables in translations -->
    <p>{{ _('Hello, %(name)s!', name=user.name) }}</p>
{% endblock %}
```

---

## Server-Side vs Client-Side Rendering

### Server-Side Rendering (SSR)

**How it works:**
1. User requests a page
2. Server generates HTML from templates
3. Complete HTML sent to browser
4. Browser displays page immediately

**Advantages:**
- ✅ Better SEO (search engines see full content)
- ✅ Faster initial page load
- ✅ Works without JavaScript
- ✅ Simpler state management
- ✅ Better for static content

**Disadvantages:**
- ❌ Full page reloads
- ❌ More server resources
- ❌ Less interactive
- ❌ Slower subsequent navigation

**Example:**

```python
@app.route('/blog/<int:post_id>')
def blog_post(post_id):
    post = Post.query.get_or_404(post_id)
    comments = Comment.query.filter_by(post_id=post_id).all()
    
    # Server generates complete HTML
    return render_template('blog_post.html',
                         post=post,
                         comments=comments)
```

### Client-Side Rendering (CSR)

**How it works:**
1. User requests page
2. Server sends minimal HTML + JavaScript
3. JavaScript fetches data via API
4. JavaScript generates HTML in browser

**Advantages:**
- ✅ Highly interactive
- ✅ Faster navigation after initial load
- ✅ Rich user experience
- ✅ Less server load

**Disadvantages:**
- ❌ Slower initial load
- ❌ SEO challenges
- ❌ Requires JavaScript
- ❌ More complex state management

**Example:**

```html
<!-- Minimal HTML -->
<div id="app">Loading...</div>
<script src="app.js"></script>
```

```javascript
// JavaScript fetches data and renders
fetch('/api/posts/123')
    .then(response => response.json())
    .then(data => {
        document.getElementById('app').innerHTML = `
            <h1>${data.title}</h1>
            <p>${data.content}</p>
        `;
    });
```

### Hybrid Approach (SSR + Hydration)

Modern frameworks like Next.js use SSR for initial load, then CSR for interactions:

```python
# Flask example: SSR with progressive enhancement
@app.route('/products')
def products():
    products = Product.query.all()
    
    # Render initial HTML on server
    return render_template('products.html', 
                         products=products,
                         # Pass data to JavaScript
                         products_json=products_schema.dump(products))
```

```html
<!-- templates/products.html -->
{% extends "base.html" %}

{% block content %}
<!-- Server-rendered content (works without JS) -->
<div id="products">
    {% for product in products %}
        <div class="product">
            <h3>{{ product.name }}</h3>
            <p>${{ product.price }}</p>
        </div>
    {% endfor %}
</div>

<!-- Enhanced with JavaScript -->
<script>
// Data from server
const productsData = {{ products_json|tojson }};

// Add interactivity
document.querySelectorAll('.product').forEach((el, i) => {
    el.addEventListener('click', () => {
        showProductDetails(productsData[i]);
    });
});
</script>
{% endblock %}
```

### When to Use Each

**Use SSR when:**
- SEO is critical (blogs, e-commerce, public content)
- Users have slow connections
- Content is mostly static
- Accessibility is paramount
- JavaScript might be disabled

**Use CSR when:**
- Building a web application (dashboard, admin panel)
- High interactivity needed
- SEO is less important
- Users have modern browsers
- Real-time updates required

**Use Hybrid when:**
- Need both SEO and interactivity
- Building a complex application
- Performance is critical
- You have the resources for complexity

---

## Complete Example: Blog Application

### Flask Blog with Templates

```python
# app.py
from flask import Flask, render_template, request, redirect, url_for
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///blog.db'
db = SQLAlchemy(app)

# Models
class Post(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200), nullable=False)
    content = db.Column(db.Text, nullable=False)
    author = db.Column(db.String(100), nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    category = db.Column(db.String(50))
    
    def __repr__(self):
        return f'<Post {self.title}>'

# Create tables
with app.app_context():
    db.create_all()

# Context processor
@app.context_processor
def inject_globals():
    return {
        'site_name': 'My Blog',
        'current_year': datetime.now().year,
        'categories': ['Technology', 'Travel', 'Food', 'Lifestyle']
    }

# Routes
@app.route('/')
def index():
    page = request.args.get('page', 1, type=int)
    per_page = 5
    
    posts = Post.query.order_by(Post.created_at.desc()).paginate(
        page=page, per_page=per_page, error_out=False
    )
    
    return render_template('index.html', posts=posts)

@app.route('/post/<int:post_id>')
def post_detail(post_id):
    post = Post.query.get_or_404(post_id)
    return render_template('post_detail.html', post=post)

@app.route('/category/<category>')
def category(category):
    posts = Post.query.filter_by(category=category).order_by(
        Post.created_at.desc()
    ).all()
    return render_template('category.html', 
                         category=category, 
                         posts=posts)

@app.route('/post/new', methods=['GET', 'POST'])
def new_post():
    if request.method == 'POST':
        post = Post(
            title=request.form['title'],
            content=request.form['content'],
            author=request.form['author'],
            category=request.form['category']
        )
        db.session.add(post)
        db.session.commit()
        return redirect(url_for('post_detail', post_id=post.id))
    
    return render_template('post_form.html')

if __name__ == '__main__':
    app.run(debug=True)
```

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}{{ site_name }}{% endblock %}</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
    {% block styles %}{% endblock %}
</head>
<body>
    <header>
        <nav>
            <h1><a href="/">{{ site_name }}</a></h1>
            <ul>
                <li><a href="/">Home</a></li>
                {% for cat in categories %}
                    <li><a href="{{ url_for('category', category=cat) }}">{{ cat }}</a></li>
                {% endfor %}
                <li><a href="{{ url_for('new_post') }}">New Post</a></li>
            </ul>
        </nav>
    </header>
    
    <main>
        {% block content %}{% endblock %}
    </main>
    
    <footer>
        <p>&copy; {{ current_year }} {{ site_name }}. All rights reserved.</p>
    </footer>
    
    {% block scripts %}{% endblock %}
</body>
</html>

<!-- templates/index.html -->
{% extends "base.html" %}

{% block title %}Home | {{ site_name }}{% endblock %}

{% block content %}
<h1>Latest Posts</h1>

{% for post in posts.items %}
    <article class="post-preview">
        <h2><a href="{{ url_for('post_detail', post_id=post.id) }}">{{ post.title }}</a></h2>
        <div class="meta">
            <span class="author">By {{ post.author }}</span>
            <span class="date">{{ post.created_at.strftime('%B %d, %Y') }}</span>
            <span class="category">{{ post.category }}</span>
        </div>
        <p>{{ post.content[:200] }}...</p>
        <a href="{{ url_for('post_detail', post_id=post.id') }}">Read more →</a>
    </article>
{% else %}
    <p>No posts yet.</p>
{% endfor %}

<!-- Pagination -->
{% if posts.pages > 1 %}
<div class="pagination">
    {% if posts.has_prev %}
        <a href="{{ url_for('index', page=posts.prev_num) }}">← Previous</a>
    {% endif %}
    
    <span>Page {{ posts.page }} of {{ posts.pages }}</span>
    
    {% if posts.has_next %}
        <a href="{{ url_for('index', page=posts.next_num) }}">Next →</a>
    {% endif %}
</div>
{% endif %}
{% endblock %}

<!-- templates/post_detail.html -->
{% extends "base.html" %}

{% block title %}{{ post.title }} | {{ site_name }}{% endblock %}

{% block content %}
<article class="post-full">
    <h1>{{ post.title }}</h1>
    
    <div class="meta">
        <span>By {{ post.author }}</span>
        <span>{{ post.created_at.strftime('%B %d, %Y at %I:%M %p') }}</span>
        <span><a href="{{ url_for('category', category=post.category) }}">{{ post.category }}</a></span>
    </div>
    
    <div class="content">
        {{ post.content|safe }}
    </div>
    
    <a href="{{ url_for('index') }}">← Back to all posts</a>
</article>
{% endblock %}

<!-- templates/post_form.html -->
{% extends "base.html" %}

{% block title %}New Post | {{ site_name }}{% endblock %}

{% block content %}
<h1>Create New Post</h1>

<form method="POST" action="{{ url_for('new_post') }}">
    <div class="form-group">
        <label for="title">Title</label>
        <input type="text" id="title" name="title" required>
    </div>
    
    <div class="form-group">
        <label for="author">Author</label>
        <input type="text" id="author" name="author" required>
    </div>
    
    <div class="form-group">
        <label for="category">Category</label>
        <select id="category" name="category">
            {% for cat in categories %}
                <option value="{{ cat }}">{{ cat }}</option>
            {% endfor %}
        </select>
    </div>
    
    <div class="form-group">
        <label for="content">Content</label>
        <textarea id="content" name="content" rows="15" required></textarea>
    </div>
    
    <button type="submit">Publish Post</button>
</form>
{% endblock %}
```

---

## Summary

### Key Takeaways

**1. Template Basics:**
- Separate presentation from logic
- Use `{{ }}` for variables, `{% %}` for logic
- Templates are compiled and cached for performance

**2. Template Engines:**
- **Jinja2**: Powerful, flexible, used by Flask
- **Django Templates**: Integrated, secure, opinionated
- Both support inheritance, includes, filters, and macros

**3. Best Practices:**
- Keep logic in views, not templates
- Use template inheritance (DRY)
- Always escape user input (security)
- Organize templates logically
- Use context processors for global data
- Optimize queries to avoid N+1 problems

**4. SSR Advantages:**
- Better SEO
- Faster initial load
- Works without JavaScript
- Simpler architecture

**5. When to Use SSR:**
- Content-heavy sites (blogs, news, e-commerce)
- SEO is critical
- Users on slow connections
- Accessibility requirements

Templates and server-side rendering remain powerful tools for building robust, maintainable web applications!