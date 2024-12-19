# Creational Pattern - Use Case Set-up

Lets imagine we are building a video game to solve a maze.


The game might involve:
- A player exploring rooms in the maze.
- Some rooms might have doors that lead to other rooms, while others might be blocked by walls.
- In this case, the maze is made up of objects like Room, Wall, and Door. These are the components of the maze.

![Maze Schema](./images/creational_pattern_maze_schema.png)


- A Room knows its neighboring rooms. 
- A Door can connect two rooms, allowing the player to move between them.
- A Wall blocks the way.
- Each room in the maze has four sides: North, South, East, and West.

### Classes

Notes:
1. A single leading underscore in a variable or method name (e.g., _variable) is a convention to indicate that the variable or method is intended for internal use only. It's a hint to programmers that it is part of the implementation and not part of the public API.
Python does not enforce any restrictions based on this convention. These variables and methods can still be accessed from outside the class.

2.  A double-leading underscore (e.g., __variable) triggers name mangling, which alters the variable name to avoid accidental access or modification in subclasses. The variable name is internally changed to _ClassName__variable, which makes it more difficult to access from outside the class.

- The `MapSite` class is an abstract class that represents all the components of a maze (rooms, walls, doors).
  - `Enter()`: Each component has an `Enter` method, which defines what happens when the player interacts with it.
  - For example, entering a room changes the player’s location, but entering a door checks if the door is open or closed.
```python
class MapSite:
    def enter(self):
        pass
```

- Room: A concrete subclass of MapSite. A room knows its neighbors (other rooms, walls, or doors) and has a room number to identify it.
```python
class Room(MapSite):
    def __init__(self, room_number):
        self._sides = [None] * 4  # Each room has 4 sides
        self._room_number = room_number

    def get_side(self, direction):
        return self._sides[direction]

    def set_side(self, direction, site):
        self._sides[direction] = site

    def enter(self):
        print(f"Entering room {self._room_number}")
```

- Wall: Represents a wall in the maze. It blocks the player from moving.
```python
class Wall(MapSite):
    def enter(self):
        print("You hit a wall!")
```

- Door: Connects two rooms. A door can be open or closed.
```python
class Door(MapSite):
    def __init__(self, room1=None, room2=None):
        self._room1 = room1
        self._room2 = room2
        self._is_open = False

    def enter(self):
        if self._is_open:
            print("You pass through the door.")
        else:
            print("The door is closed, you hurt your nose!")

    def other_side_from(self, room):
        if room == self._room1:
            return self._room2
        else:
            return self._room1
```

- The Maze class represents a collection of rooms. It allows adding rooms to the maze and finding rooms by their number.
```python
class Maze:
    def __init__(self):
        self._rooms = {}

    def add_room(self, room):
        self._rooms[room._room_number] = room

    def room_no(self, room_number):
        return self._rooms.get(room_number, None)
```

### Problem Definition
This is what a typical maze setup would look like in OOP.

Here’s the initial way of creating a maze with two rooms and a door between them. 
```python
def create_maze():
    a_maze = Maze()
    r1 = Room(1)
    r2 = Room(2)
    the_door = Door(r1, r2)

    a_maze.add_room(r1)
    a_maze.add_room(r2)

    r1.set_side(0, Wall())  # North
    r1.set_side(1, the_door)  # East
    r1.set_side(2, Wall())  # South
    r1.set_side(3, Wall())  # West

    r2.set_side(0, Wall())  # North
    r2.set_side(1, Wall())  # East
    r2.set_side(2, Wall())  # South
    r2.set_side(3, the_door)  # West

    return a_maze
```

### Problems:
1. This approach is hard-coded, which means changing the layout requires rewriting the entire function.
2. This function is rigid because if you want to add new types of rooms, doors, or walls (for example, an enchanted room or a magic door), you need to manually rewrite large portions of the code. This approach does not scale well.


### Solutions
To solve this problem, we can use creational patterns to make the object creation process more flexible.

- 1. Factory Method
In this approach, instead of hard-coding the object creation (i.e., directly calling constructors), we define virtual functions like MakeRoom() or MakeDoor(). By subclassing and overriding these methods, we can easily change which classes are instantiated without rewriting the maze creation logic.

- 2. Abstract Factory
The Abstract Factory pattern allows passing an object (a factory) that handles the creation of maze components. This factory can be replaced with another one to change the types of rooms, walls, and doors used in the maze.

- 3. Builder Pattern
With the Builder pattern, you can construct complex objects step-by-step. For example, you can have a builder that handles the creation of the maze, including rooms, doors, and walls, allowing for more flexibility in the building process.

- 4. Prototype Pattern
In this pattern, the maze creation function can be parameterized by prototypes of the objects it needs to create (like a prototypical room, door, or wall). These prototypes are then cloned to create new instances. This approach allows you to change the maze’s structure by replacing the prototypes with different ones.

Note: This may be really abstract solutions.
Please check the next 4 notes to understand how to implement each of these pattern with the maze class and how they handle the problem.
