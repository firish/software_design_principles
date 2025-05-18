# What is Chain of Responsibility?

The Chain of Responsibility (CoR) pattern arranges a set of handler objects in a linked‐list-like chain; 
an incoming request is passed along that chain until one handler claims it (handles it) or the chain ends. 
Each handler knows only its immediate successor, so sender and final receiver stay loosely coupled.

### Why it is used?

CoR allows you to build flexible, late-bound “pipelines” of logic: 
you can add, reorder, remove, or replace steps without touching the code that issues the request. 
That is valuable when processing rules vary by deployment, evolve over time, or must be combined dynamically (e.g., enabling an extra security check only in production).

### Where you meet it in real life?
- HTTP frameworks (Express, FastAPI, Django’s middleware) walk a request through a list of middlewares; 
- GUI toolkits bubble mouse-events from the deepest widget up to the window;
- enterprise help-desk systems escalate tickets from L1 to L3 support;

### Pros.
- it localises individual policies and keeps callers unaware of concrete handlers
- makes it trivial to plug new behaviour in via configuration—often with no recompilation or redeploy.
- handlers can also decide to partially process and then forward, supporting layered concerns (e.g., auth → rate-limit → business logic).

### Disadvantages.
- indirection can hide who will eventually handle a request and in what order, making debugging harder
- latency becomes less predictable.
- if no handler accepts, the request can vanish silently unless you provide a default “sink.”
- because each step touches the request, deep chains risk performance hits.

### Key considerations.
- establish a clear convention for “claim versus pass-on” (return value, raised exception, or a modified request flag).
- decide whether multiple handlers may act or only the first that matches.
- guard against cycles
- guard against accidentally long chains—especially when handlers can be added at runtime.

### Realistic Examples

Imagine a micro-service that receives a Python dict representing an HTTP request. 
We want to enforce authentication, rate-limiting, and finally run the business action. 
The chain must be configurable, so ops can insert or drop steps without editing core logic.

```Python
"""
Minimal Chain-of-Responsibility pipeline for request handling.
Run this file to see the chain in action.
"""

from __future__ import annotations
import time
from abc import ABC, abstractmethod

class Request(dict):
    """Just a dict subclass for clarity; holds headers, body, etc."""

# ---------- Handler base ----------

class Handler(ABC):
    def __init__(self, successor: "Handler | None" = None):
        self._next = successor

    def set_successor(self, successor: "Handler") -> "Handler":
        self._next = successor
        return successor                # fluent API

    def handle(self, request: Request):
        if self._process(request) and self._next:
            self._next.handle(request)

    @abstractmethod
    def _process(self, request: Request) -> bool:
        """Return True to keep propagating, False to stop here."""

# ---------- Concrete handlers ----------

class AuthHandler(Handler):
    def _process(self, request: Request):
        token = request.get("headers", {}).get("Authorization")
        if token != "Bearer secret123":
            raise PermissionError("401 Unauthorized")
        return True                     # authenticated → keep going

class RateLimitHandler(Handler):
    _user_timestamps: dict[str, list[float]] = {}

    def _process(self, request: Request):
        user = request["headers"].get("User-ID")
        now  = time.time()
        window = 60                     # seconds
        bucket = self._user_timestamps.setdefault(user, [])
        bucket[:] = [t for t in bucket if now - t < window]
        if len(bucket) >= 5:
            raise RuntimeError("429 Too Many Requests")
        bucket.append(now)
        return True

class BusinessLogicHandler(Handler):
    def _process(self, request: Request):
        item_id = request["params"]["item_id"]
        print(f"⚙️  Fetching item {item_id}")
        request["response"] = {"status": "ok", "item_id": item_id}
        return False                    # handled → stop chain

# ---------- Building the chain ----------

def build_chain() -> Handler:
    return AuthHandler(
        RateLimitHandler(
            BusinessLogicHandler()
        )
    )

# ---------- Example run ----------

if __name__ == "__main__":
    chain = build_chain()

    req = Request(
        headers={"Authorization": "Bearer secret123", "User-ID": "u42"},
        params={"item_id": 7},
    )

    chain.handle(req)
    print(req["response"])              # {'status': 'ok', 'item_id': 7}
```

