### Factory Method Pattern

The **Factory Method** pattern defines an interface for creating objects, 
but allows subclasses to decide which specific class to instantiate. 
This pattern enables a class to defer the instantiation of objects to its subclasses, making the code more flexible and extensible.



### **Motivation**
In frameworks that deal with multiple object types, 
like an application that manages different document types, we face the challenge of creating these objects without knowing their exact classes in advance. 

For instance, consider a framework with two main abstract classes:
- **Application**: Manages documents.
- **Document**: Represents the documents themselves.

When a user wants to create a new document, 
the `Application` class cannot know in advance what specific `Document` subclass (like `DrawingDocument` or `TextDocument`) to instantiate. 
This is where the Factory Method comes in: it encapsulates the knowledge of which specific `Document` subclass to create.
By defining a factory method (like `CreateDocument`), 
the `Application` class delegates the responsibility of instantiating documents to its subclasses. 
Each subclass implements the factory method to create the appropriate document type without the parent class needing to know the details.

![factory_example_1](./images/factory_1.png)


### **When to Use the Factory Method Pattern**
You should use the **Factory Method** pattern when:
- A class cannot anticipate the type of objects it needs to create.
- You want subclasses to specify which objects to create.
- Classes delegate the responsibility of creating objects to one of several helper subclasses, and you want to keep the knowledge of which helper to use localized.


### **Structure of the Factory Method Pattern**
1. **Product**: This is the base class that defines what a product is. It sets the standard for all products created by the factory method. For example, in our case, `Document` is a product.
2. **ConcreteProduct**: This is a specific implementation of the Product. It defines what a particular product looks like and how it behaves. In our example, `MyDocument` is a `ConcreteProduct` that implements the Document interface.
3. **Creator**: This is an abstract class that declares the factory method (`create_document`). Typically, it defines how to create a product without specifying which exact class of product will be created.
4. **ConcreteCreator**: This is a subclass of `Creator` that overrides the factory method to create and return a specific type of `ConcreteProduct`. In our example, `MyApplication` is a `ConcreteCreator` that creates `MyDocument`.
   
1. The `Creator` class contains the factory method, which is a placeholder for creating products. Subclasses (like `MyApplication`) implement this method to specify exactly what product to create.
2. Clients use the `Creator` class to request products. They don’t need to know the details about which specific class of product is being created. They simply call the factory method.

![factory_example_2](./images/factory_2.png)


### **Consequences of Using the Factory Method Pattern**
1. **Decoupling Code**: Factory methods allow you to eliminate the need to bind application-specific classes into your code. This means the code can work with any subclass of Product through the Product interface.
2. **Subclass Hooks**: Creating objects through a factory method provides a hook for subclasses to extend or customize the object creation process.
3. **Connecting Class Hierarchies**: The factory method connects two parallel class hierarchies (like figures and manipulators), allowing clients to create instances without knowing specific details about the implementations.

![factory_example_3](./images/factory_3.png)

### **Example Implementation**
Here’s a simple example using the Factory Method pattern in Python:
```python
# Product

# Document: This is the abstract base class (Product).
# It defines a method open(), which must be implemented by any subclass. If you try to call open() on Document directly, it raises an error.
class Document:
    def open(self):
        raise NotImplementedError("Subclasses should implement this!")

# DefaultDocument: This is a ConcreteProduct that inherits from Document.
# It provides its own implementation of the open() method, which returns a message indicating that the document is being opened.
class DefaultDocument(Document):
    def __init__(self):
        self.document = "Default Document"

    def read(self):
        "logic for reading document"
        return self.document

    def open(self):
        return self.read()

# MyDocument: This is a another ConcreteProduct that inherits from Document.
# It provides its own implementation of the open() method, which returns a message indicating that the document is being opened.
# This is how it can act as a hook for custom logic; this function opens the document by converting it to a different language
class MyDocument(Document):
    def __init__(self):
        self.document = "Default Document"

    def read(self):
        "logic for reading document"
        return self.document

    def translate(self, text):
        "logic for translating document"
        return translated_text

    def open(self):
        document_content = self.read()
        return self.translate(document_content)


# Application: This is the abstract Creator class.
# It declares the factory method create_document() but doesn’t implement it.
# The open_document() method uses this factory method to create a document and then calls the open() method on that document.
class Application:
    def create_document(self):
        raise NotImplementedError("Subclasses should implement this!")

    def open_document(self):
        doc = self.create_document()
        print(doc.open())

# DefaultApplication: This is the ConcreteCreator. It overrides the create_document() method to create a Default instance.
# Concrete Creator
class DefaultApplication(Application):
    def create_document(self):
        return DefaultDocument()

# Client code
app = DefaultApplication()
app.open_document()
```

