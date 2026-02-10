# Background Tasks

## 10.1 Background Tasks Basics

### BackgroundTasks

FastAPI provides a `BackgroundTasks` class to run functions after returning a response. This allows you to perform operations without making the client wait.

**Why Use Background Tasks?**
- Send emails after user registration
- Process uploaded files
- Update cache or database
- Send notifications
- Log analytics events
- Clean up temporary files

**Key Characteristics**:
- Executed after the response is sent to the client
- Client doesn't wait for completion
- Run in the same process (not suitable for heavy/long tasks)
- Simple to use, no external dependencies needed

### Adding Background Tasks

**Basic Syntax**:
```python
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

def write_log(message: str):
    """Background task function."""
    with open("log.txt", mode="a") as log:
        log.write(f"{message}\n")

@app.post("/send-notification/")
async def send_notification(
    email: str,
    background_tasks: BackgroundTasks
):
    # Add task to background
    background_tasks.add_task(write_log, f"Notification sent to {email}")
    
    # Response is sent immediately
    return {"message": "Notification sent"}

# Client receives response immediately
# write_log() executes after response is sent
```

**With Parameters**:
```python
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

def send_email(email: str, subject: str, body: str):
    """Simulated email sending."""
    print(f"Sending email to {email}")
    print(f"Subject: {subject}")
    print(f"Body: {body}")
    # Actual email sending logic here

@app.post("/register/")
async def register_user(
    email: str,
    username: str,
    background_tasks: BackgroundTasks
):
    # Register user logic here
    user = {"email": email, "username": username}
    
    # Send welcome email in background
    background_tasks.add_task(
        send_email,
        email=email,
        subject="Welcome!",
        body=f"Hello {username}, welcome to our platform!"
    )
    
    return {"message": "User registered", "user": user}
```

**Multiple Background Tasks**:
```python
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

def task_1(message: str):
    print(f"Task 1: {message}")

def task_2(message: str):
    print(f"Task 2: {message}")

def task_3(message: str):
    print(f"Task 3: {message}")

@app.post("/process/")
async def process_data(background_tasks: BackgroundTasks):
    # Add multiple tasks
    background_tasks.add_task(task_1, "First task")
    background_tasks.add_task(task_2, "Second task")
    background_tasks.add_task(task_3, "Third task")
    
    return {"message": "Processing started"}

# Tasks execute in order after response is sent
```

### Background Task Execution

**Execution Order**:
```python
from fastapi import FastAPI, BackgroundTasks
import time

app = FastAPI()

def slow_task(name: str, duration: int):
    print(f"{name} started")
    time.sleep(duration)
    print(f"{name} completed")

@app.get("/demo/")
async def demo(background_tasks: BackgroundTasks):
    print("1. Request received")
    
    background_tasks.add_task(slow_task, "Task A", 2)
    background_tasks.add_task(slow_task, "Task B", 1)
    
    print("2. About to send response")
    return {"message": "Tasks queued"}
    # 3. Response sent to client
    # 4. Task A executes (2 seconds)
    # 5. Task B executes (1 second)

# Console output:
# 1. Request received
# 2. About to send response
# [Response sent to client here]
# Task A started
# Task A completed
# Task B started
# Task B completed
```

**Async Background Tasks**:
```python
from fastapi import FastAPI, BackgroundTasks
import asyncio

app = FastAPI()

async def async_task(name: str):
    """Async background task."""
    print(f"{name} started")
    await asyncio.sleep(2)
    print(f"{name} completed")

def sync_task(name: str):
    """Sync background task."""
    print(f"{name} started")
    time.sleep(2)
    print(f"{name} completed")

@app.get("/mixed/")
async def mixed_tasks(background_tasks: BackgroundTasks):
    # Can mix sync and async tasks
    background_tasks.add_task(async_task, "Async Task")
    background_tasks.add_task(sync_task, "Sync Task")
    
    return {"message": "Tasks queued"}
```

### Use Cases for Background Tasks

**1. Email Sending**:
```python
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

async def send_welcome_email(email: str, username: str):
    # Simulated email sending
    await asyncio.sleep(2)  # Network delay
    print(f"Welcome email sent to {email}")

@app.post("/signup/")
async def signup(
    email: str,
    username: str,
    background_tasks: BackgroundTasks
):
    # Create user (fast)
    user = {"email": email, "username": username}
    
    # Send email (slow) - don't make user wait
    background_tasks.add_task(send_welcome_email, email, username)
    
    return {"message": "Signup successful"}
```

