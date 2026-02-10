# WebSockets & Real-Time

## 11.1 WebSocket Basics

### What are WebSockets?

WebSockets provide full-duplex, bidirectional communication channels over a single TCP connection. Unlike HTTP's request-response model, WebSockets enable real-time, persistent connections between client and server.

**Key Characteristics**:
- **Persistent Connection**: Stays open for continuous communication
- **Bidirectional**: Both client and server can send messages anytime
- **Low Latency**: No need to establish new connections for each message
- **Real-Time**: Perfect for live updates and interactive applications

**Use Cases**:
- Chat applications
- Live notifications
- Real-time dashboards
- Online gaming
- Collaborative editing
- Live sports scores
- Stock tickers
- IoT device communication

### WebSocket vs HTTP

| Feature | HTTP | WebSocket |
|---------|------|-----------|
| **Connection** | Request-Response | Persistent |
| **Direction** | Unidirectional | Bidirectional |
| **Overhead** | High (new connection each time) | Low (single connection) |
| **Latency** | Higher | Lower |
| **Headers** | Sent with every request | Only on handshake |
| **Server Push** | Not native (polling required) | Native support |
| **Use Case** | REST APIs, web pages | Real-time updates |

**HTTP Request-Response**:
```
Client → Server: GET /data HTTP/1.1
Server → Client: 200 OK {data}

Client → Server: GET /data HTTP/1.1
Server → Client: 200 OK {data}

(New connection each time)
```

**WebSocket Communication**:
```
Client ↔ Server: [Handshake]
Client ↔ Server: Connected!

Client → Server: message
Server → Client: message
Client → Server: message
Server → Client: message

(Same connection throughout)
```

### WebSocket Handshake

The WebSocket connection starts with an HTTP upgrade request.

**Handshake Process**:

1. **Client Request** (HTTP Upgrade):
```http
GET /ws HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

2. **Server Response**:
```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

3. **Connection Established**: Now both can send/receive messages

**In FastAPI**:
```python
from fastapi import FastAPI, WebSocket

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    # Accept connection (completes handshake)
    await websocket.accept()
    
    # Connection is now open
    await websocket.send_text("Connected!")
```

### WebSocket Connection Lifecycle

**Lifecycle Stages**:

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    # 1. CONNECTING - Handshake in progress
    
    # 2. OPEN - Accept connection
    await websocket.accept()
    print("Client connected")
    
    try:
        # 3. COMMUNICATING - Exchange messages
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"Echo: {data}")
    
    except WebSocketDisconnect:
        # 4. CLOSING - Connection closing
        print("Client disconnected")
    
    # 5. CLOSED - Connection terminated
```

**Connection States**:
- **CONNECTING (0)**: Initial handshake
- **OPEN (1)**: Connection established, can send/receive
- **CLOSING (2)**: Connection closing
- **CLOSED (3)**: Connection closed

---

## 11.2 WebSocket Implementation

### WebSocket Endpoint

**Basic WebSocket Endpoint**:
```python
from fastapi import FastAPI, WebSocket

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    await websocket.send_text("Hello WebSocket!")
```

**With Path Parameters**:
```python
@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: int):
    await websocket.accept()
    await websocket.send_text(f"Welcome, client {client_id}")
```

**With Query Parameters**:
```python
@app.websocket("/ws")
async def websocket_endpoint(
    websocket: WebSocket,
    token: str = None
):
    if not token:
        await websocket.close(code=1008)  # Policy violation
        return
    
    await websocket.accept()
    await websocket.send_text("Authenticated!")
```

### Accepting Connections

```python
from fastapi import FastAPI, WebSocket

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    # Accept with default subprotocol
    await websocket.accept()
    
    # Or specify subprotocol
    # await websocket.accept(subprotocol="chat")
    
    await websocket.send_text("Connected!")
```

**Conditional Acceptance**:
```python
@app.websocket("/ws")
async def websocket_endpoint(
    websocket: WebSocket,
    api_key: str = None
):
    # Validate before accepting
    if api_key != "secret-key":
        await websocket.close(code=1008, reason="Invalid API key")
        return
    
    await websocket.accept()
    await websocket.send_text("Authenticated connection established")
```

### Sending Messages

**Text Messages**:
```python
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    
    # Send text
    await websocket.send_text("Hello!")
    await websocket.send_text("Another message")
```

**JSON Messages**:
```python
import json

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    
    # Send JSON
    await websocket.send_json({
        "type": "greeting",
        "message": "Hello!",
        "timestamp": "2024-01-01T12:00:00"
    })
```

**Binary Messages**:
```python
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    
    # Send binary data
    binary_data = b"Binary content"
    await websocket.send_bytes(binary_data)
