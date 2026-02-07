# Authentication & Authorization in FastAPI

## 7.1 Authentication Basics

### What is Authentication vs Authorization?

**Authentication** verifies *who you are* (identity verification)
**Authorization** determines *what you can do* (permission verification)

```python
# Authentication: "Are you really John Doe?"
user = authenticate_user(username, password)

# Authorization: "Can John Doe access this resource?"
if user.role == "admin":
    allow_access()
```

### Session-Based Auth

Traditional server-side session management using cookies.

```python
from fastapi import FastAPI, Cookie, Response
from uuid import uuid4

app = FastAPI()

# In-memory session store (use Redis in production)
sessions = {}

@app.post("/login")
async def login(username: str, password: str, response: Response):
    # Verify credentials
    user = verify_credentials(username, password)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid credentials")
    
    # Create session
    session_id = str(uuid4())
    sessions[session_id] = {
        "user_id": user.id,
        "username": user.username,
        "role": user.role
    }
    
    # Set session cookie
    response.set_cookie(
        key="session_id",
        value=session_id,
        httponly=True,
        secure=True,
        samesite="lax",
        max_age=3600  # 1 hour
    )
    
    return {"message": "Logged in successfully"}

@app.get("/profile")
async def get_profile(session_id: str = Cookie(None)):
    if not session_id or session_id not in sessions:
        raise HTTPException(status_code=401, detail="Not authenticated")
    
    session_data = sessions[session_id]
    return {"username": session_data["username"]}

@app.post("/logout")
async def logout(response: Response, session_id: str = Cookie(None)):
    if session_id in sessions:
        del sessions[session_id]
    response.delete_cookie("session_id")
    return {"message": "Logged out successfully"}
```

### Token-Based Auth

Stateless authentication using tokens sent with each request.

```python
from fastapi import FastAPI, Header, HTTPException

app = FastAPI()

# In-memory token store
valid_tokens = {}

@app.post("/login")
async def login(username: str, password: str):
    user = verify_credentials(username, password)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid credentials")
    
    # Generate token
    token = generate_token()
    valid_tokens[token] = user.id
    
    return {"access_token": token, "token_type": "bearer"}

@app.get("/profile")
async def get_profile(authorization: str = Header(None)):
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Not authenticated")
    
    token = authorization.replace("Bearer ", "")
    if token not in valid_tokens:
        raise HTTPException(status_code=401, detail="Invalid token")
    
    user_id = valid_tokens[token]
    return {"user_id": user_id}
```

### OAuth2 Overview

OAuth2 is an authorization framework that enables applications to obtain limited access to user accounts.

**Key Concepts:**
- **Resource Owner**: The user
- **Client**: The application requesting access
- **Authorization Server**: Issues tokens
- **Resource Server**: Hosts protected resources

**Common Flows:**
- **Authorization Code**: For web applications
- **Implicit**: For browser-based apps (deprecated)
- **Resource Owner Password**: For trusted applications
- **Client Credentials**: For machine-to-machine

## 7.2 OAuth2 with Password Flow

### OAuth2PasswordBearer

FastAPI's utility for OAuth2 password flow.

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

app = FastAPI()

# OAuth2 scheme
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

@app.get("/users/me")
async def read_users_me(token: str = Depends(oauth2_scheme)):
    # token is automatically extracted from Authorization header
    user = get_user_from_token(token)
    return user
```

### OAuth2PasswordRequestForm

Standard form data for login requests.

```python
from fastapi.security import OAuth2PasswordRequestForm

@app.post("/token")
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    # form_data.username
    # form_data.password
    # form_data.scopes (optional)
    
    user = authenticate_user(form_data.username, form_data.password)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    
    access_token = create_access_token(data={"sub": user.username})
    return {"access_token": access_token, "token_type": "bearer"}
```

### Token Generation

Create tokens with user information.

```python
from datetime import datetime, timedelta
from typing import Optional

SECRET_KEY = "your-secret-key-keep-it-secret"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None):
    to_encode = data.copy()
    
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=15)
    
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    
    return encoded_jwt
```

### Token Validation

Verify and decode tokens.

```python
from jose import JWTError, jwt
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

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
    except JWTError:
        raise credentials_exception
    
    user = get_user(username=username)
    if user is None:
        raise credentials_exception
    
    return user
```

### Login Endpoint

Complete login implementation.

```python
from pydantic import BaseModel

class Token(BaseModel):
    access_token: str
    token_type: str

class User(BaseModel):
    username: str
    email: str
    disabled: bool = False

@app.post("/token", response_model=Token)
async def login_for_access_token(
    form_data: OAuth2PasswordRequestForm = Depends()
):
    user = authenticate_user(form_data.username, form_data.password)
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
```

### Protected Routes

Protect endpoints with authentication.

```python
@app.get("/users/me", response_model=User)
async def read_users_me(current_user: User = Depends(get_current_user)):
    return current_user

