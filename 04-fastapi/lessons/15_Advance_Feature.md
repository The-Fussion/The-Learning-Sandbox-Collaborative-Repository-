# Advanced Features in FastAPI

## 15.1 GraphQL

### GraphQL vs REST

Understanding the differences between GraphQL and REST APIs.

```python
"""
REST API:
- Multiple endpoints for different resources
- Fixed data structure per endpoint
- Over-fetching or under-fetching data
- Versioning through URLs (v1, v2)

GET /users/1
GET /users/1/posts
GET /users/1/friends

GraphQL:
- Single endpoint for all queries
- Client specifies exact data needed
- No over-fetching or under-fetching
- Schema evolution without versioning

POST /graphql
{
  user(id: 1) {
    name
    posts {
      title
    }
    friends {
      name
    }
  }
}
"""
```

**When to use GraphQL:**
- Complex data relationships
- Mobile apps (minimize data transfer)
- Multiple clients with different needs
- Rapid frontend development

**When to use REST:**
- Simple CRUD operations
- File uploads/downloads
- Caching is critical
- Standard HTTP features needed

### Strawberry Integration

Strawberry is a modern GraphQL library for Python with excellent FastAPI integration.

```bash
pip install "strawberry-graphql[fastapi]"
```

```python
from fastapi import FastAPI
import strawberry
from strawberry.fastapi import GraphQLRouter
from typing import List, Optional

# Define GraphQL types
@strawberry.type
class User:
    id: int
    name: str
    email: str
    age: Optional[int] = None

@strawberry.type
class Post:
    id: int
    title: str
    content: str
    author_id: int

# Sample data
users_db = [
    User(id=1, name="Alice", email="alice@example.com", age=30),
    User(id=2, name="Bob", email="bob@example.com", age=25),
]

posts_db = [
    Post(id=1, title="First Post", content="Hello World", author_id=1),
    Post(id=2, title="Second Post", content="GraphQL is great", author_id=1),
    Post(id=3, title="Bob's Post", content="My first post", author_id=2),
]

# Define queries
@strawberry.type
class Query:
    @strawberry.field
    def user(self, id: int) -> Optional[User]:
        return next((u for u in users_db if u.id == id), None)
    
    @strawberry.field
    def users(self) -> List[User]:
        return users_db
    
    @strawberry.field
    def post(self, id: int) -> Optional[Post]:
        return next((p for p in posts_db if p.id == id), None)
    
    @strawberry.field
    def posts(self, author_id: Optional[int] = None) -> List[Post]:
        if author_id:
            return [p for p in posts_db if p.author_id == author_id]
        return posts_db

# Define mutations
@strawberry.type
class Mutation:
    @strawberry.mutation
    def create_user(self, name: str, email: str, age: Optional[int] = None) -> User:
        new_id = max(u.id for u in users_db) + 1
        user = User(id=new_id, name=name, email=email, age=age)
        users_db.append(user)
        return user
    
    @strawberry.mutation
    def create_post(self, title: str, content: str, author_id: int) -> Post:
        new_id = max(p.id for p in posts_db) + 1
        post = Post(id=new_id, title=title, content=content, author_id=author_id)
        posts_db.append(post)
        return post

# Create schema
schema = strawberry.Schema(query=Query, mutation=Mutation)

# Create FastAPI app
app = FastAPI()

# Add GraphQL router
graphql_app = GraphQLRouter(schema)
app.include_router(graphql_app, prefix="/graphql")

@app.get("/")
async def root():
    return {"message": "GraphQL API is running at /graphql"}

# Example queries:
"""
# Get user by ID
query {
  user(id: 1) {
    id
    name
    email
    age
  }
}

# Get all users
query {
  users {
    id
    name
    email
  }
}

# Get posts by author
query {
  posts(authorId: 1) {
    id
    title
    content
  }
}

# Create user mutation
mutation {
  createUser(name: "Charlie", email: "charlie@example.com", age: 28) {
    id
    name
    email
  }
}
"""
```

### Graphene Integration

Graphene is another popular GraphQL library for Python.

```bash
pip install graphene fastapi-graphql
```