```

### Receiving Messages

**Receive Text**:
```python
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    
    while True:
        # Receive text message
        data = await websocket.receive_text()
        print(f"Received: {data}")
        
        # Echo back
        await websocket.send_text(f"Echo: {data}")
```

**Receive JSON**:
```python
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    
    while True:
        # Receive JSON
        data = await websocket.receive_json()
        
        message_type = data.get("type")
        if message_type == "ping":
            await websocket.send_json({"type": "pong"})
        elif message_type == "message":
            await websocket.send_json({
                "type": "ack",
                "message": data.get("message")
            })
```

**Receive Any Format**:
```python
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    
    while True:
        # Receive any message type
        message = await websocket.receive()
        
        if "text" in message:
            text_data = message["text"]
            await websocket.send_text(f"Text: {text_data}")
        
        elif "bytes" in message:
            binary_data = message["bytes"]
            await websocket.send_text(f"Received {len(binary_data)} bytes")
```

### Closing Connections

**Normal Closure**:
```python
from fastapi import FastAPI, WebSocket

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    
    await websocket.send_text("Goodbye!")
    
    # Close connection
    await websocket.close()
```

**Close with Code and Reason**:
```python
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    
    # Close with specific code
    await websocket.close(code=1000, reason="Normal closure")
    
    # Common close codes:
    # 1000 - Normal closure
    # 1001 - Going away
    # 1002 - Protocol error
    # 1003 - Unsupported data
    # 1008 - Policy violation
    # 1011 - Internal server error
```

**Handling Disconnection**:
```python
from fastapi import WebSocketDisconnect

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    
    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"Echo: {data}")
    
    except WebSocketDisconnect:
        print("Client disconnected")
    
    except Exception as e:
        print(f"Error: {e}")
        await websocket.close(code=1011, reason="Internal error")
```

### WebSocket Authentication

**Token-Based Authentication**:
```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
import jwt

app = FastAPI()

SECRET_KEY = "your-secret-key"

def verify_token(token: str) -> dict:
    """Verify JWT token."""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        return payload
    except jwt.InvalidTokenError:
        return None

@app.websocket("/ws")
async def websocket_endpoint(
    websocket: WebSocket,
    token: str = None
):
    # Verify token before accepting
    user_data = verify_token(token) if token else None
    
    if not user_data:
        await websocket.close(code=1008, reason="Authentication required")
        return
    
    await websocket.accept()
    
    await websocket.send_json({
        "type": "welcome",
        "user": user_data
    })
    
    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"User {user_data['username']}: {data}")
    
    except WebSocketDisconnect:
        print(f"User {user_data['username']} disconnected")
```

**Header-Based Authentication**:
```python
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    # Access headers
    headers = dict(websocket.headers)
    auth_header = headers.get("authorization")
    
    if not auth_header or not auth_header.startswith("Bearer "):
        await websocket.close(code=1008, reason="Invalid auth header")
        return
    
    token = auth_header.replace("Bearer ", "")
    user_data = verify_token(token)
    
    if not user_data:
        await websocket.close(code=1008, reason="Invalid token")
        return
    
    await websocket.accept()
```

---

## 11.3 Real-Time Applications

### Chat Applications

**Simple Chat Room**:
```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import List

app = FastAPI()

# Store active connections
active_connections: List[WebSocket] = []

@app.websocket("/ws/chat")
async def chat_endpoint(websocket: WebSocket, username: str):
    await websocket.accept()
    active_connections.append(websocket)
    
    # Notify others
    for connection in active_connections:
        if connection != websocket:
            await connection.send_json({
                "type": "user_joined",
                "username": username
            })
    
    try:
        while True:
            # Receive message
            message = await websocket.receive_text()
            
            # Broadcast to all
            for connection in active_connections:
                await connection.send_json({
                    "type": "message",
                    "username": username,
                    "message": message
                })
    
    except WebSocketDisconnect:
        active_connections.remove(websocket)
        
        # Notify others
        for connection in active_connections:
            await connection.send_json({
                "type": "user_left",
                "username": username
            })
```

**Client Example (JavaScript)**:
```javascript
// Connect to WebSocket
const ws = new WebSocket("ws://localhost:8000/ws/chat?username=Alice");

ws.onopen = () => {
    console.log("Connected to chat");
};

ws.onmessage = (event) => {
    const data = JSON.parse(event.data);
    
    if (data.type === "message") {
        console.log(`${data.username}: ${data.message}`);
    } else if (data.type === "user_joined") {
        console.log(`${data.username} joined`);
    } else if (data.type === "user_left") {
        console.log(`${data.username} left`);
    }
};

