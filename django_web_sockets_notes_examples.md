# Django + WebSockets — Notes, Examples & Line-by-line Explanations

> Concise, practical notes to get WebSockets working in Django (with Channels). This version expands the previous document with **file-level and line-by-line explanations** for the main code files so you can understand exactly what each line does.

---

## Table of contents
1. Overview
2. Files we'll explain line-by-line
3. `asgi.py` — full file + line-by-line explanation
4. `chat/routing.py` — full file + explanation
5. `chat/consumers.py` — full file + detailed line-by-line explanation
6. `settings.py` snippets (ASGI + CHANNEL_LAYERS) — explanation
7. Frontend `chat` JavaScript — explanation
8. TokenAuthMiddleware outline — explanation
9. Notes about async <-> sync and database calls
10. Quick checklist & next steps

---

## 1. Overview (short)
* **WebSocket**: a persistent, full-duplex connection between client and server. Good for chat, live notifications, real-time updates.
* **Django + WebSocket**: Django's standard request/response (WSGI) is synchronous and short-lived. For WebSockets we use **ASGI** (async) and **Django Channels** to handle long-lived socket connections, background tasks, and channel layers (Redis).

---
### When to use

* Chat apps, live notifications, collaborative editing, real-time dashboards.

---
## 2. Key components

* **ASGI application**: entry point for WebSocket & HTTP in async servers (Daphne, Uvicorn).
* **Django Channels** (`channels`): integrates ASGI into Django, provides `Consumer` classes and channel layers.
* **Consumers**: like Django views but for WebSocket/async events. Two flavors: `AsyncConsumer` (async) and `SyncConsumer` (sync).
* **Channel layer**: message transport used by multiple consumers/processes — commonly Redis (via `channels_redis`).
* **Routing**: maps URL paths to Consumers (similar to `urls.py`).

---

1. Create virtualenv and install:

```bash
python -m venv venv
source venv/bin/activate
pip install django channels channels_redis daphne
```
2. Add `channels` to `INSTALLED_APPS` in `settings.py`.

3. Set ASGI application in `settings.py`:

```py
ASGI_APPLICATION = 'project_name.asgi.application'
```

4. Configure channel layer (Redis):

```py
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {
            'hosts': [('127.0.0.1', 6379)],
        },
    },
}
```

(Install and run Redis: `sudo apt install redis-server` or use Docker.)

---

## 4. Project layout (example)

```
project_name/
├─ project_name/
│  ├─ asgi.py
│  ├─ settings.py
│  └─ urls.py
├─ chat/
│  ├─ consumers.py
│  ├─ routing.py
│  ├─ templates/chat/
│  └─ static/chat/
└─ manage.py
```
---

## 3. `asgi.py` — file + line-by-line explanation

**Full file**

```py
import os
from django.core.asgi import get_asgi_application
from channels.routing import ProtocolTypeRouter, URLRouter
import chat.routing
from channels.auth import AuthMiddlewareStack

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'project_name.settings')

django_asgi_app = get_asgi_application()

application = ProtocolTypeRouter({
    "http": django_asgi_app,
    "websocket": AuthMiddlewareStack(
        URLRouter(
            chat.routing.websocket_urlpatterns
        )
    ),
})
```

**Line-by-line explanation**

- `import os`  
  Standard Python module used to access environment variables. ASGI entrypoints commonly set environment vars for Django settings.

- `from django.core.asgi import get_asgi_application`  
  Imports Django helper that returns the ASGI callable that handles normal HTTP requests (the ASGI equivalent of `get_wsgi_application`). This ensures traditional Django views continue to work under ASGI.

- `from channels.routing import ProtocolTypeRouter, URLRouter`  
  `ProtocolTypeRouter` lets you route by connection type (http, websocket, etc.). `URLRouter` maps path patterns to ASGI application callables (similar to `urlpatterns`).

- `import chat.routing`  
  Import your app's routing module which defines `websocket_urlpatterns`. We import the module so we can reference those patterns below.

- `from channels.auth import AuthMiddlewareStack`  
  This middleware stack reads the Django session cookie and populates `scope['user']` with Django `User` for each connection, so your consumers can access authentication state.

- `os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'project_name.settings')`  
  Ensures Django knows which settings module to use when starting the ASGI app. `setdefault` only assigns if not already set externally.

- `django_asgi_app = get_asgi_application()`  
  Create the ASGI application that handles HTTP. We save it to `django_asgi_app` so we can route HTTP requests to the normal Django stack.

