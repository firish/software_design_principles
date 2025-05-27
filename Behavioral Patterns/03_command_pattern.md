## What is the Command Pattern?
“Command” turns an action into an object. 
Instead of calling a method such as door.open() directly, 
the client creates a OpenDoorCommand(receiver=door) object and hands it to someone else (an invoker).
That object stores everything required to perform the action: target, parameters, and even how to undo.
So the invoker can queue it, log it, retry it, or execute it later without caring what the action actually does.

### Why it is used?
Packaging work in this way decouples the code that requests an operation from the code that performs it. 
Because requests are now first-class objects they can be:
- persisted to a database,
- replayed after a crash,
- moved to a different thread or process,
- logged for auditing, and
- access control—all without editing the business logic that lives inside the command.

### Where you meet it in practice
- GUI frameworks keep an “Undo/Redo” stack of command objects;
- task queues such as Celery, RQ, or Sidekiq pickle commands and run them in background workers;
- event-sourced systems record commands in append-only logs;
- message buses in micro-service architectures transmit serialized commands between services;
- database migrations are executed through an ordered list of command objects;

### Advantages
- Commands are self-contained and serialisable, making deferred or distributed execution trivial.
- They centralise cross-cutting policies (logging, retries, transactions).
- They make undo/redo natural because each command can know how to reverse itself.
- Testing becomes easier: you feed a command to an executor and assert on side-effects.

Note:
When Python (or any language) creates an object, that object lives inside the interpreter’s memory—it is made of pointers, type flags, and OS-specific addresses that have meaning only inside that single process. 
Serialization is the act of turning that in-memory object into a stream of bytes (or a text string) that can be:
- Stored on disk, in a database, or in a message queue
- Sent over a pipe, socket, or HTTP request to another process—perhaps running on a different machine or even written in another language
- The reverse operation: rebuilding a live object from that byte sequence—is called deserialization (or “unmarshalling”).

Note:
Your commands like, ResizeImageCommand, SendEmailCommand, etc. are not going to be executed immediately in the same Python interpreter that created them. 
Instead they will be put on a queue (Redis, SQS, Kafka…), read by a different worker process, and only then executed.
Because two different processes are involved, the command object must travel as inert data—no live pointers, no open file handles.
Hence you must choose a concrete encoding—a serialization format—that both the producer and the consumer understand.

### Disadvantages
- Turning every operation into an object introduces boilerplate.
- Each concrete command usually needs its own class.
- If you create commands dynamically you may face serialization headaches. 

### Design considerations
- Decide early whether commands must be undoable and design a matching unexecute() method.
- Settle on a serialization format (pickle, JSON, Protocol Buffers) if commands will cross process boundaries.
- Choose where retries and idempotency checks live—inside the command or in the executor.
- Finally, keep the Invoker thin; orchestration policies (delivery guarantees, scheduling, back-off, tracing) belong in a separate command bus layer.


### Practical example

Modern SaaS back-ends off-load slow work—sending email, resizing images, generating PDF invoices—to background workers so that the HTTP response returns quickly. 
Each unit of work is a command placed on a queue; one or more worker processes pop commands and execute them.

```python
"""
$ python job_queue.py
→ Pushed 3 commands onto the queue
→ Worker executes: ResizeImageCommand(path='assets/photo.jpg', size=(800, 600))
   Resizing assets/photo.jpg to (800, 600) ...
→ Worker executes: SendEmailCommand(to='alice@example.com', subject='Welcome!')
   Sending email to alice@example.com ...
→ Worker executes: GenerateInvoicePDFCommand(order_id=1234)
   Generating PDF invoice for order 1234 ...
"""
from __future__ import annotations
from abc import ABC, abstractmethod
from dataclasses import dataclass, asdict
import json
from queue import SimpleQueue        # stand-in for Redis / SQS

# ---------- Command infrastructure ----------

class Command(ABC):
    """Abstract base class every job must implement."""
    
    @abstractmethod
    def execute(self) -> None: ...

    # optional: payload for persistence / transport
    def serialize(self) -> str:
        return json.dumps({"type": self.__class__.__name__, "data": asdict(self)})

    @staticmethod
    def deserialize(payload: str) -> "Command":
        registry = {cls.__name__: cls for cls in Command.__subclasses__()}
        obj = json.loads(payload)
        return registry[obj["type"]](**obj["data"])

# ---------- Concrete commands ----------

@dataclass
class ResizeImageCommand(Command):
    path: str
    size: tuple[int, int]

    def execute(self):
        print(f"   Resizing {self.path} to {self.size} ...")

@dataclass
class SendEmailCommand(Command):
    to: str
    subject: str
    body: str = ""

    def execute(self):
        print(f"   Sending email to {self.to} ...")

@dataclass
class GenerateInvoicePDFCommand(Command):
    order_id: int

    def execute(self):
        print(f"   Generating PDF invoice for order {self.order_id} ...")

# ---------- Simple in-memory queue (invoker) ----------

class JobQueue:
    def __init__(self):
        self._q = SimpleQueue()

    def push(self, cmd: Command):
        self._q.put(cmd.serialize())

    def pop(self) -> Command | None:
        if self._q.empty():
            return None
        return Command.deserialize(self._q.get())

# ---------- Worker loop (receiver) ----------

def worker_loop(job_queue: JobQueue):
    while (cmd := job_queue.pop()) is not None:
        print(f"→ Worker executes: {cmd}")
        cmd.execute()

# ---------- Demonstration ----------

if __name__ == "__main__":
    q = JobQueue()

    # web layer enqueues jobs
    q.push(ResizeImageCommand("assets/photo.jpg", (800, 600)))
    q.push(SendEmailCommand("alice@example.com", "Welcome!"))
    q.push(GenerateInvoicePDFCommand(1234))
    print("→ Pushed 3 commands onto the queue")

    # a separate worker process (simulated here) consumes them
    worker_loop(q)

```
This example showcases:
- Loose coupling: the HTTP handler never imports SendEmailCommand; it just pushes a serialized job onto the queue.
- Durability and retries: because each command is JSON-serialised, the queue could be Redis, RabbitMQ, or AWS SQS, and a worker can retry the same payload after a crash.
- Extensibility: adding a PostToSlackCommand means writing one dataclass; no changes to the queue or worker code.