// Send message
function sendMessage(message) {
    ws.send(message);
}
```

### Live Notifications

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import Dict
import asyncio

app = FastAPI()

# User connections: user_id -> WebSocket
user_connections: Dict[int, WebSocket] = {}

@app.websocket("/ws/notifications/{user_id}")
async def notification_endpoint(websocket: WebSocket, user_id: int):
    await websocket.accept()
    user_connections[user_id] = websocket
    
    try:
        # Send initial connection message
        await websocket.send_json({
            "type": "connected",
            "message": "Notifications enabled"
        })
        
        # Keep connection alive
        while True:
            # Receive heartbeat or commands
            data = await websocket.receive_json()
            
            if data.get("type") == "ping":
                await websocket.send_json({"type": "pong"})
    
    except WebSocketDisconnect:
        del user_connections[user_id]

# Function to send notification to specific user
async def send_notification(user_id: int, notification: dict):
    """Send notification to specific user."""
    if user_id in user_connections:
        websocket = user_connections[user_id]
        await websocket.send_json({
            "type": "notification",
            **notification
        })

# Example: Trigger notification from another endpoint
@app.post("/notify/{user_id}")
async def notify_user(user_id: int, message: str):
    await send_notification(user_id, {
        "message": message,
        "timestamp": datetime.now().isoformat()
    })
    return {"status": "notification sent"}
```

### Real-Time Dashboards

```python
from fastapi import FastAPI, WebSocket
import asyncio
import random
from datetime import datetime

app = FastAPI()

@app.websocket("/ws/dashboard")
async def dashboard_endpoint(websocket: WebSocket):
    await websocket.accept()
    
    try:
        while True:
            # Generate mock metrics
            metrics = {
                "timestamp": datetime.now().isoformat(),
                "cpu_usage": random.uniform(10, 90),
                "memory_usage": random.uniform(20, 80),
                "requests_per_second": random.randint(100, 1000),
                "active_users": random.randint(50, 500)
            }
            
            # Send metrics
            await websocket.send_json(metrics)
            
            # Update every second
            await asyncio.sleep(1)
    
    except WebSocketDisconnect:
        print("Dashboard client disconnected")
```

**Dashboard Client (JavaScript)**:
```javascript
const ws = new WebSocket("ws://localhost:8000/ws/dashboard");

ws.onmessage = (event) => {
    const metrics = JSON.parse(event.data);
    
    // Update dashboard UI
    updateChart("cpu", metrics.cpu_usage);
    updateChart("memory", metrics.memory_usage);
    updateCounter("rps", metrics.requests_per_second);
    updateCounter("users", metrics.active_users);
};
```

### Collaborative Editing

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import Dict, List
import json

app = FastAPI()

# Document sessions: doc_id -> List[WebSocket]
document_sessions: Dict[str, List[WebSocket]] = {}

# Document content: doc_id -> content
documents: Dict[str, str] = {}

@app.websocket("/ws/document/{doc_id}")
async def collaborative_edit(websocket: WebSocket, doc_id: str, user_id: str):
    await websocket.accept()
    
    # Initialize document if needed
    if doc_id not in documents:
        documents[doc_id] = ""
    
    if doc_id not in document_sessions:
        document_sessions[doc_id] = []
    
    document_sessions[doc_id].append(websocket)
    
    # Send current document content
    await websocket.send_json({
        "type": "init",
        "content": documents[doc_id]
    })
    
    # Notify others
    for connection in document_sessions[doc_id]:
        if connection != websocket:
            await connection.send_json({
                "type": "user_joined",
                "user_id": user_id
            })
    
    try:
        while True:
            data = await websocket.receive_json()
            
            if data["type"] == "edit":
                # Update document
                documents[doc_id] = data["content"]
                
                # Broadcast change to others
                for connection in document_sessions[doc_id]:
                    if connection != websocket:
                        await connection.send_json({
                            "type": "edit",
                            "user_id": user_id,
                            "content": data["content"],
                            "cursor_position": data.get("cursor_position")
                        })
    
    except WebSocketDisconnect:
        document_sessions[doc_id].remove(websocket)
        
        # Notify others
        for connection in document_sessions[doc_id]:
            await connection.send_json({
                "type": "user_left",
                "user_id": user_id
            })
```

### Live Data Streaming

```python
from fastapi import FastAPI, WebSocket
import asyncio
import random

app = FastAPI()

async def generate_stock_data():
    """Generate mock stock price data."""
    price = 100.0
    
    while True:
        # Random price movement
        change = random.uniform(-2, 2)
        price += change
        
        yield {
            "symbol": "AAPL",
            "price": round(price, 2),
            "change": round(change, 2),
            "volume": random.randint(1000, 10000)
        }
        
        await asyncio.sleep(0.5)

@app.websocket("/ws/stocks")
async def stock_stream(websocket: WebSocket):
    await websocket.accept()
    
    try:
        async for data in generate_stock_data():
            await websocket.send_json(data)
    
    except WebSocketDisconnect:
        print("Client disconnected from stock stream")