@app.get("/users/me/items")
async def read_own_items(current_user: User = Depends(get_current_user)):
    return [{"item_id": "Foo", "owner": current_user.username}]

# Require active user
async def get_current_active_user(
    current_user: User = Depends(get_current_user)
):
    if current_user.disabled:
        raise HTTPException(status_code=400, detail="Inactive user")
    return current_user

@app.get("/users/me/active")
async def read_active_user(
    current_user: User = Depends(get_current_active_user)
):
    return current_user
```

## 7.3 JWT (JSON Web Tokens)

### JWT Structure and Format

JWT consists of three parts separated by dots: `header.payload.signature`

```python
# Example JWT
# eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

# Decoded:
# Header: {"alg": "HS256", "typ": "JWT"}
# Payload: {"sub": "1234567890", "name": "John Doe", "iat": 1516239022}
# Signature: HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)
```

### Creating JWT Tokens (python-jose)

Install and use python-jose for JWT operations.

```bash
pip install "python-jose[cryptography]"
```

```python
from jose import jwt
from datetime import datetime, timedelta

SECRET_KEY = "09d25e094faa6ca2556c818166b7a9563b93f7099f6f0f4caa6cf63b88e8d3e7"
ALGORITHM = "HS256"

def create_access_token(data: dict, expires_delta: timedelta = None):
    to_encode = data.copy()
    
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=15)
    
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    
    return encoded_jwt

# Create token
token = create_access_token(
    data={"sub": "john@example.com", "role": "admin"},
    expires_delta=timedelta(hours=1)
)
```

### Token Payload

Standard and custom claims in JWT payload.

```python
from datetime import datetime, timedelta

def create_token_with_claims(user_id: int, username: str, role: str):
    # Standard claims
    payload = {
        "sub": str(user_id),           # Subject (user identifier)
        "iat": datetime.utcnow(),      # Issued at
        "exp": datetime.utcnow() + timedelta(hours=1),  # Expiration
        "nbf": datetime.utcnow(),      # Not before
        "iss": "myapp.com",            # Issuer
        "aud": "myapp-users",          # Audience
    }
    
    # Custom claims
    payload.update({
        "username": username,
        "role": role,
        "permissions": ["read", "write"]
    })
    
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

def decode_token(token: str):
    try:
        payload = jwt.decode(
            token,
            SECRET_KEY,
            algorithms=[ALGORITHM],
            audience="myapp-users"
        )
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token has expired")
    except jwt.JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")
```

### Token Expiration

Handle token expiration properly.

```python
from jose import jwt, JWTError, ExpiredSignatureError

