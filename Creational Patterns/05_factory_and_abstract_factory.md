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
```python
# Product
class MapSite:
    def enter(self):
        pass

class Room(MapSite):
    def __init__(self, room_number):
        self._sides = [None] * 4
        self._room_number = room_number

    def set_side(self, direction, site):
        self._sides[direction] = site

class Wall(MapSite):
    def enter(self):
        print("You hit a wall!")

class Door(MapSite):
    def __init__(self, room1, room2):
        self._room1 = room1
        self._room2 = room2
        self._is_open = False

class Maze:
    def __init__(self):
        self._rooms = {}

    def add_room(self, room):
        self._rooms[room._room_number] = room

# Abstract Factory
# The Abstract Factory pattern is implemented through the MazeFactory class and its subclasses.
# It allows the creation of families of related objects (Maze, Room, Wall, Door) without specifying their concrete classes.
class MazeFactory:
    def make_maze(self):
        return Maze()

    def make_room(self, number):
        raise NotImplementedError()

    def make_wall(self):
        return Wall()

    def make_door(self, room1, room2):
        return Door(room1, room2)

# Concrete Factory
# Implement the MazeFactory interface to create specific types of maze components.
class StandardMazeFactory(MazeFactory):
    def make_room(self, number):
        return Room(number)

class EnchantedMazeFactory(MazeFactory):
    def make_room(self, number):
        return EnchantedRoom(number)

    def make_door(self, room1, room2):
        return DoorNeedingSpell(room1, room2)

class EnchantedRoom(Room):
    def __init__(self, room_number):
        super().__init__(room_number)
        self.spell = "Magic Spell"

class DoorNeedingSpell(Door):
    pass

# Creator
class MazeGame:
    def create_maze(self, factory):
        maze = factory.make_maze()
        room1 = factory.make_room(1)
        room2 = factory.make_room(2)
        door = factory.make_door(room1, room2)

        maze.add_room(room1)
        maze.add_room(room2)
        room1.set_side(0, factory.make_wall())  # North
        room1.set_side(1, door)                  # East
        room1.set_side(2, factory.make_wall())  # South
        room1.set_side(3, factory.make_wall())  # West

        room2.set_side(0, factory.make_wall())  # North
        room2.set_side(1, factory.make_wall())  # East
        room2.set_side(2, factory.make_wall())  # South
        room2.set_side(3, door)                  # West

        return maze


# Client code
game = MazeGame()

# Using Standard Factory
standard_factory = StandardMazeFactory()
maze1 = game.create_maze(standard_factory)

# Using Enchanted Factory
enchanted_factory = EnchantedMazeFactory()
maze2 = game.create_maze(enchanted_factory)
```


The Abstract Factory pattern is implemented through the `MazeFactory` class and its subclasses. 
It allows the creation of families of related objects (Maze, Room, Wall, Door) without specifying their concrete classes.

Within the Abstract Factory (MazeFactory), the methods used to create maze components (make_maze, make_room, make_wall, make_door) are Factory Methods.
Creator: MazeFactory; Declares factory methods for creating products.
Concrete Creators: StandardMazeFactory, EnchantedMazeFactory; Override the factory methods to return instances of concrete products.
Products: Room, Wall, Door, EnchantedRoom, DoorNeedingSpell

Abstract Factory (MazeFactory) provides an interface for creating families of related objects.
Factory Methods within the abstract factory define how each individual product is created.
By combining both patterns, you can create complex systems where you need to create families of related objects, and also allow subclasses to decide how to instantiate individual products.
