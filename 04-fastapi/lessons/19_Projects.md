# Real-World Projects in FastAPI

## 19.1 REST API Projects

### Blog API (CRUD Operations)

A complete blog API with posts, comments, categories, and tags.

```python
# models.py
from sqlalchemy import Column, Integer, String, Text, Boolean, DateTime, ForeignKey, Table
from sqlalchemy.orm import relationship
from datetime import datetime
from database import Base

post_tags = Table(
    "post_tags", Base.metadata,
    Column("post_id", Integer, ForeignKey("posts.id")),
    Column("tag_id",  Integer, ForeignKey("tags.id"))
)

class User(Base):
    __tablename__ = "users"
    id       = Column(Integer, primary_key=True)
    username = Column(String(50), unique=True, index=True)
    email    = Column(String(100), unique=True, index=True)
    hashed_pw = Column(String(255))
    posts    = relationship("Post", back_populates="author")
    comments = relationship("Comment", back_populates="author")

class Category(Base):
    __tablename__ = "categories"
    id    = Column(Integer, primary_key=True)
    name  = Column(String(50), unique=True)
    slug  = Column(String(50), unique=True)
    posts = relationship("Post", back_populates="category")

class Tag(Base):
    __tablename__ = "tags"
    id   = Column(Integer, primary_key=True)
    name = Column(String(30), unique=True)
    slug = Column(String(30), unique=True)

class Post(Base):
    __tablename__ = "posts"
    id          = Column(Integer, primary_key=True)
    title       = Column(String(200))
    slug        = Column(String(200), unique=True, index=True)
    content     = Column(Text)
    excerpt     = Column(String(500))
    published   = Column(Boolean, default=False)
    views       = Column(Integer, default=0)
    created_at  = Column(DateTime, default=datetime.utcnow)
    updated_at  = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    author_id   = Column(Integer, ForeignKey("users.id"))
    category_id = Column(Integer, ForeignKey("categories.id"))
    author      = relationship("User", back_populates="posts")
    category    = relationship("Category", back_populates="posts")
    comments    = relationship("Comment", back_populates="post")
    tags        = relationship("Tag", secondary=post_tags)

class Comment(Base):
    __tablename__ = "comments"
    id         = Column(Integer, primary_key=True)
    content    = Column(Text)
    created_at = Column(DateTime, default=datetime.utcnow)
    post_id    = Column(Integer, ForeignKey("posts.id"))
    author_id  = Column(Integer, ForeignKey("users.id"))
    post       = relationship("Post", back_populates="comments")
    author     = relationship("User", back_populates="comments")
```

```python
# schemas.py
from pydantic import BaseModel, Field
from datetime import datetime
from typing import List, Optional

class TagOut(BaseModel):
    id: int
    name: str
    slug: str
    model_config = {"from_attributes": True}

class AuthorOut(BaseModel):
    id: int
    username: str
    model_config = {"from_attributes": True}

class PostCreate(BaseModel):
    title:       str = Field(..., min_length=3, max_length=200)
    content:     str = Field(..., min_length=10)
    excerpt:     Optional[str] = Field(None, max_length=500)
    published:   bool = False
    category_id: Optional[int] = None
    tag_ids:     List[int] = []

class PostOut(BaseModel):
    id:         int
    title:      str
    slug:       str
    excerpt:    Optional[str]
    published:  bool
    views:      int
    created_at: datetime
    author:     AuthorOut
    tags:       List[TagOut]
    model_config = {"from_attributes": True}

class PagedPostsOut(BaseModel):
    items:       List[PostOut]
    total:       int
    page:        int
    page_size:   int
    total_pages: int
```

