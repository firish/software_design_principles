# S — Single Responsibility Principle (SRP)

## Where it came from
Robert C. Martin ("Uncle Bob") set out five "design principles" in his 2000 paper *Design Principles and Design Patterns*. Michael Feathers later coined the acronym **SOLID** from their initials. SRP is the first of the five: *"A module should have one, and only one, reason to change."*

## What it is & why we have it
SRP asks you to group together code that changes for the *same* reason and to separate code that changes for *different* reasons. Doing so keeps each class or module small, focused, and easier to test. When business rules, UI formats, or persistence strategies evolve, you can touch the class that owns that concern—and nothing else.

## What goes wrong without it
When a single class juggles many concerns (business logic, database I/O, formatting, logging…) every change risks breaking unrelated behavior. Such "God classes" attract merge conflicts, make unit tests brittle, and slow new-feature velocity because developers hesitate to touch them.

## Typical spots to apply SRP
- Domain entities vs. infrastructure (e.g., an `Invoice` object versus the repository that stores it)
- Controller–service–repository slices in web apps
- Micro-utilities such as loggers, validators, formatters, email senders

## A tiny Python example

### Violation of SRP

```python
import sqlite3
import smtplib

class Invoice:
    def __init__(self, amount, customer_email):
        self.amount = amount
        self.customer_email = customer_email
    
    # 1️⃣ business rule
    def apply_discount(self, rate):
        self.amount *= 1 - rate
    
    # 2️⃣ persistence concern
    def save(self):
        with sqlite3.connect("invoices.db") as db:
            db.execute("INSERT INTO invoices(amount,email) VALUES (?,?)", 
                      (self.amount, self.customer_email))
    
    # 3️⃣ presentation / infrastructure
    def send_receipt(self):
        smtp = smtplib.SMTP("mail.acme.com")
        smtp.sendmail("sales@acme.com", self.customer_email, 
                     f"Thanks for your payment of ${self.amount:.2f}")
```

**Problems**: Every time the mailing method changes (SMTP → API) or the database schema evolves, you must edit **Invoice**, risking its business logic and any unit tests that mock it.

### Refactored with SRP

```python
# domain.py
class Invoice:
    def __init__(self, amount):
        self.amount = amount
    
    def apply_discount(self, rate):
        self.amount *= 1 - rate

# repo.py
import sqlite3

class InvoiceRepository:
    def __init__(self, db_path="invoices.db"):
        self._db_path = db_path
    
    def save(self, invoice, email):
        with sqlite3.connect(self._db_path) as db:
            db.execute("INSERT INTO invoices(amount,email) VALUES (?,?)", 
                      (invoice.amount, email))

# notifier.py
import smtplib

class EmailNotifier:
    def __init__(self, host="mail.acme.com"):
        self._host = host
    
    def send_receipt(self, invoice, to_addr):
        with smtplib.SMTP(self._host) as smtp:
            smtp.sendmail("sales@acme.com", to_addr, 
                         f"Thanks for your payment of ${invoice.amount:.2f}")
```

**How SRP helped**: Each class now has one reason to change:
- **Invoice** — business calculations
- **InvoiceRepository** — persistence details  
- **EmailNotifier** — outbound messaging

Switching to a NoSQL store or an AWS SES email gateway touches only the corresponding class. Tests for discount logic no longer need a database or SMTP server, boosting reliability and speed.