**2. File Processing**:
```python
import os
from fastapi import FastAPI, BackgroundTasks, UploadFile

app = FastAPI()

def process_file(filename: str):
    """Process uploaded file."""
    print(f"Processing {filename}")
    # Resize images, extract text, etc.
    time.sleep(5)  # Simulated processing
    print(f"Finished processing {filename}")

@app.post("/upload/")
async def upload_file(
    file: UploadFile,
    background_tasks: BackgroundTasks
):
    # Save file quickly
    filepath = f"uploads/{file.filename}"
    content = await file.read()
    with open(filepath, "wb") as f:
        f.write(content)
    
    # Process in background
    background_tasks.add_task(process_file, filepath)
    
    return {"filename": file.filename, "status": "uploaded"}
```

**3. Analytics/Logging**:
```python
from fastapi import FastAPI, BackgroundTasks, Request
from datetime import datetime

app = FastAPI()

def log_request(endpoint: str, method: str, ip: str):
    """Log request analytics."""
    log_entry = {
        "timestamp": datetime.now().isoformat(),
        "endpoint": endpoint,
        "method": method,
        "ip": ip
    }
    # Save to database or file
    print(f"Logged: {log_entry}")

@app.get("/products/")
async def get_products(
    request: Request,
    background_tasks: BackgroundTasks
):
    # Log analytics without slowing down response
    background_tasks.add_task(
        log_request,
        endpoint=request.url.path,
        method=request.method,
        ip=request.client.host
    )
    
    return {"products": []}
```

**4. Cache Updates**:
```python
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

cache = {}

def update_cache(key: str, value: any):
    """Update cache in background."""
    print(f"Updating cache: {key} = {value}")
    cache[key] = value

@app.post("/items/")
async def create_item(
    name: str,
    price: float,
    background_tasks: BackgroundTasks
):
    item = {"name": name, "price": price}
    
    # Update cache without blocking
    background_tasks.add_task(update_cache, f"item:{name}", item)
    
    return item
```

---

## 10.2 Task Implementation

### Sending Emails in Background

```python
from fastapi import FastAPI, BackgroundTasks
from pydantic import BaseModel, EmailStr
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

app = FastAPI()

class EmailSchema(BaseModel):
    to: EmailStr
    subject: str
    body: str

def send_email(email: EmailSchema):
    """Send email via SMTP."""
    try:
        # Email configuration
        smtp_server = "smtp.gmail.com"
        smtp_port = 587
        sender_email = "your-email@gmail.com"
        sender_password = "your-password"
        
        # Create message
        message = MIMEMultipart()
        message["From"] = sender_email
        message["To"] = email.to
        message["Subject"] = email.subject
        message.attach(MIMEText(email.body, "plain"))
        
        # Send email
        with smtplib.SMTP(smtp_server, smtp_port) as server:
            server.starttls()
            server.login(sender_email, sender_password)
            server.send_message(message)
        
        print(f"Email sent to {email.to}")
    except Exception as e:
        print(f"Failed to send email: {e}")

@app.post("/send-email/")
async def send_email_endpoint(
    email: EmailSchema,
    background_tasks: BackgroundTasks
):
    background_tasks.add_task(send_email, email)
    return {"message": "Email queued for sending"}

# Template-based emails
def send_template_email(to: str, template: str, **kwargs):
    """Send email using template."""
    templates = {
        "welcome": "Hello {username}, welcome to our platform!",
        "reset_password": "Click here to reset your password: {link}",
        "notification": "You have a new notification: {message}"
    }
    
    body = templates[template].format(**kwargs)
    email = EmailSchema(to=to, subject=template.title(), body=body)
    send_email(email)

@app.post("/register/")
async def register(
    email: str,
    username: str,
    background_tasks: BackgroundTasks
):
    background_tasks.add_task(
        send_template_email,
        to=email,
        template="welcome",
        username=username
    )
    return {"message": "User registered"}
```

### File Processing

