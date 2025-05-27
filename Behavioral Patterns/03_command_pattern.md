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

### Disadvantages
- Turning every operation into an object introduces boilerplate.
- Each concrete command usually needs its own class.
- If you create commands dynamically you may face serialization headaches. 

### Design considerations
- Decide early whether commands must be undoable and design a matching unexecute() method.
- Settle on a serialization format (pickle, JSON, Protocol Buffers) if commands will cross process boundaries.
- Choose where retries and idempotency checks live—inside the command or in the executor.
- Finally, keep the Invoker thin; orchestration policies (delivery guarantees, scheduling, back-off, tracing) belong in a separate command bus layer.