```python
from fastapi import FastAPI
import graphene
from starlette.graphql import GraphQLApp

# Define types
class UserType(graphene.ObjectType):
    id = graphene.Int()
    name = graphene.String()
    email = graphene.String()
    age = graphene.Int()

class PostType(graphene.ObjectType):
    id = graphene.Int()
    title = graphene.String()
    content = graphene.String()
    author_id = graphene.Int()
    author = graphene.Field(UserType)
    
    def resolve_author(self, info):
        return next((u for u in users_db if u['id'] == self.author_id), None)

# Sample data
users_db = [
    {"id": 1, "name": "Alice", "email": "alice@example.com", "age": 30},
    {"id": 2, "name": "Bob", "email": "bob@example.com", "age": 25},
]

posts_db = [
    {"id": 1, "title": "First Post", "content": "Hello", "author_id": 1},
    {"id": 2, "title": "Second Post", "content": "World", "author_id": 1},
]

# Define queries
class Query(graphene.ObjectType):
    user = graphene.Field(UserType, id=graphene.Int(required=True))
    users = graphene.List(UserType)
    post = graphene.Field(PostType, id=graphene.Int(required=True))
    posts = graphene.List(PostType, author_id=graphene.Int())
    
    def resolve_user(self, info, id):
        return next((u for u in users_db if u['id'] == id), None)
    
    def resolve_users(self, info):
        return users_db
    
    def resolve_post(self, info, id):
        return next((p for p in posts_db if p['id'] == id), None)
    
    def resolve_posts(self, info, author_id=None):
        if author_id:
            return [p for p in posts_db if p['author_id'] == author_id]
        return posts_db

# Define mutations
class CreateUser(graphene.Mutation):
    class Arguments:
        name = graphene.String(required=True)
        email = graphene.String(required=True)
        age = graphene.Int()
    
    user = graphene.Field(UserType)
    
    def mutate(self, info, name, email, age=None):
        new_id = max(u['id'] for u in users_db) + 1
        user = {"id": new_id, "name": name, "email": email, "age": age}
        users_db.append(user)
        return CreateUser(user=user)

class Mutation(graphene.ObjectType):
    create_user = CreateUser.Field()

# Create schema
schema = graphene.Schema(query=Query, mutation=Mutation)

# Create FastAPI app
app = FastAPI()
app.add_route("/graphql", GraphQLApp(schema=schema))
```

### GraphQL Queries and Mutations

Advanced query and mutation patterns.

```python
import strawberry
from typing import List, Optional
from datetime import datetime

@strawberry.type
class Comment:
    id: int
    content: str
    post_id: int
    user_id: int
    created_at: datetime

@strawberry.type
class Post:
    id: int
    title: str
    content: str
    author_id: int
    created_at: datetime
    
    @strawberry.field
    def author(self) -> Optional["User"]:
        return next((u for u in users_db if u.id == self.author_id), None)
    
    @strawberry.field
    def comments(self) -> List[Comment]:
        return [c for c in comments_db if c.post_id == self.id]

@strawberry.type
class User:
    id: int
    name: str
    email: str
    
    @strawberry.field
    def posts(self) -> List[Post]:
        return [p for p in posts_db if p.author_id == self.id]

# Advanced queries
@strawberry.type
class Query:
    @strawberry.field
    def search_posts(
        self,
        query: str,
        limit: int = 10,
        offset: int = 0
    ) -> List[Post]:
        """Search posts by title or content"""
        results = [
            p for p in posts_db
            if query.lower() in p.title.lower() or query.lower() in p.content.lower()
        ]
        return results[offset:offset + limit]
    
    @strawberry.field
    def user_with_posts(self, user_id: int) -> Optional[User]:
        """Get user with all their posts in one query"""
        return next((u for u in users_db if u.id == user_id), None)

# Complex mutations
@strawberry.type
class Mutation:
    @strawberry.mutation
    def update_post(
        self,
        post_id: int,
        title: Optional[str] = None,
        content: Optional[str] = None
    ) -> Optional[Post]:
        """Update post fields"""
        post = next((p for p in posts_db if p.id == post_id), None)
        if not post:
            return None
        
        if title:
            post.title = title
        if content:
            post.content = content
        
        return post
    
    @strawberry.mutation
    def delete_post(self, post_id: int) -> bool:
        """Delete a post"""
        global posts_db
        initial_length = len(posts_db)
        posts_db = [p for p in posts_db if p.id != post_id]
        return len(posts_db) < initial_length

# Example complex query:
"""
query {
  userWithPosts(userId: 1) {
    id
    name
    email
    posts {
      id
      title
      comments {
        id
        content
      }
    }
  }
}
"""
```

### GraphQL Subscriptions

Real-time updates using GraphQL subscriptions.

```python
import strawberry
from typing import AsyncGenerator
import asyncio

@strawberry.type
class Message:
    id: int
    content: str
    user_id: int
    timestamp: str

messages_queue = asyncio.Queue()

@strawberry.type
class Subscription:
    @strawberry.subscription
    async def message_added(self, user_id: Optional[int] = None) -> AsyncGenerator[Message, None]:
        """Subscribe to new messages"""
        while True:
            message = await messages_queue.get()
            
            # Filter by user_id if provided
            if user_id is None or message.user_id == user_id:
                yield message

@strawberry.type
class Mutation:
    @strawberry.mutation
    async def send_message(self, content: str, user_id: int) -> Message:
        """Send a new message"""
        from datetime import datetime
        
        message = Message(
            id=len(messages_db) + 1,
            content=content,
            user_id=user_id,
            timestamp=datetime.now().isoformat()
        )
        
        messages_db.append(message)
        await messages_queue.put(message)
        
        return message

# Create schema with subscriptions
schema = strawberry.Schema(
    query=Query,
    mutation=Mutation,
    subscription=Subscription
)

# Use with WebSocket support
from strawberry.fastapi import GraphQLRouter

graphql_app = GraphQLRouter(
    schema,
    subscription_protocols=[
        "graphql-ws",
        "graphql-transport-ws"
    ]
)

app.include_router(graphql_app, prefix="/graphql")

# Example subscription:
"""
subscription {
  messageAdded(userId: 1) {
    id
    content
    timestamp
  }
}
"""
```