async def verify_token(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        
        if username is None:
            raise HTTPException(status_code=401, detail="Invalid token")
        
        return payload
        
    except ExpiredSignatureError:
        raise HTTPException(
            status_code=401,
            detail="Token has expired",
            headers={"WWW-Authenticate": "Bearer"},
        )
    except JWTError:
        raise HTTPException(
            status_code=401,
            detail="Could not validate credentials",
            headers={"WWW-Authenticate": "Bearer"},
        )
```

### Refresh Tokens

Implement refresh token mechanism for long-lived sessions.

```python
from pydantic import BaseModel

class TokenResponse(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"

# Store refresh tokens (use database in production)
refresh_tokens_db = {}

def create_refresh_token(data: dict):
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(days=7)  # Refresh tokens last longer
    to_encode.update({"exp": expire, "type": "refresh"})
    
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

@app.post("/token", response_model=TokenResponse)
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    user = authenticate_user(form_data.username, form_data.password)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid credentials")
    
    # Create access and refresh tokens
    access_token = create_access_token(
        data={"sub": user.username},
        expires_delta=timedelta(minutes=30)
    )
    refresh_token = create_refresh_token(data={"sub": user.username})
    
    # Store refresh token
    refresh_tokens_db[refresh_token] = user.username
    
    return {
        "access_token": access_token,
        "refresh_token": refresh_token,
        "token_type": "bearer"
    }

@app.post("/refresh", response_model=TokenResponse)
async def refresh_access_token(refresh_token: str):
    # Verify refresh token
    try:
        payload = jwt.decode(refresh_token, SECRET_KEY, algorithms=[ALGORITHM])
        
        if payload.get("type") != "refresh":
            raise HTTPException(status_code=401, detail="Invalid token type")
        
        username = payload.get("sub")
        
        # Check if token is in database
        if refresh_token not in refresh_tokens_db:
            raise HTTPException(status_code=401, detail="Invalid refresh token")
        
        # Create new access token
        new_access_token = create_access_token(
            data={"sub": username},
            expires_delta=timedelta(minutes=30)
        )
        
        return {
            "access_token": new_access_token,
            "refresh_token": refresh_token,
            "token_type": "bearer"
        }
        
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid refresh token")
```

### Token Revocation Strategies

Implement token revocation for logout and security.

```python
from datetime import datetime

# Strategy 1: Token Blacklist
blacklisted_tokens = set()

@app.post("/logout")
async def logout(token: str = Depends(oauth2_scheme)):
    blacklisted_tokens.add(token)
    return {"message": "Successfully logged out"}

async def verify_token_not_blacklisted(token: str = Depends(oauth2_scheme)):
    if token in blacklisted_tokens:
        raise HTTPException(status_code=401, detail="Token has been revoked")
    return token

# Strategy 2: Token Version in Database
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    username = Column(String)
    token_version = Column(Integer, default=0)

def create_token_with_version(user: User):
    payload = {
        "sub": user.username,
        "token_version": user.token_version,
        "exp": datetime.utcnow() + timedelta(hours=1)
    }
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

async def verify_token_version(token: str = Depends(oauth2_scheme), db: Session = Depends(get_db)):
    payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    username = payload.get("sub")
    token_version = payload.get("token_version")
    
    user = db.query(User).filter(User.username == username).first()
    if user.token_version != token_version:
        raise HTTPException(status_code=401, detail="Token has been revoked")
    
    return user

@app.post("/logout-all-devices")
async def logout_all_devices(
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    current_user.token_version += 1
    db.commit()
    return {"message": "Logged out from all devices"}
```

### Token Blacklisting

Efficient token blacklist implementation with Redis.

```python
import redis
from datetime import timedelta

# Redis connection
redis_client = redis.Redis(host='localhost', port=6379, db=0, decode_responses=True)

def blacklist_token(token: str, expires_in: int = 3600):
    """Add token to blacklist with expiration"""
    redis_client.setex(f"blacklist:{token}", expires_in, "1")

def is_token_blacklisted(token: str) -> bool:
    """Check if token is blacklisted"""
    return redis_client.exists(f"blacklist:{token}") > 0

@app.post("/logout")
async def logout(token: str = Depends(oauth2_scheme)):
    # Get token expiration from payload
    payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    exp = payload.get("exp")
    
    # Calculate remaining time
    remaining_time = exp - datetime.utcnow().timestamp()
    
    if remaining_time > 0:
        blacklist_token(token, int(remaining_time))
    
    return {"message": "Successfully logged out"}

async def get_current_user(token: str = Depends(oauth2_scheme)):
    # Check blacklist first
    if is_token_blacklisted(token):
        raise HTTPException(status_code=401, detail="Token has been revoked")
    
    # Verify token
    payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    username = payload.get("sub")
    
    user = get_user(username)
    return user
```

## 7.4 Password Security

### Password Hashing with passlib

Never store passwords in plain text. Always hash them.

```bash
pip install "passlib[bcrypt]"
```

```python
from passlib.context import CryptContext

# Create password context
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password: str) -> str:
    """Hash a password"""
    return pwd_context.hash(password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """Verify a password against its hash"""
    return pwd_context.verify(plain_password, hashed_password)

# Usage
hashed = hash_password("mySecurePassword123!")
print(hashed)  # $2b$12$...

is_valid = verify_password("mySecurePassword123!", hashed)
print(is_valid)  # True
```

### Bcrypt Algorithm

Bcrypt is the recommended algorithm for password hashing.

```python
from passlib.context import CryptContext

# Configure bcrypt
pwd_context = CryptContext(
    schemes=["bcrypt"],
    deprecated="auto",
    bcrypt__rounds=12,  # Cost factor (default is 12)
)

def authenticate_user(username: str, password: str, db: Session):
    user = db.query(User).filter(User.username == username).first()
    
    if not user:
        return False
    
    if not verify_password(password, user.hashed_password):
        return False
    
    return user

@app.post("/register")
async def register(username: str, email: str, password: str, db: Session = Depends(get_db)):
    # Hash password before storing
    hashed_password = hash_password(password)
    
    user = User(
        username=username,
        email=email,
        hashed_password=hashed_password
    )
    
    db.add(user)
    db.commit()
    db.refresh(user)
    
    return {"message": "User created successfully"}
```

### Password Validation

Enforce password strength requirements.

```python
import re
from fastapi import HTTPException

def validate_password(password: str) -> bool:
    """
    Validate password strength:
    - At least 8 characters
    - At least one uppercase letter
    - At least one lowercase letter
    - At least one digit
    - At least one special character
    """
    if len(password) < 8:
        raise HTTPException(
            status_code=400,
            detail="Password must be at least 8 characters long"
        )
    
    if not re.search(r"[A-Z]", password):
        raise HTTPException(
            status_code=400,
            detail="Password must contain at least one uppercase letter"
        )
    
    if not re.search(r"[a-z]", password):
        raise HTTPException(
            status_code=400,
            detail="Password must contain at least one lowercase letter"
        )
    
    if not re.search(r"\d", password):
        raise HTTPException(
            status_code=400,
            detail="Password must contain at least one digit"
        )
    
    if not re.search(r"[!@#$%^&*()_+\-=\[\]{};':\"\\|,.<>\/?]", password):
        raise HTTPException(
            status_code=400,
            detail="Password must contain at least one special character"
        )
    
    return True

@app.post("/register")
async def register(username: str, password: str, db: Session = Depends(get_db)):
    # Validate password
    validate_password(password)
    
    # Hash and store
    hashed_password = hash_password(password)
    user = User(username=username, hashed_password=hashed_password)
    db.add(user)
    db.commit()
    
    return {"message": "User registered successfully"}
```

### Password Reset Flow

Implement secure password reset functionality.

```python
import secrets
from datetime import datetime, timedelta

class PasswordResetToken(Base):
    __tablename__ = "password_reset_tokens"
    
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey("users.id"))
    token = Column(String(100), unique=True, index=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    expires_at = Column(DateTime)
    used = Column(Boolean, default=False)

@app.post("/password-reset/request")
async def request_password_reset(email: str, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.email == email).first()
    
    if not user:
        # Don't reveal if email exists
        return {"message": "If email exists, reset link will be sent"}
    
    # Generate reset token
    reset_token = secrets.token_urlsafe(32)
    
    # Store token
    token_entry = PasswordResetToken(
        user_id=user.id,
        token=reset_token,
        expires_at=datetime.utcnow() + timedelta(hours=1)
    )
    db.add(token_entry)
    db.commit()
    
    # Send email (implement email sending)
    send_reset_email(user.email, reset_token)
    
    return {"message": "If email exists, reset link will be sent"}

@app.post("/password-reset/confirm")
async def reset_password(
    token: str,
    new_password: str,
    db: Session = Depends(get_db)
):
    # Validate password
    validate_password(new_password)
    
    # Find token
    reset_token = db.query(PasswordResetToken).filter(
        PasswordResetToken.token == token,
        PasswordResetToken.used == False,
        PasswordResetToken.expires_at > datetime.utcnow()
    ).first()
    
    if not reset_token:
        raise HTTPException(status_code=400, detail="Invalid or expired token")
    
    # Update password
    user = db.query(User).filter(User.id == reset_token.user_id).first()
    user.hashed_password = hash_password(new_password)
    
    # Mark token as used
    reset_token.used = True
    
    db.commit()
    
    return {"message": "Password reset successfully"}
```

### Password Strength Requirements

Implement comprehensive password strength checking.

```python
from pydantic import BaseModel, validator

class PasswordStrength:
    @staticmethod
    def check_strength(password: str) -> dict:
        """Return password strength analysis"""
        score = 0
        feedback = []
        
        # Length check
        if len(password) >= 8:
            score += 1
        else:
            feedback.append("Password should be at least 8 characters")
        
        if len(password) >= 12:
            score += 1
        
        # Character variety
        if re.search(r"[a-z]", password):
            score += 1
        else:
            feedback.append("Add lowercase letters")
        
        if re.search(r"[A-Z]", password):
            score += 1
        else:
            feedback.append("Add uppercase letters")
        
        if re.search(r"\d", password):
            score += 1
        else:
            feedback.append("Add numbers")
        
        if re.search(r"[!@#$%^&*()_+\-=\[\]{};':\"\\|,.<>\/?]", password):
            score += 1
        else:
            feedback.append("Add special characters")
        
        # Common password check
        common_passwords = ["password", "123456", "qwerty"]
        if password.lower() in common_passwords:
            score = 0
            feedback.append("Password is too common")
        
        strength = "weak"
        if score >= 5:
            strength = "strong"
        elif score >= 3:
            strength = "medium"
        
        return {
            "score": score,
            "strength": strength,
            "feedback": feedback
        }

class UserRegister(BaseModel):
    username: str
    email: str
    password: str
    
    @validator('password')
    def validate_password_strength(cls, v):
        analysis = PasswordStrength.check_strength(v)
        
        if analysis["strength"] == "weak":
            raise ValueError(f"Password too weak: {', '.join(analysis['feedback'])}")
        
        return v

@app.post("/register")
async def register(user_data: UserRegister, db: Session = Depends(get_db)):
    hashed_password = hash_password(user_data.password)
    user = User(
        username=user_data.username,
        email=user_data.email,
        hashed_password=hashed_password
    )
    db.add(user)
    db.commit()
    
    return {"message": "User registered successfully"}

@app.post("/password/check-strength")
async def check_password_strength(password: str):
    return PasswordStrength.check_strength(password)
```

## 7.5 OAuth2 Scopes

### Understanding Scopes

Scopes define specific permissions that can be granted to tokens.

```python
# Example scopes
SCOPES = {
    "users:read": "Read user information",
    "users:write": "Modify user information",
    "posts:read": "Read posts",
    "posts:write": "Create and modify posts",
    "posts:delete": "Delete posts",
    "admin": "Full administrative access"
}
```

### Defining Scopes

Configure OAuth2 with scopes.

```python
from fastapi.security import OAuth2PasswordBearer, SecurityScopes
from pydantic import BaseModel, ValidationError

oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="token",
    scopes={
        "users:read": "Read user information",
        "users:write": "Modify user information",
        "posts:read": "Read posts",
        "posts:write": "Create and modify posts",
    }
)

class Token(BaseModel):
    access_token: str
    token_type: str

def create_access_token_with_scopes(username: str, scopes: list[str]):
    payload = {
        "sub": username,
        "scopes": scopes,
        "exp": datetime.utcnow() + timedelta(hours=1)
    }
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

@app.post("/token")
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    user = authenticate_user(form_data.username, form_data.password)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid credentials")
    
    # Grant scopes based on user role
    if user.role == "admin":
        scopes = ["users:read", "users:write", "posts:read", "posts:write"]
    else:
        scopes = ["users:read", "posts:read"]
    
    access_token = create_access_token_with_scopes(user.username, scopes)
    
    return {"access_token": access_token, "token_type": "bearer"}
```

### Checking Scopes in Dependencies

Verify required scopes in protected endpoints.

```python
from fastapi import Security

async def get_current_user_with_scopes(
    security_scopes: SecurityScopes,
    token: str = Depends(oauth2_scheme)
):
    if security_scopes.scopes:
        authenticate_value = f'Bearer scope="{security_scopes.scope_str}"'
    else:
        authenticate_value = "Bearer"
    
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": authenticate_value},
    )
    
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        token_scopes = payload.get("scopes", [])
        
        if username is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception
    
    # Check if token has required scopes
    for scope in security_scopes.scopes:
        if scope not in token_scopes:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Not enough permissions",
                headers={"WWW-Authenticate": authenticate_value},
            )
    
    user = get_user(username)
    if user is None:
        raise credentials_exception
    
    return user

