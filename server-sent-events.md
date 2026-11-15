---
title: "Server-Sent Events (SSE)"
description: "Building Real-Time Notifications with React, FastAPI and PostgreSQL"
date: "2026-09-13"
tags:
  - Server-Sent-Events
  - React
  - Python
  - Postgres
  - Systems
---

Suppose you own a web application. A user is logged in and sitting on the notifications page.

Somewhere else in the system, an event happens:

> A new notification has been created for that user.

You want the notification to appear in the browser **immediately**. You don't want the user to refresh the page to get new notifications.

You don't want the browser to repeatedly ask:

> "Do I have a new notification?"

This is where **real-time communication** becomes useful.

One relatively simple technology that can solve this problem is **Server-Sent Events (SSE)**.

In this article, we will build a notification system using:

* **React** — frontend
* **FastAPI** — backend
* **PostgreSQL** — database
* **SSE** — communication mechanism between the browser and server

We will assume that our application has **only one FastAPI server instance**.

Later, we will see why this assumption is important and why the architecture needs to change when we have multiple server instances.

---

### The Problem We Are Trying to Solve

Forget about SSE.

Suppose our application has this architecture:

```mermaid
flowchart LR
    Browser[React Browser]
    Server[FastAPI Server]
    DB[(PostgreSQL)]

    Browser --> Server
    Server --> DB
```

A notification is created.

For example:

```text
"You have a new friend request."
```

The backend stores it in PostgreSQL.

```mermaid
sequenceDiagram
    participant User
    participant Server as FastAPI
    participant DB as PostgreSQL

    User->>Server: Create notification
    Server->>DB: INSERT notification
    DB-->>Server: Success
```

So far, everything is easy. But now there is a problem. The browser doesn't automatically know that a new row was inserted into PostgreSQL.

The database knows. The FastAPI server knows. But the browser doesn't.

We need some way to tell the browser:

> "Hey, a new notification has arrived."

There are several ways we could do this.

---

### The Simplest Approach: Polling

Polling simply means:

> The client repeatedly asks the server whether something has changed.

For example, React could call:

```text
GET /notifications
```

every 5 minutes. The flow would look like this:

```mermaid
sequenceDiagram
    participant React
    participant FastAPI
    participant PostgreSQL

    loop Every 5 minutes
        React->>FastAPI: GET /notifications
        FastAPI->>PostgreSQL: Check notifications
        PostgreSQL-->>FastAPI: Notifications
        FastAPI-->>React: Notifications
    end
```

This works. But it isn't truly real-time. Suppose a notification arrives at:

```text
10:01:20
```

and the browser checks at:

```text
10:05:00
```

The user might wait almost four minutes before seeing the notification. We could reduce the polling interval:

```mermaid
flowchart TD
    A["Every 5 minutes"]
    --> B["Every 1 minute"]
    --> C["Every 10 seconds"]
    --> D["Every 1 second"]
```

But now we are making many unnecessary requests.

Imagine 10,000 users. If every user makes a request every 5 seconds, the server has to handle a large number of requests even when there are no new notifications.

There is another problem,  most of those requests are asking:

> "Anything new?"

and the answer is often:

> "No."

This is wasteful. We want something better.

---

### What If the Server Could Talk to the Browser?

Instead of the browser repeatedly asking:

> "Do you have anything for me?"

what if the browser could say:

> "Keep a connection open. Whenever something happens, tell me."

That is the basic idea behind **Server-Sent Events**.

SSE allows a browser to establish a long-lived HTTP connection with the server. The connection stays open. The server can then send events to the browser whenever it has something to send.

The basic architecture becomes:

```mermaid
flowchart LR
    Browser[React Browser]
    Server[FastAPI Server]

    Browser -->|Open SSE connection| Server
    Server -->|Push event whenever needed| Browser
```

The browser does **not** repeatedly create requests. Instead, it opens one connection and keeps it alive.

---

### What Does "Server-Sent Events" Mean?

Let's break the name down.

#### Server

The server is our FastAPI application.

#### Sent

The server sends information to the client.

#### Events

The information is sent as events.

So:

> **Server-Sent Events are a mechanism where a server can continuously send event messages to a client over a long-lived HTTP connection.**

The direction is primarily:

```mermaid
flowchart LR
    A["Server"] --> B["Browser"]
```

This is why SSE is particularly useful for things like:

* notifications
* live activity feeds
* progress updates
* job status
* monitoring dashboards
* live logs
* stock/price updates
* news feeds

---

### SSE Is One-Way Communication

This is one of the most important things to understand.

SSE is designed for:

```mermaid
flowchart TD
    A["Server"] -->|events| B["Browser"]
```

The server sends events to the browser. The browser does not use the SSE connection to send arbitrary messages back to the server.

If the browser needs to send something to the server, it can use normal HTTP requests.

For example:

```mermaid
flowchart TD
    R["React / Browser"]
    F["FastAPI"]

    R -->|"POST /notifications/read"| F
    R -->|"SSE connection (GET /notifications/stream)"| F
    F -->|"notifications (SSE events)"| R
```

So we can have:

```mermaid
flowchart LR
    B["Browser"]
    S["Server"]

    B -->|"HTTP request"| S
    B -->|"SSE connection / GET request"| S
    S -->|"SSE events"| B
```

This makes SSE a very good fit for **server-to-client updates**.

---

# SSE vs WebSockets

You may have heard of another technology called **WebSocket**.

WebSocket provides two-way communication:

```mermaid
flowchart LR
    B["Browser"] <--> S["Server"]
```

Both sides can continuously send messages.

SSE is different:

```mermaid
flowchart LR
    S["Server"] --> B["Browser"]
```

The server continuously sends events to the browser.

For a notification system, we often don't need the full power of WebSockets.

Our requirement is simply:

> "When the server has a notification for the user, send it to the browser."

SSE fits this requirement nicely.

---

### What Actually Happens When We Use SSE?

Let's look at the process.

First, React opens an SSE connection:

```text
GET /notifications/stream
```

The server accepts the request.

Instead of returning a normal response and closing the connection, the server keeps the connection open.

```mermaid
sequenceDiagram
    participant React
    participant FastAPI

    React->>FastAPI: GET /notifications/stream
    FastAPI-->>React: Connection established

    Note over React,FastAPI: Connection remains open

    FastAPI-->>React: notification event
    FastAPI-->>React: notification event
    FastAPI-->>React: notification event
```

This is the key difference. With a normal HTTP request:

```mermaid
flowchart TD
    A["Request"]
    --> B["Response"]
    --> C["Connection / Request Completed"]
```

With SSE:

```mermaid
flowchart TD
    A["Request"]
    --> B["Connection Established"]
    --> C["Connection Remains Open"]
    --> D["Event"]
    --> E["Event"]
    --> F["Event"]
    --> G["..."]
```

---

### What Does an SSE Event Look Like?

SSE uses a simple text-based format. Example:

```text
event: notification
data: {"id": 10, "message": "You have a new friend request"}

```

The blank line at the end is important because it separates one event from the next. Another event might look like:

```text
event: notification
data: {"id": 11, "message": "Your post received a new comment"}

```

The browser receives these events and JavaScript can react to them.

---

### Our Notification Architecture

Now let's build the actual application.

Our stack is:

```mermaid
flowchart TD
    A["React"] -->|"HTTP + SSE"| B["FastAPI"]
    B --> C["PostgreSQL"]
```

But there is one additional component inside our FastAPI process.

We will call it:

> **Notification Manager**

The Notification Manager is simply an in-memory component responsible for keeping track of connected users and sending events to them.

The complete architecture is:

```mermaid
flowchart TB

    React[React Browser]

    FastAPI[FastAPI Server]

    Manager[Notification Manager]

    DB[(PostgreSQL)]

    React -->|GET /notifications/stream| FastAPI
    FastAPI --> Manager

    FastAPI -->|Read / Write notifications| DB

    Manager -->|SSE event| React
```

There is an important assumption here:

> **We have only one FastAPI server instance.**

We will discuss why this matters later.

---

### Why Do We Need a Notification Manager?

Suppose Alice opens our application in Chrome. Her browser establishes an SSE connection.

We need somewhere inside our FastAPI process to keep track of that connection.

But there is an important detail:

> **A user can have more than one active SSE connection.**

Alice might have:

* Chrome open on her laptop
* Safari open on the same laptop
* another browser tab open
* the application open on her phone

Each of these can create a separate SSE connection. So we should not model our connections as:

```text
One user → One connection
```

Instead, we should model them as:

```text
One user → Many connections
```

For example:

```text
Notification Manager

Alice
 ├── Chrome → Connection A
 ├── Safari → Connection B
 └── Chrome Tab 2 → Connection C

Bob
 └── Chrome → Connection D
```

When a notification is created for Alice, we want to send it to **all of Alice's active connections**.

```mermaid
flowchart TD
    A["Notification"] --> B["Notification Manager"]

    B --> C["Connection A"]
    B --> D["Connection B"]
    B --> E["Connection C"]

    C --> F["Chrome"]
    D --> G["Safari"]
    E --> H["Chrome Tab 2"]
```