```python
# routers/posts.py
from fastapi import APIRouter, Depends, HTTPException, Query
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, func
from math import ceil
import re

router = APIRouter(prefix="/posts", tags=["Posts"])

def slugify(text: str) -> str:
    text = text.lower()
    text = re.sub(r"[^\w\s-]", "", text)
    text = re.sub(r"[\s_-]+", "-", text)
    return text.strip("-")

@router.post("/", response_model=PostOut, status_code=201)
async def create_post(
    post_in: PostCreate,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    slug = slugify(post_in.title)
    existing = await db.execute(select(Post).where(Post.slug == slug))
    if existing.scalar():
        slug = f"{slug}-{int(datetime.utcnow().timestamp())}"

    post = Post(
        title=post_in.title, slug=slug, content=post_in.content,
        excerpt=post_in.excerpt, published=post_in.published,
        category_id=post_in.category_id, author_id=current_user.id
    )
    if post_in.tag_ids:
        tags = await db.execute(select(Tag).where(Tag.id.in_(post_in.tag_ids)))
        post.tags = tags.scalars().all()

    db.add(post)
    await db.commit()
    await db.refresh(post)
    return post

@router.get("/", response_model=PagedPostsOut)
async def list_posts(
    page:      int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    category:  Optional[str] = None,
    search:    Optional[str] = None,
    db: AsyncSession = Depends(get_db)
):
    query = select(Post).where(Post.published == True)
    if category:
        query = query.join(Category).where(Category.slug == category)
    if search:
        query = query.where(Post.title.ilike(f"%{search}%"))

    total_q = await db.execute(select(func.count()).select_from(query.subquery()))
    total   = total_q.scalar()

    result = await db.execute(query.offset((page - 1) * page_size).limit(page_size))
    posts  = result.scalars().all()

    return PagedPostsOut(
        items=posts, total=total, page=page,
        page_size=page_size, total_pages=ceil(total / page_size)
    )

@router.get("/{slug}", response_model=PostOut)
async def get_post(slug: str, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(Post).where(Post.slug == slug))
    post   = result.scalar_one_or_none()
    if not post:
        raise HTTPException(404, "Post not found")
    post.views += 1
    await db.commit()
    return post

@router.put("/{post_id}", response_model=PostOut)
async def update_post(
    post_id: int, post_in: PostCreate,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    result = await db.execute(select(Post).where(Post.id == post_id))
    post   = result.scalar_one_or_none()
    if not post:
        raise HTTPException(404, "Post not found")
    if post.author_id != current_user.id:
        raise HTTPException(403, "Not authorized")
    for key, val in post_in.dict(exclude={"tag_ids"}, exclude_unset=True).items():
        setattr(post, key, val)
    await db.commit()
    await db.refresh(post)
    return post

@router.delete("/{post_id}", status_code=204)
async def delete_post(
    post_id: int,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    result = await db.execute(select(Post).where(Post.id == post_id))
    post   = result.scalar_one_or_none()
    if not post:
        raise HTTPException(404, "Post not found")
    if post.author_id != current_user.id:
        raise HTTPException(403, "Not authorized")
    await db.delete(post)
    await db.commit()
```

---

### E-Commerce API

Complete e-commerce system with products, carts, orders, and payments.

```python
# models.py (E-commerce)
class Product(Base):
    __tablename__ = "products"
    id          = Column(Integer, primary_key=True)
    name        = Column(String(200))
    description = Column(Text)
    price       = Column(Float)
    stock       = Column(Integer, default=0)
    sku         = Column(String(50), unique=True)
    order_items = relationship("OrderItem", back_populates="product")

class Cart(Base):
    __tablename__ = "carts"
    id      = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey("users.id"), unique=True)
    items   = relationship("CartItem", back_populates="cart")

class CartItem(Base):
    __tablename__ = "cart_items"
    id         = Column(Integer, primary_key=True)
    cart_id    = Column(Integer, ForeignKey("carts.id"))
    product_id = Column(Integer, ForeignKey("products.id"))
    quantity   = Column(Integer, default=1)
    cart       = relationship("Cart", back_populates="items")
    product    = relationship("Product")

class Order(Base):
    __tablename__ = "orders"
    id             = Column(Integer, primary_key=True)
    user_id        = Column(Integer, ForeignKey("users.id"))
    total          = Column(Float)
    status         = Column(String(20), default="pending")
    payment_status = Column(String(20), default="unpaid")
    created_at     = Column(DateTime, default=datetime.utcnow)
    items          = relationship("OrderItem", back_populates="order")

class OrderItem(Base):
    __tablename__ = "order_items"
    id         = Column(Integer, primary_key=True)
    order_id   = Column(Integer, ForeignKey("orders.id"))
    product_id = Column(Integer, ForeignKey("products.id"))
    quantity   = Column(Integer)
    unit_price = Column(Float)
    order      = relationship("Order", back_populates="items")
    product    = relationship("Product")
```

