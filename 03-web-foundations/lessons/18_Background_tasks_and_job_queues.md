# Background Tasks and Job Queues in Python

## Table of Contents
1. [Introduction](#introduction)
2. [Why Use Background Tasks?](#why-use-background-tasks)
3. [Celery - Distributed Task Queue](#celery)
4. [RQ (Redis Queue) - Simple Task Queue](#rq)
5. [Comparison: Celery vs RQ](#comparison)
6. [Best Practices](#best-practices)
7. [Production Considerations](#production-considerations)

---

## Introduction

Background tasks and job queues are essential for building scalable web applications. They allow you to offload time-consuming operations from your main application thread, ensuring your application remains responsive while processing intensive tasks asynchronously.

### What are Background Tasks?

Background tasks are operations that run independently of your main application flow. Instead of making users wait for slow operations to complete, you queue them for processing and immediately return a response.

### What are Job Queues?

Job queues are systems that manage the execution of background tasks. They typically consist of:
- **Producer**: Application code that creates tasks
- **Broker**: Message queue (Redis, RabbitMQ) that stores tasks
- **Worker**: Process that executes tasks
- **Backend**: Storage for task results (optional)

---

## Why Use Background Tasks?

### Common Use Cases

1. **Email Sending**
   - Newsletter distribution
   - Transactional emails
   - Email notifications

2. **Data Processing**
   - Report generation
   - Image/video processing
   - Data import/export
   - ETL operations

3. **External API Calls**
   - Third-party service integration
   - Webhook processing
   - Payment processing

4. **Scheduled Jobs**
   - Daily database cleanup
   - Periodic data synchronization
   - Scheduled reports

5. **Resource-Intensive Operations**
   - Machine learning model training
   - Data analytics
   - File conversion

### Benefits

- **Improved User Experience**: Users don't wait for slow operations
- **Better Resource Utilization**: Distribute work across multiple workers
- **Scalability**: Add more workers to handle increased load
- **Reliability**: Retry failed tasks automatically
- **Priority Management**: Execute critical tasks first

---

## Celery - Distributed Task Queue

Celery is a powerful, production-ready distributed task queue that supports multiple message brokers and result backends.

### Architecture

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│             │  Task   │             │  Task   │             │
│ Application ├────────►│   Broker    ├────────►│   Worker    │
│             │         │ (Redis/     │         │             │
└─────────────┘         │  RabbitMQ)  │         └──────┬──────┘
                        └─────────────┘                │
                                                       │ Result
                        ┌─────────────┐                │
                        │   Result    │◄───────────────┘
                        │   Backend   │
                        └─────────────┘
```

### Installation

```bash
# Basic Celery with Redis
pip install celery redis

# With RabbitMQ support
pip install celery[amqp]

# Full installation with all features
pip install celery[redis,auth,msgpack]
```

### Basic Setup

#### 1. Project Structure

```
project/
├── celery_app.py      # Celery configuration
├── tasks.py           # Task definitions
├── config.py          # Application config
└── app.py             # Main application
```

#### 2. Celery Configuration (`celery_app.py`)

```python
from celery import Celery
from kombu import Exchange, Queue

# Initialize Celery
app = Celery(
    'myapp',
    broker='redis://localhost:6379/0',
    backend='redis://localhost:6379/1'
)

# Configuration
app.conf.update(
    # Task execution settings
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='UTC',
    enable_utc=True,
    
    # Task result settings
    result_expires=3600,  # Results expire after 1 hour
    result_backend_transport_options={'master_name': 'mymaster'},
    
    # Task routing
    task_routes={
        'tasks.send_email': {'queue': 'emails'},
        'tasks.process_data': {'queue': 'processing'},
    },
    
    # Worker settings
    worker_prefetch_multiplier=4,
    worker_max_tasks_per_child=1000,
    
    # Task retry settings
    task_acks_late=True,
    task_reject_on_worker_lost=True,
)

# Define queues
app.conf.task_queues = (
    Queue('default', Exchange('default'), routing_key='default'),
    Queue('emails', Exchange('emails'), routing_key='email.#'),
    Queue('processing', Exchange('processing'), routing_key='process.#'),
)
```

#### 3. Task Definitions (`tasks.py`)

```python
from celery import shared_task
from celery.utils.log import get_task_logger
import time
import requests

logger = get_task_logger(__name__)

# Basic task
@shared_task
def add(x, y):
    """Simple addition task"""
    return x + y

# Email sending task
@shared_task(bind=True, max_retries=3)
def send_email(self, to_email, subject, body):
    """
    Send email with retry logic
    
    Args:
        to_email: Recipient email address
        subject: Email subject
        body: Email body
    """
    try:
        logger.info(f"Sending email to {to_email}")
        
        # Simulate email sending
        # In production, use actual email service
        time.sleep(2)
        
        # Example with an email service
        # send_mail(subject, body, 'from@example.com', [to_email])
        
        logger.info(f"Email sent successfully to {to_email}")
        return {'status': 'success', 'email': to_email}
        
    except Exception as exc:
        logger.error(f"Error sending email: {exc}")
        # Retry with exponential backoff
        raise self.retry(exc=exc, countdown=60 * (2 ** self.request.retries))

# Data processing task
@shared_task(bind=True, time_limit=300, soft_time_limit=280)
def process_large_dataset(self, dataset_id):
    """
    Process large dataset with time limits
    
    Args:
        dataset_id: ID of dataset to process
    """
    try:
        logger.info(f"Processing dataset {dataset_id}")
        
        # Simulate data processing
        for i in range(100):
            time.sleep(0.1)
            # Update task progress
            self.update_state(
                state='PROGRESS',
                meta={'current': i, 'total': 100}
            )
        
        logger.info(f"Dataset {dataset_id} processed successfully")
        return {'status': 'complete', 'dataset_id': dataset_id}
        
    except Exception as exc:
        logger.error(f"Error processing dataset: {exc}")
        raise

# External API task
@shared_task(bind=True, max_retries=5, default_retry_delay=30)
def fetch_external_data(self, url):
    """
    Fetch data from external API with retry logic
    
    Args:
        url: API endpoint URL
    """
    try:
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        return response.json()
        
    except requests.RequestException as exc:
        logger.error(f"API request failed: {exc}")
        raise self.retry(exc=exc)

# Scheduled task (requires celery beat)
@shared_task
def cleanup_old_data():
    """Daily cleanup of old data"""
    logger.info("Running daily cleanup")
    # Implement cleanup logic
    deleted_count = 0
    # ... cleanup code ...
    logger.info(f"Cleanup complete. Deleted {deleted_count} records")
    return deleted_count

# Chain of tasks
@shared_task
def process_and_notify(data_id):
    """Process data and send notification"""
    from celery import chain
    
    # Create task chain
    result = chain(
        process_large_dataset.s(data_id),
        send_email.s('admin@example.com', 'Processing Complete', 'Data processed')
    )()
    
    return result

# Group tasks (parallel execution)
@shared_task
def batch_process_users(user_ids):
    """Process multiple users in parallel"""
    from celery import group
    
    job = group(send_email.s(f'user{uid}@example.com', 'Hello', 'Message') 
                for uid in user_ids)
    result = job.apply_async()
    return result.id
```

#### 4. Using Tasks in Your Application (`app.py`)

```python
from flask import Flask, jsonify, request
from tasks import send_email, process_large_dataset, add
from celery.result import AsyncResult
from celery_app import app as celery_app

app = Flask(__name__)

@app.route('/send-email', methods=['POST'])
def send_email_endpoint():
    """Queue email sending task"""
    data = request.json
    
    # Queue the task
    task = send_email.delay(
        data['email'],
        data['subject'],
        data['body']
    )
    
    return jsonify({
        'task_id': task.id,
        'status': 'queued'
    }), 202

@app.route('/process-data/<int:dataset_id>', methods=['POST'])
def process_data_endpoint(dataset_id):
    """Queue data processing task"""
    task = process_large_dataset.delay(dataset_id)
    
    return jsonify({
        'task_id': task.id,
        'status': 'processing'
    }), 202

@app.route('/task-status/<task_id>')
def task_status(task_id):
    """Check task status"""
    task = AsyncResult(task_id, app=celery_app)
    
    if task.state == 'PENDING':
        response = {
            'state': task.state,
            'status': 'Task is waiting...'
        }
    elif task.state == 'PROGRESS':
        response = {
            'state': task.state,
            'current': task.info.get('current', 0),
            'total': task.info.get('total', 100),
            'status': 'Processing...'
        }
    elif task.state == 'SUCCESS':
        response = {
            'state': task.state,
            'result': task.result,
            'status': 'Task completed!'
        }
    else:
        # Something went wrong
        response = {
            'state': task.state,
            'status': str(task.info),  # Exception raised
        }
    
    return jsonify(response)

if __name__ == '__main__':
    app.run(debug=True)
```

### Periodic Tasks with Celery Beat

Celery Beat is a scheduler that runs periodic tasks.

#### Configuration

```python
# celery_app.py
from celery.schedules import crontab

app.conf.beat_schedule = {
    # Execute every day at midnight
    'cleanup-daily': {
        'task': 'tasks.cleanup_old_data',
        'schedule': crontab(hour=0, minute=0),
    },
    # Execute every 15 minutes
    'sync-data': {
        'task': 'tasks.sync_external_data',
        'schedule': 900.0,  # seconds
    },
    # Execute every Monday at 7:30 AM
    'weekly-report': {
        'task': 'tasks.generate_weekly_report',
        'schedule': crontab(hour=7, minute=30, day_of_week=1),
    },
}
```

### Running Celery

```bash
# Start Celery worker
celery -A celery_app worker --loglevel=info

# Start worker with specific queue
celery -A celery_app worker -Q emails --loglevel=info

# Start multiple workers
celery -A celery_app worker --concurrency=4 --loglevel=info

# Start Celery Beat scheduler
celery -A celery_app beat --loglevel=info

# Start worker and beat together
celery -A celery_app worker --beat --loglevel=info

# Monitor with Flower (web-based monitoring tool)
pip install flower
celery -A celery_app flower
```

### Advanced Celery Features

#### Task Callbacks and Error Handling

```python
from celery import Task

class CallbackTask(Task):
    """Custom task class with callbacks"""
    
    def on_success(self, retval, task_id, args, kwargs):
        """Called on successful task execution"""
        print(f'Task {task_id} succeeded with result: {retval}')
    
    def on_failure(self, exc, task_id, args, kwargs, einfo):
        """Called on task failure"""
        print(f'Task {task_id} failed: {exc}')
    
    def on_retry(self, exc, task_id, args, kwargs, einfo):
        """Called on task retry"""
        print(f'Task {task_id} retrying due to: {exc}')

@shared_task(base=CallbackTask, bind=True)
def important_task(self, data):
    """Task with custom callbacks"""
    # Task implementation
    pass
```

#### Task Chaining and Workflows

```python
from celery import chain, group, chord

# Sequential execution (chain)
result = chain(
    add.s(2, 2),
    add.s(4),
    add.s(8)
)()
# Result: ((2 + 2) + 4) + 8 = 16

# Parallel execution (group)
job = group([
    add.s(2, 2),
    add.s(4, 4),
    add.s(8, 8)
])
result = job.apply_async()

# Chord (parallel then callback)
callback = send_email.s('admin@example.com', 'Done', 'All tasks complete')
header = [process_large_dataset.s(i) for i in range(5)]
result = chord(header)(callback)
```

---

## RQ (Redis Queue) - Simple Task Queue

RQ is a simple, lightweight Python task queue backed by Redis. It's easier to set up than Celery but has fewer features.

### Installation

```bash
pip install rq redis
```

### Basic Setup

#### 1. Project Structure

```
project/
├── worker.py          # Worker configuration
├── tasks.py           # Task definitions
└── app.py             # Main application
```

#### 2. Task Definitions (`tasks.py`)

```python
import time
import requests
from rq import get_current_job

def send_email(to_email, subject, body):
    """
    Send email task
    
    Args:
        to_email: Recipient email
        subject: Email subject
        body: Email body
    """
    print(f"Sending email to {to_email}")
    time.sleep(2)  # Simulate email sending
    print(f"Email sent to {to_email}")
    return {'status': 'sent', 'email': to_email}

def process_data(dataset_id):
    """
    Process dataset with progress tracking
    
    Args:
        dataset_id: ID of dataset to process
    """
    job = get_current_job()
    
    print(f"Processing dataset {dataset_id}")
    
    for i in range(100):
        time.sleep(0.1)
        # Update job metadata
        job.meta['progress'] = i
        job.save_meta()
    
    job.meta['progress'] = 100
    job.save_meta()
    
    return {'status': 'complete', 'dataset_id': dataset_id}

def fetch_api_data(url):
    """
    Fetch data from external API
    
    Args:
        url: API endpoint
    """
    response = requests.get(url, timeout=10)
    response.raise_for_status()
    return response.json()

def long_running_task(duration):
    """
    Simulate long-running task
    
    Args:
        duration: Task duration in seconds
    """
    job = get_current_job()
    
    for i in range(duration):
        time.sleep(1)
        job.meta['seconds_elapsed'] = i + 1
        job.save_meta()
    
    return f"Task completed in {duration} seconds"
```

#### 3. Using RQ in Your Application (`app.py`)

```python
from flask import Flask, jsonify, request
from redis import Redis
from rq import Queue
from rq.job import Job
import tasks

app = Flask(__name__)

# Connect to Redis
redis_conn = Redis(host='localhost', port=6379, db=0)

# Create queues
default_queue = Queue('default', connection=redis_conn)
email_queue = Queue('emails', connection=redis_conn)
processing_queue = Queue('processing', connection=redis_conn)

@app.route('/send-email', methods=['POST'])
def send_email_endpoint():
    """Queue email sending task"""
    data = request.json
    
    # Enqueue the task
    job = email_queue.enqueue(
        tasks.send_email,
        data['email'],
        data['subject'],
        data['body'],
        job_timeout='5m',  # Task timeout
        result_ttl=3600,   # Result stored for 1 hour
        failure_ttl=86400  # Failed job info stored for 24 hours
    )
    
    return jsonify({
        'job_id': job.id,
        'status': 'queued'
    }), 202

@app.route('/process-data/<int:dataset_id>', methods=['POST'])
def process_data_endpoint(dataset_id):
    """Queue data processing task"""
    job = processing_queue.enqueue(
        tasks.process_data,
        dataset_id,
        job_timeout='10m'
    )
    
    return jsonify({
        'job_id': job.id,
        'status': 'queued'
    }), 202

@app.route('/job-status/<job_id>')
def job_status(job_id):
    """Check job status"""
    try:
        job = Job.fetch(job_id, connection=redis_conn)
        
        response = {
            'job_id': job.id,
            'status': job.get_status(),
            'created_at': job.created_at.isoformat() if job.created_at else None,
            'started_at': job.started_at.isoformat() if job.started_at else None,
            'ended_at': job.ended_at.isoformat() if job.ended_at else None,
        }
        
        # Add progress if available
        if job.meta:
            response['meta'] = job.meta
        
        # Add result if finished
        if job.is_finished:
            response['result'] = job.result
        
        # Add error if failed
        if job.is_failed:
            response['error'] = str(job.exc_info)
        
        return jsonify(response)
        
    except Exception as e:
        return jsonify({'error': str(e)}), 404

@app.route('/cancel-job/<job_id>', methods=['POST'])
def cancel_job(job_id):
    """Cancel a queued job"""
    try:
        job = Job.fetch(job_id, connection=redis_conn)
        job.cancel()
        return jsonify({'status': 'cancelled', 'job_id': job_id})
    except Exception as e:
        return jsonify({'error': str(e)}), 404

if __name__ == '__main__':
    app.run(debug=True)
```

#### 4. Worker Configuration (`worker.py`)

```python
from redis import Redis
from rq import Worker, Queue, Connection

# Connect to Redis
redis_conn = Redis(host='localhost', port=6379, db=0)

if __name__ == '__main__':
    with Connection(redis_conn):
        # Create worker for specific queues
        queues = [Queue('emails'), Queue('processing'), Queue('default')]
        worker = Worker(queues)
        worker.work()
```

### Running RQ Workers

```bash
# Start a worker for all queues
rq worker default emails processing

# Start worker with specific configuration
rq worker --url redis://localhost:6379 default

# Start multiple workers
rq worker high &
rq worker default &
rq worker low &

# Monitor with RQ Dashboard
pip install rq-dashboard
rq-dashboard
```

### Advanced RQ Features

#### Job Dependencies

```python
# Create dependent jobs
job1 = queue.enqueue(tasks.process_data, 1)
job2 = queue.enqueue(tasks.send_email, 'user@example.com', 'Done', 'Processing complete',
                     depends_on=job1)
```

#### Scheduled Jobs

```python
from datetime import datetime, timedelta
from rq.scheduler import Scheduler

scheduler = Scheduler(connection=redis_conn)

# Schedule job to run in 10 minutes
scheduler.enqueue_in(
    timedelta(minutes=10),
    tasks.send_email,
    'user@example.com',
    'Reminder',
    'This is your reminder'
)

# Schedule job at specific time
scheduler.enqueue_at(
    datetime(2024, 1, 1, 0, 0),
    tasks.cleanup_old_data
)

# Run scheduler
# rq scheduler
```

#### Custom Job Classes

```python
from rq import Queue
from rq.job import Job

class CustomJob(Job):
    """Custom job with additional functionality"""
    
    @classmethod
    def create(cls, func, args=None, kwargs=None, **options):
        """Override create to add custom behavior"""
        job = super().create(func, args, kwargs, **options)
        job.meta['custom_field'] = 'custom_value'
        job.save()
        return job

# Use custom job class
queue = Queue('default', connection=redis_conn, job_class=CustomJob)
job = queue.enqueue(tasks.send_email, 'user@example.com', 'Test', 'Body')
```

---

## Comparison: Celery vs RQ

| Feature | Celery | RQ |
|---------|--------|-----|
| **Complexity** | High (more features) | Low (simple, minimal) |
| **Broker Support** | Redis, RabbitMQ, Amazon SQS, etc. | Redis only |
| **Result Backend** | Multiple options | Redis only |
| **Scheduling** | Built-in (Celery Beat) | External (RQ Scheduler) |
| **Task Routing** | Advanced routing, priorities | Basic queues |
| **Monitoring** | Flower, built-in events | RQ Dashboard |
| **Learning Curve** | Steeper | Gentle |
| **Performance** | Higher overhead, more powerful | Lightweight, faster for simple tasks |
| **Task Chaining** | Built-in (chain, group, chord) | Manual implementation |
| **Best For** | Complex workflows, enterprise apps | Simple tasks, small to medium apps |

### When to Use Celery

- Complex task workflows (chains, chords, groups)
- Need for advanced routing and priorities
- Multiple broker/backend options required
- Large-scale applications
- Built-in periodic task scheduling needed
- Need extensive monitoring and management

### When to Use RQ

- Simple background task processing
- Already using Redis
- Prefer simplicity over features
- Smaller applications
- Easy to understand codebase
- Quick setup required

---

## Best Practices

### 1. Task Design

```python
# ✅ GOOD: Idempotent task (can be run multiple times safely)
@shared_task
def update_user_score(user_id, points):
    user = User.objects.get(id=user_id)
    user.score = points  # Set absolute value
    user.save()

# ❌ BAD: Not idempotent (running twice gives wrong result)
@shared_task
def increment_user_score(user_id, points):
    user = User.objects.get(id=user_id)
    user.score += points  # Increment - not safe to retry
    user.save()
```

### 2. Timeout Configuration

```python
# Set appropriate timeouts
@shared_task(time_limit=300, soft_time_limit=280)
def process_with_timeout(data):
    """Task with 5-minute hard limit, 4:40 soft limit"""
    try:
        # Processing logic
        pass
    except SoftTimeLimitExceeded:
        # Cleanup before hard timeout
        cleanup()
        raise
```

### 3. Error Handling and Retries

```python
@shared_task(
    bind=True,
    autoretry_for=(IOError, RequestException),
    retry_kwargs={'max_retries': 5},
    retry_backoff=True,  # Exponential backoff
    retry_backoff_max=600,  # Max 10 minutes between retries
    retry_jitter=True  # Add randomness to prevent thundering herd
)
def robust_task(self, data):
    """Task with automatic retry on specific exceptions"""
    # Task implementation
    pass
```

### 4. Logging

```python
from celery.utils.log import get_task_logger

logger = get_task_logger(__name__)

@shared_task
def well_logged_task(data):
    logger.info(f"Starting task with data: {data}")
    try:
        # Process
        result = process(data)
        logger.info(f"Task completed successfully: {result}")
        return result
    except Exception as e:
        logger.error(f"Task failed: {e}", exc_info=True)
        raise
```

### 5. Task Granularity

```python
# ✅ GOOD: Small, focused tasks
@shared_task
def process_user(user_id):
    """Process single user"""
    pass

@shared_task
def process_all_users():
    """Coordinate processing all users"""
    user_ids = get_all_user_ids()
    for user_id in user_ids:
        process_user.delay(user_id)

# ❌ BAD: One giant task
@shared_task
def process_everything():
    """Process all users in one task (hard to monitor, retry, scale)"""
    for user in get_all_users():
        # Process each user
        pass
```

### 6. Resource Management

```python
@shared_task
def resource_aware_task(file_path):
    """Properly manage resources"""
    with open(file_path, 'r') as f:
        data = f.read()
    
    try:
        result = process(data)
        return result
    finally:
        # Cleanup
        if os.path.exists(file_path):
            os.remove(file_path)
```

### 7. Task Monitoring

```python
@shared_task(bind=True)
def monitorable_task(self, items):
    """Task with progress reporting"""
    total = len(items)
    
    for i, item in enumerate(items):
        # Process item
        process_item(item)
        
        # Update progress
        self.update_state(
            state='PROGRESS',
            meta={
                'current': i + 1,
                'total': total,
                'percentage': int((i + 1) / total * 100)
            }
        )
    
    return {'processed': total}
```

---

## Production Considerations

### 1. Infrastructure Setup

#### Docker Compose Example

```yaml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  celery_worker:
    build: .
    command: celery -A celery_app worker --loglevel=info
    volumes:
      - .:/app
    depends_on:
      - redis
    environment:
      - CELERY_BROKER_URL=redis://redis:6379/0
      - CELERY_RESULT_BACKEND=redis://redis:6379/1

  celery_beat:
    build: .
    command: celery -A celery_app beat --loglevel=info
    volumes:
      - .:/app
    depends_on:
      - redis
    environment:
      - CELERY_BROKER_URL=redis://redis:6379/0

  flower:
    build: .
    command: celery -A celery_app flower
    ports:
      - "5555:5555"
    depends_on:
      - redis
      - celery_worker
    environment:
      - CELERY_BROKER_URL=redis://redis:6379/0

volumes:
  redis_data:
```

### 2. Monitoring

```python
# Celery configuration for monitoring
app.conf.update(
    # Enable events for monitoring
    worker_send_task_events=True,
    task_send_sent_event=True,
    
    # Store task history
    result_extended=True,
)

# Custom monitoring task
@shared_task
def monitor_queue_length():
    """Monitor queue lengths"""
    from celery_app import app
    
    inspect = app.control.inspect()
    stats = inspect.stats()
    
    if stats:
        for worker, info in stats.items():
            queue_length = info.get('total', {})
            # Log or alert on queue length
            if queue_length > 1000:
                alert_admin(f"Queue too long: {queue_length}")
```

### 3. Scaling Workers

```bash
# Horizontal scaling - multiple worker instances
docker-compose up --scale celery_worker=5

# Vertical scaling - more concurrent tasks per worker
celery -A celery_app worker --concurrency=8

# Auto-scaling
celery -A celery_app worker --autoscale=10,3  # Max 10, min 3
```

### 4. Security

```python
# Secure Celery configuration
app.conf.update(
    # Use secure serialization
    task_serializer='json',
    accept_content=['json'],  # Don't accept pickle
    
    # Enable message signing
    task_protocol=2,
    
    # Rate limiting
    task_default_rate_limit='100/m',
    
    # Worker pool settings
    worker_pool_restarts=True,
)

# Validate task arguments
@shared_task
def secure_task(user_id, action):
    # Validate inputs
    if not isinstance(user_id, int) or user_id < 1:
        raise ValueError("Invalid user_id")
    
    if action not in ['create', 'update', 'delete']:
        raise ValueError("Invalid action")
    
    # Process task
    pass
```

### 5. Database Connections

```python
# Properly manage database connections
from django.db import connection

@shared_task
def database_task():
    try:
        # Perform database operations
        result = perform_query()
        return result
    finally:
        # Close connection to prevent leaks
        connection.close()
```

### 6. High Availability

```python
# Redis Sentinel configuration for HA
from kombu import Exchange, Queue

app.conf.update(
    broker_url='sentinel://localhost:26379;sentinel://localhost:26380',
    broker_transport_options={
        'master_name': 'mymaster',
        'sentinel_kwargs': {
            'password': 'sentinel-password'
        }
    },
)
```

### 7. Performance Optimization

```python
# Batch processing for efficiency
@shared_task
def batch_process(batch_size=100):
    """Process items in batches"""
    items = get_pending_items()
    
    for i in range(0, len(items), batch_size):
        batch = items[i:i + batch_size]
        process_batch(batch)

# Use task compression for large payloads
@shared_task(compression='gzip')
def task_with_large_data(large_data):
    """Task with compressed data transfer"""
    pass
```

---

## Conclusion

Background tasks and job queues are essential for building scalable, responsive applications. Choose Celery for complex, enterprise-level applications requiring advanced features, and RQ for simpler use cases where ease of use is paramount. Always design tasks to be idempotent, implement proper error handling, monitor your queues, and scale appropriately for production workloads.

### Quick Decision Guide

**Choose Celery if:**
- You need complex task workflows
- Require advanced scheduling
- Building enterprise applications
- Need multiple broker options
- Want extensive monitoring capabilities

**Choose RQ if:**
- You want simplicity
- Already using Redis
- Building small to medium applications
- Need quick setup
- Prefer minimal dependencies

Both tools are production-ready and widely used. Your choice should depend on your specific requirements, team expertise, and application complexity.