# Protected endpoints with scope requirements
@app.get("/users/me")
async def read_users_me(
    current_user: User = Security(get_current_user_with_scopes, scopes=["users:read"])
):
    return current_user

@app.put("/users/me")
async def update_user_me(
    user_update: UserUpdate,
    current_user: User = Security(get_current_user_with_scopes, scopes=["users:write"])
):
    # Update user
    return {"message": "User updated"}

@app.delete("/posts/{post_id}")
async def delete_post(
    post_id: int,
    current_user: User = Security(get_current_user_with_scopes, scopes=["posts:write", "posts:delete"])
):
    # Delete post
    return {"message": "Post deleted"}
```

### Role-Based Access Control (RBAC)

Implement RBAC system with roles and permissions.

```python
from enum import Enum

class Role(str, Enum):
    ADMIN = "admin"
    MODERATOR = "moderator"
    USER = "user"
    GUEST = "guest"

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True)
    username = Column(String(50), unique=True)
    email = Column(String(100), unique=True)
    hashed_password = Column(String(255))
    role = Column(String(20), default=Role.USER)

# Role-based permissions
ROLE_PERMISSIONS = {
    Role.ADMIN: ["*"],  # All permissions
    Role.MODERATOR: ["users:read", "posts:read", "posts:write", "posts:delete"],
    Role.USER: ["users:read", "posts:read", "posts:write"],
    Role.GUEST: ["posts:read"]
}