```python
from fastapi import FastAPI, BackgroundTasks, UploadFile
from PIL import Image
import io
import os

app = FastAPI()

def process_image(filepath: str):
    """Process image: resize, optimize, create thumbnails."""
    try:
        img = Image.open(filepath)
        
        # Create thumbnail
        img.thumbnail((200, 200))
        thumb_path = filepath.replace(".", "_thumb.")
        img.save(thumb_path)
        
        # Optimize original
        img = Image.open(filepath)
        img.save(filepath, optimize=True, quality=85)
        
        print(f"Processed: {filepath}")
    except Exception as e:
        print(f"Error processing {filepath}: {e}")

def extract_text_from_pdf(filepath: str):
    """Extract text from PDF."""
    # Using PyPDF2 or similar
    print(f"Extracting text from {filepath}")
    # text = extract_text(filepath)
    # save_to_database(text)

def process_csv(filepath: str):
    """Process CSV file."""
    import csv
    
    with open(filepath, 'r') as f:
        reader = csv.DictReader(f)
        for row in reader:
            # Process each row
            print(f"Processing: {row}")

@app.post("/upload-image/")
async def upload_image(
    file: UploadFile,
    background_tasks: BackgroundTasks
):
    # Save file
    filepath = f"uploads/{file.filename}"
    content = await file.read()
    with open(filepath, "wb") as f:
        f.write(content)
    
    # Process in background
    background_tasks.add_task(process_image, filepath)
    
    return {"filename": file.filename, "status": "processing"}

@app.post("/upload-csv/")
async def upload_csv(
    file: UploadFile,
    background_tasks: BackgroundTasks
):
    filepath = f"uploads/{file.filename}"
    content = await file.read()
    with open(filepath, "wb") as f:
        f.write(content)
    
    background_tasks.add_task(process_csv, filepath)
    
    return {"filename": file.filename, "status": "queued"}
```

### Data Cleanup

```python
from fastapi import FastAPI, BackgroundTasks
from datetime import datetime, timedelta
import os

app = FastAPI()

def cleanup_old_files(directory: str, days: int = 7):
    """Delete files older than specified days."""
    cutoff = datetime.now() - timedelta(days=days)
    deleted_count = 0
    
    for filename in os.listdir(directory):
        filepath = os.path.join(directory, filename)
        
        if os.path.isfile(filepath):
            file_time = datetime.fromtimestamp(os.path.getmtime(filepath))
            
            if file_time < cutoff:
                os.remove(filepath)
                deleted_count += 1
                print(f"Deleted: {filepath}")
    
    print(f"Cleanup complete: {deleted_count} files deleted")

def cleanup_temp_data(user_id: int):
    """Remove temporary user data."""
    temp_cache = f"cache/user_{user_id}_temp"
    if os.path.exists(temp_cache):
        os.remove(temp_cache)
        print(f"Cleaned up temp data for user {user_id}")

def archive_old_records(table: str, days: int = 30):
    """Archive database records older than specified days."""
    # Move old records to archive table
    print(f"Archiving {table} records older than {days} days")
    # db.execute(f"INSERT INTO {table}_archive SELECT * FROM {table} WHERE date < ...")
    # db.execute(f"DELETE FROM {table} WHERE date < ...")

@app.post("/logout/")
async def logout(
    user_id: int,
    background_tasks: BackgroundTasks
):
    # Logout logic
    background_tasks.add_task(cleanup_temp_data, user_id)
    return {"message": "Logged out"}

@app.post("/trigger-cleanup/")
async def trigger_cleanup(background_tasks: BackgroundTasks):
    background_tasks.add_task(cleanup_old_files, "uploads", 7)
    background_tasks.add_task(archive_old_records, "logs", 30)
    return {"message": "Cleanup scheduled"}
```

### Logging and Analytics

```python
from fastapi import FastAPI, BackgroundTasks, Request
from datetime import datetime
from typing import Optional

app = FastAPI()

# Analytics database (simulated)
analytics_db = []

def log_api_call(
    endpoint: str,
    method: str,
    status_code: int,
    duration: float,
    user_id: Optional[int] = None
):
    """Log API call for analytics."""
    entry = {
        "timestamp": datetime.now().isoformat(),
        "endpoint": endpoint,
        "method": method,
        "status_code": status_code,
        "duration_ms": duration,
        "user_id": user_id
    }
    analytics_db.append(entry)
    print(f"Logged: {entry}")

def track_event(event_name: str, user_id: int, properties: dict):
    """Track custom event."""
    event = {
        "event": event_name,
        "user_id": user_id,
        "timestamp": datetime.now().isoformat(),
        "properties": properties
    }
    # Send to analytics service (Mixpanel, Amplitude, etc.)
    print(f"Event tracked: {event}")

@app.post("/purchase/")
async def purchase_item(
    item_id: int,
    user_id: int,
    price: float,
    background_tasks: BackgroundTasks
):
    # Process purchase
    purchase = {"item_id": item_id, "user_id": user_id, "price": price}
    
    # Track event
    background_tasks.add_task(
        track_event,
        event_name="purchase",
        user_id=user_id,
        properties={"item_id": item_id, "price": price}
    )
    
    return purchase

# Middleware for automatic logging
from starlette.middleware.base import BaseHTTPMiddleware
import time

class AnalyticsMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        start_time = time.time()
        response = await call_next(request)
        duration = (time.time() - start_time) * 1000
        
        # Log in background
        # Can't use BackgroundTasks here, but can spawn async task
        asyncio.create_task(
            asyncio.to_thread(
                log_api_call,
                endpoint=request.url.path,
                method=request.method,
                status_code=response.status_code,
                duration=duration
            )
        )
        
        return response

app.add_middleware(AnalyticsMiddleware)
```

