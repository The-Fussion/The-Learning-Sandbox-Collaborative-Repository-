# WebSockets in Python

## Table of Contents
1. [Introduction to WebSockets](#introduction-to-websockets)
2. [WebSockets vs HTTP](#websockets-vs-http)
3. [WebSockets Protocol](#websockets-protocol)
4. [WebSockets with Flask-SocketIO](#websockets-with-flask-socketio)
5. [WebSockets with FastAPI](#websockets-with-fastapi)
6. [WebSockets with Django Channels](#websockets-with-django-channels)
7. [Native Python WebSockets](#native-python-websockets)
8. [Building a Chat Application](#building-a-chat-application)
9. [Real-Time Notifications](#real-time-notifications)
10. [Live Data Streaming](#live-data-streaming)
11. [Collaborative Features](#collaborative-features)
12. [Broadcasting and Rooms](#broadcasting-and-rooms)
13. [Authentication and Security](#authentication-and-security)
14. [Scaling WebSockets](#scaling-websockets)
15. [Testing WebSockets](#testing-websockets)
16. [Best Practices](#best-practices)

---

## Introduction to WebSockets

### What are WebSockets?

```python
"""
WebSockets provide full-duplex communication channels over a single 
TCP connection. Unlike HTTP, where the client must initiate every 
request, WebSockets allow both client and server to send messages 
independently at any time.

Key Features:
1. Bidirectional: Both client and server can send messages
2. Real-time: Low latency communication
3. Persistent: Connection stays open
4. Efficient: No HTTP overhead for each message
5. Event-driven: Push notifications from server

Use Cases:
- Chat applications
- Live notifications
- Real-time dashboards
- Collaborative editing
- Online gaming
- Live sports scores
- Stock market tickers
- IoT device communication
"""
```

### WebSocket Lifecycle

```python
"""
WebSocket Connection Lifecycle:

1. HANDSHAKE
   Client → Server: HTTP Upgrade request
   Server → Client: HTTP 101 Switching Protocols
   
2. OPEN CONNECTION
   - Bidirectional communication begins
   - Both sides can send/receive messages
   
3. MESSAGE EXCHANGE
   Client ↔ Server: Text or binary frames
   
4. CLOSE
   Either side: Close frame
   Both sides: Acknowledge and close TCP connection

Example Flow:
┌────────┐                          ┌────────┐
│ Client │                          │ Server │
└────────┘                          └────────┘
     │                                   │
     │  HTTP Upgrade (WebSocket)         │
     │──────────────────────────────────>│
     │                                   │
     │  101 Switching Protocols          │
     │<──────────────────────────────────│
     │                                   │
     │  ══════ WebSocket Open ══════     │
     │                                   │
     │  Message: "Hello"                 │
     │──────────────────────────────────>│
     │                                   │
     │  Message: "Hello, Client!"        │
     │<──────────────────────────────────│
     │                                   │
     │  Message: "How are you?"          │
     │──────────────────────────────────>│
     │                                   │
     │  Close Frame                      │
     │──────────────────────────────────>│
     │                                   │
     │  Close Acknowledgment             │
     │<──────────────────────────────────│
     │                                   │
"""
```

---

## WebSockets vs HTTP

### Comparison Table

```python
"""
┌──────────────────┬─────────────────────┬──────────────────────┐
│    Feature       │        HTTP         │     WebSocket        │
├──────────────────┼─────────────────────┼──────────────────────┤
│ Communication    │   Unidirectional    │   Bidirectional      │
│ Pattern          │   Request/Response  │   Full-duplex        │
│ Connection       │   New per request   │   Persistent         │
│ Protocol         │   HTTP/1.1, HTTP/2  │   WebSocket (ws://)  │
│ Overhead         │   High (headers)    │   Low (frames)       │
│ Real-time        │   Polling required  │   Native support     │
│ Server Push      │   No (HTTP/2 yes)   │   Yes                │
│ Stateful         │   Stateless         │   Stateful           │
│ Firewall         │   Easy              │   Sometimes blocked  │
│ Use Case         │   RESTful APIs      │   Real-time apps     │
└──────────────────┴─────────────────────┴──────────────────────┘

HTTP Request/Response:
Client: "GET /data" → Server: "200 OK {data}"
Client: "GET /data" → Server: "200 OK {data}"
Client: "GET /data" → Server: "200 OK {data}"

WebSocket:
Client: Connect → Server: Accept
Client ↔ Server: Message exchange (anytime)
Either: Close → Both: Close
"""
```

### When to Use WebSockets

```python
"""
✅ Use WebSockets When:
- Real-time updates are critical (chat, games)
- High-frequency data exchange (live dashboards)
- Server needs to push data without client request
- Low latency is important
- Bidirectional communication is needed

❌ Don't Use WebSockets When:
- Simple request/response is sufficient
- Caching is important
- SEO is required
- Firewall/proxy compatibility is critical
- Stateless architecture is preferred
- RESTful API is adequate
"""
```

---

## WebSockets Protocol

### Handshake Process

```python
"""
1. Client sends HTTP Upgrade request:

GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

2. Server responds with:

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

3. Connection upgraded to WebSocket protocol
"""
```

### Frame Structure

```python
"""
WebSocket Frame Format:

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
:                     Payload Data continued ...                :
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                     Payload Data continued ...                |
+---------------------------------------------------------------+

Opcodes:
0x0: Continuation frame
0x1: Text frame
0x2: Binary frame
0x8: Close frame
0x9: Ping frame
0xA: Pong frame
"""
```

---

## WebSockets with Flask-SocketIO

### Installation

```bash
pip install flask-socketio python-socketio
```

### Basic Flask-SocketIO Server

```python
from flask import Flask, render_template, request
from flask_socketio import SocketIO, emit, join_room, leave_room, rooms
import time

app = Flask(__name__)
app.config['SECRET_KEY'] = 'your-secret-key'

# Initialize SocketIO
socketio = SocketIO(app, cors_allowed_origins="*")

# Store connected clients
connected_clients = {}

@app.route('/')
def index():
    """Serve the main page."""
    return render_template('index.html')

# ==================== Connection Events ====================

@socketio.on('connect')
def handle_connect():
    """Handle client connection."""
    client_id = request.sid
    
    print(f'Client connected: {client_id}')
    
    # Send welcome message to the connected client
    emit('message', {
        'type': 'system',
        'message': f'Welcome! Your ID is {client_id}',
        'timestamp': time.time()
    })
    
    # Broadcast to all other clients
    emit('user_count', {
        'count': len(connected_clients) + 1
    }, broadcast=True)

@socketio.on('disconnect')
def handle_disconnect():
    """Handle client disconnection."""
    client_id = request.sid
    
    print(f'Client disconnected: {client_id}')
    
    # Remove from connected clients
    if client_id in connected_clients:
        username = connected_clients[client_id]
        del connected_clients[client_id]
        
        # Notify others
        emit('user_left', {
            'username': username,
            'count': len(connected_clients)
        }, broadcast=True)

# ==================== Custom Events ====================

@socketio.on('join')
def handle_join(data):
    """Handle user joining with username."""
    client_id = request.sid
    username = data.get('username', 'Anonymous')
    
    # Store username
    connected_clients[client_id] = username
    
    print(f'{username} joined (ID: {client_id})')
    
    # Send confirmation to user
    emit('joined', {
        'username': username,
        'client_id': client_id
    })
    
    # Broadcast to all clients
    emit('user_joined', {
        'username': username,
        'count': len(connected_clients)
    }, broadcast=True)

@socketio.on('message')
def handle_message(data):
    """Handle incoming message."""
    client_id = request.sid
    username = connected_clients.get(client_id, 'Anonymous')
    message = data.get('message', '')
    
    print(f'Message from {username}: {message}')
    
    # Broadcast message to all clients
    emit('message', {
        'type': 'user',
        'username': username,
        'message': message,
        'timestamp': time.time()
    }, broadcast=True)

@socketio.on('typing')
def handle_typing(data):
    """Handle typing indicator."""
    client_id = request.sid
    username = connected_clients.get(client_id, 'Anonymous')
    is_typing = data.get('typing', False)
    
    # Broadcast to all except sender
    emit('user_typing', {
        'username': username,
        'typing': is_typing
    }, broadcast=True, skip_sid=client_id)

@socketio.on('private_message')
def handle_private_message(data):
    """Send private message to specific user."""
    recipient_id = data.get('recipient_id')
    message = data.get('message')
    
    sender_id = request.sid
    sender_username = connected_clients.get(sender_id, 'Anonymous')
    
    # Send to specific recipient
    emit('private_message', {
        'from': sender_username,
        'message': message,
        'timestamp': time.time()
    }, room=recipient_id)

# ==================== Room Management ====================

@socketio.on('create_room')
def handle_create_room(data):
    """Create and join a room."""
    room_name = data.get('room')
    client_id = request.sid
    username = connected_clients.get(client_id, 'Anonymous')
    
    join_room(room_name)
    
    print(f'{username} created and joined room: {room_name}')
    
    emit('joined_room', {
        'room': room_name,
        'username': username
    })
    
    # Notify room members
    emit('user_joined_room', {
        'username': username,
        'room': room_name
    }, room=room_name)

@socketio.on('join_room')
def handle_join_room(data):
    """Join an existing room."""
    room_name = data.get('room')
    client_id = request.sid
    username = connected_clients.get(client_id, 'Anonymous')
    
    join_room(room_name)
    
    print(f'{username} joined room: {room_name}')
    
    emit('joined_room', {
        'room': room_name
    })
    
    # Notify room members
    emit('user_joined_room', {
        'username': username,
        'room': room_name
    }, room=room_name)

@socketio.on('leave_room')
def handle_leave_room(data):
    """Leave a room."""
    room_name = data.get('room')
    client_id = request.sid
    username = connected_clients.get(client_id, 'Anonymous')
    
    leave_room(room_name)
    
    print(f'{username} left room: {room_name}')
    
    # Notify room members
    emit('user_left_room', {
        'username': username,
        'room': room_name
    }, room=room_name)

@socketio.on('room_message')
def handle_room_message(data):
    """Send message to specific room."""
    room_name = data.get('room')
    message = data.get('message')
    client_id = request.sid
    username = connected_clients.get(client_id, 'Anonymous')
    
    emit('room_message', {
        'username': username,
        'message': message,
        'room': room_name,
        'timestamp': time.time()
    }, room=room_name)

# ==================== Error Handling ====================

@socketio.on_error_default
def default_error_handler(e):
    """Handle errors."""
    print(f'WebSocket error: {e}')
    emit('error', {'message': 'An error occurred'})

if __name__ == '__main__':
    socketio.run(app, debug=True, host='0.0.0.0', port=5000)
```

### Flask-SocketIO Client Template

```html
<!-- templates/index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Flask-SocketIO Chat</title>
    <script src="https://cdn.socket.io/4.5.4/socket.io.min.js"></script>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 50px auto;
            padding: 20px;
        }
        #chat-container {
            border: 1px solid #ddd;
            height: 400px;
            overflow-y: scroll;
            padding: 20px;
            margin: 20px 0;
            background: #f9f9f9;
        }
        .message {
            margin: 10px 0;
            padding: 10px;
            border-radius: 5px;
        }
        .message.system {
            background: #e3f2fd;
            color: #1976d2;
        }
        .message.user {
            background: #fff;
            border: 1px solid #ddd;
        }
        .message.own {
            background: #c8e6c9;
            text-align: right;
        }
        .typing-indicator {
            color: #999;
            font-style: italic;
            margin: 5px 0;
        }
        #input-container {
            display: flex;
            gap: 10px;
        }
        #message-input {
            flex: 1;
            padding: 10px;
            border: 1px solid #ddd;
            border-radius: 5px;
        }
        button {
            padding: 10px 20px;
            background: #1976d2;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
        }
        button:hover {
            background: #1565c0;
        }
        #status {
            padding: 10px;
            border-radius: 5px;
            margin-bottom: 20px;
        }
        #status.connected {
            background: #c8e6c9;
            color: #2e7d32;
        }
        #status.disconnected {
            background: #ffcdd2;
            color: #c62828;
        }
    </style>
</head>
<body>
    <h1>Flask-SocketIO Chat</h1>
    
    <div id="status" class="disconnected">Disconnected</div>
    
    <div id="user-setup" style="margin-bottom: 20px;">
        <input type="text" id="username-input" placeholder="Enter your username">
        <button onclick="joinChat()">Join Chat</button>
    </div>
    
    <div id="chat-interface" style="display: none;">
        <div id="user-count">Users online: 0</div>
        <div id="chat-container"></div>
        <div id="typing-indicator" class="typing-indicator"></div>
        <div id="input-container">
            <input type="text" id="message-input" placeholder="Type a message...">
            <button onclick="sendMessage()">Send</button>
        </div>
    </div>
    
    <script>
        // Connect to SocketIO server
        const socket = io.connect('http://localhost:5000');
        
        let currentUsername = '';
        let typingTimeout = null;
        
        // ==================== Connection Events ====================
        
        socket.on('connect', function() {
            console.log('Connected to server');
            document.getElementById('status').textContent = 'Connected';
            document.getElementById('status').className = 'connected';
        });
        
        socket.on('disconnect', function() {
            console.log('Disconnected from server');
            document.getElementById('status').textContent = 'Disconnected';
            document.getElementById('status').className = 'disconnected';
        });
        
        // ==================== Chat Events ====================
        
        socket.on('joined', function(data) {
            currentUsername = data.username;
            document.getElementById('user-setup').style.display = 'none';
            document.getElementById('chat-interface').style.display = 'block';
        });
        
        socket.on('message', function(data) {
            displayMessage(data);
        });
        
        socket.on('user_joined', function(data) {
            displaySystemMessage(`${data.username} joined the chat`);
            updateUserCount(data.count);
        });
        
        socket.on('user_left', function(data) {
            displaySystemMessage(`${data.username} left the chat`);
            updateUserCount(data.count);
        });
        
        socket.on('user_count', function(data) {
            updateUserCount(data.count);
        });
        
        socket.on('user_typing', function(data) {
            if (data.typing) {
                document.getElementById('typing-indicator').textContent = 
                    `${data.username} is typing...`;
            } else {
                document.getElementById('typing-indicator').textContent = '';
            }
        });
        
        // ==================== Functions ====================
        
        function joinChat() {
            const username = document.getElementById('username-input').value.trim();
            if (username) {
                socket.emit('join', { username: username });
            } else {
                alert('Please enter a username');
            }
        }
        
        function sendMessage() {
            const input = document.getElementById('message-input');
            const message = input.value.trim();
            
            if (message) {
                socket.emit('message', { message: message });
                input.value = '';
                
                // Stop typing indicator
                socket.emit('typing', { typing: false });
            }
        }
        
        function displayMessage(data) {
            const container = document.getElementById('chat-container');
            const messageDiv = document.createElement('div');
            messageDiv.className = 'message ' + data.type;
            
            if (data.username === currentUsername) {
                messageDiv.classList.add('own');
            }
            
            if (data.type === 'system') {
                messageDiv.textContent = data.message;
            } else {
                const time = new Date(data.timestamp * 1000).toLocaleTimeString();
                messageDiv.innerHTML = `
                    <strong>${data.username}</strong>
                    <span style="color: #999; font-size: 0.8em;">${time}</span>
                    <div>${data.message}</div>
                `;
            }
            
            container.appendChild(messageDiv);
            container.scrollTop = container.scrollHeight;
        }
        
        function displaySystemMessage(message) {
            displayMessage({
                type: 'system',
                message: message
            });
        }
        
        function updateUserCount(count) {
            document.getElementById('user-count').textContent = 
                `Users online: ${count}`;
        }
        
        // ==================== Event Listeners ====================
        
        // Send message on Enter
        document.getElementById('message-input').addEventListener('keypress', function(e) {
            if (e.key === 'Enter') {
                sendMessage();
            }
        });
        
        // Typing indicator
        document.getElementById('message-input').addEventListener('input', function() {
            socket.emit('typing', { typing: true });
            
            clearTimeout(typingTimeout);
            typingTimeout = setTimeout(function() {
                socket.emit('typing', { typing: false });
            }, 1000);
        });
        
        // Join on Enter in username input
        document.getElementById('username-input').addEventListener('keypress', function(e) {
            if (e.key === 'Enter') {
                joinChat();
            }
        });
    </script>
</body>
</html>
```

---

## WebSockets with FastAPI

### Installation

```bash
pip install fastapi uvicorn websockets
```

### Basic FastAPI WebSocket Server

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from fastapi.responses import HTMLResponse
from typing import List, Dict
import json
from datetime import datetime

app = FastAPI()

# Connection manager
class ConnectionManager:
    """Manage WebSocket connections."""
    
    def __init__(self):
        self.active_connections: List[WebSocket] = []
        self.user_connections: Dict[str, WebSocket] = {}
        self.rooms: Dict[str, List[WebSocket]] = {}
    
    async def connect(self, websocket: WebSocket, user_id: str = None):
        """Accept and store new connection."""
        await websocket.accept()
        self.active_connections.append(websocket)
        
        if user_id:
            self.user_connections[user_id] = websocket
    
    def disconnect(self, websocket: WebSocket, user_id: str = None):
        """Remove connection."""
        if websocket in self.active_connections:
            self.active_connections.remove(websocket)
        
        if user_id and user_id in self.user_connections:
            del self.user_connections[user_id]
        
        # Remove from all rooms
        for room_connections in self.rooms.values():
            if websocket in room_connections:
                room_connections.remove(websocket)
    
    async def send_personal_message(self, message: str, websocket: WebSocket):
        """Send message to specific connection."""
        await websocket.send_text(message)
    
    async def send_to_user(self, message: str, user_id: str):
        """Send message to specific user by ID."""
        if user_id in self.user_connections:
            websocket = self.user_connections[user_id]
            await websocket.send_text(message)
    
    async def broadcast(self, message: str, exclude: WebSocket = None):
        """Broadcast message to all connections."""
        for connection in self.active_connections:
            if connection != exclude:
                await connection.send_text(message)
    
    async def join_room(self, websocket: WebSocket, room: str):
        """Add connection to room."""
        if room not in self.rooms:
            self.rooms[room] = []
        
        if websocket not in self.rooms[room]:
            self.rooms[room].append(websocket)
    
    async def leave_room(self, websocket: WebSocket, room: str):
        """Remove connection from room."""
        if room in self.rooms and websocket in self.rooms[room]:
            self.rooms[room].remove(websocket)
            
            # Clean up empty rooms
            if not self.rooms[room]:
                del self.rooms[room]
    
    async def broadcast_to_room(self, message: str, room: str, exclude: WebSocket = None):
        """Broadcast message to all connections in room."""
        if room in self.rooms:
            for connection in self.rooms[room]:
                if connection != exclude:
                    await connection.send_text(message)

manager = ConnectionManager()

# ==================== HTML Client ====================

@app.get("/")
async def get():
    """Serve HTML client."""
    return HTMLResponse(html_content)

# ==================== WebSocket Endpoints ====================

@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: str):
    """Main WebSocket endpoint."""
    await manager.connect(websocket, client_id)
    
    try:
        # Send welcome message
        await manager.send_personal_message(
            json.dumps({
                'type': 'system',
                'message': f'Welcome! You are connected as {client_id}',
                'timestamp': datetime.now().isoformat()
            }),
            websocket
        )
        
        # Notify others
        await manager.broadcast(
            json.dumps({
                'type': 'user_joined',
                'user_id': client_id,
                'count': len(manager.active_connections)
            }),
            exclude=websocket
        )
        
        # Listen for messages
        while True:
            data = await websocket.receive_text()
            message_data = json.loads(data)
            
            # Handle different message types
            if message_data.get('type') == 'message':
                # Broadcast chat message
                await manager.broadcast(
                    json.dumps({
                        'type': 'message',
                        'user_id': client_id,
                        'message': message_data.get('message'),
                        'timestamp': datetime.now().isoformat()
                    })
                )
            
            elif message_data.get('type') == 'private':
                # Send private message
                recipient = message_data.get('recipient')
                await manager.send_to_user(
                    json.dumps({
                        'type': 'private_message',
                        'from': client_id,
                        'message': message_data.get('message'),
                        'timestamp': datetime.now().isoformat()
                    }),
                    recipient
                )
            
            elif message_data.get('type') == 'typing':
                # Broadcast typing indicator
                await manager.broadcast(
                    json.dumps({
                        'type': 'typing',
                        'user_id': client_id,
                        'typing': message_data.get('typing', False)
                    }),
                    exclude=websocket
                )
    
    except WebSocketDisconnect:
        manager.disconnect(websocket, client_id)
        
        # Notify others
        await manager.broadcast(
            json.dumps({
                'type': 'user_left',
                'user_id': client_id,
                'count': len(manager.active_connections)
            })
        )

@app.websocket("/ws/room/{room_name}/{client_id}")
async def room_websocket(websocket: WebSocket, room_name: str, client_id: str):
    """WebSocket endpoint for rooms."""
    await manager.connect(websocket, client_id)
    await manager.join_room(websocket, room_name)
    
    try:
        # Notify room members
        await manager.broadcast_to_room(
            json.dumps({
                'type': 'user_joined_room',
                'user_id': client_id,
                'room': room_name
            }),
            room_name,
            exclude=websocket
        )
        
        while True:
            data = await websocket.receive_text()
            message_data = json.loads(data)
            
            # Broadcast to room
            await manager.broadcast_to_room(
                json.dumps({
                    'type': 'room_message',
                    'user_id': client_id,
                    'room': room_name,
                    'message': message_data.get('message'),
                    'timestamp': datetime.now().isoformat()
                }),
                room_name
            )
    
    except WebSocketDisconnect:
        await manager.leave_room(websocket, room_name)
        manager.disconnect(websocket, client_id)
        
        # Notify room members
        await manager.broadcast_to_room(
            json.dumps({
                'type': 'user_left_room',
                'user_id': client_id,
                'room': room_name
            }),
            room_name
        )

# ==================== REST API for Stats ====================

@app.get("/api/stats")
async def get_stats():
    """Get connection statistics."""
    return {
        'total_connections': len(manager.active_connections),
        'total_users': len(manager.user_connections),
        'total_rooms': len(manager.rooms),
        'rooms': {
            room: len(connections)
            for room, connections in manager.rooms.items()
        }
    }

# HTML client
html_content = """
<!DOCTYPE html>
<html>
<head>
    <title>FastAPI WebSocket</title>
</head>
<body>
    <h1>FastAPI WebSocket Chat</h1>
    <div>
        <input type="text" id="clientId" placeholder="Your username">
        <button onclick="connect()">Connect</button>
    </div>
    <div id="status">Disconnected</div>
    <div id="messages" style="height: 400px; overflow-y: scroll; border: 1px solid #ccc; margin: 20px 0; padding: 10px;"></div>
    <div>
        <input type="text" id="messageInput" placeholder="Type a message..." style="width: 70%;">
        <button onclick="sendMessage()">Send</button>
    </div>
    
    <script>
        let ws = null;
        let clientId = '';
        
        function connect() {
            clientId = document.getElementById('clientId').value || 'User_' + Date.now();
            ws = new WebSocket(`ws://localhost:8000/ws/${clientId}`);
            
            ws.onopen = function() {
                document.getElementById('status').textContent = 'Connected';
                document.getElementById('status').style.color = 'green';
            };
            
            ws.onmessage = function(event) {
                const data = JSON.parse(event.data);
                displayMessage(data);
            };
            
            ws.onclose = function() {
                document.getElementById('status').textContent = 'Disconnected';
                document.getElementById('status').style.color = 'red';
            };
            
            ws.onerror = function(error) {
                console.error('WebSocket error:', error);
            };
        }
        
        function sendMessage() {
            const input = document.getElementById('messageInput');
            const message = input.value.trim();
            
            if (message && ws) {
                ws.send(JSON.stringify({
                    type: 'message',
                    message: message
                }));
                input.value = '';
            }
        }
        
        function displayMessage(data) {
            const messagesDiv = document.getElementById('messages');
            const messageDiv = document.createElement('div');
            messageDiv.style.margin = '10px 0';
            
            if (data.type === 'system') {
                messageDiv.style.color = 'blue';
                messageDiv.textContent = data.message;
            } else if (data.type === 'message') {
                const time = new Date(data.timestamp).toLocaleTimeString();
                messageDiv.innerHTML = `<strong>${data.user_id}</strong> <small>${time}</small><br>${data.message}`;
            } else if (data.type === 'user_joined') {
                messageDiv.style.color = 'green';
                messageDiv.textContent = `${data.user_id} joined (${data.count} users online)`;
            } else if (data.type === 'user_left') {
                messageDiv.style.color = 'red';
                messageDiv.textContent = `${data.user_id} left (${data.count} users online)`;
            }
            
            messagesDiv.appendChild(messageDiv);
            messagesDiv.scrollTop = messagesDiv.scrollHeight;
        }
        
        document.getElementById('messageInput').addEventListener('keypress', function(e) {
            if (e.key === 'Enter') sendMessage();
        });
    </script>
</body>
</html>
"""

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## WebSockets with Django Channels

### Installation

```bash
pip install channels channels-redis daphne
```

### Django Channels Setup

```python
# settings.py

INSTALLED_APPS = [
    'daphne',  # Add at the top
    'django.contrib.admin',
    'django.contrib.auth',
    # ... other apps
    'channels',
    'chat',  # Your app
]

# ASGI application
ASGI_APPLICATION = 'myproject.asgi.application'

# Channel layers
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {
            "hosts": [('127.0.0.1', 6379)],
        },
    },
}

# Alternative: In-memory channel layer (development only)
# CHANNEL_LAYERS = {
#     'default': {
#         'BACKEND': 'channels.layers.InMemoryChannelLayer'
#     }
# }
```

```python
# myproject/asgi.py

import os
from django.core.asgi import get_asgi_application
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.auth import AuthMiddlewareStack
from channels.security.websocket import AllowedHostsOriginValidator
import chat.routing

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myproject.settings')

application = ProtocolTypeRouter({
    'http': get_asgi_application(),
    'websocket': AllowedHostsOriginValidator(
        AuthMiddlewareStack(
            URLRouter(
                chat.routing.websocket_urlpatterns
            )
        )
    ),
})
```

```python
# chat/routing.py

from django.urls import re_path
from . import consumers

websocket_urlpatterns = [
    re_path(r'ws/chat/(?P<room_name>\w+)/$', consumers.ChatConsumer.as_asgi()),
]
```

```python
# chat/consumers.py

import json
from channels.generic.websocket import AsyncWebsocketConsumer
from channels.db import database_sync_to_async
from django.contrib.auth.models import User
from .models import Message, Room

class ChatConsumer(AsyncWebsocketConsumer):
    """WebSocket consumer for chat."""
    
    async def connect(self):
        """Handle WebSocket connection."""
        self.room_name = self.scope['url_route']['kwargs']['room_name']
        self.room_group_name = f'chat_{self.room_name}'
        self.user = self.scope['user']
        
        # Join room group
        await self.channel_layer.group_add(
            self.room_group_name,
            self.channel_name
        )
        
        # Accept connection
        await self.accept()
        
        # Send welcome message
        await self.send(text_data=json.dumps({
            'type': 'system',
            'message': f'Welcome to room {self.room_name}!'
        }))
        
        # Notify room
        await self.channel_layer.group_send(
            self.room_group_name,
            {
                'type': 'user_join',
                'username': self.user.username if self.user.is_authenticated else 'Anonymous'
            }
        )
    
    async def disconnect(self, close_code):
        """Handle WebSocket disconnection."""
        # Notify room
        await self.channel_layer.group_send(
            self.room_group_name,
            {
                'type': 'user_leave',
                'username': self.user.username if self.user.is_authenticated else 'Anonymous'
            }
        )
        
        # Leave room group
        await self.channel_layer.group_discard(
            self.room_group_name,
            self.channel_name
        )
    
    async def receive(self, text_data):
        """Receive message from WebSocket."""
        data = json.loads(text_data)
        message_type = data.get('type')
        
        if message_type == 'chat_message':
            message = data['message']
            
            # Save message to database
            await self.save_message(message)
            
            # Send message to room group
            await self.channel_layer.group_send(
                self.room_group_name,
                {
                    'type': 'chat_message',
                    'message': message,
                    'username': self.user.username if self.user.is_authenticated else 'Anonymous'
                }
            )
        
        elif message_type == 'typing':
            # Broadcast typing indicator
            await self.channel_layer.group_send(
                self.room_group_name,
                {
                    'type': 'typing_indicator',
                    'username': self.user.username if self.user.is_authenticated else 'Anonymous',
                    'is_typing': data.get('is_typing', False)
                }
            )
    
    async def chat_message(self, event):
        """Receive message from room group."""
        message = event['message']
        username = event['username']
        
        # Send message to WebSocket
        await self.send(text_data=json.dumps({
            'type': 'message',
            'message': message,
            'username': username
        }))
    
    async def user_join(self, event):
        """Handle user join event."""
        await self.send(text_data=json.dumps({
            'type': 'user_joined',
            'username': event['username']
        }))
    
    async def user_leave(self, event):
        """Handle user leave event."""
        await self.send(text_data=json.dumps({
            'type': 'user_left',
            'username': event['username']
        }))
    
    async def typing_indicator(self, event):
        """Handle typing indicator."""
        # Don't send typing indicator back to sender
        if event['username'] != (self.user.username if self.user.is_authenticated else 'Anonymous'):
            await self.send(text_data=json.dumps({
                'type': 'typing',
                'username': event['username'],
                'is_typing': event['is_typing']
            }))
    
    @database_sync_to_async
    def save_message(self, message):
        """Save message to database."""
        room, created = Room.objects.get_or_create(name=self.room_name)
        Message.objects.create(
            room=room,
            user=self.user if self.user.is_authenticated else None,
            content=message
        )
```

```python
# chat/models.py

from django.db import models
from django.contrib.auth.models import User

class Room(models.Model):
    """Chat room."""
    name = models.CharField(max_length=255, unique=True)
    created_at = models.DateTimeField(auto_now_add=True)
    
    def __str__(self):
        return self.name

class Message(models.Model):
    """Chat message."""
    room = models.ForeignKey(Room, on_delete=models.CASCADE, related_name='messages')
    user = models.ForeignKey(User, on_delete=models.CASCADE, null=True, blank=True)
    content = models.TextField()
    timestamp = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        ordering = ['timestamp']
    
    def __str__(self):
        username = self.user.username if self.user else 'Anonymous'
        return f'{username}: {self.content[:50]}'
```

---

## Native Python WebSockets

### Using `websockets` Library

```bash
pip install websockets
```

```python
# server.py

import asyncio
import websockets
import json
from datetime import datetime
from typing import Set

# Connected clients
connected_clients: Set[websockets.WebSocketServerProtocol] = set()

async def handler(websocket: websockets.WebSocketServerProtocol, path: str):
    """Handle WebSocket connection."""
    # Register client
    connected_clients.add(websocket)
    
    try:
        # Send welcome message
        await websocket.send(json.dumps({
            'type': 'system',
            'message': 'Welcome to the chat!',
            'timestamp': datetime.now().isoformat()
        }))
        
        # Broadcast join message
        await broadcast(json.dumps({
            'type': 'user_joined',
            'count': len(connected_clients)
        }), exclude=websocket)
        
        # Listen for messages
        async for message in websocket:
            data = json.loads(message)
            
            # Broadcast message to all clients
            await broadcast(json.dumps({
                'type': 'message',
                'message': data.get('message'),
                'timestamp': datetime.now().isoformat()
            }))
    
    except websockets.exceptions.ConnectionClosed:
        print("Client disconnected")
    
    finally:
        # Unregister client
        connected_clients.remove(websocket)
        
        # Broadcast leave message
        await broadcast(json.dumps({
            'type': 'user_left',
            'count': len(connected_clients)
        }))

async def broadcast(message: str, exclude: websockets.WebSocketServerProtocol = None):
    """Broadcast message to all connected clients."""
    if connected_clients:
        tasks = [
            client.send(message)
            for client in connected_clients
            if client != exclude
        ]
        await asyncio.gather(*tasks, return_exceptions=True)

async def main():
    """Start WebSocket server."""
    print("Starting WebSocket server on ws://localhost:8765")
    
    async with websockets.serve(handler, "localhost", 8765):
        await asyncio.Future()  # Run forever

if __name__ == "__main__":
    asyncio.run(main())
```

```python
# client.py

import asyncio
import websockets
import json

async def receive_messages(websocket):
    """Receive and print messages."""
    async for message in websocket:
        data = json.loads(message)
        
        if data['type'] == 'system':
            print(f"[SYSTEM] {data['message']}")
        elif data['type'] == 'message':
            print(f"[MESSAGE] {data['message']}")
        elif data['type'] == 'user_joined':
            print(f"[INFO] User joined ({data['count']} online)")
        elif data['type'] == 'user_left':
            print(f"[INFO] User left ({data['count']} online)")

async def send_messages(websocket):
    """Send user input as messages."""
    while True:
        message = await asyncio.get_event_loop().run_in_executor(
            None, input, "You: "
        )
        
        if message.lower() == '/quit':
            break
        
        await websocket.send(json.dumps({
            'message': message
        }))

async def main():
    """Connect to WebSocket server."""
    uri = "ws://localhost:8765"
    
    async with websockets.connect(uri) as websocket:
        # Run receive and send concurrently
        await asyncio.gather(
            receive_messages(websocket),
            send_messages(websocket)
        )

if __name__ == "__main__":
    asyncio.run(main())
```

---

## Building a Chat Application

### Complete Chat Application with Flask-SocketIO

```python
# app.py

from flask import Flask, render_template, request, session, redirect, url_for
from flask_socketio import SocketIO, emit, join_room, leave_room
from datetime import datetime
import secrets

app = Flask(__name__)
app.config['SECRET_KEY'] = secrets.token_hex(16)
socketio = SocketIO(app, cors_allowed_origins="*")

# Store users and rooms
users = {}
rooms = {'General': {'users': [], 'messages': []}}

@app.route('/')
def index():
    """Main page."""
    return render_template('chat.html')

@app.route('/login', methods=['GET', 'POST'])
def login():
    """Login page."""
    if request.method == 'POST':
        username = request.form.get('username')
        if username:
            session['username'] = username
            return redirect(url_for('index'))
    
    return render_template('login.html')

@socketio.on('connect')
def handle_connect():
    """Handle connection."""
    username = session.get('username', 'Anonymous')
    users[request.sid] = {
        'username': username,
        'rooms': []
    }
    
    emit('connected', {
        'username': username,
        'sid': request.sid
    })

@socketio.on('disconnect')
def handle_disconnect():
    """Handle disconnection."""
    if request.sid in users:
        username = users[request.sid]['username']
        
        # Leave all rooms
        for room in list(users[request.sid]['rooms']):
            leave_room(room)
            if room in rooms:
                rooms[room]['users'].remove(username)
                
                emit('user_left_room', {
                    'username': username,
                    'room': room,
                    'users': rooms[room]['users']
                }, room=room)
        
        del users[request.sid]

@socketio.on('create_room')
def handle_create_room(data):
    """Create a new room."""
    room_name = data.get('room')
    
    if room_name and room_name not in rooms:
        rooms[room_name] = {
            'users': [],
            'messages': []
        }
        
        emit('room_created', {
            'room': room_name
        }, broadcast=True)

@socketio.on('join_room')
def handle_join_room(data):
    """Join a room."""
    room = data.get('room')
    username = users[request.sid]['username']
    
    if room in rooms:
        join_room(room)
        users[request.sid]['rooms'].append(room)
        rooms[room]['users'].append(username)
        
        # Send room history to user
        emit('room_history', {
            'room': room,
            'messages': rooms[room]['messages']
        })
        
        # Notify room
        emit('user_joined_room', {
            'username': username,
            'room': room,
            'users': rooms[room]['users']
        }, room=room)

@socketio.on('leave_room')
def handle_leave_room(data):
    """Leave a room."""
    room = data.get('room')
    username = users[request.sid]['username']
    
    if room in rooms and username in rooms[room]['users']:
        leave_room(room)
        users[request.sid]['rooms'].remove(room)
        rooms[room]['users'].remove(username)
        
        emit('user_left_room', {
            'username': username,
            'room': room,
            'users': rooms[room]['users']
        }, room=room)

@socketio.on('send_message')
def handle_send_message(data):
    """Send message to room."""
    room = data.get('room')
    message = data.get('message')
    username = users[request.sid]['username']
    
    if room in rooms:
        message_data = {
            'username': username,
            'message': message,
            'timestamp': datetime.now().isoformat()
        }
        
        # Store message
        rooms[room]['messages'].append(message_data)
        
        # Broadcast to room
        emit('new_message', {
            'room': room,
            **message_data
        }, room=room)

@socketio.on('typing')
def handle_typing(data):
    """Handle typing indicator."""
    room = data.get('room')
    username = users[request.sid]['username']
    is_typing = data.get('typing', False)
    
    emit('user_typing', {
        'username': username,
        'room': room,
        'typing': is_typing
    }, room=room, skip_sid=request.sid)

if __name__ == '__main__':
    socketio.run(app, debug=True, host='0.0.0.0', port=5000)
```

---

## Real-Time Notifications

### Notification System with WebSockets

```python
from flask import Flask, jsonify
from flask_socketio import SocketIO, emit
import threading
import time
import random

app = Flask(__name__)
socketio = SocketIO(app)

# Store user subscriptions
user_subscriptions = {}

@socketio.on('connect')
def handle_connect():
    """Handle connection."""
    user_id = request.args.get('user_id')
    user_subscriptions[request.sid] = {
        'user_id': user_id,
        'topics': []
    }
    
    emit('connected', {'user_id': user_id})

@socketio.on('subscribe')
def handle_subscribe(data):
    """Subscribe to notification topics."""
    topic = data.get('topic')
    
    if request.sid in user_subscriptions:
        if topic not in user_subscriptions[request.sid]['topics']:
            user_subscriptions[request.sid]['topics'].append(topic)
    
    emit('subscribed', {'topic': topic})

@socketio.on('unsubscribe')
def handle_unsubscribe(data):
    """Unsubscribe from notification topics."""
    topic = data.get('topic')
    
    if request.sid in user_subscriptions:
        if topic in user_subscriptions[request.sid]['topics']:
            user_subscriptions[request.sid]['topics'].remove(topic)
    
    emit('unsubscribed', {'topic': topic})

def send_notification(topic, notification):
    """Send notification to subscribers."""
    for sid, subscription in user_subscriptions.items():
        if topic in subscription['topics']:
            socketio.emit('notification', {
                'topic': topic,
                'notification': notification
            }, room=sid)

# Simulate notifications
def notification_worker():
    """Background worker to send random notifications."""
    topics = ['orders', 'messages', 'alerts']
    
    while True:
        time.sleep(random.randint(5, 15))
        
        topic = random.choice(topics)
        notification = {
            'title': f'New {topic} notification',
            'message': f'You have a new {topic} update',
            'timestamp': time.time()
        }
        
        send_notification(topic, notification)

# Start background worker
threading.Thread(target=notification_worker, daemon=True).start()

if __name__ == '__main__':
    socketio.run(app, debug=True)
```

```javascript
// Client-side notification handling
const socket = io('http://localhost:5000?user_id=123');

socket.on('connect', function() {
    console.log('Connected to notification server');
    
    // Subscribe to topics
    socket.emit('subscribe', { topic: 'orders' });
    socket.emit('subscribe', { topic: 'messages' });
});

socket.on('notification', function(data) {
    console.log('Notification received:', data);
    
    // Show browser notification
    if ('Notification' in window && Notification.permission === 'granted') {
        new Notification(data.notification.title, {
            body: data.notification.message,
            icon: '/static/icon.png'
        });
    }
    
    // Update UI
    showNotificationBadge(data.topic);
});

// Request notification permission
if ('Notification' in window && Notification.permission === 'default') {
    Notification.requestPermission();
}
```

---

## Live Data Streaming

### Stock Ticker with WebSockets

```python
from fastapi import FastAPI, WebSocket
from fastapi.responses import HTMLResponse
import asyncio
import random
import json

app = FastAPI()

# Mock stock data
stocks = {
    'AAPL': 150.00,
    'GOOGL': 2800.00,
    'MSFT': 300.00,
    'AMZN': 3400.00,
    'TSLA': 700.00
}

async def generate_stock_updates():
    """Generate random stock price updates."""
    while True:
        symbol = random.choice(list(stocks.keys()))
        change = random.uniform(-5, 5)
        stocks[symbol] += change
        
        yield {
            'symbol': symbol,
            'price': round(stocks[symbol], 2),
            'change': round(change, 2)
        }
        
        await asyncio.sleep(1)

@app.websocket("/ws/stocks")
async def stock_ticker(websocket: WebSocket):
    """Stream stock prices."""
    await websocket.accept()
    
    try:
        async for update in generate_stock_updates():
            await websocket.send_json(update)
    except:
        pass

@app.get("/")
async def get():
    return HTMLResponse("""
    <!DOCTYPE html>
    <html>
    <head>
        <title>Live Stock Ticker</title>
        <style>
            .stock { padding: 20px; margin: 10px; border: 1px solid #ddd; }
            .positive { color: green; }
            .negative { color: red; }
        </style>
    </head>
    <body>
        <h1>Live Stock Prices</h1>
        <div id="stocks"></div>
        
        <script>
            const ws = new WebSocket('ws://localhost:8000/ws/stocks');
            const stocksDiv = document.getElementById('stocks');
            const stockElements = {};
            
            ws.onmessage = function(event) {
                const data = JSON.parse(event.data);
                
                if (!stockElements[data.symbol]) {
                    const stockDiv = document.createElement('div');
                    stockDiv.className = 'stock';
                    stockDiv.id = data.symbol;
                    stocksDiv.appendChild(stockDiv);
                    stockElements[data.symbol] = stockDiv;
                }
                
                const changeClass = data.change >= 0 ? 'positive' : 'negative';
                const changeSymbol = data.change >= 0 ? '+' : '';
                
                stockElements[data.symbol].innerHTML = `
                    <h2>${data.symbol}</h2>
                    <p>Price: $${data.price.toFixed(2)}</p>
                    <p class="${changeClass}">Change: ${changeSymbol}${data.change.toFixed(2)}</p>
                `;
            };
        </script>
    </body>
    </html>
    """)
```

---

## Broadcasting and Rooms

### Advanced Room Management

```python
from flask_socketio import SocketIO, emit, join_room, leave_room, rooms
from typing import Dict, Set

class RoomManager:
    """Manage WebSocket rooms."""
    
    def __init__(self):
        self.rooms: Dict[str, Set[str]] = {}
        self.user_rooms: Dict[str, Set[str]] = {}
    
    def create_room(self, room_name: str):
        """Create a new room."""
        if room_name not in self.rooms:
            self.rooms[room_name] = set()
    
    def join_room(self, sid: str, room_name: str):
        """Add user to room."""
        if room_name not in self.rooms:
            self.create_room(room_name)
        
        self.rooms[room_name].add(sid)
        
        if sid not in self.user_rooms:
            self.user_rooms[sid] = set()
        
        self.user_rooms[sid].add(room_name)
    
    def leave_room(self, sid: str, room_name: str):
        """Remove user from room."""
        if room_name in self.rooms and sid in self.rooms[room_name]:
            self.rooms[room_name].remove(sid)
            
            # Clean up empty rooms
            if not self.rooms[room_name]:
                del self.rooms[room_name]
        
        if sid in self.user_rooms and room_name in self.user_rooms[sid]:
            self.user_rooms[sid].remove(room_name)
    
    def get_room_users(self, room_name: str) -> Set[str]:
        """Get all users in room."""
        return self.rooms.get(room_name, set())
    
    def get_user_rooms(self, sid: str) -> Set[str]:
        """Get all rooms user is in."""
        return self.user_rooms.get(sid, set())
    
    def remove_user(self, sid: str):
        """Remove user from all rooms."""
        if sid in self.user_rooms:
            for room_name in list(self.user_rooms[sid]):
                self.leave_room(sid, room_name)
            
            del self.user_rooms[sid]

# Usage
room_manager = RoomManager()

@socketio.on('join')
def handle_join(data):
    """Join room with advanced management."""
    room = data['room']
    
    join_room(room)
    room_manager.join_room(request.sid, room)
    
    users = room_manager.get_room_users(room)
    
    emit('room_users', {
        'room': room,
        'users': list(users)
    }, room=room)
```

---

## Authentication and Security

### JWT Authentication for WebSockets

```python
from flask import Flask, request
from flask_socketio import SocketIO, emit, disconnect
import jwt
from functools import wraps
from datetime import datetime, timedelta

app = Flask(__name__)
app.config['SECRET_KEY'] = 'your-secret-key'
socketio = SocketIO(app)

def create_token(user_id):
    """Create JWT token."""
    payload = {
        'user_id': user_id,
        'exp': datetime.utcnow() + timedelta(hours=24)
    }
    return jwt.encode(payload, app.config['SECRET_KEY'], algorithm='HS256')

def verify_token(token):
    """Verify JWT token."""
    try:
        payload = jwt.decode(token, app.config['SECRET_KEY'], algorithms=['HS256'])
        return payload['user_id']
    except:
        return None

def authenticated_only(f):
    """Decorator to require authentication."""
    @wraps(f)
    def wrapped(*args, **kwargs):
        token = request.args.get('token')
        
        if not token:
            disconnect()
            return
        
        user_id = verify_token(token)
        
        if not user_id:
            disconnect()
            return
        
        return f(*args, user_id=user_id, **kwargs)
    
    return wrapped

@socketio.on('connect')
@authenticated_only
def handle_connect(user_id):
    """Handle authenticated connection."""
    emit('connected', {'user_id': user_id})

@socketio.on('message')
@authenticated_only
def handle_message(data, user_id):
    """Handle authenticated message."""
    emit('message', {
        'user_id': user_id,
        'message': data['message']
    }, broadcast=True)
```

### Rate Limiting WebSockets

```python
from time import time
from collections import defaultdict

class RateLimiter:
    """Rate limiter for WebSocket messages."""
    
    def __init__(self, max_messages=10, window=60):
        self.max_messages = max_messages
        self.window = window
        self.message_times = defaultdict(list)
    
    def is_allowed(self, sid):
        """Check if message is allowed."""
        now = time()
        
        # Remove old messages
        self.message_times[sid] = [
            t for t in self.message_times[sid]
            if now - t < self.window
        ]
        
        # Check rate limit
        if len(self.message_times[sid]) >= self.max_messages:
            return False
        
        # Record message
        self.message_times[sid].append(now)
        return True

rate_limiter = RateLimiter(max_messages=10, window=60)

@socketio.on('message')
def handle_message(data):
    """Handle rate-limited message."""
    if not rate_limiter.is_allowed(request.sid):
        emit('error', {'message': 'Rate limit exceeded'})
        return
    
    # Process message
    emit('message', data, broadcast=True)
```

---

## Scaling WebSockets

### Using Redis for Scaling

```python
# Flask-SocketIO with Redis
from flask import Flask
from flask_socketio import SocketIO

app = Flask(__name__)
app.config['SECRET_KEY'] = 'secret'

# Use Redis for message queue
socketio = SocketIO(
    app,
    message_queue='redis://localhost:6379',
    cors_allowed_origins="*"
)

# Now you can run multiple instances
# All instances will share the same message queue
```

### Load Balancing with Nginx

```nginx
# nginx.conf

upstream websocket_backend {
    ip_hash;  # Important for WebSocket sticky sessions
    server backend1.example.com:5000;
    server backend2.example.com:5000;
    server backend3.example.com:5000;
}

server {
    listen 80;
    server_name example.com;
    
    location / {
        proxy_pass http://websocket_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # Timeouts
        proxy_read_timeout 86400;
        proxy_send_timeout 86400;
    }
}
```

---

## Testing WebSockets

### Unit Testing WebSocket Events

```python
# test_websockets.py

import pytest
from app import app, socketio

@pytest.fixture
def client():
    """Create test client."""
    app.config['TESTING'] = True
    return socketio.test_client(app)

def test_connect(client):
    """Test connection."""
    assert client.is_connected()

def test_message(client):
    """Test sending message."""
    client.emit('message', {'message': 'Hello'})
    
    received = client.get_received()
    assert len(received) > 0
    assert received[0]['args'][0]['message'] == 'Hello'

def test_join_room(client):
    """Test joining room."""
    client.emit('join_room', {'room': 'test'})
    
    received = client.get_received()
    assert any(
        msg['name'] == 'room_joined'
        for msg in received
    )

def test_broadcast(client):
    """Test broadcasting."""
    # Create two clients
    client1 = socketio.test_client(app)
    client2 = socketio.test_client(app)
    
    # Send message from client1
    client1.emit('message', {'message': 'Broadcast test'})
    
    # Check both clients received
    received1 = client1.get_received()
    received2 = client2.get_received()
    
    assert len(received1) > 0
    assert len(received2) > 0

def test_disconnect(client):
    """Test disconnection."""
    client.disconnect()
    assert not client.is_connected()
```

### Integration Testing

```python
import pytest
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

@pytest.fixture
def browser():
    """Create browser instance."""
    driver = webdriver.Chrome()
    yield driver
    driver.quit()

def test_chat_flow(browser):
    """Test complete chat flow."""
    # Open page
    browser.get('http://localhost:5000')
    
    # Enter username
    username_input = browser.find_element(By.ID, 'username')
    username_input.send_keys('TestUser')
    
    join_button = browser.find_element(By.ID, 'join')
    join_button.click()
    
    # Wait for connection
    wait = WebDriverWait(browser, 10)
    status = wait.until(
        EC.text_to_be_present_in_element((By.ID, 'status'), 'Connected')
    )
    
    # Send message
    message_input = browser.find_element(By.ID, 'message')
    message_input.send_keys('Hello, World!')
    
    send_button = browser.find_element(By.ID, 'send')
    send_button.click()
    
    # Verify message appears
    messages = wait.until(
        EC.presence_of_element_located((By.CLASS_NAME, 'message'))
    )
    
    assert 'Hello, World!' in messages.text
```

---

## Best Practices

### WebSocket Best Practices Checklist

```python
"""
✅ WebSocket Best Practices:

1. Connection Management
   ✅ Handle reconnection automatically
   ✅ Implement exponential backoff for retries
   ✅ Clean up resources on disconnect
   ✅ Heartbeat/ping-pong to detect dead connections

2. Message Handling
   ✅ Validate all incoming messages
   ✅ Use JSON for structured data
   ✅ Implement message queuing for offline users
   ✅ Handle large messages appropriately

3. Security
   ✅ Authenticate connections
   ✅ Validate origin
   ✅ Implement rate limiting
   ✅ Sanitize user input
   ✅ Use WSS (WebSocket Secure) in production

4. Performance
   ✅ Use binary frames for large data
   ✅ Compress messages when appropriate
   ✅ Limit broadcast frequency
   ✅ Batch updates when possible

5. Scalability
   ✅ Use Redis or similar for multi-server setups
   ✅ Implement proper load balancing
   ✅ Monitor connection counts
   ✅ Set connection limits

6. Error Handling
   ✅ Handle network errors gracefully
   ✅ Log errors for debugging
   ✅ Provide user feedback
   ✅ Implement fallback mechanisms

7. Testing
   ✅ Unit test event handlers
   ✅ Integration test full flows
   ✅ Load test with realistic scenarios
   ✅ Test reconnection logic
"""
```

### Production-Ready WebSocket Implementation

```python
from flask import Flask
from flask_socketio import SocketIO, emit
import logging
from time import time
from functools import wraps

app = Flask(__name__)
app.config['SECRET_KEY'] = 'production-secret-key'

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Initialize SocketIO with production settings
socketio = SocketIO(
    app,
    cors_allowed_origins="*",
    async_mode='eventlet',
    logger=True,
    engineio_logger=True,
    ping_timeout=60,
    ping_interval=25,
    max_http_buffer_size=1e6,  # 1MB
    message_queue='redis://localhost:6379'
)

# Rate limiting
class RateLimiter:
    def __init__(self, max_requests=100, window=60):
        self.max_requests = max_requests
        self.window = window
        self.requests = {}
    
    def is_allowed(self, sid):
        now = time()
        
        if sid not in self.requests:
            self.requests[sid] = []
        
        # Clean old requests
        self.requests[sid] = [
            t for t in self.requests[sid]
            if now - t < self.window
        ]
        
        if len(self.requests[sid]) >= self.max_requests:
            return False
        
        self.requests[sid].append(now)
        return True

rate_limiter = RateLimiter()

def rate_limit(f):
    """Rate limiting decorator."""
    @wraps(f)
    def wrapped(*args, **kwargs):
        if not rate_limiter.is_allowed(request.sid):
            emit('error', {
                'message': 'Rate limit exceeded. Please slow down.'
            })
            logger.warning(f'Rate limit exceeded for {request.sid}')
            return
        
        return f(*args, **kwargs)
    
    return wrapped

@socketio.on('connect')
def handle_connect():
    """Handle connection with logging."""
    logger.info(f'Client connected: {request.sid}')
    emit('connected', {'sid': request.sid})

@socketio.on('disconnect')
def handle_disconnect():
    """Handle disconnection with cleanup."""
    logger.info(f'Client disconnected: {request.sid}')
    # Clean up resources

@socketio.on('message')
@rate_limit
def handle_message(data):
    """Handle message with validation."""
    try:
        # Validate message
        if not isinstance(data, dict):
            raise ValueError('Invalid message format')
        
        if 'message' not in data:
            raise ValueError('Message field required')
        
        message = data['message']
        
        if len(message) > 1000:
            raise ValueError('Message too long')
        
        # Process message
        emit('message', data, broadcast=True)
        
    except Exception as e:
        logger.error(f'Error handling message: {e}')
        emit('error', {'message': str(e)})

@socketio.on_error_default
def default_error_handler(e):
    """Global error handler."""
    logger.error(f'WebSocket error: {e}', exc_info=True)
    emit('error', {'message': 'An error occurred'})

if __name__ == '__main__':
    socketio.run(
        app,
        host='0.0.0.0',
        port=5000,
        debug=False
    )
```

---

## Summary

This comprehensive guide covered:

1. **Introduction**: WebSocket basics and lifecycle
2. **Protocol**: Handshake process and frame structure
3. **Flask-SocketIO**: Complete implementation with events and rooms
4. **FastAPI**: Modern async WebSocket implementation
5. **Django Channels**: Channel layers and consumers
6. **Native Python**: Using websockets library
7. **Chat Application**: Full-featured chat with rooms
8. **Notifications**: Real-time notification system
9. **Live Streaming**: Stock ticker and data feeds
10. **Broadcasting**: Advanced room management
11. **Authentication**: JWT tokens and rate limiting
12. **Scaling**: Redis and load balancing
13. **Testing**: Unit and integration tests
14. **Best Practices**: Production-ready implementation

Key Takeaways:
- WebSockets enable real-time bidirectional communication
- Choose the right library for your framework
- Implement proper authentication and rate limiting
- Handle disconnections and reconnections gracefully
- Use rooms for targeted messaging
- Scale with Redis and load balancing
- Test thoroughly including connection flows
- Monitor performance and connection counts
- Implement fallbacks for unreliable connections
- Secure connections with WSS in production

WebSockets power modern real-time web applications!