def get_user_permissions(role: Role) -> list[str]:
    return ROLE_PERMISSIONS.get(role, [])

def has_permission(user_role: Role, required_permission: str) -> bool:
    permissions = get_user_permissions(user_role)
    return "*" in permissions or required_permission in permissions

# Dependency for role-based access
def require_role(allowed_roles: list[Role]):
    async def role_checker(current_user: User = Depends(get_current_user)):
        if current_user.role not in allowed_roles:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Insufficient permissions"
            )
        return current_user
    return role_checker

# Usage
@app.get("/admin/users")
async def get_all_users(
    current_user: User = Depends(require_role([Role.ADMIN]))
):
    # Only admins can access
    return {"users": []}

@app.delete("/posts/{post_id}")
async def delete_post(
    post_id: int,
    current_user: User = Depends(require_role([Role.ADMIN, Role.MODERATOR]))
):
    # Admins and moderators can delete
    return {"message": "Post deleted"}
```

### Permission Systems

Advanced permission system with fine-grained control.

```python
from sqlalchemy import Table

# Permission table
class Permission(Base):
    __tablename__ = "permissions"
    
    id = Column(Integer, primary_key=True)
    name = Column(String(50), unique=True)
    description = Column(String(200))

# Many-to-many relationship between roles and permissions
role_permissions = Table(
    'role_permissions',
    Base.metadata,
    Column('role_id', Integer, ForeignKey('roles.id')),
    Column('permission_id', Integer, ForeignKey('permissions.id'))
)

class RoleModel(Base):
    __tablename__ = "roles"
    
    id = Column(Integer, primary_key=True)
    name = Column(String(50), unique=True)
    permissions = relationship("Permission", secondary=role_permissions)

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True)
    username = Column(String(50))
    role_id = Column(Integer, ForeignKey("roles.id"))
    role = relationship("RoleModel")

# Check permissions
def has_permission(user: User, permission_name: str, db: Session) -> bool:
    if not user.role:
        return False
    
    for permission in user.role.permissions:
        if permission.name == permission_name or permission.name == "*":
            return True
    
    return False