```

---

## 11.4 WebSocket Broadcasting

### Connection Manager

**Centralized Connection Management**:
```python
from fastapi import WebSocket
from typing import List, Dict
import json

class ConnectionManager:
    def __init__(self):
        # List of active connections
        self.active_connections: List[WebSocket] = []
    
    async def connect(self, websocket: WebSocket):
        """Accept and store connection."""
        await websocket.accept()
        self.active_connections.append(websocket)
    
    def disconnect(self, websocket: WebSocket):
        """Remove connection."""
        self.active_connections.remove(websocket)
    
    async def send_personal_message(self, message: str, websocket: WebSocket):
        """Send message to specific connection."""
        await websocket.send_text(message)
    
    async def broadcast(self, message: str):
        """Send message to all connections."""
        for connection in self.active_connections:
            await connection.send_text(message)
    
    async def broadcast_json(self, data: dict):
        """Broadcast JSON to all connections."""
        for connection in self.active_connections:
            await connection.send_json(data)

# Create manager instance
manager = ConnectionManager()

# Use in endpoint
from fastapi import FastAPI, WebSocketDisconnect

app = FastAPI()

@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: int):
    await manager.connect(websocket)
    
    try:
        while True:
            data = await websocket.receive_text()
            
            # Send to sender
            await manager.send_personal_message(
                f"You wrote: {data}",
                websocket
            )
            
            # Broadcast to all
            await manager.broadcast(f"Client #{client_id} says: {data}")
    
    except WebSocketDisconnect:
        manager.disconnect(websocket)
        await manager.broadcast(f"Client #{client_id} left")
```

### Broadcasting to Multiple Clients

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import List

app = FastAPI()

class Broadcaster:
    def __init__(self):
        self.connections: List[WebSocket] = []
    
    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.connections.append(websocket)
    
    def disconnect(self, websocket: WebSocket):
        self.connections.remove(websocket)
    
    async def broadcast(self, message: dict, exclude: WebSocket = None):
        """Broadcast to all except excluded connection."""
        disconnected = []
        
        for connection in self.connections:
            if connection == exclude:
                continue
            
            try:
                await connection.send_json(message)
            except Exception:
                disconnected.append(connection)
        
        # Remove dead connections
        for connection in disconnected:
            self.connections.remove(connection)
    
    @property
    def connection_count(self):
        return len(self.connections)

broadcaster = Broadcaster()

@app.websocket("/ws/broadcast")
async def broadcast_endpoint(websocket: WebSocket, username: str):
    await broadcaster.connect(websocket)
    
    # Announce join
    await broadcaster.broadcast(
        {"type": "join", "username": username, "count": broadcaster.connection_count},
        exclude=websocket
    )
    
    try:
        while True:
            data = await websocket.receive_json()
            
            # Broadcast message
            await broadcaster.broadcast({
                "type": "message",
                "username": username,
                "message": data.get("message")
            })
    
    except WebSocketDisconnect:
        broadcaster.disconnect(websocket)
        await broadcaster.broadcast({
            "type": "leave",
            "username": username,
            "count": broadcaster.connection_count
        })
```

### Room-Based Messaging

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import Dict, List

app = FastAPI()

class RoomManager:
    def __init__(self):
        # room_id -> List[WebSocket]
        self.rooms: Dict[str, List[WebSocket]] = {}
        # websocket -> room_id
        self.user_rooms: Dict[WebSocket, str] = {}
    
    async def join_room(self, room_id: str, websocket: WebSocket):
        """Add user to room."""
        await websocket.accept()
        
        if room_id not in self.rooms:
            self.rooms[room_id] = []
        
        self.rooms[room_id].append(websocket)
        self.user_rooms[websocket] = room_id
    
    def leave_room(self, websocket: WebSocket):
        """Remove user from room."""
        room_id = self.user_rooms.get(websocket)
        
        if room_id and room_id in self.rooms:
            self.rooms[room_id].remove(websocket)
            
            # Clean up empty rooms
            if not self.rooms[room_id]:
                del self.rooms[room_id]
        
        if websocket in self.user_rooms:
            del self.user_rooms[websocket]
    
    async def broadcast_to_room(self, room_id: str, message: dict, exclude: WebSocket = None):
        """Send message to all users in room."""
        if room_id not in self.rooms:
            return
        
        for connection in self.rooms[room_id]:
            if connection != exclude:
                await connection.send_json(message)
    
    def get_room_size(self, room_id: str) -> int:
        """Get number of users in room."""
        return len(self.rooms.get(room_id, []))

room_manager = RoomManager()

