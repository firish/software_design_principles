## Super Method

The `super` method recap:
super() is a built-in function that returns a proxy object, which allows you to refer to the parent class (also known as the superclass). 
This is especially useful in class inheritance hierarchies to call methods from a parent class without explicitly naming it, which makes your code more maintainable and flexible.
```python
# Example usage
class Vehicle:
    def __init__(self, make, model):
        self.make = make
        self.model = model

    def start_engine(self):
        print("Starting the engine.")

class Car(Vehicle):
    def __init__(self, make, model, fuel_type):
        # Call the __init__ method of Vehicle
        super().__init__(make, model)
        self.fuel_type = fuel_type

    def start_engine(self):
        # Extend the start_engine method of Vehicle
        super().start_engine()
        print(f"The car uses {self.fuel_type} fuel.")

class ElectricCar(Car):
    def __init__(self, make, model, battery_capacity):
        # Call the __init__ method of Car
        super().__init__(make, model, fuel_type='electric')
        self.battery_capacity = battery_capacity

    def start_engine(self):
        # Override and extend the start_engine method
        print("Powering electric motor.")
        super().start_engine()
        print(f"Battery capacity is {self.battery_capacity} kWh.")

# Usage
tesla = ElectricCar("Tesla", "Model S", 100)
tesla.start_engine()
```

Notes:
1. If there is a hierarchy, A → B → C → D, a super() on D will reference C or A?
In a linear inheritance chain where:

A is the base class.
B inherits from A.
C inherits from B.
D inherits from C.
When you call super() in class D, it will reference class C, which is the immediate superclass in the MRO after D.
```python
class A:
    def show(self):
        print("Class A")

class B(A):
    def show(self):
        print("Class B")
        super().show()

class C(B):
    def show(self):
        print("Class C")
        super().show()

class D(C):
    def show(self):
        print("Class D")
        super().show()

d = D()
d.show()

# Output:
# Class D
# Class C
# Class B
# Class A
```

2. How does super work with Multi-inheritance?
a class in Python can inherit from multiple parent classes—this is known as multiple inheritance. When you use super() in such a class, it refers to the next class in the Method Resolution Order (MRO), not necessarily a specific parent class. The MRO is a linearization of all classes that are involved in the inheritance hierarchy.
```python
class ParentA:
    def greet(self):
        print("Hello from ParentA")

class ParentB:
    def greet(self):
        print("Hello from ParentB")

class Child(ParentA, ParentB):
    def greet(self):
        super().greet()

child = Child()
child.greet()  # Output: Hello from ParentA
```
You can view the MRO of a class using: 
```python
print(Child.__mro__)
# Output: (<class '__main__.Child'>, <class '__main__.ParentA'>, <class '__main__.ParentB'>, <class 'object'>)
```

3. Can I do super().super()?
No, you cannot do super().super() in Python. The super() function returns a proxy object that delegates method calls to the next class in the MRO. Calling super() again on the result doesn't make sense and is not valid syntax.

What Can You Do Instead?
If you need to call a method from a class further up in the MRO, you can: `Use super() with explicit arguments`
```python
class A:
    def show(self):
        print("Class A")

class B(A):
    def show(self):
        print("Class B")
        super().show()

class C(B):
    def show(self):
        print("Class C")
        super().show()

class D(C):
    def show(self):
        print("Class D")
        super(B, self).show()  # Skip C and call A

d = D()
d.show()

# Output:
# Class D
# Class A
```