```python
# routers/orders.py
import stripe

stripe.api_key = settings.STRIPE_SECRET_KEY

@router.post("/checkout", status_code=201)
async def checkout(
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    cart_result = await db.execute(
        select(Cart).where(Cart.user_id == current_user.id)
        .options(selectinload(Cart.items).selectinload(CartItem.product))
    )
    cart = cart_result.scalar_one_or_none()
    if not cart or not cart.items:
        raise HTTPException(400, "Cart is empty")

    for item in cart.items:
        if item.product.stock < item.quantity:
            raise HTTPException(400, f"Insufficient stock for {item.product.name}")

    total = sum(i.quantity * i.product.price for i in cart.items)
    order = Order(user_id=current_user.id, total=total)
    db.add(order)
    await db.flush()

    for item in cart.items:
        db.add(OrderItem(
            order_id=order.id, product_id=item.product_id,
            quantity=item.quantity, unit_price=item.product.price
        ))
        item.product.stock -= item.quantity
        await db.delete(item)

    payment_intent = stripe.PaymentIntent.create(
        amount=int(total * 100), currency="usd",
        metadata={"order_id": order.id}
    )

    await db.commit()
    return {"order_id": order.id, "total": total, "client_secret": payment_intent.client_secret}

@router.post("/webhook/stripe")
async def stripe_webhook(request: Request, db: AsyncSession = Depends(get_db)):
    payload   = await request.body()
    sig       = request.headers.get("stripe-signature")
    try:
        event = stripe.Webhook.construct_event(payload, sig, settings.STRIPE_WEBHOOK_SECRET)
    except Exception:
        raise HTTPException(400, "Invalid webhook")

    if event["type"] == "payment_intent.succeeded":
        order_id = event["data"]["object"]["metadata"]["order_id"]
        result   = await db.execute(select(Order).where(Order.id == order_id))
        order    = result.scalar_one_or_none()
        if order:
            order.payment_status = "paid"
            order.status         = "confirmed"
            await db.commit()
    return {"status": "ok"}
```

---

### Task Management API

Project and task management (Trello-style) with drag-and-drop ordering.

```python
# models.py (Task Management)
class Board(Base):
    __tablename__ = "boards"
    id         = Column(Integer, primary_key=True)
    name       = Column(String(100))
    project_id = Column(Integer, ForeignKey("projects.id"))
    columns    = relationship("BoardColumn", back_populates="board", order_by="BoardColumn.position")

class BoardColumn(Base):
    __tablename__ = "board_columns"
    id       = Column(Integer, primary_key=True)
    name     = Column(String(50))
    position = Column(Integer)
    board_id = Column(Integer, ForeignKey("boards.id"))
    board    = relationship("Board", back_populates="columns")
    tasks    = relationship("Task", back_populates="column", order_by="Task.position")

class Task(Base):
    __tablename__ = "tasks"
    id          = Column(Integer, primary_key=True)
    title       = Column(String(200))
    description = Column(Text)
    priority    = Column(String(20), default="medium")
    position    = Column(Integer)
    due_date    = Column(DateTime)
    column_id   = Column(Integer, ForeignKey("board_columns.id"))
    assignee_id = Column(Integer, ForeignKey("users.id"))
    column      = relationship("BoardColumn", back_populates="tasks")
    assignee    = relationship("User")

@router.patch("/tasks/{task_id}/move")
async def move_task(
    task_id:      int,
    column_id:    int,
    new_position: int,
    db: AsyncSession = Depends(get_db)
):
    result = await db.execute(select(Task).where(Task.id == task_id))
    task   = result.scalar_one_or_none()
    if not task:
        raise HTTPException(404, "Task not found")

    # Shift other tasks
    tasks_result = await db.execute(
        select(Task).where(Task.column_id == column_id).order_by(Task.position)
    )
    for i, t in enumerate(tasks_result.scalars().all()):
        if i >= new_position:
            t.position = i + 1

    task.column_id = column_id
    task.position  = new_position
    await db.commit()
    return {"message": "Task moved"}
```

## 19.2 Authentication Systems

### User Registration / Login System

Full auth system with JWT, email verification, and refresh tokens.

