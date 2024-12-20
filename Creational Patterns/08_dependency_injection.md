### Dependency Injection Pattern

**Dependency Injection (DI)** is not one of the original Gang of Four (GoF) design patterns. 
Instead, it's considered a technique or a pattern often associated with `Inversion of Control (IoC)`. 
The main idea behind DI is to prevent objects from creating or managing their dependencies by themselves. 
Instead, these dependencies (services, or other required objects) are "injected" into them from the outside.
 
**Key Idea**: Rather than having a class construct find the objects it needs to work, 
those objects are provided from the outside. 
This leads to code that is more modular, flexible, testable, and easier to change, since changing a dependency doesn’t require modifying the class that uses it.


### When to Use Dependency Injection

- You want to reduce coupling between classes.
- You want to make it easier to replace dependencies with alternatives (for example, for testing).
- You want to improve code flexibility and maintainability.


### Applying Dependency Injection to the Maze Example

Look at the basic classes we have in the maze game:
```python
class MapSite:
    def enter(self):
        pass

class Room(MapSite):
    def __init__(self, room_number):
        self._room_number = room_number
        self._sides = [None] * 4

    def set_side(self, direction, site):
        self._sides[direction] = site

    def enter(self):
        print(f"Entering room {self._room_number}")

class Wall(MapSite):
    def enter(self):
        print("You hit a wall!")

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

class Maze:
    def __init__(self):
        self._rooms = {}

    def add_room(self, room):
        self._rooms[room._room_number] = room

    def room_no(self, room_number):
        return self._rooms.get(room_number, None)

```


In our maze game examples, we've typically had a `MazeGame` class that directly instantiates its components or relies on a specific `MazeFactory` to create maze elements (rooms, walls, doors). 
```python
class MazeFactory:
    def make_maze(self):
        raise NotImplementedError()

    def make_room(self, number):
        raise NotImplementedError()

    def make_wall(self):
        raise NotImplementedError()

    def make_door(self, room1, room2):
        raise NotImplementedError()

class StandardMazeFactory(MazeFactory):
    def make_maze(self):
        return Maze()

    def make_room(self, number):
        return Room(number)

    def make_wall(self):
        return Wall()

    def make_door(self, room1, room2):
        return Door(room1, room2)

class EnchantedMazeFactory(MazeFactory):
    def make_maze(self):
        return Maze()

    def make_room(self, number):
        # Imagine EnchantedRoom is different
        room = EnchantedRoom(number)
        return room

    def make_wall(self):
        # Could return a special EnchantedWall or a regular wall
        return EnchantedWall() id random.randint(1, 10) % 2 == 0 else Wall()

    def make_door(self, room1, room2):
        # Could return a special DoorNeedingSpell
        return EnchantedDoor(room1, room2, spell)

```

Without DI, `MazeGame` might look like this:

```python
class MazeGame:
    def create_maze(self, is_enchanted):
        # MazeGame decides what factory to use, and creates it here
        if is_enchanted:
            factory = EnchantedMazeFactory() # Hard-coded dependency
        else:
            factory = StandardMazeFactory()  # Hard-coded dependency
        maze = factory.make_maze()
        r1 = factory.make_room(1)
        r2 = factory.make_room(2)
        door = factory.make_door(r1, r2)

        maze.add_room(r1)
        maze.add_room(r2)

        r1.set_side(0, factory.make_wall())  # North
        r1.set_side(1, door)                 # East
        r1.set_side(2, factory.make_wall())  # South
        r1.set_side(3, factory.make_wall())  # West

        r2.set_side(0, factory.make_wall())  # North
        r2.set_side(1, factory.make_wall())  # East
        r2.set_side(2, factory.make_wall())  # South
        r2.set_side(3, door)                 # West

        return maze
```
Here, MazeGame decides which factory to use (StandardMazeFactory or EnchantedMazeFactory). 
If we want to switch to HauntedMazeFactory or BombedMazeFactory, we must modify MazeGame. 

Important: Also, testing MazeGame becomes harder because we can't easily provide a mock or test factory.
With Dependency Injection, we can have MazeGame receive its dependencies from the outside.
This makes MazeGame more flexible and easier to test.


MazeGame with Dependency Injection
```python
class MazeGame:
    def __init__(self, factory: MazeFactory):
        # Instead of deciding which factory to use, we accept it as a dependency.
        self.factory = factory # Dependency Injection

    def create_maze(self):
        maze = self.factory.make_maze()
        r1 = self.factory.make_room(1)
        r2 = self.factory.make_room(2)
        door = self.factory.make_door(r1, r2)

        maze.add_room(r1)
        maze.add_room(r2)

        r1.set_side(0, self.factory.make_wall())  # North
        r1.set_side(1, door)                      # East
        r1.set_side(2, self.factory.make_wall())  # South
        r1.set_side(3, self.factory.make_wall())  # West

        r2.set_side(0, self.factory.make_wall())  # North
        r2.set_side(1, self.factory.make_wall())  # East
        r2.set_side(2, self.factory.make_wall())  # South
        r2.set_side(3, door)                      # West

        return maze

# With dependency injection, we decide which factory to use at the time we create MazeGame:
if __name__ == "__main__":
    # Injecting a StandardMazeFactory
    standard_factory = StandardMazeFactory()
    game = MazeGame(factory=standard_factory)
    maze = game.create_maze()

    # If we want an enchanted maze:
    enchanted_factory = EnchantedMazeFactory()
    enchanted_game = MazeGame(factory=enchanted_factory)
    enchanted_maze = enchanted_game.create_maze()

    # you can see how this makes testing easier as we can easily mock the dependent objects and their states
```

This example illustrates:
- Flexibility: To switch factories, we just pass a different one to the MazeGame constructor. No code changes in MazeGame are required.
- Testability: For testing, we can pass a mock or fake factory to MazeGame. This makes it easier to isolate and test MazeGame in unit tests.
- Loose Coupling: MazeGame no longer depends on the concrete factory classes. It just uses whatever MazeFactory is passed in.


