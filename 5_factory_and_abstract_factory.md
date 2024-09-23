# Factory Method Pattern

When to Use:
- Single Product Type: Use the Factory Method when you want to create instances of a single type of product (e.g., a specific class of objects) but want to allow subclasses to decide which specific class to instantiate.
- Decoupling Instantiation: When you want to decouple the code that uses the product from the specific classes that implement the product interface, so that you can easily change the class of objects being created without modifying client code.
- Subclasses Control Creation: When the class cannot anticipate the type of objects it needs to create, and you want subclasses to specify which objects to create.

Example Use Cases:
In a game, you might have a base class Character and subclasses like Warrior and Mage. 
A factory method in a CharacterFactory can create the correct type of character based on game requirements.


# Abstract Factory Pattern

When to Use:
- Family of Related Products: Use the Abstract Factory when you need to create families of related or dependent objects (like a set of widgets that belong to a specific look and feel).
- Interrelated Objects: When you want to ensure that the created objects are compatible with each other, such as ensuring that a button matches the style of a window in a GUI toolkit.
- Multiple Variants: When the product types may vary (like different themes or configurations), and you want to encapsulate the creation logic for these variants in one place.

Example Use Cases:
In a GUI framework, you might have WindowsFactory and MacFactory that create components like Button, TextField, and Checkbox, ensuring that all components are consistent with the operating system style.

## Combined Usage
