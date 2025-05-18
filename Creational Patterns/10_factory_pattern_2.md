# What is Factory Pattern?

A Factory pattern replaces direct constructor calls with a dedicated method (factory)
that decides which concrete class to instantiate and returns it through a common interface or abstract base class. 
Creation logic is therefore delegated to the factory, while the calling code stays agnostic of the actual class names.

# why is Factory used?

Hiding the new/__init__ decision point isolates the rest of the program from changes in concrete classes (e.g., swapping a StripeProcessor for a PayPalProcessor). 
This lowers coupling, supports the Open/Closed Principle, and lets us defer or vary instantiation based on runtime parameters (configuration files, environment variables, user input).

Note: The Open/Closed Principle (OCP) states that a software module should be open for extension but closed for modification. 
This means you should be able to add new functionality to a module without having to alter the existing code. 
In essence, the principle encourages designing systems where changes are made by adding new code, not by altering the existing, tested code.
Interfaces/ABCs are a key tool for achieving OCP. 
By using interfaces, you can define a contract that different classes can implement, allowing them to be extended without modifying the core code. 

# Collocial examples

Everyday libraries already shield you with factories:
- logging.getLogger() chooses or creates the appropriate Logger instance;
- multiprocessing.Manager() hands back subtype managers depending on your platform;
- sqlalchemy.create_engine(), maps a URI string to the correct dialect class, hiding dozens of subclasses behind one call.

# Advantages
Because creation code lives in one place, it easy to:
- swap/add implementations
- add caching, pooling, or dependency-injection hooks (requires local edits only).
- unit tests can pass in mocks through the factory; callers keep using the same interface.

# Drawbacks and Caveats
- a factory adds indirection.
- for projects with only one or two concrete classes, it can feel like ceremony.
- finally, careless factories become “God objects,” knowing too many specifics; keep them thin.

# Practical Considerations
- Configuration source: decide whether selection is compile-time (subclass registry) or run-time (env vars, JSON, CLI flags).
- Lifecycle: if objects are heavy or scarce, embed pooling or caching inside the factory.
- Dependency Injection (DI): in large systems, treat the factory itself as an injectable dependency to keep tests fully decoupled.


# Real World Example of Factory

Scenario: an e-commerce platform wants to switch between Stripe, PayPal, or a sandbox “dummy” gateway without changing checkout code. At deployment, an env-var PAYMENT_PROVIDER controls the choice.

```python
from __future__ import annotations
import os
import abc
from dataclasses import dataclass

# ---------- Common contract ----------
class PaymentProcessor(abc.ABC):
    """Interface all concrete processors must implement."""
    
    @abc.abstractmethod
    def charge(self, *, amount_cents: int, currency: str, token: str) -> str:
        #  Keyword-Only Arguments: A bare asterisk * in the parameter list signifies that all parameters following it must be specified as keyword arguments when the function is called.
        """Returns a provider-specific charge ID or raises an exception."""
        raise NotImplementedError

# ---------- Concrete implementations ----------
@dataclass
class StripeProcessor(PaymentProcessor):
    api_key: str
    
    def charge(self, *, amount_cents, currency, token) -> str:
        # imagine stripe_sdk.Charge.create(...)
        print(f"[Stripe] Charging {amount_cents/100:.2f} {currency} using token={token}")
        return "stripe_ch_123"


@dataclass
class PayPalProcessor(PaymentProcessor):
    client_id: str
    secret: str
    
    def charge(self, *, amount_cents, currency, token) -> str:
        print(f"[PayPal] Charging {amount_cents/100:.2f} {currency} using token={token}")
        return "paypal_txn_456"


class DummyProcessor(PaymentProcessor):
    """A stub for local development or unit tests."""
    
    def charge(self, *, amount_cents, currency, token) -> str:
        print(f"[Dummy] Pretending to charge {amount_cents/100:.2f} {currency}")
        return "dummy_ok"

# ---------- Factory ----------
class PaymentProcessorFactory:
    """Creates the proper processor based on config or environment."""
    
    _registry = {
        "stripe": StripeProcessor,
        "paypal": PayPalProcessor,
        "dummy": DummyProcessor,
    }
    
    @classmethod
    def from_env(cls) -> PaymentProcessor:
        provider = os.getenv("PAYMENT_PROVIDER", "dummy").lower()
        try:
            ctor = cls._registry[provider]
        except KeyError as exc:
            raise ValueError(f"Unsupported payment provider: {provider}") from exc
        
        # In real life pull secrets from Vault or AWS Secrets Manager
        if provider == "stripe":
            return ctor(api_key=os.getenv("STRIPE_API_KEY"))
        if provider == "paypal":
            return ctor(client_id=os.getenv("PAYPAL_ID"),
                        secret=os.getenv("PAYPAL_SECRET"))
        return ctor()

# ---------- Client code ----------
def checkout(total_cents: int, token: str, currency: str = "USD") -> str:
    processor = PaymentProcessorFactory.from_env()
    charge_id = processor.charge(amount_cents=total_cents,
                                 currency=currency,
                                 token=token)
    return charge_id


if __name__ == "__main__":
    os.environ["PAYMENT_PROVIDER"] = "stripe"
    os.environ["STRIPE_API_KEY"] = "sk_test_abc"
    
    charge_reference = checkout(total_cents=2599, token="tok_test_42")
    print("Charge successful →", charge_reference)

```
