# FastAPI — Tutorial + Interview Q&A

Target: build and defend production FastAPI services. Work through §1–§7 as a tutorial, then drills.

---

## 1. What & why

FastAPI = **Starlette** (async web toolkit) + **Pydantic** (validation/serialization), Python 3.8+, with **automatic OpenAPI** and interactive docs at `/docs` and `/redoc`. Modern default for Python APIs: speed (async), DX (typing → validation + docs for free), and ecosystem fit for LLM apps.

```bash
python -m venv .venv && source .venv/bin/activate
pip install "fastapi[standard]" uvicorn[standard]
uvicorn app.main:app --reload          # dev server
```

```python
# app/main.py
from fastapi import FastAPI
app = FastAPI(title="Demo API")

@app.get("/health")
def health():
    return {"status": "ok"}
```

## 2. Request/response basics

```python
from fastapi import FastAPI, HTTPException, status, Path, Query, Body
from pydantic import BaseModel

class ItemIn(BaseModel):
    name: str
    price: float = Query(gt=0)
    tags: list[str] = []

class ItemOut(BaseModel):
    id: int
    name: str
    price: float

@app.post("/items", response_model=ItemOut, status_code=status.HTTP_201_CREATED)
def create_item(payload: ItemIn):        # body validated by Pydantic
    ...
    if dup:
        raise HTTPException(409, "item exists")
    ...

@app.get("/items/{item_id}")
def get_item(item_id: int = Path(gt=0), verbose: bool = False):   # path + query
    ...
```

- Path > query > header > cookie > body. Types drive coercion + validation errors (422 by default).
- `response_model` filters/serializes output (hides secrets, enforces shape). `response_model=None` only when you must (document why).
- Async or sync endpoints: `def` runs in a threadpool (fine for sync DB drivers); `async def` runs on the event loop — **never block it** (no sync HTTP/DB inside without `run_in_executor`).

## 3. Pydantic v2 essentials

```python
from pydantic import BaseModel, Field, field_validator, EmailStr
from datetime import datetime

class UserIn(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8)
    created: datetime = Field(default_factory=datetime.utcnow)

    @field_validator("password")
    @classmethod
    def strong(cls, v: str):
        if v.isdigit(): raise ValueError("needs letters")
        return v

user = UserIn(email="a@b.com", password="secret12")
user.model_dump()            # dict
user.model_dump_json()       # str
UserIn.model_validate({"email": "a@b.com", "password": "secret12"})
UserIn.model_json_schema()   # powers OpenAPI
```

Know: defaults vs `default_factory`, `Optional`, nested models, `model_config = ConfigDict(extra="forbid")`, computed fields, and that validation errors return 422 with a detailed body.

## 4. Dependency injection (the superpower)

```python
from fastapi import Depends, Header, HTTPException
from sqlalchemy.orm import Session

def get_db():
    db = SessionLocal()
    try: yield db              # yield = setup + teardown
    finally: db.close()

def current_user(token: str = Depends(oauth2_scheme), db: Session = Depends(get_db)) -> User:
    user = decode_and_fetch(token, db)
    if not user: raise HTTPException(401)
    return user

@app.get("/me")
def me(user: User = Depends(current_user)):
    return user
```

- Sub-dependencies nest; FastAPI caches **per-request** (same `Depends(get_db)` = one DB session per request).
- `app.dependency_overrides[get_db] = fake_db` in tests — killer feature.
- Use for: DB sessions, auth, feature flags, request IDs, tenant resolution, pagination params.

## 5. Security (JWT pattern)

```python
from passlib.context import CryptContext
pwd = CryptContext(schemes=["bcrypt"], deprecated="auto")
# hash on register: pwd.hash(password)   verify on login: pwd.verify(plain, hashed)

from datetime import datetime, timedelta, timezone
import jwt  # pyjwt
def create_token(sub: str) -> str:
    payload = {"sub": sub, "exp": datetime.now(timezone.utc) + timedelta(minutes=30)}
    return jwt.encode(payload, settings.SECRET, algorithm="HS256")
```

- Flow: `POST /token` (OAuth2PasswordRequestForm) → client sends `Authorization: Bearer …` → dependency verifies → returns user.
- Access token short-lived; refresh token rotation for longer sessions. Secrets from env/Secret Manager. HTTPS-only cookies (`HttpOnly`, `Secure`, `SameSite`) if you use cookies.
- Passwords: bcrypt/argon2 only; rate-limit login; generic error message ("invalid credentials") to avoid user enumeration.