## 15.2 gRPC

### gRPC Overview

gRPC is a high-performance RPC framework using Protocol Buffers.

```python
"""
gRPC Advantages:
- Fast binary serialization (Protocol Buffers)
- Bi-directional streaming
- Built-in authentication
- Language-agnostic
- Code generation from .proto files

gRPC vs REST:
- gRPC: Binary, faster, smaller payload
- REST: Text-based, human-readable, wider support

Use gRPC for:
- Microservices communication
- Real-time systems
- Mobile clients
- Polyglot environments
"""
```

### gRPC with FastAPI

Integrate gRPC services with FastAPI.

```bash
pip install grpcio grpcio-tools
```

```python
# user.proto
"""
syntax = "proto3";

package user;

service UserService {
  rpc GetUser (UserRequest) returns (UserResponse);
  rpc CreateUser (CreateUserRequest) returns (UserResponse);
  rpc StreamUsers (StreamRequest) returns (stream UserResponse);
}

message UserRequest {
  int32 id = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message UserResponse {
  int32 id = 1;
  string name = 2;
  string email = 3;
}

message StreamRequest {
  int32 limit = 1;
}
"""

# Generate Python code from .proto file:
# python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. user.proto

# user_service.py
import grpc
from concurrent import futures
import user_pb2
import user_pb2_grpc

class UserServicer(user_pb2_grpc.UserServiceServicer):
    def __init__(self):
        self.users = {}
        self.next_id = 1
    
    def GetUser(self, request, context):
        user = self.users.get(request.id)
        if not user:
            context.set_code(grpc.StatusCode.NOT_FOUND)
            context.set_details('User not found')
            return user_pb2.UserResponse()
        
        return user_pb2.UserResponse(
            id=user['id'],
            name=user['name'],
            email=user['email']
        )
    
    def CreateUser(self, request, context):
        user = {
            'id': self.next_id,
            'name': request.name,
            'email': request.email
        }
        self.users[self.next_id] = user
        self.next_id += 1
        
        return user_pb2.UserResponse(
            id=user['id'],
            name=user['name'],
            email=user['email']
        )
    
    def StreamUsers(self, request, context):
        for user_id, user in list(self.users.items())[:request.limit]:
            yield user_pb2.UserResponse(
                id=user['id'],
                name=user['name'],
                email=user['email']
            )

# Start gRPC server
def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    user_pb2_grpc.add_UserServiceServicer_to_server(UserServicer(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
    server.wait_for_termination()

# Run alongside FastAPI
from fastapi import FastAPI
import threading

app = FastAPI()

@app.on_event("startup")
async def startup_event():
    # Start gRPC server in background thread
    grpc_thread = threading.Thread(target=serve, daemon=True)
    grpc_thread.start()

@app.get("/")
async def root():
    return {"message": "FastAPI running", "grpc_port": 50051}
```

### Protocol Buffers

Define data structures with Protocol Buffers.

```protobuf
// product.proto
syntax = "proto3";

package product;

// Enums
enum Category {
  ELECTRONICS = 0;
  CLOTHING = 1;
  FOOD = 2;
  BOOKS = 3;
}

// Nested messages
message Price {
  double amount = 1;
  string currency = 2;
}

message Product {
  int32 id = 1;
  string name = 2;
  string description = 3;
  Price price = 4;
  Category category = 5;
  repeated string tags = 6;  // Array of strings
  map<string, string> metadata = 7;  // Key-value pairs
}

// Request/Response messages
message GetProductRequest {
  int32 id = 1;
}

message ListProductsRequest {
  int32 page = 1;
  int32 page_size = 2;
  Category category = 3;
}

message ProductList {
  repeated Product products = 1;
  int32 total_count = 2;
}

// Service definition
service ProductService {
  rpc GetProduct(GetProductRequest) returns (Product);
  rpc ListProducts(ListProductsRequest) returns (ProductList);
  rpc CreateProduct(Product) returns (Product);
  rpc UpdateProduct(Product) returns (Product);
  rpc DeleteProduct(GetProductRequest) returns (google.protobuf.Empty);
}
```

### gRPC Services

Implement complete gRPC services.