### Notification Sending

```python
from fastapi import FastAPI, BackgroundTasks
from typing import List
from enum import Enum

app = FastAPI()

class NotificationType(str, Enum):
    EMAIL = "email"
    SMS = "sms"
    PUSH = "push"
    SLACK = "slack"

def send_notification(
    user_id: int,
    message: str,
    notification_type: NotificationType
):
    """Send notification via specified channel."""
    print(f"Sending {notification_type} to user {user_id}: {message}")
    
    if notification_type == NotificationType.EMAIL:
        # Send email
        pass
    elif notification_type == NotificationType.SMS:
        # Send SMS via Twilio
        pass
    elif notification_type == NotificationType.PUSH:
        # Send push notification
        pass
    elif notification_type == NotificationType.SLACK:
        # Send Slack message
        pass

def notify_multiple_users(
    user_ids: List[int],
    message: str,
    notification_type: NotificationType
):
    """Send notification to multiple users."""
    for user_id in user_ids:
        send_notification(user_id, message, notification_type)
        time.sleep(0.1)  # Rate limiting

@app.post("/announce/")
async def announce(
    message: str,
    background_tasks: BackgroundTasks
):
    """Send announcement to all users."""
    user_ids = [1, 2, 3, 4, 5]  # Get from database
    
    background_tasks.add_task(
        notify_multiple_users,
        user_ids=user_ids,
        message=message,
        notification_type=NotificationType.EMAIL
    )
    
    return {"message": "Announcement queued"}

@app.post("/alert/{user_id}")
async def send_alert(
    user_id: int,
    message: str,
    background_tasks: BackgroundTasks
):
    """Send alert to specific user via multiple channels."""
    background_tasks.add_task(
        send_notification,
        user_id=user_id,
        message=message,
        notification_type=NotificationType.EMAIL
    )
    background_tasks.add_task(
        send_notification,
        user_id=user_id,
        message=message,
        notification_type=NotificationType.PUSH
    )
    
    return {"message": "Alert sent"}
```

---

## 10.3 Task Queues

For production applications with heavy background processing, use dedicated task queues.

### Celery Integration

**Installation**:
```bash
pip install celery redis
```

**celery_worker.py**:
```python
from celery import Celery

# Configure Celery
celery_app = Celery(
    "tasks",
    broker="redis://localhost:6379/0",
    backend="redis://localhost:6379/1"
)

celery_app.conf.update(
    task_serializer="json",
    result_serializer="json",
    accept_content=["json"],
    timezone="UTC",
    enable_utc=True,
)

@celery_app.task
def send_email_task(email: str, subject: str, body: str):
    """Celery task for sending email."""
    print(f"Sending email to {email}")
    time.sleep(5)  # Simulate email sending
    print(f"Email sent to {email}")
    return {"status": "sent", "email": email}

@celery_app.task
def process_image_task(filepath: str):
    """Celery task for image processing."""
    from PIL import Image
    
    img = Image.open(filepath)
    img.thumbnail((200, 200))
    thumb_path = filepath.replace(".", "_thumb.")
    img.save(thumb_path)
    
    return {"status": "processed", "thumbnail": thumb_path}

@celery_app.task
def generate_report_task(user_id: int, report_type: str):
    """Long-running report generation."""
    print(f"Generating {report_type} report for user {user_id}")
    time.sleep(30)  # Simulate long processing
    return {"status": "complete", "report_id": 12345}
```

**main.py**:
```python
from fastapi import FastAPI
from celery_worker import send_email_task, process_image_task, generate_report_task

app = FastAPI()

@app.post("/send-email/")
async def send_email(email: str, subject: str, body: str):
    """Queue email task."""
    task = send_email_task.delay(email, subject, body)
    return {
        "message": "Email queued",
        "task_id": task.id
    }

@app.get("/task-status/{task_id}")
async def get_task_status(task_id: str):
    """Check task status."""
    from celery.result import AsyncResult
    
    task = AsyncResult(task_id, app=celery_app)
    
    return {
        "task_id": task_id,
        "status": task.state,
        "result": task.result if task.ready() else None
    }

@app.post("/generate-report/")
async def generate_report(user_id: int, report_type: str):
    """Queue long-running report generation."""
    task = generate_report_task.delay(user_id, report_type)
    return {
        "message": "Report generation started",
        "task_id": task.id,
        "status_url": f"/task-status/{task.id}"
    }
```

