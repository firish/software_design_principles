# Structural Pattern - Decorator
- Using the same maze example from creational patterns

### Understanding the Decorator Pattern

The **Decorator** pattern is a **structural** design pattern that allows behavior to be added to individual objects, dynamically, 
without affecting the behavior of other objects from the same class. 
Decorators provide a flexible alternative to subclassing for extending functionality.

**Key Idea**: 
Instead of modifying or subclassing a class directly to add extra features, we wrap the original object in another object (the decorator) that "decorates" the original, adding new behaviors or responsibilities.


### When to Use the Decorator Pattern

- When you want to add responsibilities to individual objects dynamically and transparently.
- When extending a class by subclassing would result in an explosion of subclasses.
- When you want to avoid affecting other objects of the same class.


### Decorator Pattern Structure

1. **Component (MapSite)**: Defines an interface for objects that can have responsibilities added to them dynamically.
2. **Concrete Component (Room, Wall, Door)**: Objects that can have responsibilities added to them.
3. **Decorator (MapSiteDecorator)**: Maintains a reference to a Component object and defines an interface that conforms to the Component.
4. **Concrete Decorators (e.g., TrapDecorator, TreasureDecorator)**: Add responsibilities to the component by modifying the behavior of `enter()` or adding additional state.


### Applying the Decorator to the Maze Example

We have a maze with `MapSite`, `Room`, `Wall`, and `Door` as before. 
Normally, entering a room or a door is straightforward. 
Suppose we want to add extra features, like traps or treasure, without changing the `Room` or `Door` classes. 
We can use decorators!


**Scenario**:  
- We have rooms (just like before).  
- We introduce a decorator that can add a trap or treasure to any `MapSite`.  
- By wrapping a `Room` with a `TrapDecorator`, the room now has a trap waiting for the player.  
- By wrapping a `Room` with a `TreasureDecorator`, the room now contains treasure.

We don't subclass `Room` into `TrapRoom` or `TreasureRoom`; instead, we keep `Room` as is and decorate it at runtime.


### Code Example in Python

#### Maze Components (Unchanged)

```python
class MapSite:
    def enter(self):
        pass

class Room(MapSite):
    def __init__(self, room_number):
        self._room_number = room_number
        self._sides = [None] * 4  # directions: 0=N,1=E,2=S,3=W

    def set_side(self, direction, map_site):
        self._sides[direction] = map_site

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

    def set_room(self, room_number, room):
        self._rooms[room_number] = room

def create_simple_maze():
    maze = Maze()
    r1 = Room(1)
    r2 = Room(2)
    r3 = Room(3)
    r4 = Room(4)
    door_12 = Door(r1, r2)
    door_13 = Door(r1, r3)
    door_34 = Door(r3, r4)

    maze.add_room(r1)
    maze.add_room(r2)
    maze.add_room(r3)
    maze.add_room(r4)

    r1.set_side(0, Wall())  # North
    r1.set_side(1, door_12) # East
    r1.set_side(2, Wall())  # South
    r1.set_side(3, door_13) # West

    r2.set_side(0, Wall())  # North
    r2.set_side(1, Wall())  # East
    r2.set_side(2, Wall())  # South
    r2.set_side(3, door_12) # West

    r3.set_side(0, door_13) # North
    r3.set_side(1, door_34) # East
    r3.set_side(2, Wall())  # South
    r3.set_side(3, Wall()   # West

    r4.set_side(0, Wall())  # North
    r4.set_side(1, Wall())  # East
    r4.set_side(2, door_34) # South
    r4.set_side(3, Wall()   # West

    return maze
```

