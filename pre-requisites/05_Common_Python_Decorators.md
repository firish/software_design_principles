In Python, the leading @ simply means “take the thing that follows and pass it through a decorator before Python stores it.” 
A decorator is just a function that receives another function or class, adds or changes behavior, and hands back a new (or modified) object.


# Dataclass

@dataclass — an auto-boilerplate class decorator
The standard-library function dataclasses.dataclass() looks at the attributes you declare and automatically writes methods that every “plain data holder” usually needs:

__init__ with typed arguments for each field
__repr__ that prints something readable
comparisons like __eq__ (and <, > if you ask for ordering)

```Python
from dataclasses import dataclass

# Without @dataclass you’d hand-code __init__, __repr__, etc.
@dataclass
class Point:
    x: float
    y: float
    label: str = "origin"      # default value

p = Point(3, 4, "A")
print(p)                 # Point(x=3, y=4, label='A')
print(p == Point(3, 4))  # True – default label matches "origin"
```

Behind the scenes, python does `Point = dataclass(Point)`


# Classmethod

@classmethod — methods that belong to the class itself
When you define a normal method, the first parameter is self, the instance that called it. 
If you mark the method with @classmethod, Python passes the class (cls) instead. 
That lets you write alternative constructors or utilities that should work the same for every subclass.

```Python
from dataclasses import dataclass
from datetime import date

@dataclass
class Employee:
    name: str
    start: date
    hourly_rate: float

    # an alternative constructor that turns weekly salary into hourly rate
    @classmethod
    def from_weekly_salary(cls, name: str, weekly_salary: float, start=None):
        if start is None:
            start = date.today()
        hourly = weekly_salary / 40            # assume 40-hour week
        return cls(name=name, start=start, hourly_rate=hourly)

emp = Employee.from_weekly_salary("Jamal", 1200)
print(emp)        # Employee(name='Jamal', start=datetime.date(...), hourly_rate=30.0)
```

If PartTimeEmployee later subclasses Employee, 
calling PartTimeEmployee.from_weekly_salary(...) will return a PartTimeEmployee, 
because cls refers to the class that received the call, not the one in which the method was first defined.
