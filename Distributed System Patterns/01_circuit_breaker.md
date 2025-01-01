# Circuit Breaker Pattern

**Circuit Breaker** is not part of the original Gang of Four (GoF) design patterns. 
It's a **resilience** pattern often used in **distributed systems** or **microservices**. 

The goal is to **prevent** an application from repeatedly trying to execute an operation that's likely to fail—like calling an **unreachable** or **unresponsive** service. 

This pattern is **inspired** by an electrical circuit breaker:  
- **Closed** = Everything is working normally, calls pass through.  
- **Open** = Calls fail immediately (the breaker “trips”), giving the system time to recover or fail fast without wasting resources.  
- **Half-Open** = A trial state, letting a limited number of calls through to see if the service has recovered.


## Why Use the Circuit Breaker Pattern?

1. **Fail Fast**: If a service is down, return an error immediately instead of waiting for repeated timeouts.  
2. **Prevent Resource Exhaustion**: Stop saturating resources (threads, CPU, network) with calls destined to fail.  
3. **Enable Recovery**: After a period, allow small test calls to check if the service is back online.


## A Real-World Example: Payment Gateway in an E-Commerce System

Let's imagine you run an **e-commerce** site. 
When a user **checks out**, your application calls an **external payment gateway** to process credit card transactions. 
If this external gateway becomes **unavailable** or **very slow**, we don’t want to keep hammering it with new requests. 
Instead, we want to trip a **Circuit Breaker** after repeated failures.

1. **PaymentService**: The local service that orchestrates payments.
2. **ExternalPaymentGateway**: A hypothetical external service that can sometimes fail or time out.
3. **CircuitBreaker**: Maintains the state (Closed, Open, Half-Open) and decides what to do when a new request comes in.



## Circuit Breaker States

- **Closed**:  
  - All calls pass through to the external service.  
  - If calls keep succeeding, remain closed.  
  - If failures exceed a threshold, switch to **Open**.

- **Open**:  
  - Calls fail immediately, returning an error (or fallback).  
  - A timer starts. After some “sleep window,” transition to **Half-Open**.

- **Half-Open**:  
  - Allow a limited number of test calls.  
  - If a test call succeeds, the breaker moves to **Closed**.  
  - If a test call fails, it goes back to **Open** again.


## Code Example

### 1. CircuitBreaker Class

We’ll define a simple `CircuitBreaker` that has the following configuration:

- **failure_threshold**: Number of consecutive failures before opening the circuit.  
- **recovery_timeout**: How long the circuit stays open before attempting a half-open test.  
- **max_half_open_calls**: How many requests are allowed in half-open state to test if the service has recovered.

```python
import time
import functools

class CircuitBreakerState:
    CLOSED = "CLOSED"
    OPEN = "OPEN"
    HALF_OPEN = "HALF_OPEN"

class CircuitBreaker:
    def __init__(self, failure_threshold=3, recovery_timeout=5, max_half_open_calls=1):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.max_half_open_calls = max_half_open_calls
        # standard is 1, but it can be optimized increasing the count and introducing a open_call_success_threshold
        
        self.state = CircuitBreakerState.CLOSED
        self.failure_count = 0
        self.last_failure_time = None
        self.half_open_calls = 0

    def is_closed(self):
        return self.state == CircuitBreakerState.CLOSED

    def is_open(self):
        return self.state == CircuitBreakerState.OPEN

    def is_half_open(self):
        return self.state == CircuitBreakerState.HALF_OPEN

    def _move_to_open(self):
        self.state = CircuitBreakerState.OPEN
        self.last_failure_time = time.time()
        print("[CircuitBreaker] State changed to OPEN. Will refuse calls until after recovery timeout.")

    def _move_to_half_open(self):
        self.state = CircuitBreakerState.HALF_OPEN
        self.half_open_calls = 0
        print("[CircuitBreaker] State changed to HALF_OPEN. Testing service with limited calls.")

    def _move_to_closed(self):
        self.state = CircuitBreakerState.CLOSED
        self.failure_count = 0
        self.last_failure_time = None
        print("[CircuitBreaker] State changed to CLOSED. Service considered healthy.")

    def record_success(self):
        if self.is_half_open():
            # If a call is successful in half-open, we can move to closed
            self._move_to_closed()
        else:
            # If closed, just reset failure count
            self.failure_count = 0

    def record_failure(self):
        self.failure_count += 1
        if self.is_closed() and self.failure_count >= self.failure_threshold:
            self._move_to_open()
        elif self.is_half_open():
            # A failure in half-open means we go straight back to open
            self._move_to_open()

    def allow_request(self):
        """
        Decide if the request is allowed based on the current circuit state.
        """
        if self.is_open():
            # Check if recovery timeout has passed
            elapsed = time.time() - self.last_failure_time
            if elapsed > self.recovery_timeout:
                self._move_to_half_open()
            else:
                # Still open, fail fast
                return False

        if self.is_half_open():
            # Allow limited calls
            if self.half_open_calls < self.max_half_open_calls:
                self.half_open_calls += 1
                return True
            else:
                # If we exceed half-open calls, refuse
                return False

        # If CLOSED, we allow the request
        return True
```