The Notification Manager is therefore responsible for:

1. Registering new SSE connections.
2. Keeping track of multiple connections for each user.
3. Sending a notification to all active connections belonging to that user.
4. Removing a specific connection when it disconnects.


---

### Database vs Notification Manager

At this point, you might ask:

> "Why don't we just keep the notification in the Notification Manager?"

Because the Notification Manager is **not our permanent storage**. It exists in the memory of our FastAPI process. If the server crashes, the memory is gone.

The notification stored only in memory would be lost. PostgreSQL, on the other hand, is our persistent storage.

Therefore, PostgreSQL is the source of truth.

The Notification Manager is only responsible for delivering the event to currently connected users.

---

### The Complete Flow

Suppose Alice is logged in. First, React establishes an SSE connection.

```mermaid
sequenceDiagram
    participant React
    participant FastAPI
    participant Manager

    React->>FastAPI: GET /notifications/stream
    FastAPI->>Manager: Register Alice connection
    Manager-->>FastAPI: Registered
    FastAPI-->>React: Keep connection open
```

Now another part of our application creates a notification.

```mermaid
sequenceDiagram
    participant Application
    participant FastAPI
    participant DB as PostgreSQL
    participant Manager
    participant React

    Application->>FastAPI: Create notification
    FastAPI->>DB: INSERT notification
    DB-->>FastAPI: Success

    FastAPI->>Manager: Send notification to Alice
    Manager-->>React: SSE event
    React->>React: Update notification UI
```

This gives us near-immediate delivery.

---

### PostgreSQL Notification Table

Let's create a simple table.

```sql
CREATE TABLE notifications (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    message TEXT NOT NULL,
    is_read BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Here:

#### `id`

Uniquely identifies the notification.

#### `user_id`

Identifies which user owns the notification.

#### `message`

The notification text.

#### `is_read`

Whether the user has read the notification.

#### `created_at`

When the notification was created.

Notice that we are using:

```sql
TIMESTAMPTZ
```

rather than storing timestamps as strings. The database should store an actual timestamp type, not something like:

```text
"2026-09-13 10:30:00"
```

as plain text.

---

### FastAPI Project Structure

We can keep the example simple:

```text
backend/
│
├── main.py
├── database.py
├── models.py
├── notification_manager.py
└── routes/
    └── notifications.py
```

---

### Database Connection

For simplicity, we'll use SQLAlchemy.

```python
# database.py

from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, DeclarativeBase

DATABASE_URL = "postgresql://postgres:password@localhost/notifications"

engine = create_engine(DATABASE_URL)

SessionLocal = sessionmaker(
    bind=engine,
    autocommit=False,
    autoflush=False
)


class Base(DeclarativeBase):
    pass


def get_db():
    db = SessionLocal()

    try:
        yield db
    finally:
        db.close()
```

In a real application, the database URL should come from environment variables rather than being hard-coded.

For example:

---

### Notification Model

```python
# models.py

from sqlalchemy import Boolean, DateTime, Integer, Text
from sqlalchemy.orm import Mapped, mapped_column
from datetime import datetime

from database import Base


class Notification(Base):
    __tablename__ = "notifications"

    id: Mapped[int] = mapped_column(
        Integer,
        primary_key=True
    )

    user_id: Mapped[int] = mapped_column(
        Integer,
        nullable=False
    )

    message: Mapped[str] = mapped_column(
        Text,
        nullable=False
    )

    is_read: Mapped[bool] = mapped_column(
        Boolean,
        default=False,
        nullable=False
    )

    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        nullable=False
    )
```

---

### Notification Manager

We can represent the connections using a dictionary.

The key is the user's ID, and the value is a set of queues representing that user's active SSE connections.

```python
# notification_manager.py

import asyncio


class NotificationManager:

    def __init__(self):
        self.connections: dict[
            int,
            set[asyncio.Queue]
        ] = {}


    async def connect(
        self,
        user_id: int
    ) -> asyncio.Queue:

        queue = asyncio.Queue()

        self.connections \
            .setdefault(user_id, set()) \
            .add(queue)

        return queue


    def disconnect(
        self,
        user_id: int,
        queue: asyncio.Queue
    ):

        user_connections = self.connections.get(user_id)

        if user_connections is None:
            return

        user_connections.discard(queue)

        if not user_connections:
            self.connections.pop(user_id, None)


    async def send(
        self,
        user_id: int,
        notification: dict
    ):

        user_connections = self.connections.get(user_id)

        if not user_connections:
            return

        for queue in user_connections:
            await queue.put(notification)
