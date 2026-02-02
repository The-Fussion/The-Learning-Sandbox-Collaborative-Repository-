# Deployment and Production in Python

## Table of Contents
1. [Introduction to Deployment](#introduction-to-deployment)
2. [Environment Variables and Configuration](#environment-variables-and-configuration)
3. [Configuration Management](#configuration-management)
4. [Production-Ready Code](#production-ready-code)
5. [WSGI and ASGI Servers](#wsgi-and-asgi-servers)
6. [Reverse Proxies with Nginx](#reverse-proxies-with-nginx)
7. [Process Management](#process-management)
8. [Database Migrations](#database-migrations)
9. [Static Files in Production](#static-files-in-production)
10. [Logging and Monitoring](#logging-and-monitoring)
11. [Docker Deployment](#docker-deployment)
12. [Cloud Platform Deployment](#cloud-platform-deployment)
13. [CI/CD Pipelines](#cicd-pipelines)
14. [Security Hardening](#security-hardening)
15. [Performance Optimization](#performance-optimization)
16. [Best Practices](#best-practices)

---

## Introduction to Deployment

### Development vs Production

```python
"""
Development Environment:
- Debug mode enabled
- SQLite database
- Development server (Flask dev server, Django runserver)
- Hot reloading
- Detailed error messages
- No caching
- Local file storage

Production Environment:
- Debug mode disabled
- PostgreSQL/MySQL database
- Production WSGI/ASGI server (Gunicorn, uWSGI)
- No hot reloading
- Generic error messages
- Aggressive caching
- Cloud file storage (S3)
- HTTPS enabled
- Load balancing
- Monitoring and logging

Key Differences:
┌────────────────────┬──────────────────┬──────────────────┐
│     Aspect         │   Development    │    Production    │
├────────────────────┼──────────────────┼──────────────────┤
│ Debug Mode         │   Enabled        │    Disabled      │
│ Database           │   SQLite         │    PostgreSQL    │
│ Server             │   Dev Server     │    Gunicorn      │
│ SSL/HTTPS          │   Optional       │    Required      │
│ Error Messages     │   Detailed       │    Generic       │
│ Static Files       │   Dev Server     │    Nginx/CDN     │
│ Logging            │   Console        │    Files/Service │
│ Environment        │   .env file      │    Env vars      │
│ Dependencies       │   All            │    Production    │
└────────────────────┴──────────────────┴──────────────────┘
"""
```

### Deployment Checklist

```python
"""
✅ Pre-Deployment Checklist:

1. Code Quality
   ✅ All tests passing
   ✅ Code reviewed
   ✅ No debug code
   ✅ No hardcoded credentials
   ✅ Dependencies updated

2. Configuration
   ✅ Environment variables set
   ✅ Database configured
   ✅ Secret key generated
   ✅ Allowed hosts configured
   ✅ Debug mode disabled

3. Security
   ✅ HTTPS enabled
   ✅ Security headers set
   ✅ CSRF protection enabled
   ✅ SQL injection prevented
   ✅ XSS prevention implemented

4. Performance
   ✅ Static files optimized
   ✅ Database indexed
   ✅ Caching configured
   ✅ Gzip compression enabled
   ✅ CDN configured

5. Monitoring
   ✅ Logging configured
   ✅ Error tracking set up
   ✅ Performance monitoring
   ✅ Uptime monitoring
   ✅ Backup strategy

6. Infrastructure
   ✅ Server provisioned
   ✅ Database created
   ✅ Domain configured
   ✅ SSL certificate installed
   ✅ Firewall configured
"""
```

---

## Environment Variables and Configuration

### Using python-dotenv

```bash
# Install python-dotenv
pip install python-dotenv
```

```python
# .env file (DO NOT COMMIT TO GIT!)
SECRET_KEY=your-secret-key-here
DATABASE_URL=postgresql://user:password@localhost/dbname
DEBUG=False
ALLOWED_HOSTS=example.com,www.example.com
REDIS_URL=redis://localhost:6379
AWS_ACCESS_KEY_ID=your-access-key
AWS_SECRET_ACCESS_KEY=your-secret-key
SENTRY_DSN=https://your-sentry-dsn
```

```python
# config.py

from dotenv import load_dotenv
import os

# Load environment variables from .env file
load_dotenv()

class Config:
    """Base configuration."""
    
    # Secret key for sessions
    SECRET_KEY = os.environ.get('SECRET_KEY') or 'dev-secret-key'
    
    # Database
    DATABASE_URL = os.environ.get('DATABASE_URL') or 'sqlite:///dev.db'
    
    # Application
    DEBUG = os.environ.get('DEBUG', 'False').lower() == 'true'
    TESTING = False
    
    # Server
    HOST = os.environ.get('HOST', '0.0.0.0')
    PORT = int(os.environ.get('PORT', 5000))
    
    # Security
    SESSION_COOKIE_SECURE = True
    SESSION_COOKIE_HTTPONLY = True
    SESSION_COOKIE_SAMESITE = 'Lax'
    
    # Static files
    STATIC_FOLDER = 'static'
    STATIC_URL = os.environ.get('STATIC_URL', '/static/')
    
    # File uploads
    UPLOAD_FOLDER = os.environ.get('UPLOAD_FOLDER', 'uploads')
    MAX_CONTENT_LENGTH = 16 * 1024 * 1024  # 16MB
    
    # Redis
    REDIS_URL = os.environ.get('REDIS_URL', 'redis://localhost:6379')
    
    # Email
    MAIL_SERVER = os.environ.get('MAIL_SERVER')
    MAIL_PORT = int(os.environ.get('MAIL_PORT', 587))
    MAIL_USE_TLS = os.environ.get('MAIL_USE_TLS', 'True').lower() == 'true'
    MAIL_USERNAME = os.environ.get('MAIL_USERNAME')
    MAIL_PASSWORD = os.environ.get('MAIL_PASSWORD')
    
    # AWS
    AWS_ACCESS_KEY_ID = os.environ.get('AWS_ACCESS_KEY_ID')
    AWS_SECRET_ACCESS_KEY = os.environ.get('AWS_SECRET_ACCESS_KEY')
    AWS_S3_BUCKET = os.environ.get('AWS_S3_BUCKET')
    
    # Monitoring
    SENTRY_DSN = os.environ.get('SENTRY_DSN')

class DevelopmentConfig(Config):
    """Development configuration."""
    DEBUG = True
    TESTING = False
    DATABASE_URL = 'sqlite:///dev.db'

class ProductionConfig(Config):
    """Production configuration."""
    DEBUG = False
    TESTING = False
    
    # Ensure required variables are set
    @classmethod
    def init_app(cls, app):
        assert os.environ.get('SECRET_KEY'), 'SECRET_KEY must be set'
        assert os.environ.get('DATABASE_URL'), 'DATABASE_URL must be set'

class TestingConfig(Config):
    """Testing configuration."""
    DEBUG = False
    TESTING = True
    DATABASE_URL = 'sqlite:///:memory:'

# Configuration dictionary
config = {
    'development': DevelopmentConfig,
    'production': ProductionConfig,
    'testing': TestingConfig,
    'default': DevelopmentConfig
}

# Get configuration
def get_config(env=None):
    """Get configuration based on environment."""
    if env is None:
        env = os.environ.get('FLASK_ENV', 'development')
    
    return config.get(env, config['default'])
```

### Flask Application Setup

```python
# app.py

from flask import Flask
from config import get_config
import os

def create_app(config_name=None):
    """Application factory."""
    app = Flask(__name__)
    
    # Load configuration
    if config_name is None:
        config_name = os.environ.get('FLASK_ENV', 'development')
    
    app.config.from_object(get_config(config_name))
    
    # Initialize extensions
    # db.init_app(app)
    # migrate.init_app(app, db)
    # login_manager.init_app(app)
    
    # Register blueprints
    # app.register_blueprint(main_bp)
    # app.register_blueprint(api_bp)
    
    return app

if __name__ == '__main__':
    app = create_app()
    app.run()
```

### FastAPI Configuration

```python
# config.py

from pydantic_settings import BaseSettings
from functools import lru_cache

class Settings(BaseSettings):
    """Application settings."""
    
    # Application
    app_name: str = "My API"
    debug: bool = False
    
    # Database
    database_url: str = "sqlite:///./test.db"
    
    # Security
    secret_key: str
    algorithm: str = "HS256"
    access_token_expire_minutes: int = 30
    
    # CORS
    allowed_origins: list = ["*"]
    
    # Redis
    redis_url: str = "redis://localhost:6379"
    
    # AWS
    aws_access_key_id: str | None = None
    aws_secret_access_key: str | None = None
    aws_region: str = "us-east-1"
    s3_bucket: str | None = None
    
    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"

@lru_cache()
def get_settings() -> Settings:
    """Get cached settings."""
    return Settings()

# Usage
from fastapi import FastAPI, Depends

app = FastAPI()

@app.get("/info")
async def info(settings: Settings = Depends(get_settings)):
    return {
        "app_name": settings.app_name,
        "debug": settings.debug
    }
```

### Django Settings

```python
# settings.py

import os
from pathlib import Path
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

# Build paths
BASE_DIR = Path(__file__).resolve().parent.parent

# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = os.environ.get('SECRET_KEY')

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = os.environ.get('DEBUG', 'False').lower() == 'true'

ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', '').split(',')

# Database
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME'),
        'USER': os.environ.get('DB_USER'),
        'PASSWORD': os.environ.get('DB_PASSWORD'),
        'HOST': os.environ.get('DB_HOST', 'localhost'),
        'PORT': os.environ.get('DB_PORT', '5432'),
    }
}

# Static files
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')

# Media files
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')

# Security settings for production
if not DEBUG:
    SECURE_SSL_REDIRECT = True
    SESSION_COOKIE_SECURE = True
    CSRF_COOKIE_SECURE = True
    SECURE_BROWSER_XSS_FILTER = True
    SECURE_CONTENT_TYPE_NOSNIFF = True
    X_FRAME_OPTIONS = 'DENY'
    SECURE_HSTS_SECONDS = 31536000
    SECURE_HSTS_INCLUDE_SUBDOMAINS = True
    SECURE_HSTS_PRELOAD = True
```

---

## Configuration Management

### Using ConfigParser

```python
# config.ini

[DEFAULT]
ServerAliveInterval = 45
Compression = yes
CompressionLevel = 9

[development]
database_url = sqlite:///dev.db
debug = true
log_level = DEBUG

[production]
database_url = postgresql://user:password@localhost/prod
debug = false
log_level = INFO
allowed_hosts = example.com,www.example.com
```

```python
# Read configuration
import configparser

config = configparser.ConfigParser()
config.read('config.ini')

# Access values
db_url = config['production']['database_url']
debug = config.getboolean('production', 'debug')
allowed_hosts = config['production']['allowed_hosts'].split(',')
```

### Using YAML Configuration

```bash
pip install pyyaml
```

```yaml
# config.yaml

common: &common
  debug: false
  log_level: INFO
  
development:
  <<: *common
  debug: true
  database:
    engine: sqlite
    name: dev.db
  
production:
  <<: *common
  database:
    engine: postgresql
    host: localhost
    port: 5432
    name: prod_db
    user: ${DB_USER}
    password: ${DB_PASSWORD}
  
  redis:
    host: localhost
    port: 6379
    db: 0
  
  aws:
    region: us-east-1
    s3_bucket: ${S3_BUCKET}
```

```python
# Load YAML configuration
import yaml
import os

def load_config(env='development'):
    """Load configuration from YAML."""
    with open('config.yaml', 'r') as f:
        config = yaml.safe_load(f)
    
    env_config = config.get(env, {})
    
    # Replace environment variables
    return replace_env_vars(env_config)

def replace_env_vars(config):
    """Replace ${VAR} with environment variables."""
    if isinstance(config, dict):
        return {k: replace_env_vars(v) for k, v in config.items()}
    elif isinstance(config, list):
        return [replace_env_vars(item) for item in config]
    elif isinstance(config, str):
        # Replace ${VAR} with environment variable
        import re
        pattern = r'\$\{([^}]+)\}'
        
        def replacer(match):
            var_name = match.group(1)
            return os.environ.get(var_name, match.group(0))
        
        return re.sub(pattern, replacer, config)
    else:
        return config

# Usage
config = load_config('production')
db_config = config['database']
```

---

## Production-Ready Code

### Error Handling

```python
from flask import Flask, jsonify
import logging
import sentry_sdk
from sentry_sdk.integrations.flask import FlaskIntegration

app = Flask(__name__)

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('app.log'),
        logging.StreamHandler()
    ]
)

logger = logging.getLogger(__name__)

# Initialize Sentry for error tracking
if app.config.get('SENTRY_DSN'):
    sentry_sdk.init(
        dsn=app.config['SENTRY_DSN'],
        integrations=[FlaskIntegration()],
        traces_sample_rate=1.0,
        environment=app.config.get('FLASK_ENV', 'production')
    )

@app.errorhandler(404)
def not_found(error):
    """Handle 404 errors."""
    logger.warning(f'404 error: {request.url}')
    return jsonify({'error': 'Not found'}), 404

@app.errorhandler(500)
def internal_error(error):
    """Handle 500 errors."""
    logger.error(f'500 error: {str(error)}', exc_info=True)
    
    # Don't expose internal errors in production
    if app.debug:
        return jsonify({'error': str(error)}), 500
    else:
        return jsonify({'error': 'Internal server error'}), 500

@app.errorhandler(Exception)
def handle_exception(error):
    """Handle all uncaught exceptions."""
    logger.error(f'Uncaught exception: {str(error)}', exc_info=True)
    
    # Sentry will automatically capture this
    
    return jsonify({'error': 'An unexpected error occurred'}), 500
```

### Health Check Endpoint

```python
from flask import Flask, jsonify
from datetime import datetime
import psutil

@app.route('/health')
def health_check():
    """Health check endpoint for monitoring."""
    try:
        # Check database connection
        db.session.execute('SELECT 1')
        db_status = 'healthy'
    except Exception as e:
        db_status = 'unhealthy'
        logger.error(f'Database health check failed: {e}')
    
    # System metrics
    cpu_percent = psutil.cpu_percent()
    memory = psutil.virtual_memory()
    disk = psutil.disk_usage('/')
    
    health_status = {
        'status': 'healthy' if db_status == 'healthy' else 'degraded',
        'timestamp': datetime.utcnow().isoformat(),
        'version': '1.0.0',
        'database': db_status,
        'system': {
            'cpu_percent': cpu_percent,
            'memory_percent': memory.percent,
            'disk_percent': disk.percent
        }
    }
    
    status_code = 200 if health_status['status'] == 'healthy' else 503
    
    return jsonify(health_status), status_code

@app.route('/ready')
def readiness_check():
    """Readiness check for load balancer."""
    # Check if application is ready to serve traffic
    try:
        # Check critical dependencies
        db.session.execute('SELECT 1')
        # Check Redis connection
        redis_client.ping()
        
        return jsonify({'status': 'ready'}), 200
    except Exception as e:
        logger.error(f'Readiness check failed: {e}')
        return jsonify({'status': 'not ready'}), 503
```

---

## WSGI and ASGI Servers

### Gunicorn (WSGI)

```bash
# Install Gunicorn
pip install gunicorn
```

```python
# gunicorn_config.py

import os
import multiprocessing

# Server socket
bind = f"0.0.0.0:{os.environ.get('PORT', 8000)}"
backlog = 2048

# Worker processes
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = 'sync'
worker_connections = 1000
timeout = 30
keepalive = 2

# Logging
accesslog = '-'  # stdout
errorlog = '-'   # stderr
loglevel = 'info'
access_log_format = '%(h)s %(l)s %(u)s %(t)s "%(r)s" %(s)s %(b)s "%(f)s" "%(a)s"'

# Process naming
proc_name = 'myapp'

# Server mechanics
daemon = False
pidfile = None
umask = 0
user = None
group = None
tmp_upload_dir = None

# SSL
keyfile = os.environ.get('SSL_KEYFILE')
certfile = os.environ.get('SSL_CERTFILE')
```

```bash
# Run Gunicorn
gunicorn -c gunicorn_config.py app:app

# Or with command line options
gunicorn --workers 4 --bind 0.0.0.0:8000 --timeout 30 app:app
```

### Uvicorn (ASGI) for FastAPI

```bash
# Install Uvicorn
pip install uvicorn[standard]
```

```python
# main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}
```

```bash
# Run Uvicorn in production
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4

# With SSL
uvicorn main:app --host 0.0.0.0 --port 443 \
  --ssl-keyfile=/path/to/key.pem \
  --ssl-certfile=/path/to/cert.pem

# With Gunicorn as process manager
gunicorn main:app -w 4 -k uvicorn.workers.UvicornWorker --bind 0.0.0.0:8000
```

### uWSGI

```bash
# Install uWSGI
pip install uwsgi
```

```ini
# uwsgi.ini

[uwsgi]
# Application
module = app:app
callable = app

# Master process
master = true

# Workers
processes = 4
threads = 2

# Socket
http-socket = 0.0.0.0:8000
# Or use unix socket for Nginx
# socket = /tmp/uwsgi.sock
# chmod-socket = 666

# Performance
enable-threads = true
single-interpreter = true
need-app = true

# Buffer size
buffer-size = 32768

# Logging
logto = /var/log/uwsgi/app.log
log-maxsize = 20971520

# Reload
touch-reload = /tmp/reload.txt
py-autoreload = 1

# Process management
vacuum = true
die-on-term = true
```

```bash
# Run uWSGI
uwsgi --ini uwsgi.ini
```

---

## Reverse Proxies with Nginx

### Basic Nginx Configuration

```nginx
# /etc/nginx/sites-available/myapp

server {
    listen 80;
    server_name example.com www.example.com;
    
    # Redirect HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com www.example.com;
    
    # SSL Configuration
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    
    # Logs
    access_log /var/log/nginx/myapp_access.log;
    error_log /var/log/nginx/myapp_error.log;
    
    # Max upload size
    client_max_body_size 10M;
    
    # Static files
    location /static {
        alias /var/www/myapp/static;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
    
    # Media files
    location /media {
        alias /var/www/myapp/media;
        expires 30d;
        add_header Cache-Control "public";
    }
    
    # Proxy to application
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

### Nginx with Load Balancing

```nginx
# Upstream servers
upstream app_servers {
    least_conn;  # Load balancing method
    
    server 127.0.0.1:8001 weight=3;
    server 127.0.0.1:8002 weight=2;
    server 127.0.0.1:8003;
    
    # Health check
    keepalive 32;
}

server {
    listen 443 ssl http2;
    server_name example.com;
    
    # SSL configuration...
    
    location / {
        proxy_pass http://app_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Connection upgrade for WebSockets
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### Nginx Caching

```nginx
# Cache configuration
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=1g inactive=60m use_temp_path=off;

server {
    listen 443 ssl http2;
    server_name example.com;
    
    # Cache static assets
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2)$ {
        proxy_pass http://127.0.0.1:8000;
        proxy_cache my_cache;
        proxy_cache_valid 200 1y;
        proxy_cache_use_stale error timeout http_500 http_502 http_503 http_504;
        add_header X-Cache-Status $upstream_cache_status;
        expires 1y;
    }
    
    # Cache API responses
    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_cache my_cache;
        proxy_cache_valid 200 5m;
        proxy_cache_methods GET HEAD;
        proxy_cache_key "$scheme$request_method$host$request_uri";
        add_header X-Cache-Status $upstream_cache_status;
    }
}
```

### Enable Nginx Configuration

```bash
# Create symbolic link to enable site
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/

# Test configuration
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx

# Or restart
sudo systemctl restart nginx
```

---

## Process Management

### Systemd Service

```ini
# /etc/systemd/system/myapp.service

[Unit]
Description=My Python Application
After=network.target

[Service]
Type=notify
User=www-data
Group=www-data
WorkingDirectory=/var/www/myapp
Environment="PATH=/var/www/myapp/venv/bin"
Environment="FLASK_ENV=production"
EnvironmentFile=/var/www/myapp/.env
ExecStart=/var/www/myapp/venv/bin/gunicorn -c gunicorn_config.py app:app
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=mixed
TimeoutStopSec=5
PrivateTmp=true

# Restart policy
Restart=always
RestartSec=3

# Security
NoNewPrivileges=true
PrivateDevices=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/www/myapp/uploads

[Install]
WantedBy=multi-user.target
```

```bash
# Manage service
sudo systemctl daemon-reload
sudo systemctl enable myapp
sudo systemctl start myapp
sudo systemctl status myapp
sudo systemctl restart myapp
sudo systemctl stop myapp

# View logs
sudo journalctl -u myapp -f
```

### Supervisor

```bash
# Install Supervisor
sudo apt-get install supervisor
```

```ini
# /etc/supervisor/conf.d/myapp.conf

[program:myapp]
command=/var/www/myapp/venv/bin/gunicorn -c gunicorn_config.py app:app
directory=/var/www/myapp
user=www-data
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/var/log/myapp/app.log
stdout_logfile_maxbytes=10MB
stdout_logfile_backups=10
environment=FLASK_ENV="production"
```

```bash
# Manage with Supervisor
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start myapp
sudo supervisorctl stop myapp
sudo supervisorctl restart myapp
sudo supervisorctl status myapp
```

---

## Database Migrations

### Alembic (SQLAlchemy)

```bash
# Install Alembic
pip install alembic
```

```bash
# Initialize Alembic
alembic init alembic

# Create migration
alembic revision --autogenerate -m "Create users table"

# Apply migrations
alembic upgrade head

# Rollback
alembic downgrade -1

# Show current version
alembic current

# Show migration history
alembic history
```

```python
# alembic/env.py

from logging.config import fileConfig
from sqlalchemy import engine_from_config, pool
from alembic import context
import os
import sys

# Add project directory to path
sys.path.insert(0, os.path.dirname(os.path.dirname(__file__)))

# Import your models
from app import db
from app.models import User, Post

# Set target metadata
target_metadata = db.metadata

# Get database URL from environment
config = context.config
config.set_main_option(
    'sqlalchemy.url',
    os.environ.get('DATABASE_URL', 'sqlite:///app.db')
)

def run_migrations_online():
    """Run migrations in 'online' mode."""
    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix='sqlalchemy.',
        poolclass=pool.NullPool,
    )
    
    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata
        )
        
        with context.begin_transaction():
            context.run_migrations()

run_migrations_online()
```

### Django Migrations

```bash
# Create migrations
python manage.py makemigrations

# Apply migrations
python manage.py migrate

# Show migrations
python manage.py showmigrations

# Rollback
python manage.py migrate myapp 0003

# Create empty migration
python manage.py makemigrations --empty myapp
```

### Production Migration Strategy

```python
# deploy.sh

#!/bin/bash

set -e  # Exit on error

echo "Starting deployment..."

# Pull latest code
git pull origin main

# Activate virtual environment
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Run database migrations
echo "Running database migrations..."
python manage.py migrate --noinput

# Collect static files
echo "Collecting static files..."
python manage.py collectstatic --noinput

# Restart application
echo "Restarting application..."
sudo systemctl restart myapp

echo "Deployment completed successfully!"
```

---

## Static Files in Production

### WhiteNoise for Django

```bash
pip install whitenoise
```

```python
# settings.py

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',  # Add this
    # ... other middleware
]

# Static files
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')
STATIC_URL = '/static/'

# WhiteNoise configuration
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
```

### Serving Static Files with Nginx

```nginx
# Nginx configuration for static files

server {
    # ... other configuration
    
    # Static files with caching
    location /static/ {
        alias /var/www/myapp/staticfiles/;
        
        # Cache for 1 year
        expires 1y;
        add_header Cache-Control "public, immutable";
        
        # Gzip compression
        gzip on;
        gzip_vary on;
        gzip_types text/css application/javascript image/svg+xml;
        
        # Security
        add_header X-Content-Type-Options "nosniff";
    }
    
    # Media files
    location /media/ {
        alias /var/www/myapp/media/;
        expires 30d;
        add_header Cache-Control "public";
    }
}
```

### CDN Configuration

```python
# Using AWS S3 + CloudFront

# settings.py (Django)
AWS_ACCESS_KEY_ID = os.environ.get('AWS_ACCESS_KEY_ID')
AWS_SECRET_ACCESS_KEY = os.environ.get('AWS_SECRET_ACCESS_KEY')
AWS_STORAGE_BUCKET_NAME = os.environ.get('AWS_STORAGE_BUCKET_NAME')
AWS_S3_REGION_NAME = 'us-east-1'
AWS_S3_CUSTOM_DOMAIN = f'{AWS_STORAGE_BUCKET_NAME}.s3.amazonaws.com'

# CloudFront domain
CLOUDFRONT_DOMAIN = os.environ.get('CLOUDFRONT_DOMAIN')

# Static files
STATIC_URL = f'https://{CLOUDFRONT_DOMAIN}/static/'
STATICFILES_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'

# Media files
MEDIA_URL = f'https://{CLOUDFRONT_DOMAIN}/media/'
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
```

---

## Logging and Monitoring

### Structured Logging

```python
import logging
import json
from datetime import datetime

class JSONFormatter(logging.Formatter):
    """JSON log formatter."""
    
    def format(self, record):
        log_data = {
            'timestamp': datetime.utcnow().isoformat(),
            'level': record.levelname,
            'logger': record.name,
            'message': record.getMessage(),
            'module': record.module,
            'function': record.funcName,
            'line': record.lineno
        }
        
        # Add exception info if present
        if record.exc_info:
            log_data['exception'] = self.formatException(record.exc_info)
        
        # Add extra fields
        if hasattr(record, 'user_id'):
            log_data['user_id'] = record.user_id
        
        if hasattr(record, 'request_id'):
            log_data['request_id'] = record.request_id
        
        return json.dumps(log_data)

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    handlers=[
        logging.FileHandler('app.log'),
        logging.StreamHandler()
    ]
)

# Set JSON formatter
for handler in logging.root.handlers:
    handler.setFormatter(JSONFormatter())

logger = logging.getLogger(__name__)

# Usage
logger.info('User logged in', extra={'user_id': 123})
```

### Application Performance Monitoring

```python
# Using New Relic

# Install
# pip install newrelic

# newrelic.ini configuration file created with:
# newrelic-admin generate-config YOUR_LICENSE_KEY newrelic.ini

# Run application with New Relic
# NEW_RELIC_CONFIG_FILE=newrelic.ini newrelic-admin run-program gunicorn app:app
```

```python
# Using Datadog

# pip install ddtrace

from ddtrace import patch_all
patch_all()

# Run with Datadog
# DD_SERVICE="myapp" DD_ENV="production" ddtrace-run gunicorn app:app
```

### Error Tracking with Sentry

```python
import sentry_sdk
from sentry_sdk.integrations.flask import FlaskIntegration
from sentry_sdk.integrations.sqlalchemy import SqlalchemyIntegration

sentry_sdk.init(
    dsn=os.environ.get('SENTRY_DSN'),
    integrations=[
        FlaskIntegration(),
        SqlalchemyIntegration(),
    ],
    
    # Set traces_sample_rate to 1.0 to capture 100% of transactions
    traces_sample_rate=1.0,
    
    # Environment
    environment=os.environ.get('FLASK_ENV', 'production'),
    
    # Release tracking
    release='myapp@1.0.0',
    
    # Before send hook
    before_send=lambda event, hint: event if not event.get('logger') == 'health_check' else None
)

# Capture custom events
from sentry_sdk import capture_message, capture_exception

try:
    risky_operation()
except Exception as e:
    capture_exception(e)

# Add breadcrumbs
from sentry_sdk import add_breadcrumb

add_breadcrumb(
    category='auth',
    message='User logged in',
    level='info',
    data={'user_id': user.id}
)
```

---

## Docker Deployment

### Dockerfile

```dockerfile
# Dockerfile

# Use official Python runtime
FROM python:3.11-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# Set work directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --upgrade pip && \
    pip install -r requirements.txt

# Copy project
COPY . .

# Create non-root user
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app

USER appuser

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Run application
CMD ["gunicorn", "-c", "gunicorn_config.py", "app:app"]
```

### Docker Compose

```yaml
# docker-compose.yml

version: '3.8'

services:
  web:
    build: .
    command: gunicorn -c gunicorn_config.py app:app
    volumes:
      - ./:/app
      - static_volume:/app/staticfiles
      - media_volume:/app/media
    expose:
      - 8000
    env_file:
      - .env
    depends_on:
      - db
      - redis
    restart: unless-stopped
  
  db:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=myapp
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    restart: unless-stopped
  
  redis:
    image: redis:7-alpine
    restart: unless-stopped
  
  nginx:
    image: nginx:alpine
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - static_volume:/var/www/static
      - media_volume:/var/www/media
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - web
    restart: unless-stopped

volumes:
  postgres_data:
  static_volume:
  media_volume:
```

```bash
# Build and run
docker-compose up -d --build

# View logs
docker-compose logs -f web

# Execute commands
docker-compose exec web python manage.py migrate
docker-compose exec web python manage.py createsuperuser

# Stop services
docker-compose down

# Stop and remove volumes
docker-compose down -v
```

### Multi-stage Build

```dockerfile
# Multi-stage Dockerfile for production

# Build stage
FROM python:3.11-slim as builder

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip wheel --no-cache-dir --no-deps --wheel-dir /app/wheels -r requirements.txt

# Runtime stage
FROM python:3.11-slim

WORKDIR /app

# Install runtime dependencies
RUN apt-get update && apt-get install -y \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Copy wheels from builder
COPY --from=builder /app/wheels /wheels
RUN pip install --no-cache /wheels/*

# Copy application
COPY . .

# Create non-root user
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app

USER appuser

EXPOSE 8000

CMD ["gunicorn", "-c", "gunicorn_config.py", "app:app"]
```

---

## Cloud Platform Deployment

### Heroku

```bash
# Install Heroku CLI
curl https://cli-assets.heroku.com/install.sh | sh

# Login
heroku login

# Create app
heroku create myapp

# Add PostgreSQL
heroku addons:create heroku-postgresql:mini

# Add Redis
heroku addons:create heroku-redis:mini

# Set environment variables
heroku config:set SECRET_KEY=your-secret-key
heroku config:set FLASK_ENV=production

# Deploy
git push heroku main

# Run migrations
heroku run python manage.py migrate

# View logs
heroku logs --tail

# Scale dynos
heroku ps:scale web=2

# Open app
heroku open
```

```python
# Procfile
web: gunicorn app:app

# Or for Django
web: gunicorn myproject.wsgi

# For worker processes
worker: celery -A app.celery worker --loglevel=info
```

```python
# runtime.txt
python-3.11.0
```

### AWS Elastic Beanstalk

```bash
# Install EB CLI
pip install awsebcli

# Initialize
eb init -p python-3.11 myapp

# Create environment
eb create myapp-env

# Deploy
eb deploy

# Set environment variables
eb setenv SECRET_KEY=your-secret-key DEBUG=False

# View logs
eb logs

# SSH into instance
eb ssh

# Open application
eb open

# Terminate environment
eb terminate myapp-env
```

```.ebextensions
# .ebextensions/01_packages.config

packages:
  yum:
    postgresql-devel: []
    
commands:
  01_migrate:
    command: "source /var/app/venv/*/bin/activate && python manage.py migrate --noinput"
    leader_only: true
```

### DigitalOcean App Platform

```yaml
# .do/app.yaml

name: myapp
region: nyc

services:
- name: web
  github:
    repo: username/myapp
    branch: main
    deploy_on_push: true
  
  build_command: pip install -r requirements.txt
  run_command: gunicorn -c gunicorn_config.py app:app
  
  envs:
  - key: SECRET_KEY
    value: ${SECRET_KEY}
  - key: DATABASE_URL
    value: ${db.DATABASE_URL}
  - key: REDIS_URL
    value: ${redis.REDIS_URL}
  
  health_check:
    http_path: /health
  
  instance_count: 2
  instance_size_slug: basic-xxs
  
  routes:
  - path: /

databases:
- name: db
  engine: PG
  version: "15"
  
- name: redis
  engine: REDIS
  version: "7"
```

### Google Cloud Run

```bash
# Build container
gcloud builds submit --tag gcr.io/PROJECT_ID/myapp

# Deploy
gcloud run deploy myapp \
  --image gcr.io/PROJECT_ID/myapp \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars SECRET_KEY=your-secret-key

# Update service
gcloud run services update myapp \
  --set-env-vars DATABASE_URL=your-database-url

# View logs
gcloud run services logs read myapp
```

---

## CI/CD Pipelines

### GitHub Actions

```yaml
# .github/workflows/deploy.yml

name: Deploy to Production

on:
  push:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install pytest pytest-cov
    
    - name: Run tests
      env:
        DATABASE_URL: postgresql://postgres:postgres@localhost/test
      run: |
        pytest --cov=app tests/
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
  
  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Deploy to Heroku
      uses: akhileshns/heroku-deploy@v3.12.12
      with:
        heroku_api_key: ${{secrets.HEROKU_API_KEY}}
        heroku_app_name: myapp
        heroku_email: your-email@example.com
    
    - name: Run migrations
      run: |
        heroku run python manage.py migrate --app myapp
```

### GitLab CI/CD

```yaml
# .gitlab-ci.yml

stages:
  - test
  - build
  - deploy

variables:
  POSTGRES_DB: test
  POSTGRES_USER: test
  POSTGRES_PASSWORD: test

test:
  stage: test
  image: python:3.11
  services:
    - postgres:15
  variables:
    DATABASE_URL: postgresql://test:test@postgres/test
  before_script:
    - pip install -r requirements.txt
    - pip install pytest pytest-cov
  script:
    - pytest --cov=app tests/
  coverage: '/TOTAL.*\s+(\d+%)$/'

build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  only:
    - main

deploy:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
  script:
    - ssh user@server 'cd /var/www/myapp && git pull && ./deploy.sh'
  only:
    - main
  when: manual
```

---

## Security Hardening

### Security Checklist

```python
"""
✅ Production Security Checklist:

1. Environment
   ✅ DEBUG = False
   ✅ SECRET_KEY is random and secret
   ✅ ALLOWED_HOSTS configured
   ✅ No credentials in code

2. HTTPS
   ✅ SSL certificate installed
   ✅ Force HTTPS redirect
   ✅ HSTS enabled
   ✅ Secure cookies

3. Headers
   ✅ X-Frame-Options
   ✅ X-Content-Type-Options
   ✅ X-XSS-Protection
   ✅ Content-Security-Policy
   ✅ Referrer-Policy

4. Database
   ✅ Strong passwords
   ✅ Limited user privileges
   ✅ SQL injection prevention
   ✅ Encrypted connections

5. Files
   ✅ Proper file permissions
   ✅ Validated file uploads
   ✅ No directory listing
   ✅ Secure file storage

6. Authentication
   ✅ Strong password policy
   ✅ Password hashing (bcrypt/Argon2)
   ✅ Session management
   ✅ CSRF protection

7. Infrastructure
   ✅ Firewall configured
   ✅ SSH key authentication
   ✅ Regular updates
   ✅ Intrusion detection

8. Monitoring
   ✅ Error tracking
   ✅ Access logs
   ✅ Security alerts
   ✅ Regular audits
"""
```

### Security Middleware

```python
from flask import Flask
from flask_talisman import Talisman

app = Flask(__name__)

# Force HTTPS and set security headers
talisman = Talisman(
    app,
    force_https=True,
    strict_transport_security=True,
    strict_transport_security_max_age=31536000,
    content_security_policy={
        'default-src': "'self'",
        'script-src': ["'self'", "'unsafe-inline'", "cdn.example.com"],
        'style-src': ["'self'", "'unsafe-inline'"],
        'img-src': ["'self'", "data:", "https:"],
        'font-src': ["'self'"],
        'connect-src': ["'self'"],
        'frame-ancestors': ["'none'"],
    },
    content_security_policy_nonce_in=['script-src'],
    feature_policy={
        'geolocation': "'none'",
        'microphone': "'none'",
        'camera': "'none'"
    }
)
```

---

## Performance Optimization

### Caching

```python
from flask import Flask
from flask_caching import Cache

app = Flask(__name__)

# Configure cache
app.config['CACHE_TYPE'] = 'RedisCache'
app.config['CACHE_REDIS_URL'] = os.environ.get('REDIS_URL')
app.config['CACHE_DEFAULT_TIMEOUT'] = 300

cache = Cache(app)

@app.route('/expensive-operation')
@cache.cached(timeout=60)
def expensive_operation():
    """Cached endpoint."""
    # Expensive computation
    result = compute_something()
    return jsonify(result)

@app.route('/user/<int:user_id>')
@cache.cached(timeout=300, key_prefix='user_profile')
def user_profile(user_id):
    """Cached with custom key."""
    user = User.query.get_or_404(user_id)
    return jsonify(user.to_dict())

# Clear cache
@app.route('/clear-cache')
def clear_cache():
    cache.clear()
    return 'Cache cleared'
```

### Database Connection Pooling

```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    DATABASE_URL,
    poolclass=QueuePool,
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True,
    pool_recycle=3600
)
```

### Query Optimization

```python
# Use select_related and prefetch_related (Django)
users = User.objects.select_related('profile').all()

# Use joinedload (SQLAlchemy)
from sqlalchemy.orm import joinedload

users = session.query(User).options(
    joinedload(User.profile)
).all()

# Pagination
from flask_sqlalchemy import Pagination

page = request.args.get('page', 1, type=int)
per_page = 20

pagination = User.query.paginate(
    page=page,
    per_page=per_page,
    error_out=False
)
```

---

## Best Practices

### Deployment Best Practices

```python
"""
✅ Deployment Best Practices:

1. Version Control
   ✅ Use Git tags for releases
   ✅ Semantic versioning
   ✅ Keep main branch deployable
   ✅ Use feature branches

2. Testing
   ✅ Run tests before deployment
   ✅ Integration tests
   ✅ Load testing
   ✅ Security testing

3. Database
   ✅ Backup before deployment
   ✅ Test migrations locally
   ✅ Reversible migrations
   ✅ Monitor query performance

4. Deployment
   ✅ Use CI/CD pipeline
   ✅ Blue-green deployment
   ✅ Canary releases
   ✅ Automated rollback

5. Monitoring
   ✅ Health checks
   ✅ Error tracking
   ✅ Performance monitoring
   ✅ Alert on critical issues

6. Documentation
   ✅ Deployment procedures
   ✅ Rollback procedures
   ✅ Architecture diagrams
   ✅ Runbooks

7. Communication
   ✅ Announce maintenance windows
   ✅ Status page
   ✅ Incident response plan
   ✅ Post-mortems
"""
```

### Zero-Downtime Deployment

```bash
#!/bin/bash
# Blue-green deployment script

set -e

BLUE_PORT=8001
GREEN_PORT=8002
CURRENT_PORT=$(cat /var/www/current_port)

if [ "$CURRENT_PORT" = "$BLUE_PORT" ]; then
    NEW_PORT=$GREEN_PORT
    NEW_COLOR="green"
else
    NEW_PORT=$BLUE_PORT
    NEW_COLOR="blue"
fi

echo "Deploying to $NEW_COLOR environment (port $NEW_PORT)..."

# Pull latest code
git pull origin main

# Install dependencies
source venv/bin/activate
pip install -r requirements.txt

# Run migrations
python manage.py migrate

# Collect static files
python manage.py collectstatic --noinput

# Start new instance
gunicorn -c gunicorn_config.py --bind 0.0.0.0:$NEW_PORT app:app --daemon

# Wait for health check
sleep 5
curl -f http://localhost:$NEW_PORT/health || exit 1

# Update Nginx to point to new port
sed -i "s/localhost:$CURRENT_PORT/localhost:$NEW_PORT/" /etc/nginx/sites-available/myapp
nginx -t && nginx -s reload

# Save new port
echo $NEW_PORT > /var/www/current_port

# Stop old instance
pkill -f "gunicorn.*--bind 0.0.0.0:$CURRENT_PORT"

echo "Deployment completed successfully!"
```

---

## Summary

This comprehensive guide covered:

1. **Introduction**: Development vs production differences
2. **Environment Variables**: Using python-dotenv and configuration
3. **Configuration Management**: Multiple environments and settings
4. **Production-Ready Code**: Error handling and health checks
5. **WSGI/ASGI Servers**: Gunicorn, Uvicorn, uWSGI
6. **Nginx**: Reverse proxy, load balancing, caching
7. **Process Management**: Systemd and Supervisor
8. **Database Migrations**: Alembic and Django migrations
9. **Static Files**: WhiteNoise, Nginx, CDN
10. **Logging/Monitoring**: Structured logging, APM, Sentry
11. **Docker**: Dockerfile, Docker Compose, multi-stage builds
12. **Cloud Platforms**: Heroku, AWS, DigitalOcean, Google Cloud
13. **CI/CD**: GitHub Actions, GitLab CI/CD
14. **Security**: Hardening and best practices
15. **Performance**: Caching, connection pooling, optimization

Key Takeaways:
- Never debug in production
- Use environment variables for configuration
- Implement proper logging and monitoring
- Use production-grade servers (Gunicorn, Uvicorn)
- Configure Nginx as reverse proxy
- Automate deployment with CI/CD
- Implement zero-downtime deployments
- Monitor application health and performance
- Secure your application at every layer
- Have rollback and disaster recovery plans

Deployment is a continuous process - monitor, optimize, and improve!