External Payment Gateway (Stub)
Assume that Amazon uses the same gateway API and during Thanksgiving, their service is overloaded, and can fail due to the traffic.
```python
import random

class ExternalPaymentGateway:
    @staticmethod
    def process_payment(amount):
        """
        Simulate a payment call that can fail or succeed randomly.
        We'll say there's a 10% chance of failure for demonstration.
        """
        fail_chance = 0.10
        if random.random() < fail_chance:
            # Simulate failure
            raise ConnectionError("Payment gateway is unreachable or responded with an error.")
        # Simulate success
        return "PAYMENT_SUCCESS"
```

PaymentService with CircuitBreaker
We wrap calls to ExternalPaymentGateway.process_payment inside a circuit breaker logic.
```python
class PaymentService:
    def __init__(self, circuit_breaker: CircuitBreaker):
        self.circuit_breaker = circuit_breaker

    def pay(self, amount):
        # Check if we're allowed to call the external service
        if not self.circuit_breaker.allow_request():
            print("[PaymentService] Circuit is OPEN or HALF-OPEN limit reached. Failing fast.")
            return "PAYMENT_FAILED_FAST"

        try:
            result = ExternalPaymentGateway.process_payment(amount)
            print(f"[PaymentService] Payment processed successfully, amount={amount}")
            self.circuit_breaker.record_success()
            return result
        except ConnectionError as e:
            print(f"[PaymentService] Payment failed: {e}")
            self.circuit_breaker.record_failure()
            return "PAYMENT_FAILED"
```

Running a mock test with circuit breaker
```python
def main():
    # Create a circuit breaker with:
    # - 3 consecutive failures to open the circuit
    # - 5 seconds recovery timeout
    # - 1 test call in half-open
    circuit_breaker = CircuitBreaker(failure_threshold=3, recovery_timeout=5, max_half_open_calls=1)
    payment_service = PaymentService(circuit_breaker)

    # Simulate multiple payment attempts
    for i in range(10):
        amount = 100 + i * 10
        print(f"\nAttempting payment of ${amount} ...")
        result = payment_service.pay(amount)
        print(f"Payment result: {result}")

        # Sleep 1 second between calls to see circuit breaker transitions
        time.sleep(1)

if __name__ == "__main__":
    main()
```

```text
Attempting payment of $100 ...
[PaymentService] Payment processed successfully, amount=100
[CircuitBreaker] State changed to CLOSED. Service considered healthy.
Payment result: PAYMENT_SUCCESS

Attempting payment of $110 ...
[PaymentService] Payment failed: Payment gateway ...
[CircuitBreaker] State changed to OPEN. Will refuse calls until after recovery timeout.
Payment result: PAYMENT_FAILED
...

Attempting payment of $120 ...
[PaymentService] Circuit is OPEN ... Failing fast.
Payment result: PAYMENT_FAILED_FAST
...

... after 5 seconds ...
[CircuitBreaker] State changed to HALF_OPEN. Testing service ...
[PaymentService] Payment processed successfully ...
[CircuitBreaker] State changed to CLOSED. ...
Payment result: PAYMENT_SUCCESS
```
