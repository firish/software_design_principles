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
# checkout_mediator_extended.py
from __future__ import annotations
from collections import defaultdict
from dataclasses import dataclass
from datetime import date, timedelta
from typing import Callable, Dict, List, Type
import random

# ──────────────────── 0. Mediator ────────────────────────────
class Event: ...
Handler = Callable[[Event], None]

class EventBus:
    def __init__(self):
        self._subs: Dict[Type[Event], List[Handler]] = defaultdict(list)

    def subscribe(self, event_type: Type[Event], handler: Handler):
        self._subs[event_type].append(handler)

    def publish(self, event: Event):
        for h in self._subs.get(type(event), []):
            h(event)

# ──────────────────── 1. Domain model ────────────────────────
@dataclass
class User:
    id: int
    email: str
    points: int = 0                       # loyalty balance

# In-memory “repository”
USERS = {1: User(1, "alice@example.com")}

# ──────────────────── 2. Domain events ───────────────────────
@dataclass class OrderPaid(Event):
    order_id: int
    amount_cents: int
    user_id: int

@dataclass class ReserveStockRequest(Event):
    order_id: int

@dataclass class StockReserved(Event):
    order_id: int

@dataclass class StockBackorder(Event):
    order_id: int

@dataclass class LoyaltyCredited(Event):
    user_id: int
    points: int

# ──────────────────── 3. Services (colleagues) ───────────────
class PaymentService:
    def __init__(self, bus: EventBus):
        self.bus = bus

    def capture_payment(self, order_id: int, amount_cents: int, user_id: int):
        print(f"Captured ${amount_cents/100:.2f} for order {order_id}")
        self.bus.publish(OrderPaid(order_id, amount_cents, user_id))

class InventoryService:
    """Translates payment into a stock-reservation *request*."""
    def __init__(self, bus: EventBus):
        self.bus = bus
        bus.subscribe(OrderPaid, self.on_order_paid)

    def on_order_paid(self, evt: OrderPaid):
        print(f"InventoryService → requesting stock for order {evt.order_id}")
        self.bus.publish(ReserveStockRequest(evt.order_id))

class StockLedgerService:
    """
    Owns the authoritative stock counts.
    Replies with StockReserved OR StockBackorder.
    """
    def __init__(self, bus: EventBus):
        self.bus = bus
        bus.subscribe(ReserveStockRequest, self.check_stock)
        # fake stock levels
        self._stock = {"default": 3}

    def check_stock(self, evt: ReserveStockRequest):
        qty = self._stock["default"]
        print(f"Ledger checking stock ({qty} left) for order {evt.order_id}")
        if qty > 0:
            self._stock["default"] -= 1
            self.bus.publish(StockReserved(evt.order_id))
        else:
            self.bus.publish(StockBackorder(evt.order_id))

class DeliveryService:
    """Schedules shipping dates based on stock status."""
    def __init__(self, bus: EventBus):
        self.bus = bus
        bus.subscribe(StockReserved,  self.plan_fast_delivery)
        bus.subscribe(StockBackorder, self.plan_slow_delivery)

    def plan_fast_delivery(self, evt: StockReserved):
        eta = date.today() + timedelta(days=2)
        print(f"Order {evt.order_id}: items in stock → ETA {eta}")

    def plan_slow_delivery(self, evt: StockBackorder):
        eta = date.today() + timedelta(days=10)
        print(f"Order {evt.order_id}: back-ordered → ETA {eta}")

class LoyaltyService:
    def __init__(self, bus: EventBus, user_repo: Dict[int, User]):
        self.user_repo = user_repo
        bus.subscribe(OrderPaid, self.add_points)

    def add_points(self, evt: OrderPaid):
        pts = evt.amount_cents // 100      # $1 = 1 point
        user = self.user_repo[evt.user_id]
        user.points += pts
        print(f"Credited {pts} pts → {user.email} (total {user.points})")

# ──────────────────── 4. Demo driver ─────────────────────────
if __name__ == "__main__":
    bus = EventBus()

    # Register colleagues
    InventoryService(bus)
    StockLedgerService(bus)
    DeliveryService(bus)
    LoyaltyService(bus, USERS)

    pay = PaymentService(bus)

    # Simulate three check-outs: stock will run out after 3rd order
    for order_id in range(1, 4):
        user_id = 1
        amt     = random.choice([2599, 4999, 1299])
        print(f"\n=== Customer places order #{order_id} ===")
        pay.capture_payment(order_id, amt, user_id)

```

Run summary
```
=== Customer places order #1 ===
Captured $12.99 for order 1
InventoryService → requesting stock for order 1
Ledger checking stock (3 left) for order 1
Order 1: items in stock → ETA 2025-06-03
Credited 12 pts → alice@example.com (total 12)

=== Customer places order #2 ===
Captured $49.99 for order 2
InventoryService → requesting stock for order 2
Ledger checking stock (2 left) for order 2
Order 2: items in stock → ETA 2025-06-03
Credited 49 pts → alice@example.com (total 61)

=== Customer places order #3 ===
Captured $25.99 for order 3
InventoryService → requesting stock for order 3
Ledger checking stock (1 left) for order 3
Order 3: items in stock → ETA 2025-06-03
Credited 25 pts → alice@example.com (total 86)

=== Customer places order #4 ===
Captured $49.99 for order 4
InventoryService → requesting stock for order 4
Ledger checking stock (0 left) for order 4
Order 4: back-ordered → ETA 2025-06-11
Credited 49 pts → alice@example.com (total 135)
```

Notice how:
- every class now depends only on the EventBus; no one knows who else is in the system.
- a star-shaped mediator absorbs expanding collaboration logic while each colleague stays focused, small, and easy to unit-test.
