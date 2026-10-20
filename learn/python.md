# Python — Tutorial + Interview Q&A

Target: pass Python-focused backend interviews at senior level. Run every snippet.

---

## 1. Language fundamentals

```python
a = [1, 2, 3]
b = a
b.append(4)          # a is also [1,2,3,4] — same object
c = a[:]             # shallow copy
d = a + []           # new list

s = "hello"
s[1:4]               # "ell" — slicing works on any sequence
s[::2]               # "hlo"

x, y = 1, 2
x, y = y, x          # swap without a temp variable

n = 5
print(f"{n=} ")      # debug: prints "n=5"
```

- **Mutable** (list, dict, set, bytearray) vs **immutable** (int, str, tuple, frozenset). Default-arg gotcha comes from mutability:

```python
def add(item, bucket=[]):   # WRONG — the list is created ONCE at def time
    bucket.append(item)
    return bucket
add(1)  # [1]
add(2)  # [1, 2]  ← surprise

def add(item, bucket=None): # RIGHT
    if bucket is None:
        bucket = []
    ...
```

## 2. Functions, closures, decorators

```python
def f(*args, **kwargs): ...   # args: tuple, kwargs: dict

def outer(n):
    def inner():
        return n * 2          # closure captures n
    return inner

import functools
def timer(fn):
    @functools.wraps(fn)      # keeps fn.__name__, __doc__
    def wrapper(*a, **kw):
        import time; t = time.perf_counter()
        r = fn(*a, **kw)
        print(time.perf_counter() - t)
        return r
    return wrapper

@timer
def work(): ...
```

Real-world decorators you should be able to write: retry with backoff, cache (`@functools.lru_cache`), timing, permission check, `@app.get()`-style registration (see FastAPI).

## 3. OOP

```python
from dataclasses import dataclass

@dataclass
class User:
    id: int
    name: str
    active: bool = True

class Repo:
    def __init__(self, db): self._db = db
    def __repr__(self): return f"Repo({self._db!r})"
    def __len__(self): return self._db.count()
    def __eq__(self, other): return isinstance(other, Repo) and self._db == other._db
```

- **MRO** (Method Resolution Order): `C.mro()` / `type(C).__mro__` — C3 linearization. Know why `super()` matters in multiple inheritance.
- **Properties**: `@property` for computed attributes with validation on setter.
- **`__slots__`**: fixes instance dict → less memory, blocks arbitrary attributes. Use for millions of small objects.
- **Class vs instance variables**, and the mutable class-variable trap.

## 4. Iterators, generators, comprehensions

```python
def countdown(n):
    while n > 0:
        yield n          # generator: lazy, one value at a time
        n -= 1

sum(x * x for x in range(10**7))     # generator expr — O(1) extra memory
[x * x for x in range(10)]           # list comp — materializes

# generator pipeline = cheap ETL style processing
lines   = (l for l in open("big.log"))
errors  = (json.loads(l) for l in lines if '"error"' in l)
```

Interview gold: generators vs list = memory (lazy vs eager), infinite sequences, `yield from`, `send()` basics.

## 5. Exceptions & context managers

```python
class NotFoundError(Exception): ...

try:
    get_user(1)
except NotFoundError as e:
    ...
else:        # runs only if no exception
    ...
finally:     # always runs (cleanup)
    ...

with open("f.txt") as f:   # __enter__/__exit__
    data = f.read()

from contextlib import contextmanager
@contextmanager
def db_session():
    s = Session(); 
    try: yield s; s.commit()
    except: s.rollback(); raise
    finally: s.close()
```

- Catch specific exceptions; never bare `except:`. `except Exception` + log + re-raise/convert is the professional pattern.
- Custom exception hierarchy per domain (`PaymentError` → `DeclinedError`, `InsufficientFundsError`).

## 6. Modules, packaging, environment

- `if __name__ == "__main__":` — guard for scripts.
- Virtualenv: `python -m venv .venv && source .venv/bin/activate`; deps in `pyproject.toml` (poetry/uv/pip-tools).
- `pathlib.Path` over `os.path`. `os.environ` or `pydantic-settings` for config; `secrets` module for tokens (never `random`).
- Relative imports inside packages; avoid import cycles (move shared code down).

