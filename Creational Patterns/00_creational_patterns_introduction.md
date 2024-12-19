# Creational Design Patterns

Creational design patterns are all about how objects are created in a system. 
The idea is to make a system flexible and independent of the exact process used to create objects. 
By using these patterns, you can make your system adaptable to future changes without needing to modify the parts that rely on those objects.


### What are Creational Patterns?
- These patterns focus on the process of creating objects.
- They allow systems to decouple the way objects are created from the logic that uses those objects.
- Instead of directly instantiating classes (which can be rigid and hard to change later), creational patterns abstract this process, giving you more control and flexibility.


Class Creational Patterns: 
Use inheritance to determine which class to instantiate. 
This means that the type of object you create is determined by which class you are working with.

Object Creational Patterns: 
Use composition to delegate the instantiation to another object. 
So instead of hardcoding object creation, you let other objects handle it.

As software systems grow more complex, object composition becomes more important than class inheritance. 
Rather than relying on inheritance to create new behaviors, you combine objects in different ways to produce new functionality.


### Key Themes in Creational Patterns
- Encapsulating knowledge of which concrete classes are used: These patterns hide the details of which specific classes are being instantiated. All that matters is that the objects conform to the interfaces you’ve defined.

- Hiding how and when the objects are created: The system knows that it needs certain objects, but it doesn’t know or care about the exact details of their creation. This gives you a lot of flexibility—both in terms of which objects are created and how they are created (whether at compile-time or dynamically at runtime).

- Sometimes, creational patterns can be used in competition with each other. For example, you might find that both the Prototype pattern and Abstract Factory pattern could be used to solve the same problem, and you'll need to choose between them. At other times, the patterns are complementary and can work together. For example, the Builder pattern can use other creational patterns to specify which components it will build. 