- `application = ProtocolTypeRouter({...})`  
  Define the top-level ASGI `application` variable. ASGI servers look for this callable. Inside:
  - `"http": django_asgi_app` routes normal HTTP traffic to standard Django.
  - `"websocket": AuthMiddlewareStack(URLRouter(...))` wraps websocket routes with authentication middleware and URL routing. The nested `URLRouter` uses `chat.routing.websocket_urlpatterns` to match websocket paths to consumers.

**Why structured this way?**  
This composition lets a single process serve both HTTP and WebSocket connections. The `AuthMiddlewareStack` is optional — use it when you want `scope['user']` populated from session/cookies.

---

## 4. `chat/routing.py` — file + explanation

**Full file**

```py
from django.urls import re_path
from . import consumers

websocket_urlpatterns = [
    re_path(r'ws/chat/(?P<room_name>[^/]+)/$', consumers.ChatConsumer.as_asgi()),
]
```

**Explanation**

- `from django.urls import re_path`  
  Use `re_path` for regex-based URL matching (works well for websockets where you may want groups like `room_name`).

- `from . import consumers`  
  Import the consumers module where `ChatConsumer` is defined.

- `websocket_urlpatterns = [...]`  
  A list of websocket URL patterns (analogous to `urlpatterns` for HTTP). The ASGI `URLRouter` in `asgi.py` expects a sequence named like this.

- `re_path(r'ws/chat/(?P<room_name>[^/]+)/$', consumers.ChatConsumer.as_asgi())`  
  Pattern explanation:
  - `ws/chat/` — base path
  - `(?P<room_name>[^/]+)` — capture group named `room_name` matching one or more characters that are not `/`.
  - `$` end of string
  - `ChatConsumer.as_asgi()` returns an ASGI application instance for that consumer class (channels converts your class into a callable).

**Note:** You can have multiple websocket patterns (for different features like notifications, live-edit rooms, etc.).

---

## 5. `chat/consumers.py` — file + detailed line-by-line explanation

**Full file**

```py
import json
from channels.generic.websocket import AsyncWebsocketConsumer

class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.room_name = self.scope['url_route']['kwargs']['room_name']
        self.room_group_name = f'chat_{self.room_name}'

        # Join room group
        await self.channel_layer.group_add(
            self.room_group_name,
            self.channel_name
        )

        await self.accept()

    async def disconnect(self, close_code):
        # Leave room group
        await self.channel_layer.group_discard(
            self.room_group_name,
            self.channel_name
        )

    # Receive message from WebSocket
    async def receive(self, text_data=None, bytes_data=None):
        if text_data is None:
            return
        data = json.loads(text_data)
        message = data.get('message')
        username = self.scope['user'].username if self.scope.get('user') and self.scope['user'].is_authenticated else 'Anon'

        # Broadcast to group
        await self.channel_layer.group_send(
            self.room_group_name,
            {
                'type': 'chat_message',
                'message': message,
                'username': username,
            }
        )

    # Receive message from group
    async def chat_message(self, event):
        await self.send(text_data=json.dumps({
            'message': event['message'],
            'username': event['username'],
        }))
```

**Line-by-line explanation**

- `import json`  
  We'll encode/decode JSON strings when sending/receiving messages. WebSocket frames are strings/bytes; JSON is an easy structured format.

- `from channels.generic.websocket import AsyncWebsocketConsumer`  
  Import a helpful base class that wires up the ASGI mechanics for WebSocket consumers. It's async-friendly and gives you `connect`, `receive`, `disconnect`, and `send` helpers.

- `class ChatConsumer(AsyncWebsocketConsumer):`  
  Define a consumer for chat. Using `AsyncWebsocketConsumer` means handler methods can be `async def`.

### `connect` method

- `async def connect(self):`  
  Called automatically when a client initiates a WebSocket connection.

- `self.room_name = self.scope['url_route']['kwargs']['room_name']`  
  `scope['url_route']['kwargs']` is populated from the `re_path` group in routing. This reads which room the client requested.

- `self.room_group_name = f'chat_{self.room_name}'`  
  Create a group name (prefixing reduces collision risk). Groups are logical channels in `channel_layer` where many connections can join and receive the same messages.

- `await self.channel_layer.group_add(self.room_group_name, self.channel_name)`  
  Add this connection's `channel_name` to the named group. `channel_name` is a unique string representing this connection within Channels.

- `await self.accept()`  
  Accept the WebSocket handshake. If you don't call accept (or you call `close()`), the connection won't be established.

### `disconnect` method

- `async def disconnect(self, close_code):`  
  Called when connection closes (client disconnects or server closes). `close_code` mirrors the WebSocket close code.

- `await self.channel_layer.group_discard(self.room_group_name, self.channel_name)`  
  Remove channel from group — helps cleanup and prevents memory leaks.

### `receive` method