## 7. Memory & garbage collection

- **Reference counting** frees objects immediately; the **cyclic GC** (`gc` module) collects reference cycles periodically.
- `sys.getsizeof`, `tracemalloc` to find leaks; `weakref` to avoid keeping objects alive.
- Common leaks: module-level caches that grow forever, listeners/observers holding references, unclosed file/socket handles (use `with`).

## 8. GIL, threads, processes, asyncio

- **GIL**: one process executes Python bytecode at a time. Threads still useful for **I/O-bound** work (network, disk) because the GIL is released during I/O. **CPU-bound** work needs `multiprocessing` (or native libs/NumPy that release the GIL, or Rust/C extensions).
- **asyncio**: single-threaded cooperative concurrency — thousands of open sockets/requests. Requires `async`/`await` all the way down (async DB driver, async HTTP).
- Rule of thumb: I/O-bound high concurrency → asyncio; I/O-bound few calls / already-sync libs → threads; CPU-bound → processes/actors.

```python
import asyncio, httpx

async def fetch(url):
    async with httpx.AsyncClient() as c:
        r = await c.get(url)
        return r.status_code

async def main():
    urls = ["https://example.com"] * 20
    return await asyncio.gather(*[fetch(u) for u in urls])   # concurrent
```

## 9. Typing

```python
from typing import Any, Protocol, TypedDict, TypeVar
def first[T](items: list[T]) -> T: ...          # 3.12 generic syntax

class JSONDict(TypedDict):
    id: int
    name: str

class Repo(Protocol):          # structural typing — any class with these methods fits
    def get(self, id: int) -> Any: ...
```

- Type hints don't enforce at runtime (Pydantic/dataclasses do); they power mypy/pyright + IDEs.
- Know `Optional[X]`, `X | None`, `Literal`, `Annotated`, and why `Any` should be rare.

## 10. Standard library shortlist (name-drop safely)

`collections` (defaultdict, Counter, deque), `itertools` (chain, groupby, islice), `functools` (lru_cache, partial, reduce), `dataclasses`, `pathlib`, `datetime` (timezone-aware!), `json`, `logging`, `unittest.mock`, `sqlite3`, `hashlib`, `secrets`, `asyncio`, `concurrent.futures`.

## 11. Testing & performance

```python
# pytest
def test_sum():
    assert sum([1, 2]) == 3

@pytest.fixture
def client():
    with TestClient(app) as c: yield c

@pytest.mark.parametrize("n,expected", [(1,1),(2,4)])
def test_pow(n, expected): assert n ** 2 == expected
```

- Profiling: `python -m cProfile -s cumtime app.py`, `py-spy` for production, `timeit` for micro-benchmarks.
- Speed tools: `lru_cache`, vectorize with NumPy, move hot loops to native, avoid work in `__init__`/loops (N+1 queries!).

## 12. Security basics

- SQL injection → parameterized queries / ORM only.
- Secrets via env/secret manager, never in git. `hashlib.sha256` for checksums; passwords via bcrypt/argon2 (passlib) — never MD5/SHA1.
- Validate/escape everything from users (Pydantic at the boundary).

---

## Interview Q&A — Python

**Q: List vs tuple; when each?** List mutable (rows you change); tuple immutable (fixed records, dict keys, cheaper).

**Q: `*args` vs `**kwargs`?** Positional extras → tuple; keyword extras → dict. Common signature: `f(*args, **kwargs)` for pass-through wrappers/decorators.

**Q: What is the GIL and does it matter for an API server?** Bytecode lock, one thread per process runs Python at a time. For I/O-heavy APIs it barely matters (I/O releases it); for CPU-heavy endpoints you scale with processes/workers (uvicorn workers, Gunicorn) or push work to native/queue.

**Q: Threading vs multiprocessing vs asyncio?** Threads = cheap I/O concurrency, shared memory. Processes = true parallelism, CPU work, memory isolation. Asyncio = very high connection counts, cooperative, needs async libs.

