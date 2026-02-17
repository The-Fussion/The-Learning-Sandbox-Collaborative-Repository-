# Deployment in FastAPI

## 17.1 Production Servers

### Uvicorn Production Mode

Uvicorn is the recommended ASGI server for FastAPI in production.

```bash
# Basic production run
uvicorn main:app --host 0.0.0.0 --port 8000

# Production configuration
uvicorn main:app \
  --host 0.0.0.0 \
  --port 8000 \
  --workers 4 \
  --loop uvloop \
  --http httptools \
  --log-level info \
  --access-log \
  --no-use-colors \
  --proxy-headers \
  --forwarded-allow-ips='*'
```

```python
# main.py - Programmatic configuration
import uvicorn

if __name__ == "__main__":
    uvicorn.run(
        "app.main:app",
        host="0.0.0.0",
        port=8000,
        workers=4,
        loop="uvloop",
        http="httptools",
        log_level="info",
        access_log=True,
        proxy_headers=True,
        forwarded_allow_ips="*",
        timeout_keep_alive=30,
        timeout_graceful_shutdown=30,
    )
```

```ini
# uvicorn.ini config file
[uvicorn]
host = 0.0.0.0
port = 8000
workers = 4
log_level = info
access_log = true
proxy_headers = true
```

### Gunicorn with Uvicorn Workers

Gunicorn acts as a process manager, spawning Uvicorn worker processes.

```bash
# Install
pip install gunicorn uvicorn[standard]

# Basic production command
gunicorn main:app \
  --workers 4 \
  --worker-class uvicorn.workers.UvicornWorker \
  --bind 0.0.0.0:8000

# Full production configuration
gunicorn main:app \
  --workers 4 \
  --worker-class uvicorn.workers.UvicornWorker \
  --bind 0.0.0.0:8000 \
  --timeout 120 \
  --keepalive 5 \
  --max-requests 1000 \
  --max-requests-jitter 100 \
  --preload \
  --access-logfile - \
  --error-logfile - \
  --log-level info
```

```python
# gunicorn.conf.py
import os
import multiprocessing

# Server socket
bind = f"0.0.0.0:{os.getenv('PORT', '8000')}"

# Worker processes
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = "uvicorn.workers.UvicornWorker"
worker_connections = 1000

# Timeouts
timeout = 120
keepalive = 5
graceful_timeout = 30

# Requests
max_requests = 1000
max_requests_jitter = 100  # Prevents all workers restarting at once

# Logging
accesslog = "-"    # stdout
errorlog = "-"     # stdout
loglevel = "info"
access_log_format = '%(h)s %(l)s %(u)s %(t)s "%(r)s" %(s)s %(b)s "%(f)s" "%(a)s" %(L)s'

# Process naming
proc_name = "fastapi_app"

# Preload app for faster worker startup
preload_app = True

# Security
limit_request_line = 4094
limit_request_fields = 100
limit_request_field_size = 8190
```

### Number of Workers Calculation

Calculate optimal worker count for your server.

```python
import multiprocessing

# Formula: (2 x CPU cores) + 1
def recommended_workers() -> int:
    cpu_count = multiprocessing.cpu_count()
    return (2 * cpu_count) + 1

# For I/O-bound applications (most FastAPI apps)
# Use more workers: 4 x CPU cores
def io_bound_workers() -> int:
    cpu_count = multiprocessing.cpu_count()
    return 4 * cpu_count

# For CPU-bound applications
# Use fewer workers: CPU cores
def cpu_bound_workers() -> int:
    return multiprocessing.cpu_count()

"""
Example sizing:
- 1 CPU core:  3 workers  (2*1+1)
- 2 CPU cores: 5 workers  (2*2+1)
- 4 CPU cores: 9 workers  (2*4+1)
- 8 CPU cores: 17 workers (2*8+1)

Memory consideration:
- Each worker uses ~50-150MB RAM
- 4 workers on 1GB RAM = safe
- 4 workers on 512MB RAM = tight, reduce to 2
"""

print(f"Recommended workers: {recommended_workers()}")
```

### Worker Timeout Configuration

Configure timeouts to handle slow requests.

```python
# gunicorn.conf.py
timeout = 120          # Worker killed if no response in 120s
keepalive = 5          # Keep connections alive for 5s after request
graceful_timeout = 30  # Allow 30s for in-flight requests to complete

# For long-running operations, increase timeout
timeout = 300  # 5 minutes for heavy processing

# Per-endpoint timeout using asyncio
import asyncio
from fastapi import FastAPI, HTTPException

app = FastAPI()

@app.get("/slow-operation")
async def slow_operation():
    try:
        result = await asyncio.wait_for(
            some_slow_operation(),
            timeout=30.0  # 30 second timeout
        )
        return {"result": result}
    except asyncio.TimeoutError:
        raise HTTPException(
            status_code=504,
            detail="Operation timed out"
        )
```

