
# Singleton Pattern</br>

**Singleton** is one of the classic **creational** design patterns.
It ensures that a class has **only one instance** throughout the life of the application,
and it provides a global access point to this instance.</br>


## Why Use the Singleton Pattern?</br>
1. **Global Access**: You need a single, shared resource accessible from different parts of your application (e.g., a single database connection).  
2. **Controlled Instantiation**: You want to prevent multiple instances from being created (e.g., to avoid conflicts or high resource usage).  
3. **Centralized Resource Management**: Some services or managers (like configuration or logging) are typically limited to one instance, ensuring consistency and coordinated behavior.</br>


## Classic Real-World Example: **Application Logger**</br>
**Scenario**: In many applications, you have a **Logger** class that writes logs to a file or console.
You want to make sure all parts of your system write logs to the **same** destination in a **thread-safe** manner, without creating multiple log objects that might conflict or duplicate output.
This is a great fit for the Singleton pattern.

### Key Points</br>
- The Logger instance is **unique** (only one in the entire application).  
- Everyone uses the **same** instance to log messages, preventing confusion about which log file or output stream is being written to.


## Singleton in Python (Multiple Approaches)</br>
Below are **two** different approaches to implement the Singleton in Python.

### 1. **Using `__new__` Method**</br>

```python
class LoggerSingleton:
    _instance = None  # Class variable to store the single instance

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            print("[LoggerSingleton] Creating a new instance.")
            cls._instance = super(LoggerSingleton, cls).__new__(cls)
            # Initialize any additional attributes here
            cls._instance.log_file = "app.log"
        else:
            print("[LoggerSingleton] Using existing instance.")
        return cls._instance

    def write_log(self, message):
        # In a real system, write to self.log_file
        print(f"[{self.log_file}] {message}")
```

**Explanation**:</br>
- We keep a class-level `_instance` that stores the **single** instance.  
- In `__new__`, if `_instance` is `None`, we create a new instance. Otherwise, we return the existing instance.</br>

**Usage**:</br>
```python
def main_singleton():
    logger1 = LoggerSingleton()
    logger1.write_log("Logger1: Application started.")
    logger2 = LoggerSingleton()
    logger2.write_log("Logger2: Logging another message.")

    print("Are logger1 and logger2 the same?", logger1 is logger2)

if __name__ == "__main__":
    main_singleton()
```

**Expected Output**:</br>
```
[LoggerSingleton] Creating a new instance.
[app.log] Logger1: Application started.
[LoggerSingleton] Using existing instance.
[app.log] Logger2: Logging another message.
Are logger1 and logger2 the same? True
```

---

### 2. **Using a Decorator**</br>
We can also create a **decorator** that transforms any class into a Singleton:

```python
def singleton(cls):
    instances = {}

    def wrapper(*args, **kwargs):
        if cls not in instances:
            print(f"[singleton decorator] Creating a new instance of {cls.__name__}.")
            instances[cls] = cls(*args, **kwargs)
        else:
            print(f"[singleton decorator] Using existing instance of {cls.__name__}.")
        return instances[cls]
    return wrapper

@singleton
class DecoratedLogger:
    def __init__(self):
        self.log_file = "decorated.log"

    def write_log(self, message):
        print(f"[{self.log_file}] {message}")

def main_decorator():
    logger1 = DecoratedLogger()
    logger1.write_log("Logger1: Using decorator singleton.")
    logger2 = DecoratedLogger()
    logger2.write_log("Logger2: Another log message.")

    print("Are logger1 and logger2 the same?", logger1 is logger2)

if __name__ == "__main__":
    main_decorator()
```

**Explanation**:</br>
- The `singleton` decorator keeps a dictionary `instances`.  
- Each time the decorated class is called, we check if an instance already exists. If not, create it; otherwise, reuse the existing instance.</br>

**Expected Output**:</br>
```
[singleton decorator] Creating a new instance of DecoratedLogger.
[decorated.log] Logger1: Using decorator singleton.
[singleton decorator] Using existing instance of DecoratedLogger.
[decorated.log] Logger2: Another log message.
Are logger1 and logger2 the same? True
```

---

## Pros and Cons of the Singleton Pattern</br>

**Pros**:</br>
1. **Global Access**: Simple way to provide a single, shared resource or manager.  
2. **Reduced Name-Space Pollution**: Instead of global variables, you have a single class controlling instantiation.</br>

**Cons**:</br>
1. **Testing Challenges**: Singletons can hold global state that’s hard to reset or mock in tests.  
2. **Potential Hidden Dependencies**: If many parts of the code rely on the singleton, it can create subtle coupling.  
3. **Limited Scalability**: For some scenarios, having only one instance might become a bottleneck (like if you want multiple loggers for parallel streams).</br>
```
