# Sessions and Authentication in Python

## Table of Contents
1. [Introduction](#introduction)
2. [Understanding Sessions](#understanding-sessions)
3. [Password Hashing](#password-hashing)
4. [Session-Based Authentication](#session-based-authentication)
5. [Token-Based Authentication (JWT)](#token-based-authentication-jwt)
6. [Best Practices](#best-practices)
7. [Common Security Vulnerabilities](#common-security-vulnerabilities)

---

## Introduction

Authentication is the process of verifying the identity of a user, while sessions maintain that authenticated state across multiple requests. In web applications, proper authentication and session management are critical for security.

### Key Concepts

- **Authentication**: Verifying who a user is
- **Authorization**: Determining what a user can do
- **Session**: A way to persist user state across HTTP requests
- **Token**: A cryptographic string that represents a user's authenticated session

---

## Understanding Sessions

Sessions allow web applications to maintain state across multiple HTTP requests. Since HTTP is stateless by default, sessions provide a way to remember user information.

### How Sessions Work

1. User logs in with credentials
2. Server validates credentials
3. Server creates a session and stores session data
4. Server sends session ID to client (usually via cookie)
5. Client sends session ID with each subsequent request
6. Server validates session ID and retrieves session data

### Session Storage Options

- **Server-side**: Redis, Memcached, database
- **Client-side**: Encrypted cookies (less common for sensitive data)
- **Hybrid**: Session ID in cookie, data on server

---

## Password Hashing

Never store passwords in plain text. Always use secure hashing algorithms with salt.

### Using bcrypt

`bcrypt` is a password hashing function designed to be slow, making brute-force attacks difficult.

#### Installation

```bash
pip install bcrypt
```

#### Basic Usage

```python
import bcrypt

# Hashing a password
def hash_password(password: str) -> bytes:
    """Hash a password using bcrypt."""
    # Generate a salt and hash the password
    salt = bcrypt.gensalt(rounds=12)  # rounds = cost factor (4-31)
    hashed = bcrypt.hashpw(password.encode('utf-8'), salt)
    return hashed

# Verifying a password
def verify_password(password: str, hashed: bytes) -> bool:
    """Verify a password against its hash."""
    return bcrypt.checkpw(password.encode('utf-8'), hashed)

# Example usage
if __name__ == "__main__":
    password = "MySecurePassword123!"
    
    # Hash the password
    hashed_pw = hash_password(password)
    print(f"Hashed password: {hashed_pw}")
    
    # Verify correct password
    is_valid = verify_password(password, hashed_pw)
    print(f"Password valid: {is_valid}")  # True
    
    # Verify incorrect password
    is_valid = verify_password("WrongPassword", hashed_pw)
    print(f"Wrong password valid: {is_valid}")  # False
```

### Using passlib

`passlib` is a comprehensive password hashing library supporting multiple algorithms.

#### Installation

```bash
pip install passlib
```

#### Basic Usage

```python
from passlib.context import CryptContext

# Create a password context with bcrypt
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# Hashing a password
def hash_password_passlib(password: str) -> str:
    """Hash a password using passlib."""
    return pwd_context.hash(password)

# Verifying a password
def verify_password_passlib(password: str, hashed: str) -> bool:
    """Verify a password against its hash."""
    return pwd_context.verify(password, hashed)

# Example usage
if __name__ == "__main__":
    password = "MySecurePassword123!"
    
    # Hash the password
    hashed_pw = hash_password_passlib(password)
    print(f"Hashed password: {hashed_pw}")
    
    # Verify password
    is_valid = verify_password_passlib(password, hashed_pw)
    print(f"Password valid: {is_valid}")  # True
```

### Argon2 (Recommended for New Projects)

Argon2 won the Password Hashing Competition in 2015 and is considered more secure than bcrypt.

```python
from passlib.context import CryptContext

# Create context with Argon2
pwd_context = CryptContext(schemes=["argon2"], deprecated="auto")

# Usage is identical to bcrypt example above
hashed = pwd_context.hash("password123")
is_valid = pwd_context.verify("password123", hashed)
```

---

## Session-Based Authentication

Session-based authentication stores user state on the server and uses a session ID to identify the user.

### Flask Example with Flask-Session

```python
from flask import Flask, session, redirect, url_for, request, render_template_string
from flask_session import Session
import bcrypt
import secrets
from datetime import timedelta

app = Flask(__name__)

# Configure session
app.config['SECRET_KEY'] = secrets.token_hex(16)
app.config['SESSION_TYPE'] = 'filesystem'  # Can also use 'redis', 'memcached', etc.
app.config['PERMANENT_SESSION_LIFETIME'] = timedelta(minutes=30)
Session(app)

# Mock user database
users_db = {
    "john@example.com": {
        "password_hash": bcrypt.hashpw(b"password123", bcrypt.gensalt()),
        "name": "John Doe"
    }
}

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        email = request.form.get('email')
        password = request.form.get('password')
        
        # Check if user exists
        user = users_db.get(email)
        if user and bcrypt.checkpw(password.encode('utf-8'), user['password_hash']):
            # Create session
            session['user_email'] = email
            session['user_name'] = user['name']
            session.permanent = True
            return redirect(url_for('dashboard'))
        else:
            return "Invalid credentials", 401
    
    return '''
        <form method="post">
            <input type="email" name="email" placeholder="Email" required>
            <input type="password" name="password" placeholder="Password" required>
            <button type="submit">Login</button>
        </form>
    '''

@app.route('/dashboard')
def dashboard():
    # Check if user is logged in
    if 'user_email' not in session:
        return redirect(url_for('login'))
    
    return f"Welcome, {session['user_name']}! <a href='/logout'>Logout</a>"

@app.route('/logout')
def logout():
    # Clear session
    session.clear()
    return redirect(url_for('login'))

if __name__ == '__main__':
    app.run(debug=True)
```

### FastAPI Example with Session Middleware

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from pydantic import BaseModel
from passlib.context import CryptContext
import secrets
from datetime import datetime, timedelta
from typing import Optional, Dict

app = FastAPI()

# Password hashing
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# Security
security = HTTPBearer()

# Mock database
users_db = {
    "john@example.com": {
        "password_hash": pwd_context.hash("password123"),
        "name": "John Doe"
    }
}

# Session storage (in production, use Redis or similar)
sessions: Dict[str, dict] = {}

class LoginRequest(BaseModel):
    email: str
    password: str

class UserResponse(BaseModel):
    email: str
    name: str

def create_session(email: str) -> str:
    """Create a new session and return session ID."""
    session_id = secrets.token_urlsafe(32)
    sessions[session_id] = {
        "email": email,
        "created_at": datetime.utcnow(),
        "expires_at": datetime.utcnow() + timedelta(hours=24)
    }
    return session_id

def get_current_user(credentials: HTTPAuthorizationCredentials = Depends(security)) -> dict:
    """Dependency to get current user from session."""
    session_id = credentials.credentials
    session_data = sessions.get(session_id)
    
    if not session_data:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid session"
        )
    
    # Check if session expired
    if datetime.utcnow() > session_data['expires_at']:
        del sessions[session_id]
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Session expired"
        )
    
    user = users_db.get(session_data['email'])
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="User not found"
        )
    
    return {"email": session_data['email'], "name": user['name']}

@app.post("/login")
async def login(login_data: LoginRequest):
    """Login endpoint."""
    user = users_db.get(login_data.email)
    
    if not user or not pwd_context.verify(login_data.password, user['password_hash']):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid credentials"
        )
    
    session_id = create_session(login_data.email)
    
    return {
        "session_id": session_id,
        "message": "Login successful"
    }

@app.get("/me", response_model=UserResponse)
async def get_me(current_user: dict = Depends(get_current_user)):
    """Get current user information."""
    return current_user

@app.post("/logout")
async def logout(credentials: HTTPAuthorizationCredentials = Depends(security)):
    """Logout endpoint."""
    session_id = credentials.credentials
    if session_id in sessions:
        del sessions[session_id]
    
    return {"message": "Logout successful"}
```

---

## Token-Based Authentication (JWT)

JSON Web Tokens (JWT) are a stateless authentication mechanism where the server doesn't need to store session data.

### JWT Structure

A JWT consists of three parts separated by dots:

```
header.payload.signature
```

- **Header**: Contains token type and hashing algorithm
- **Payload**: Contains claims (user data)
- **Signature**: Ensures token hasn't been tampered with

### Using PyJWT

#### Installation

```bash
pip install pyjwt
```

#### Basic JWT Implementation

```python
import jwt
from datetime import datetime, timedelta
from typing import Optional

SECRET_KEY = "your-secret-key-keep-it-secret"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None) -> str:
    """Create a JWT access token."""
    to_encode = data.copy()
    
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=15)
    
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    
    return encoded_jwt

def decode_access_token(token: str) -> Optional[dict]:
    """Decode and verify a JWT token."""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload
    except jwt.ExpiredSignatureError:
        print("Token has expired")
        return None
    except jwt.InvalidTokenError:
        print("Invalid token")
        return None

# Example usage
if __name__ == "__main__":
    # Create token
    user_data = {"sub": "john@example.com", "name": "John Doe"}
    token = create_access_token(
        data=user_data,
        expires_delta=timedelta(minutes=30)
    )
    print(f"Token: {token}")
    
    # Decode token
    decoded = decode_access_token(token)
    print(f"Decoded: {decoded}")
```

### Complete FastAPI JWT Authentication Example

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from pydantic import BaseModel, EmailStr
from passlib.context import CryptContext
from jose import JWTError, jwt
from datetime import datetime, timedelta
from typing import Optional

app = FastAPI()

# Configuration
SECRET_KEY = "09d25e094faa6ca2556c818166b7a9563b93f7099f6f0f4caa6cf63b88e8d3e7"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

# Password hashing
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# OAuth2 scheme
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

# Mock database
users_db = {
    "john@example.com": {
        "username": "john@example.com",
        "full_name": "John Doe",
        "email": "john@example.com",
        "hashed_password": pwd_context.hash("password123"),
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
    email: Optional[EmailStr] = None
    full_name: Optional[str] = None
    disabled: Optional[bool] = None

class UserInDB(User):
    hashed_password: str

# Helper functions
def verify_password(plain_password: str, hashed_password: str) -> bool:
    """Verify a password against its hash."""
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    """Hash a password."""
    return pwd_context.hash(password)

def get_user(db, username: str) -> Optional[UserInDB]:
    """Get user from database."""
    if username in db:
        user_dict = db[username]
        return UserInDB(**user_dict)
    return None

def authenticate_user(db, username: str, password: str) -> Optional[UserInDB]:
    """Authenticate a user."""
    user = get_user(db, username)
    if not user:
        return None
    if not verify_password(password, user.hashed_password):
        return None
    return user

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None) -> str:
    """Create JWT access token."""
    to_encode = data.copy()
    
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=15)
    
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    
    return encoded_jwt

async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    """Get current user from JWT token."""
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
    
    user = get_user(users_db, username=token_data.username)
    if user is None:
        raise credentials_exception
    
    return user

async def get_current_active_user(current_user: User = Depends(get_current_user)) -> User:
    """Check if user is active."""
    if current_user.disabled:
        raise HTTPException(status_code=400, detail="Inactive user")
    return current_user

# Routes
@app.post("/token", response_model=Token)
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    """Login endpoint to get access token."""
    user = authenticate_user(users_db, form_data.username, form_data.password)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    
    access_token_expires = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = create_access_token(
        data={"sub": user.username},
        expires_delta=access_token_expires
    )
    
    return {"access_token": access_token, "token_type": "bearer"}

@app.get("/users/me", response_model=User)
async def read_users_me(current_user: User = Depends(get_current_active_user)):
    """Get current user information."""
    return current_user

@app.get("/users/me/items")
async def read_own_items(current_user: User = Depends(get_current_active_user)):
    """Get current user's items."""
    return [{"item_id": "Foo", "owner": current_user.username}]
```

### Refresh Tokens

Refresh tokens are long-lived tokens used to obtain new access tokens without re-authentication.

```python
from datetime import datetime, timedelta
from typing import Optional
import jwt

SECRET_KEY = "your-secret-key"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 15
REFRESH_TOKEN_EXPIRE_DAYS = 7

def create_tokens(data: dict):
    """Create both access and refresh tokens."""
    # Access token (short-lived)
    access_token_expires = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = create_access_token(
        data=data,
        expires_delta=access_token_expires
    )
    
    # Refresh token (long-lived)
    refresh_token_expires = timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS)
    to_encode = data.copy()
    expire = datetime.utcnow() + refresh_token_expires
    to_encode.update({"exp": expire, "type": "refresh"})
    refresh_token = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    
    return {
        "access_token": access_token,
        "refresh_token": refresh_token,
        "token_type": "bearer"
    }

def verify_refresh_token(token: str) -> Optional[dict]:
    """Verify refresh token and return payload."""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        
        # Check if it's actually a refresh token
        if payload.get("type") != "refresh":
            return None
        
        return payload
    except jwt.ExpiredSignatureError:
        return None
    except jwt.InvalidTokenError:
        return None

# In your FastAPI app
@app.post("/refresh")
async def refresh_token(refresh_token: str):
    """Get new access token using refresh token."""
    payload = verify_refresh_token(refresh_token)
    
    if not payload:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid refresh token"
        )
    
    # Create new access token
    access_token = create_access_token(
        data={"sub": payload.get("sub")},
        expires_delta=timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    )
    
    return {"access_token": access_token, "token_type": "bearer"}
```

---

## Best Practices

### 1. Password Security

```python
# DO: Use strong password requirements
def is_strong_password(password: str) -> bool:
    """Check if password meets security requirements."""
    if len(password) < 12:
        return False
    if not any(char.isupper() for char in password):
        return False
    if not any(char.islower() for char in password):
        return False
    if not any(char.isdigit() for char in password):
        return False
    if not any(char in "!@#$%^&*()_+-=[]{}|;:,.<>?" for char in password):
        return False
    return True

# DO: Use appropriate cost factors for hashing
bcrypt.gensalt(rounds=12)  # Good balance of security and performance

# DON'T: Store passwords in plain text
# DON'T: Use weak hashing algorithms like MD5 or SHA1
```

### 2. Session Security

```python
# DO: Set appropriate session expiration
app.config['PERMANENT_SESSION_LIFETIME'] = timedelta(minutes=30)

# DO: Use secure session cookies
app.config['SESSION_COOKIE_SECURE'] = True  # Only send over HTTPS
app.config['SESSION_COOKIE_HTTPONLY'] = True  # Prevent JavaScript access
app.config['SESSION_COOKIE_SAMESITE'] = 'Lax'  # CSRF protection

# DO: Regenerate session ID after login
@app.route('/login', methods=['POST'])
def login():
    # ... authentication logic ...
    session.regenerate()  # Prevent session fixation attacks
```

### 3. JWT Security

```python
# DO: Use strong secret keys
import secrets
SECRET_KEY = secrets.token_urlsafe(32)

# DO: Set appropriate expiration times
ACCESS_TOKEN_EXPIRE_MINUTES = 15  # Short-lived
REFRESH_TOKEN_EXPIRE_DAYS = 7  # Longer-lived

# DO: Include necessary claims
def create_access_token(data: dict):
    to_encode = data.copy()
    to_encode.update({
        "exp": datetime.utcnow() + timedelta(minutes=15),
        "iat": datetime.utcnow(),  # Issued at
        "jti": secrets.token_urlsafe(16),  # JWT ID (for revocation)
    })
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

# DON'T: Store sensitive data in JWT payload (it's base64 encoded, not encrypted)
# DON'T: Use weak signing algorithms like 'none'
```

### 4. Rate Limiting

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.post("/login")
@limiter.limit("5/minute")  # Limit login attempts
async def login(request: Request, form_data: OAuth2PasswordRequestForm = Depends()):
    # ... login logic ...
    pass
```

### 5. Multi-Factor Authentication (MFA)

```python
import pyotp

def generate_totp_secret() -> str:
    """Generate a TOTP secret for a user."""
    return pyotp.random_base32()

def verify_totp(secret: str, token: str) -> bool:
    """Verify a TOTP token."""
    totp = pyotp.TOTP(secret)
    return totp.verify(token, valid_window=1)

def get_totp_uri(secret: str, username: str, issuer: str) -> str:
    """Get provisioning URI for QR code."""
    totp = pyotp.TOTP(secret)
    return totp.provisioning_uri(name=username, issuer_name=issuer)

# Usage in login flow
@app.post("/login")
async def login(credentials: LoginRequest):
    user = authenticate_user(credentials.email, credentials.password)
    
    if not user:
        raise HTTPException(status_code=401, detail="Invalid credentials")
    
    # If MFA is enabled, require TOTP
    if user.get("mfa_enabled"):
        if not credentials.totp_token:
            return {"requires_mfa": True}
        
        if not verify_totp(user["totp_secret"], credentials.totp_token):
            raise HTTPException(status_code=401, detail="Invalid MFA token")
    
    # Create session/token
    token = create_access_token({"sub": user["email"]})
    return {"access_token": token}
```

---

## Common Security Vulnerabilities

### 1. Session Fixation

**Problem**: Attacker sets a user's session ID before authentication.

**Solution**: Regenerate session ID after successful login.

```python
@app.route('/login', methods=['POST'])
def login():
    if authenticate_user(request.form['username'], request.form['password']):
        session.regenerate()  # Create new session ID
        session['user_id'] = get_user_id(request.form['username'])
        return redirect('/dashboard')
```

### 2. Cross-Site Request Forgery (CSRF)

**Problem**: Attacker tricks user into making unwanted requests.

**Solution**: Use CSRF tokens.

```python
from flask_wtf.csrf import CSRFProtect

csrf = CSRFProtect(app)

# CSRF protection is automatic for forms using Flask-WTF
# For API endpoints using JWT, CSRF is less of a concern
# as tokens are typically in headers, not cookies
```

### 3. JWT Token Theft

**Problem**: JWT tokens can be stolen via XSS attacks.

**Solution**: Store tokens securely and use short expiration times.

```python
# Store in httpOnly cookie (can't be accessed by JavaScript)
@app.post("/login")
async def login(response: Response, credentials: LoginRequest):
    token = create_access_token({"sub": credentials.email})
    
    response.set_cookie(
        key="access_token",
        value=f"Bearer {token}",
        httponly=True,  # Prevent JavaScript access
        secure=True,  # Only send over HTTPS
        samesite="lax",  # CSRF protection
        max_age=1800  # 30 minutes
    )
    
    return {"message": "Login successful"}
```

### 4. Timing Attacks

**Problem**: Attackers can infer information from response times.

**Solution**: Use constant-time comparison for sensitive operations.

```python
import hmac

def constant_time_compare(val1: str, val2: str) -> bool:
    """Compare two strings in constant time."""
    return hmac.compare_digest(val1, val2)

# Use when comparing tokens, passwords hashes, etc.
if constant_time_compare(provided_token, expected_token):
    # Valid token
    pass
```

### 5. Brute Force Attacks

**Problem**: Attackers try many password combinations.

**Solution**: Implement account lockout and rate limiting.

```python
from datetime import datetime, timedelta
from collections import defaultdict

# Simple in-memory rate limiter (use Redis in production)
login_attempts = defaultdict(list)

def check_login_attempts(email: str, max_attempts: int = 5, window_minutes: int = 15) -> bool:
    """Check if user has exceeded login attempts."""
    now = datetime.utcnow()
    cutoff = now - timedelta(minutes=window_minutes)
    
    # Remove old attempts
    login_attempts[email] = [
        attempt for attempt in login_attempts[email]
        if attempt > cutoff
    ]
    
    # Check if limit exceeded
    if len(login_attempts[email]) >= max_attempts:
        return False
    
    return True

def record_login_attempt(email: str):
    """Record a failed login attempt."""
    login_attempts[email].append(datetime.utcnow())

@app.post("/login")
async def login(credentials: LoginRequest):
    if not check_login_attempts(credentials.email):
        raise HTTPException(
            status_code=429,
            detail="Too many login attempts. Please try again later."
        )
    
    user = authenticate_user(credentials.email, credentials.password)
    
    if not user:
        record_login_attempt(credentials.email)
        raise HTTPException(status_code=401, detail="Invalid credentials")
    
    # Clear attempts on successful login
    login_attempts.pop(credentials.email, None)
    
    # ... create token ...
```

---

## Complete Production-Ready Example

Here's a comprehensive example combining all best practices:

```python
from fastapi import FastAPI, Depends, HTTPException, status, Request, Response
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from pydantic import BaseModel, EmailStr, validator
from passlib.context import CryptContext
from jose import JWTError, jwt
from datetime import datetime, timedelta
from typing import Optional
import secrets
import pyotp
from slowapi import Limiter
from slowapi.util import get_remote_address

# Configuration
SECRET_KEY = secrets.token_urlsafe(32)
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 15
REFRESH_TOKEN_EXPIRE_DAYS = 7

# Initialize app
app = FastAPI()
limiter = Limiter(key_func=get_remote_address)

# Password hashing
pwd_context = CryptContext(schemes=["argon2"], deprecated="auto")

# OAuth2
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

# Models
class UserCreate(BaseModel):
    email: EmailStr
    password: str
    full_name: str
    
    @validator('password')
    def password_strength(cls, v):
        if len(v) < 12:
            raise ValueError('Password must be at least 12 characters')
        if not any(char.isupper() for char in v):
            raise ValueError('Password must contain uppercase letter')
        if not any(char.islower() for char in v):
            raise ValueError('Password must contain lowercase letter')
        if not any(char.isdigit() for char in v):
            raise ValueError('Password must contain digit')
        if not any(char in "!@#$%^&*()_+-=[]{}|;:,.<>?" for char in v):
            raise ValueError('Password must contain special character')
        return v

class LoginRequest(BaseModel):
    email: EmailStr
    password: str
    totp_token: Optional[str] = None

class User(BaseModel):
    email: EmailStr
    full_name: str
    mfa_enabled: bool = False

# Mock database (use real database in production)
users_db = {}

# Helper functions
def create_access_token(data: dict, expires_delta: Optional[timedelta] = None):
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode.update({
        "exp": expire,
        "iat": datetime.utcnow(),
        "jti": secrets.token_urlsafe(16)
    })
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

async def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        email: str = payload.get("sub")
        if email is None or email not in users_db:
            raise HTTPException(status_code=401, detail="Invalid token")
        return users_db[email]
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")

# Routes
@app.post("/register")
@limiter.limit("3/hour")
async def register(request: Request, user: UserCreate):
    if user.email in users_db:
        raise HTTPException(status_code=400, detail="Email already registered")
    
    users_db[user.email] = {
        "email": user.email,
        "full_name": user.full_name,
        "hashed_password": pwd_context.hash(user.password),
        "mfa_enabled": False,
        "totp_secret": None,
        "created_at": datetime.utcnow()
    }
    
    return {"message": "User registered successfully"}

@app.post("/login")
@limiter.limit("5/minute")
async def login(request: Request, response: Response, credentials: LoginRequest):
    user = users_db.get(credentials.email)
    
    if not user or not pwd_context.verify(credentials.password, user["hashed_password"]):
        raise HTTPException(status_code=401, detail="Invalid credentials")
    
    # Check MFA
    if user["mfa_enabled"]:
        if not credentials.totp_token:
            return {"requires_mfa": True}
        
        totp = pyotp.TOTP(user["totp_secret"])
        if not totp.verify(credentials.totp_token, valid_window=1):
            raise HTTPException(status_code=401, detail="Invalid MFA token")
    
    # Create tokens
    access_token = create_access_token(
        data={"sub": credentials.email},
        expires_delta=timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    )
    
    # Set secure cookie
    response.set_cookie(
        key="access_token",
        value=f"Bearer {access_token}",
        httponly=True,
        secure=True,
        samesite="lax",
        max_age=ACCESS_TOKEN_EXPIRE_MINUTES * 60
    )
    
    return {"message": "Login successful"}

@app.get("/me")
async def get_me(current_user: dict = Depends(get_current_user)):
    return User(
        email=current_user["email"],
        full_name=current_user["full_name"],
        mfa_enabled=current_user["mfa_enabled"]
    )

@app.post("/logout")
async def logout(response: Response):
    response.delete_cookie("access_token")
    return {"message": "Logout successful"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## Summary

This guide covered the essential aspects of sessions and authentication in Python:

1. **Sessions**: Maintaining state across HTTP requests
2. **Password Hashing**: Using bcrypt, passlib, and Argon2
3. **Session-Based Auth**: Traditional server-side session management
4. **JWT Authentication**: Stateless token-based authentication
5. **Best Practices**: Security considerations and implementations
6. **Vulnerabilities**: Common attacks and how to prevent them

Remember: Security is not a one-time implementation but an ongoing process. Always stay updated with the latest security best practices and vulnerabilities.