### Graceful Shutdown

Handle shutdown signals for clean termination.

```python
from fastapi import FastAPI
from contextlib import asynccontextmanager
import asyncio
import signal

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    print("Starting up...")
    await startup_tasks()

    yield

    # Shutdown
    print("Shutting down gracefully...")

    # Complete in-flight requests (Gunicorn handles this)
    # Close database connections
    await app.state.db_engine.dispose()

    # Close Redis connections
    await app.state.redis.close()

    # Cancel background tasks
    for task in app.state.tasks:
        task.cancel()
        try:
            await task
        except asyncio.CancelledError:
            pass

    print("Shutdown complete")

app = FastAPI(lifespan=lifespan)

# Handle SIGTERM for Docker/Kubernetes
def handle_sigterm(signum, frame):
    print("SIGTERM received, initiating shutdown...")
    raise SystemExit(0)

signal.signal(signal.SIGTERM, handle_sigterm)
```

## 17.2 Docker

### Creating Dockerfile for FastAPI

Build a production-ready Docker image.

```dockerfile
# Dockerfile
FROM python:3.11-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create non-root user
RUN adduser --disabled-password --gecos "" appuser && \
    chown -R appuser:appuser /app
USER appuser

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Run application
CMD ["gunicorn", "main:app", \
     "--workers", "4", \
     "--worker-class", "uvicorn.workers.UvicornWorker", \
     "--bind", "0.0.0.0:8000", \
     "--timeout", "120", \
     "--access-logfile", "-", \
     "--error-logfile", "-"]
```

### Multi-Stage Builds

Use multi-stage builds for smaller images.

```dockerfile
# Dockerfile.multistage

# Stage 1: Builder
FROM python:3.11-slim as builder

WORKDIR /build

# Install build dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Stage 2: Production
FROM python:3.11-slim as production

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PATH="/home/appuser/.local/bin:$PATH"

WORKDIR /app

# Install only runtime dependencies
RUN apt-get update && apt-get install -y \
    libpq5 \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN adduser --disabled-password --gecos "" appuser

# Copy installed packages from builder
COPY --from=builder --chown=appuser:appuser \
    /root/.local /home/appuser/.local

# Copy application code
COPY --chown=appuser:appuser . .

USER appuser

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

CMD ["gunicorn", "main:app", \
     "--workers", "4", \
     "--worker-class", "uvicorn.workers.UvicornWorker", \
     "--bind", "0.0.0.0:8000"]
```

### Docker Compose

Orchestrate multiple services with Docker Compose.

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: fastapi_app
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql+asyncpg://user:password@db:5432/mydb
      - REDIS_URL=redis://redis:6379
      - SECRET_KEY=${SECRET_KEY}
      - ENVIRONMENT=production
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - app-network
    volumes:
      - ./logs:/app/logs

  db:
    image: postgres:15-alpine
    container_name: postgres_db
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    container_name: redis_cache
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  nginx:
    image: nginx:alpine
    container_name: nginx_proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/certs:/etc/nginx/certs
    depends_on:
      - app
    networks:
      - app-network

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - app-network

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  postgres_data:
  redis_data:
  prometheus_data:
  grafana_data:
```

### Container Optimization

Optimize Docker images for production.

```dockerfile
# .dockerignore
__pycache__/
*.pyc
*.pyo
*.pyd
.Python
*.egg-info/
.git/
.gitignore
.env
.env.*
tests/
docs/
*.md
.vscode/
.idea/
Makefile
docker-compose*.yml
```

```bash
# Build optimized image
docker build \
  --no-cache \
  --compress \
  --tag myapp:latest \
  --tag myapp:1.0.0 \
  .

# Check image size
docker images myapp

# Inspect layers
docker history myapp:latest

# Scan for vulnerabilities
docker scout cves myapp:latest
```

### Health Checks

Implement comprehensive health checks.

```python
from fastapi import FastAPI
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, text
import redis.asyncio as aioredis
import time

app = FastAPI()

@app.get("/health")
async def health_check():
    """Basic health check - returns 200 if app is running"""
    return {
        "status": "healthy",
        "timestamp": time.time()
    }