```python
# auth/router.py
from fastapi import APIRouter, Depends, HTTPException, BackgroundTasks
from fastapi.security import OAuth2PasswordRequestForm
from passlib.context import CryptContext
from jose import jwt
from datetime import datetime, timedelta
import secrets, smtplib
from email.mime.text import MIMEText

router     = APIRouter(prefix="/auth", tags=["Auth"])
pwd_ctx    = CryptContext(schemes=["bcrypt"], deprecated="auto")

async def send_verification_email(email: str, token: str):
    link    = f"https://myapp.com/verify?token={token}"
    msg     = MIMEText(f"Verify your email: {link}")
    msg["Subject"] = "Email Verification"
    msg["From"]    = "noreply@myapp.com"
    msg["To"]      = email
    with smtplib.SMTP(settings.SMTP_HOST, settings.SMTP_PORT) as s:
        s.starttls()
        s.login(settings.SMTP_USER, settings.SMTP_PASS)
        s.send_message(msg)

@router.post("/register", status_code=201)
async def register(
    user_in: UserCreate,
    background_tasks: BackgroundTasks,
    db: AsyncSession = Depends(get_db)
):
    existing = await db.execute(
        select(User).where((User.email == user_in.email) | (User.username == user_in.username))
    )
    if existing.scalar():
        raise HTTPException(400, "Email or username already registered")

    token = secrets.token_urlsafe(32)
    user  = User(
        username=user_in.username, email=user_in.email,
        hashed_pw=pwd_ctx.hash(user_in.password),
        verification_token=token, is_verified=False
    )
    db.add(user)
    await db.commit()
    background_tasks.add_task(send_verification_email, user.email, token)
    return {"message": "Registered. Please verify your email."}

@router.post("/verify-email")
async def verify_email(token: str, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).where(User.verification_token == token))
    user   = result.scalar_one_or_none()
    if not user:
        raise HTTPException(400, "Invalid token")
    user.is_verified        = True
    user.verification_token = None
    await db.commit()
    return {"message": "Email verified"}

@router.post("/login")
async def login(
    form: OAuth2PasswordRequestForm = Depends(),
    db: AsyncSession = Depends(get_db)
):
    result = await db.execute(select(User).where(User.username == form.username))
    user   = result.scalar_one_or_none()
    if not user or not pwd_ctx.verify(form.password, user.hashed_pw):
        raise HTTPException(401, "Invalid credentials")
    if not user.is_verified:
        raise HTTPException(403, "Please verify your email")

    access_token = jwt.encode(
        {"sub": str(user.id), "exp": datetime.utcnow() + timedelta(minutes=30)},
        settings.SECRET_KEY, algorithm="HS256"
    )
    refresh_token        = secrets.token_urlsafe(64)
    user.refresh_token   = refresh_token
    await db.commit()

    return {"access_token": access_token, "refresh_token": refresh_token, "token_type": "bearer"}

@router.post("/refresh")
async def refresh(refresh_token: str, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).where(User.refresh_token == refresh_token))
    user   = result.scalar_one_or_none()
    if not user:
        raise HTTPException(401, "Invalid refresh token")
    access_token = jwt.encode(
        {"sub": str(user.id), "exp": datetime.utcnow() + timedelta(minutes=30)},
        settings.SECRET_KEY, algorithm="HS256"
    )
    return {"access_token": access_token, "token_type": "bearer"}
```

### Multi-Factor Authentication (TOTP)

```python
import pyotp, qrcode, io
from base64 import b64encode

@router.post("/mfa/enable")
async def enable_mfa(
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    secret  = pyotp.random_base32()
    totp    = pyotp.TOTP(secret)
    uri     = totp.provisioning_uri(current_user.email, issuer_name="MyApp")

    qr  = qrcode.make(uri)
    buf = io.BytesIO()
    qr.save(buf, format="PNG")
    qr_b64 = b64encode(buf.getvalue()).decode()

    current_user.mfa_secret  = secret
    current_user.mfa_pending = True
    await db.commit()

    return {"secret": secret, "qr_code": f"data:image/png;base64,{qr_b64}"}

@router.post("/mfa/verify")
async def verify_mfa_setup(
    code: str,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    if not pyotp.TOTP(current_user.mfa_secret).verify(code, valid_window=1):
        raise HTTPException(400, "Invalid MFA code")
    current_user.mfa_enabled = True
    current_user.mfa_pending = False
    await db.commit()
    return {"message": "MFA enabled"}
```

### Role-Based Access Control (RBAC)

```python
from enum import Enum

class Permission(str, Enum):
    READ_POSTS   = "read:posts"
    WRITE_POSTS  = "write:posts"
    DELETE_POSTS = "delete:posts"
    MANAGE_USERS = "manage:users"
    ADMIN        = "admin:all"

ROLE_PERMISSIONS = {
    "guest":     [Permission.READ_POSTS],
    "author":    [Permission.READ_POSTS, Permission.WRITE_POSTS],
    "editor":    [Permission.READ_POSTS, Permission.WRITE_POSTS, Permission.DELETE_POSTS],
    "admin":     list(Permission),
}

def require_permission(permission: Permission):
    async def checker(current_user: User = Depends(get_current_user)):
        perms = ROLE_PERMISSIONS.get(current_user.role, [])
        if Permission.ADMIN in perms or permission in perms:
            return current_user
        raise HTTPException(403, f"Missing permission: {permission}")
    return checker

@router.delete("/posts/{post_id}")
async def delete_post(
    post_id: int,
    db: AsyncSession = Depends(get_db),
    _: User = Depends(require_permission(Permission.DELETE_POSTS))
):
    result = await db.execute(select(Post).where(Post.id == post_id))
    post   = result.scalar_one_or_none()
    if not post:
        raise HTTPException(404, "Post not found")
    await db.delete(post)
    await db.commit()
```

