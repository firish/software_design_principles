## Metaclasses

A metaclass in Python is a class of a class that defines how a class behaves. 
Just like an object is an instance of a class, a class is an instance of a metaclass. 
Metaclasses allow you to modify or customize class creation.

### Use
- Class Registration: Automatically register classes in a registry upon creation.
- Enforcing Class Constraints: Ensure that classes have certain attributes or methods.
- Automatic Attribute Creation: Modify or add class attributes during creation.

### Real-World Example
The Singleton pattern ensures that a class has only one instance and provides a global point of access to it. 
This is useful for managing shared resources like database connections, configuration settings, or logging mechanisms. 
Using a metaclass to implement the Singleton pattern centralizes the instance control logic, making it easier to manage and extend.

```python
class SingletonMeta(type):
    """
    Metaclass that creates a Singleton base type when called.
    """
    _instances = {}

    # __call__ Method: Overrides the default instantiation behavior.
    # It checks if an instance of the class already exists in _instances. If it does, it returns that instance; otherwise, it creates a new one.
    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            # Create a new instance and store it in the _instances dictionary
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

# Example usage
class Logger(metaclass=SingletonMeta):
    def __init__(self):
        self.log_file = 'app.log'

    def log(self, message):
        # Simulate writing to a log file
        print(f"Logging message: {message}")

# Test the Singleton behavior
logger1 = Logger()
logger2 = Logger()

print(f"logger1 is logger2: {logger1 is logger2}")  # Should print True
```

Another example:
Enforcing Interface Compliance Using a Metaclass.
In large-scale applications, it's crucial to ensure that classes adhere to certain interfaces or contracts, especially when multiple developers are involved. 
A metaclass can enforce that any subclass implements specific methods or properties, similar to interfaces in languages like Java or C#.

```python
class InterfaceMeta(type):
    """
    Metaclass that enforces implementation of specified methods.
    """
    def __new__(mcs, name, bases, namespace):
        # Only enforce checks on non-base classes
        if 'Interface' not in [base.__name__ for base in bases]:
            required_methods = namespace.get('_required_methods', [])
            for method in required_methods:
                if method not in namespace:
                    raise TypeError(f"Class '{name}' must implement method '{method}'")
        return super().__new__(mcs, name, bases, namespace)

# Base Interface Class
class Interface(metaclass=InterfaceMeta):
    _required_methods = []

# Define an Interface with Required Methods
class DataStorageInterface(Interface):
    _required_methods = ['save', 'load']

# Correct Implementation of the Interface
class FileStorage(DataStorageInterface):
    def save(self, data):
        print("Saving data to a file.")

    def load(self):
        print("Loading data from a file.")

# Incorrect Implementation (Will Raise TypeError)
try:
    class InMemoryStorage(DataStorageInterface):
        def save(self, data):
            print("Saving data in memory.")
except TypeError as e:
    print(e)
```

Example 3:
Automatic Class Registration with a Metaclass.
In plugin architectures or when building extensible applications, 
it's helpful to automatically register all subclasses of a base class. 
A metaclass can be used to automatically add each new subclass to a registry, which can then be used to discover and utilize these classes dynamically.

```python
class PluginMeta(type):
    """
    Metaclass that automatically registers subclasses in a registry.
    """
    registry = {}

    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        if not namespace.get('abstract', False):
            PluginMeta.registry[name] = cls
        return cls

# Base Plugin Class
class BasePlugin(metaclass=PluginMeta):
    abstract = True  # This class should not be registered

    def run(self):
        raise NotImplementedError("Subclasses must implement 'run' method.")

# Plugin Implementations
class PluginA(BasePlugin):
    def run(self):
        print("Running PluginA")

class PluginB(BasePlugin):
    def run(self):
        print("Running PluginB")

# Usage
print("Registered Plugins:", PluginMeta.registry)
# Registered Plugins: {'PluginA': <class '__main__.PluginA'>, 'PluginB': <class '__main__.PluginB'>}

```
