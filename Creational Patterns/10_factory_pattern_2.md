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