```python
import grpc
from concurrent import futures
import product_pb2
import product_pb2_grpc
from typing import Dict

class ProductServicer(product_pb2_grpc.ProductServiceServicer):
    def __init__(self):
        self.products: Dict[int, product_pb2.Product] = {}
        self.next_id = 1
    
    def GetProduct(self, request, context):
        """Get a single product"""
        product = self.products.get(request.id)
        
        if not product:
            context.set_code(grpc.StatusCode.NOT_FOUND)
            context.set_details(f'Product {request.id} not found')
            return product_pb2.Product()
        
        return product
    
    def ListProducts(self, request, context):
        """List products with pagination"""
        # Filter by category if specified
        products = list(self.products.values())
        if request.category:
            products = [p for p in products if p.category == request.category]
        
        # Pagination
        start = (request.page - 1) * request.page_size
        end = start + request.page_size
        page_products = products[start:end]
        
        return product_pb2.ProductList(
            products=page_products,
            total_count=len(products)
        )
    
    def CreateProduct(self, request, context):
        """Create a new product"""
        product = product_pb2.Product()
        product.CopyFrom(request)
        product.id = self.next_id
        
        self.products[self.next_id] = product
        self.next_id += 1
        
        return product
    
    def UpdateProduct(self, request, context):
        """Update an existing product"""
        if request.id not in self.products:
            context.set_code(grpc.StatusCode.NOT_FOUND)
            context.set_details(f'Product {request.id} not found')
            return product_pb2.Product()
        
        self.products[request.id].CopyFrom(request)
        return self.products[request.id]
    
    def DeleteProduct(self, request, context):
        """Delete a product"""
        if request.id not in self.products:
            context.set_code(grpc.StatusCode.NOT_FOUND)
            context.set_details(f'Product {request.id} not found')
        else:
            del self.products[request.id]
        
        return product_pb2.google_dot_protobuf_dot_empty__pb2.Empty()

# gRPC Client
class ProductClient:
    def __init__(self, host='localhost', port=50051):
        self.channel = grpc.insecure_channel(f'{host}:{port}')
        self.stub = product_pb2_grpc.ProductServiceStub(self.channel)
    
    def get_product(self, product_id: int):
        request = product_pb2.GetProductRequest(id=product_id)
        return self.stub.GetProduct(request)
    
    def create_product(self, name: str, price: float, category: int):
        product = product_pb2.Product(
            name=name,
            price=product_pb2.Price(amount=price, currency="USD"),
            category=category
        )
        return self.stub.CreateProduct(product)

# Use with FastAPI
from fastapi import FastAPI, HTTPException

app = FastAPI()

# Start gRPC server
def start_grpc_server():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    product_pb2_grpc.add_ProductServiceServicer_to_server(
        ProductServicer(), 
        server
    )
    server.add_insecure_port('[::]:50051')
    server.start()
    server.wait_for_termination()

@app.on_event("startup")
async def startup():
    import threading
    grpc_thread = threading.Thread(target=start_grpc_server, daemon=True)
    grpc_thread.start()

# REST endpoint that calls gRPC
client = ProductClient()

@app.get("/products/{product_id}")
async def get_product_rest(product_id: int):
    try:
        product = client.get_product(product_id)
        return {
            "id": product.id,
            "name": product.name,
            "price": product.price.amount
        }
    except grpc.RpcError as e:
        raise HTTPException(status_code=404, detail=str(e))
```

## 15.3 Server-Sent Events

### SSE Implementation

Server-Sent Events allow servers to push updates to clients.

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import asyncio
from datetime import datetime

app = FastAPI()

async def event_generator():
    """Generate server-sent events"""
    count = 0
    while True:
        count += 1
        # SSE format: data: {message}\n\n
        yield f"data: {{'count': {count}, 'timestamp': '{datetime.now().isoformat()}'}}\n\n"
        await asyncio.sleep(1)

@app.get("/events")
async def events():
    """SSE endpoint"""
    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no"  # Disable buffering in nginx
        }
    )

# Named events
async def named_event_generator():
    """Generate events with custom event names"""
    while True:
        # Send different event types
        yield f"event: message\ndata: New message available\n\n"
        await asyncio.sleep(2)
        
        yield f"event: notification\ndata: {{'type': 'info', 'text': 'Update'}}\n\n"
        await asyncio.sleep(3)
        
        yield f"event: heartbeat\ndata: ping\n\n"
        await asyncio.sleep(5)

@app.get("/events/named")
async def named_events():
    return StreamingResponse(
        named_event_generator(),
        media_type="text/event-stream"
    )
```

### Streaming Responses

Advanced streaming patterns with SSE.

```python
from fastapi import FastAPI, BackgroundTasks
from fastapi.responses import StreamingResponse
import asyncio
import json
from typing import AsyncGenerator

app = FastAPI()

# Pub/Sub pattern for SSE
class EventBroadcaster:
    def __init__(self):
        self.subscribers = set()
    
    async def subscribe(self) -> AsyncGenerator[str, None]:
        """Subscribe to events"""
        queue = asyncio.Queue()
        self.subscribers.add(queue)
        
        try:
            while True:
                event = await queue.get()
                yield f"data: {json.dumps(event)}\n\n"
        finally:
            self.subscribers.remove(queue)
    
    async def publish(self, event: dict):
        """Publish event to all subscribers"""
        for queue in self.subscribers:
            await queue.put(event)

broadcaster = EventBroadcaster()

@app.get("/stream")
async def stream():
    """SSE endpoint with pub/sub"""
    return StreamingResponse(
        broadcaster.subscribe(),
        media_type="text/event-stream"
    )

@app.post("/broadcast")
async def broadcast(message: str):
    """Broadcast message to all connected clients"""
    await broadcaster.publish({"message": message, "timestamp": datetime.now().isoformat()})
    return {"status": "broadcasted"}

