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
