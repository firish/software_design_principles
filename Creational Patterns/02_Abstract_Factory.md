# Abstract Factory Pattern

- Note: (*Important*) `Abstract Factory` is also known as `Kit`.

### Definition:
Provide an interface for creating families of related or dependent objects without specifying their concrete classes.

### Understanding what it means?
Imagine you are building a user interface (UI) application. 
You want it to work with different **themes** (or "look-and-feel" styles) like **Motif** or **Presentation Manager**. 
Each theme defines its own version of widgets like buttons, scroll bars, and windows. 
You don’t want to hard-code any particular theme into your application, because you want to be able to switch between these themes easily.

The **Abstract Factory** pattern solves this by creating a **common interface** for producing each kind of widget (like buttons or scroll bars). 
This interface doesn't create the actual widgets—it only defines what kinds of widgets can be made. 
Then, you create **concrete factories** that produce the widgets for each specific theme (Motif, Presentation Manager, etc.).

#### What does this look like?
- There is an **abstract factory** that declares methods for creating widgets (like `createButton()`, `createScrollBar()`, etc.).
- Each **concrete factory** implements the methods and returns specific widgets for that theme (like a Motif button, or a Motif scroll bar).
- The **client** (your application) doesn't know or care about the actual widget classes;
- it only interacts with the factory through the abstract interface.

This way, your application can **switch between different themes** by simply using a different concrete factory (e.g., MotifFactory or PresentationManagerFactory), without changing any of the code that interacts with the widgets.

![Example Image](./images/abstract_factory_example_1.png)


### When to Use Abstract Factory?

Use the **Abstract Factory** pattern when:
1. **You want the system to be independent of how objects are created**. The system shouldn’t need to know how the widgets (or any other objects) are made.
2. **You need to configure the system with one of several families of objects**. For example, in the widget example, you configure the system with either Motif widgets or PM widgets.
3. **You want to enforce consistency**. You want all the widgets in your UI to come from the same family (all Motif widgets or all PM widgets), not a mix.
4. **You want to provide a library of related products**. You can give users access to different types of widgets without showing them how the widgets are implemented.

#### **Key Participants**
- **AbstractFactory (WidgetFactory)**: Declares the operations for creating abstract product objects.
- **ConcreteFactory (MotifWidgetFactory, PMWidgetFactory)**: Implements the operations to create concrete product objects (like the specific theme widgets).
- **AbstractProduct (Button, ScrollBar)**: Declares the interface for each type of product (like buttons or scroll bars).
- **ConcreteProduct (MotifButton, PMScrollBar)**: Defines a product that a concrete factory will create. These implement the abstract product interface.
- **Client**: The application that uses the abstract factory to create product objects. The client interacts only with the factory's interface and doesn’t know about the concrete classes.


#### **Consequences of Using Abstract Factory**
1. **Isolates Concrete Classes**: Clients use the abstract factory’s interface, so they don’t depend on the concrete product classes. This **isolates** the client from specific classes, making it easier to change or update the underlying objects.
2. **Makes Exchanging Product Families Easy**: You can change the look and feel of your application by simply switching the concrete factory. For example, switching from Motif widgets to Presentation Manager widgets is easy—just use a different factory.
3. **Promotes Consistency Among Products**: If product objects in a family are designed to work together, the Abstract Factory pattern ensures that only related objects (like Motif widgets) are used together.
4. **(Important) Difficult to Support New Products**: It’s not easy to add new types of products to an existing Abstract Factory. If you want to add a new widget type (like a new kind of button), you’d have to change the abstract factory interface and all its subclasses.

![Example Image](./images/abstract_factory_example_2.png)

#### Summary
The **Abstract Factory** pattern is all about providing a consistent way to create related objects, such as UI components, without specifying their exact classes. It allows systems to switch between different families of objects (like different themes or styles) easily while maintaining consistency. However, adding new product types can be challenging because you need to modify the factory interface and all its subclasses.


# Applying the Abstract Factory pattern to the Maze Game

In this section, we apply the **Abstract Factory** pattern to create mazes in a more flexible way. 
The `MazeFactory` class is used to create the components of a maze (rooms, walls, doors). 
This factory approach allows us to create different kinds of mazes, such as enchanted mazes or bombed mazes, by simply swapping out the factory that builds the components.

