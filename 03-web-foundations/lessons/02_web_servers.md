# Web Servers with Python

## Table of Contents
1. [Introduction to Web Servers](#introduction-to-web-servers)
2. [How Web Servers Work](#how-web-servers-work)
3. [Python's Built-in HTTP Server](#pythons-built-in-http-server)
4. [WSGI - Web Server Gateway Interface](#wsgi---web-server-gateway-interface)
5. [ASGI - Asynchronous Server Gateway Interface](#asgi---asynchronous-server-gateway-interface)
6. [Production Web Servers](#production-web-servers)
7. [Building a Custom Web Server](#building-a-custom-web-server)
8. [Performance and Optimization](#performance-and-optimization)
9. [Deployment Strategies](#deployment-strategies)

---

## Introduction to Web Servers

A **web server** is software that accepts HTTP requests from clients (browsers, apps) and serves HTTP responses (HTML pages, JSON data, files). Web servers are the foundation of web applications.

### Types of Web Servers

1. **Static Web Servers**: Serve files as-is (Apache, Nginx)
2. **Application Servers**: Execute code to generate dynamic content (Gunicorn, uWSGI)
3. **Development Servers**: For local testing (Flask dev server, Django runserver)

### Web Server Responsibilities

- Accept incoming HTTP connections
- Parse HTTP requests
- Route requests to appropriate handlers
- Generate HTTP responses
- Manage concurrent connections
- Handle errors and timeouts
- Log requests and errors
- Serve static files
- Support SSL/TLS (HTTPS)

---

## How Web Servers Work

### Basic Web Server Flow

```
1. Client sends HTTP request
   ↓
2. Server accepts TCP connection
   ↓
3. Server reads and parses HTTP request
   ↓
4. Server routes to appropriate handler
   ↓
5. Application processes request
   ↓
6. Application generates response
   ↓
7. Server sends HTTP response
   ↓
8. Connection closed (or kept alive)
```

### TCP/IP Socket Communication

Web servers use TCP sockets to communicate:

```python
import socket

# Create a TCP socket
server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Bind to address and port
server_socket.bind(('localhost', 8000))

# Listen for connections (backlog of 5)
server_socket.listen(5)

print("Server listening on port 8000...")

while True:
    # Accept incoming connection
    client_socket, address = server_socket.accept()
    print(f"Connection from {address}")
    
    # Receive data
    request = client_socket.recv(1024).decode('utf-8')
    print(request)
    
    # Send response
    response = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n<h1>Hello World!</h1>"
    client_socket.sendall(response.encode('utf-8'))
    
    # Close connection
    client_socket.close()
```

---

## Python's Built-in HTTP Server

Python includes `http.server` for simple HTTP serving tasks.

### Simple File Server

```python
# Command line usage
# python -m http.server 8000

# Serve files from specific directory
# python -m http.server 8000 --directory /path/to/files

# Bind to specific address
# python -m http.server 8000 --bind 127.0.0.1
```

### Custom HTTP Server with http.server

```python
from http.server import HTTPServer, BaseHTTPRequestHandler
import json

class MyHandler(BaseHTTPRequestHandler):
    
    def do_GET(self):
        """Handle GET requests"""
        if self.path == '/':
            self.send_response(200)
            self.send_header('Content-type', 'text/html')
            self.end_headers()
            html = """
            <html>
                <head><title>My Server</title></head>
                <body>
                    <h1>Welcome to My Server</h1>
                    <p><a href="/api/data">Get JSON Data</a></p>
                </body>
            </html>
            """
            self.wfile.write(html.encode())
        
        elif self.path == '/api/data':
            self.send_response(200)
            self.send_header('Content-type', 'application/json')
            self.end_headers()
            data = {
                'message': 'Hello from API',
                'status': 'success',
                'items': [1, 2, 3, 4, 5]
            }
            self.wfile.write(json.dumps(data).encode())
        
        else:
            self.send_error(404, "Page not found")
    
    def do_POST(self):
        """Handle POST requests"""
        content_length = int(self.headers['Content-Length'])
        post_data = self.rfile.read(content_length)
        
        self.send_response(200)
        self.send_header('Content-type', 'application/json')
        self.end_headers()
        
        response = {
            'received': post_data.decode('utf-8'),
            'method': 'POST'
        }
        self.wfile.write(json.dumps(response).encode())
    
    def log_message(self, format, *args):
        """Custom logging"""
        print(f"[{self.log_date_time_string()}] {format % args}")

# Run the server
def run(server_class=HTTPServer, handler_class=MyHandler, port=8000):
    server_address = ('', port)
    httpd = server_class(server_address, handler_class)
    print(f"Server running on port {port}...")
    httpd.serve_forever()

if __name__ == '__main__':
    run()
```

### Serving Static Files

```python
from http.server import HTTPServer, SimpleHTTPRequestHandler
import os

class CustomFileHandler(SimpleHTTPRequestHandler):
    def __init__(self, *args, **kwargs):
        # Set the directory to serve files from
        super().__init__(*args, directory='./public', **kwargs)
    
    def end_headers(self):
        # Add custom headers
        self.send_header('Cache-Control', 'no-store, no-cache, must-revalidate')
        self.send_header('X-Custom-Header', 'MyValue')
        super().end_headers()

# Create public directory structure
os.makedirs('public', exist_ok=True)
with open('public/index.html', 'w') as f:
    f.write('<html><body><h1>Static File Server</h1></body></html>')

# Run server
httpd = HTTPServer(('localhost', 8000), CustomFileHandler)
print("Serving files from ./public on port 8000")
httpd.serve_forever()
```

### Threading for Concurrent Requests

```python
from http.server import HTTPServer, BaseHTTPRequestHandler
from socketserver import ThreadingMixIn
import time

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        # Simulate slow processing
        time.sleep(2)
        
        self.send_response(200)
        self.send_header('Content-type', 'text/html')
        self.end_headers()
        self.wfile.write(b"<h1>Response after 2 seconds</h1>")

class ThreadedHTTPServer(ThreadingMixIn, HTTPServer):
    """Handle requests in separate threads"""
    pass

# This server can handle multiple requests concurrently
server = ThreadedHTTPServer(('localhost', 8000), Handler)
print("Threaded server running on port 8000")
server.serve_forever()
```

---

## WSGI - Web Server Gateway Interface

**WSGI** (PEP 3333) is a specification that defines how web servers communicate with Python web applications. It's the standard interface between web servers and Python web frameworks.

### WSGI Architecture

```
Web Server (Gunicorn, uWSGI) 
         ↓
    WSGI Interface
         ↓
Web Framework (Flask, Django)
         ↓
    Your Application
```

### Simple WSGI Application

```python
def application(environ, start_response):
    """
    Minimal WSGI application
    
    Args:
        environ: Dictionary containing CGI-like environment variables
        start_response: Callable to begin the HTTP response
    
    Returns:
        Iterable of bytes representing the response body
    """
    # HTTP status
    status = '200 OK'
    
    # Response headers
    headers = [
        ('Content-Type', 'text/html'),
        ('X-Custom-Header', 'WSGI-App')
    ]
    
    # Call start_response
    start_response(status, headers)
    
    # Return response body as iterable of bytes
    return [b'<h1>Hello from WSGI!</h1>']

# Run with wsgiref (built-in WSGI server)
if __name__ == '__main__':
    from wsgiref.simple_server import make_server
    
    server = make_server('localhost', 8000, application)
    print("WSGI server running on port 8000...")
    server.serve_forever()
```

### Understanding the environ Dictionary

```python
def application(environ, start_response):
    """Display all WSGI environment variables"""
    
    # Important environ keys:
    request_method = environ['REQUEST_METHOD']  # GET, POST, etc.
    path_info = environ['PATH_INFO']            # URL path
    query_string = environ['QUERY_STRING']      # Query parameters
    content_type = environ.get('CONTENT_TYPE', '')
    content_length = environ.get('CONTENT_LENGTH', 0)
    server_name = environ['SERVER_NAME']
    server_port = environ['SERVER_PORT']
    
    # Read POST data
    if request_method == 'POST':
        try:
            request_body_size = int(content_length)
            request_body = environ['wsgi.input'].read(request_body_size)
        except (ValueError, KeyError):
            request_body = b''
    
    # Build response
    status = '200 OK'
    headers = [('Content-Type', 'text/html')]
    start_response(status, headers)
    
    html = f"""
    <html>
    <body>
        <h1>WSGI Environment</h1>
        <p>Method: {request_method}</p>
        <p>Path: {path_info}</p>
        <p>Query: {query_string}</p>
        <h2>All Variables:</h2>
        <pre>
    """
    
    for key, value in sorted(environ.items()):
        html += f"{key}: {value}\n"
    
    html += """
        </pre>
    </body>
    </html>
    """
    
    return [html.encode('utf-8')]

if __name__ == '__main__':
    from wsgiref.simple_server import make_server
    server = make_server('localhost', 8000, application)
    server.serve_forever()
```

### Building a Router with WSGI

```python
import json
from urllib.parse import parse_qs

class WSGIRouter:
    def __init__(self):
        self.routes = {}
    
    def route(self, path, methods=['GET']):
        """Decorator to register routes"""
        def decorator(func):
            for method in methods:
                key = f"{method} {path}"
                self.routes[key] = func
            return func
        return decorator
    
    def __call__(self, environ, start_response):
        """WSGI application callable"""
        method = environ['REQUEST_METHOD']
        path = environ['PATH_INFO']
        key = f"{method} {path}"
        
        # Find matching route
        handler = self.routes.get(key)
        
        if handler:
            return handler(environ, start_response)
        else:
            # 404 Not Found
            status = '404 NOT FOUND'
            headers = [('Content-Type', 'text/html')]
            start_response(status, headers)
            return [b'<h1>404 - Page Not Found</h1>']

# Create router
app = WSGIRouter()

@app.route('/')
def index(environ, start_response):
    status = '200 OK'
    headers = [('Content-Type', 'text/html')]
    start_response(status, headers)
    return [b'<h1>Home Page</h1><p><a href="/api/users">Users API</a></p>']

@app.route('/api/users')
def users(environ, start_response):
    status = '200 OK'
    headers = [('Content-Type', 'application/json')]
    start_response(status, headers)
    
    data = {
        'users': [
            {'id': 1, 'name': 'Alice'},
            {'id': 2, 'name': 'Bob'}
        ]
    }
    return [json.dumps(data).encode('utf-8')]

@app.route('/api/users', methods=['POST'])
def create_user(environ, start_response):
    # Read POST data
    try:
        content_length = int(environ.get('CONTENT_LENGTH', 0))
        post_data = environ['wsgi.input'].read(content_length)
        data = json.loads(post_data.decode('utf-8'))
    except (ValueError, KeyError):
        status = '400 BAD REQUEST'
        headers = [('Content-Type', 'application/json')]
        start_response(status, headers)
        return [json.dumps({'error': 'Invalid JSON'}).encode('utf-8')]
    
    status = '201 CREATED'
    headers = [('Content-Type', 'application/json')]
    start_response(status, headers)
    
    response = {'message': 'User created', 'data': data}
    return [json.dumps(response).encode('utf-8')]

# Run the app
if __name__ == '__main__':
    from wsgiref.simple_server import make_server
    server = make_server('localhost', 8000, app)
    print("WSGI Router running on port 8000...")
    server.serve_forever()
```

### WSGI Middleware

Middleware sits between the server and application, modifying requests/responses:

```python
class LoggingMiddleware:
    """Middleware to log all requests"""
    
    def __init__(self, app):
        self.app = app
    
    def __call__(self, environ, start_response):
        # Log request
        print(f"{environ['REQUEST_METHOD']} {environ['PATH_INFO']}")
        
        # Call the wrapped application
        return self.app(environ, start_response)

class AuthMiddleware:
    """Middleware to check authentication"""
    
    def __init__(self, app):
        self.app = app
    
    def __call__(self, environ, start_response):
        # Check for auth token
        auth_header = environ.get('HTTP_AUTHORIZATION', '')
        
        if not auth_header.startswith('Bearer '):
            status = '401 UNAUTHORIZED'
            headers = [('Content-Type', 'text/html')]
            start_response(status, headers)
            return [b'<h1>401 - Unauthorized</h1>']
        
        # Token is valid, proceed
        return self.app(environ, start_response)

class CORSMiddleware:
    """Middleware to add CORS headers"""
    
    def __init__(self, app):
        self.app = app
    
    def __call__(self, environ, start_response):
        def custom_start_response(status, headers):
            # Add CORS headers
            headers.append(('Access-Control-Allow-Origin', '*'))
            headers.append(('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE'))
            headers.append(('Access-Control-Allow-Headers', 'Content-Type'))
            return start_response(status, headers)
        
        return self.app(environ, custom_start_response)

# Simple WSGI app
def simple_app(environ, start_response):
    status = '200 OK'
    headers = [('Content-Type', 'text/plain')]
    start_response(status, headers)
    return [b'Hello World']

# Wrap app with middleware (order matters!)
app = simple_app
app = CORSMiddleware(app)
app = LoggingMiddleware(app)
# app = AuthMiddleware(app)  # Uncomment to require auth

# Run
if __name__ == '__main__':
    from wsgiref.simple_server import make_server
    server = make_server('localhost', 8000, app)
    print("Middleware demo running on port 8000...")
    server.serve_forever()
```

---

## ASGI - Asynchronous Server Gateway Interface

**ASGI** is the spiritual successor to WSGI, designed for async Python web apps. It supports WebSockets, HTTP/2, and long-lived connections.

### Simple ASGI Application

```python
async def application(scope, receive, send):
    """
    Simple ASGI application
    
    Args:
        scope: Dictionary containing connection info
        receive: Async callable to receive events
        send: Async callable to send events
    """
    if scope['type'] == 'http':
        await send({
            'type': 'http.response.start',
            'status': 200,
            'headers': [
                [b'content-type', b'text/html'],
            ],
        })
        await send({
            'type': 'http.response.body',
            'body': b'<h1>Hello from ASGI!</h1>',
        })

# Run with uvicorn:
# uvicorn filename:application --reload
```

### ASGI with WebSocket Support

```python
async def application(scope, receive, send):
    if scope['type'] == 'http':
        # Handle HTTP
        await send({
            'type': 'http.response.start',
            'status': 200,
            'headers': [[b'content-type', b'text/html']],
        })
        await send({
            'type': 'http.response.body',
            'body': b'<h1>HTTP Response</h1>',
        })
    
    elif scope['type'] == 'websocket':
        # Handle WebSocket
        await send({'type': 'websocket.accept'})
        
        while True:
            message = await receive()
            
            if message['type'] == 'websocket.disconnect':
                break
            
            if message['type'] == 'websocket.receive':
                text = message.get('text', '')
                # Echo message back
                await send({
                    'type': 'websocket.send',
                    'text': f'Echo: {text}'
                })

# Run with: uvicorn filename:application
```

---

## Production Web Servers

### Gunicorn (Green Unicorn)

A popular WSGI HTTP server for UNIX systems.

**Installation:**
```bash
pip install gunicorn
```

**Running a WSGI app:**
```bash
# Basic usage
gunicorn myapp:application

# Specify host and port
gunicorn myapp:application --bind 0.0.0.0:8000

# Multiple worker processes
gunicorn myapp:application --workers 4

# Worker class (sync, gthread, eventlet, gevent)
gunicorn myapp:application --workers 4 --worker-class gthread --threads 2

# Timeout for worker processes
gunicorn myapp:application --timeout 30

# Access logging
gunicorn myapp:application --access-logfile access.log

# Error logging
gunicorn myapp:application --error-logfile error.log --log-level info

# Daemon mode (background)
gunicorn myapp:application --daemon

# Configuration file
gunicorn myapp:application -c gunicorn_config.py
```

**gunicorn_config.py:**
```python
# Gunicorn configuration file

# Server Socket
bind = '0.0.0.0:8000'
backlog = 2048

# Worker Processes
workers = 4
worker_class = 'sync'  # or 'gthread', 'eventlet', 'gevent'
worker_connections = 1000
timeout = 30
keepalive = 2

# Logging
accesslog = 'access.log'
errorlog = 'error.log'
loglevel = 'info'

# Process Naming
proc_name = 'myapp'

# Server Mechanics
daemon = False
pidfile = 'gunicorn.pid'
user = None
group = None
tmp_upload_dir = None

# SSL
keyfile = None
certfile = None

# Worker Lifecycle Hooks
def on_starting(server):
    print("Server is starting")

def on_reload(server):
    print("Server is reloading")

def when_ready(server):
    print("Server is ready")

def on_exit(server):
    print("Server is exiting")
```

**Running Flask with Gunicorn:**
```python
# app.py
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return '<h1>Hello from Flask with Gunicorn!</h1>'

if __name__ == '__main__':
    app.run()

# Run with: gunicorn app:app --workers 4
```

### uWSGI

A full-featured application server for various languages, including Python.

**Installation:**
```bash
pip install uwsgi
```

**Running a WSGI app:**
```bash
# Basic usage
uwsgi --http :8000 --wsgi-file myapp.py

# With more workers
uwsgi --http :8000 --wsgi-file myapp.py --processes 4 --threads 2

# Master process (recommended for production)
uwsgi --http :8000 --wsgi-file myapp.py --master --processes 4

# Stats server
uwsgi --http :8000 --wsgi-file myapp.py --stats 127.0.0.1:9191

# Configuration file
uwsgi --ini uwsgi.ini
```

**uwsgi.ini:**
```ini
[uwsgi]
# Application
module = myapp:application
callable = app

# Process management
master = true
processes = 4
threads = 2

# Socket
http = :8000
# Or use socket for nginx
# socket = /tmp/myapp.sock
# chmod-socket = 666

# Performance
enable-threads = true
single-interpreter = true
lazy-apps = true

# Reload on changes
py-autoreload = 1

# Logging
logto = /var/log/uwsgi/myapp.log
log-maxsize = 50000000

# Limits
post-buffering = 8192
buffer-size = 32768

# Vacuum: Remove socket on shutdown
vacuum = true

# Die on term signal
die-on-term = true
```

### Uvicorn (for ASGI)

High-performance ASGI server built on uvloop and httptools.

**Installation:**
```bash
pip install uvicorn
```

**Running an ASGI app:**
```bash
# Basic usage
uvicorn myapp:application

# Specify host and port
uvicorn myapp:application --host 0.0.0.0 --port 8000

# Auto-reload on code changes
uvicorn myapp:application --reload

# Multiple workers
uvicorn myapp:application --workers 4

# Access log
uvicorn myapp:application --access-log

# SSL
uvicorn myapp:application --ssl-keyfile key.pem --ssl-certfile cert.pem

# Configuration
uvicorn myapp:application --port 8000 --workers 4 --log-level info
```

**Running FastAPI with Uvicorn:**
```python
# app.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello from FastAPI with Uvicorn"}

# Run with: uvicorn app:app --reload
```

### Comparison of Production Servers

| Server | Protocol | Async Support | Best For |
|--------|----------|---------------|----------|
| Gunicorn | WSGI | No | Traditional Python apps (Flask, Django) |
| uWSGI | WSGI | Limited | Enterprise deployments, complex configs |
| Uvicorn | ASGI | Yes | Async apps (FastAPI, Starlette) |
| Daphne | ASGI | Yes | Django Channels, WebSockets |
| Hypercorn | ASGI | Yes | HTTP/2, HTTP/3 support |

---

## Building a Custom Web Server

### Minimal HTTP Server from Scratch

```python
import socket
from datetime import datetime

class SimpleHTTPServer:
    def __init__(self, host='localhost', port=8000):
        self.host = host
        self.port = port
        self.socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    
    def start(self):
        """Start the server"""
        self.socket.bind((self.host, self.port))
        self.socket.listen(5)
        print(f"Server listening on {self.host}:{self.port}")
        
        while True:
            client_socket, address = self.socket.accept()
            print(f"Connection from {address}")
            self.handle_client(client_socket)
    
    def handle_client(self, client_socket):
        """Handle a client connection"""
        try:
            # Receive request
            request = client_socket.recv(1024).decode('utf-8')
            
            # Parse request
            request_line = request.split('\n')[0]
            method, path, version = request_line.split()
            
            print(f"{method} {path}")
            
            # Generate response
            if path == '/':
                response = self.build_response(200, '<h1>Home Page</h1>')
            elif path == '/about':
                response = self.build_response(200, '<h1>About Page</h1>')
            elif path == '/time':
                now = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
                response = self.build_response(200, f'<h1>Current Time</h1><p>{now}</p>')
            else:
                response = self.build_response(404, '<h1>404 - Not Found</h1>')
            
            # Send response
            client_socket.sendall(response.encode('utf-8'))
        
        except Exception as e:
            print(f"Error: {e}")
            response = self.build_response(500, '<h1>500 - Internal Server Error</h1>')
            client_socket.sendall(response.encode('utf-8'))
        
        finally:
            client_socket.close()
    
    def build_response(self, status_code, body):
        """Build an HTTP response"""
        status_messages = {
            200: 'OK',
            404: 'Not Found',
            500: 'Internal Server Error'
        }
        
        status_message = status_messages.get(status_code, 'Unknown')
        
        response = f"HTTP/1.1 {status_code} {status_message}\r\n"
        response += f"Content-Type: text/html\r\n"
        response += f"Content-Length: {len(body)}\r\n"
        response += f"Server: SimpleHTTPServer/1.0\r\n"
        response += f"Connection: close\r\n"
        response += f"\r\n"
        response += body
        
        return response

if __name__ == '__main__':
    server = SimpleHTTPServer()
    try:
        server.start()
    except KeyboardInterrupt:
        print("\nShutting down server...")
```

### Multithreaded Custom Server

```python
import socket
import threading
from datetime import datetime

class ThreadedHTTPServer:
    def __init__(self, host='localhost', port=8000, max_connections=10):
        self.host = host
        self.port = port
        self.max_connections = max_connections
        self.socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.active_threads = 0
    
    def start(self):
        """Start the server"""
        self.socket.bind((self.host, self.port))
        self.socket.listen(self.max_connections)
        print(f"Threaded server listening on {self.host}:{self.port}")
        
        while True:
            try:
                client_socket, address = self.socket.accept()
                
                # Create a new thread for each client
                client_thread = threading.Thread(
                    target=self.handle_client,
                    args=(client_socket, address)
                )
                client_thread.daemon = True
                client_thread.start()
                
                self.active_threads += 1
                print(f"Active threads: {self.active_threads}")
                
            except KeyboardInterrupt:
                print("\nShutting down server...")
                break
    
    def handle_client(self, client_socket, address):
        """Handle a client in a separate thread"""
        try:
            print(f"[Thread {threading.current_thread().name}] Connection from {address}")
            
            # Receive request
            request = client_socket.recv(4096).decode('utf-8')
            
            if not request:
                return
            
            # Parse request line
            lines = request.split('\r\n')
            request_line = lines[0]
            method, path, version = request_line.split()
            
            # Route to handler
            if path == '/':
                content = '<h1>Multithreaded Server</h1><p>Each request is handled in a separate thread.</p>'
                response = self.build_response(200, content)
            elif path == '/slow':
                # Simulate slow processing
                import time
                time.sleep(3)
                content = '<h1>Slow Response</h1><p>This took 3 seconds to process.</p>'
                response = self.build_response(200, content)
            else:
                response = self.build_response(404, '<h1>Not Found</h1>')
            
            client_socket.sendall(response.encode('utf-8'))
        
        except Exception as e:
            print(f"Error in thread: {e}")
        
        finally:
            client_socket.close()
            self.active_threads -= 1
    
    def build_response(self, status_code, body):
        """Build HTTP response"""
        status_text = {200: 'OK', 404: 'Not Found'}.get(status_code, 'Unknown')
        
        response = f"HTTP/1.1 {status_code} {status_text}\r\n"
        response += "Content-Type: text/html; charset=utf-8\r\n"
        response += f"Content-Length: {len(body.encode('utf-8'))}\r\n"
        response += "Connection: close\r\n"
        response += "\r\n"
        response += body
        
        return response

if __name__ == '__main__':
    server = ThreadedHTTPServer(port=8000, max_connections=20)
    server.start()
```

---

## Performance and Optimization

### Worker Process Models

**1. Synchronous Workers (default in Gunicorn)**
- One request per worker at a time
- Simple and reliable
- Good for CPU-bound tasks
- Limited concurrency

**2. Threaded Workers**
- Multiple threads per worker
- Better for I/O-bound tasks
- Shared memory between threads
- Watch for thread-safety issues

```bash
# Gunicorn with threads
gunicorn app:app --workers 2 --threads 4
```

**3. Async Workers (Gevent, Eventlet)**
- Event-driven, non-blocking I/O
- Excellent for many concurrent connections
- Best for I/O-bound workloads
- Requires async-compatible libraries

```bash
# Gunicorn with gevent
gunicorn app:app --worker-class gevent --workers 4 --worker-connections 1000
```

**4. ASGI Workers (Uvicorn)**
- Native async/await support
- WebSocket support
- Modern Python async ecosystem

### Calculating Optimal Workers

**Formula:** `(2 x CPU_CORES) + 1`

```python
import multiprocessing

# Recommended number of workers
workers = (multiprocessing.cpu_count() * 2) + 1
print(f"Recommended workers: {workers}")
```

### Load Balancing Strategies

**1. Round Robin** (default in Nginx)
```nginx
upstream myapp {
    server 127.0.0.1:8000;
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
}
```

**2. Least Connections**
```nginx
upstream myapp {
    least_conn;
    server 127.0.0.1:8000;
    server 127.0.0.1:8001;
}
```

**3. IP Hash** (sticky sessions)
```nginx
upstream myapp {
    ip_hash;
    server 127.0.0.1:8000;
    server 127.0.0.1:8001;
}
```

### Performance Best Practices

1. **Use a reverse proxy** (Nginx, HAProxy)
2. **Enable HTTP keep-alive** for connection reuse
3. **Implement caching** (Redis, Memcached)
4. **Serve static files** directly from Nginx
5. **Use CDN** for static assets
6. **Enable gzip compression**
7. **Monitor and profile** regularly
8. **Set appropriate timeouts**
9. **Use connection pooling** for databases
10. **Implement rate limiting**

---

## Deployment Strategies

### Nginx + Gunicorn Configuration

**nginx.conf:**
```nginx
upstream myapp_backend {
    # Load balancing across multiple Gunicorn workers
    server 127.0.0.1:8000 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:8001 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    server_name example.com;
    
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header X-XSS-Protection "1; mode=block";
    
    # Client max body size
    client_max_body_size 10M;
    
    # Serve static files directly
    location /static/ {
        alias /var/www/myapp/static/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
    
    location /media/ {
        alias /var/www/myapp/media/;
        expires 7d;
    }
    
    # Proxy to Gunicorn
    location / {
        proxy_pass http://myapp_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Buffering
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }
    
    # WebSocket support
    location /ws/ {
        proxy_pass http://myapp_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 86400;
    }
}
```

### Systemd Service Configuration

**myapp.service:**
```ini
[Unit]
Description=Gunicorn instance for MyApp
After=network.target

[Service]
Type=notify
User=www-data
Group=www-data
RuntimeDirectory=gunicorn
WorkingDirectory=/var/www/myapp
Environment="PATH=/var/www/myapp/venv/bin"
ExecStart=/var/www/myapp/venv/bin/gunicorn \
    --workers 4 \
    --bind 127.0.0.1:8000 \
    --timeout 60 \
    --access-logfile /var/log/myapp/access.log \
    --error-logfile /var/log/myapp/error.log \
    --log-level info \
    --pid /run/gunicorn/myapp.pid \
    app:application
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=mixed
TimeoutStopSec=5
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

**Commands:**
```bash
# Reload systemd
sudo systemctl daemon-reload

# Start service
sudo systemctl start myapp

# Enable on boot
sudo systemctl enable myapp

# Check status
sudo systemctl status myapp

# View logs
sudo journalctl -u myapp -f

# Restart
sudo systemctl restart myapp

# Stop
sudo systemctl stop myapp
```

### Docker Deployment

**Dockerfile:**
```dockerfile
FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Create non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

# Expose port
EXPOSE 8000

# Run Gunicorn
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "4", "app:application"]
```

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:password@db:5432/myapp
    volumes:
      - ./static:/app/static
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: postgres:15
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=myapp
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./static:/var/www/static
    depends_on:
      - web
    restart: unless-stopped

volumes:
  postgres_data:
```

### Zero-Downtime Deployments

**Using Gunicorn graceful reload:**
```bash
# Send HUP signal to reload workers gracefully
kill -HUP $(cat gunicorn.pid)

# Or using systemctl
sudo systemctl reload myapp
```

**Blue-Green Deployment Strategy:**
```bash
# Start new version on different port
gunicorn app:application --bind 127.0.0.1:8001

# Update Nginx upstream
# Switch traffic to new version
sudo systemctl reload nginx

# Stop old version
kill $(cat old_gunicorn.pid)
```

### Health Checks

```python
# healthcheck.py
from flask import Flask, jsonify
import psutil
import os

app = Flask(__name__)

@app.route('/health')
def health():
    """Basic health check"""
    return jsonify({
        'status': 'healthy',
        'service': 'myapp'
    }), 200

@app.route('/readiness')
def readiness():
    """Readiness probe - check if app is ready to serve traffic"""
    try:
        # Check database connection
        # Check Redis connection
        # Check any dependencies
        
        return jsonify({
            'status': 'ready',
            'checks': {
                'database': 'ok',
                'redis': 'ok'
            }
        }), 200
    except Exception as e:
        return jsonify({
            'status': 'not ready',
            'error': str(e)
        }), 503

@app.route('/metrics')
def metrics():
    """System metrics"""
    return jsonify({
        'cpu_percent': psutil.cpu_percent(),
        'memory_percent': psutil.virtual_memory().percent,
        'disk_usage': psutil.disk_usage('/').percent,
        'process_count': len(psutil.pids()),
        'uptime': psutil.boot_time()
    }), 200

if __name__ == '__main__':
    app.run(port=8001)
```

---

## Summary

Web servers are the backbone of web applications. Key takeaways:

### Development
- Use Python's built-in `http.server` for quick testing
- Understand the request/response cycle
- Learn socket programming fundamentals

### Standards
- **WSGI**: Standard for traditional Python web apps
- **ASGI**: Modern async standard for real-time apps
- Middleware pattern for cross-cutting concerns

### Production Servers
- **Gunicorn**: Most popular WSGI server, easy to use
- **uWSGI**: Full-featured, complex configuration
- **Uvicorn**: High-performance ASGI server for async apps

### Deployment
- Use Nginx as reverse proxy
- Configure systemd for process management
- Implement health checks and monitoring
- Use Docker for containerized deployments
- Plan for zero-downtime deployments

### Performance
- Choose appropriate worker model for your workload
- Calculate optimal number of workers
- Enable caching and compression
- Monitor and profile regularly
- Use load balancing for horizontal scaling

Understanding web servers allows you to deploy production-ready applications with confidence, optimize performance, and troubleshoot issues effectively.