This pattern promotes flexibility and makes it easier to add new product types without changing existing code.



## Applying the Factory Method Pattern to Maze Creation Example

In this section, we will refactor the maze creation process to use the **Factory Method** pattern. 
This approach will allow us to avoid hard-coding the specific classes for the maze, rooms, walls, and doors. 
Instead, we'll use factory methods to let subclasses decide which components to create.


### **Defining the MazeGame Class**

We start with the `MazeGame` class, 
which will contain the methods for creating various maze components. The class has factory methods for creating the maze, rooms, walls, and doors.

- A product is made at the factory
- The concrete products define the actual product
- The creator is the factory
- The concrete creators are the factory methods
  
```python
# product
class Maze:
    def add_room(self, room):
        # Code to add a room to the maze
        pass

# concrete product
class Room:
    def __init__(self, number):
        self.number = number

    def set_side(self, direction, wall_or_door):
        # Code to set a side of the room
        pass

# concrete product
class Wall:
    pass

# concrete product
class Door:
    def __init__(self, room1, room2):
        # Code to create a door between two rooms
        pass

# creator
class MazeGame:
    def create_maze(self):
        maze = self.make_maze()  # Uses factory method to create a maze
        room1 = self.make_room(1)  # Uses factory method to create room 1
        room2 = self.make_room(2)  # Uses factory method to create room 2
        door = self.make_door(room1, room2)  # Uses factory method to create a door
        
        maze.add_room(room1)  # Add room 1 to the maze
        maze.add_room(room2)  # Add room 2 to the maze
        room1.set_side("North", self.make_wall())  # Set North side for room 1
        room1.set_side("East", door)  # Set East side for room 1
        room1.set_side("South", self.make_wall())  # Set South side for room 1
        room1.set_side("West", self.make_wall())  # Set West side for room 1
        
        room2.set_side("North", self.make_wall())  # Set North side for room 2
        room2.set_side("East", self.make_wall())  # Set East side for room 2
        room2.set_side("South", self.make_wall())  # Set South side for room 2
        room2.set_side("West", door)  # Set West side for room 2

        return maze  # Return the created maze

    # Concrete Creator
    def make_maze(self):
        return Maze()  # Factory method to create a Maze

    # Concrete Creator
    def make_room(self, number):
        return Room(number)  # Factory method to create a Room

    # Concrete Creator
    def make_wall(self):
        return Wall()  # Factory method to create a Wall

    # Concrete Creator
    def make_door(self, room1, room2):
        return Door(room1, room2)  # Factory method to create a Door
```
Creating a Maze: The create_maze method uses these factory methods to create the maze structure. 

Factory Methods: The methods make_maze, make_room, make_wall, and make_door are factory methods that return instances of the respective components. By using these methods, we avoid hard-coding the specific classes for the maze components.


Now that our base class is set up (product) (here, it is Maze()), 
we can create subclasses (concrete products) that customize the maze components. 
For example, an EnchantedMazeGame (Creator) can override the factory methods (concrete creators) to create enchanted versions of rooms and walls.
```python
# Concrete Product
class EnchantedRoom(Room):
    def __init__(self, number, spell):
        super().__init__(number)
        self.spell = spell  # Additional attribute for enchantment

# Concrete Product
class DoorNeedingSpell(Door):
    def __init__(self, room1, room2):
        super().__init__(room1, room2)
        # Additional code for a door that requires a spell

# Enchanted Maze Game
class EnchantedMazeGame(MazeGame):
    # super lets me use MazeGame's create_maze
    super().__init__()

    # Override the factory methods to create enchanted components
    def make_maze(self):
        return Maze()

    def make_room(self, number):
        return EnchantedRoom(number, self.cast_spell())  # Creates an enchanted room

    def make_door(self, room1, room2):
        return DoorNeedingSpell(room1, room2)  # Creates a door that requires a spell

    def cast_spell(self):
        return "Magic Spell"  # Spell for the enchanted room
```

I can switch between the two as following
```python
# Client code
def create_regular_maze():
    regular_game = MazeGame()  # Create a regular maze game
    maze = regular_game.create_maze()  # Build the regular maze
    return maze

def create_enchanted_maze():
    enchanted_game = EnchantedMazeGame()  # Create an enchanted maze game
    maze = enchanted_game.create_maze()  # Build the enchanted maze
    return maze

# Example usage
regular_maze = create_regular_maze()
enchanted_maze = create_enchanted_maze()

start_game(maze_type) # where I can start the game with regular_maze or enchanted_maze
```
        