@app.get("/health/ready")
async def readiness_check():
    """Readiness check - verifies all dependencies are available"""
    checks = {}
    healthy = True

    # Database check
    try:
        async with AsyncSessionLocal() as session:
            await session.execute(text("SELECT 1"))
        checks["database"] = "healthy"
    except Exception as e:
        checks["database"] = f"unhealthy: {str(e)}"
        healthy = False

    # Redis check
    try:
        await redis_client.ping()
        checks["redis"] = "healthy"
    except Exception as e:
        checks["redis"] = f"unhealthy: {str(e)}"
        healthy = False

    status_code = 200 if healthy else 503
    return JSONResponse(
        status_code=status_code,
        content={"status": "ready" if healthy else "not ready", "checks": checks}
    )

@app.get("/health/live")
async def liveness_check():
    """Liveness check - returns 200 if app is alive"""
    return {"status": "alive", "timestamp": time.time()}
```

### Environment Variables in Docker

Manage configuration through environment variables.

```python
# config.py
from pydantic_settings import BaseSettings
from typing import List

class Settings(BaseSettings):
    # Application
    APP_NAME: str = "FastAPI App"
    ENVIRONMENT: str = "development"
    DEBUG: bool = False
    SECRET_KEY: str
    ALLOWED_HOSTS: List[str] = ["*"]

    # Database
    DATABASE_URL: str
    DB_POOL_SIZE: int = 20
    DB_MAX_OVERFLOW: int = 10

    # Redis
    REDIS_URL: str = "redis://localhost:6379"
    CACHE_TTL: int = 300

    # AWS
    AWS_ACCESS_KEY_ID: str = ""
    AWS_SECRET_ACCESS_KEY: str = ""
    AWS_REGION: str = "us-east-1"
    S3_BUCKET: str = ""

    # Monitoring
    SENTRY_DSN: str = ""

    class Config:
        env_file = ".env"
        case_sensitive = True

settings = Settings()
```

```bash
# .env.production
APP_NAME="My FastAPI App"
ENVIRONMENT=production
DEBUG=false
SECRET_KEY=your-super-secret-key-here
DATABASE_URL=postgresql+asyncpg://user:pass@db:5432/mydb
REDIS_URL=redis://:password@redis:6379
SENTRY_DSN=https://xxx@sentry.io/123
```

## 17.3 Cloud Deployment

### AWS (EC2, ECS, Lambda)

Deploy FastAPI on AWS infrastructure.

```bash
# EC2 Deployment
# 1. Launch EC2 instance (Ubuntu 22.04)
# 2. SSH into instance
ssh -i key.pem ubuntu@ec2-xx-xx-xx-xx.compute.amazonaws.com

# 3. Install dependencies
sudo apt-get update
sudo apt-get install -y python3.11 python3-pip nginx

# 4. Clone and setup app
git clone https://github.com/myapp/api.git
cd api
pip install -r requirements.txt

# 5. Setup systemd service
sudo nano /etc/systemd/system/fastapi.service
```

```ini
# /etc/systemd/system/fastapi.service
[Unit]
Description=FastAPI Application
After=network.target

[Service]
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu/api
Environment="PATH=/home/ubuntu/api/venv/bin"
ExecStart=/home/ubuntu/api/venv/bin/gunicorn main:app \
    --workers 4 \
    --worker-class uvicorn.workers.UvicornWorker \
    --bind 0.0.0.0:8000
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```yaml
# ECS Task Definition (ecs-task.json)
{
  "family": "fastapi-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::ACCOUNT_ID:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "fastapi-app",
      "image": "ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/fastapi-app:latest",
      "portMappings": [
        {
          "containerPort": 8000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {"name": "ENVIRONMENT", "value": "production"}
      ],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:REGION:ACCOUNT_ID:secret:db-url"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/fastapi-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"],
        "interval": 30,
        "timeout": 10,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
```

### Google Cloud (Cloud Run, App Engine)

Deploy to Google Cloud Platform.

```yaml
# Cloud Run - cloudbuild.yaml
steps:
  # Build image
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'gcr.io/$PROJECT_ID/fastapi-app:$COMMIT_SHA', '.']

  # Push image
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'gcr.io/$PROJECT_ID/fastapi-app:$COMMIT_SHA']

  # Deploy to Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args:
      - 'run'
      - 'deploy'
      - 'fastapi-app'
      - '--image'
      - 'gcr.io/$PROJECT_ID/fastapi-app:$COMMIT_SHA'
      - '--region'
      - 'us-central1'
      - '--platform'
      - 'managed'
      - '--allow-unauthenticated'
      - '--memory'
      - '512Mi'
      - '--cpu'
      - '1'
      - '--min-instances'
      - '1'
      - '--max-instances'
      - '10'
      - '--concurrency'
      - '100'

images:
  - 'gcr.io/$PROJECT_ID/fastapi-app:$COMMIT_SHA'
```