# Permission dependency
def require_permission(permission_name: str):
    async def permission_checker(
        current_user: User = Depends(get_current_user),
        db: Session = Depends(get_db)
    ):
        if not has_permission(current_user, permission_name, db):
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Permission '{permission_name}' required"
            )
        return current_user
    return permission_checker

# Usage
@app.post("/posts")
async def create_post(
    post: PostCreate,
    current_user: User = Depends(require_permission("posts:create"))
):
    return {"message": "Post created"}

@app.delete("/users/{user_id}")
async def delete_user(
    user_id: int,
    current_user: User = Depends(require_permission("users:delete"))
):
    return {"message": "User deleted"}
```

## 7.6 Third-Party OAuth

### Google OAuth

Implement Google OAuth2 authentication.

```bash
pip install authlib httpx
```

```python
from authlib.integrations.starlette_client import OAuth
from starlette.config import Config

# Configuration
config = Config('.env')
oauth = OAuth(config)

oauth.register(
    name='google',
    client_id=config('GOOGLE_CLIENT_ID'),
    client_secret=config('GOOGLE_CLIENT_SECRET'),
    server_metadata_url='https://accounts.google.com/.well-known/openid-configuration',
    client_kwargs={
        'scope': 'openid email profile'
    }
)

@app.get("/login/google")
async def login_google(request: Request):
    redirect_uri = request.url_for('auth_google')
    return await oauth.google.authorize_redirect(request, redirect_uri)

@app.get("/auth/google")
async def auth_google(request: Request):
    token = await oauth.google.authorize_access_token(request)
    user_info = token.get('userinfo')
    
    if user_info:
        # Create or get user
        user = get_or_create_user(
            email=user_info['email'],
            name=user_info.get('name'),
            picture=user_info.get('picture')
        )
        
        # Create access token
        access_token = create_access_token(data={"sub": user.email})
        
        return {"access_token": access_token, "token_type": "bearer"}
    
    raise HTTPException(status_code=400, detail="Failed to get user info")
```

### GitHub OAuth

Implement GitHub OAuth2 authentication.

```python
oauth.register(
    name='github',
    client_id=config('GITHUB_CLIENT_ID'),
    client_secret=config('GITHUB_CLIENT_SECRET'),
    access_token_url='https://github.com/login/oauth/access_token',
    authorize_url='https://github.com/login/oauth/authorize',
    api_base_url='https://api.github.com/',
    client_kwargs={'scope': 'user:email'},
)

@app.get("/login/github")
async def login_github(request: Request):
    redirect_uri = request.url_for('auth_github')
    return await oauth.github.authorize_redirect(request, redirect_uri)

@app.get("/auth/github")
async def auth_github(request: Request):
    token = await oauth.github.authorize_access_token(request)
    
    # Get user info
    resp = await oauth.github.get('user', token=token)
    user_info = resp.json()
    
    # Create or get user
    user = get_or_create_user(
        email=user_info.get('email'),
        username=user_info.get('login'),
        avatar=user_info.get('avatar_url')
    )
    
    # Create access token
    access_token = create_access_token(data={"sub": user.email})
    
    return {"access_token": access_token, "token_type": "bearer"}
```

### Facebook OAuth

Implement Facebook OAuth2 authentication.

```python
oauth.register(
    name='facebook',
    client_id=config('FACEBOOK_CLIENT_ID'),
    client_secret=config('FACEBOOK_CLIENT_SECRET'),
    access_token_url='https://graph.facebook.com/oauth/access_token',
    authorize_url='https://www.facebook.com/dialog/oauth',
    api_base_url='https://graph.facebook.com/',
    client_kwargs={'scope': 'email public_profile'},
)

@app.get("/login/facebook")
async def login_facebook(request: Request):
    redirect_uri = request.url_for('auth_facebook')
    return await oauth.facebook.authorize_redirect(request, redirect_uri)

@app.get("/auth/facebook")
async def auth_facebook(request: Request):
    token = await oauth.facebook.authorize_access_token(request)
    
    # Get user info
    resp = await oauth.facebook.get('me?fields=id,name,email,picture', token=token)
    user_info = resp.json()
    
    # Create or get user
    user = get_or_create_user(
        email=user_info.get('email'),
        name=user_info.get('name'),
        picture=user_info.get('picture', {}).get('data', {}).get('url')
    )
    
    # Create access token
    access_token = create_access_token(data={"sub": user.email})
    
    return {"access_token": access_token, "token_type": "bearer"}
```

### Social Authentication

Unified social authentication handler.

```python
from pydantic import BaseModel

class SocialUser(BaseModel):
    email: str
    name: str | None = None
    picture: str | None = None
    provider: str