**Running Celery Worker**:
```bash
celery -A celery_worker worker --loglevel=info
```

### Redis as Message Broker

Redis is commonly used as the message broker for Celery.

**Configuration**:
```python
from celery import Celery

celery_app = Celery(
    "tasks",
    broker="redis://localhost:6379/0",      # Message broker
    backend="redis://localhost:6379/1"      # Result backend
)

# Advanced Redis configuration
celery_app.conf.update(
    broker_url="redis://localhost:6379/0",
    result_backend="redis://localhost:6379/1",
    broker_connection_retry_on_startup=True,
    redis_max_connections=50,
    redis_socket_timeout=5,
)
```

**Using Redis Directly**:
```python
import redis
from fastapi import FastAPI
import json

app = FastAPI()
redis_client = redis.Redis(host='localhost', port=6379, db=0)

def background_worker():
    """Simple Redis-based worker."""
    while True:
        # Pop task from queue
        task = redis_client.blpop("task_queue", timeout=1)
        if task:
            task_data = json.loads(task[1])
            process_task(task_data)

@app.post("/queue-task/")
async def queue_task(task_type: str, data: dict):
    """Add task to Redis queue."""
    task = {
        "type": task_type,
        "data": data,
        "timestamp": datetime.now().isoformat()
    }
    redis_client.rpush("task_queue", json.dumps(task))
    return {"message": "Task queued"}
```

### ARQ (Async Task Queue)

ARQ is an async task queue built on asyncio and Redis.

**Installation**:
```bash
pip install arq
```

**worker.py**:
```python
import asyncio
from arq import create_pool
from arq.connections import RedisSettings

async def send_email(ctx, email: str, subject: str, body: str):
    """Async task for sending email."""
    print(f"Sending email to {email}")
    await asyncio.sleep(2)  # Simulate async email sending
    print(f"Email sent to {email}")
    return {"status": "sent"}

async def process_data(ctx, data_id: int):
    """Async task for data processing."""
    print(f"Processing data {data_id}")
    await asyncio.sleep(5)
    return {"status": "processed", "data_id": data_id}

class WorkerSettings:
    functions = [send_email, process_data]
    redis_settings = RedisSettings(host='localhost', port=6379)
```

**main.py**:
```python
from fastapi import FastAPI
from arq import create_pool
from arq.connections import RedisSettings

app = FastAPI()

@app.on_event("startup")
async def startup():
    app.state.arq_pool = await create_pool(
        RedisSettings(host='localhost', port=6379)
    )

@app.on_event("shutdown")
async def shutdown():
    await app.state.arq_pool.close()

@app.post("/send-email/")
async def send_email_endpoint(email: str, subject: str, body: str):
    job = await app.state.arq_pool.enqueue_job(
        'send_email',
        email,
        subject,
        body
    )
    return {"job_id": job.job_id}
```

**Running ARQ Worker**:
```bash
arq worker.WorkerSettings
```

### RQ (Redis Queue)

RQ is a simple Redis-based task queue.

**Installation**:
```bash
pip install rq
```

**tasks.py**:
```python
import time

def long_task(duration: int):
    """Long-running task."""
    print(f"Task started, will run for {duration} seconds")
    time.sleep(duration)
    print("Task completed")
    return {"status": "complete", "duration": duration}

def send_email(to: str, subject: str, body: str):
    """Send email task."""
    print(f"Sending email to {to}")
    time.sleep(2)
    return {"sent": True}
```

**main.py**:
```python
from fastapi import FastAPI
from redis import Redis
from rq import Queue
from tasks import long_task, send_email

app = FastAPI()

# Connect to Redis
redis_conn = Redis(host='localhost', port=6379)
queue = Queue(connection=redis_conn)

@app.post("/enqueue-task/")
async def enqueue_task(duration: int):
    """Enqueue task to RQ."""
    job = queue.enqueue(long_task, duration)
    return {
        "job_id": job.id,
        "status": job.get_status()
    }

@app.get("/job-status/{job_id}")
async def get_job_status(job_id: str):
    """Check job status."""
    from rq.job import Job
    
    job = Job.fetch(job_id, connection=redis_conn)
    return {
        "job_id": job_id,
        "status": job.get_status(),
        "result": job.result
    }
```

**Running RQ Worker**:
```bash
rq worker --with-scheduler
```

### Task Scheduling