Advanced, with undo,
```python
"""
job_queue_topics.py
-----------------------------------
Demonstrates:
  • Topic-based routing: each worker subscribes to specific command types.
  • Per-worker undo stack.
"""
from __future__ import annotations
from abc import ABC, abstractmethod
from dataclasses import dataclass, asdict
from collections import defaultdict
from queue import SimpleQueue
import json
import time

# ──────────────────  Command infrastructure  ──────────────────
class Command(ABC):
    """Every concrete job must implement execute() *and* undo()."""

    @abstractmethod
    def execute(self) -> None: ...
    @abstractmethod
    def undo(self) -> None: ...

    # ——— Serialization helpers (JSON for language-agnostic wire format) ———
    def serialize(self) -> str:
        return json.dumps({"type": self.__class__.__name__, "data": asdict(self)})

    @staticmethod
    def deserialize(payload: str) -> "Command":
        registry = {cls.__name__: cls for cls in Command.__subclasses__()}
        obj = json.loads(payload)
        return registry[obj["type"]](**obj["data"])

# ──────────────────  Concrete commands  ──────────────────
@dataclass
class ResizeImageCommand(Command):
    path: str
    size: tuple[int, int]

    def execute(self):
        print(f"🖼️  Resizing {self.path} to {self.size} ...")
        # record “old size” for undo (simulated)
        time.sleep(0.1)
        self._old_size = (1920, 1080)

    def undo(self):
        print(f"↩️  Reverting {self.path} back to {self._old_size} ...")

@dataclass
class SendEmailCommand(Command):
    to: str
    subject: str
    body: str = ""

    def execute(self):
        print(f"✉️  Sending email to {self.to!r} with subject '{self.subject}'")
        time.sleep(0.1)

    def undo(self):
        print(f"⚠️  Cannot unsend email to {self.to!r} — logging compensation action")

@dataclass
class GenerateInvoicePDFCommand(Command):
    order_id: int

    def execute(self):
        print(f"📄 Generating PDF invoice for order {self.order_id}")
        time.sleep(0.1)
        self._filepath = f"invoices/{self.order_id}.pdf"

    def undo(self):
        print(f"🗑️  Deleting generated file {self._filepath}")

# ──────────────────  Topic-aware queue  ──────────────────
class JobQueue:
    """
    Very small stand-in for a broker that supports *topics*.
    One SimpleQueue per command type.
    """
    def __init__(self):
        self._topics: dict[str, SimpleQueue[str]] = defaultdict(SimpleQueue)

    def push(self, cmd: Command):
        topic = cmd.__class__.__name__
        self._topics[topic].put(cmd.serialize())

    def pop(self, topics: list[str]) -> Command | None:
        """
        Try each subscribed topic in order; return first available command.
        """
        for t in topics:
            q = self._topics[t]
            if not q.empty():
                return Command.deserialize(q.get())
        return None

# ──────────────────  Worker  ──────────────────
class Worker:
    """
    A worker subscribes to one or more topics (command classes) and
    keeps its own undo stack for rollback capability.
    """
    worker_counter = 0

    def __init__(self, queue: JobQueue, subscriptions: list[type[Command]]):
        Worker.worker_counter += 1
        self.id = Worker.worker_counter
        self.queue = queue
        self.subscribed_names = [cls.__name__ for cls in subscriptions]
        self._undo_stack: list[Command] = []

    def run_once(self):
        cmd = self.queue.pop(self.subscribed_names)
        if cmd:
            print(f"\n▶️ Worker#{self.id} executing {cmd}")
            cmd.execute()
            self._undo_stack.append(cmd)

    # Simple rollback of the last executed command
    def rollback_last(self):
        if self._undo_stack:
            cmd = self._undo_stack.pop()
            print(f"\n⏪ Worker#{self.id} rolling back {cmd}")
            cmd.undo()
        else:
            print(f"Worker#{self.id} undo stack empty")

# ──────────────────  Demo  ──────────────────
if __name__ == "__main__":
    queue = JobQueue()

    # Simulate web/front-end layer enqueuing three different jobs
    queue.push(ResizeImageCommand("assets/photo.jpg", (800, 600)))
    queue.push(SendEmailCommand("alice@example.com", "Welcome!"))
    queue.push(GenerateInvoicePDFCommand(1234))
    print("📥 Pushed 3 jobs")

    # Create *specialised* workers
    img_worker   = Worker(queue, [ResizeImageCommand])
    email_worker = Worker(queue, [SendEmailCommand])
    pdf_worker   = Worker(queue, [GenerateInvoicePDFCommand])

    # Pretend to be a simple scheduler: each worker polls once per tick
    for _ in range(3):
        img_worker.run_once()
        email_worker.run_once()
        pdf_worker.run_once()

    # Roll back the last PDF generation
    pdf_worker.rollback_last()
```

Each command’s class name acts as a topic key.
When the front-end enqueues a job, the queue automatically drops it into self._topics["ResizeImageCommand"], ...["SendEmailCommand"], etc.
Each worker subscribes to the topics it cares about; a photo-processing container never even sees email jobs, just like a real world systems.