def get_or_create_user(
    email: str,
    name: str = None,
    picture: str = None,
    provider: str = "local",
    db: Session = None
):
    # Check if user exists
    user = db.query(User).filter(User.email == email).first()
    
    if not user:
        # Create new user
        user = User(
            email=email,
            username=email.split('@')[0],
            name=name,
            avatar_url=picture,
            auth_provider=provider,
            is_verified=True  # Social auth users are pre-verified
        )
        db.add(user)
        db.commit()
        db.refresh(user)
    
    return user

@app.get("/auth/{provider}")
async def social_auth(provider: str, request: Request):
    if provider not in ['google', 'github', 'facebook']:
        raise HTTPException(status_code=400, detail="Invalid provider")
    
    oauth_client = getattr(oauth, provider)
    redirect_uri = request.url_for(f'auth_{provider}')
    
    return await oauth_client.authorize_redirect(request, redirect_uri)
```

### OAuth Libraries (authlib)

Advanced usage of Authlib for OAuth2.

```python
from authlib.integrations.starlette_client import OAuth, OAuthError
from starlette.middleware.sessions import SessionMiddleware

app = FastAPI()

# Add session middleware (required for OAuth)
app.add_middleware(SessionMiddleware, secret_key="your-secret-key")

# Configure OAuth
oauth = OAuth()

# Custom OAuth provider
oauth.register(
    name='custom',
    client_id='your-client-id',
    client_secret='your-client-secret',
    authorize_url='https://provider.com/oauth/authorize',
    access_token_url='https://provider.com/oauth/token',
    userinfo_endpoint='https://provider.com/oauth/userinfo',
    client_kwargs={
        'scope': 'openid profile email',
        'token_endpoint_auth_method': 'client_secret_post',
    }
)

@app.exception_handler(OAuthError)
async def oauth_error_handler(request: Request, exc: OAuthError):
    return JSONResponse(
        status_code=400,
        content={"detail": f"OAuth error: {exc.error}"}
    )
```

## 7.7 API Keys

### API Key Authentication

Simple API key authentication scheme.

```python
from fastapi import Security, HTTPException, status
from fastapi.security import APIKeyHeader

API_KEY_NAME = "X-API-Key"
api_key_header = APIKeyHeader(name=API_KEY_NAME, auto_error=False)

# Store API keys (use database in production)
API_KEYS = {
    "sk_test_1234567890": {"user_id": 1, "name": "Test Key"},
    "sk_prod_abcdefghij": {"user_id": 2, "name": "Production Key"},
}

async def get_api_key(api_key: str = Security(api_key_header)):
    if api_key is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="API Key required"
        )
    
    if api_key not in API_KEYS:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Invalid API Key"
        )
    
    return API_KEYS[api_key]

@app.get("/protected")
async def protected_route(api_key_data: dict = Depends(get_api_key)):
    return {
        "message": "Access granted",
        "user_id": api_key_data["user_id"]
    }
```

### API Key in Headers

Best practice for API key authentication using headers.

```python
from fastapi.security import APIKeyHeader
from sqlalchemy.orm import Session

class APIKey(Base):
    __tablename__ = "api_keys"
    
    id = Column(Integer, primary_key=True)
    key = Column(String(64), unique=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id"))
    name = Column(String(100))
    created_at = Column(DateTime, default=datetime.utcnow)
    last_used_at = Column(DateTime)
    is_active = Column(Boolean, default=True)
    
    user = relationship("User")

api_key_header = APIKeyHeader(name="X-API-Key", auto_error=True)

async def validate_api_key(
    api_key: str = Security(api_key_header),
    db: Session = Depends(get_db)
):
    # Query database for API key
    key_obj = db.query(APIKey).filter(
        APIKey.key == api_key,
        APIKey.is_active == True
    ).first()
    
    if not key_obj:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Invalid or inactive API key"
        )
    
    # Update last used timestamp
    key_obj.last_used_at = datetime.utcnow()
    db.commit()
    
    return key_obj

@app.get("/api/data")
async def get_data(api_key: APIKey = Depends(validate_api_key)):
    return {
        "data": "sensitive information",
        "user_id": api_key.user_id
    }
```

### API Key in Query Parameters

Alternative API key authentication via query parameters (less secure).

```python
from fastapi.security import APIKeyQuery

api_key_query = APIKeyQuery(name="api_key", auto_error=True)

async def get_api_key_from_query(
    api_key: str = Security(api_key_query),
    db: Session = Depends(get_db)
):
    key_obj = db.query(APIKey).filter(APIKey.key == api_key).first()
    
    if not key_obj or not key_obj.is_active:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Invalid API key"
        )
    
    return key_obj

@app.get("/webhook")
async def webhook(
    data: dict,
    api_key: APIKey = Depends(get_api_key_from_query)
):
    # Process webhook
    return {"status": "received"}
```

### API Key Management

Complete API key management system.

```python
import secrets
from pydantic import BaseModel

class APIKeyCreate(BaseModel):
    name: str

class APIKeyResponse(BaseModel):
    id: int
    key: str
    name: str
    created_at: datetime
    
    class Config:
        from_attributes = True

def generate_api_key() -> str:
    """Generate secure API key"""
    return f"sk_{''.join(secrets.token_hex(32))}"