**Celery Beat (Periodic Tasks)**:
```python
from celery import Celery
from celery.schedules import crontab

celery_app = Celery("tasks", broker="redis://localhost:6379/0")

# Configure periodic tasks
celery_app.conf.beat_schedule = {
    'cleanup-every-day': {
        'task': 'tasks.cleanup_old_files',
        'schedule': crontab(hour=2, minute=0),  # Every day at 2 AM
    },
    'send-reports-weekly': {
        'task': 'tasks.send_weekly_report',
        'schedule': crontab(day_of_week=1, hour=9, minute=0),  # Monday 9 AM
    },
    'check-status-every-5-minutes': {
        'task': 'tasks.check_system_status',
        'schedule': 300.0,  # Every 5 minutes (in seconds)
    },
}

@celery_app.task
def cleanup_old_files():
    """Scheduled cleanup task."""
    print("Running scheduled cleanup")
    # Cleanup logic

@celery_app.task
def send_weekly_report():
    """Send weekly report."""
    print("Sending weekly report")

@celery_app.task
def check_system_status():
    """Check system status."""
    print("Checking system status")
```

**Run Celery Beat**:
```bash
celery -A celery_worker beat --loglevel=info
celery -A celery_worker worker --loglevel=info
```

### Periodic Tasks

**APScheduler with FastAPI**:
```python
from fastapi import FastAPI
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from datetime import datetime

app = FastAPI()
scheduler = AsyncIOScheduler()

async def scheduled_task():
    """Task that runs periodically."""
    print(f"Scheduled task executed at {datetime.now()}")

@app.on_event("startup")
async def startup_event():
    # Add periodic job
    scheduler.add_job(scheduled_task, 'interval', minutes=5)
    
    # Add cron job
    scheduler.add_job(
        scheduled_task,
        'cron',
        day_of_week='mon-fri',
        hour=9,
        minute=0
    )
    
    scheduler.start()

@app.on_event("shutdown")
async def shutdown_event():
    scheduler.shutdown()
```

### Task Monitoring

**Celery Flower (Web-based monitoring)**:
```bash
pip install flower
flower -A celery_worker --port=5555
```

**Custom Monitoring Endpoint**:
```python
from fastapi import FastAPI
from celery_worker import celery_app

app = FastAPI()

@app.get("/tasks/stats")
async def get_task_stats():
    """Get task statistics."""
    inspect = celery_app.control.inspect()
    
    return {
        "active": inspect.active(),
        "scheduled": inspect.scheduled(),
        "reserved": inspect.reserved(),
        "stats": inspect.stats()
    }

@app.get("/tasks/active")
async def get_active_tasks():
    """Get currently running tasks."""
    inspect = celery_app.control.inspect()
    active = inspect.active()
    return {"active_tasks": active}
```

---

## 10.4 Async Background Processing

### asyncio.create_task()

For truly asynchronous background tasks without blocking.

```python
from fastapi import FastAPI
import asyncio

app = FastAPI()

async def long_running_task(task_id: int):
    """Long-running async task."""
    print(f"Task {task_id} started")
    await asyncio.sleep(10)
    print(f"Task {task_id} completed")
    return {"task_id": task_id, "status": "complete"}

@app.post("/start-task/")
async def start_task(task_id: int):
    """Start task without waiting."""
    # Create task - it runs independently
    asyncio.create_task(long_running_task(task_id))
    
    # Return immediately
    return {"message": f"Task {task_id} started"}

# Multiple concurrent tasks
@app.post("/batch-tasks/")
async def batch_tasks(count: int):
    """Start multiple tasks concurrently."""
    tasks = [
        asyncio.create_task(long_running_task(i))
        for i in range(count)
    ]
    
    return {"message": f"Started {count} tasks"}
```

### Background Task Patterns

**Fire and Forget**:
```python
from fastapi import FastAPI
import asyncio

app = FastAPI()

async def fire_and_forget_task(data: dict):
    """Task that runs independently."""
    await asyncio.sleep(5)
    print(f"Task completed with data: {data}")

@app.post("/trigger/")
async def trigger_task(data: dict):
    # Start task without waiting
    asyncio.create_task(fire_and_forget_task(data))
    return {"message": "Task triggered"}
```

**Task with Result Storage**:
```python
from fastapi import FastAPI
import asyncio
from typing import Dict

app = FastAPI()

# Store task results
task_results: Dict[str, any] = {}

async def task_with_result(task_id: str, duration: int):
    """Task that stores its result."""
    await asyncio.sleep(duration)
    result = {"completed_at": datetime.now().isoformat()}
    task_results[task_id] = result
    return result

@app.post("/start/{task_id}")
async def start_task(task_id: str, duration: int):
    asyncio.create_task(task_with_result(task_id, duration))
    return {"task_id": task_id, "status": "started"}

@app.get("/result/{task_id}")
async def get_result(task_id: str):
    if task_id in task_results:
        return task_results[task_id]
    return {"status": "not ready"}
```