```bash
# Deploy manually to Cloud Run
gcloud run deploy fastapi-app \
  --image gcr.io/PROJECT_ID/fastapi-app:latest \
  --region us-central1 \
  --platform managed \
  --allow-unauthenticated \
  --memory 512Mi \
  --cpu 1 \
  --set-env-vars "ENVIRONMENT=production" \
  --set-secrets "DATABASE_URL=db-url:latest"
```

### Azure (App Service, Container Instances)

Deploy to Microsoft Azure.

```yaml
# Azure Container Instances - azure-deploy.yml
resources:
  - type: Microsoft.ContainerInstance/containerGroups
    apiVersion: '2021-03-01'
    name: fastapi-container
    location: eastus
    properties:
      containers:
        - name: fastapi-app
          properties:
            image: myregistry.azurecr.io/fastapi-app:latest
            ports:
              - port: 8000
            environmentVariables:
              - name: ENVIRONMENT
                value: production
              - name: DATABASE_URL
                secureValue: "postgresql+asyncpg://..."
            resources:
              requests:
                cpu: 1
                memoryInGb: 1.5
      osType: Linux
      ipAddress:
        type: Public
        ports:
          - port: 8000
            protocol: TCP
```

### Heroku

Deploy to Heroku using containers.

```bash
# Procfile
web: gunicorn main:app --workers 4 --worker-class uvicorn.workers.UvicornWorker --bind 0.0.0.0:$PORT --timeout 120
```

```bash
# Deploy to Heroku
heroku create my-fastapi-app
heroku config:set SECRET_KEY=your-secret-key
heroku config:set DATABASE_URL=postgresql://...

# Container deployment
heroku container:push web
heroku container:release web

# Or Git deployment
git push heroku main

heroku ps:scale web=1
heroku logs --tail
```

### Railway

Railway is a modern cloud platform for easy deployments.

```bash
# railway.toml
[build]
builder = "dockerfile"
dockerfilePath = "Dockerfile"

[deploy]
startCommand = "gunicorn main:app --workers 4 --worker-class uvicorn.workers.UvicornWorker --bind 0.0.0.0:$PORT"
healthcheckPath = "/health"
healthcheckTimeout = 30
restartPolicyType = "ON_FAILURE"
restartPolicyMaxRetries = 3
```

```bash
# Deploy to Railway
railway login
railway init
railway up
railway vars set SECRET_KEY=your-secret-key
```

## 17.4 Serverless

### AWS Lambda with Mangum

Deploy FastAPI as a serverless function on AWS Lambda.

```bash
pip install mangum
```

```python
# main.py
from fastapi import FastAPI
from mangum import Mangum

app = FastAPI(root_path="/prod")

@app.get("/")
async def root():
    return {"message": "Hello from Lambda!"}

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    return {"user_id": user_id}

# Lambda handler
handler = Mangum(
    app,
    lifespan="off",  # Disable lifespan for Lambda
    api_gateway_base_path="/prod"
)
```

```yaml
# serverless.yml (Serverless Framework)
service: fastapi-serverless

provider:
  name: aws
  runtime: python3.11
  region: us-east-1
  memorySize: 512
  timeout: 30
  environment:
    DATABASE_URL: ${ssm:/myapp/database_url}
    SECRET_KEY: ${ssm:/myapp/secret_key}

functions:
  api:
    handler: main.handler
    events:
      - http:
          path: /{proxy+}
          method: ANY
      - http:
          path: /
          method: ANY

plugins:
  - serverless-python-requirements

custom:
  pythonRequirements:
    dockerizePip: true
    zip: true
    slim: true
```

```yaml
# AWS SAM template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Timeout: 30
    MemorySize: 512

Resources:
  FastAPIFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: .
      Handler: main.handler
      Runtime: python3.11
      Events:
        Api:
          Type: HttpApi
          Properties:
            Path: /{proxy+}
            Method: ANY
```

### Google Cloud Functions

Deploy as Google Cloud Functions.

```python
# main.py for Cloud Functions
from fastapi import FastAPI
from functions_framework import create_app

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello from Cloud Functions!"}

# Cloud Functions entry point
import functions_framework
from starlette.middleware.base import BaseHTTPMiddleware

@functions_framework.http
def fastapi_handler(request):
    """HTTP Cloud Function handler."""
    from asgiref.wsgi import WsgiToAsgi
    
    # Convert Flask request to ASGI
    return app
```