@app.post("/api-keys", response_model=APIKeyResponse)
async def create_api_key(
    key_data: APIKeyCreate,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    # Generate new API key
    new_key = generate_api_key()
    
    api_key = APIKey(
        key=new_key,
        name=key_data.name,
        user_id=current_user.id
    )
    
    db.add(api_key)
    db.commit()
    db.refresh(api_key)
    
    return api_key

@app.get("/api-keys", response_model=list[APIKeyResponse])
async def list_api_keys(
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    keys = db.query(APIKey).filter(APIKey.user_id == current_user.id).all()
    
    # Mask keys for security
    for key in keys:
        key.key = f"{key.key[:8]}...{key.key[-4:]}"
    
    return keys

@app.delete("/api-keys/{key_id}")
async def delete_api_key(
    key_id: int,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    api_key = db.query(APIKey).filter(
        APIKey.id == key_id,
        APIKey.user_id == current_user.id
    ).first()
    
    if not api_key:
        raise HTTPException(status_code=404, detail="API key not found")
    
    db.delete(api_key)
    db.commit()
    
    return {"message": "API key deleted"}

@app.put("/api-keys/{key_id}/revoke")
async def revoke_api_key(
    key_id: int,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    api_key = db.query(APIKey).filter(
        APIKey.id == key_id,
        APIKey.user_id == current_user.id
    ).first()
    
    if not api_key:
        raise HTTPException(status_code=404, detail="API key not found")
    
    api_key.is_active = False
    db.commit()
    
    return {"message": "API key revoked"}
```

---

## Complete Authentication System Example

```python
# Complete implementation combining JWT, password hashing, and role-based access

from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from sqlalchemy.orm import Session
from passlib.context import CryptContext
from jose import JWTError, jwt
from datetime import datetime, timedelta
from pydantic import BaseModel, EmailStr

# Configuration
SECRET_KEY = "your-secret-key-change-this-in-production"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

app = FastAPI()
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

# Models
class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True)
    email = Column(String(100), unique=True, index=True)
    username = Column(String(50), unique=True)
    hashed_password = Column(String(255))
    is_active = Column(Boolean, default=True)
    role = Column(String(20), default="user")

# Schemas
class UserCreate(BaseModel):
    email: EmailStr
    username: str
    password: str

class UserResponse(BaseModel):
    id: int
    email: str
    username: str
    role: str
    
    class Config:
        from_attributes = True

class Token(BaseModel):
    access_token: str
    token_type: str

# Utilities
def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    return pwd_context.hash(password)

def create_access_token(data: dict, expires_delta: timedelta = None):
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

def authenticate_user(db: Session, username: str, password: str):
    user = db.query(User).filter(User.username == username).first()
    if not user or not verify_password(password, user.hashed_password):
        return None
    return user

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: Session = Depends(get_db)
):
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
    except JWTError:
        raise credentials_exception
    
    user = db.query(User).filter(User.username == username).first()
    if user is None:
        raise credentials_exception
    
    return user

async def get_current_active_user(
    current_user: User = Depends(get_current_user)
):
    if not current_user.is_active:
        raise HTTPException(status_code=400, detail="Inactive user")
    return current_user

# Routes
@app.post("/register", response_model=UserResponse)
async def register(user: UserCreate, db: Session = Depends(get_db)):
    # Check if user exists
    if db.query(User).filter(User.email == user.email).first():
        raise HTTPException(status_code=400, detail="Email already registered")
    
    # Create user
    db_user = User(
        email=user.email,
        username=user.username,
        hashed_password=get_password_hash(user.password)
    )
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    
    return db_user

@app.post("/token", response_model=Token)
async def login(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: Session = Depends(get_db)
):
    user = authenticate_user(db, form_data.username, form_data.password)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    
    access_token_expires = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = create_access_token(
        data={"sub": user.username, "role": user.role},
        expires_delta=access_token_expires
    )
    
    return {"access_token": access_token, "token_type": "bearer"}

@app.get("/users/me", response_model=UserResponse)
async def read_users_me(current_user: User = Depends(get_current_active_user)):
    return current_user

@app.get("/admin/users")
async def admin_route(current_user: User = Depends(get_current_active_user)):
    if current_user.role != "admin":
        raise HTTPException(status_code=403, detail="Not enough permissions")
    return {"message": "Admin access granted"}
```

---

**Key Takeaways:**
- Always hash passwords with bcrypt
- Use JWT for stateless authentication
- Implement proper token validation and expiration
- Use OAuth2 scopes for fine-grained permissions
- Implement RBAC for role-based access control
- Support social login for better UX
- Use API keys for service-to-service authentication
- Always validate and sanitize user input
- Implement rate limiting and account lockout

**Additional Resources:**
- [FastAPI Security Documentation](https://fastapi.tiangolo.com/tutorial/security/)
- [OAuth 2.0 Specification](https://oauth.net/2/)
- [JWT.io](https://jwt.io/)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)