**Task with Progress Tracking**:
```python
from fastapi import FastAPI
import asyncio

app = FastAPI()

task_progress = {}

async def task_with_progress(task_id: str, steps: int):
    """Task that reports progress."""
    task_progress[task_id] = {"progress": 0, "status": "running"}
    
    for i in range(steps):
        await asyncio.sleep(1)
        progress = ((i + 1) / steps) * 100
        task_progress[task_id] = {
            "progress": progress,
            "status": "running"
        }
    
    task_progress[task_id] = {"progress": 100, "status": "complete"}

@app.post("/process/{task_id}")
async def start_processing(task_id: str, steps: int = 10):
    asyncio.create_task(task_with_progress(task_id, steps))
    return {"task_id": task_id}

@app.get("/progress/{task_id}")
async def get_progress(task_id: str):
    return task_progress.get(task_id, {"status": "not found"})
```

### Long-Running Tasks

```python
from fastapi import FastAPI
import asyncio
from datetime import datetime

app = FastAPI()

class TaskManager:
    def __init__(self):
        self.tasks = {}
    
    async def run_task(self, task_id: str, duration: int):
        """Run long task with tracking."""
        self.tasks[task_id] = {
            "status": "running",
            "started_at": datetime.now().isoformat(),
            "progress": 0
        }
        
        try:
            for i in range(duration):
                await asyncio.sleep(1)
                self.tasks[task_id]["progress"] = (i + 1) / duration * 100
            
            self.tasks[task_id]["status"] = "completed"
            self.tasks[task_id]["completed_at"] = datetime.now().isoformat()
        except asyncio.CancelledError:
            self.tasks[task_id]["status"] = "cancelled"
            raise
        except Exception as e:
            self.tasks[task_id]["status"] = "failed"
            self.tasks[task_id]["error"] = str(e)
    
    def get_task_status(self, task_id: str):
        return self.tasks.get(task_id, {"status": "not found"})

task_manager = TaskManager()

@app.post("/long-task/{task_id}")
async def start_long_task(task_id: str, duration: int):
    asyncio.create_task(task_manager.run_task(task_id, duration))
    return {"task_id": task_id, "message": "Task started"}

@app.get("/task-status/{task_id}")
async def get_task_status(task_id: str):
    return task_manager.get_task_status(task_id)
```

### Task Cancellation

```python
from fastapi import FastAPI
import asyncio

app = FastAPI()

# Store running tasks
running_tasks = {}

async def cancellable_task(task_id: str):
    """Task that can be cancelled."""
    try:
        print(f"Task {task_id} started")
        for i in range(100):
            await asyncio.sleep(1)
            print(f"Task {task_id}: step {i}")
    except asyncio.CancelledError:
        print(f"Task {task_id} was cancelled")
        raise
    finally:
        print(f"Task {task_id} cleanup")

@app.post("/start/{task_id}")
async def start_cancellable_task(task_id: str):
    task = asyncio.create_task(cancellable_task(task_id))
    running_tasks[task_id] = task
    return {"task_id": task_id, "status": "started"}

@app.post("/cancel/{task_id}")
async def cancel_task(task_id: str):
    if task_id in running_tasks:
        task = running_tasks[task_id]
        task.cancel()
        try:
            await task
        except asyncio.CancelledError:
            pass
        del running_tasks[task_id]
        return {"task_id": task_id, "status": "cancelled"}
    return {"error": "Task not found"}

@app.get("/tasks/")
async def list_tasks():
    return {
        "running_tasks": list(running_tasks.keys()),
        "count": len(running_tasks)
    }
```

---

## Complete Example: Image Processing Service