@app.websocket("/ws/room/{room_id}")
async def room_endpoint(websocket: WebSocket, room_id: str, username: str):
    await room_manager.join_room(room_id, websocket)
    
    # Notify room
    await room_manager.broadcast_to_room(room_id, {
        "type": "user_joined",
        "username": username,
        "room_size": room_manager.get_room_size(room_id)
    }, exclude=websocket)
    
    try:
        while True:
            data = await websocket.receive_json()
            
            # Broadcast to room
            await room_manager.broadcast_to_room(room_id, {
                "type": "message",
                "username": username,
                "message": data.get("message")
            })
    
    except WebSocketDisconnect:
        room_manager.leave_room(websocket)
        
        await room_manager.broadcast_to_room(room_id, {
            "type": "user_left",
            "username": username,
            "room_size": room_manager.get_room_size(room_id)
        })
```

### Private Messaging

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import Dict

app = FastAPI()

class PrivateMessageManager:
    def __init__(self):
        # user_id -> WebSocket
        self.connections: Dict[str, WebSocket] = {}
    
    async def connect(self, user_id: str, websocket: WebSocket):
        await websocket.accept()
        self.connections[user_id] = websocket
    
    def disconnect(self, user_id: str):
        if user_id in self.connections:
            del self.connections[user_id]
    
    async def send_private_message(
        self,
        from_user: str,
        to_user: str,
        message: str
    ):
        """Send private message between users."""
        if to_user in self.connections:
            await self.connections[to_user].send_json({
                "type": "private_message",
                "from": from_user,
                "message": message
            })
            return True
        return False
    
    def is_user_online(self, user_id: str) -> bool:
        return user_id in self.connections

pm_manager = PrivateMessageManager()

@app.websocket("/ws/private/{user_id}")
async def private_message_endpoint(websocket: WebSocket, user_id: str):
    await pm_manager.connect(user_id, websocket)
    
    try:
        while True:
            data = await websocket.receive_json()
            
            to_user = data.get("to")
            message = data.get("message")
            
            # Send private message
            sent = await pm_manager.send_private_message(user_id, to_user, message)
            
            # Send delivery confirmation
            await websocket.send_json({
                "type": "delivery_status",
                "to": to_user,
                "delivered": sent
            })
    
    except WebSocketDisconnect:
        pm_manager.disconnect(user_id)

# Check if user is online
@app.get("/user/{user_id}/online")
async def check_user_online(user_id: str):
    return {"online": pm_manager.is_user_online(user_id)}
```

### Connection Pooling

```python
from fastapi import FastAPI, WebSocket
import asyncio
from typing import Dict, Set

app = FastAPI()

class ConnectionPool:
    def __init__(self, max_connections_per_user: int = 3):
        self.max_connections_per_user = max_connections_per_user
        # user_id -> Set[WebSocket]
        self.user_connections: Dict[str, Set[WebSocket]] = {}
    
    async def add_connection(self, user_id: str, websocket: WebSocket) -> bool:
        """Add connection with limit check."""
        if user_id not in self.user_connections:
            self.user_connections[user_id] = set()
        
        # Check limit
        if len(self.user_connections[user_id]) >= self.max_connections_per_user:
            return False
        
        await websocket.accept()
        self.user_connections[user_id].add(websocket)
        return True
    
    def remove_connection(self, user_id: str, websocket: WebSocket):
        """Remove connection."""
        if user_id in self.user_connections:
            self.user_connections[user_id].discard(websocket)
            
            if not self.user_connections[user_id]:
                del self.user_connections[user_id]
    
    async def send_to_user(self, user_id: str, message: dict):
        """Send to all user's connections."""
        if user_id in self.user_connections:
            for connection in self.user_connections[user_id]:
                await connection.send_json(message)
    
    def get_user_connection_count(self, user_id: str) -> int:
        return len(self.user_connections.get(user_id, set()))

pool = ConnectionPool(max_connections_per_user=3)

@app.websocket("/ws/pooled/{user_id}")
async def pooled_endpoint(websocket: WebSocket, user_id: str):
    # Try to add connection
    added = await pool.add_connection(user_id, websocket)
    
    if not added:
        await websocket.close(code=1008, reason="Max connections reached")
        return
    
    try:
        # Send connection info
        await websocket.send_json({
            "type": "connected",
            "active_connections": pool.get_user_connection_count(user_id)
        })
        
        while True:
            data = await websocket.receive_json()
            
            # Echo to all user's connections
            await pool.send_to_user(user_id, {
                "type": "sync",
                "data": data
            })
    
    except WebSocketDisconnect:
        pool.remove_connection(user_id, websocket)
```

---

## 11.5 Server-Sent Events (SSE)

### SSE vs WebSockets