### Azure Functions

Deploy FastAPI to Azure Functions.

```python
# function_app.py
import azure.functions as func
from fastapi import FastAPI
from azure.functions import AsgiMiddleware

app = FastAPI()

@app.get("/api/users")
async def get_users():
    return [{"id": 1, "name": "Alice"}]

# Azure Functions handler
main = func.AsgiFunctionApp(
    app=app,
    http_auth_level=func.AuthLevel.ANONYMOUS
)
```

### Vercel

Deploy FastAPI to Vercel.

```json
// vercel.json
{
  "builds": [
    {
      "src": "main.py",
      "use": "@vercel/python"
    }
  ],
  "routes": [
    {
      "src": "/(.*)",
      "dest": "main.py"
    }
  ]
}
```

```python
# main.py for Vercel
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello from Vercel!"}

# Vercel needs the app to be named 'app'
```

## 17.5 Kubernetes

### Kubernetes Basics

Essential Kubernetes concepts for FastAPI deployment.

```yaml
# Basic pod definition
apiVersion: v1
kind: Pod
metadata:
  name: fastapi-pod
  labels:
    app: fastapi
spec:
  containers:
    - name: fastapi
      image: myregistry/fastapi-app:latest
      ports:
        - containerPort: 8000
      env:
        - name: ENVIRONMENT
          value: production
      resources:
        requests:
          memory: "256Mi"
          cpu: "250m"
        limits:
          memory: "512Mi"
          cpu: "500m"
```

### Deployment Manifests

Full Kubernetes deployment configuration.

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-deployment
  namespace: production
  labels:
    app: fastapi
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fastapi
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: fastapi
        version: "1.0.0"
    spec:
      containers:
        - name: fastapi
          image: myregistry/fastapi-app:1.0.0
          ports:
            - containerPort: 8000
          env:
            - name: ENVIRONMENT
              value: production
            - name: SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: fastapi-secrets
                  key: secret-key
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: fastapi-secrets
                  key: database-url
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8000
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /health/live
              port: 8000
            failureThreshold: 30
            periodSeconds: 10
      terminationGracePeriodSeconds: 60
```

### Services and Ingress

Expose FastAPI with Kubernetes services.

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: fastapi-service
  namespace: production
spec:
  selector:
    app: fastapi
  ports:
    - name: http
      port: 80
      targetPort: 8000
  type: ClusterIP
```

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fastapi-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: fastapi-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: fastapi-service
                port:
                  number: 80
```

### ConfigMaps and Secrets

Manage configuration and sensitive data.

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fastapi-config
  namespace: production
data:
  ENVIRONMENT: "production"
  LOG_LEVEL: "info"
  DB_POOL_SIZE: "20"
  CACHE_TTL: "300"
  ALLOWED_HOSTS: "api.example.com"
```

```yaml
# k8s/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: fastapi-secrets
  namespace: production
type: Opaque
data:
  # base64 encoded values
  # echo -n "value" | base64
  secret-key: eW91ci1zdXBlci1zZWNyZXQta2V5
  database-url: cG9zdGdyZXNxbCsuLi4=
  redis-url: cmVkaXM6Ly86cGFzc3dvcmRAcmVkaXM6NjM3OQ==
```

```bash
# Apply configurations
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secrets.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml

# Check status
kubectl get pods -n production
kubectl describe pod fastapi-xxx -n production
kubectl logs fastapi-xxx -n production
```

### Horizontal Pod Autoscaling

Auto-scale based on metrics.

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: fastapi-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: fastapi-deployment
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
```

## 17.6 CI/CD

### GitHub Actions

Automate testing and deployment with GitHub Actions.

```yaml
# .github/workflows/deploy.yml
name: Deploy FastAPI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: Lint with ruff
        run: ruff check .

      - name: Type check with mypy
        run: mypy .

      - name: Run tests
        env:
          DATABASE_URL: postgresql+asyncpg://test:test@localhost/testdb
          REDIS_URL: redis://localhost:6379
          SECRET_KEY: test-secret-key
        run: |
          pytest tests/ -v --cov=app --cov-report=xml

      - name: Upload coverage
        uses: codecov/codecov-action@v3

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix={{branch}}-
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production

    steps:
      - name: Deploy to production
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /app
            docker compose pull
            docker compose up -d --no-deps app
            docker compose ps