# Progress streaming
async def progress_generator(task_id: str):
    """Stream progress updates"""
    for progress in range(0, 101, 10):
        yield f"data: {{'task_id': '{task_id}', 'progress': {progress}}}\n\n"
        await asyncio.sleep(0.5)
    
    yield f"data: {{'task_id': '{task_id}', 'status': 'completed'}}\n\n"

@app.get("/progress/{task_id}")
async def track_progress(task_id: str):
    """Stream task progress"""
    return StreamingResponse(
        progress_generator(task_id),
        media_type="text/event-stream"
    )

# Log streaming
async def log_stream():
    """Stream log entries"""
    logs = [
        "Starting process...",
        "Loading configuration...",
        "Connecting to database...",
        "Processing data...",
        "Generating report...",
        "Completed successfully!"
    ]
    
    for log in logs:
        yield f"data: {{'level': 'info', 'message': '{log}'}}\n\n"
        await asyncio.sleep(1)

@app.get("/logs")
async def stream_logs():
    return StreamingResponse(log_stream(), media_type="text/event-stream")
```

### EventSource API

Client-side implementation using EventSource.

```html
<!DOCTYPE html>
<html>
<head>
    <title>SSE Client</title>
</head>
<body>
    <h1>Server-Sent Events Demo</h1>
    <div id="events"></div>
    
    <script>
        // Basic SSE connection
        const eventSource = new EventSource('/events');
        
        eventSource.onmessage = function(event) {
            const data = JSON.parse(event.data);
            console.log('Received:', data);
            
            const div = document.getElementById('events');
            div.innerHTML += `<p>${data.count}: ${data.timestamp}</p>`;
        };
        
        eventSource.onerror = function(error) {
            console.error('SSE error:', error);
            eventSource.close();
        };
        
        // Named events
        const namedSource = new EventSource('/events/named');
        
        namedSource.addEventListener('message', function(event) {
            console.log('Message event:', event.data);
        });
        
        namedSource.addEventListener('notification', function(event) {
            const data = JSON.parse(event.data);
            console.log('Notification:', data);
        });
        
        namedSource.addEventListener('heartbeat', function(event) {
            console.log('Heartbeat:', event.data);
        });
        
        // Reconnection logic
        namedSource.onerror = function(error) {
            console.log('Connection lost, reconnecting...');
            // EventSource automatically reconnects
        };
        
        // Close connection
        window.addEventListener('beforeunload', function() {
            eventSource.close();
            namedSource.close();
        });
    </script>
</body>
</html>
```

### SSE vs WebSockets

Understanding when to use each technology.

```python
"""
Server-Sent Events (SSE):
✓ One-way communication (server → client)
✓ Built on HTTP
✓ Automatic reconnection
✓ Event IDs for missed messages
✓ Simpler to implement
✓ Works through proxies/firewalls
✗ Text-only (no binary)
✗ Limited browser connections (6 per domain)

WebSockets:
✓ Bi-directional communication
✓ Binary and text data
✓ Lower latency
✓ Unlimited connections
✗ More complex implementation
✗ Requires WebSocket support
✗ Manual reconnection logic

Use SSE for:
- Live feeds (news, stock prices)
- Notifications
- Progress updates
- One-way data streams

Use WebSockets for:
- Chat applications
- Real-time collaboration
- Gaming
- Bi-directional communication
"""

from fastapi import FastAPI, WebSocket
from fastapi.responses import StreamingResponse
import asyncio

app = FastAPI()

# SSE example
async def price_updates():
    """Stream stock prices via SSE"""
    import random
    while True:
        price = round(random.uniform(100, 200), 2)
        yield f"data: {{'symbol': 'AAPL', 'price': {price}}}\n\n"
        await asyncio.sleep(1)

@app.get("/stock-prices")
async def stock_prices_sse():
    return StreamingResponse(price_updates(), media_type="text/event-stream")

# WebSocket example
@app.websocket("/ws/stock-prices")
async def stock_prices_ws(websocket: WebSocket):
    """Stream stock prices via WebSocket"""
    await websocket.accept()
    
    try:
        while True:
            import random
            price = round(random.uniform(100, 200), 2)
            await websocket.send_json({"symbol": "AAPL", "price": price})
            await asyncio.sleep(1)
    except Exception:
        pass
```

## 15.4 Request/Response Lifecycle Events

### Startup Events

Execute code when the application starts.

```python
from fastapi import FastAPI
from contextlib import asynccontextmanager

# Modern approach with lifespan (FastAPI 0.93+)
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    print("Application starting...")
    
    # Initialize database
    await init_database()
    
    # Load ML model
    model = load_model()
    app.state.model = model
    
    # Start background tasks
    app.state.background_task = asyncio.create_task(background_worker())
    
    yield  # Application is running
    
    # Shutdown
    print("Application shutting down...")
    
    # Cancel background tasks
    app.state.background_task.cancel()
    
    # Close database
    await close_database()

app = FastAPI(lifespan=lifespan)

# Legacy approach (still works)
@app.on_event("startup")
async def startup_event():
    """Called when application starts"""
    print("Starting up...")
    
    # Database connection
    app.state.db = await connect_to_database()
    
    # Cache initialization
    app.state.cache = await init_cache()
    
    # Load configuration
    app.state.config = load_config()