## 19.3 Real-Time Applications

### Chat Application

Full-featured real-time chat with WebSockets and history.

```python
# chat/manager.py
from fastapi import WebSocket
from typing import Dict, Set

class ConnectionManager:
    def __init__(self):
        self.rooms: Dict[str, Set[WebSocket]] = {}
        self.users: Dict[WebSocket, dict]     = {}

    async def connect(self, ws: WebSocket, room_id: str, user: dict):
        await ws.accept()
        self.rooms.setdefault(room_id, set()).add(ws)
        self.users[ws] = {**user, "room_id": room_id}
        await self.broadcast(room_id, {"type": "user_joined", "username": user["username"]}, exclude=ws)

    async def disconnect(self, ws: WebSocket):
        user = self.users.pop(ws, None)
        if user:
            room_id = user["room_id"]
            self.rooms[room_id].discard(ws)
            if not self.rooms[room_id]:
                del self.rooms[room_id]
            else:
                await self.broadcast(room_id, {"type": "user_left", "username": user["username"]})

    async def broadcast(self, room_id: str, message: dict, exclude: WebSocket = None):
        dead = set()
        for ws in self.rooms.get(room_id, set()):
            if ws == exclude:
                continue
            try:
                await ws.send_json(message)
            except Exception:
                dead.add(ws)
        for ws in dead:
            await self.disconnect(ws)

    def room_users(self, room_id: str):
        return [self.users[ws]["username"] for ws in self.rooms.get(room_id, set())]

manager = ConnectionManager()

@router.websocket("/ws/{room_id}")
async def chat_ws(websocket: WebSocket, room_id: str, token: str, db: AsyncSession = Depends(get_db)):
    try:
        user = await get_user_from_token(token, db)
    except Exception:
        await websocket.close(code=4001)
        return

    await manager.connect(websocket, room_id, {"id": user.id, "username": user.username})

    try:
        # Send recent chat history
        history = await db.execute(
            select(ChatMessage).where(ChatMessage.room_id == room_id)
            .order_by(ChatMessage.created_at.desc()).limit(50)
        )
        for msg in reversed(history.scalars().all()):
            await websocket.send_json({
                "type": "history", "username": msg.author.username,
                "content": msg.content, "timestamp": msg.created_at.isoformat()
            })

        while True:
            data    = await websocket.receive_json()
            content = data.get("content", "").strip()
            if not content:
                continue

            # Save to DB
            message = ChatMessage(room_id=room_id, user_id=user.id, content=content)
            db.add(message)
            await db.commit()

            await manager.broadcast(room_id, {
                "type":      "message",
                "username":  user.username,
                "content":   content,
                "timestamp": message.created_at.isoformat()
            })
    except Exception:
        pass
    finally:
        await manager.disconnect(websocket)
```

### Live Notification System

Real-time notifications via SSE with per-user queues.

```python
# notifications/broadcaster.py
import asyncio, json

notification_queues: dict[int, asyncio.Queue] = {}

async def push_to_user(user_id: int, notification: dict):
    if user_id in notification_queues:
        await notification_queues[user_id].put(notification)

async def push_to_many(user_ids: list[int], notification: dict):
    await asyncio.gather(*[push_to_user(uid, notification) for uid in user_ids])

@router.get("/stream")
async def notification_stream(current_user: User = Depends(get_current_user)):
    async def generator():
        queue = asyncio.Queue()
        notification_queues[current_user.id] = queue
        try:
            while True:
                try:
                    notif = await asyncio.wait_for(queue.get(), timeout=30.0)
                    yield f"data: {json.dumps(notif)}\n\n"
                except asyncio.TimeoutError:
                    yield ": heartbeat\n\n"
        finally:
            notification_queues.pop(current_user.id, None)

    return StreamingResponse(generator(), media_type="text/event-stream",
                             headers={"Cache-Control": "no-cache"})

# Usage: notify post author when someone comments
@router.post("/posts/{post_id}/comments", status_code=201)
async def add_comment(
    post_id: int, comment_in: CommentCreate,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    comment = Comment(content=comment_in.content, post_id=post_id, author_id=current_user.id)
    db.add(comment)
    await db.commit()

    post = (await db.execute(select(Post).where(Post.id == post_id))).scalar_one_or_none()
    if post and post.author_id != current_user.id:
        await push_to_user(post.author_id, {
            "type":    "comment",
            "message": f"{current_user.username} commented on your post",
            "link":    f"/posts/{post.slug}"
        })
    return comment
```

