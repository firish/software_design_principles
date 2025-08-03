# O — Open/Closed Principle (OCP)

## Core Concept & Definition

**The Principle**: *"Software entities should be open for extension, but closed for modification."*

OCP asks us to design modules that can gain new behaviour without changing their existing source code. Instead of editing a class every time requirements shift, we attach new functionality through inheritance, composition, or higher-order functions. The original, battle-tested code stays stable; fresh features arrive in small, focused extensions.

## Historical Context

Bertrand Meyer introduced the idea in *Object-Oriented Software Construction* (1988). Robert C. Martin ("Uncle Bob") later folded it into the SOLID acronym and popularized it in his Clean Code talks and articles.

## The Problem: Why OCP Matters

### What goes wrong without it
- Code that is "closed" (unchangeable) can't evolve
- Code that is too "open" (constantly edited) becomes a land-mine of side-effects
- If every new business rule forces you to edit the same if/elif tower or "God class," you soon fear touching it
- Velocity stalls, bugs sneak in, and parallel teams trip over each other

### Pain points in practice
- Every new feature forces you to edit and re-deploy existing code
- Methods mix multiple concerns (HTTP details, business logic, validation)
- Parallel teams cannot add features independently; merge conflicts loom
- Regression risk increases with every modification

## Simple Example: The Problem in Action

### Violation of OCP
```python
class TaxCalculator:
    def net_price(self, country, gross):
        if country == "US":
            return gross * 0.93          # 7% tax
        elif country == "DE":
            return gross * 0.81          # 19% VAT
        elif country == "IN":
            return gross * 0.88          # 12% GST
        # more elifs every quarter...
```

**Problems**: Every time finance adds a jurisdiction, you must reopen `TaxCalculator` and change shipped code—risking old math and re-deployment headaches.

### Simple Fix: Strategy Pattern
```python
# strategy.py
from abc import ABC, abstractmethod

class TaxStrategy(ABC):
    @abstractmethod
    def net(self, gross: float) -> float: ...

class USTax(TaxStrategy):
    def net(self, gross): return gross * 0.93

class DETax(TaxStrategy):
    def net(self, gross): return gross * 0.81

class INTax(TaxStrategy):
    def net(self, gross): return gross * 0.88

# calculator.py
class TaxCalculator:
    def __init__(self, strategy: TaxStrategy):
        self._strategy = strategy

    def net_price(self, gross):
        return self._strategy.net(gross)

# client code
calc = TaxCalculator(DETax())
print(calc.net_price(100))   # €81
```

**How OCP helped**:
- Adding Canada tomorrow means adding `CATax`—no edits to `TaxCalculator`
- Existing tests for US/DE/IN stay untouched; regression risk is isolated
- Teams can deliver new strategies in parallel; merging is painless

## Real-World Example: E-commerce Payment System

### The Growing Problem
```python
# checkout.py - VIOLATES OCP
import requests

class CheckoutService:
    def charge(self, provider: str, amount: float, card_data: dict):
        """Return a dict with {id, status, fee} or raise on failure."""
        if provider == "stripe":
            r = requests.post("https://api.stripe.com/v1/charges",
                              json={"amount": int(amount * 100), **card_data})
            fee = 0.029 * amount + 0.30

        elif provider == "paypal":
            r = requests.post("https://api.paypal.com/v2/checkout/orders",
                              json={"purchase_units": [{"amount": amount}]})
            fee = 0.034 * amount + 0.49

        elif provider == "apple_pay":
            r = requests.post("https://apple-pay-gw.com/pay",
                              json={"price": amount, **card_data})
            fee = 0.03 * amount                # promo

        else:
            raise ValueError("Unsupported provider")

        r.raise_for_status()
        return {"id": r.json()["id"], "status": r.json()["status"], "fee": fee}
```

## Extension Techniques