```

### GitLab CI

GitLab CI/CD pipeline configuration.

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  DOCKER_IMAGE_LATEST: $CI_REGISTRY_IMAGE:latest

test:
  stage: test
  image: python:3.11-slim
  services:
    - postgres:15
    - redis:7
  variables:
    POSTGRES_USER: test
    POSTGRES_PASSWORD: test
    POSTGRES_DB: testdb
    DATABASE_URL: postgresql+asyncpg://test:test@postgres/testdb
    REDIS_URL: redis://redis:6379
    SECRET_KEY: test-secret-key
  before_script:
    - pip install -r requirements.txt
    - pip install -r requirements-dev.txt
  script:
    - ruff check .
    - mypy .
    - pytest tests/ -v --cov=app --cov-report=xml
  coverage: '/TOTAL.*\s+(\d+%)$/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
  only:
    - merge_requests
    - main

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $DOCKER_IMAGE -t $DOCKER_IMAGE_LATEST .
    - docker push $DOCKER_IMAGE
    - docker push $DOCKER_IMAGE_LATEST
  only:
    - main

deploy-production:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | ssh-add -
  script:
    - ssh -o StrictHostKeyChecking=no $DEPLOY_USER@$DEPLOY_HOST "
        cd /app &&
        docker compose pull &&
        docker compose up -d --no-deps app &&
        docker compose ps
      "
  only:
    - main
  environment:
    name: production
    url: https://api.example.com
  when: manual
```

### Blue-Green Deployment

Zero-downtime deployments with blue-green strategy.

```yaml
# Blue-green deployment with Kubernetes
# k8s/blue-green.yaml

# Blue deployment (current production)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-blue
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fastapi
      slot: blue
  template:
    metadata:
      labels:
        app: fastapi
        slot: blue
    spec:
      containers:
        - name: fastapi
          image: myregistry/fastapi-app:1.0.0

---
# Green deployment (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-green
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fastapi
      slot: green
  template:
    metadata:
      labels:
        app: fastapi
        slot: green
    spec:
      containers:
        - name: fastapi
          image: myregistry/fastapi-app:2.0.0

---
# Service (switch between blue and green)
apiVersion: v1
kind: Service
metadata:
  name: fastapi-service
spec:
  selector:
    app: fastapi
    slot: blue  # Change to "green" to switch traffic
  ports:
    - port: 80
      targetPort: 8000
```

```bash
# Blue-green switch script
#!/bin/bash
CURRENT=$(kubectl get svc fastapi-service -o jsonpath='{.spec.selector.slot}')
if [ "$CURRENT" = "blue" ]; then
    NEW_SLOT="green"
else
    NEW_SLOT="blue"
fi

echo "Switching from $CURRENT to $NEW_SLOT..."

kubectl patch service fastapi-service \
  -p "{\"spec\":{\"selector\":{\"slot\":\"$NEW_SLOT\"}}}"

echo "Traffic switched to $NEW_SLOT"
```

### Canary Deployment

Gradually roll out changes to a subset of users.

```yaml
# k8s/canary.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-canary
  namespace: production
spec:
  replicas: 1  # 1 out of 4 total = 25% traffic
  selector:
    matchLabels:
      app: fastapi
      track: canary
  template:
    metadata:
      labels:
        app: fastapi
        track: canary
    spec:
      containers:
        - name: fastapi
          image: myregistry/fastapi-app:2.0.0-canary
```

```yaml
# Nginx ingress canary annotation
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fastapi-canary-ingress
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"  # 20% traffic
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: fastapi-canary-service
                port:
                  number: 80
```

## 17.7 Monitoring & Logging

### Application Logging

Set up structured application logging.

```python
import logging
import sys
from datetime import datetime
from fastapi import FastAPI, Request

app = FastAPI()

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    handlers=[
        logging.StreamHandler(sys.stdout),
        logging.FileHandler("app.log")
    ]
)

logger = logging.getLogger(__name__)

@app.middleware("http")
async def log_requests(request: Request, call_next):
    start_time = datetime.now()
    
    logger.info(
        f"Request started",
        extra={
            "method": request.method,
            "path": request.url.path,
            "client": request.client.host
        }
    )
    
    response = await call_next(request)
    
    duration = (datetime.now() - start_time).total_seconds()
    
    logger.info(
        f"Request completed",
        extra={
            "method": request.method,
            "path": request.url.path,
            "status": response.status_code,
            "duration": duration
        }
    )
    
    return response
```

### Structured Logging

Use structured logging for better log analysis.