### Real-Time Analytics Dashboard

Live analytics broadcast via WebSocket.

```python
# analytics/broadcaster.py
import asyncio
from collections import defaultdict
from fastapi import WebSocket

class AnalyticsBroadcaster:
    def __init__(self):
        self.connections: set[WebSocket]        = set()
        self.page_views:  defaultdict[str, int] = defaultdict(int)
        self.active_users: int                  = 0

    async def connect(self, ws: WebSocket):
        await ws.accept()
        self.connections.add(ws)
        self.active_users += 1
        await self.broadcast_state()

    async def disconnect(self, ws: WebSocket):
        self.connections.discard(ws)
        self.active_users = max(0, self.active_users - 1)

    def record_view(self, path: str):
        self.page_views[path] += 1

    async def broadcast_state(self):
        top_pages = dict(sorted(self.page_views.items(), key=lambda x: x[1], reverse=True)[:10])
        state     = {"type": "state", "active_users": self.active_users, "top_pages": top_pages}
        dead      = set()
        for ws in self.connections:
            try:
                await ws.send_json(state)
            except Exception:
                dead.add(ws)
        for ws in dead:
            await self.disconnect(ws)

    async def start(self, interval: int = 5):
        while True:
            await asyncio.sleep(interval)
            await self.broadcast_state()

broadcaster = AnalyticsBroadcaster()

@app.on_event("startup")
async def start_analytics():
    asyncio.create_task(broadcaster.start())

@app.websocket("/analytics/live")
async def analytics_ws(websocket: WebSocket):
    await broadcaster.connect(websocket)
    try:
        while True:
            await websocket.receive_text()
    except Exception:
        pass
    finally:
        await broadcaster.disconnect(websocket)

@app.middleware("http")
async def track_views(request: Request, call_next):
    response = await call_next(request)
    if not request.url.path.startswith(("/health", "/analytics", "/metrics")):
        broadcaster.record_view(request.url.path)
    return response
```

## 19.4 Microservices

### Service Architecture Design

```python
"""
Microservices Layout:

┌──────────────────────────────────┐
│          API Gateway :8000        │
└───┬──────────┬──────────┬────────┘
    ▼          ▼          ▼
┌───────┐  ┌───────┐  ┌──────────┐
│Users  │  │Posts  │  │Notif.    │
│:8001  │  │:8002  │  │:8003     │
└───────┘  └───────┘  └──────────┘
    │          │          │
    └──────────┴──────────┘
               │
       ┌───────────────┐
       │  Message Bus   │
       │  (RabbitMQ)    │
       └───────────────┘
"""
```

### Inter-Service Communication

```python
# gateway/clients.py
import httpx

class ServiceClient:
    def __init__(self, base_url: str, timeout: float = 10.0):
        self.client = httpx.AsyncClient(
            base_url=base_url, timeout=timeout,
            headers={"X-Service-Token": settings.SERVICE_TOKEN}
        )

    async def get(self, path: str, **kw):
        r = await self.client.get(path, **kw)
        r.raise_for_status()
        return r.json()

    async def post(self, path: str, data: dict, **kw):
        r = await self.client.post(path, json=data, **kw)
        r.raise_for_status()
        return r.json()

class UserServiceClient(ServiceClient):
    def __init__(self): super().__init__(settings.USER_SERVICE_URL)

    async def get_user(self, user_id: int):
        try:
            return await self.get(f"/users/{user_id}")
        except httpx.HTTPStatusError as e:
            if e.response.status_code == 404:
                return None
            raise

# Message bus with RabbitMQ
import aio_pika

class MessageBus:
    def __init__(self):
        self.connection = None
        self.channel    = None

    async def connect(self):
        self.connection = await aio_pika.connect_robust(settings.RABBITMQ_URL)
        self.channel    = await self.connection.channel()

    async def publish(self, event_type: str, data: dict):
        exchange = await self.channel.declare_exchange("events", aio_pika.ExchangeType.TOPIC, durable=True)
        await exchange.publish(
            aio_pika.Message(
                body=json.dumps({"type": event_type, "data": data}).encode(),
                delivery_mode=aio_pika.DeliveryMode.PERSISTENT
            ),
            routing_key=event_type
        )

    async def subscribe(self, pattern: str, handler):
        exchange = await self.channel.declare_exchange("events", aio_pika.ExchangeType.TOPIC, durable=True)
        queue    = await self.channel.declare_queue("", exclusive=True)
        await queue.bind(exchange, routing_key=pattern)
        async def on_msg(msg: aio_pika.IncomingMessage):
            async with msg.process():
                await handler(json.loads(msg.body))
        await queue.consume(on_msg)

bus = MessageBus()
```