```

Let's understand the structure. The `connections` dictionary might look like this:

```text
{
    Alice: {
        Queue A,
        Queue B
    },

    Bob: {
        Queue C
    }
}
```

So if Alice has Chrome and Safari open:

```text
Alice
 ├── Queue A → Chrome
 └── Queue B → Safari
```

When we send a notification to Alice:

```python
for queue in user_connections:
    await queue.put(notification)
```

we put the notification into **every queue belonging to Alice**. Both browsers will therefore receive the notification.

---

#### Why do we use a `set`?

A `set` is a collection that stores unique values. We don't particularly care about the order of the connections. We simply need to keep track of all active connections belonging to a user.

So:

```python
set[asyncio.Queue]
```

is a natural fit.

---

#### Why does `disconnect()` receive the queue?

This is also important. Suppose Alice has:

```text
Chrome  → Queue A
Safari  → Queue B
```

Now Alice closes Chrome. We need to remove **Queue A**, not all of Alice's connections.

That's why we do:

```python
disconnect(user_id, queue)
```

rather than:

```python
disconnect(user_id)
```

After Chrome disconnects:

```text
Alice
 └── Safari → Queue B
```

Safari remains connected.

---

### What Is `asyncio.Queue`?

A **queue** is a data structure where items are placed in one side and consumed from the other.

Think of it like a line:

```text
             Queue

        ┌──────────────┐
        │ Notification │
        │ Notification │
        │ Notification │
        └──────────────┘
```

Our server puts notifications into the queue:

```mermaid
flowchart TD
    A["FastAPI"] --> B["Queue"]
```

The SSE endpoint waits for notifications from that queue.

```mermaid
flowchart TD
    A["Queue"] --> B["SSE Endpoint"]
    B --> C["Browser"]
```

The important property is that the SSE endpoint can **wait without constantly polling the database**.

---

### Creating the SSE Endpoint

Now let's create:

```text
GET /notifications/stream
```

```python
# routes/notifications.py

import asyncio
import json

from fastapi import APIRouter
from fastapi.responses import StreamingResponse

from notification_manager import NotificationManager


router = APIRouter()

manager = NotificationManager()


@router.get("/notifications/stream")
async def notification_stream(user_id: int):

    queue = await manager.connect(user_id)


    async def event_generator():

        try:

            while True:

                notification = await queue.get()

                yield (
                    "event: notification\n"
                    f"data: {json.dumps(notification)}\n\n"
                )

        except asyncio.CancelledError:

            manager.disconnect(
                user_id,
                queue
            )

            raise


    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream"
    )
```

Notice that we keep the queue returned by:

```python
queue = await manager.connect(user_id)
```

This queue represents **this particular SSE connection**.

If Alice has two browser connections:

```text
Alice
 ├── Connection A → Queue A
 └── Connection B → Queue B
```

each connection gets its own queue.

If Connection A closes, the server calls:

```python
manager.disconnect(
    alice_id,
    queue_a
)
```

Only that connection is removed. Connection B continues to work.


---

### What Is a Streaming Response?

Normally, a server might do this:

```mermaid
flowchart TD
    A["Request"]
    --> B["Process"]
    --> C["Create Response"]
    --> D["Send Response"]
    --> E["Done"]
```

A streaming response works differently.

```mermaid
flowchart TD
    A["Request"]
    --> B["Connection Opened"]
    --> C["Send Some Data"]
    --> D["Wait"]
    --> E["Send More Data"]
    --> F["Wait"]
    --> G["Send More Data"]
    --> H["..."]
```

This is exactly what SSE needs.
There can also be multiple streaming responses for the same user.

For example:

```text
Alice
 ├── Chrome → StreamingResponse A
 └── Safari → StreamingResponse B
```

Each response represents a separate SSE connection and has its own queue.

The Notification Manager is responsible for sending the same notification to both connections.


---

### Understand the Generator

The important part is:

```python
async def event_generator():

    while True:

        notification = await queue.get()

        yield (
            f"event: notification\n"
            f"data: {json.dumps(notification)}\n\n"
        )
```

The `while True` means:

> Continue waiting for events.

Then:

```python
await queue.get()
```

means:

> Wait until a notification is available.

When a notification arrives, we send it:

```python
yield ...
```

Then we go back to waiting. So conceptually:

```mermaid
flowchart TD
    A["Wait for Notification"]
    --> B["Notification Arrives"]
    --> C["Send SSE Event"]
    --> A