| Feature | SSE | WebSocket |
|---------|-----|-----------|
| **Direction** | Server → Client only | Bidirectional |
| **Protocol** | HTTP | WebSocket protocol |
| **Reconnection** | Automatic | Manual |
| **Browser Support** | Wide (EventSource API) | Wide |
| **Complexity** | Simple | More complex |
| **Use Case** | Live updates, feeds | Interactive communication |
| **Data Format** | Text only | Text or binary |

**When to Use SSE**:
- Live news feeds
- Stock price updates
- Notifications
- Progress updates
- Log streaming
- One-way data push from server

**When to Use WebSockets**:
- Chat applications
- Online gaming
- Collaborative editing
- Two-way communication needed

### Implementing SSE

**Basic SSE Endpoint**:
```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import asyncio

app = FastAPI()

async def event_generator():
    """Generate server-sent events."""
    count = 0
    
    while True:
        count += 1
        
        # SSE format: data: <message>\n\n
        yield f"data: Message {count}\n\n"
        
        await asyncio.sleep(1)

@app.get("/sse")
async def sse_endpoint():
    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream"
    )
```

**Client (JavaScript)**:
```javascript
const eventSource = new EventSource("http://localhost:8000/sse");

eventSource.onmessage = (event) => {
    console.log("Received:", event.data);
};

eventSource.onerror = (error) => {
    console.error("SSE error:", error);
    eventSource.close();
};
```

### Streaming Data with SSE

**JSON Data Streaming**:
```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import asyncio
import json
from datetime import datetime

app = FastAPI()

async def json_event_stream():
    """Stream JSON data as SSE."""
    while True:
        data = {
            "timestamp": datetime.now().isoformat(),
            "value": random.randint(1, 100)
        }
        
        # Format as SSE
        yield f"data: {json.dumps(data)}\n\n"
        
        await asyncio.sleep(1)

@app.get("/sse/json")
async def json_sse():
    return StreamingResponse(
        json_event_stream(),
        media_type="text/event-stream"
    )
```

**Named Events**:
```python
async def named_event_stream():
    """Stream different event types."""
    count = 0
    
    while True:
        count += 1
        
        if count % 5 == 0:
            # Custom event name
            yield f"event: milestone\n"
            yield f"data: Reached count {count}\n\n"
        else:
            # Default message event
            yield f"data: Count: {count}\n\n"
        
        await asyncio.sleep(1)

@app.get("/sse/events")
async def named_events():
    return StreamingResponse(
        named_event_stream(),
        media_type="text/event-stream"
    )
```

**Client for Named Events**:
```javascript
const eventSource = new EventSource("http://localhost:8000/sse/events");

// Listen to default messages
eventSource.onmessage = (event) => {
    console.log("Message:", event.data);
};

// Listen to custom event
eventSource.addEventListener("milestone", (event) => {
    console.log("Milestone:", event.data);
});
```

**With Event IDs (for reconnection)**:
```python
async def event_stream_with_id():
    """Stream with event IDs for reconnection."""
    event_id = 0
    
    while True:
        event_id += 1
        
        yield f"id: {event_id}\n"
        yield f"data: Event {event_id}\n\n"
        
        await asyncio.sleep(1)

@app.get("/sse/with-id")
async def sse_with_id():
    return StreamingResponse(
        event_stream_with_id(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no"  # Disable nginx buffering
        }
    )
```

### SSE Use Cases

**1. Live Progress Updates**:
```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import asyncio

app = FastAPI()

async def process_with_progress(task_id: str):
    """Simulate long task with progress."""
    for progress in range(0, 101, 10):
        yield f"data: {json.dumps({'task_id': task_id, 'progress': progress})}\n\n"
        await asyncio.sleep(0.5)
    
    yield f"data: {json.dumps({'task_id': task_id, 'status': 'complete'})}\n\n"

@app.get("/sse/progress/{task_id}")
async def progress_stream(task_id: str):
    return StreamingResponse(
        process_with_progress(task_id),
        media_type="text/event-stream"
    )
```

**2. Log Streaming**:
```python
import asyncio

async def stream_logs(service: str):
    """Stream service logs."""
    log_file = f"/var/log/{service}.log"
    
    # Simulated log streaming
    while True:
        log_entry = {
            "service": service,
            "timestamp": datetime.now().isoformat(),
            "level": "INFO",
            "message": f"Log entry from {service}"
        }
        
        yield f"data: {json.dumps(log_entry)}\n\n"
        await asyncio.sleep(2)

@app.get("/sse/logs/{service}")
async def log_stream(service: str):
    return StreamingResponse(
        stream_logs(service),
        media_type="text/event-stream"
    )
```