## 6. Middleware, errors, CORS

```python
from fastapi.middleware.cors import CORSMiddleware
app.add_middleware(CORSMiddleware, allow_origins=settings.CORS, allow_credentials=True)

@app.middleware("http")
async def add_request_id(request, call_next):
    rid = request.headers.get("X-Request-ID", uuid4().hex)
    with structlog.contextvars.bound_contextvars(request_id=rid):
        response = await call_next(request)
        response.headers["X-Request-ID"] = rid
        return response

@app.exception_handler(NotFoundError)
async def not_found(request, exc):
    return JSONResponse(status_code=404, content={"detail": str(exc)})
```

Middleware order: last `add_middleware` added = outermost. Exception handlers turn domain exceptions into clean HTTP errors; log with stack on 5xx, never leak internals.

## 7. Background tasks, streaming, WebSockets

```python
from fastapi import BackgroundTasks
@app.post("/orders")
def place_order(order: OrderIn, background: BackgroundTasks, db=Depends(get_db)):
    db.add(order)
    background.add_task(send_email, order.customer_email)   # runs AFTER response
    return {"ok": True}

@app.get("/stream")
async def stream():                      # Server-Sent Events style
    async def gen():
        for chunk in llm.stream("tell me a joke"):
            yield f"data: {chunk}\n\n"
    return StreamingResponse(gen(), media_type="text/event-stream")
```

- BackgroundTasks = in-process, best-effort, **not** a queue. Anything important/retryable → Celery/RQ/Cloud Tasks/Pub-Sub.
- WebSockets: `@app.websocket("/ws")` + `await ws.accept()/receive_json()/send_json()`; manage connection state carefully (scale-out needs pub/sub fan-out).

## 8. Databases (SQLAlchemy 2.0 async + Alembic)

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
engine = create_async_engine(settings.DB_URL, pool_pre_ping=True)
Session = async_sessionmaker(engine, expire_on_commit=False)

class Task(Base):
    __tablename__ = "tasks"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str]
    done: Mapped[bool] = default=False

@app.get("/tasks")
async def list_tasks(db: AsyncSession = Depends(get_db)):
    result = await db.scalars(select(Task).where(Task.done == False))
    return result.all()
```

- Use `asyncpg` (Postgres), `aiosqlite` (SQLite), `aiomysql`. Sync SQLAlchemy works too via threadpool endpoints — but don't mix blocking driver into `async def`.
- **Alembic** for migrations (autogenerate from models). Indexes for query columns. Connection pooling: size to DB limits, `pool_pre_ping` for stale connections.
- On GCP: Cloud SQL Auth proxy (or private IP) + connection pooler if many instances.

## 9. Testing

```python
from fastapi.testclient import TestClient
client = TestClient(app)          # sync; or httpx.AsyncClient(app=app) for async

def test_create(client, monkeypatch):
    monkeypatch.setenv("TOKEN", "test")
    r = client.post("/items", json={"name": "x", "price": 2}, headers=auth())
    assert r.status_code == 201
    assert r.json()["name"] == "x"