- `async def receive(self, text_data=None, bytes_data=None):`  
  Called when a WebSocket frame is received. Channels passes text or bytes depending on frame type.

- `if text_data is None: return`  
  Safety: if only bytes are sent or no payload, ignore. (You can handle `bytes_data` if using binary protocols.)

- `data = json.loads(text_data)`  
  Parse JSON from client. We expect `{'message': '...'} ` but you can design richer shapes.

- `message = data.get('message')`  
  Extract the message content safely.

- `username = self.scope['user'].username if self.scope.get('user') and self.scope['user'].is_authenticated else 'Anon'`  
  If session-authenticated, use `scope['user']`. `AuthMiddlewareStack` must be in ASGI stack for this to work. Otherwise default to `'Anon'`.

- `await self.channel_layer.group_send(self.room_group_name, { 'type': 'chat_message', 'message': message, 'username': username, })`  
  Send a message to the whole group. `type: 'chat_message'` means Channels will call the `chat_message` method on *each* consumer in that group, passing the event dict.

### `chat_message` handler

- `async def chat_message(self, event):`  
  When `group_send` with `type: 'chat_message'` is received by this consumer, Channels dispatches to this method with `event` as the payload.

- `await self.send(text_data=json.dumps({'message': event['message'], 'username': event['username'], }))`  
  Forward the message to the connected WebSocket client as JSON.

**Important notes about this consumer**
- DB calls: If you need to read/write the database in `receive` or `connect`, wrap those calls with `database_sync_to_async` or use `sync_to_async` to avoid blocking the event loop.
- Error handling: Add try/except around parsing and sending to avoid crashing the consumer on malformed messages.

---

## 6. `settings.py` snippets and explanation

**ASGI_APPLICATION**

```py
ASGI_APPLICATION = 'project_name.asgi.application'
```

- This tells Django/Channels what variable to import as the ASGI application when running under an ASGI server. It's analogous to `WSGI_APPLICATION` but for ASGI.

**CHANNEL_LAYERS (Redis example)**

```py
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {
            'hosts': [('127.0.0.1', 6379)],
        },
    },
}
```

- `BACKEND`: Implementation for channel layer — `channels_redis` is standard and uses Redis pub/sub.
- `hosts`: Redis host(s) and port(s). For production use managed Redis or a secured service.

**INSTALLED_APPS** must include `'channels'` so Django recognizes the package and patches runserver for Channels during development.

---

## 7. Frontend JavaScript — explanation

**Full snippet**

```html
<script>
const roomName = 'lobby'; // generate dynamically
const wsScheme = window.location.protocol === 'https:' ? 'wss' : 'ws';
const chatSocket = new WebSocket(`${wsScheme}://${window.location.host}/ws/chat/${roomName}/`);

chatSocket.onopen = function(e) {
    console.log('connected');
};

chatSocket.onmessage = function(e) {
    const data = JSON.parse(e.data);
    console.log(data.username + ': ' + data.message);
};

chatSocket.onclose = function(e) {
    console.log('closed');
};

function sendMessage(text){
    chatSocket.send(JSON.stringify({ 'message': text }));
}
</script>
```

**Line-by-line**

- `const roomName = 'lobby';`  
  Replace with a variable derived from the UI or URL so users join the intended room.

- `const wsScheme = window.location.protocol === 'https:' ? 'wss' : 'ws';`  
  Choose secure WebSocket (`wss`) for HTTPS pages — browsers require WSS on secure origins.

- `const chatSocket = new WebSocket(`${wsScheme}://${window.location.host}/ws/chat/${roomName}/`);`  
  Create the WebSocket object. `window.location.host` includes host and port.

- `chatSocket.onopen = ...`  
  Called when connection opens. Useful for UI state changes (enable send input, show connected status).

- `chatSocket.onmessage = ...`  
  Called when a message arrives. `e.data` is the string payload from server — parse JSON and update UI accordingly.

- `chatSocket.onclose = ...`  
  Called when socket closes — update UI and optionally attempt reconnects.

- `function sendMessage(text) { chatSocket.send(JSON.stringify({ 'message': text })); }`  
  Sends a JSON object to server. Keep payloads small; consider adding `type` fields for richer message handling.

**Authentication note**: If your frontend is same-origin, the browser will attach session cookies automatically to the WebSocket handshake. For cross-origin or mobile clients, you'll need token-based auth.

---

## 8. TokenAuthMiddleware outline — explanation

If you want token-based authentication (e.g., JWT or custom tokens) instead of session cookies, you'll need a middleware that extracts the token from the query string or headers and populates `scope['user']`.

**Outline**

```py
from urllib.parse import parse_qs
from channels.middleware import BaseMiddleware
from django.contrib.auth import get_user_model
from django.db import close_old_connections
from django.contrib.auth.models import AnonymousUser

