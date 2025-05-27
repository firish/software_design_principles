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