```python
class MapSiteDecorator(MapSite):
    def __init__(self, map_site):
        self._map_site = map_site

    def enter(self):
        # Default behavior: just delegate to the wrapped map site
        self._map_site.enter()

# Concrete Decorator: TrapDecorator
class TrapDecorator(MapSiteDecorator):
    def __init__(self, map_site):
        super().__init__(map_site)
        self._trap_triggered = False

    def enter(self):
        # First, call the original enter logic
        self._map_site.enter()
        # Then add trap behavior
        if not self._trap_triggered:
            print("A trap springs! You lose some health.")
            self._trap_triggered = True
        else:
            print("The trap here has already been triggered.")

# Concrete Decorator: TreasureDecorator
class TreasureDecorator(MapSiteDecorator):
    def __init__(self, map_site):
        super().__init__(map_site)
        self._treasure_taken = False

    def enter(self):
        # First, call the original enter logic
        self._map_site.enter()
        # Then add treasure behavior
        if not self._treasure_taken:
            print("You found treasure! Your wealth increases.")
            self._treasure_taken = True
        else:
            print("You've already taken the treasure from here.")
```

Using these decorators in runtime 
```python
def decorate_maze(maze, room_number_to_booby_trap, room_number_with_treasure):

    # Let's decorate room #1 with a trap and room #2 with treasure.
    # Get the rooms
    r1 = maze.room_no(room_number_to_booby_trap)
    r2 = maze.room_no(room_number_with_treasure)

    # Decorate them
    booby_trapped_room = TrapDecorator(r1)
    treasure_room = TreasureDecorator(r2)

    # replace the maze rooms with trapped rooms
    maze.set_room(r1, booby_trapped_room)
    maze.set_room(r2, treasure_room)
    return maze

# Usage Example
if __name__ == "__main__":
    maze = create_simple_maze()

    # dynamically update the maze in runtime
    room_number_to_trap = int(input("Please enter the room to booby trap for the opponent"))
    treasure_room_number = int(input("Please enter the room to hide your treasure"))

    # decorate the maze
    decorate_maze(maze, room_number_to_trap, treasure_room_number)

    # Enter the trapped room
    trapped_room.enter()   # Enters room 1, then triggers a trap
    trapped_room.enter()   # Trap is already triggered, no new effect

    # Enter the treasure room
    treasure_room.enter()  # Enters room 2, finds treasure
    treasure_room.enter()  # Treasure already taken
```


### How is this different from the traditional approach

The important difference is what we're creating classes for and how they're used, not just how many classes exist.

#### Traditional Inheritance Approach (Without Decorators)
If you wanted different kinds of rooms with different behaviors using just inheritance, you might create:
`TrapRoom`
`TreasureRoom`
Maybe later you want a `TrapTreasureRoom` (a room with both a trap and treasure).
That would lead to a growing number of subclasses. For each new feature or combination, you’d create a new subclass. Over time, this could explode into a large number of classes.

#### Decorator Approach
With decorators, we still create classes for each new feature, but these classes aren’t subclasses of Room; 
they are decorators that can be attached to any MapSite (room, door, etc.).
Instead of creating a `TrapRoom`, we create a `TrapDecorator` that can wrap a Room.
The key difference: Decorators can be combined dynamically and applied to many different components without creating new subclasses.
For example:
To get a room with a trap: Wrap the Room with a `TrapDecorator`.
To get a room with treasure: Wrap the Room with a `TreasureDecorator`.
To get a room with both a trap and treasure: Wrap the Room first with a TrapDecorator and then wrap that with a TreasureDecorator (or vice versa). You don't need a new TrapTreasureRoom class.

#### Why This Is Better
 - Flexibility: Instead of a subclass for every possible combination (Trap+Treasure, Trap+Treasure+AnotherFeature, etc.), you just combine decorators as needed.
 - Reusability: `TrapDecorator` can be used on any MapSite (rooms, doors, etc.), and TreasureDecorator can do the same. You don’t have to write `TreasureDoor`, `TreasureWall`, `TrapDoor`, `TrapWall`, etc. Just wrap the original objects in the decorators.
- Reduced Class Explosion: Yes, you create decorator classes for new features, but you don’t need separate subclasses for every combination of features and base classes. Two decorators can be combined in multiple ways. If you add a third decorator (say `HealingDecorator`), you can combine it with TrapDecorator and TreasureDecorator in any configuration, all without creating new subclasses.