### 1. Inheritance + Polymorphism
```python
# gateways.py
from abc import ABC, abstractmethod
import requests, decimal

class PaymentGateway(ABC):
    @abstractmethod
    def charge(self, amount: decimal.Decimal, card_data: dict) -> dict: ...

class StripeGateway(PaymentGateway):
    FEE_RATE, FIXED = decimal.Decimal("0.029"), decimal.Decimal("0.30")
    
    def charge(self, amount, card_data):
        cents = int(amount * 100)
        r = requests.post("https://api.stripe.com/v1/charges",
                          json={"amount": cents, **card_data})
        fee = amount * self.FEE_RATE + self.FIXED
        return {"id": r.json()["id"], "status": r.json()["status"], "fee": fee}

class PayPalGateway(PaymentGateway):
    FEE_RATE, FIXED = decimal.Decimal("0.034"), decimal.Decimal("0.49")
    
    def charge(self, amount, card_data):
        r = requests.post("https://api.paypal.com/v2/checkout/orders",
                          json={"purchase_units": [{"amount": str(amount)}]})
        fee = amount * self.FEE_RATE + self.FIXED
        return {"id": r.json()["id"], "status": r.json()["status"], "fee": fee}

# checkout.py
class CheckoutService:
    def __init__(self, gateway: PaymentGateway):
        self._gw = gateway                 # injected

    def charge(self, amount, card_data):
        return self._gw.charge(amount, card_data)
```

**Extending**: Drop a new file `afterpay.py` with `class AfterpayGateway(PaymentGateway): ...`. No edits to `CheckoutService`.

### 2. Composition via Registry Pattern
```python
# registry.py
_registry = {}

def gateway(name):
    def decorator(cls):
        _registry[name] = cls()
        return cls
    return decorator

def get(name):
    return _registry[name]

# gateways.py
@gateway("stripe")
class StripeGateway(PaymentGateway):
    # ... implementation

# checkout.py
from registry import get as get_gateway

class CheckoutService:
    def charge(self, provider_name, amount, card_data):
        gw = get_gateway(provider_name)
        return gw.charge(amount, card_data)
```

**Extending**: A plugin team ships `mollie_gateway.py`, marks it with `@gateway("mollie")`, and `CheckoutService` automatically supports it—zero edits.

### 3. Higher-Order Functions / Callbacks
```python
# types.py
from typing import Protocol, Callable, Dict
ChargeFn = Callable[[float, dict], Dict]   # (amount, card_data) → result

# checkout.py
class CheckoutService:
    def __init__(self, charge_fn: ChargeFn):
        self._charge_fn = charge_fn

    def charge(self, amount, card_data):
        return self._charge_fn(amount, card_data)

# composition root
from gateways import StripeGateway
service = CheckoutService(StripeGateway().charge)
```

**Extending**: Any future payment stack (serverless Lambda, mock for tests, A/B experiment proxy) only needs to supply a compatible function—classic functional OCP.

## Common Application Areas

- **Calculators** that vary by country, tax rule, or currency
- **Payment, notification, or logging pipelines** that sprout new channels
- **Game engines** where new enemy types or power-ups appear each sprint
- **Data-import tools** that must parse emergent file formats

## Advanced Extension Mechanisms

| Mechanism | When handy | Extension payload |
|-----------|------------|-------------------|
| **Dynamic import plugins** | Large apps (Django, Airflow, Home-Assistant) that auto-discover entrypoints | Pip-install a package; add entry_points in setup.py |
| **Configuration + DI** | Enterprise Spring/Guice / FastAPI apps | Edit YAML/env var to swap bean |
| **Feature flags** | Gradual rollout or beta gating | Toggle flag → new strategy injected |

## Key Benefits

- **Reduces regression risk**: Existing, validated code stays unchanged
- **Enables parallel development**: Teams can work on extensions independently  
- **Simplifies testing**: New features are isolated; old tests remain valid
- **Improves maintainability**: Extensions are focused and easy to reason about
- **Supports gradual rollout**: New behavior can be added incrementally

All these techniques keep existing, validated code **closed**, while letting behavior grow outward in easy-to-reason-about slices—fulfilling Meyer's Open/Closed dream.