```python
from fastapi import FastAPI, BackgroundTasks, UploadFile, HTTPException
from celery import Celery
import asyncio
from typing import Dict, Optional
from enum import Enum
from datetime import datetime
import os

app = FastAPI(title="Image Processing Service")

# Celery configuration
celery_app = Celery(
    "tasks",
    broker="redis://localhost:6379/0",
    backend="redis://localhost:6379/1"
)

# Task status storage
task_status: Dict[str, dict] = {}

class ProcessingStatus(str, Enum):
    QUEUED = "queued"
    PROCESSING = "processing"
    COMPLETED = "completed"
    FAILED = "failed"

# ========== Celery Tasks ==========
@celery_app.task(bind=True)
def process_image_celery(self, task_id: str, filepath: str):
    """Heavy image processing with Celery."""
    from PIL import Image
    
    try:
        task_status[task_id] = {"status": ProcessingStatus.PROCESSING}
        
        # Open image
        img = Image.open(filepath)
        
        # Create thumbnail
        img.thumbnail((200, 200))
        thumb_path = filepath.replace(".", "_thumb.")
        img.save(thumb_path)
        
        # Create medium size
        img = Image.open(filepath)
        img.thumbnail((800, 800))
        medium_path = filepath.replace(".", "_medium.")
        img.save(medium_path)
        
        task_status[task_id] = {
            "status": ProcessingStatus.COMPLETED,
            "original": filepath,
            "thumbnail": thumb_path,
            "medium": medium_path,
            "completed_at": datetime.now().isoformat()
        }
        
        return task_status[task_id]
    except Exception as e:
        task_status[task_id] = {
            "status": ProcessingStatus.FAILED,
            "error": str(e)
        }
        raise

# ========== Background Tasks (Light Processing) ==========
async def quick_resize(filepath: str, size: tuple):
    """Quick resize for small operations."""
    from PIL import Image
    
    img = Image.open(filepath)
    img.thumbnail(size)
    output_path = filepath.replace(".", f"_{size[0]}x{size[1]}.")
    img.save(output_path)
    return output_path

def log_upload(filename: str, user_id: Optional[int] = None):
    """Log file upload."""
    print(f"Upload logged: {filename} by user {user_id}")

# ========== Endpoints ==========
@app.post("/upload/quick/")
async def upload_quick(
    file: UploadFile,
    background_tasks: BackgroundTasks
):
    """Quick processing with BackgroundTasks."""
    # Save file
    filepath = f"uploads/{file.filename}"
    content = await file.read()
    with open(filepath, "wb") as f:
        f.write(content)
    
    # Quick resize in background
    background_tasks.add_task(quick_resize, filepath, (400, 400))
    background_tasks.add_task(log_upload, file.filename)
    
    return {
        "filename": file.filename,
        "status": "processing",
        "method": "background_tasks"
    }

@app.post("/upload/heavy/")
async def upload_heavy(file: UploadFile):
    """Heavy processing with Celery."""
    # Save file
    filepath = f"uploads/{file.filename}"
    content = await file.read()
    with open(filepath, "wb") as f:
        f.write(content)
    
    # Queue Celery task
    task_id = f"task_{datetime.now().timestamp()}"
    task_status[task_id] = {"status": ProcessingStatus.QUEUED}
    
    process_image_celery.delay(task_id, filepath)
    
    return {
        "task_id": task_id,
        "filename": file.filename,
        "status": "queued",
        "method": "celery",
        "status_url": f"/status/{task_id}"
    }

@app.post("/upload/async/")
async def upload_async(file: UploadFile):
    """Async processing with create_task."""
    filepath = f"uploads/{file.filename}"
    content = await file.read()
    with open(filepath, "wb") as f:
        f.write(content)
    
    task_id = f"async_{datetime.now().timestamp()}"
    
    async def async_process():
        task_status[task_id] = {"status": ProcessingStatus.PROCESSING}
        try:
            result = await quick_resize(filepath, (600, 600))
            task_status[task_id] = {
                "status": ProcessingStatus.COMPLETED,
                "result": result
            }
        except Exception as e:
            task_status[task_id] = {
                "status": ProcessingStatus.FAILED,
                "error": str(e)
            }
    
    asyncio.create_task(async_process())
    
    return {
        "task_id": task_id,
        "status": "processing",
        "method": "asyncio"
    }

@app.get("/status/{task_id}")
async def get_status(task_id: str):
    """Get task status."""
    if task_id not in task_status:
        raise HTTPException(status_code=404, detail="Task not found")
    
    return task_status[task_id]

@app.get("/health")
async def health_check():
    """Health check endpoint."""
    return {
        "status": "healthy",
        "tasks_tracked": len(task_status),
        "celery_active": celery_app.control.inspect().active() is not None
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## Summary

You've learned about Background Tasks in FastAPI:

1. **Background Tasks Basics**: Using `BackgroundTasks` for simple operations after response
2. **Task Implementation**: Emails, file processing, cleanup, logging, and notifications
3. **Task Queues**: Celery, Redis, ARQ, RQ, scheduling, and monitoring
4. **Async Background Processing**: `asyncio.create_task()`, patterns, long-running tasks, and cancellation

**When to Use What**:
- **BackgroundTasks**: Simple, quick operations (< 1 second)
- **asyncio.create_task()**: Async operations within the same process
- **Celery/ARQ/RQ**: Heavy, long-running tasks, distributed processing
- **Scheduled Tasks**: Periodic operations, cleanup, reports

Choose the right tool based on your task complexity, duration, and infrastructure requirements!