```

---

### Creating a Notification

Now let's create another endpoint:

```text
POST /notifications
```

This endpoint will:

1. Insert the notification into PostgreSQL.
2. Send the notification to the user's active SSE connection.

```python
from fastapi import Depends
from sqlalchemy.orm import Session

from database import get_db
from models import Notification


@router.post("/notifications")
async def create_notification(
    user_id: int,
    message: str,
    db: Session = Depends(get_db)
):

    notification = Notification(
        user_id=user_id,
        message=message
    )

    db.add(notification)
    db.commit()
    db.refresh(notification)


    notification_data = {
        "id": notification.id,
        "user_id": notification.user_id,
        "message": notification.message,
        "created_at": notification.created_at.isoformat()
    }


    await manager.send(
        user_id,
        notification_data
    )


    return notification_data
```

Now we have connected PostgreSQL and SSE.

---

### The Important Ordering

Notice that we first save the notification:

```python
db.add(notification)
db.commit()
```

and only then send the event:

```python
await manager.send(
    user_id,
    notification_data
)
```

Why?

Because PostgreSQL is our source of truth. We don't want to tell the browser:

> "Your notification exists."

before successfully storing it.

Our flow is:

```mermaid
flowchart TD
    A["Create Notification"]
    --> B["Save to PostgreSQL"]
    --> C["Database Commit Succeeds"]
    --> D["Send SSE Event"]
    --> E["React Receives Notification"]
```

---

### What If the User Is Not Connected?

Suppose Alice closes her browser. Her SSE connection disappears. Then someone creates a notification for Alice.

The server executes:

```python
await manager.send(user_id, notification)
```

The manager checks:

```python
queue = self.connections.get(user_id)
```

There is no connection. Therefore, nothing is pushed. Is the notification lost?

No.

It is already stored in PostgreSQL. This is another reason why the database must remain the source of truth.

When Alice opens the application again, React can fetch her existing notifications using a normal HTTP endpoint.

For example:

```text
GET /notifications
```

The architecture is therefore:

```mermaid
flowchart TD
    A["PostgreSQL"] --> B["Initial Notification Fetch"]
    A --> C["SSE Stream"]

    B --> D["React"]
    C --> E["Real-time Updates"]

    E --> D
```

SSE is therefore not a replacement for your notification API.

It complements it.

---

### Should We Still Have `GET /notifications`?

Yes. This is an important design point.

We can have:

```text
GET /notifications
```

for retrieving existing notifications.

And:

```text
GET /notifications/stream
```

for receiving new notifications in real time.

For example:

```mermaid
flowchart LR

    React -->|GET /notifications| API[FastAPI]
    API --> DB[(PostgreSQL)]
    DB --> API
    API --> React

    React -->|GET /notifications/stream| SSE[FastAPI SSE]
    SSE --> React
```

The first endpoint answers:

> "What notifications do I already have?"

The SSE endpoint answers:

> "Tell me whenever a new notification arrives."

Both serve different purposes.

---

### React: Connecting to SSE

Browsers provide a built-in API called **`EventSource`** for SSE.

We can use it in React.

```jsx
import { useEffect, useState } from "react";


function Notifications({ userId }) {

    const [notifications, setNotifications] = useState([]);


    useEffect(() => {

        const eventSource = new EventSource(
            `http://localhost:8000/notifications/stream?user_id=${userId}`
        );


        eventSource.addEventListener(
            "notification",
            (event) => {

                const notification =
                    JSON.parse(event.data);


                setNotifications(
                    previous => [
                        notification,
                        ...previous
                    ]
                );
            }
        );


        eventSource.onerror = () => {

            console.log(
                "SSE connection error"
            );

        };


        return () => {

            eventSource.close();

        };

    }, [userId]);


    return (
        <div>

            <h2>Notifications</h2>

            {notifications.map(
                notification => (

                    <div key={notification.id}>
                        {notification.message}
                    </div>

                )
            )}

        </div>
    );
}


export default Notifications;
```

That's all we need on the frontend to establish the basic SSE connection.

---

### What Does `EventSource` Do?

This line:

```javascript
const eventSource = new EventSource(
    "/notifications/stream"
);
```

opens an SSE connection.

Conceptually:

```mermaid
flowchart TD
    A["React"] -->|"GET /notifications/stream"| B["FastAPI"]

    B -->|"connection stays open"| C["SSE Stream"]

    C -->|"event"| A
    C -->|"event"| A
    C -->|"event"| A