Note:
- Unit tests instantiate only the handler under test and supply a mock successor, keeping tests isolated.
- Because the service code never imports AuthHandler directly, swapping to OAuth or disabling rate-limiting in a staging environment is simply a matter of assembling a different chain at startup—exactly the maintenance flexibility that Chain of Responsibility is designed to provide.


Below is one way to make the same micro-service handlers configurable:
- Start-up assembly: read an ordered list of handler names from an .ini, ENV var, or feature-flag service and build the chain before the first request arrives.
- Run-time extension: expose a management endpoint that can splice an extra handler into the already running chain (for example, turn on verbose tracing in production when you chase a bug). The new step is an ordinary Python object that already lives in the codebase, so the process does not need a redeploy—just a signal to re-wire the chain.

```Python
class AuthHandler(Handler): ...
class RateLimitHandler(Handler): ...
class BusinessLogicHandler(Handler): ...
class TraceHandler(Handler):             # new, optional
    def _process(self, request):
        print("🔍 TRACE →", request)
        return True

HANDLER_REGISTRY = {
    "auth"       : "AuthHandler",
    "ratelimit"  : "RateLimitHandler",
    "trace"      : "TraceHandler",
    "business"   : "BusinessLogicHandler",
}

def build_chain_from_env() -> Handler:
    """
    Read HANDLERS env var like:
      HANDLERS='["auth","ratelimit","business"]'
    """
    order = json.loads(os.getenv("HANDLERS", '["auth","business"]'))
    chain_head = None
    current = None

    for key in order:
        cls_name = HANDLER_REGISTRY[key]
        # assume handlers live in this module; otherwise importlib.import_module
        cls = globals()[cls_name]
        node = cls()
        if chain_head is None:
            chain_head = node
        else:
            current.set_successor(node)
        current = node

    return chain_head
```

A system administrator can now switch the pipeline by editing an environment variable and restarting the service:

```Python
# standard pipeline
export HANDLERS='["auth","ratelimit","business"]'     # default
python app.py

# add tracing without touching code
export HANDLERS='["trace","auth","ratelimit","business"]'
python app.py
```


Live re-wiring while the process runs
Add a very small “control plane” endpoint. (In real life you would secure this with mTLS or an admin token.)

```Python
from fastapi import FastAPI, HTTPException

app = FastAPI()
chain = build_chain_from_env()        # initial pipeline

@app.post("/__admin/insert_trace")
def enable_trace():
    global chain
    # walk to the end so Trace comes first (acts like pre-hook)
    trace = TraceHandler()
    trace.set_successor(chain)        # Trace → existing head
    chain = trace                     # new head becomes active immediately
    return {"status": "trace inserted"}

@app.post("/__admin/remove_trace")
def disable_trace():
    global chain
    if isinstance(chain, TraceHandler):
        chain = chain._next           # drop the head → no Trace
        return {"status": "trace removed"}
    raise HTTPException(400, "trace not enabled")
```
Ops can now toggle tracing without redeploying and without losing in-flight connections:

```bash
curl -XPOST http://service/__admin/insert_trace     # turn on
# … reproduce the bug, collect logs …
curl -XPOST http://service/__admin/remove_trace     # turn off
```

Note:
Hot insertion of a tracing or feature-flag handler is standard practice in production micro-services (e.g., Envoy filters, API-gateway plugins) when you need extra diagnostics or to roll out a new policy gradually.

Neither technique requires code edits in the business layer or a redeploy of containers—only a config change or an admin API call—illustrating the core benefit of the Chain of Responsibility pattern in live systems.