**3. Stock Price Feed**:
```python
import random

async def stock_price_feed(symbol: str):
    """Stream stock prices."""
    price = 100.0
    
    while True:
        # Random price change
        change = random.uniform(-2, 2)
        price = max(0.01, price + change)
        
        data = {
            "symbol": symbol,
            "price": round(price, 2),
            "change": round(change, 2),
            "timestamp": datetime.now().isoformat()
        }
        
        yield f"data: {json.dumps(data)}\n\n"
        await asyncio.sleep(1)

@app.get("/sse/stock/{symbol}")
async def stock_stream(symbol: str):
    return StreamingResponse(
        stock_price_feed(symbol),
        media_type="text/event-stream"
    )
```

**4. Notification Feed**:
```python
from typing import Dict
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import asyncio
import queue

app = FastAPI()

# Notification queues: user_id -> queue
notification_queues: Dict[str, asyncio.Queue] = {}

async def notification_stream(user_id: str):
    """Stream notifications for user."""
    # Create queue for this user
    if user_id not in notification_queues:
        notification_queues[user_id] = asyncio.Queue()
    
    user_queue = notification_queues[user_id]
    
    try:
        while True:
            # Wait for notification
            notification = await user_queue.get()
            
            yield f"data: {json.dumps(notification)}\n\n"
    
    finally:
        # Cleanup
        if user_id in notification_queues:
            del notification_queues[user_id]

@app.get("/sse/notifications/{user_id}")
async def notifications(user_id: str):
    return StreamingResponse(
        notification_stream(user_id),
        media_type="text/event-stream"
    )

# Endpoint to send notification
@app.post("/notify/{user_id}")
async def send_notification(user_id: str, message: str):
    if user_id in notification_queues:
        notification = {
            "message": message,
            "timestamp": datetime.now().isoformat()
        }
        await notification_queues[user_id].put(notification)
        return {"status": "sent"}
    return {"status": "user not connected"}
```

---

## Complete Example: Real-Time Chat Application

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from fastapi.responses import HTMLResponse, StreamingResponse
from typing import Dict, List, Set
import json
from datetime import datetime
import asyncio

app = FastAPI(title="Real-Time Chat")

# ========== Connection Management ==========
class ChatRoom:
    def __init__(self):
        # room_id -> Set[WebSocket]
        self.rooms: Dict[str, Set[WebSocket]] = {}
        # websocket -> (room_id, username)
        self.connections: Dict[WebSocket, tuple] = {}
        # room_id -> List[message]
        self.message_history: Dict[str, List[dict]] = {}
    
    async def join(self, room_id: str, username: str, websocket: WebSocket):
        """User joins room."""
        await websocket.accept()
        
        if room_id not in self.rooms:
            self.rooms[room_id] = set()
            self.message_history[room_id] = []
        
        self.rooms[room_id].add(websocket)
        self.connections[websocket] = (room_id, username)
        
        # Send message history
        await websocket.send_json({
            "type": "history",
            "messages": self.message_history[room_id]
        })
        
        # Announce join
        await self.broadcast(room_id, {
            "type": "system",
            "message": f"{username} joined the room",
            "timestamp": datetime.now().isoformat(),
            "users_count": len(self.rooms[room_id])
        })
    
    def leave(self, websocket: WebSocket):
        """User leaves room."""
        if websocket not in self.connections:
            return
        
        room_id, username = self.connections[websocket]
        
        self.rooms[room_id].discard(websocket)
        del self.connections[websocket]
        
        # Announce leave
        asyncio.create_task(self.broadcast(room_id, {
            "type": "system",
            "message": f"{username} left the room",
            "timestamp": datetime.now().isoformat(),
            "users_count": len(self.rooms[room_id])
        }))
    
    async def broadcast(self, room_id: str, message: dict):
        """Broadcast message to room."""
        if room_id not in self.rooms:
            return
        
        # Save to history if it's a user message
        if message.get("type") == "message":
            self.message_history[room_id].append(message)
            
            # Keep only last 100 messages
            if len(self.message_history[room_id]) > 100:
                self.message_history[room_id] = self.message_history[room_id][-100:]
        
        # Send to all connections
        disconnected = []
        for websocket in self.rooms[room_id]:
            try:
                await websocket.send_json(message)
            except:
                disconnected.append(websocket)
        
        # Remove disconnected
        for websocket in disconnected:
            self.leave(websocket)
    
    async def send_private(self, room_id: str, from_username: str, to_username: str, message: str):
        """Send private message."""
        for websocket, (r_id, username) in self.connections.items():
            if r_id == room_id and username == to_username:
                await websocket.send_json({
                    "type": "private",
                    "from": from_username,
                    "message": message,
                    "timestamp": datetime.now().isoformat()
                })
                return True
        return False

chat_room = ChatRoom()

