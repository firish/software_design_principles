### Callable

| Name                                            | Where it lives     | What it is                                                                                                                        |
| ----------------------------------------------- | ------------------ | --------------------------------------------------------------------------------------------------------------------------------- |
| **`callable()`** (all lower-case, a *function*) | built-in namespace | *Checks* at **runtime** whether an object can be “called” — i.e., `obj()` will not crash.                                         |
| **`typing.Callable`** (capital C, a *class*)    | `typing` module    | A **type hint** you put in annotations to describe “this parameter/variable is some function-like object with a given signature.” |

When you work with python, you may see code like,
```python
from typing import Callable
Handler = Callable[[Event], None]
```
It means “Handler is any callable that takes one Event argument and returns None.”
Static type checkers such as mypy, pyright, pybalance, or your IDE can now verify you wire things up correctly.

```python
from typing import Callable

# Generic higher-order function: it accepts *any* callable
#   that receives `int` and returns `str`
IntToStr = Callable[[int], str]

def repeat_three_times(func: IntToStr) -> None:
    for n in range(3):
        print(func(n))

# Concrete callables that satisfy that signature:
def to_hex(n: int) -> str:           # ordinary function
    return hex(n)

class Prefixer:                      # instance made "callable" via __call__
    def __init__(self, prefix: str):
        self.prefix = prefix
    def __call__(self, n: int) -> str:
        return f"{self.prefix}{n}"

repeat_three_times(to_hex)           # ok
repeat_three_times(Prefixer("#"))    # ok
```

```python
def bad(n: int) -> int:              # returns int, not str
    return n * 2

repeat_three_times(bad)              # mypy/pyright flags a type error
```

`Handler = Callable[[Event], None]` is typical in event busses and message brokers; 
to ensure that handlers being registered to the bus follow the required input and output data type rules.