# dependency override instead of a real DB
app.dependency_overrides[get_db] = lambda: fake_session()
```

Pytest + fixtures + `dependency_overrides` + factory-boy for data. Test the error paths (401/404/422), not just the happy path. `TestClient` raises the real exception by default — good for debugging.

## 10. Deployment (Docker + Cloud Run)

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY pyproject.toml poetry.lock ./
RUN pip install --no-cache-dir .
COPY app ./app
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

- Cloud Run: container must listen on `$PORT` (8080), scale-to-zero, autoscale on concurrency (default 80/request concurrency is a good start).
- Config via env (`pydantic-settings`), secrets via Secret Manager; structured JSON logs → Cloud Logging (forward to Kibana if the org is Elastic-centric); metrics/traces → Cloud Monitoring/Grafana.
- Health checks: `/health` (liveness) + `/ready` (dependencies OK). Graceful shutdown: finish in-flight requests before exit.
- Gunicorn with several uvicorn workers only if you're not purely event-loop bound; Cloud Run likes 1 process + async, tune concurrency first.

## 11. Performance rules

1. Async all the way for I/O paths; no blocking calls in `async def` (they stall the whole loop).
2. Batch DB access (avoid N+1), select only needed columns.
3. Cache hot reads (Redis) with explicit invalidation strategy.
4. `response_model` keep payloads small; paginate everything.
5. Rate limiting (slowapi / API gateway) per client.
6. Load test with `locust`/`k6` before you promise numbers.

## 12. Mini-projects to practice

1. **URL shortener** — models, POST/GET, 404 handling, rate limit, tests, Docker.
2. **Auth service** — register/login JWT, `current_user` dependency, dependency overrides in tests.
3. **LLM chat proxy** — FastAPI + LangChain, SSE streaming, API-key auth, token accounting (see stack.md §2).

---

## Interview Q&A — FastAPI

**Q: FastAPI vs Flask vs Django?** Flask: micro, manual wiring (blueprints, validation by hand). Django: full framework (ORM, admin, auth) — great for monoliths, heavier for APIs. FastAPI: typed request/response, DI, auto OpenAPI, async-native — best default for new Python APIs, especially AI/LLM services.

**Q: How does FastAPI use your type hints?** FastAPI inspects annotations at import time → generates Pydantic models, validation, OpenAPI schema, and docs. That's why `def create(item: ItemIn)` validates the body automatically.

**Q: `def` vs `async def` endpoints?** `def` → run in threadpool (safe with sync drivers). `async def` → event loop; must use async libs or you block everyone. Mixing: one slow sync call inside `async def` can stall the entire worker.

**Q: What is dependency injection used for in real apps?** DB sessions, auth/user resolution, tenant lookup, feature flags, pagination, clock/uuid injection for tests. Composable, cached per request, overridable in tests.

**Q: What does `response_model` actually do?** Filters the returned object through the model: removes extra fields, coerces types, documents the response in OpenAPI. It also validates your own code's output in dev.

**Q: 422 vs 400?** 422 = Pydantic validation failure (structured error list). 400/409/404 = your business errors via `HTTPException`. Return field-level messages when possible.

**Q: BackgroundTasks vs Celery?** BackgroundTasks runs in-process after the response — fine for emails/logs; lost on crash, no retries, ties up the worker. Use a real queue (Celery/RQ/GCP Cloud Tasks + Pub/Sub) for anything critical, long, or retryable.

**Q: How do you version an API?** Path (`/v1/items`) simplest and clearest; headers/`Accept` for media-type versioning. Deprecate with sunsets; keep old versions until clients migrate.

**Q: How do you handle auth on WebSockets?** Token via query param/first message or cookie at upgrade time; validate in the connect handler, then manage heartbeats and reconnection with backoff on the client.

**Q: How would you deploy on GCP?** Containerize → Cloud Run (serverless containers, autoscaling, revisions/rollback). Cloud SQL for Postgres, Secret Manager for secrets, Pub/Sub+Cloud Tasks for async work, Cloud Storage for files, Cloud Monitoring/Grafana for dashboards, Cloud Run Jobs for batch.

**Q: How do you keep OpenAPI useful?** Good model names, `response_model` per status, `summary`/`description`/examples on routes, tags for grouping, error responses documented — it becomes the contract for clients and generated SDKs.

**Q: Testing strategy?** Unit (pure logic) → API tests with TestClient + dependency overrides → one smoke test against a real Postgres in CI → contract checks on OpenAPI for breaking changes. Mock LLM calls; never call paid APIs in CI.

**Q: Structured logging — how?** `structlog` or stdlib with JSON formatter; bind `request_id`, `user_id`, latency per request via middleware context; ship to Cloud Logging/Kibana; dashboards in Grafana.

**Q: What breaks first in production, and how do you find it?** Downstream DB/API latency, then event-loop blocking, then memory from caches. Triage: Grafana (RED metrics) → traces → Kibana logs by `request_id`.

**Q: Pydantic v1 vs v2?** v2 is Rust-core, much faster, new API (`model_dump`, `field_validator`, `model_config`). If you see `orm_mode`/`.dict()` in code, it's v1 — know the migration names.

**Q: How do you do pagination?** Offset for admin UIs; cursor/keyset (`WHERE id > last ORDER BY id LIMIT n`) for large/infinite feeds — stable and fast with the right index.

**Q: Rate limiting approach?** Token bucket per API key/IP; enforce at gateway (API Gateway/Cloud Armor) for coarse limits, `slowapi`/Redis for app-level rules; return 429 + `Retry-After`.

**Q: When NOT to use FastAPI?** Django-shop-style apps wanting admin/ORM/templating out of the box; teams already standardized on Flask/Django; extreme CPU-bound workloads where language choice matters more than framework.