```

The browser handles the SSE protocol for us.

We don't have to manually implement the underlying connection management.

---

### Listening for Events

Our server sends:

```text
event: notification
data: {...}
```

Therefore React can listen for:

```javascript
eventSource.addEventListener(
    "notification",
    (event) => {

        const notification =
            JSON.parse(event.data);

    }
);
```

The browser gives us the event data. For example:

```json
{
    "id": 15,
    "user_id": 1,
    "message": "You have a new friend request"
}
```

We then update React's state.

```javascript
setNotifications(
    previous => [
        notification,
        ...previous
    ]
);
```

React re-renders the UI. The user sees the notification.

---

### The Complete Flow

Suppose Alice is logged in.

1. **React loads existing notifications**

    ```mermaid
   flowchart TD
    A["React"] -->|"GET /notifications"| B["FastAPI"]
    B --> C["PostgreSQL"]
    C --> D["Existing Notifications"]
    D --> A
    ```

2. **React opens the SSE connection**

    ```mermaid
    flowchart TD
    A["React"] -->|"GET /notifications/stream"| B["FastAPI"]
    B --> C["Notification Manager"]
    ```

    The connection remains open.

3. **A new notification is created**

    ```mermaid
    flowchart TD
    A["Something Happens"] --> B["FastAPI"]

    B -->|"INSERT"| C["PostgreSQL"]

    B --> D["Notification Manager"]
    ```

4. **Notification Manager sends it**

    ```mermaid
    flowchart TD
    A["Notification Manager"]
    --> B["Alice's SSE Connections"]
    --> C["React"]
    --> D["Update UI"]
    ```

    The entire system looks like this:

    ```mermaid
    flowchart TB

        React1[React - Chrome]
        React2[React - Safari]
        React3[React - Another Tab]

        API[FastAPI Server]

        Manager[Notification Manager]

        DB[(PostgreSQL)]

        React1 -->|SSE connection| API
        React2 -->|SSE connection| API
        React3 -->|SSE connection| API

        API --> Manager

        API -->|INSERT notification| DB
        DB --> API

        API -->|Publish notification| Manager

        Manager -->|Event| React1
        Manager -->|Event| React2
        Manager -->|Event| React3
    ```

---

### Why This Is Better Than Polling

Let's compare the two approaches.

#### Polling

```mermaid
sequenceDiagram
    participant React
    participant Server
    participant DB

    loop Every 5 minutes
        React->>Server: Any new notifications?
        Server->>DB: Check database
        DB-->>Server: Nothing new
        Server-->>React: No
    end
```

Most requests may return:

```text
Nothing new.
```

#### SSE

```mermaid
sequenceDiagram
    participant React
    participant Server

    React->>Server: Open notification stream

    Note over React,Server: Connection stays open

    Note over Server: Nothing happens...

    Server-->>React: New notification!

    Note over Server: Nothing happens...

    Server-->>React: Another notification!
```

The server does not need to repeatedly answer requests when nothing has happened.

---

### But Does SSE Constantly Send Data?

No. This is another common misunderstanding.

A connection being open does not mean that the server is continuously sending application data.

It can be:

```mermaid
flowchart TD
    A["Connection"]
    --> B["Waiting..."]
    B --> C["Waiting..."]
    C --> D["Waiting..."]
    D --> E["New Notification"]
    E --> F["Send Event"]
```

The server sends application events when they are available.

In production systems, you may also send a small **heartbeat** periodically.

A heartbeat is simply a small message used to make sure that the connection is still alive and that intermediaries such as proxies do not close an otherwise idle connection.

---

### Authentication

Our example uses:

```text
?user_id=1
```

This is only for demonstrating the concept.

We should **not** trust a user ID supplied by the browser like this in a real application.

A malicious user could try:

```text
/notifications/stream?user_id=2
```

and attempt to receive another user's notifications.

Instead, the server should determine the user from the authenticated request.

For example:

```mermaid
flowchart TD
    A["Browser"]
    -->|"Authenticated Request"| B["FastAPI"]
    B --> C["Authenticated User = Alice"]
```

Then:

```python
user_id = current_user.id
```

rather than:

```python
user_id = request.query_params["user_id"]
```

The same security principle applies to normal notification APIs.

---

### Where Does the Notification Come From?

At this point, someone might ask:

> "How does the Notification Manager know that a new notification was created?"

There are several possible designs.

For our example, the same FastAPI application creates the notification.

So we can do:

```mermaid
flowchart TD
    A["Create Notification"]
    A --> B["PostgreSQL"]
    A --> C["Notification Manager"]