### API Gateway

```python
# gateway/main.py
from fastapi import FastAPI, Request, HTTPException, Depends
import httpx

app = FastAPI(title="API Gateway")

SERVICES = {
    "users": settings.USER_SERVICE_URL,
    "posts": settings.POST_SERVICE_URL,
    "media": settings.MEDIA_SERVICE_URL,
}

async def authenticate(request: Request):
    token = request.headers.get("Authorization", "").replace("Bearer ", "")
    if not token:
        raise HTTPException(401, "Not authenticated")
    async with httpx.AsyncClient() as client:
        r = await client.post(f"{SERVICES['users']}/auth/verify", json={"token": token})
        if r.status_code != 200:
            raise HTTPException(401, "Invalid token")
        return r.json()

@app.api_route("/{service}/{path:path}", methods=["GET","POST","PUT","PATCH","DELETE"])
async def route(service: str, path: str, request: Request, user: dict = Depends(authenticate)):
    if service not in SERVICES:
        raise HTTPException(404, f"Service '{service}' not found")
    body = await request.body()
    async with httpx.AsyncClient() as client:
        r = await client.request(
            method=request.method,
            url=f"{SERVICES[service]}/{path}",
            headers={"X-User-ID": str(user["id"]), "X-User-Role": user["role"]},
            params=dict(request.query_params),
            content=body, timeout=30.0
        )
    return r.json()
```

## 19.5 Full-Stack Projects

### FastAPI + React

```python
# backend/main.py
from fastapi.middleware.cors import CORSMiddleware
from fastapi.staticfiles import StaticFiles
from fastapi.responses import FileResponse

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "https://myapp.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Serve React build in production
app.mount("/static", StaticFiles(directory="frontend/build/static"), name="static")

@app.get("/{full_path:path}")
async def serve_react(full_path: str):
    if full_path.startswith("api/"):
        raise HTTPException(404)
    return FileResponse("frontend/build/index.html")
```

```javascript
// frontend/src/api/client.js
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL || 'http://localhost:8000',
  withCredentials: true,
});

// Attach JWT to every request
api.interceptors.request.use(config => {
  const token = localStorage.getItem('access_token');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Auto-refresh on 401
api.interceptors.response.use(
  res => res,
  async err => {
    if (err.response?.status === 401) {
      const refresh = localStorage.getItem('refresh_token');
      if (refresh) {
        const { data } = await axios.post('/auth/refresh', { refresh_token: refresh });
        localStorage.setItem('access_token', data.access_token);
        return api(err.config);
      }
    }
    return Promise.reject(err);
  }
);

export const postsAPI = {
  list:   params    => api.get('/posts', { params }),
  get:    slug      => api.get(`/posts/${slug}`),
  create: data      => api.post('/posts', data),
  update: (id, data)=> api.put(`/posts/${id}`, data),
  delete: id        => api.delete(`/posts/${id}`),
};

export default api;
```

### FastAPI + Next.js

```javascript
// frontend/lib/api.js
const BASE = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000';

export async function fetchPosts(page = 1) {
  const res = await fetch(`${BASE}/posts?page=${page}`, {
    next: { revalidate: 60 }  // ISR: revalidate every 60s
  });
  if (!res.ok) throw new Error('Failed to fetch posts');
  return res.json();
}

// app/posts/page.jsx (App Router)
import { fetchPosts } from '@/lib/api';

export default async function PostsPage({ searchParams }) {
  const page                           = Number(searchParams.page) || 1;
  const { items, total, total_pages }  = await fetchPosts(page);

  return (
    <main>
      <h1>Blog Posts</h1>
      {items.map(post => (
        <article key={post.id}>
          <h2>{post.title}</h2>
          <p>{post.excerpt}</p>
          <a href={`/posts/${post.slug}`}>Read more →</a>
        </article>
      ))}
    </main>
  );
}
```

