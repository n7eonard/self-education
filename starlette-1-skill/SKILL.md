# Starlette 1.0 — Complete Feature Skill Guide

> **Version**: 1.0.0 (released March 22, 2026)  
> **Install**: `pip install starlette uvicorn`  
> **Full install** (all optional deps): `pip install starlette[full] uvicorn`  
> **Python**: 3.10+  
> **Source**: https://github.com/encode/starlette

Starlette is a lightweight ASGI framework/toolkit for building async web services in Python. After nearly eight years, version 1.0 marks the first stable release. It underpins FastAPI, the Python MCP SDK, and serves ~10 million PyPI downloads/day.

---

## Table of Contents

1. [Application](#1-application)
2. [Routing](#2-routing)
3. [Requests](#3-requests)
4. [Responses](#4-responses)
5. [WebSockets](#5-websockets)
6. [Endpoints (Class-Based Views)](#6-endpoints-class-based-views)
7. [Middleware](#7-middleware)
8. [Background Tasks](#8-background-tasks)
9. [Static Files](#9-static-files)
10. [Templates (Jinja2)](#10-templates-jinja2)
11. [Authentication](#11-authentication)
12. [Sessions](#12-sessions)
13. [Exceptions](#13-exceptions)
14. [Lifespan & State](#14-lifespan--state)
15. [Configuration](#15-configuration)
16. [Schemas (OpenAPI)](#16-schemas-openapi)
17. [Test Client](#17-test-client)
18. [Server Push (HTTP/2)](#18-server-push-http2)
19. [Thread Pool](#19-thread-pool)
20. [Pure ASGI Usage](#20-pure-asgi-usage)
21. [What Changed in 1.0 (Breaking Removals)](#21-what-changed-in-10-breaking-removals)

---

## 1. Application

The `Starlette` class ties everything together: routes, middleware, exception handlers, and the lifespan.

```python
from contextlib import asynccontextmanager
from starlette.applications import Starlette
from starlette.responses import PlainTextResponse, JSONResponse
from starlette.routing import Route, Mount, WebSocketRoute
from starlette.staticfiles import StaticFiles
from starlette.middleware import Middleware
from starlette.middleware.cors import CORSMiddleware


def homepage(request):
    return PlainTextResponse("Hello, world!")

def user(request):
    username = request.path_params["username"]
    return PlainTextResponse(f"Hello, {username}!")

async def websocket_endpoint(websocket):
    await websocket.accept()
    await websocket.send_text("Hello, websocket!")
    await websocket.close()

@asynccontextmanager
async def lifespan(app):
    print("Startup")
    yield
    print("Shutdown")


routes = [
    Route("/", homepage),
    Route("/user/{username}", user),
    WebSocketRoute("/ws", websocket_endpoint),
    Mount("/static", StaticFiles(directory="static")),
]

middleware = [
    Middleware(CORSMiddleware, allow_origins=["*"]),
]

app = Starlette(
    debug=True,
    routes=routes,
    middleware=middleware,
    lifespan=lifespan,
)
```

### Storing arbitrary state on the app

```python
app.state.ADMIN_EMAIL = "admin@example.org"

# Access it from any endpoint:
async def admin_info(request):
    return JSONResponse({"admin": request.app.state.ADMIN_EMAIL})
```

---

## 2. Routing

### Basic routes

```python
from starlette.routing import Route, Mount, WebSocketRoute, Router
from starlette.responses import PlainTextResponse

async def homepage(request):
    return PlainTextResponse("Homepage")

async def about(request):
    return PlainTextResponse("About")

routes = [
    Route("/", homepage),
    Route("/about", about),
]
```

### Path parameters & convertors

Available convertors: `str` (default), `int`, `float`, `uuid`, `path`.

```python
async def user(request):
    user_id = request.path_params["user_id"]       # int
    return PlainTextResponse(str(user_id))

async def catch_all(request):
    rest = request.path_params["rest_of_path"]     # includes slashes
    return PlainTextResponse(rest)

routes = [
    Route("/users/{user_id:int}", user),
    Route("/files/{rest_of_path:path}", catch_all),
]
```

### Custom convertor

```python
from datetime import datetime
from starlette.convertors import Convertor, register_url_convertor

class DateTimeConvertor(Convertor):
    regex = r"[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9]{2}:[0-9]{2}:[0-9]{2}"

    def convert(self, value: str) -> datetime:
        return datetime.strptime(value, "%Y-%m-%dT%H:%M:%S")

    def to_string(self, value: datetime) -> str:
        return value.strftime("%Y-%m-%dT%H:%M:%S")

register_url_convertor("datetime", DateTimeConvertor())

routes = [Route("/history/{date:datetime}", history)]
```

### Submounting routes

```python
routes = [
    Route("/", homepage),
    Mount("/users", routes=[
        Route("/", list_users, methods=["GET", "POST"]),
        Route("/{username}", get_user),
    ]),
]
```

### Reverse URL lookups

```python
# Named route
routes = [Route("/users/{username}", user, name="user_detail")]

# In an endpoint (has request):
url = request.url_for("user_detail", username="tom")

# Without a request (returns path only):
url = app.url_path_for("user_detail", username="tom")
```

### Host-based routing

```python
from starlette.routing import Host

routes = [
    Host("api.example.org", app=api_router, name="api"),
    Host("{subdomain}.example.org", app=subdomain_router, name="sub"),
]
```

### Middleware per-route / per-mount

```python
from starlette.middleware.gzip import GZipMiddleware

routes = [
    Route("/heavy", heavy_endpoint, middleware=[Middleware(GZipMiddleware)]),
    Mount("/api", routes=api_routes, middleware=[Middleware(GZipMiddleware)]),
]
```

### Bare Router (no middleware wrapping)

```python
app = Router(routes=[
    Route("/", homepage),
    Mount("/users", routes=user_routes),
])
```

---

## 3. Requests

```python
from starlette.requests import Request
from starlette.responses import JSONResponse

async def inspect(request: Request):
    return JSONResponse({
        "method":  request.method,
        "path":    request.url.path,
        "scheme":  request.url.scheme,
        "host":    request.url.hostname,
        "query":   dict(request.query_params),
        "headers": dict(request.headers),
        "client":  str(request.client),
    })
```

### Body variants

```python
# Raw bytes
body = await request.body()

# JSON
data = await request.json()

# Streaming (no buffering)
async def stream_body(request):
    body = b""
    async for chunk in request.stream():
        body += chunk
    return PlainTextResponse(f"Got {len(body)} bytes")

# Disconnect detection (long-polling)
disconnected = await request.is_disconnected()
```

### Form data & file uploads

```python
async def upload(request):
    async with request.form(max_files=10, max_fields=20, max_part_size=5*1024*1024) as form:
        name   = form["name"]                         # str field
        upload = form["file"]                         # UploadFile
        filename = upload.filename
        content  = await upload.read()
        size     = upload.size                        # bytes
    return JSONResponse({"filename": filename, "size": size})
```

### Cookies

```python
session_id = request.cookies.get("session_id")
```

### Request state (arbitrary per-request data)

```python
request.state.time_started = time.time()
# Later in the same request lifecycle:
elapsed = time.time() - request.state.time_started
```

---

## 4. Responses

### Response types

```python
from starlette.responses import (
    Response, HTMLResponse, PlainTextResponse,
    JSONResponse, RedirectResponse, StreamingResponse, FileResponse,
)

# Plain base response
Response("hello", status_code=200, media_type="text/plain")

# HTML
HTMLResponse("<h1>Hello</h1>")

# Plain text
PlainTextResponse("Hello")

# JSON
JSONResponse({"key": "value"})

# Redirect (307 by default)
RedirectResponse(url="/new-path")
RedirectResponse(url="/new-path", status_code=301)

# File (with Range request support built-in)
FileResponse("path/to/file.pdf", filename="report.pdf")

# Streaming
async def generator():
    for i in range(5):
        yield f"chunk {i}\n"
        await asyncio.sleep(0.1)

StreamingResponse(generator(), media_type="text/plain")
```

### Cookies

```python
async def set_cookie(request):
    response = JSONResponse({"status": "ok"})
    response.set_cookie(
        key="session",
        value="abc123",
        max_age=3600,
        httponly=True,
        secure=True,
        samesite="lax",
    )
    return response

async def delete_cookie(request):
    response = JSONResponse({"status": "logged out"})
    response.delete_cookie(key="session")
    return response
```

### Custom JSON serializer

```python
import orjson
from starlette.responses import JSONResponse

class OrjsonResponse(JSONResponse):
    def render(self, content) -> bytes:
        return orjson.dumps(content)
```

### FileResponse with HTTP Range

`FileResponse` automatically handles `Range` headers, returning `206 Partial Content` for valid ranges and `416 Range Not Satisfiable` for invalid ones. The `Accept-Ranges: bytes` header is always included.

```python
FileResponse("large_video.mp4")  # Range requests handled automatically
```

---

## 5. WebSockets

```python
from starlette.websockets import WebSocket

async def ws_endpoint(websocket: WebSocket):
    await websocket.accept()

    # Send / receive text
    await websocket.send_text("Hello!")
    msg = await websocket.receive_text()

    # Send / receive bytes
    await websocket.send_bytes(b"\x00\x01")
    data = await websocket.receive_bytes()

    # Send / receive JSON
    await websocket.send_json({"event": "update"})
    payload = await websocket.receive_json()

    # Iterator (exits cleanly on disconnect)
    async for message in websocket.iter_text():
        await websocket.send_text(f"echo: {message}")

    await websocket.close(code=1000, reason="Done")
```

### WebSocket denial response

```python
from starlette.exceptions import HTTPException

async def secure_ws(websocket: WebSocket):
    token = websocket.headers.get("authorization")
    if token != "Bearer secret":
        raise HTTPException(status_code=401, detail="Unauthorized")
    await websocket.accept()
    await websocket.send_text("Welcome!")
    await websocket.close()
```

### Type-safe state via generics (new in 1.0)

```python
from typing import TypedDict
import httpx

class AppState(TypedDict):
    http_client: httpx.AsyncClient

async def ws_with_state(websocket: WebSocket[AppState]):
    await websocket.accept()
    client = websocket.state["http_client"]   # typed as httpx.AsyncClient
    resp = await client.get("https://api.example.com/data")
    await websocket.send_json(resp.json())
    await websocket.close()
```

---

## 6. Endpoints (Class-Based Views)

### HTTPEndpoint

```python
from starlette.endpoints import HTTPEndpoint
from starlette.responses import JSONResponse

class UserEndpoint(HTTPEndpoint):
    async def get(self, request):
        user_id = request.path_params["user_id"]
        return JSONResponse({"id": user_id, "name": "Alice"})

    async def post(self, request):
        data = await request.json()
        return JSONResponse({"created": data}, status_code=201)

    async def delete(self, request):
        return JSONResponse({"deleted": True})

# Unhandled methods get 405 automatically
routes = [Route("/users/{user_id:int}", UserEndpoint)]
```

### WebSocketEndpoint

```python
from starlette.endpoints import WebSocketEndpoint

class EchoEndpoint(WebSocketEndpoint):
    encoding = "text"   # "text" | "bytes" | "json"

    async def on_connect(self, websocket):
        await websocket.accept()

    async def on_receive(self, websocket, data):
        await websocket.send_text(f"Echo: {data}")

    async def on_disconnect(self, websocket, close_code):
        print(f"Disconnected with code {close_code}")

routes = [WebSocketRoute("/ws", EchoEndpoint)]
```

---

## 7. Middleware

### Built-in middleware

```python
from starlette.applications import Starlette
from starlette.middleware import Middleware
from starlette.middleware.cors import CORSMiddleware
from starlette.middleware.sessions import SessionMiddleware
from starlette.middleware.httpsredirect import HTTPSRedirectMiddleware
from starlette.middleware.trustedhost import TrustedHostMiddleware
from starlette.middleware.gzip import GZipMiddleware

middleware = [
    Middleware(TrustedHostMiddleware, allowed_hosts=["example.com", "*.example.com"]),
    Middleware(HTTPSRedirectMiddleware),
    Middleware(CORSMiddleware,
        allow_origins=["https://app.example.com"],
        allow_methods=["GET", "POST", "PUT", "DELETE"],
        allow_headers=["Authorization", "Content-Type"],
        allow_credentials=True,
        allow_private_network=True,   # PNA support (added 0.51)
        max_age=600,
    ),
    Middleware(SessionMiddleware,
        secret_key="super-secret",
        https_only=True,              # Secure flag in production
        max_age=14 * 24 * 3600,
    ),
    Middleware(GZipMiddleware, minimum_size=1000, compresslevel=9),
]

app = Starlette(routes=routes, middleware=middleware)
```

### BaseHTTPMiddleware (simple)

```python
from starlette.middleware.base import BaseHTTPMiddleware

class TimingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        start = time.time()
        response = await call_next(request)
        elapsed = time.time() - start
        response.headers["X-Process-Time"] = str(elapsed)
        return response

class ConfigurableMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, header_value="default"):
        super().__init__(app)
        self.header_value = header_value

    async def dispatch(self, request, call_next):
        response = await call_next(request)
        response.headers["X-Custom"] = self.header_value
        return response
```

### Pure ASGI Middleware (recommended for advanced use)

Avoids `BaseHTTPMiddleware`'s `contextvars` propagation limitation.

```python
from starlette.types import ASGIApp, Scope, Receive, Send, Message
from starlette.datastructures import MutableHeaders

class AddResponseHeaderMiddleware:
    def __init__(self, app: ASGIApp) -> None:
        self.app = app

    async def __call__(self, scope: Scope, receive: Receive, send: Send) -> None:
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return

        async def send_wrapper(message: Message) -> None:
            if message["type"] == "http.response.start":
                headers = MutableHeaders(scope=message)
                headers.append("X-Powered-By", "Starlette")
            await send(message)

        await self.app(scope, receive, send_wrapper)
```

### Redirect middleware (pure ASGI pattern)

```python
from starlette.datastructures import URL
from starlette.responses import RedirectResponse

class RedirectsMiddleware:
    def __init__(self, app, path_mapping: dict):
        self.app = app
        self.path_mapping = path_mapping

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return
        url = URL(scope=scope)
        if url.path in self.path_mapping:
            new_url = url.replace(path=self.path_mapping[url.path])
            response = RedirectResponse(new_url, status_code=301)
            await response(scope, receive, send)
            return
        await self.app(scope, receive, send)

app = Starlette(routes=routes, middleware=[
    Middleware(RedirectsMiddleware, path_mapping={"/old": "/new"}),
])
```

### CORSMiddleware as outermost wrapper (for error responses)

```python
from starlette.middleware.cors import CORSMiddleware

inner_app = Starlette(routes=routes)
app = CORSMiddleware(app=inner_app, allow_origins=["*"])
```

---

## 8. Background Tasks

```python
from starlette.background import BackgroundTask, BackgroundTasks
from starlette.responses import JSONResponse

# Single task
async def send_email(to: str, subject: str):
    ...  # your email logic

async def signup(request):
    data = await request.json()
    task = BackgroundTask(send_email, to=data["email"], subject="Welcome!")
    return JSONResponse({"status": "ok"}, background=task)

# Multiple tasks (run in order)
async def notify_admin(username: str):
    ...

async def signup_multi(request):
    data = await request.json()
    tasks = BackgroundTasks()
    tasks.add_task(send_email, to=data["email"], subject="Welcome!")
    tasks.add_task(notify_admin, username=data["username"])
    return JSONResponse({"status": "ok"}, background=tasks)
```

> ⚠️ Tasks run in order. If one raises, subsequent tasks are skipped.

---

## 9. Static Files

```python
from starlette.staticfiles import StaticFiles
from starlette.routing import Mount

routes = [
    # Basic static directory
    Mount("/static", StaticFiles(directory="static"), name="static"),

    # HTML mode (auto-loads index.html, serves 404.html for missing files)
    Mount("/docs", StaticFiles(directory="docs", html=True), name="docs"),

    # From Python packages
    Mount("/vendor", StaticFiles(packages=["bootstrap4"]), name="vendor"),

    # Custom package directory
    Mount("/vendor2", StaticFiles(packages=[("mypackage", "assets")]), name="vendor2"),

    # Follow symlinks
    Mount("/linked", StaticFiles(directory="linked_dir", follow_symlinks=True)),
]
```

---

## 10. Templates (Jinja2)

> Requires: `pip install jinja2`

```python
from starlette.templating import Jinja2Templates

templates = Jinja2Templates(directory="templates")
# Autoescape is ON by default for .html/.htm/.xml (XSS protection)

async def homepage(request):
    return templates.TemplateResponse(request, "index.html", {"title": "Home"})
```

### Template — `index.html`

```html
<!DOCTYPE html>
<html>
<head>
  <title>{{ title }}</title>
  <link rel="stylesheet" href="{{ url_for('static', path='/css/app.css') }}">
</head>
<body>
  <h1>{{ title }}</h1>
</body>
</html>
```

### Custom Jinja2 Environment

```python
import jinja2
from starlette.templating import Jinja2Templates

env = jinja2.Environment(loader=jinja2.FileSystemLoader("templates"), autoescape=True)
env.globals["site_name"] = "My App"
templates = Jinja2Templates(env=env)
```

### Custom filters

```python
def currency_filter(value):
    return f"${value:,.2f}"

templates = Jinja2Templates(directory="templates")
templates.env.filters["currency"] = currency_filter
```

### Context processors

```python
from starlette.requests import Request

def app_context(request: Request):
    return {"app_name": "MyApp", "debug": request.app.debug}

templates = Jinja2Templates(
    directory="templates",
    context_processors=[app_context],
)
```

### Testing templates

```python
from starlette.testclient import TestClient

def test_homepage():
    client = TestClient(app)
    response = client.get("/")
    assert response.status_code == 200
    assert response.template.name == "index.html"
    assert "request" in response.context
```

---

## 11. Authentication

> Note: `AuthenticationMiddleware` is separate from `SessionMiddleware`.

```python
import base64, binascii
from starlette.authentication import (
    AuthCredentials, AuthenticationBackend, AuthenticationError,
    SimpleUser, requires,
)
from starlette.middleware.authentication import AuthenticationMiddleware

class BasicAuthBackend(AuthenticationBackend):
    async def authenticate(self, conn):
        if "Authorization" not in conn.headers:
            return None   # anonymous

        auth = conn.headers["Authorization"]
        try:
            scheme, credentials = auth.split()
            if scheme.lower() != "basic":
                return None
            decoded = base64.b64decode(credentials).decode("ascii")
        except (ValueError, UnicodeDecodeError, binascii.Error):
            raise AuthenticationError("Invalid basic auth credentials")

        username, _, password = decoded.partition(":")
        # TODO: verify password in a real app
        return AuthCredentials(["authenticated"]), SimpleUser(username)


async def homepage(request):
    if request.user.is_authenticated:
        return PlainTextResponse(f"Hello, {request.user.display_name}")
    return PlainTextResponse("Hello, anonymous")

@requires("authenticated")
async def dashboard(request):
    return PlainTextResponse("Dashboard")

@requires(["authenticated", "admin"], status_code=404)
async def admin(request):
    return PlainTextResponse("Admin panel")

@requires("authenticated", redirect="login")
async def profile(request):
    return PlainTextResponse("Profile")

# Class-based endpoint
class AdminView(HTTPEndpoint):
    @requires("authenticated")
    async def get(self, request):
        return PlainTextResponse("Admin GET")


middleware = [
    Middleware(AuthenticationMiddleware, backend=BasicAuthBackend()),
]
app = Starlette(routes=routes, middleware=middleware)
```

### Custom auth error response

```python
from starlette.responses import JSONResponse

def on_auth_error(request, exc):
    return JSONResponse({"error": str(exc)}, status_code=401)

middleware = [
    Middleware(AuthenticationMiddleware, backend=BasicAuthBackend(), on_error=on_auth_error),
]
```

---

## 12. Sessions

Sessions are signed cookies — readable but not forgeable.

```python
from starlette.middleware.sessions import SessionMiddleware

middleware = [
    Middleware(SessionMiddleware,
        secret_key="CHANGE-THIS-IN-PRODUCTION",
        session_cookie="session",
        max_age=14 * 24 * 3600,   # 2 weeks; None = browser session
        same_site="lax",
        https_only=True,           # set True in production
        domain=None,               # share across subdomains if needed
    )
]

async def login(request):
    request.session["user_id"] = 42
    return JSONResponse({"status": "logged in"})

async def logout(request):
    request.session.clear()
    return JSONResponse({"status": "logged out"})

async def me(request):
    user_id = request.session.get("user_id")
    return JSONResponse({"user_id": user_id})
```

> **1.0 Note**: `SessionMiddleware` now tracks whether the session was accessed or modified in the request, avoiding unnecessary cookie writes.

---

## 13. Exceptions

```python
from starlette.exceptions import HTTPException, WebSocketException
from starlette.requests import Request
from starlette.responses import JSONResponse, HTMLResponse
from starlette.websockets import WebSocket

async def http_exception_handler(request: Request, exc: HTTPException):
    return JSONResponse(
        {"detail": exc.detail},
        status_code=exc.status_code,
        headers=exc.headers or {},
    )

async def not_found_handler(request: Request, exc: HTTPException):
    return HTMLResponse("<h1>404 — Not found</h1>", status_code=404)

async def server_error_handler(request: Request, exc: Exception):
    return JSONResponse({"detail": "Internal server error"}, status_code=500)

async def ws_exception_handler(websocket: WebSocket, exc: WebSocketException):
    await websocket.close(code=1008)

app = Starlette(
    routes=routes,
    exception_handlers={
        HTTPException: http_exception_handler,
        404: not_found_handler,
        500: server_error_handler,
        WebSocketException: ws_exception_handler,
    }
)
```

### Raising exceptions in endpoints

```python
from starlette.exceptions import HTTPException

async def get_item(request):
    item_id = request.path_params["item_id"]
    if item_id not in db:
        raise HTTPException(status_code=404, detail="Item not found")
    return JSONResponse(db[item_id])
```

### HTTPException before WebSocket accept

```python
async def secure_ws(websocket: WebSocket):
    if not websocket.headers.get("x-token"):
        raise HTTPException(status_code=400, detail="Missing token")
    await websocket.accept()
    ...
```

---

## 14. Lifespan & State

### Basic lifespan

```python
import contextlib
from starlette.applications import Starlette

@contextlib.asynccontextmanager
async def lifespan(app):
    # Startup
    app.state.db = await create_db_pool()
    print("App started")
    yield
    # Shutdown
    await app.state.db.close()
    print("App stopped")

app = Starlette(routes=routes, lifespan=lifespan)
```

### Typed lifespan state (new in 0.52, recommended pattern)

```python
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager
from typing import TypedDict
import httpx
from starlette.applications import Starlette
from starlette.requests import Request

class AppState(TypedDict):
    http_client: httpx.AsyncClient

@asynccontextmanager
async def lifespan(app: Starlette) -> AsyncIterator[AppState]:
    async with httpx.AsyncClient() as client:
        yield {"http_client": client}

async def homepage(request: Request[AppState]):
    client = request.state["http_client"]       # typed: httpx.AsyncClient
    resp = await client.get("https://httpbin.org/get")
    return JSONResponse(resp.json())

app = Starlette(lifespan=lifespan, routes=[Route("/", homepage)])
```

### Testing with lifespan

```python
from starlette.testclient import TestClient

def test_homepage():
    with TestClient(app) as client:          # lifespan runs on context enter
        response = client.get("/")
        assert response.status_code == 200
    # lifespan teardown runs on context exit
```

---

## 15. Configuration

```python
# main.py
from starlette.config import Config
from starlette.datastructures import CommaSeparatedStrings, Secret

config = Config(".env")                              # reads .env + env vars

DEBUG          = config("DEBUG", cast=bool, default=False)
DATABASE_URL   = config("DATABASE_URL")
SECRET_KEY     = config("SECRET_KEY", cast=Secret)  # won't leak in repr
ALLOWED_HOSTS  = config("ALLOWED_HOSTS", cast=CommaSeparatedStrings)

# Namespaced env vars
config_prefixed = Config(env_prefix="APP_")
# APP_DEBUG -> config_prefixed("DEBUG")

# Custom encoding for .env file
config_latin = Config(".env", encoding="latin-1")
```

```ini
# .env  (do NOT commit to VCS)
DEBUG=True
DATABASE_URL=postgresql://user:pass@localhost/mydb
SECRET_KEY=super-secret-key-here
ALLOWED_HOSTS=127.0.0.1,localhost
```

```python
>>> str(config("SECRET_KEY", cast=Secret))
'super-secret-key-here'
>>> config("SECRET_KEY", cast=Secret)
Secret('**********')

>>> list(config("ALLOWED_HOSTS", cast=CommaSeparatedStrings))
['127.0.0.1', 'localhost']
```

### Override in tests

```python
# tests/conftest.py
from starlette.config import environ
environ["DEBUG"] = "TRUE"   # must be set before importing settings module
```

---

## 16. Schemas (OpenAPI)

> Requires: `pip install pyyaml`

```python
import sys
import yaml
import uvicorn
from starlette.applications import Starlette
from starlette.routing import Route
from starlette.schemas import SchemaGenerator

schemas = SchemaGenerator({
    "openapi": "3.0.0",
    "info": {"title": "My API", "version": "1.0"},
})

def list_users(request):
    """
    responses:
      200:
        description: A list of users.
        examples:
          [{"username": "tom"}, {"username": "lucy"}]
    """
    ...

def create_user(request):
    """
    responses:
      200:
        description: Created user.
        examples:
          {"username": "tom"}
    """
    ...

def openapi_schema(request):
    return schemas.OpenAPIResponse(request=request)

routes = [
    Route("/users",  list_users,   methods=["GET"]),
    Route("/users",  create_user,  methods=["POST"]),
    Route("/schema", openapi_schema, include_in_schema=False),
]

app = Starlette(routes=routes)

if __name__ == "__main__":
    if sys.argv[-1] == "schema":
        print(yaml.dump(schemas.get_schema(routes=app.routes), default_flow_style=False))
    else:
        uvicorn.run("main:app", host="0.0.0.0", port=8000)
```

---

## 17. Test Client

> Requires: `pip install httpx`

```python
from starlette.testclient import TestClient

# Basic usage
def test_homepage():
    client = TestClient(app)
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == {"hello": "world"}

# With lifespan
def test_with_lifespan():
    with TestClient(app) as client:
        response = client.get("/")
        assert response.status_code == 200

# Custom headers & auth
client = TestClient(app)
client.headers = {"Authorization": "Bearer token"}
response = client.get("/protected")

# File upload
def test_upload():
    client = TestClient(app)
    with open("example.txt", "rb") as f:
        response = client.post("/upload", files={"file": f})
    assert response.status_code == 200

# Suppress 500 exceptions (test error pages)
def test_error_page():
    client = TestClient(app, raise_server_exceptions=False)
    response = client.get("/broken")
    assert response.status_code == 500

# Custom client address
client = TestClient(app, client=("127.0.0.1", 9000))

# WebSocket testing
def test_ws():
    client = TestClient(app)
    with client.websocket_connect("/ws") as ws:
        ws.send_text("hello")
        data = ws.receive_text()
        assert data == "Echo: hello"
        ws.send_json({"event": "ping"})
        payload = ws.receive_json()

# Trio backend
def test_trio():
    with TestClient(app, backend="trio") as client:
        response = client.get("/")
        assert response.status_code == 200

# Async tests (without TestClient)
async def test_async():
    from httpx import AsyncClient, ASGITransport
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://testserver") as client:
        r = await client.get("/")
        assert r.status_code == 200
```

---

## 18. Server Push (HTTP/2)

```python
from starlette.responses import HTMLResponse

async def homepage(request):
    await request.send_push_promise("/static/style.css")
    await request.send_push_promise("/static/app.js")
    return HTMLResponse("""
        <html>
          <head>
            <link rel="stylesheet" href="/static/style.css">
            <script src="/static/app.js"></script>
          </head>
          <body><h1>Fast page</h1></body>
        </html>
    """)
```

> `send_push_promise` is a no-op if the ASGI server doesn't support HTTP/2 server push.

---

## 19. Thread Pool

Starlette automatically runs sync endpoints, sync background tasks, and file I/O in the thread pool using `anyio.to_thread.run_sync`.

```python
# Sync endpoint — automatically offloaded to thread pool
def sync_endpoint(request):
    result = heavy_cpu_work()             # doesn't block event loop
    return JSONResponse({"result": result})

# Increase thread pool size if needed (default: 40)
import anyio.to_thread

limiter = anyio.to_thread.current_default_thread_limiter()
limiter.total_tokens = 100
```

---

## 20. Pure ASGI Usage

Starlette components work standalone without the `Starlette` app class.

```python
from starlette.responses import PlainTextResponse

# Minimal ASGI app
async def app(scope, receive, send):
    assert scope["type"] == "http"
    response = PlainTextResponse("Hello, world!")
    await response(scope, receive, send)
```

```python
# Using Request directly in pure ASGI
from starlette.requests import Request

async def app(scope, receive, send):
    assert scope["type"] == "http"
    request = Request(scope, receive)
    body = await request.body()
    response = PlainTextResponse(f"Got {len(body)} bytes")
    await response(scope, receive, send)
```

```python
# Standalone Router (no middleware overhead)
from starlette.routing import Router, Route

app = Router(routes=[
    Route("/", homepage),
    Route("/about", about),
])
```

---

## 21. What Changed in 1.0 (Breaking Removals)

The 1.0 release cleaned up all previously-deprecated APIs. If upgrading from pre-1.0:

| Removed | Replacement |
|--------|------------|
| `app.on_startup` / `app.on_shutdown` params | `lifespan=` parameter |
| `@app.on_event("startup")` decorator | `lifespan=` parameter |
| `app.add_event_handler()` | `lifespan=` parameter |
| `router.startup()` / `router.shutdown()` | `lifespan=` parameter |
| `@app.route("/path")` decorator | `Route(...)` in `routes=` list |
| `@app.websocket_route("/ws")` decorator | `WebSocketRoute(...)` in `routes=` list |
| `@app.exception_handler(...)` decorator | `exception_handlers=` dict parameter |
| `@app.middleware("http")` decorator | `Middleware(...)` in `middleware=` list |
| `iscoroutinefunction_or_partial()` | use `asyncio.iscoroutinefunction()` |
| `Jinja2Templates(**env_options)` kwargs | `Jinja2Templates(env=jinja2.Environment(...))` |
| `TemplateResponse(name, context)` signature | `TemplateResponse(request, name, context)` |
| `FileResponse(method=...)` parameter | removed entirely |
| `WS_1004_NO_STATUS_RCVD`, `WS_1005_ABNORMAL_CLOSURE` | removed from `starlette.status` |
| `ExceptionMiddleware` import from `starlette.exceptions` | import from `starlette.middleware.exceptions` |

### Correct 1.0 lifespan pattern

```python
# ✅ Correct for 1.0
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app):
    # startup logic
    yield
    # shutdown logic

app = Starlette(routes=routes, lifespan=lifespan)
```

```python
# ❌ Removed in 1.0
app = Starlette(
    routes=routes,
    on_startup=[startup_handler],    # REMOVED
    on_shutdown=[shutdown_handler],  # REMOVED
)
```

### Correct 1.0 template pattern

```python
# ✅ Correct for 1.0
templates.TemplateResponse(request, "index.html", {"key": "value"})

# ❌ Removed in 1.0
templates.TemplateResponse("index.html", {"request": request})  # old signature
```

### Jinja2 autoescape is now ON by default

```python
# 1.0: autoescape enabled by default for .html/.htm/.xml
templates = Jinja2Templates(directory="templates")

# To disable (not recommended):
import jinja2
env = jinja2.Environment(loader=jinja2.FileSystemLoader("templates"), autoescape=False)
templates = Jinja2Templates(env=env)
```

---

## Quick Reference: Dependencies

| Feature | Extra install |
|---------|--------------|
| TestClient | `pip install httpx` |
| Templates | `pip install jinja2` |
| Form parsing / file upload | `pip install python-multipart` |
| Sessions | `pip install itsdangerous` |
| OpenAPI schema | `pip install pyyaml` |
| All at once | `pip install starlette[full]` |