```

This is simple and works well when our application is small and has one server instance.

But there is another technology in PostgreSQL called:

> **LISTEN / NOTIFY**

It can allow PostgreSQL to notify a process when a database notification is published.

However, we don't actually need PostgreSQL `LISTEN / NOTIFY` for this particular architecture.

Our FastAPI application already knows that it created the notification. So it can directly publish the event to the Notification Manager.

---

### Why Not Just Read PostgreSQL Continuously?

We could theoretically have our server repeatedly check PostgreSQL:

```mermaid
flowchart TD
    A["Check DB"]
    --> B["Anything New?"]
    B --> C["Check DB"]
    C --> D["Anything New?"]
    D --> E["Check DB"]
    E --> F["..."]
```

But then we have simply moved polling from the browser to the server.

That's usually unnecessary.

If the FastAPI application itself creates the notification, it already knows about the event.

Therefore:

```mermaid
flowchart TD
    A["Create Notification"]
    A --> B["Save to DB"]
    A --> C["Push to Notification Manager"]
```

is much simpler.

---

# The Biggest Limitation: Multiple Server Instances

Now we reach the most important architectural limitation.

Everything we have built so far assumes:

> **There is only one FastAPI server instance.**

Why?

Because our Notification Manager exists in the memory of that one process.

Imagine:

```mermaid
flowchart TD
    A["Load Balancer"] --> B["FastAPI A"]
    A --> C["FastAPI B"]

    B --> D["Manager A"]
    C --> E["Manager B"]
```

A **load balancer** is a component that distributes incoming requests across multiple servers.

Now suppose Alice's SSE connection happens to go to:

```text
FastAPI A
```

So:

```mermaid
flowchart TD
    A["Alice"] --> B["FastAPI A"]
    B --> C["Manager A"]
```

Alice's active connections are stored inside Manager A. 
For example:
```text
Manager A

Alice
 ├── Chrome
 ├── Safari
 └── Another Tab

Bob
 └── Chrome
```

Now another request creates Alice's notification. The load balancer sends that request to:

```text
FastAPI B
```

FastAPI B saves the notification:

```mermaid
flowchart TD
    A["FastAPI B"] --> B["PostgreSQL"]
```

Then FastAPI B tries:

```python
manager.send(alice_id, notification)
```

If Alice's Chrome connection is connected to FastAPI A, but a notification is created through FastAPI B, FastAPI B cannot directly access Alice's connections because Manager A and Manager B are separate in-memory objects.

So Manager B says:

```text
"I don't have Alice's connection."
```

The notification is in PostgreSQL, but the real-time event isn't delivered through FastAPI B to Alice.

This is the fundamental scaling problem.

---

### Visualizing the Problem

```mermaid
flowchart TB

    Browser1[Alice - Chrome]
    Browser2[Alice - Safari]

    LB[Load Balancer]

    A[FastAPI Instance A]
    B[FastAPI Instance B]

    MA[Manager A]
    MB[Manager B]

    DB[(PostgreSQL)]

    Browser1 -->|SSE| LB
    Browser2 -->|SSE| LB

    LB --> A
    LB --> B

    A --> MA
    B --> MB

    MA -. "Alice's Chrome connection" .-> Browser1
    MB -. "Alice's Safari connection" .-> Browser2

    B -->|Create notification| DB
    B --> MB

    Note[Problem: connections are stored in separate server memories]

    MB -.-> Note
```

The problem is not PostgreSQL.

The problem is that:

> **The active SSE connection exists only inside one server's memory.**

---

### How Do We Solve the Multiple-Instance Problem?

When we have multiple server instances, we generally need a **shared messaging system**.

A messaging system is a component that allows one server to publish a message and other servers to receive it.

Common choices include:

* Redis Pub/Sub
* Apache Kafka
* RabbitMQ
* other message brokers

For example:

```mermaid
flowchart TB

    React1[React Browser]
    React2[React Browser]

    A[FastAPI A]
    B[FastAPI B]

    Redis[Redis Pub/Sub]

    DB[(PostgreSQL)]

    React1 -->|SSE| A
    React2 -->|SSE| B

    A --> Redis
    B --> Redis

    A --> DB
    B --> DB

    Redis --> A
    Redis --> B
```

Now suppose Alice is connected to Server A. Another request reaches Server B and creates Alice's notification.

Server B:

```mermaid
flowchart TD
    A["Save Notification"]
    --> B["Publish Event to Redis"]
```

Server A is subscribed to Redis:

```mermaid
flowchart TD
    A["Redis"]
    --> B["Server A"]
    --> C["Alice's SSE Connection"]
    --> D["Browser"]