```python
# On-demand revalidation from FastAPI
@app.post("/api/revalidate")
async def revalidate(path: str, secret: str):
    if secret != settings.REVALIDATION_SECRET:
        raise HTTPException(403, "Invalid secret")
    async with httpx.AsyncClient() as client:
        await client.post(
            f"{settings.NEXTJS_URL}/api/revalidate",
            params={"path": path, "secret": secret}
        )
    return {"revalidated": True}
```

### FastAPI + Mobile App

```python
# mobile/router.py
from pydantic import BaseModel

class DeviceRegister(BaseModel):
    token:     str
    platform:  str   # "ios" or "android"
    device_id: str

@router.post("/register-device")
async def register_device(
    device: DeviceRegister,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    existing = (await db.execute(
        select(DeviceRegistration).where(DeviceRegistration.device_id == device.device_id)
    )).scalar_one_or_none()

    if existing:
        existing.token = device.token
    else:
        db.add(DeviceRegistration(
            user_id=current_user.id, token=device.token,
            platform=device.platform, device_id=device.device_id
        ))
    await db.commit()
    return {"message": "Device registered"}

async def send_push(user_id: int, title: str, body: str, db: AsyncSession):
    """Send push notification to all user devices"""
    result  = await db.execute(select(DeviceRegistration).where(DeviceRegistration.user_id == user_id))
    devices = result.scalars().all()
    for device in devices:
        if device.platform == "android":
            async with httpx.AsyncClient() as client:
                await client.post(
                    "https://fcm.googleapis.com/fcm/send",
                    headers={"Authorization": f"key={settings.FCM_KEY}"},
                    json={"to": device.token, "notification": {"title": title, "body": body}}
                )

# Optimized feed for mobile (minimal payload)
@router.get("/feed")
async def mobile_feed(
    page: int = 1, page_size: int = 10,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    result = await db.execute(
        select(Post.id, Post.title, Post.excerpt, Post.slug, Post.created_at)
        .where(Post.published == True)
        .order_by(Post.created_at.desc())
        .offset((page - 1) * page_size).limit(page_size)
    )
    posts = result.all()
    return {
        "posts":    [{"id": p.id, "title": p.title, "excerpt": p.excerpt,
                      "slug": p.slug, "date": p.created_at.isoformat()} for p in posts],
        "has_more": len(posts) == page_size
    }
```

---

## Project Bootstrap Template

```python
# main.py - Production-ready bootstrap
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.gzip import GZipMiddleware
from contextlib import asynccontextmanager
import sentry_sdk
from sentry_sdk.integrations.fastapi import FastApiIntegration

from routers import auth, users, posts, notifications
from database import engine, Base
from config import settings

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    app.state.start_time = __import__("time").time()
    yield
    await engine.dispose()

if settings.SENTRY_DSN:
    sentry_sdk.init(
        dsn=settings.SENTRY_DSN,
        integrations=[FastApiIntegration()],
        environment=settings.ENVIRONMENT,
        traces_sample_rate=0.1,
    )

app = FastAPI(
    title=settings.APP_NAME,
    version=settings.VERSION,
    lifespan=lifespan,
    docs_url  = "/docs"  if settings.ENVIRONMENT != "production" else None,
    redoc_url = "/redoc" if settings.ENVIRONMENT != "production" else None,
)

app.add_middleware(GZipMiddleware, minimum_size=1000)
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(auth.router)
app.include_router(users.router)
app.include_router(posts.router)
app.include_router(notifications.router)

@app.get("/health")
async def health():
    import time
    return {
        "status":      "healthy",
        "environment": settings.ENVIRONMENT,
        "uptime":      round(time.time() - app.state.start_time, 2)
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run("main:app", host="0.0.0.0", port=8000,
                reload=settings.ENVIRONMENT == "development")
```

---

**Key Takeaways:**
- Separate concerns into routers, models, schemas, and services from day one
- Use async database drivers for every I/O operation
- Implement authentication, authorization, and email verification properly
- WebSockets power chat; SSE powers one-way notifications and analytics
- Design microservices around business domains with clear contracts
- Connect frontends with CORS, token refresh interceptors, and ISR for Next.js
- Mobile apps need push notification registration and lightweight payload endpoints

**Additional Resources:**
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [SQLAlchemy Async](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
- [Stripe Python SDK](https://stripe.com/docs/api?lang=python)
- [PyOTP — TOTP Library](https://pyauth.github.io/pyotp/)
- [aio-pika — RabbitMQ](https://aio-pika.readthedocs.io/)
- [Next.js App Router](https://nextjs.org/docs/app)