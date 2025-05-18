# What is Chain of Responsibility?

The Chain of Responsibility (CoR) pattern arranges a set of handler objects in a linked‐list-like chain; 
an incoming request is passed along that chain until one handler claims it (handles it) or the chain ends. 
Each handler knows only its immediate successor, so sender and final receiver stay loosely coupled.

# Why it is used?

CoR allows you to build flexible, late-bound “pipelines” of logic: 
you can add, reorder, remove, or replace steps without touching the code that issues the request. 
That is valuable when processing rules vary by deployment, evolve over time, or must be combined dynamically (e.g., enabling an extra security check only in production).

# Where you meet it in real life?
- HTTP frameworks (Express, FastAPI, Django’s middleware) walk a request through a list of middlewares; 
- GUI toolkits bubble mouse-events from the deepest widget up to the window;
- enterprise help-desk systems escalate tickets from L1 to L3 support;

# Pros.
- it localises individual policies and keeps callers unaware of concrete handlers
- makes it trivial to plug new behaviour in via configuration—often with no recompilation or redeploy.
- handlers can also decide to partially process and then forward, supporting layered concerns (e.g., auth → rate-limit → business logic).

# Disadvantages.
- indirection can hide who will eventually handle a request and in what order, making debugging harder
- latency becomes less predictable.
- if no handler accepts, the request can vanish silently unless you provide a default “sink.”
- because each step touches the request, deep chains risk performance hits.

# Key considerations.
- establish a clear convention for “claim versus pass-on” (return value, raised exception, or a modified request flag).
- decide whether multiple handlers may act or only the first that matches.
- guard against cycles
- guard against accidentally long chains—especially when handlers can be added at runtime.