```

Now it doesn't matter which server receives the request that creates the notification. The shared messaging system connects the servers.

---

### SSE Itself Isn't the Scaling Problem

It is tempting to say:

> "SSE doesn't work with multiple servers."

That's not quite correct. SSE can absolutely be used with multiple servers.

The problem is our **in-memory connection manager**.

A single process's memory is not shared with other processes. We therefore need to separate two responsibilities:

```text
Connection management
        +
Event distribution
```

SSE handles:

```mermaid
flowchart LR
    A["Server"] --> B["Browser"]
```

A shared messaging system handles:

```mermaid
flowchart LR
    A["Server A"] <--> B["Server B"] <--> C["Server C"]
```

So a scalable architecture might be:

```mermaid
flowchart TD
    R["Redis<br/>Pub/Sub"]

    R --> A["FastAPI A"]
    R --> B["FastAPI B"]
    R --> C["FastAPI C"]

    A --> A1["SSE"]
    B --> B1["SSE"]
    C --> C1["SSE"]

    A1 --> A2["Browsers"]
    B1 --> B2["Browsers"]
    C1 --> C2["Browsers"]
```

---

### When Should You Use SSE?

SSE is a good choice when:

1. **Communication is primarily server to client**

    For example:

    ```text
    Server to Notification
    Server to Progress update
    Server to Status change
    ```

2. **You want real-time updates**

    For example:

    ```text
    Notification appears immediately.
    ```

3. **You don't need two-way communication**

    If the browser doesn't need to continuously send messages back through the same connection, SSE can be simpler than WebSockets.

4. **The data is naturally event-based**

    Examples:

    ```text
    New notification
    New activity
    Job completed
    Build finished
    Processing progress changed
    ```

---

### When Is SSE Not a Good Choice?

SSE is not automatically the best solution for every real-time application.

1. **You need continuous two-way communication**

    For example:

    ```text
    Online multiplayer game
    Real-time collaborative editing
    Interactive chat protocol
    ```

    where both sides continuously send messages.

    WebSockets may be more appropriate.

    ---

2. **You need extremely high-frequency communication**

    If the server is sending a very large number of messages continuously, the architecture needs careful consideration.

    SSE is designed for server-to-client event streams, not arbitrary high-frequency bidirectional communication.

    ---

3. **Your application has multiple server instances but no shared event mechanism**

    This is especially relevant to our architecture.

    If you have:

    ```text
    FastAPI A
    FastAPI B
    FastAPI C
    ```

    and every server keeps connections only in local memory, then an event created on A may not reach a user connected to B.

    You need something shared, such as:

    ```text
    Redis
    Kafka
    RabbitMQ
    ```

    depending on the requirements.

    ---

4. **You don't actually need real-time updates**

    If checking once every few minutes is perfectly acceptable, polling may be simpler.

    Don't introduce SSE simply because it sounds more advanced.

    Choose the simplest architecture that satisfies the requirement.

---

### SSE vs Polling vs WebSockets

A simple comparison:

| Feature                 | Polling     | SSE         | WebSocket   |
| ----------------------- | ----------- | ----------- | ----------- |
| Server → Client         | Yes         | Yes         | Yes         |
| Client → Server         | Normal HTTP | Normal HTTP | Yes         |
| Persistent connection   | No          | Yes         | Yes         |
| Real-time updates       | Limited     | Yes         | Yes         |
| Two-way communication   | No          | No          | Yes         |
| Browser support         | Excellent   | Excellent   | Excellent   |
| Complexity              | Low         | Low/Medium  | Medium/High |
| Great for notifications | Works       | Excellent   | Works       |
| Great for multiplayer   | Poor        | Poor        | Excellent   |

> "It is not about which mechanism is better, it is about what your application really needs."

---

### Conclusion

Server-Sent Events are a relatively simple way to add real-time, server-to-browser communication to a web application.

Instead of repeatedly asking:

```text
"Do I have a new notification?"
```

the browser can establish a long-lived connection:

```text
"Keep this connection open and tell me when something happens."
```

The server can then push events through that connection.

For our single-server application:

```mermaid
flowchart TD
    A["React"]
    --> B["SSE Connection"]
    --> C["FastAPI"]
    --> D["Notification Manager"]
    --> A
```

while PostgreSQL remains the persistent source of truth.

The architecture is simple and effective when there is only one server instance.

However, when the application grows to multiple server instances, the in-memory Notification Manager becomes a problem because each server has its own memory.

At that point, we need a shared event-distribution mechanism such as Redis Pub/Sub, Kafka, or another messaging system.

So SSE is not simply:

> "A technology for real-time communication."

It is better understood as:

> **A mechanism for maintaining a long-lived HTTP connection through which a server can push events to a browser.**

And once you understand that, the architecture becomes much easier to design.