# ========== WebSocket Endpoints ==========
@app.websocket("/ws/chat/{room_id}")
async def chat_endpoint(websocket: WebSocket, room_id: str, username: str):
    await chat_room.join(room_id, username, websocket)
    
    try:
        while True:
            data = await websocket.receive_json()
            message_type = data.get("type")
            
            if message_type == "message":
                # Public message
                await chat_room.broadcast(room_id, {
                    "type": "message",
                    "username": username,
                    "message": data.get("message"),
                    "timestamp": datetime.now().isoformat()
                })
            
            elif message_type == "private":
                # Private message
                sent = await chat_room.send_private(
                    room_id,
                    username,
                    data.get("to"),
                    data.get("message")
                )
                
                await websocket.send_json({
                    "type": "delivery",
                    "sent": sent
                })
            
            elif message_type == "typing":
                # Typing indicator
                await chat_room.broadcast(room_id, {
                    "type": "typing",
                    "username": username
                })
    
    except WebSocketDisconnect:
        chat_room.leave(websocket)

# ========== SSE for Announcements ==========
announcement_queue = asyncio.Queue()

async def announcement_stream():
    """Stream system announcements."""
    while True:
        announcement = await announcement_queue.get()
        yield f"data: {json.dumps(announcement)}\n\n"

@app.get("/sse/announcements")
async def announcements():
    return StreamingResponse(
        announcement_stream(),
        media_type="text/event-stream"
    )

@app.post("/announce")
async def make_announcement(message: str):
    """Send system-wide announcement."""
    await announcement_queue.put({
        "message": message,
        "timestamp": datetime.now().isoformat()
    })
    return {"status": "announced"}

# ========== HTML Client ==========
@app.get("/")
async def get_client():
    return HTMLResponse("""
    <!DOCTYPE html>
    <html>
    <head>
        <title>Chat Room</title>
        <style>
            body { font-family: Arial, sans-serif; max-width: 800px; margin: 50px auto; }
            #messages { border: 1px solid #ccc; height: 400px; overflow-y: scroll; padding: 10px; }
            .message { margin: 5px 0; }
            .system { color: #666; font-style: italic; }
            .private { background: #ffffcc; padding: 5px; }
            input { width: 70%; padding: 10px; }
            button { padding: 10px 20px; }
        </style>
    </head>
    <body>
        <h1>Chat Room</h1>
        <div id="messages"></div>
        <input type="text" id="messageInput" placeholder="Type a message...">
        <button onclick="sendMessage()">Send</button>
        
        <script>
            const roomId = "general";
            const username = prompt("Enter your username:");
            const ws = new WebSocket(`ws://localhost:8000/ws/chat/${roomId}?username=${username}`);
            
            ws.onmessage = (event) => {
                const data = JSON.parse(event.data);
                const messagesDiv = document.getElementById('messages');
                const messageEl = document.createElement('div');
                
                if (data.type === 'message') {
                    messageEl.className = 'message';
                    messageEl.textContent = `${data.username}: ${data.message}`;
                } else if (data.type === 'system') {
                    messageEl.className = 'message system';
                    messageEl.textContent = data.message;
                } else if (data.type === 'private') {
                    messageEl.className = 'message private';
                    messageEl.textContent = `[Private from ${data.from}]: ${data.message}`;
                }
                
                messagesDiv.appendChild(messageEl);
                messagesDiv.scrollTop = messagesDiv.scrollHeight;
            };
            
            function sendMessage() {
                const input = document.getElementById('messageInput');
                const message = input.value.trim();
                
                if (message) {
                    ws.send(JSON.stringify({
                        type: 'message',
                        message: message
                    }));
                    input.value = '';
                }
            }
            
            document.getElementById('messageInput').addEventListener('keypress', (e) => {
                if (e.key === 'Enter') sendMessage();
            });
        </script>
    </body>
    </html>
    """)

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## Summary

You've learned about WebSockets & Real-Time communication in FastAPI:

1. **WebSocket Basics**: What WebSockets are, comparison with HTTP, handshake process, and connection lifecycle
2. **WebSocket Implementation**: Creating endpoints, accepting connections, sending/receiving messages, closing connections, and authentication
3. **Real-Time Applications**: Chat apps, live notifications, dashboards, collaborative editing, and data streaming
4. **WebSocket Broadcasting**: Connection managers, broadcasting to multiple clients, room-based messaging, private messaging, and connection pooling
5. **Server-Sent Events (SSE)**: SSE vs WebSockets, implementation, data streaming, and use cases

**Key Takeaways**:
- Use **WebSockets** for bidirectional, interactive communication
- Use **SSE** for one-way server-to-client updates
- Implement proper **connection management** for scalability
- Handle **disconnections** gracefully
- Consider **authentication** and **authorization** for secure real-time apps

WebSockets and SSE enable powerful real-time features in your FastAPI applications!