@app.on_event("startup")
async def startup_second():
    """Multiple startup events execute in order"""
    print("Second startup event")

@app.get("/")
async def root():
    # Access application state
    return {"db": str(app.state.db), "cache": str(app.state.cache)}
```

### Shutdown Events

Clean up resources on application shutdown.

```python
from fastapi import FastAPI

app = FastAPI()

@app.on_event("shutdown")
async def shutdown_event():
    """Called when application shuts down"""
    print("Shutting down...")
    
    # Close database connections
    if hasattr(app.state, 'db'):
        await app.state.db.close()
    
    # Close cache connections
    if hasattr(app.state, 'cache'):
        await app.state.cache.close()
    
    # Cancel background tasks
    if hasattr(app.state, 'background_tasks'):
        for task in app.state.background_tasks:
            task.cancel()
    
    # Save state to disk
    save_application_state()

@app.on_event("shutdown")
async def shutdown_cleanup():
    """Multiple shutdown events"""
    print("Final cleanup")
```

### Lifespan Events

Modern lifespan context manager pattern.

```python
from fastapi import FastAPI
from contextlib import asynccontextmanager
import asyncio

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Manage application lifespan"""
    
    # --- STARTUP ---
    print("🚀 Application starting...")
    
    # Database
    from sqlalchemy.ext.asyncio import create_async_engine
    engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")
    app.state.db_engine = engine
    
    # Redis
    import aioredis
    redis = await aioredis.create_redis_pool("redis://localhost")
    app.state.redis = redis
    
    # Background tasks
    async def periodic_task():
        while True:
            print("Running periodic task...")
            await asyncio.sleep(60)
    
    task = asyncio.create_task(periodic_task())
    app.state.periodic_task = task
    
    # Machine Learning model
    import joblib
    model = joblib.load("model.pkl")
    app.state.ml_model = model
    
    print("✅ Startup complete")
    
    yield  # Application runs here
    
    # --- SHUTDOWN ---
    print("🛑 Application shutting down...")
    
    # Cancel background tasks
    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        pass
    
    # Close connections
    await engine.dispose()
    redis.close()
    await redis.wait_closed()
    
    print("✅ Shutdown complete")

app = FastAPI(lifespan=lifespan)

@app.get("/predict")
async def predict(data: dict):
    # Access model from app state
    prediction = app.state.ml_model.predict([data])
    return {"prediction": prediction.tolist()}
```

### Application State Management

Manage global application state.

```python
from fastapi import FastAPI, Request
from pydantic import BaseModel
from typing import Dict, Any

app = FastAPI()

# Application state
class AppState:
    def __init__(self):
        self.request_count: int = 0
        self.active_users: set = set()
        self.cache: Dict[str, Any] = {}
        self.config: dict = {}
    
    def increment_requests(self):
        self.request_count += 1
    
    def add_user(self, user_id: str):
        self.active_users.add(user_id)
    
    def remove_user(self, user_id: str):
        self.active_users.discard(user_id)

@app.on_event("startup")
async def startup():
    app.state.app_state = AppState()
    app.state.app_state.config = {"max_upload_size": 10_000_000}

# Middleware to track requests
@app.middleware("http")
async def track_requests(request: Request, call_next):
    app.state.app_state.increment_requests()
    response = await call_next(request)
    return response

@app.get("/stats")
async def get_stats():
    return {
        "total_requests": app.state.app_state.request_count,
        "active_users": len(app.state.app_state.active_users),
        "cache_size": len(app.state.app_state.cache)
    }

# Dependency for accessing state
from fastapi import Depends

def get_app_state(request: Request) -> AppState:
    return request.app.state.app_state

@app.get("/config")
async def get_config(state: AppState = Depends(get_app_state)):
    return state.config
```

## 15.5 Sub-Applications

### Mounting Sub-Applications

Create modular applications by mounting sub-applications.

```python
from fastapi import FastAPI

# Main application
app = FastAPI(title="Main API")

# Sub-application 1: User management
users_app = FastAPI(title="Users API")

@users_app.get("/")
async def list_users():
    return [{"id": 1, "name": "Alice"}, {"id": 2, "name": "Bob"}]

@users_app.get("/{user_id}")
async def get_user(user_id: int):
    return {"id": user_id, "name": "User"}

@users_app.post("/")
async def create_user(name: str):
    return {"id": 3, "name": name}

# Sub-application 2: Products
products_app = FastAPI(title="Products API")

@products_app.get("/")
async def list_products():
    return [{"id": 1, "name": "Widget"}, {"id": 2, "name": "Gadget"}]

@products_app.get("/{product_id}")
async def get_product(product_id: int):
    return {"id": product_id, "name": "Product"}

# Mount sub-applications
app.mount("/users", users_app)
app.mount("/products", products_app)

@app.get("/")
async def root():
    return {
        "message": "Main API",
        "sub_apps": {
            "users": "/users/docs",
            "products": "/products/docs"
        }
    }

# Accessing endpoints:
# GET http://localhost:8000/
# GET http://localhost:8000/users/
# GET http://localhost:8000/users/1
# GET http://localhost:8000/products/
```

### Application Composition

Build complex applications from smaller components.

```python
from fastapi import FastAPI, APIRouter

# Create routers for different modules
auth_router = APIRouter(prefix="/auth", tags=["Authentication"])
api_v1_router = APIRouter(prefix="/v1", tags=["API V1"])
api_v2_router = APIRouter(prefix="/v2", tags=["API V2"])

@auth_router.post("/login")
async def login(username: str, password: str):
    return {"access_token": "token"}

@auth_router.post("/register")
async def register(username: str, email: str, password: str):
    return {"message": "User created"}

@api_v1_router.get("/items")
async def get_items_v1():
    return {"version": 1, "items": []}

@api_v2_router.get("/items")
async def get_items_v2():
    return {"version": 2, "items": [], "metadata": {}}

# Admin sub-application
admin_app = FastAPI(title="Admin Panel")

@admin_app.get("/users")
async def admin_list_users():
    return {"users": []}

@admin_app.get("/settings")
async def admin_settings():
    return {"settings": {}}

# Main application
app = FastAPI(title="Composed Application")

# Include routers
app.include_router(auth_router)
app.include_router(api_v1_router, prefix="/api")
app.include_router(api_v2_router, prefix="/api")

# Mount sub-application
app.mount("/admin", admin_app)

@app.get("/")
async def root():
    return {"message": "Main application"}

# URL structure:
# /auth/login
# /auth/register
# /api/v1/items
# /api/v2/items
# /admin/users
# /admin/settings
```

### Microservices Architecture

Build microservices with FastAPI.

```python
# Service 1: User Service (port 8001)
from fastapi import FastAPI
import httpx

users_service = FastAPI(title="User Service")

users_db = {
    1: {"id": 1, "name": "Alice", "email": "alice@example.com"},
    2: {"id": 2, "name": "Bob", "email": "bob@example.com"},
}

@users_service.get("/users/{user_id}")
async def get_user(user_id: int):
    return users_db.get(user_id, {})

@users_service.get("/users")
async def list_users():
    return list(users_db.values())

# Service 2: Orders Service (port 8002)
orders_service = FastAPI(title="Orders Service")

orders_db = {
    1: {"id": 1, "user_id": 1, "total": 99.99, "status": "pending"},
    2: {"id": 2, "user_id": 2, "total": 149.99, "status": "completed"},
}

@orders_service.get("/orders/{order_id}")
async def get_order(order_id: int):
    order = orders_db.get(order_id)
    if not order:
        return {}
    
    # Call user service
    async with httpx.AsyncClient() as client:
        user_response = await client.get(
            f"http://localhost:8001/users/{order['user_id']}"
        )
        order["user"] = user_response.json()
    
    return order

@orders_service.get("/orders")
async def list_orders():
    return list(orders_db.values())

# Service 3: API Gateway (port 8000)
gateway = FastAPI(title="API Gateway")

@gateway.get("/api/users/{user_id}")
async def gateway_get_user(user_id: int):
    async with httpx.AsyncClient() as client:
        response = await client.get(f"http://localhost:8001/users/{user_id}")
        return response.json()

@gateway.get("/api/orders/{order_id}")
async def gateway_get_order(order_id: int):
    async with httpx.AsyncClient() as client:
        response = await client.get(f"http://localhost:8002/orders/{order_id}")
        return response.json()

@gateway.get("/api/user-orders/{user_id}")
async def get_user_with_orders(user_id: int):
    async with httpx.AsyncClient() as client:
        # Parallel requests
        user_task = client.get(f"http://localhost:8001/users/{user_id}")
        orders_task = client.get(f"http://localhost:8002/orders")
        
        user_response, orders_response = await asyncio.gather(user_task, orders_task)
        
        user = user_response.json()
        all_orders = orders_response.json()
        user_orders = [o for o in all_orders if o["user_id"] == user_id]
        
        return {
            "user": user,
            "orders": user_orders
        }

# Run services:
# uvicorn users_service:users_service --port 8001
# uvicorn orders_service:orders_service --port 8002
# uvicorn gateway:gateway --port 8000
```

### Service Communication

Implement service-to-service communication patterns.

```python
from fastapi import FastAPI, HTTPException
import httpx
from typing import Optional
import asyncio

app = FastAPI()

# Service Registry
class ServiceRegistry:
    def __init__(self):
        self.services = {
            "users": "http://localhost:8001",
            "orders": "http://localhost:8002",
            "products": "http://localhost:8003",
        }
    
    def get_url(self, service_name: str) -> str:
        return self.services.get(service_name, "")

registry = ServiceRegistry()

# HTTP Client with retry logic
class ServiceClient:
    def __init__(self, service_name: str):
        self.base_url = registry.get_url(service_name)
        self.client = httpx.AsyncClient(timeout=10.0)
    
    async def get(self, path: str, retries: int = 3):
        """GET request with retry logic"""
        for attempt in range(retries):
            try:
                response = await self.client.get(f"{self.base_url}{path}")
                response.raise_for_status()
                return response.json()
            except httpx.HTTPError as e:
                if attempt == retries - 1:
                    raise HTTPException(
                        status_code=503,
                        detail=f"Service unavailable: {str(e)}"
                    )
                await asyncio.sleep(2 ** attempt)  # Exponential backoff
    
    async def post(self, path: str, data: dict):
        """POST request"""
        response = await self.client.post(
            f"{self.base_url}{path}",
            json=data
        )
        response.raise_for_status()
        return response.json()

# Circuit Breaker Pattern
class CircuitBreaker:
    def __init__(self, failure_threshold: int = 5, timeout: int = 60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failures = 0
        self.last_failure_time = None
        self.state = "closed"  # closed, open, half-open
    
    def is_open(self):
        if self.state == "open":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "half-open"
                return False
            return True
        return False
    
    def record_success(self):
        self.failures = 0
        self.state = "closed"
    
    def record_failure(self):
        self.failures += 1
        self.last_failure_time = time.time()
        
        if self.failures >= self.failure_threshold:
            self.state = "open"

# Usage
user_client = ServiceClient("users")
order_client = ServiceClient("orders")

@app.get("/user-profile/{user_id}")
async def get_user_profile(user_id: int):
    """Aggregate data from multiple services"""
    
    # Parallel requests
    user_data, orders_data = await asyncio.gather(
        user_client.get(f"/users/{user_id}"),
        order_client.get(f"/orders?user_id={user_id}"),
        return_exceptions=True
    )
    
    # Handle partial failures
    if isinstance(user_data, Exception):
        raise HTTPException(status_code=500, detail="User service unavailable")
    
    if isinstance(orders_data, Exception):
        orders_data = []  # Graceful degradation
    
    return {
        "user": user_data,
        "orders": orders_data
    }

# Event-driven communication
from fastapi import BackgroundTasks

async def publish_event(event_type: str, data: dict):
    """Publish event to message queue"""
    # Integration with RabbitMQ, Kafka, etc.
    async with httpx.AsyncClient() as client:
        await client.post(
            "http://message-queue:5000/publish",
            json={"event_type": event_type, "data": data}
        )

@app.post("/orders")
async def create_order(order_data: dict, background_tasks: BackgroundTasks):
    # Create order
    order = await order_client.post("/orders", order_data)
    
    # Publish event asynchronously
    background_tasks.add_task(
        publish_event,
        "order.created",
        {"order_id": order["id"], "user_id": order["user_id"]}
    )
    
    return order
```

---

## Complete Advanced Features Example

```python
from fastapi import FastAPI, WebSocket, BackgroundTasks
from fastapi.responses import StreamingResponse
from contextlib import asynccontextmanager
import asyncio
import strawberry
from strawberry.fastapi import GraphQLRouter
from typing import AsyncGenerator
import httpx

# Lifespan management
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    print("🚀 Starting application...")
    
    # Initialize services
    app.state.cache = {}
    app.state.websocket_connections = set()
    
    # Start background task
    async def background_worker():
        while True:
            print("Background task running...")
            await asyncio.sleep(60)
    
    app.state.worker = asyncio.create_task(background_worker())
    
    yield
    
    # Shutdown
    print("🛑 Shutting down...")
    app.state.worker.cancel()

app = FastAPI(lifespan=lifespan, title="Advanced Features Demo")

# GraphQL
@strawberry.type
class User:
    id: int
    name: str
    email: str

@strawberry.type
class Query:
    @strawberry.field
    def user(self, id: int) -> User:
        return User(id=id, name="John Doe", email="john@example.com")

schema = strawberry.Schema(query=Query)
graphql_app = GraphQLRouter(schema)
app.include_router(graphql_app, prefix="/graphql")

# Server-Sent Events
async def event_stream() -> AsyncGenerator[str, None]:
    count = 0
    while True:
        count += 1
        yield f"data: {{\"count\": {count}}}\n\n"
        await asyncio.sleep(1)

@app.get("/events")
async def events():
    return StreamingResponse(event_stream(), media_type="text/event-stream")

# WebSocket
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    app.state.websocket_connections.add(websocket)
    
    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"Echo: {data}")
    finally:
        app.state.websocket_connections.remove(websocket)

# Sub-applications
admin_app = FastAPI(title="Admin Panel")

@admin_app.get("/stats")
async def admin_stats():
    return {
        "cache_size": len(app.state.cache),
        "websocket_connections": len(app.state.websocket_connections)
    }

app.mount("/admin", admin_app)

@app.get("/")
async def root():
    return {
        "graphql": "/graphql",
        "events": "/events",
        "websocket": "/ws",
        "admin": "/admin/docs"
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

**Key Takeaways:**
- GraphQL provides flexible querying but adds complexity
- gRPC is excellent for microservices communication
- Use SSE for one-way real-time updates
- Lifecycle events manage application state
- Sub-applications enable modular architecture
- Microservices require proper service communication patterns

**Additional Resources:**
- [Strawberry GraphQL Documentation](https://strawberry.rocks/)
- [gRPC Python Documentation](https://grpc.io/docs/languages/python/)
- [Server-Sent Events Specification](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- [Microservices Patterns](https://microservices.io/patterns/)