The `MazeFactory` class is responsible for creating the maze components, such as rooms, walls, and doors. 
It allows flexibility by letting programs specify which types of rooms, walls, and doors to construct, rather than hard-coding these details into the maze creation process.

```python
class MazeFactory:
    def make_maze(self):
        return Maze()

    def make_wall(self):
        return Wall()

    def make_room(self, n):
        return Room(n)

    def make_door(self, r1, r2):
        return Door(r1, r2)
```
This MazeFactory class can be used in programs to create different mazes. For example, you might have a program that reads a maze plan from a file and uses MazeFactory to build the maze based on that plan. 

Here’s an updated version of the maze creation function that uses the MazeFactory as a parameter. This allows us to specify which factory to use (standard, enchanted, bombed, etc.):
```python
class MazeGame:
    def create_maze(self, factory):
        a_maze = factory.make_maze()
        r1 = factory.make_room(1)
        r2 = factory.make_room(2)
        a_door = factory.make_door(r1, r2)

        a_maze.add_room(r1)
        a_maze.add_room(r2)

        r1.set_side("North", factory.make_wall())
        r1.set_side("East", a_door)
        r1.set_side("South", factory.make_wall())
        r1.set_side("West", factory.make_wall())

        r2.set_side("North", factory.make_wall())
        r2.set_side("East", factory.make_wall())
        r2.set_side("South", factory.make_wall())
        r2.set_side("West", a_door)

        return a_maze
```
Here, the function takes a MazeFactory object as a parameter and uses it to create the maze components (rooms, walls, doors). The flexibility comes from the fact that you can easily pass in different factories to create different kinds of mazes.


EnchantedMazeFactory:
If we want to create an enchanted maze (a maze with magical components), we can subclass the MazeFactory to create an EnchantedMazeFactory. This factory will override certain methods to return enchanted rooms and doors that require spells to open.
```python
class EnchantedMazeFactory(MazeFactory):

    __magic_spell = "Abra_ka_dabra"

    def make_room(self, n):
        return EnchantedRoom(n, self.cast_spell())

    def make_door(self, r1, r2):
        return DoorNeedingSpell(r1, r2)

    def cast_spell(self):
        return __magic_spell
```
Here, we’ve overridden the make_room and make_door methods to return enchanted versions of the room and door. The cast_spell() method simulates casting a spell that is required to open the magical doors.


BombedMazeFactory:
If we want to create a bombed maze (a maze where rooms can have bombs and walls can be damaged), we can create another subclass, BombedMazeFactory, which overrides methods to create bombed rooms and damaged walls.
```python
class BombedMazeFactory(MazeFactory):
    def make_wall(self):
        return BombedWall()

    def make_room(self, n):
        return RoomWithABomb(n)
```
This factory ensures that the maze will be made up of RoomWithABomb objects and BombedWall objects. These special components will track the damage done when a bomb goes off.


#### Creating Different Mazes
To create a maze with different components, we simply call the create_maze method with the appropriate factory:
```python
# Creating a bombed maze
game = MazeGame()
bombed_factory = BombedMazeFactory()
bombed_maze = game.create_maze(bombed_factory)

# Creating an enchanted maze
enchanted_factory = EnchantedMazeFactory()
enchanted_maze = game.create_maze(enchanted_factory)
```
By passing different factory objects (BombedMazeFactory or EnchantedMazeFactory) to the create_maze method, we can easily create different kinds of mazes without modifying the core logic.


#### Key Insights
1.
MazeFactory is just a collection of factory methods. Each method (make_maze, make_room, make_wall, make_door) is responsible for creating a different part of the maze.
EnchantedMazeFactory and BombedMazeFactory are subclasses of MazeFactory that override specific methods to return different types of components (e.g., enchanted rooms, bombed walls).
This pattern allows us to easily switch between different types of mazes by passing different factories to the maze creation function.

2.
In the original maze creation function, we hard-coded the class names for rooms, walls, and doors. This made it difficult to change the maze components. By using the Abstract Factory pattern, we can remove those hard-coded class names and instead rely on a factory to produce the correct objects. This makes it easy to create new kinds of mazes with different components, such as enchanted mazes or bombed mazes, without rewriting the maze creation logic.

Conclusion
By using the Abstract Factory pattern, we can easily create different kinds of mazes by swapping out the factory used to create the components. This makes the code more flexible and modular, allowing us to create enchanted mazes, bombed mazes, or any other type of maze simply by changing the factory that is passed to the create_maze method.
