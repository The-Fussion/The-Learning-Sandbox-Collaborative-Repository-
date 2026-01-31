# Databases and ORMs in Python

## Table of Contents
1. [Introduction](#introduction)
2. [Database Fundamentals](#database-fundamentals)
3. [SQLite with Python](#sqlite-with-python)
4. [PostgreSQL with Python](#postgresql-with-python)
5. [MySQL with Python](#mysql-with-python)
6. [Raw SQL Operations](#raw-sql-operations)
7. [SQLAlchemy ORM](#sqlalchemy-orm)
8. [Django ORM](#django-orm)
9. [Database Migrations](#database-migrations)
10. [Best Practices](#best-practices)
11. [Advanced Topics](#advanced-topics)

---

## Introduction

### What is a Database?

A database is an organized collection of structured data stored electronically. Databases allow you to:
- Store data persistently
- Query data efficiently
- Maintain data integrity
- Handle concurrent access
- Ensure data security

### What is an ORM?

Object-Relational Mapping (ORM) is a programming technique that converts data between incompatible type systems (relational databases and object-oriented programming languages).

**Benefits of ORMs:**
- Write database queries in Python instead of SQL
- Database-agnostic code (mostly)
- Automatic SQL injection protection
- Built-in migration tools
- Cleaner, more maintainable code

**Drawbacks of ORMs:**
- Performance overhead
- Learning curve
- Less control over SQL queries
- Can be overkill for simple projects

---

## Database Fundamentals

### Relational Database Concepts

```sql
-- Tables: Store data in rows and columns
CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Relationships: Connect tables together
CREATE TABLE posts (
    id INTEGER PRIMARY KEY,
    user_id INTEGER,
    title VARCHAR(200),
    content TEXT,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Indexes: Speed up queries
CREATE INDEX idx_username ON users(username);

-- Constraints: Enforce data integrity
ALTER TABLE users ADD CONSTRAINT check_email CHECK (email LIKE '%@%');
```

### Database Connection Components

1. **Connection**: Establishes link to database
2. **Cursor**: Executes SQL commands
3. **Transaction**: Group of operations (commit/rollback)
4. **Connection Pool**: Reuses connections for efficiency

---

## SQLite with Python

SQLite is a lightweight, serverless database perfect for development and small applications.

### Basic SQLite Operations

```python
import sqlite3
from datetime import datetime
from typing import List, Optional, Tuple

# Connect to database (creates file if doesn't exist)
def create_connection(db_file: str) -> sqlite3.Connection:
    """Create a database connection to SQLite database."""
    try:
        conn = sqlite3.connect(db_file)
        conn.row_factory = sqlite3.Row  # Access columns by name
        print(f"SQLite version: {sqlite3.version}")
        return conn
    except sqlite3.Error as e:
        print(f"Error connecting to database: {e}")
        return None

# Create tables
def create_tables(conn: sqlite3.Connection):
    """Create tables in the database."""
    cursor = conn.cursor()
    
    # Users table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT UNIQUE NOT NULL,
            email TEXT UNIQUE NOT NULL,
            password_hash TEXT NOT NULL,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    # Posts table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS posts (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER NOT NULL,
            title TEXT NOT NULL,
            content TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE CASCADE
        )
    """)
    
    # Create indexes
    cursor.execute("CREATE INDEX IF NOT EXISTS idx_posts_user_id ON posts(user_id)")
    
    conn.commit()
    print("Tables created successfully")

# INSERT operations
def create_user(conn: sqlite3.Connection, username: str, email: str, 
                password_hash: str) -> int:
    """Create a new user."""
    cursor = conn.cursor()
    
    try:
        cursor.execute("""
            INSERT INTO users (username, email, password_hash)
            VALUES (?, ?, ?)
        """, (username, email, password_hash))
        
        conn.commit()
        return cursor.lastrowid
    except sqlite3.IntegrityError as e:
        print(f"Error creating user: {e}")
        return None

def create_post(conn: sqlite3.Connection, user_id: int, 
                title: str, content: str) -> int:
    """Create a new post."""
    cursor = conn.cursor()
    
    cursor.execute("""
        INSERT INTO posts (user_id, title, content)
        VALUES (?, ?, ?)
    """, (user_id, title, content))
    
    conn.commit()
    return cursor.lastrowid

# SELECT operations
def get_user_by_id(conn: sqlite3.Connection, user_id: int) -> Optional[sqlite3.Row]:
    """Get user by ID."""
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users WHERE id = ?", (user_id,))
    return cursor.fetchone()

def get_user_by_username(conn: sqlite3.Connection, username: str) -> Optional[sqlite3.Row]:
    """Get user by username."""
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users WHERE username = ?", (username,))
    return cursor.fetchone()

def get_all_users(conn: sqlite3.Connection) -> List[sqlite3.Row]:
    """Get all users."""
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users ORDER BY created_at DESC")
    return cursor.fetchall()

def get_user_posts(conn: sqlite3.Connection, user_id: int) -> List[sqlite3.Row]:
    """Get all posts by a user."""
    cursor = conn.cursor()
    cursor.execute("""
        SELECT p.*, u.username 
        FROM posts p
        JOIN users u ON p.user_id = u.id
        WHERE p.user_id = ?
        ORDER BY p.created_at DESC
    """, (user_id,))
    return cursor.fetchall()

# UPDATE operations
def update_user_email(conn: sqlite3.Connection, user_id: int, new_email: str):
    """Update user's email."""
    cursor = conn.cursor()
    cursor.execute("""
        UPDATE users 
        SET email = ?
        WHERE id = ?
    """, (new_email, user_id))
    conn.commit()
    return cursor.rowcount

def update_post(conn: sqlite3.Connection, post_id: int, title: str, content: str):
    """Update a post."""
    cursor = conn.cursor()
    cursor.execute("""
        UPDATE posts 
        SET title = ?, content = ?
        WHERE id = ?
    """, (title, content, post_id))
    conn.commit()
    return cursor.rowcount

# DELETE operations
def delete_user(conn: sqlite3.Connection, user_id: int):
    """Delete a user (will cascade to delete posts)."""
    cursor = conn.cursor()
    cursor.execute("DELETE FROM users WHERE id = ?", (user_id,))
    conn.commit()
    return cursor.rowcount

def delete_post(conn: sqlite3.Connection, post_id: int):
    """Delete a post."""
    cursor = conn.cursor()
    cursor.execute("DELETE FROM posts WHERE id = ?", (post_id,))
    conn.commit()
    return cursor.rowcount

# Example usage
if __name__ == "__main__":
    # Create connection
    conn = create_connection("example.db")
    
    if conn:
        # Create tables
        create_tables(conn)
        
        # Create user
        user_id = create_user(conn, "john_doe", "john@example.com", "hashed_password")
        print(f"Created user with ID: {user_id}")
        
        # Create post
        post_id = create_post(conn, user_id, "My First Post", "Hello, World!")
        print(f"Created post with ID: {post_id}")
        
        # Query user
        user = get_user_by_username(conn, "john_doe")
        if user:
            print(f"User: {user['username']}, Email: {user['email']}")
        
        # Query posts
        posts = get_user_posts(conn, user_id)
        for post in posts:
            print(f"Post: {post['title']} by {post['username']}")
        
        # Close connection
        conn.close()
```

### Context Manager for SQLite

```python
from contextlib import contextmanager

@contextmanager
def get_db_connection(db_file: str):
    """Context manager for database connections."""
    conn = sqlite3.connect(db_file)
    conn.row_factory = sqlite3.Row
    try:
        yield conn
        conn.commit()
    except Exception as e:
        conn.rollback()
        raise e
    finally:
        conn.close()

# Usage
with get_db_connection("example.db") as conn:
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users")
    users = cursor.fetchall()
```

### Transactions in SQLite

```python
def transfer_points(conn: sqlite3.Connection, from_user_id: int, 
                    to_user_id: int, points: int):
    """Transfer points between users with transaction."""
    cursor = conn.cursor()
    
    try:
        # Start transaction (implicit in sqlite3)
        
        # Deduct points from sender
        cursor.execute("""
            UPDATE users 
            SET points = points - ?
            WHERE id = ? AND points >= ?
        """, (points, from_user_id, points))
        
        if cursor.rowcount == 0:
            raise ValueError("Insufficient points or user not found")
        
        # Add points to receiver
        cursor.execute("""
            UPDATE users 
            SET points = points + ?
            WHERE id = ?
        """, (points, to_user_id))
        
        if cursor.rowcount == 0:
            raise ValueError("Recipient user not found")
        
        # Commit transaction
        conn.commit()
        print("Transfer successful")
        
    except Exception as e:
        # Rollback on error
        conn.rollback()
        print(f"Transfer failed: {e}")
        raise
```

---

## PostgreSQL with Python

PostgreSQL is a powerful, open-source relational database with advanced features.

### Installation

```bash
pip install psycopg2-binary
# or for production
pip install psycopg2
```

### Basic PostgreSQL Operations

```python
import psycopg2
from psycopg2 import pool
from psycopg2.extras import RealDictCursor
from contextlib import contextmanager
from typing import List, Optional, Dict

# Database configuration
DB_CONFIG = {
    'host': 'localhost',
    'database': 'myapp',
    'user': 'postgres',
    'password': 'password',
    'port': 5432
}

# Connection pool for better performance
connection_pool = None

def initialize_pool():
    """Initialize connection pool."""
    global connection_pool
    try:
        connection_pool = psycopg2.pool.SimpleConnectionPool(
            minconn=1,
            maxconn=10,
            **DB_CONFIG
        )
        print("Connection pool created successfully")
    except psycopg2.Error as e:
        print(f"Error creating connection pool: {e}")

@contextmanager
def get_db_connection():
    """Get connection from pool."""
    conn = connection_pool.getconn()
    try:
        yield conn
        conn.commit()
    except Exception as e:
        conn.rollback()
        raise e
    finally:
        connection_pool.putconn(conn)

# Create tables
def create_tables():
    """Create tables in PostgreSQL."""
    with get_db_connection() as conn:
        cursor = conn.cursor()
        
        # Users table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS users (
                id SERIAL PRIMARY KEY,
                username VARCHAR(50) UNIQUE NOT NULL,
                email VARCHAR(100) UNIQUE NOT NULL,
                password_hash VARCHAR(255) NOT NULL,
                is_active BOOLEAN DEFAULT TRUE,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        
        # Posts table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS posts (
                id SERIAL PRIMARY KEY,
                user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
                title VARCHAR(200) NOT NULL,
                content TEXT,
                published BOOLEAN DEFAULT FALSE,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        
        # Comments table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS comments (
                id SERIAL PRIMARY KEY,
                post_id INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
                user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
                content TEXT NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        
        # Indexes
        cursor.execute("CREATE INDEX IF NOT EXISTS idx_posts_user_id ON posts(user_id)")
        cursor.execute("CREATE INDEX IF NOT EXISTS idx_comments_post_id ON comments(post_id)")
        
        print("Tables created successfully")

# CRUD Operations
class UserRepository:
    """Repository pattern for user operations."""
    
    @staticmethod
    def create(username: str, email: str, password_hash: str) -> Optional[int]:
        """Create a new user."""
        with get_db_connection() as conn:
            cursor = conn.cursor()
            try:
                cursor.execute("""
                    INSERT INTO users (username, email, password_hash)
                    VALUES (%s, %s, %s)
                    RETURNING id
                """, (username, email, password_hash))
                
                user_id = cursor.fetchone()[0]
                return user_id
            except psycopg2.IntegrityError as e:
                print(f"Error creating user: {e}")
                return None
    
    @staticmethod
    def get_by_id(user_id: int) -> Optional[Dict]:
        """Get user by ID."""
        with get_db_connection() as conn:
            cursor = conn.cursor(cursor_factory=RealDictCursor)
            cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
            return cursor.fetchone()
    
    @staticmethod
    def get_by_username(username: str) -> Optional[Dict]:
        """Get user by username."""
        with get_db_connection() as conn:
            cursor = conn.cursor(cursor_factory=RealDictCursor)
            cursor.execute("SELECT * FROM users WHERE username = %s", (username,))
            return cursor.fetchone()
    
    @staticmethod
    def get_all(limit: int = 100, offset: int = 0) -> List[Dict]:
        """Get all users with pagination."""
        with get_db_connection() as conn:
            cursor = conn.cursor(cursor_factory=RealDictCursor)
            cursor.execute("""
                SELECT * FROM users 
                ORDER BY created_at DESC 
                LIMIT %s OFFSET %s
            """, (limit, offset))
            return cursor.fetchall()
    
    @staticmethod
    def update(user_id: int, **kwargs) -> bool:
        """Update user fields."""
        if not kwargs:
            return False
        
        # Build dynamic UPDATE query
        set_clause = ", ".join([f"{key} = %s" for key in kwargs.keys()])
        values = list(kwargs.values()) + [user_id]
        
        with get_db_connection() as conn:
            cursor = conn.cursor()
            cursor.execute(f"""
                UPDATE users 
                SET {set_clause}, updated_at = CURRENT_TIMESTAMP
                WHERE id = %s
            """, values)
            return cursor.rowcount > 0
    
    @staticmethod
    def delete(user_id: int) -> bool:
        """Delete a user."""
        with get_db_connection() as conn:
            cursor = conn.cursor()
            cursor.execute("DELETE FROM users WHERE id = %s", (user_id,))
            return cursor.rowcount > 0
    
    @staticmethod
    def search(query: str) -> List[Dict]:
        """Search users by username or email."""
        with get_db_connection() as conn:
            cursor = conn.cursor(cursor_factory=RealDictCursor)
            cursor.execute("""
                SELECT * FROM users 
                WHERE username ILIKE %s OR email ILIKE %s
                ORDER BY created_at DESC
            """, (f"%{query}%", f"%{query}%"))
            return cursor.fetchall()

class PostRepository:
    """Repository pattern for post operations."""
    
    @staticmethod
    def create(user_id: int, title: str, content: str, published: bool = False) -> Optional[int]:
        """Create a new post."""
        with get_db_connection() as conn:
            cursor = conn.cursor()
            cursor.execute("""
                INSERT INTO posts (user_id, title, content, published)
                VALUES (%s, %s, %s, %s)
                RETURNING id
            """, (user_id, title, content, published))
            return cursor.fetchone()[0]
    
    @staticmethod
    def get_by_id(post_id: int) -> Optional[Dict]:
        """Get post by ID with author information."""
        with get_db_connection() as conn:
            cursor = conn.cursor(cursor_factory=RealDictCursor)
            cursor.execute("""
                SELECT p.*, u.username as author_username, u.email as author_email
                FROM posts p
                JOIN users u ON p.user_id = u.id
                WHERE p.id = %s
            """, (post_id,))
            return cursor.fetchone()
    
    @staticmethod
    def get_user_posts(user_id: int) -> List[Dict]:
        """Get all posts by a user."""
        with get_db_connection() as conn:
            cursor = conn.cursor(cursor_factory=RealDictCursor)
            cursor.execute("""
                SELECT p.*, 
                       (SELECT COUNT(*) FROM comments WHERE post_id = p.id) as comment_count
                FROM posts p
                WHERE p.user_id = %s
                ORDER BY p.created_at DESC
            """, (user_id,))
            return cursor.fetchall()
    
    @staticmethod
    def get_published_posts(limit: int = 20, offset: int = 0) -> List[Dict]:
        """Get published posts with pagination."""
        with get_db_connection() as conn:
            cursor = conn.cursor(cursor_factory=RealDictCursor)
            cursor.execute("""
                SELECT p.*, u.username as author_username,
                       (SELECT COUNT(*) FROM comments WHERE post_id = p.id) as comment_count
                FROM posts p
                JOIN users u ON p.user_id = u.id
                WHERE p.published = TRUE
                ORDER BY p.created_at DESC
                LIMIT %s OFFSET %s
            """, (limit, offset))
            return cursor.fetchall()

# Example usage
if __name__ == "__main__":
    # Initialize connection pool
    initialize_pool()
    
    # Create tables
    create_tables()
    
    # Create user
    user_id = UserRepository.create("alice", "alice@example.com", "hashed_password")
    print(f"Created user with ID: {user_id}")
    
    # Get user
    user = UserRepository.get_by_username("alice")
    print(f"User: {user}")
    
    # Create post
    post_id = PostRepository.create(user_id, "My First Post", "Hello PostgreSQL!", True)
    print(f"Created post with ID: {post_id}")
    
    # Get posts
    posts = PostRepository.get_published_posts(limit=10)
    print(f"Found {len(posts)} published posts")
```

### Advanced PostgreSQL Features

```python
# Full-text search
def search_posts(query: str) -> List[Dict]:
    """Full-text search in posts."""
    with get_db_connection() as conn:
        cursor = conn.cursor(cursor_factory=RealDictCursor)
        cursor.execute("""
            SELECT p.*, u.username,
                   ts_rank(to_tsvector('english', p.title || ' ' || p.content), 
                          plainto_tsquery('english', %s)) as rank
            FROM posts p
            JOIN users u ON p.user_id = u.id
            WHERE to_tsvector('english', p.title || ' ' || p.content) @@ 
                  plainto_tsquery('english', %s)
            ORDER BY rank DESC
        """, (query, query))
        return cursor.fetchall()

# JSON operations (PostgreSQL 9.2+)
def create_user_profile(user_id: int, profile_data: dict):
    """Store JSON profile data."""
    with get_db_connection() as conn:
        cursor = conn.cursor()
        cursor.execute("""
            UPDATE users 
            SET profile_data = %s::jsonb
            WHERE id = %s
        """, (psycopg2.extras.Json(profile_data), user_id))

# Array operations
def add_tags_to_post(post_id: int, tags: List[str]):
    """Add tags to a post using PostgreSQL arrays."""
    with get_db_connection() as conn:
        cursor = conn.cursor()
        cursor.execute("""
            UPDATE posts 
            SET tags = %s
            WHERE id = %s
        """, (tags, post_id))

# Window functions
def get_top_users_by_posts():
    """Get users ranked by post count."""
    with get_db_connection() as conn:
        cursor = conn.cursor(cursor_factory=RealDictCursor)
        cursor.execute("""
            SELECT u.username, 
                   COUNT(p.id) as post_count,
                   RANK() OVER (ORDER BY COUNT(p.id) DESC) as rank
            FROM users u
            LEFT JOIN posts p ON u.id = p.user_id
            GROUP BY u.id, u.username
            ORDER BY post_count DESC
            LIMIT 10
        """)
        return cursor.fetchall()
```

---

## MySQL with Python

MySQL is another popular relational database management system.

### Installation

```bash
pip install mysql-connector-python
# or
pip install PyMySQL
```

### Basic MySQL Operations

```python
import mysql.connector
from mysql.connector import Error, pooling
from contextlib import contextmanager
from typing import List, Optional, Dict

# Database configuration
DB_CONFIG = {
    'host': 'localhost',
    'database': 'myapp',
    'user': 'root',
    'password': 'password',
    'port': 3306
}

# Connection pool
connection_pool = None

def initialize_pool():
    """Initialize connection pool."""
    global connection_pool
    try:
        connection_pool = mysql.connector.pooling.MySQLConnectionPool(
            pool_name="myapp_pool",
            pool_size=5,
            **DB_CONFIG
        )
        print("MySQL connection pool created")
    except Error as e:
        print(f"Error creating pool: {e}")

@contextmanager
def get_db_connection():
    """Get connection from pool."""
    conn = connection_pool.get_connection()
    cursor = conn.cursor(dictionary=True)
    try:
        yield cursor
        conn.commit()
    except Exception as e:
        conn.rollback()
        raise e
    finally:
        cursor.close()
        conn.close()

# Create tables
def create_tables():
    """Create tables in MySQL."""
    with get_db_connection() as cursor:
        # Users table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS users (
                id INT AUTO_INCREMENT PRIMARY KEY,
                username VARCHAR(50) UNIQUE NOT NULL,
                email VARCHAR(100) UNIQUE NOT NULL,
                password_hash VARCHAR(255) NOT NULL,
                is_active BOOLEAN DEFAULT TRUE,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                INDEX idx_username (username),
                INDEX idx_email (email)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
        """)
        
        # Posts table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS posts (
                id INT AUTO_INCREMENT PRIMARY KEY,
                user_id INT NOT NULL,
                title VARCHAR(200) NOT NULL,
                content TEXT,
                published BOOLEAN DEFAULT FALSE,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
                INDEX idx_user_id (user_id),
                INDEX idx_published (published)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
        """)
        
        print("MySQL tables created successfully")

# CRUD operations
class MySQLUserRepository:
    """User repository for MySQL."""
    
    @staticmethod
    def create(username: str, email: str, password_hash: str) -> Optional[int]:
        """Create a new user."""
        with get_db_connection() as cursor:
            try:
                cursor.execute("""
                    INSERT INTO users (username, email, password_hash)
                    VALUES (%s, %s, %s)
                """, (username, email, password_hash))
                return cursor.lastrowid
            except Error as e:
                print(f"Error creating user: {e}")
                return None
    
    @staticmethod
    def get_by_id(user_id: int) -> Optional[Dict]:
        """Get user by ID."""
        with get_db_connection() as cursor:
            cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
            return cursor.fetchone()
    
    @staticmethod
    def update(user_id: int, **kwargs) -> bool:
        """Update user fields."""
        if not kwargs:
            return False
        
        set_clause = ", ".join([f"{key} = %s" for key in kwargs.keys()])
        values = list(kwargs.values()) + [user_id]
        
        with get_db_connection() as cursor:
            cursor.execute(f"""
                UPDATE users 
                SET {set_clause}
                WHERE id = %s
            """, values)
            return cursor.rowcount > 0

# Example usage
if __name__ == "__main__":
    initialize_pool()
    create_tables()
    
    # Create user
    user_id = MySQLUserRepository.create("bob", "bob@example.com", "hashed_password")
    print(f"Created user: {user_id}")
```

---

## Raw SQL Operations

### SQL Query Basics

```python
# Basic SELECT queries
def basic_queries():
    """Examples of basic SQL queries."""
    
    # Simple SELECT
    query = "SELECT * FROM users"
    
    # SELECT with WHERE
    query = "SELECT * FROM users WHERE is_active = TRUE"
    
    # SELECT with ORDER BY
    query = "SELECT * FROM users ORDER BY created_at DESC"
    
    # SELECT with LIMIT
    query = "SELECT * FROM users LIMIT 10 OFFSET 20"
    
    # SELECT with JOIN
    query = """
        SELECT u.username, p.title, p.created_at
        FROM users u
        INNER JOIN posts p ON u.id = p.user_id
        WHERE p.published = TRUE
        ORDER BY p.created_at DESC
    """
    
    # SELECT with aggregation
    query = """
        SELECT u.username, COUNT(p.id) as post_count
        FROM users u
        LEFT JOIN posts p ON u.id = p.user_id
        GROUP BY u.id, u.username
        HAVING COUNT(p.id) > 5
        ORDER BY post_count DESC
    """
    
    # SELECT with subquery
    query = """
        SELECT * FROM users
        WHERE id IN (
            SELECT DISTINCT user_id FROM posts
            WHERE published = TRUE
        )
    """

# Complex joins
def complex_queries():
    """Examples of complex SQL queries."""
    
    # Multiple JOINs
    query = """
        SELECT u.username, p.title, c.content as comment
        FROM users u
        INNER JOIN posts p ON u.id = p.user_id
        LEFT JOIN comments c ON p.id = c.post_id
        WHERE p.published = TRUE
        ORDER BY p.created_at DESC, c.created_at ASC
    """
    
    # Self-join (e.g., for hierarchical data)
    query = """
        SELECT e1.name as employee, e2.name as manager
        FROM employees e1
        LEFT JOIN employees e2 ON e1.manager_id = e2.id
    """
    
    # UNION
    query = """
        SELECT 'user' as type, username as name FROM users
        UNION
        SELECT 'post' as type, title as name FROM posts
        ORDER BY type, name
    """

# Parameterized queries (prevents SQL injection)
def parameterized_queries(conn, username: str, email: str):
    """Always use parameterized queries!"""
    cursor = conn.cursor()
    
    # GOOD: Parameterized query
    cursor.execute(
        "SELECT * FROM users WHERE username = ? AND email = ?",
        (username, email)
    )
    
    # BAD: String formatting (SQL injection risk!)
    # cursor.execute(f"SELECT * FROM users WHERE username = '{username}'")
    # NEVER DO THIS!
```

### SQL Injection Prevention

```python
def vulnerable_query(conn, username: str):
    """VULNERABLE - DO NOT USE!"""
    cursor = conn.cursor()
    # This is vulnerable to SQL injection
    query = f"SELECT * FROM users WHERE username = '{username}'"
    cursor.execute(query)
    return cursor.fetchone()

def safe_query(conn, username: str):
    """SAFE - Always use parameterized queries."""
    cursor = conn.cursor()
    # This is safe from SQL injection
    cursor.execute("SELECT * FROM users WHERE username = ?", (username,))
    return cursor.fetchone()

# Example of SQL injection attack:
# If username = "admin' OR '1'='1"
# Vulnerable query becomes: SELECT * FROM users WHERE username = 'admin' OR '1'='1'
# This would return all users!
```

---

## SQLAlchemy ORM

SQLAlchemy is the most popular Python ORM, offering both high-level ORM and low-level SQL expression language.

### Installation

```bash
pip install sqlalchemy
```

### SQLAlchemy Core Concepts

```python
from sqlalchemy import create_engine, Column, Integer, String, Boolean, DateTime, Text, ForeignKey
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, relationship
from datetime import datetime

# Create base class for models
Base = declarative_base()

# Define models
class User(Base):
    """User model."""
    __tablename__ = 'users'
    
    id = Column(Integer, primary_key=True, autoincrement=True)
    username = Column(String(50), unique=True, nullable=False, index=True)
    email = Column(String(100), unique=True, nullable=False, index=True)
    password_hash = Column(String(255), nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    # Relationships
    posts = relationship("Post", back_populates="author", cascade="all, delete-orphan")
    comments = relationship("Comment", back_populates="user", cascade="all, delete-orphan")
    
    def __repr__(self):
        return f"<User(username='{self.username}', email='{self.email}')>"

class Post(Base):
    """Post model."""
    __tablename__ = 'posts'
    
    id = Column(Integer, primary_key=True, autoincrement=True)
    user_id = Column(Integer, ForeignKey('users.id', ondelete='CASCADE'), nullable=False)
    title = Column(String(200), nullable=False)
    content = Column(Text)
    published = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    # Relationships
    author = relationship("User", back_populates="posts")
    comments = relationship("Comment", back_populates="post", cascade="all, delete-orphan")
    
    def __repr__(self):
        return f"<Post(title='{self.title}')>"

class Comment(Base):
    """Comment model."""
    __tablename__ = 'comments'
    
    id = Column(Integer, primary_key=True, autoincrement=True)
    post_id = Column(Integer, ForeignKey('posts.id', ondelete='CASCADE'), nullable=False)
    user_id = Column(Integer, ForeignKey('users.id', ondelete='CASCADE'), nullable=False)
    content = Column(Text, nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    
    # Relationships
    post = relationship("Post", back_populates="comments")
    user = relationship("User", back_populates="comments")
    
    def __repr__(self):
        return f"<Comment(content='{self.content[:20]}...')>"

# Create engine and session
# SQLite
engine = create_engine('sqlite:///app.db', echo=True)

# PostgreSQL
# engine = create_engine('postgresql://user:password@localhost/dbname')

# MySQL
# engine = create_engine('mysql+pymysql://user:password@localhost/dbname')

# Create tables
Base.metadata.create_all(engine)

# Create session factory
Session = sessionmaker(bind=engine)
```

### CRUD Operations with SQLAlchemy

```python
from sqlalchemy.orm import Session
from sqlalchemy import and_, or_, desc, func
from typing import List, Optional

class UserRepository:
    """Repository for User operations."""
    
    def __init__(self, session: Session):
        self.session = session
    
    def create(self, username: str, email: str, password_hash: str) -> User:
        """Create a new user."""
        user = User(
            username=username,
            email=email,
            password_hash=password_hash
        )
        self.session.add(user)
        self.session.commit()
        self.session.refresh(user)
        return user
    
    def get_by_id(self, user_id: int) -> Optional[User]:
        """Get user by ID."""
        return self.session.query(User).filter(User.id == user_id).first()
    
    def get_by_username(self, username: str) -> Optional[User]:
        """Get user by username."""
        return self.session.query(User).filter(User.username == username).first()
    
    def get_all(self, skip: int = 0, limit: int = 100) -> List[User]:
        """Get all users with pagination."""
        return self.session.query(User)\
            .order_by(desc(User.created_at))\
            .offset(skip)\
            .limit(limit)\
            .all()
    
    def update(self, user_id: int, **kwargs) -> Optional[User]:
        """Update user."""
        user = self.get_by_id(user_id)
        if user:
            for key, value in kwargs.items():
                if hasattr(user, key):
                    setattr(user, key, value)
            self.session.commit()
            self.session.refresh(user)
        return user
    
    def delete(self, user_id: int) -> bool:
        """Delete user."""
        user = self.get_by_id(user_id)
        if user:
            self.session.delete(user)
            self.session.commit()
            return True
        return False
    
    def search(self, query: str) -> List[User]:
        """Search users."""
        search_pattern = f"%{query}%"
        return self.session.query(User)\
            .filter(
                or_(
                    User.username.ilike(search_pattern),
                    User.email.ilike(search_pattern)
                )
            )\
            .all()
    
    def get_active_users(self) -> List[User]:
        """Get all active users."""
        return self.session.query(User)\
            .filter(User.is_active == True)\
            .all()

class PostRepository:
    """Repository for Post operations."""
    
    def __init__(self, session: Session):
        self.session = session
    
    def create(self, user_id: int, title: str, content: str, 
               published: bool = False) -> Post:
        """Create a new post."""
        post = Post(
            user_id=user_id,
            title=title,
            content=content,
            published=published
        )
        self.session.add(post)
        self.session.commit()
        self.session.refresh(post)
        return post
    
    def get_by_id(self, post_id: int) -> Optional[Post]:
        """Get post by ID."""
        return self.session.query(Post).filter(Post.id == post_id).first()
    
    def get_user_posts(self, user_id: int) -> List[Post]:
        """Get all posts by a user."""
        return self.session.query(Post)\
            .filter(Post.user_id == user_id)\
            .order_by(desc(Post.created_at))\
            .all()
    
    def get_published_posts(self, skip: int = 0, limit: int = 20) -> List[Post]:
        """Get published posts."""
        return self.session.query(Post)\
            .filter(Post.published == True)\
            .order_by(desc(Post.created_at))\
            .offset(skip)\
            .limit(limit)\
            .all()
    
    def update(self, post_id: int, **kwargs) -> Optional[Post]:
        """Update post."""
        post = self.get_by_id(post_id)
        if post:
            for key, value in kwargs.items():
                if hasattr(post, key):
                    setattr(post, key, value)
            self.session.commit()
            self.session.refresh(post)
        return post
    
    def delete(self, post_id: int) -> bool:
        """Delete post."""
        post = self.get_by_id(post_id)
        if post:
            self.session.delete(post)
            self.session.commit()
            return True
        return False

# Example usage
if __name__ == "__main__":
    # Create session
    session = Session()
    
    try:
        # Create repositories
        user_repo = UserRepository(session)
        post_repo = PostRepository(session)
        
        # Create user
        user = user_repo.create("alice", "alice@example.com", "hashed_password")
        print(f"Created user: {user}")
        
        # Create post
        post = post_repo.create(user.id, "My First Post", "Hello SQLAlchemy!", True)
        print(f"Created post: {post}")
        
        # Query with relationships
        user_with_posts = session.query(User)\
            .filter(User.username == "alice")\
            .first()
        
        print(f"User {user_with_posts.username} has {len(user_with_posts.posts)} posts")
        for post in user_with_posts.posts:
            print(f"  - {post.title}")
        
        # Advanced queries
        # Count posts by user
        post_counts = session.query(
            User.username,
            func.count(Post.id).label('post_count')
        ).join(Post).group_by(User.id).all()
        
        print("\nPost counts:")
        for username, count in post_counts:
            print(f"{username}: {count} posts")
        
    finally:
        session.close()
```

### Advanced SQLAlchemy Features

```python
from sqlalchemy import Table, MetaData, select, and_, or_, func, case
from sqlalchemy.orm import joinedload, selectinload, subqueryload

# Eager loading to prevent N+1 queries
def get_users_with_posts(session: Session) -> List[User]:
    """Get users with their posts (prevents N+1 queries)."""
    # Using joinedload (single query with JOIN)
    users = session.query(User)\
        .options(joinedload(User.posts))\
        .all()
    
    # Using selectinload (separate query with IN)
    users = session.query(User)\
        .options(selectinload(User.posts))\
        .all()
    
    return users

# Subqueries
def get_users_with_post_count(session: Session):
    """Get users with their post count."""
    post_count = session.query(
        Post.user_id,
        func.count(Post.id).label('count')
    ).group_by(Post.user_id).subquery()
    
    results = session.query(
        User,
        func.coalesce(post_count.c.count, 0).label('post_count')
    ).outerjoin(post_count, User.id == post_count.c.user_id).all()
    
    return results

# Bulk operations
def bulk_create_users(session: Session, users_data: List[dict]):
    """Bulk create users efficiently."""
    users = [User(**data) for data in users_data]
    session.bulk_save_objects(users)
    session.commit()

def bulk_update_users(session: Session, updates: List[dict]):
    """Bulk update users."""
    session.bulk_update_mappings(User, updates)
    session.commit()

# Raw SQL with SQLAlchemy
def execute_raw_sql(session: Session, query: str, params: dict = None):
    """Execute raw SQL query."""
    result = session.execute(query, params or {})
    return result.fetchall()

# Context manager for sessions
from contextlib import contextmanager

@contextmanager
def get_session():
    """Provide a transactional scope for database operations."""
    session = Session()
    try:
        yield session
        session.commit()
    except Exception as e:
        session.rollback()
        raise e
    finally:
        session.close()

# Usage
with get_session() as session:
    user = User(username="bob", email="bob@example.com", password_hash="hash")
    session.add(user)
    # Automatically commits or rolls back
```

### SQLAlchemy with FastAPI

```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
from pydantic import BaseModel, EmailStr
from typing import List, Optional

app = FastAPI()

# Dependency to get DB session
def get_db():
    """Get database session."""
    db = Session()
    try:
        yield db
    finally:
        db.close()

# Pydantic schemas
class UserCreate(BaseModel):
    username: str
    email: EmailStr
    password: str

class UserResponse(BaseModel):
    id: int
    username: str
    email: EmailStr
    is_active: bool
    
    class Config:
        from_attributes = True

class PostCreate(BaseModel):
    title: str
    content: str
    published: bool = False

class PostResponse(BaseModel):
    id: int
    title: str
    content: str
    published: bool
    author: UserResponse
    
    class Config:
        from_attributes = True

# API endpoints
@app.post("/users/", response_model=UserResponse)
def create_user(user: UserCreate, db: Session = Depends(get_db)):
    """Create a new user."""
    # Check if user exists
    existing_user = db.query(User).filter(User.username == user.username).first()
    if existing_user:
        raise HTTPException(status_code=400, detail="Username already registered")
    
    # Create user (hash password in real app)
    db_user = User(
        username=user.username,
        email=user.email,
        password_hash=user.password  # Should be hashed!
    )
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user

@app.get("/users/", response_model=List[UserResponse])
def get_users(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    """Get all users."""
    users = db.query(User).offset(skip).limit(limit).all()
    return users

@app.get("/users/{user_id}", response_model=UserResponse)
def get_user(user_id: int, db: Session = Depends(get_db)):
    """Get user by ID."""
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

@app.post("/posts/", response_model=PostResponse)
def create_post(post: PostCreate, user_id: int, db: Session = Depends(get_db)):
    """Create a new post."""
    # Check if user exists
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    
    db_post = Post(
        user_id=user_id,
        title=post.title,
        content=post.content,
        published=post.published
    )
    db.add(db_post)
    db.commit()
    db.refresh(db_post)
    return db_post

@app.get("/posts/", response_model=List[PostResponse])
def get_posts(skip: int = 0, limit: int = 20, db: Session = Depends(get_db)):
    """Get all posts."""
    posts = db.query(Post)\
        .options(joinedload(Post.author))\
        .filter(Post.published == True)\
        .offset(skip)\
        .limit(limit)\
        .all()
    return posts
```

---

## Django ORM

Django ORM is part of the Django web framework and provides a high-level abstraction for database operations.

### Django Models

```python
# models.py
from django.db import models
from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    """Custom user model."""
    bio = models.TextField(max_length=500, blank=True)
    location = models.CharField(max_length=100, blank=True)
    birth_date = models.DateField(null=True, blank=True)
    avatar = models.ImageField(upload_to='avatars/', null=True, blank=True)
    
    class Meta:
        db_table = 'users'
        ordering = ['-date_joined']
    
    def __str__(self):
        return self.username

class Post(models.Model):
    """Post model."""
    author = models.ForeignKey(
        User,
        on_delete=models.CASCADE,
        related_name='posts'
    )
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200, unique=True)
    content = models.TextField()
    published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        db_table = 'posts'
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['author', '-created_at']),
            models.Index(fields=['published', '-created_at']),
        ]
    
    def __str__(self):
        return self.title

class Comment(models.Model):
    """Comment model."""
    post = models.ForeignKey(
        Post,
        on_delete=models.CASCADE,
        related_name='comments'
    )
    author = models.ForeignKey(
        User,
        on_delete=models.CASCADE,
        related_name='comments'
    )
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        db_table = 'comments'
        ordering = ['created_at']
    
    def __str__(self):
        return f"Comment by {self.author.username} on {self.post.title}"

class Tag(models.Model):
    """Tag model."""
    name = models.CharField(max_length=50, unique=True)
    posts = models.ManyToManyField(Post, related_name='tags')
    
    class Meta:
        db_table = 'tags'
        ordering = ['name']
    
    def __str__(self):
        return self.name
```

### Django ORM Queries

```python
from django.db.models import Q, Count, Avg, Sum, F, Prefetch
from django.utils import timezone
from datetime import timedelta

# Basic queries
def basic_django_queries():
    """Examples of basic Django ORM queries."""
    
    # Get all users
    users = User.objects.all()
    
    # Filter
    active_users = User.objects.filter(is_active=True)
    
    # Get single object
    user = User.objects.get(username='alice')
    # Or use get_or_404 in views
    # user = get_object_or_404(User, username='alice')
    
    # Get or create
    user, created = User.objects.get_or_create(
        username='bob',
        defaults={'email': 'bob@example.com'}
    )
    
    # Exclude
    inactive_users = User.objects.exclude(is_active=True)
    
    # Order by
    users = User.objects.order_by('-date_joined')
    
    # Limit
    recent_users = User.objects.all()[:10]
    
    # Count
    user_count = User.objects.count()
    
    # Exists
    has_posts = Post.objects.filter(published=True).exists()

# Complex queries
def complex_django_queries():
    """Examples of complex Django ORM queries."""
    
    # Q objects for complex filters
    users = User.objects.filter(
        Q(username__icontains='john') | Q(email__icontains='john')
    )
    
    # Multiple conditions
    posts = Post.objects.filter(
        Q(published=True) & (Q(author__username='alice') | Q(title__icontains='python'))
    )
    
    # Negation
    posts = Post.objects.filter(~Q(published=True))
    
    # F expressions (field comparisons)
    # Get posts where updated_at is different from created_at
    edited_posts = Post.objects.filter(updated_at__gt=F('created_at'))
    
    # Aggregation
    from django.db.models import Count, Avg
    
    # Count posts per user
    users_with_counts = User.objects.annotate(
        post_count=Count('posts')
    ).filter(post_count__gt=5)
    
    # Average comments per post
    avg_comments = Post.objects.aggregate(
        avg_comments=Avg('comments__count')
    )
    
    # Select related (for foreign keys - uses JOIN)
    posts = Post.objects.select_related('author').all()
    
    # Prefetch related (for many-to-many and reverse foreign keys)
    posts = Post.objects.prefetch_related('comments', 'tags').all()
    
    # Custom prefetch
    recent_comments = Comment.objects.filter(
        created_at__gte=timezone.now() - timedelta(days=7)
    )
    posts = Post.objects.prefetch_related(
        Prefetch('comments', queryset=recent_comments, to_attr='recent_comments')
    ).all()
    
    # Values and values_list
    usernames = User.objects.values_list('username', flat=True)
    user_data = User.objects.values('id', 'username', 'email')
    
    # Distinct
    post_authors = User.objects.filter(posts__published=True).distinct()

# Create operations
def create_operations():
    """Create operations in Django ORM."""
    
    # Create single object
    user = User.objects.create(
        username='charlie',
        email='charlie@example.com'
    )
    
    # Or create and save separately
    user = User(username='david', email='david@example.com')
    user.save()
    
    # Bulk create
    users = [
        User(username=f'user{i}', email=f'user{i}@example.com')
        for i in range(100)
    ]
    User.objects.bulk_create(users)
    
    # Create related objects
    post = Post.objects.create(
        author=user,
        title='My Post',
        content='Content here'
    )
    
    # Add many-to-many
    tag1 = Tag.objects.create(name='python')
    tag2 = Tag.objects.create(name='django')
    post.tags.add(tag1, tag2)

# Update operations
def update_operations():
    """Update operations in Django ORM."""
    
    # Update single object
    user = User.objects.get(username='alice')
    user.bio = 'Updated bio'
    user.save()
    
    # Update specific fields
    user.save(update_fields=['bio'])
    
    # Bulk update
    User.objects.filter(is_active=False).update(is_active=True)
    
    # Update with F expressions
    Post.objects.filter(published=True).update(
        views=F('views') + 1
    )
    
    # Bulk update many objects
    users = User.objects.filter(date_joined__year=2024)
    for user in users:
        user.is_active = True
    User.objects.bulk_update(users, ['is_active'])

# Delete operations
def delete_operations():
    """Delete operations in Django ORM."""
    
    # Delete single object
    user = User.objects.get(username='test')
    user.delete()
    
    # Bulk delete
    User.objects.filter(is_active=False).delete()
    
    # Delete all (be careful!)
    # Post.objects.all().delete()

# Transactions
from django.db import transaction

def transaction_example():
    """Transaction example."""
    
    try:
        with transaction.atomic():
            user = User.objects.create(username='test', email='test@example.com')
            post = Post.objects.create(
                author=user,
                title='Test Post',
                content='Test content'
            )
            # If any error occurs, both operations will be rolled back
    except Exception as e:
        print(f"Transaction failed: {e}")
```

### Django Managers and QuerySets

```python
# Custom manager
class PublishedManager(models.Manager):
    """Manager for published posts."""
    
    def get_queryset(self):
        return super().get_queryset().filter(published=True)

class Post(models.Model):
    # ... fields ...
    
    objects = models.Manager()  # Default manager
    published = PublishedManager()  # Custom manager
    
    # Now you can use: Post.published.all()

# Custom QuerySet
class PostQuerySet(models.QuerySet):
    """Custom queryset for posts."""
    
    def published(self):
        return self.filter(published=True)
    
    def by_author(self, author):
        return self.filter(author=author)
    
    def recent(self, days=7):
        return self.filter(
            created_at__gte=timezone.now() - timedelta(days=days)
        )

class PostManager(models.Manager):
    def get_queryset(self):
        return PostQuerySet(self.model, using=self._db)
    
    def published(self):
        return self.get_queryset().published()
    
    def recent(self, days=7):
        return self.get_queryset().recent(days)

class Post(models.Model):
    # ... fields ...
    
    objects = PostManager()
    
    # Now you can chain: Post.objects.published().recent(30)
```

### Django Signals

```python
# signals.py
from django.db.models.signals import post_save, pre_delete
from django.dispatch import receiver
from django.core.mail import send_mail

@receiver(post_save, sender=User)
def user_created(sender, instance, created, **kwargs):
    """Send welcome email when user is created."""
    if created:
        send_mail(
            'Welcome!',
            f'Welcome to our site, {instance.username}!',
            'noreply@example.com',
            [instance.email],
        )

@receiver(post_save, sender=Post)
def post_published(sender, instance, created, **kwargs):
    """Notify followers when post is published."""
    if not created and instance.published:
        # Check if published status changed
        if instance.tracker.has_changed('published'):
            # Notify followers
            pass

@receiver(pre_delete, sender=User)
def user_deleted(sender, instance, **kwargs):
    """Clean up user data before deletion."""
    # Delete user's uploaded files, etc.
    if instance.avatar:
        instance.avatar.delete()
```

---

## Database Migrations

### SQLAlchemy with Alembic

```bash
pip install alembic
```

```python
# Initialize Alembic
# alembic init alembic

# alembic.ini - configure database URL
# sqlalchemy.url = sqlite:///./app.db

# alembic/env.py
from myapp.models import Base
target_metadata = Base.metadata

# Create migration
# alembic revision --autogenerate -m "Create users table"

# Apply migration
# alembic upgrade head

# Rollback migration
# alembic downgrade -1
```

```python
# Example migration file
"""create users table

Revision ID: 001
Revises: 
Create Date: 2024-01-01 12:00:00
"""
from alembic import op
import sqlalchemy as sa

revision = '001'
down_revision = None
branch_labels = None
depends_on = None

def upgrade():
    op.create_table(
        'users',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('username', sa.String(50), nullable=False),
        sa.Column('email', sa.String(100), nullable=False),
        sa.Column('created_at', sa.DateTime(), nullable=True),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('username'),
        sa.UniqueConstraint('email')
    )
    op.create_index('idx_username', 'users', ['username'])

def downgrade():
    op.drop_index('idx_username', table_name='users')
    op.drop_table('users')
```

### Django Migrations

```bash
# Create migrations
python manage.py makemigrations

# Apply migrations
python manage.py migrate

# Show migrations
python manage.py showmigrations

# Rollback migration
python manage.py migrate app_name migration_name

# Create empty migration
python manage.py makemigrations --empty app_name
```

```python
# Example custom migration
from django.db import migrations

def populate_tags(apps, schema_editor):
    Tag = apps.get_model('blog', 'Tag')
    Tag.objects.bulk_create([
        Tag(name='Python'),
        Tag(name='Django'),
        Tag(name='Web Development'),
    ])

def reverse_populate_tags(apps, schema_editor):
    Tag = apps.get_model('blog', 'Tag')
    Tag.objects.filter(name__in=['Python', 'Django', 'Web Development']).delete()

class Migration(migrations.Migration):
    dependencies = [
        ('blog', '0001_initial'),
    ]
    
    operations = [
        migrations.RunPython(populate_tags, reverse_populate_tags),
    ]
```

---

## Best Practices

### 1. Use Connection Pooling

```python
# SQLAlchemy
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    'postgresql://user:pass@localhost/db',
    poolclass=QueuePool,
    pool_size=10,
    max_overflow=20,
    pool_timeout=30,
    pool_recycle=3600
)
```

### 2. Use Indexes Strategically

```python
# SQLAlchemy
class User(Base):
    __tablename__ = 'users'
    
    id = Column(Integer, primary_key=True)
    username = Column(String(50), index=True)  # Single column index
    email = Column(String(100), index=True)
    created_at = Column(DateTime, index=True)
    
    # Composite index
    __table_args__ = (
        Index('idx_username_email', 'username', 'email'),
    )

# Django
class Post(models.Model):
    # ... fields ...
    
    class Meta:
        indexes = [
            models.Index(fields=['author', '-created_at']),
            models.Index(fields=['published', '-created_at']),
        ]
```

### 3. Avoid N+1 Queries

```python
# Bad - N+1 query problem
users = User.objects.all()
for user in users:
    print(user.posts.all())  # Separate query for each user!

# Good - Use select_related or prefetch_related
users = User.objects.prefetch_related('posts').all()
for user in users:
    print(user.posts.all())  # No additional queries

# SQLAlchemy
users = session.query(User).options(joinedload(User.posts)).all()
```

### 4. Use Transactions

```python
# SQLAlchemy
from sqlalchemy.orm import Session

with Session() as session:
    try:
        user = User(username='test')
        session.add(user)
        post = Post(user_id=user.id, title='Test')
        session.add(post)
        session.commit()
    except Exception as e:
        session.rollback()
        raise

# Django
from django.db import transaction

with transaction.atomic():
    user = User.objects.create(username='test')
    Post.objects.create(author=user, title='Test')
```

### 5. Use Pagination

```python
# Django
from django.core.paginator import Paginator

def list_posts(request):
    posts = Post.objects.all()
    paginator = Paginator(posts, 25)  # 25 posts per page
    page_number = request.GET.get('page')
    page_obj = paginator.get_page(page_number)
    return render(request, 'posts.html', {'page_obj': page_obj})

# SQLAlchemy
def get_paginated_posts(session, page=1, per_page=25):
    offset = (page - 1) * per_page
    posts = session.query(Post)\
        .offset(offset)\
        .limit(per_page)\
        .all()
    return posts
```

### 6. Use Database Constraints

```python
# Ensure data integrity at database level
class User(Base):
    __tablename__ = 'users'
    
    email = Column(String(100), unique=True, nullable=False)
    age = Column(Integer, CheckConstraint('age >= 18'))
    
    __table_args__ = (
        CheckConstraint('email LIKE "%@%"', name='valid_email'),
    )
```

### 7. Monitor Query Performance

```python
# SQLAlchemy - Log slow queries
import logging

logging.basicConfig()
logging.getLogger('sqlalchemy.engine').setLevel(logging.INFO)

# Django - Use Django Debug Toolbar
# pip install django-debug-toolbar

# Or log queries manually
from django.db import connection
print(len(connection.queries))
for query in connection.queries:
    print(query['sql'])
```

---

## Advanced Topics

### Database Sharding

```python
# Horizontal partitioning example
class ShardedUserRepository:
    """Repository that shards users across multiple databases."""
    
    def __init__(self, num_shards=4):
        self.engines = [
            create_engine(f'postgresql://localhost/shard_{i}')
            for i in range(num_shards)
        ]
    
    def get_shard(self, user_id):
        """Determine which shard a user belongs to."""
        return user_id % len(self.engines)
    
    def get_user(self, user_id):
        """Get user from appropriate shard."""
        shard_id = self.get_shard(user_id)
        engine = self.engines[shard_id]
        Session = sessionmaker(bind=engine)
        session = Session()
        return session.query(User).filter(User.id == user_id).first()
```

### Read Replicas

```python
# Django - Database routing
class ReplicaRouter:
    """Route reads to replica, writes to primary."""
    
    def db_for_read(self, model, **hints):
        """Send reads to replica."""
        return 'replica'
    
    def db_for_write(self, model, **hints):
        """Send writes to primary."""
        return 'default'
    
    def allow_relation(self, obj1, obj2, **hints):
        """Allow relations."""
        return True
    
    def allow_migrate(self, db, app_label, model_name=None, **hints):
        """Only migrate on primary."""
        return db == 'default'

# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'primary_db',
        # ... other settings
    },
    'replica': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'replica_db',
        # ... other settings
    }
}

DATABASE_ROUTERS = ['myapp.routers.ReplicaRouter']
```

### Caching

```python
# Django caching
from django.core.cache import cache
from django.views.decorators.cache import cache_page

# Cache view for 15 minutes
@cache_page(60 * 15)
def my_view(request):
    # ...
    pass

# Manual caching
def get_user_posts(user_id):
    cache_key = f'user_posts_{user_id}'
    posts = cache.get(cache_key)
    
    if posts is None:
        posts = list(Post.objects.filter(author_id=user_id))
        cache.set(cache_key, posts, 300)  # Cache for 5 minutes
    
    return posts

# SQLAlchemy with Redis caching
import redis
import json

redis_client = redis.Redis(host='localhost', port=6379, db=0)

def get_cached_user(user_id):
    cache_key = f'user:{user_id}'
    cached_data = redis_client.get(cache_key)
    
    if cached_data:
        return json.loads(cached_data)
    
    session = Session()
    user = session.query(User).filter(User.id == user_id).first()
    
    if user:
        redis_client.setex(
            cache_key,
            300,  # 5 minutes
            json.dumps({
                'id': user.id,
                'username': user.username,
                'email': user.email
            })
        )
    
    return user
```

---

## Summary

This comprehensive guide covered:

1. **Database Fundamentals**: Core concepts and SQL basics
2. **SQLite**: Lightweight database perfect for development
3. **PostgreSQL**: Powerful relational database with advanced features
4. **MySQL**: Popular open-source database
5. **SQLAlchemy ORM**: Flexible and powerful Python ORM
6. **Django ORM**: High-level ORM integrated with Django
7. **Migrations**: Managing database schema changes
8. **Best Practices**: Performance, security, and maintainability
9. **Advanced Topics**: Sharding, replication, and caching

Choose the right tool for your needs:
- **SQLite**: Small projects, development, testing
- **PostgreSQL**: Production apps, complex queries, JSON data
- **MySQL**: High-traffic web applications
- **SQLAlchemy**: Flexibility, database-agnostic code
- **Django ORM**: Rapid development with Django framework

Remember: Always validate user input, use parameterized queries, implement proper indexing, and monitor query performance!