User = get_user_model()

class TokenAuthMiddleware(BaseMiddleware):
    async def __call__(self, scope, receive, send):
        # Parse query string
        query_string = scope.get('query_string', b'').decode()
        qs = parse_qs(query_string)
        token_list = qs.get('token')
        if token_list:
            token = token_list[0]
            # Verify token (e.g., look up in DB or decode JWT)
            # If valid, set: scope['user'] = user
            # else: scope['user'] = AnonymousUser()
        else:
            scope['user'] = AnonymousUser()

        return await super().__call__(scope, receive, send)
```

**Key points**
- `parse_qs` converts `a=1&b=2` into dict `{ 'a': ['1'], 'b': ['2'] }`.
- Make sure to `close_old_connections()` before doing DB lookups to avoid connection leaks (use when using sync DB in async context).
- For JWT, instead of DB lookups you might decode token and create a `SimpleLazyObject` user or a custom user instance.
- Wrap `TokenAuthMiddleware` around `URLRouter` in `asgi.py` like: `TokenAuthMiddleware(AuthMiddlewareStack(URLRouter(...)))` or `AuthMiddlewareStack(TokenAuthMiddleware(URLRouter(...)))` depending on desired behavior.

---

## 9. Async ⇄ Sync & Database access

Django's ORM is synchronous. In async consumers, don't call ORM directly. Options:
- `from channels.db import database_sync_to_async` and wrap sync DB functions:
  ```py
  from channels.db import database_sync_to_async

  @database_sync_to_async
  def get_user_profile(user_id):
      return Profile.objects.get(user_id=user_id)
  ```
- Or use `async_to_sync` in rare cases when you're in sync code calling async.

**Why:** Directly calling sync ORM in an async loop blocks the whole worker and reduces throughput.

---

## 10. Deployment Checklist

### ASGI Server

Run with Daphne or Uvicorn:

```bash
uvicorn project_name.asgi:application --host 0.0.0.0 --port 8000
```
## 11. What `install_websocket.bat` Does

* Automatically installs WebSocket-related Python packages:

  * **Channels** → Adds ASGI + WebSocket support in Django.
  * **Daphne** → ASGI server that handles HTTP + WebSocket together.
  * **Channels-Redis** → Enables real-time messaging across multiple users.
* Runs Django migrations.
* Shows the command to start server.
* After Daphne gets installed, Django's `runserver` automatically switches to **ASGI mode**.

## 12. Why This File Is Useful

* Saves time.
* Avoids manual installation.
* Makes setup consistent.
* Provides a one-click WebSocket environment.

---

## 13. Code Explanation (Line-by-Line)

### `@echo off`

Hides unnecessary command output and keeps console clean.

### Heading Lines

Shows a clean title message to the user.

### Python Package Installation

```
pip install channels>=4.0.0 daphne>=4.0.0 channels-redis>=4.1.0
```

* **Channels:** Gives WebSocket + ASGI framework.
* **Daphne:** ASGI server, replaces old WSGI server.
* **Channels-Redis:** Enables pub/sub for real-time chat.

### Running Migrations

```
python manage.py makemigrations
python manage.py migrate
```

* Detects database changes.
* Applies changes to the database.

### Final Instructions

Tells the user to run:

```
python manage.py runserver
```

This now automatically uses ASGI because Daphne is installed.

### `pause`

Stops console from closing so user can read everything.

---

## 14. Difference Between Commands

### **1. `python manage.py runserver`**

* Default Django command.
* Earlier: **WSGI (only HTTP)**
* After installing Daphne + Channels: **Auto ASGI (HTTP + WebSocket)**
* Easy to use.
* Perfect for development.

### **When to use:**

* Local development.
* Testing WebSockets.
* Quick start.

---

### **2. `daphne project_name.asgi:application`**

* Directly starts Daphne ASGI server.
* More powerful.
* Used for production-like environments.
* Does NOT auto-reload like runserver.

### **When to use:**

* Production deployment.
* When using Nginx + Supervisor.
* When you want manual control.

---

## 15. Quick Summary Table

| Command                                  | Mode                       | Use Case                    |
| ---------------------------------------- | -------------------------- | --------------------------- |
| **python manage.py runserver**           | Auto ASGI (after Channels) | Development, Testing        |
| **daphne project_name.asgi:application** | Direct ASGI                | Production, Advanced setups |

---

## 16. Final Note

Because your script installed **Daphne**, Django intelligently switches `runserver` to ASGI mode. That's why you no longer need to manually run `project_name.asgi:application`.


