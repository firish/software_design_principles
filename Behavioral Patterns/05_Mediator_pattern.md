## What is the Mediator Pattern?

The Mediator pattern introduces a dedicated object (the mediator) that encapsulates how a collection of peers collaborate.
Instead of peers calling one another directly, they talk only to the mediator, which transforms, forwards, or coordinates the messages. 
Visualization: Communication dependency arrows collapse from an all-to-all mesh to a tidy star. 

### Why is it adopted?

- Tames combinatorial coupling: N components wired pair-wise need N × (N−1) references; with a mediator each holds just one.
- Centralises complex workflows: Validation, sequencing, retries, and logging live in one class instead of being half-duplicated in every participant.
- Improves replaceability & testability: A component can be mocked or swapped without touching the others; unit tests inject a fake mediator.

### Typical real-world use cases?

- Aircraft control:	The control tower mediates between planes that never radio one another directly.
- Chat / collaboration apps:	A chat-room hub receives a message and fan-outs to participants.
- Backend architectures:	An in-process event bus dispatches domain events between decoupled modules; a message broker plays the same role across processes.

### Pros & downsides

Pros:
- Cuts direct dependencies, boosting cohesion inside each colleague class.
- Central point for cross-cutting policies (transactions, tracing, security).
- Enables plug-in architectures (new component registers with mediator, no code change elsewhere).

Cons:
- Mediator itself can balloon into a “god object” if every collaboration rule moves there.
- Single point of failure
- Performance bottleneck if not designed carefully.
- Extra indirection can make call sequences harder to trace while debugging.

### Considerations

Design hints: 
- keep mediators thin and, if they grow, split them by bounded context.*
- decide whether communication is synchronous (method call) or async (queue).
- guard against cyclic chatter.
- test the mediator highly, as it is a SPOF.

Why should mediators always be kept thin?
- they are for communication management. they shouldn't hold any business logic.
- if you add business logic, they violate the "Single Responsibility Principle" from SOLID
- if a mediator starts bloating, you should create a seperate a class for offloading certain non-communication management logic off the mediator

## Code Example

Large Python back-ends (Django, FastAPI, Flask) increasingly apply “domain events” to decouple modules. 
when an order is paid, the payment module should not import inventory or email code.
A lightweight EventBus mediator routes the OrderPaid event to whatever handlers have subscribed to it.

First look at the app code where every class can talk to every other class freely.
In programming terms, this is also called `Spaghetti`
```python
# checkout_direct.py
from dataclasses import dataclass

@dataclass
class InventoryUpdate:
    def update_inventory(self, order_id: int):
        print(f"📦  Updated Inventory after processing order {order_id}")

@dataclass
class InventoryService:
    def reserve_stock(self, order_id: int):
        print(f"📦  Reserving items for order {order_id}")
        # Talks directly to class InventoryUpdate
        inv_update_service = InventoryUpdate()
        inv_update_service.update_inventory(order_id)

@dataclass
class NotificationService:
    def send_receipt(self, order_id: int, email: str):
        print(f"✉️   Emailing receipt to {email} for order {order_id}")

@dataclass
class PaymentService:
    inventory: InventoryService
    notifier:  NotificationService

    def capture_payment(self, order_id: int, amount_cents: int, email: str):
        print(f"💳  Captured ${amount_cents/100:.2f} for order {order_id}")

        # --- direct calls → hard coupling ---------------------
        self.inventory.reserve_stock(order_id)
        self.notifier.send_receipt(order_id, email)
        # (next week if a PM wants loyalty points, you will add *another* direct call)
        # ------------------------------------------------------


# ---------- Demo Driver ----------
if __name__ == "__main__":
    inv  = InventoryService()
    note = NotificationService()
    pay  = PaymentService(inv, note)

    # Trigger
    pay.capture_payment(42, 2599, "alice@example.com")
```
Adding a new listener (e.g., LoyaltyService) means editing PaymentService code and all its tests.
Additionally, in this example, paymentService is orchestrating most of the communication, with other services rarely talking internally.
But, as the inter talks increase, the dependencies get more coupled and management becomes a serious challenge. 


We can get rid of these problems by officially creating a mediator for orchestrating any inter-class communication.
This is done typically in the form of an event bus or a message broker. 
```python

```