```python
import structlog
import logging
from fastapi import FastAPI, Request

# Configure structlog
structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_log_level,
        structlog.stdlib.add_logger_name,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.JSONRenderer()
    ],
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
    wrapper_class=structlog.stdlib.BoundLogger,
    cache_logger_on_first_use=True,
)

logger = structlog.get_logger()
app = FastAPI()

@app.middleware("http")
async def structured_logging(request: Request, call_next):
    import uuid
    import time
    
    request_id = str(uuid.uuid4())
    start_time = time.time()
    
    log = logger.bind(
        request_id=request_id,
        method=request.method,
        path=request.url.path,
        client_ip=request.client.host,
    )
    
    log.info("request_started")
    
    response = await call_next(request)
    
    duration = time.time() - start_time
    
    log.info(
        "request_completed",
        status_code=response.status_code,
        duration_ms=round(duration * 1000, 2)
    )
    
    response.headers["X-Request-ID"] = request_id
    return response

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    log = logger.bind(user_id=user_id, action="get_user")
    log.info("fetching_user")
    
    try:
        user = await fetch_user(user_id)
        log.info("user_fetched", username=user.username)
        return user
    except Exception as e:
        log.error("user_fetch_failed", error=str(e))
        raise
```

### Log Aggregation (ELK, CloudWatch)

Ship logs to centralized logging systems.

```python
# ELK Stack integration with python-logstash
import logging
import logstash

app = FastAPI()

# Send logs to Logstash
logger = logging.getLogger('fastapi_app')
logger.addHandler(logstash.TCPLogstashHandler('logstash', 5000, version=1))

# CloudWatch Logging
import boto3
import watchtower

cloudwatch_handler = watchtower.CloudWatchLogHandler(
    log_group="fastapi-app",
    stream_name="production",
    boto3_client=boto3.client('logs', region_name='us-east-1')
)

logging.getLogger().addHandler(cloudwatch_handler)

# JSON logging for ELK
import json_logging

json_logging.init_fastapi(enable_json=True)
json_logging.init_request_instrument(app)
```

### Application Monitoring (Prometheus, Grafana)

Set up metrics collection and visualization.

```python
from prometheus_client import (
    Counter, Histogram, Gauge, Info,
    generate_latest, CONTENT_TYPE_LATEST
)
from fastapi import FastAPI, Request, Response
import time

app = FastAPI()

# Define metrics
http_requests_total = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'http_status']
)

http_request_duration_seconds = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    ['method', 'endpoint'],
    buckets=[0.01, 0.05, 0.1, 0.5, 1.0, 5.0]
)

active_connections = Gauge(
    'active_connections',
    'Number of active connections'
)

app_info = Info('app_info', 'Application information')
app_info.info({'version': '1.0.0', 'environment': 'production'})

@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    start_time = time.time()
    active_connections.inc()
    
    response = await call_next(request)
    
    duration = time.time() - start_time
    endpoint = request.url.path
    
    http_requests_total.labels(
        method=request.method,
        endpoint=endpoint,
        http_status=response.status_code
    ).inc()
    
    http_request_duration_seconds.labels(
        method=request.method,
        endpoint=endpoint
    ).observe(duration)
    
    active_connections.dec()
    return response

@app.get("/metrics")
async def metrics():
    return Response(
        content=generate_latest(),
        media_type=CONTENT_TYPE_LATEST
    )
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'fastapi'
    static_configs:
      - targets: ['app:8000']
    metrics_path: /metrics

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
```

### Error Tracking (Sentry)

Capture and track errors with Sentry.

```python
import sentry_sdk
from sentry_sdk.integrations.fastapi import FastApiIntegration
from sentry_sdk.integrations.sqlalchemy import SqlalchemyIntegration
from sentry_sdk.integrations.redis import RedisIntegration

sentry_sdk.init(
    dsn="https://xxx@sentry.io/123",
    integrations=[
        FastApiIntegration(transaction_style="endpoint"),
        SqlalchemyIntegration(),
        RedisIntegration(),
    ],
    traces_sample_rate=0.1,      # 10% of transactions for performance
    profiles_sample_rate=0.1,    # 10% of sampled transactions for profiling
    environment="production",
    release="myapp@1.0.0",
    before_send=lambda event, hint: event  # Filter/modify events
)

app = FastAPI()

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    with sentry_sdk.start_transaction(op="db", name="fetch user"):
        user = await fetch_user(user_id)
    return user

# Capture exceptions manually
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    sentry_sdk.capture_exception(exc)
    return JSONResponse(status_code=500, content={"detail": "Internal error"})
```