**Q: Generators — why care?** Lazy evaluation → O(1) memory on huge inputs; pipelines; `yield` as suspend/resume. `iter()`/`next()` under the hood; generators are iterators + context managers (`close()`).

**Q: Shallow vs deep copy?** Shallow: new outer container, same inner objects. Deep: recursively new. Use `copy.deepcopy` for nested mutable structures; beware of copying sockets/DB sessions.

**Q: Mutable default argument bug?** Default evaluated once at definition — shared across calls. Fix with `None` sentinel (§1).

**Q: MRO and `super()`?** C3 linearization orders bases; `super()` follows MRO — essential in cooperative multiple inheritance (e.g., mixin chains).

**Q: `__slots__`?** Predeclares attributes → no per-instance `__dict__` → memory ↓, attribute typos caught. Trade-off: no dynamic attributes.

**Q: Decorators — write one with arguments.** Three-layer wrapper: outer takes decorator args, returns decorator, returns wrapper (or use `functools.partial`/class-based).

**Q: `is` vs `==`; small-int caching?** `is` = identity (same object), `==` = value equality. CPython caches small ints (−5..256) and interns short identifier-like strings — so `a is b` can be True surprisingly; never rely on it.

**Q: What does `with` guarantee?** `__exit__` runs on all paths incl. exceptions → cleanup (files, locks, sessions). Can suppress exceptions by returning True in `__exit__` (don't, unless asked).

**Q: GC of cycles?** Refcount frees acyclic objects; generational GC detects cycles (and is why `__del__` on cyclic objects was historically problematic; `weakref.finalize` is the modern answer).

**Q: How do you handle 10k concurrent HTTP calls in Python?** `asyncio.gather` + async client (httpx/aiohttp), connection pooling, bounded concurrency with `Semaphore`, retries with backoff. Not threads × 10k.

**Q: asyncio.gather vs TaskGroup?** `gather` returns results in order, doesn't cancel siblings on failure. `asyncio.TaskGroup` (3.11+) cancels all on first exception — safer structured concurrency.

**Q: When do you use Pydantic vs dataclasses?** Pydantic: validation/serialization at boundaries (API payloads, config). Dataclasses: cheap internal structs. attrs similar to dataclasses.

**Q: Typing — Protocol vs ABC?** Protocol = structural (duck typing with static checks); ABC = nominal (explicit inheritance). Protocols decouple callers from implementations.

**Q: Logging vs print in production?** Structured `logging` (JSON in prod), levels, correlation IDs. Print has no levels, buffering issues, no context.

**Q: N+1 query problem?** Loop doing one query per row. Fix: `JOIN`/eager loading (`selectinload` in SQLAlchemy), batch `WHERE id IN (...)`.

**Q: How do you make a function thread-safe?** Prefer no shared state; immutable data; `threading.Lock` for critical sections; queues to hand off work.

**Q: `lru_cache` caveats?** Caches by argument hash (args must be hashable), keeps references (memory), unbounded `maxsize=None` can leak; not for functions with side effects or changing external state.

**Q: `async def` vs `def` in a library?** `async def` forces callers to await; you must not call blocking I/O inside it. Libraries like httpx/SQLAlchemy provide both flavors — pick per app model.

**Q: What's in your virtualenv / how do you manage deps?** venv + lockfile (`uv`/`poetry`/`pip-compile`), pinned versions, `pyproject.toml`, separate dev deps (pytest, ruff, mypy).

**Q: How do you find a memory leak?** `tracemalloc` snapshots diff, `objgraph`/`pympler` for reference chains, watch RSS in Grafana, fix the growing module-level cache/listener.

**Q: Python 3.11/3.12 features you like?** `TaskGroup`, `ExceptionGroup`, perf (faster startup/calls), TOML via `tomllib`, generic type syntax `list[T]`, `Self`, improved error messages.

**Q: What makes code "Pythonic"?** Idioms over Java-isms: comprehensions, context managers, duck typing, EAFP ("easier to ask forgiveness than permission" — try/except over `.hasattr`), the Zen of Python (`import this`).
