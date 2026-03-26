---
name: starlette-1
description: >
  Reference and construction guide for building async Python web apps with
  Starlette 1.0 (released March 2026). Use this skill whenever the user is
  building a new Starlette app, adding a feature to an existing one, or asking
  how something works in Starlette — routing, middleware, requests, responses,
  WebSockets, authentication, lifespan, background tasks, static files,
  templates, sessions, exceptions, test client, OpenAPI schemas, or
  configuration. Also trigger when the user is migrating code from Starlette
  pre-1.0 to 1.0, or when they hit a deprecation warning from the old
  on_startup/on_shutdown/decorator-based APIs. Trigger even if the user just
  says "Starlette" without asking explicitly for a skill — this is the go-to
  reference for anything Starlette 1.0.
---

# Starlette 1.0 — Build Guide

Starlette is a lightweight ASGI framework for async Python web services.
Version 1.0 (March 2026) is the first stable release.

**Install:** `pip install starlette uvicorn`
**Full install (all optional deps):** `pip install starlette[full] uvicorn`
**Python:** 3.10+

## How to use this skill

When the user asks how to build or implement something, always:
1. Give a short explanation of how the feature works conceptually
2. Follow with a minimal, runnable code example
3. Call out gotchas or 1.0-specific changes if relevant

If they ask about a migration from pre-1.0, go directly to the
[Breaking Changes](#breaking-changes) section.

---

## Application bootstrap

The `Starlette` class is the entry point. Pass routes, middleware,
exception handlers, and a lifespan context manager.

```python
from contextlib import asynccontextmanager
from starlette.applications import Starlette
from starlette.routing import Route, Mount, WebSocketRoute
from starlette.responses import JSONResponse
from starlette.middleware import Middleware
from starlette.middleware.cors import CORSMiddleware
from starlette.staticfiles import StaticFiles

@asynccontextmanager
async def lifespan(app):
    # startup: initialise resources here
    app.state.db = await connect_db()
    yield
    # shutdown: clean up here
    await app.state.db.close()

app = Starlette(
    debug=True,
    routes=[
        Route("/", homepage),
        Route("/users/{user_id:int}", user),
        WebSocketRoute("/ws", ws_handler),
        Mount("/static", StaticFiles(directory="static"), name="static"),
    ],
    middleware=[
        Middleware(CORSMiddleware, allow_origins=["*"]),
    ],
    lifespan=lifespan,
)
```

Run with: `uvicorn main:app --reload`

---

## Routing

```python
from starlette.routing import Route, Mount, Router

# HTTP methods — function endpoints accept GET only by default
Route("/items", list_items, methods=["GET", "POST"])

# Path convertors: str (default), int, float, uuid, path
Route("/users/{user_id:int}", get_user)
Route("/files/{rest:path}", get_file)       # captures slashes

# Submounting for modular apps
routes = [
    Route("/", homepage),
    Mount("/api", routes=[
        Route("/users", list_users),
        Route("/users/{id:int}", get_user),
    ]),
]

# Reverse URL lookup (named routes)
Route("/users/{username}", profile, name="user_profile")
url = request.url_for("user_profile", username="tom")   # → URL object
url = app.url_path_for("user_profile", username="tom")  # → path only

# Per-route middleware
from starlette.middleware.gzip import GZipMiddleware
Route("/export", export_handler, middleware=[Middleware(GZipMiddleware)])
```

### Custom URL convertor

```python
from starlette.convertors import Convertor, register_url_convertor
from datetime import date

class DateConvertor(Convertor):
    regex = r"[0-9]{4}-[0-9]{2}-[0-9]{2}"
    def convert(self, value: str) -> date:
        return date.fromisoformat(value)
    def to_string(self, value: date) -> str:
        return value.isoformat()

register_url_convertor("date", DateConvertor())
# Usage: Route("/reports/{day:date}", report)
```

---

## Requests

```python
from starlette.requests import Request

async def handler(request: Request):
    request.method               # "GET", "POST", …
    request.url.path             # "/users/42"
    request.url.scheme           # "https"
    request.path_params["id"]    # from route pattern
    request.query_params["q"]    # from ?q=…
    request.headers["content-type"]
    request.client.host          # remote IP
    request.cookies.get("session")

    body  = await request.body()        # raw bytes
    data  = await request.json()        # parsed JSON
    # streaming (no buffering — use when body is large)
    async for chunk in request.stream():
        ...

    # file upload
    async with request.form() as form:
        file = form["upload"]           # UploadFile
        content = await file.read()
        name = file.filename
        size = file.size                # bytes
```

---

## Responses

```python
from starlette.responses import (
    Response, HTMLResponse, PlainTextResponse,
    JSONResponse, RedirectResponse, StreamingResponse, FileResponse,
)

JSONResponse({"key": "val"})                        # 200 by default
JSONResponse({"err": "not found"}, status_code=404)
RedirectResponse("/new", status_code=301)           # 307 default
FileResponse("report.pdf", filename="report.pdf")   # Range requests built-in
HTMLResponse("<h1>hello</h1>")

# Streaming
async def gen():
    for i in range(5):
        yield f"chunk {i}\n"
        await asyncio.sleep(0.1)
StreamingResponse(gen(), media_type="text/plain")

# Cookies
response = JSONResponse({"ok": True})
response.set_cookie("session", "abc", httponly=True, secure=True, samesite="lax")
response.delete_cookie("session")

# Custom JSON serializer (e.g. orjson for speed)
import orjson
class FastJSON(JSONResponse):
    def render(self, content) -> bytes:
        return orjson.dumps(content)
```

---

## WebSockets

```python
from starlette.websockets import WebSocket

async def ws_handler(websocket: WebSocket):
    await websocket.accept()

    await websocket.send_text("hello")
    msg = await websocket.receive_text()

    await websocket.send_json({"event": "update"})
    data = await websocket.receive_json()

    # Iterator — exits cleanly on disconnect
    async for message in websocket.iter_text():
        await websocket.send_text(f"echo: {message}")

    await websocket.close(code=1000)

# Deny connection before accept (requires ASGI server support)
from starlette.exceptions import HTTPException
async def secure_ws(websocket: WebSocket):
    if not websocket.headers.get("authorization"):
        raise HTTPException(status_code=401)
    await websocket.accept()
    ...
```

---

## Class-based endpoints

```python
from starlette.endpoints import HTTPEndpoint, WebSocketEndpoint
from starlette.responses import JSONResponse

class UserEndpoint(HTTPEndpoint):
    async def get(self, request):
        return JSONResponse({"id": request.path_params["id"]})
    async def post(self, request):
        data = await request.json()
        return JSONResponse(data, status_code=201)
    # Unhandled methods → 405 automatically

class EchoEndpoint(WebSocketEndpoint):
    encoding = "text"   # "text" | "bytes" | "json"
    async def on_connect(self, ws): await ws.accept()
    async def on_receive(self, ws, data): await ws.send_text(f"echo: {data}")
    async def on_disconnect(self, ws, code): pass
```

---

## Middleware

Built-in middleware — pass via `Middleware(...)` list in `Starlette(...)`:

```python
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
        allow_methods=["*"],
        allow_headers=["*"],
        allow_credentials=True,
        allow_private_network=True,
    ),
    Middleware(SessionMiddleware, secret_key="change-me", https_only=True),
    Middleware(GZipMiddleware, minimum_size=1000),
]
```

Custom middleware — use pure ASGI (avoids `contextvars` limitation of `BaseHTTPMiddleware`):

```python
from starlette.types import ASGIApp, Scope, Receive, Send
from starlette.datastructures import MutableHeaders

class RequestIDMiddleware:
    def __init__(self, app: ASGIApp):
        self.app = app

    async def __call__(self, scope: Scope, receive: Receive, send: Send):
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return

        import uuid
        scope["request_id"] = str(uuid.uuid4())

        async def send_with_header(message):
            if message["type"] == "http.response.start":
                headers = MutableHeaders(scope=message)
                headers.append("X-Request-ID", scope["request_id"])
            await send(message)

        await self.app(scope, receive, send_with_header)
```

---

## Lifespan & typed state

Lifespan replaces the old `on_startup`/`on_shutdown` events (removed in 1.0).
Yield a `TypedDict` to get typed access to state in endpoints.

```python
from contextlib import asynccontextmanager
from typing import TypedDict
import httpx
from starlette.applications import Starlette
from starlette.requests import Request

class AppState(TypedDict):
    http_client: httpx.AsyncClient

@asynccontextmanager
async def lifespan(app: Starlette):
    async with httpx.AsyncClient() as client:
        yield {"http_client": client}       # ← dict becomes request.state

async def call_api(request: Request[AppState]):
    client = request.state["http_client"]   # typed: httpx.AsyncClient
    resp = await client.get("https://api.example.com")
    return JSONResponse(resp.json())
```

---

## Background tasks

```python
from starlette.background import BackgroundTask, BackgroundTasks

# Single task — runs after response is sent
async def send_email(to: str): ...

async def signup(request):
    data = await request.json()
    task = BackgroundTask(send_email, to=data["email"])
    return JSONResponse({"ok": True}, background=task)

# Multiple tasks — run in order; if one fails, the rest are skipped
async def notify_admin(username: str): ...

async def signup_multi(request):
    data = await request.json()
    tasks = BackgroundTasks()
    tasks.add_task(send_email, to=data["email"])
    tasks.add_task(notify_admin, username=data["username"])
    return JSONResponse({"ok": True}, background=tasks)
```

---

## Authentication

```python
import base64
from starlette.authentication import (
    AuthCredentials, AuthenticationBackend, AuthenticationError,
    SimpleUser, requires,
)
from starlette.middleware.authentication import AuthenticationMiddleware

class BasicAuthBackend(AuthenticationBackend):
    async def authenticate(self, conn):
        if "Authorization" not in conn.headers:
            return None   # anonymous — not an error
        try:
            scheme, credentials = conn.headers["Authorization"].split()
            decoded = base64.b64decode(credentials).decode("ascii")
        except Exception:
            raise AuthenticationError("Invalid credentials")
        username, _, password = decoded.partition(":")
        # validate password here…
        return AuthCredentials(["authenticated"]), SimpleUser(username)

# Protecting endpoints
@requires("authenticated")
async def dashboard(request): ...

@requires(["authenticated", "admin"], redirect="login")
async def admin(request): ...

middleware = [Middleware(AuthenticationMiddleware, backend=BasicAuthBackend())]
```

---

## Sessions

```python
from starlette.middleware.sessions import SessionMiddleware

# Add to middleware list
Middleware(SessionMiddleware, secret_key="...", https_only=True)

# Use in any endpoint via request.session (dict interface)
async def login(request):
    request.session["user_id"] = 42
    return JSONResponse({"ok": True})

async def logout(request):
    request.session.clear()
    return JSONResponse({"ok": True})
```

---

## Exception handling

```python
from starlette.exceptions import HTTPException, WebSocketException
from starlette.requests import Request
from starlette.responses import JSONResponse

async def http_error(request: Request, exc: HTTPException):
    return JSONResponse({"detail": exc.detail}, status_code=exc.status_code)

async def generic_error(request: Request, exc: Exception):
    return JSONResponse({"detail": "server error"}, status_code=500)

app = Starlette(
    routes=routes,
    exception_handlers={
        HTTPException: http_error,
        500: generic_error,
    }
)

# Raise from any endpoint — auto-handled
raise HTTPException(status_code=404, detail="Not found")
```

---

## Static files & templates

```python
# Static files
from starlette.staticfiles import StaticFiles
Mount("/static", StaticFiles(directory="static"), name="static")
Mount("/docs", StaticFiles(directory="docs", html=True))   # serves index.html

# Templates (requires: pip install jinja2)
# Autoescape is ON by default for .html/.htm/.xml in 1.0
from starlette.templating import Jinja2Templates
templates = Jinja2Templates(directory="templates")

async def homepage(request):
    return templates.TemplateResponse(request, "index.html", {"title": "Home"})

# In template: {{ url_for('static', path='/css/app.css') }}

# Context processors (injected into every template)
def globals_ctx(request): return {"app_name": "MyApp"}
templates = Jinja2Templates(directory="templates", context_processors=[globals_ctx])
```

---

## Configuration

```python
from starlette.config import Config
from starlette.datastructures import CommaSeparatedStrings, Secret

config = Config(".env")
DEBUG        = config("DEBUG", cast=bool, default=False)
DATABASE_URL = config("DATABASE_URL")
SECRET_KEY   = config("SECRET_KEY", cast=Secret)    # masked in repr
HOSTS        = config("ALLOWED_HOSTS", cast=CommaSeparatedStrings)

# Namespace env vars: APP_DEBUG → config("DEBUG")
config = Config(env_prefix="APP_")
```

---

## OpenAPI schema

Requires `pip install pyyaml`. Use docstrings on endpoints + `SchemaGenerator`.

```python
from starlette.schemas import SchemaGenerator
schemas = SchemaGenerator({"openapi": "3.0.0", "info": {"title": "API", "version": "1.0"}})

def list_users(request):
    """responses:\n  200:\n    description: List of users."""
    ...

routes = [
    Route("/users", list_users, methods=["GET"]),
    Route("/schema", lambda r: schemas.OpenAPIResponse(request=r), include_in_schema=False),
]
```

---

## Test client

```python
from starlette.testclient import TestClient

def test_api():
    with TestClient(app) as client:    # 'with' triggers lifespan
        r = client.get("/users/1")
        assert r.status_code == 200
        assert r.json()["id"] == 1

# WebSocket testing
def test_ws():
    with TestClient(app) as client:
        with client.websocket_connect("/ws") as ws:
            ws.send_text("hello")
            assert ws.receive_text() == "echo: hello"

# Test error responses without raising
client = TestClient(app, raise_server_exceptions=False)
```

---

## Breaking changes from pre-1.0

If the user is migrating, these APIs were **removed** in 1.0:

| Removed | Use instead |
|---------|-------------|
| `on_startup=` / `on_shutdown=` params | `lifespan=` async context manager |
| `@app.on_event("startup")` | `lifespan=` |
| `app.add_event_handler()` | `lifespan=` |
| `@app.route("/path")` | `Route(...)` in `routes=` list |
| `@app.websocket_route("/ws")` | `WebSocketRoute(...)` in `routes=` list |
| `@app.exception_handler(...)` | `exception_handlers=` dict |
| `@app.middleware("http")` | `Middleware(...)` in `middleware=` list |
| `TemplateResponse(name, context)` | `TemplateResponse(request, name, context)` |
| `Jinja2Templates(**env_options)` | `Jinja2Templates(env=jinja2.Environment(...))` |
| `FileResponse(method=...)` | removed entirely |

**Minimum migration for lifespan:**
```python
# Before (pre-1.0) ❌
@app.on_event("startup")
async def startup(): app.state.db = await connect()

# After (1.0) ✅
@asynccontextmanager
async def lifespan(app):
    app.state.db = await connect()
    yield
    await app.state.db.close()

app = Starlette(routes=routes, lifespan=lifespan)
```