### APM Tools (New Relic, DataDog)

Integrate Application Performance Monitoring.

```python
# Datadog APM
from ddtrace import patch_all, tracer
from ddtrace.contrib.fastapi import TraceMiddleware

patch_all()

app = FastAPI()
app.add_middleware(TraceMiddleware, tracer=tracer)

@app.get("/users")
async def get_users():
    with tracer.trace("db.query", service="users-db", resource="SELECT users"):
        users = await fetch_users()
    return users

# New Relic
import newrelic.agent

newrelic.agent.initialize('newrelic.ini')

@newrelic.agent.background_task()
async def process_background_job():
    pass

@app.get("/metrics/newrelic")
@newrelic.agent.function_trace()
async def tracked_endpoint():
    return {"data": "tracked"}
```

### Health Check Endpoints

Complete health check system.

```python
from fastapi import FastAPI, status
from fastapi.responses import JSONResponse
import time
import asyncio

app = FastAPI()

class HealthChecker:
    def __init__(self):
        self.start_time = time.time()
    
    async def check_database(self) -> dict:
        try:
            start = time.time()
            async with AsyncSessionLocal() as session:
                await session.execute(text("SELECT 1"))
            return {"status": "healthy", "latency_ms": round((time.time() - start) * 1000, 2)}
        except Exception as e:
            return {"status": "unhealthy", "error": str(e)}
    
    async def check_redis(self) -> dict:
        try:
            start = time.time()
            await redis_client.ping()
            return {"status": "healthy", "latency_ms": round((time.time() - start) * 1000, 2)}
        except Exception as e:
            return {"status": "unhealthy", "error": str(e)}
    
    def get_uptime(self) -> float:
        return round(time.time() - self.start_time, 2)

health_checker = HealthChecker()

@app.get("/health")
async def health():
    return {"status": "healthy", "uptime": health_checker.get_uptime()}

@app.get("/health/ready")
async def readiness():
    db_health, redis_health = await asyncio.gather(
        health_checker.check_database(),
        health_checker.check_redis(),
    )
    
    all_healthy = all(
        check["status"] == "healthy"
        for check in [db_health, redis_health]
    )
    
    response = {
        "status": "ready" if all_healthy else "not ready",
        "checks": {
            "database": db_health,
            "redis": redis_health,
        },
        "uptime": health_checker.get_uptime()
    }
    
    return JSONResponse(
        status_code=status.HTTP_200_OK if all_healthy else status.HTTP_503_SERVICE_UNAVAILABLE,
        content=response
    )
```

---

## Complete Production Deployment Checklist

```python
"""
PRODUCTION DEPLOYMENT CHECKLIST

Infrastructure:
✓ Use Gunicorn + Uvicorn workers
✓ Configure appropriate worker count
✓ Set up Nginx reverse proxy
✓ Enable SSL/TLS termination
✓ Configure connection pooling

Security:
✓ Set DEBUG=False
✓ Use environment variables for secrets
✓ Never commit .env files
✓ Run as non-root user in Docker
✓ Scan images for vulnerabilities

Database:
✓ Use async database connections
✓ Configure connection pooling
✓ Run migrations before deploy
✓ Set up database backups

Caching:
✓ Configure Redis for production
✓ Set appropriate TTLs
✓ Implement cache invalidation

Monitoring:
✓ Set up health check endpoints
✓ Configure Prometheus metrics
✓ Set up Grafana dashboards
✓ Integrate Sentry for errors
✓ Configure structured logging
✓ Set up log aggregation

Scaling:
✓ Configure horizontal scaling
✓ Set up load balancing
✓ Configure session persistence
✓ Use distributed caching

CI/CD:
✓ Automated testing on every PR
✓ Automated Docker builds
✓ Staging environment before production
✓ Blue-green or canary deployments
✓ Rollback strategy
"""
```

---

**Key Takeaways:**
- Use Gunicorn with Uvicorn workers in production
- Always containerize with Docker for consistent deployments
- Implement comprehensive health checks for load balancers
- Use Kubernetes for orchestration at scale
- Automate everything with CI/CD pipelines
- Monitor with Prometheus, alert with Grafana
- Track errors with Sentry
- Use structured logging for better observability

**Additional Resources:**
- [Uvicorn Deployment](https://www.uvicorn.org/deployment/)
- [Gunicorn Documentation](https://gunicorn.org/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [GitHub Actions](https://docs.github.com/en/actions)
- [Sentry Documentation](https://docs.sentry.io/)