
# Understanding `abc` (Abstract Base Classes)

Abstract Base Classes module provides a framework for defining **abstract classes**. 
An **abstract class** is intended to be a blueprint or interface for other classes to inherit from, rather than a concrete implementation that you can instantiate directly. 



## 1. Why Do We Need It?

- **Defines Interfaces or Contracts**: By marking certain methods as abstract (`@abstractmethod`), you signal that **subclasses** must provide their own implementations of these methods.
- **Encourages Consistency**: If multiple subclasses inherit from the same abstract base class, they **must** fulfill the required interface, ensuring consistent behavior across different implementations.

In Python, </br>
Without `abc`, you can still create a Python “interface” by raising `NotImplementedError` in a base class. 
However, `abc` formalizes and enforces this pattern, making your intent clearer.


## 2. When is it Used?

1. **Interface-Like Behavior**: When you want to specify methods that **must** be implemented by all subclasses (like a contract).
2. **Framework or Library Development**: When providing a library where you expect users to subclass your abstract class to plug in custom logic.
3. **Polymorphism**: When you rely on different subclasses implementing the same methods differently, but you want to ensure a consistent set of methods exist.


## 3. How is it Used?

1. **Import the `abc` module**: 
   ```python
   import abc
   ```
2. **Create an Abstract Base Class**: 
   - Inherit from `abc.ABC`, which marks the class as abstract.
   - Decorate abstract methods with `@abc.abstractmethod`.

3. **Implement Subclasses**:
   - Subclasses inherit from the abstract base.
   - They **must** implement all abstract methods, or they themselves become abstract and cannot be instantiated.


### **Code Example**

```python
import abc

class Shape(abc.ABC):
    """
    An abstract base class representing a generic shape.
    """

    @abc.abstractmethod
    def area(self):
        """
        Calculate the area of the shape.
        Must be implemented by subclasses.
        """
        pass

    @abc.abstractmethod
    def perimeter(self):
        """
        Calculate the perimeter of the shape.
        Must be implemented by subclasses.
        """
        pass


class Circle(Shape):
    def __init__(self, radius):
        self.radius = radius

    def area(self):
        return 3.14159 * (self.radius ** 2)

    def perimeter(self):
        return 2 * 3.14159 * self.radius


class Rectangle(Shape):
    def __init__(self, width, height):
        self.width = width
        self.height = height

    def area(self):
        return self.width * self.height

    def perimeter(self):
        return 2 * (self.width + self.height)


def main():
    # circle = Shape()              # This will fail because Shape is abstract
    circle = Circle(radius=5)
    rectangle = Rectangle(width=4, height=3)

    print("Circle area:", circle.area())
    print("Circle perimeter:", circle.perimeter())

    print("Rectangle area:", rectangle.area())
    print("Rectangle perimeter:", rectangle.perimeter())


if __name__ == "__main__":
    main()
```

**Explanation**:
- `Shape` is marked with `abc.ABC`, meaning Python sees it as an abstract class.
- `@abc.abstractmethod` enforces the rule that **any** subclass of `Shape` **must** provide its own `area()` and `perimeter()` methods.
- `Circle` and `Rectangle` are concrete subclasses implementing those methods. They **can** be instantiated.


## 4. Advantages

1. **Clear Intent**: Using `abc.ABC` and `@abstractmethod` explicitly states which methods **must** be implemented, making your code self-documenting.
2. **Compile-Time Check**: Attempting to instantiate a subclass that **hasn’t** implemented all abstract methods results in an error, catching mistakes early.
3. **Polymorphic Consistency**: Ensures that all subclasses fulfill the same interface, which is vital for code that depends on polymorphism.

---

## 5. Disadvantages

1. **Extra Boilerplate**: Writing abstract methods and decorators can add a bit more code compared to Python’s dynamic approach of “duck typing.”
2. **Less Flexibility**: In Python, you often rely on dynamic typing or `NotImplementedError`. Enforcing strict interfaces might feel more like statically typed languages. 
3. **Subclass Inheritance**: If a subclass fails to implement **any** abstract method, you’ll get errors at runtime when you try to instantiate it (this may be an advantage or disadvantage depending on perspective).


## 6. Best Practices

- **Use `abc`**: When you have a clear “interface” in mind that multiple subclasses must follow exactly.
- **Limit the Number of Abstract Methods**: Keep your abstract base classes small and focused, so subclasses can remain flexible.
- **Don’t Force**: If your design is simpler or your codebase is small, dynamic approach with simple base classes or docstrings might suffice.


