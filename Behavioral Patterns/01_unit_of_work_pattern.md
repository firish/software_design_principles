# The Unit of Work (UoW) Pattern

The **Unit of Work (UoW)** is **not** one of the original Gang of Four (GoF) design patterns. 
It is a behavorial type patter.
It’s a pattern frequently found in enterprise applications, popularized by [Martin Fowler](https://martinfowler.com/eaaCatalog/unitOfWork.html). 

In short, the **Unit of Work** pattern **coordinates changes** to a set of objects,
to ensure that either all changes succeed or all fail together. 
It keeps track of **new**, **changed**, and **deleted** objects so that you can **commit** them in a single transaction, ensuring consistency.


## What is the Unit of Work Pattern?

1. **Tracks Changes**: The Unit of Work object monitors changes to entities in your application.  
2. **Batch Operations**: Instead of applying changes individually to the database or data store, it batches them, and then applies them in one go (or transaction).  
3. **Transaction-Like Semantics**: If an error occurs, you can roll back the transaction, leaving your data consistent.  

Essentially, the **Unit of Work** acts like a **to-do list** of data operations (inserts, updates, deletes) that need to be performed. 
When you **commit** the Unit of Work, it executes all these operations together.


## Why Use Unit of Work?

- **Consistency**: Ensures all related data changes are committed together.  
- **Performance**: Instead of multiple small database calls, you can group them into fewer, more significant operations.  
- **Isolation**: Changes are collected and only made permanent upon commit, reducing partial updates or inconsistent states.
  

## Real-World Example: E-Commerce Checkout

Consider a **simple e-commerce** scenario. We have:

- **Order**: Represents a customer’s order.
- **OrderItem**: Represents a product within that order.
- **Payment**: Payment details for the order.

When a user checks out:
1. We create an **Order** object.
2. We add **OrderItem** objects to the order.
3. We insert or update **Payment** details.

We want to ensure all these changes happen **together**, while creating the invoice (all succeed or all fail).

Below, we’ll show a **Unit of Work** approach in Python and how it might work in an in-memory or stubbed environment.


### 1. Domain Models
```python
class Order:
    def __init__(self, order_id, customer_name):
        self.order_id = order_id
        self.customer_name = customer_name
        self.items = []
        self.is_paid = False

    def add_item(self, item):
        self.items.append(item)

    def __repr__(self):
        return (f"Order(order_id={self.order_id}, "
                f"customer={self.customer_name}, "
                f"is_paid={self.is_paid}, "
                f"items={self.items})")


class OrderItem:
    def __init__(self, product_name, quantity, price, shipping):
        self.product_name = product_name
        self.quantity = quantity
        self.price = price
        self.shipping = shipping

    def get_item_total():
      return (self.price * self.quantity) + self.shipping

    def __repr__(self):
        return (f"OrderItem(product_name={self.product_name}, "
                f"quantity={self.quantity}, price={self.price})")


class Payment:
    def __init__(self, payment_id, order_id, amount, status="PENDING"):
        self.payment_id = payment_id
        self.order_id = order_id
        self.amount = amount
        self.status = status  # {PENDING, SUCCESS, FAILED}

    def __repr__(self):
        return (f"Payment(payment_id={self.payment_id}, "
                f"order_id={self.order_id}, "
                f"amount={self.amount}, status={self.status})")
```

Let’s see how we might handle data changes without UoW. 
We might have direct calls to some repository or data layer each time we add or update something.

```python
class ECommerceServiceWithoutUoW:
    """
    This class updates data as soon as changes occur.
    No concept of a single commit. 
    Potentially multiple partial commits, leading to inconsistency if errors happen.
    """

    def __init__(self):
        self.orders = {}    # pretend these are saved in a DB
        self.payments = {}  # another data store for payments

    def create_order(self, order_id, customer_name):
        if order_id in self.orders:
            raise ValueError(f"Order with ID '{order_id}' already exists!")
        order = Order(order_id, customer_name)
        self.orders[order_id] = order
        print(f"[WithoutUoW] Order {order_id} created and saved immediately.")
        return order

    def add_item_to_order(self, order_id, product_name, quantity, price):
        order = self.orders.get(order_id)
        if not order:
            raise ValueError(f"Order '{order_id}' does not exist.")
        item = OrderItem(product_name, quantity, price)
        order.add_item(item)
        # direct save to data store
        self.orders[order_id] = order
        print(f"[WithoutUoW] Item {item} added and saved immediately to order {order_id}.")

    def make_payment(self, payment_id, order_id, amount):
        if payment_id in self.payments:
            raise ValueError(f"Payment with ID '{payment_id}' already exists!")
        payment = Payment(payment_id, order_id, amount)
        # direct save
        self.payments[payment_id] = payment
        print(f"[WithoutUoW] Payment {payment_id} created immediately.")
        return payment

    def finalize_order(self, order_id):
        # business logic .....
        order = self.orders.get(order_id)
        if not order:
            raise ValueError(f"Order '{order_id}' not found.")
        # Payment is separate
        # If payment fails or not made, we have partial data
        order.is_paid = True
        self.orders[order_id] = order
        print(f"[WithoutUoW] Order {order_id} marked as paid immediately.")
```

Problems with this approach:
- Each action writes data immediately.
- If something fails mid-process (e.g., we add items but fail to create a payment), we might have a half-created order.
- No single transaction. We can’t easily roll back partial changes.

We introduce a UnitOfWork class that tracks new, dirty (updated), and removed entities. We only commit them at the end.
```python
class OrderRepository:
    def __init__(self):
        self._orders = {}  # order_id -> Order

    def add(self, order: Order):
        self._orders[order.order_id] = order

    def get(self, order_id):
        return self._orders.get(order_id)

    def remove(self, order_id):
        if order_id in self._orders:
            del self._orders[order_id]

    def all(self):
        return list(self._orders.values())


class PaymentRepository:
    def __init__(self):
        self._payments = {}  # payment_id -> Payment

    def add(self, payment: Payment):
        self._payments[payment.payment_id] = payment

    def get(self, payment_id):
        return self._payments.get(payment_id)

    def remove(self, payment_id):
        if payment_id in self._payments:
            del self._payments[payment_id]

    def all(self):
        return list(self._payments.values())
```

```python
class UnitOfWork:
    """
    Tracks changes to Orders and Payments within a single "transaction".
    We keep references to 'new', 'modified', 'deleted' objects and 
    only commit them all at once.
    """
    def __init__(self, order_repository: OrderRepository, payment_repository: PaymentRepository):
        self.order_repository = order_repository
        self.payment_repository = payment_repository
        self.new_orders = []
        self.modified_orders = []
        self.deleted_orders = []
        self.new_payments = []
        self.modified_payments = []
        self.deleted_payments = []

    def register_new_order(self, order: Order):
        self.new_orders.append(order)

    def register_modified_order(self, order: Order):
        if order not in self.new_orders:
            self.modified_orders.append(order)

    def register_deleted_order(self, order_id: str):
        self.deleted_orders.append(order_id)

    def register_new_payment(self, payment: Payment):
        self.new_payments.append(payment)

    def register_modified_payment(self, payment: Payment):
        if payment not in self.new_payments:
            self.modified_payments.append(payment)

    def register_deleted_payment(self, payment_id: str):
        self.deleted_payments.append(payment_id)

    def commit(self):
        """
        Apply all pending changes in a single "transaction".
        For in-memory, we'll just update the repositories. 
        But in a real DB scenario, we might wrap this in an actual DB transaction.
        """
        # Orders
        for order in self.new_orders:
            self.order_repository.add(order)
        for order in self.modified_orders:
            self.order_repository.add(order)
        for order_id in self.deleted_orders:
            self.order_repository.remove(order_id)

        # Payments
        for payment in self.new_payments:
            self.payment_repository.add(payment)
        for payment in self.modified_payments:
            self.payment_repository.add(payment)
        for payment_id in self.deleted_payments:
            self.payment_repository.remove(payment_id)

        # Clear the lists after commit
        self.new_orders.clear()
        self.modified_orders.clear()
        self.deleted_orders.clear()
        self.new_payments.clear()
        self.modified_payments.clear()
        self.deleted_payments.clear()

    def rollback(self):
        """
        If something goes wrong, ignore all changes or revert. 
        In real DB code, we'd do an actual rollback.
        """
        self.new_orders.clear()
        self.modified_orders.clear()
        self.deleted_orders.clear()
        self.new_payments.clear()
        self.modified_payments.clear()
        self.deleted_payments.clear()
        print("[UnitOfWork] Rolled back changes.")
```

```python
class ECommerceServiceWithUoW:
    """
    Instead of writing data immediately, we register changes with the UoW.
    We only commit once all operations are done.
    """

    def __init__(self, uow: UnitOfWork):
        self.uow = uow

    def create_order(self, order_id, customer_name):
        existing = self.uow.order_repository.get(order_id)
        if existing:
            raise ValueError(f"Order with ID '{order_id}' already exists!")
        order = Order(order_id, customer_name)
        self.uow.register_new_order(order)
        print(f"[WithUoW] Created new Order. Changes staged, not committed yet.")
        return order

    def add_item_to_order(self, order_id, product_name, quantity, price):
        order = self.uow.order_repository.get(order_id)
        if not order:
            raise ValueError(f"Order '{order_id}' does not exist!")
        item = OrderItem(product_name, quantity, price)
        order.add_item(item)
        # Mark order as modified
        self.uow.register_modified_order(order)
        print(f"[WithUoW] Added item {item} to {order_id}. Changes staged.")

    def make_payment(self, payment_id, order_id, amount):
        existing = self.uow.payment_repository.get(payment_id)
        if existing:
            raise ValueError(f"Payment with ID '{payment_id}' already exists!")
        payment = Payment(payment_id, order_id, amount)
        self.uow.register_new_payment(payment)
        print(f"[WithUoW] Created Payment. Changes staged.")
        return payment

    def finalize_order(self, order_id):
        order = self.uow.order_repository.get(order_id)
        if not order:
            raise ValueError(f"Order '{order_id}' not found!")
        order.is_paid = True
        self.uow.register_modified_order(order)
        print(f"[WithUoW] Marked order {order_id} as paid. Changes staged.")

    def commit_changes(self):
        """
        If everything is good, commit the changes to the repositories in one go.
        """
        self.uow.commit()
        print("[WithUoW] All changes committed successfully.")

    def rollback_changes(self):
        """
        Discard any staged changes.
        """
        self.uow.rollback()
        print("[WithUoW] Changes rolled back.")
```

- Unit of Work collects changes (new, modified, deleted entities) within a single scope.
- Commit applies them all at once (like a transaction).
- Rollback discards them if an error occurs.

