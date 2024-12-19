# Builder Pattern 

Definition: 
The **Builder** pattern is used to **separate the construction** of a complex object from its representation, meaning that the same process can create different representations of an object. 
This is particularly useful when you want to create various forms or outputs from the same building steps.

  
### **An Example**

Imagine a tool that reads **RTF (Rich Text Format)** documents. 
The tool might convert the RTF document into **plain text**, **TeX**, or even into a graphical interface where the user can **edit** the text.
The challenge here is that you may want to add new conversion formats without changing the reader itself.

The **Builder** pattern solves this by allowing you to define different conversion strategies without changing how the reader works. 
  You configure the **RTFReader** class with a **TextConverter** object, which handles the conversion of each piece of RTF text. 

For example:
- An **ASCIIConverter** would ignore all style elements and convert only the plain text.
- A **TeXConverter** would convert the text while preserving styles and formatting.
- A **TextWidgetConverter** would create a user interface component that allows users to edit the text.

![Example application](./images/builder_example_1.png)

In this way, the **RTFReader** (which is like a "director") controls the building process, 
while the **TextConverter** (the "builder") handles the creation of the specific output format. 
This lets you reuse the same parsing logic but produce different document representations.

### **When to Use the Builder Pattern**

You should use the **Builder** pattern when:
1. **The algorithm for creating a complex object** (like parsing RTF) should be independent of the parts that make up the object (like plain text, widgets, etc.).
2. **The construction process must allow for different representations** of the final object. For example, converting RTF into plain text, TeX, or a user-editable widget.


![Example builder](./images/builder_example_2.png)

### **Key Participants**
1. **Builder (TextConverter)**:
   - Defines the interface for creating parts of a product. This is an abstract class that defines how to convert RTF tokens into different formats.

2. **ConcreteBuilder (ASCIIConverter, TeXConverter, TextWidgetConverter)**:
   - Implements the builder interface for creating specific types of products (e.g., converting to plain text, TeX, or a text widget).

3. **Director (RTFReader)**:
   - Uses the builder interface to construct the object. In this case, it controls the process of parsing an RTF document and issuing conversion requests to the builder.

4. **Product (ASCIIText, TeXText, TextWidget)**:
   - Represents the final product being built. Each **ConcreteBuilder** constructs a different product, such as plain text, TeX-formatted text, or a user-editable widget.


### **How It Works**

![Example sequence](./images/builder_example_3.png)

1. The **client** creates a **Director** (in this case, the RTFReader) and configures it with the desired **Builder** (like `ASCIIConverter` or `TeXConverter`).

2. The **Director** (RTFReader) parses the document and, whenever it finds an RTF token, asks the **Builder** to handle the conversion of that token.

3. The **Builder** creates parts of the final product step by step (e.g., converting text, adding formatting, etc.).

4. Once the parsing is done, the **client** retrieves the final product from the **Builder**.


### **Consequences of Using the Builder Pattern**
1. **Varying the Product’s Internal Representation**:
   - The **Builder** pattern allows you to **vary the internal representation** of a product. For instance, you could reuse the same RTF parsing logic to create **ASCII text**, **TeX**, or a **UI widget** without changing the parsing process.

2. **Isolating Construction and Representation**:
   - The pattern separates the logic of **how an object is built** from its final representation. The **Builder** handles the creation, while the **Director** simply follows the building steps. This modularity improves code clarity and makes it easier to add new builders in the future.

3. **Finer Control Over the Construction Process**:
   - Unlike some other creational patterns that build an object all at once, the **Builder** pattern constructs the object step by step. This gives the **Director** finer control over how the object is assembled and allows changes mid-construction.


### **Example: Builder in Action**

Let's translate the RTF example into Python to show how this works:

```python
# Abstract Builder (TextConverter)
class TextConverter:
    def convert_text(self, text):
        pass

# Concrete Builders
class ASCIIConverter(TextConverter):
    def __init__(self):
        self.result = ""

    def convert_text(self, text):
        self.result += text  # Ignore formatting and add plain text

    def get_result(self):
        return self.result

class TeXConverter(TextConverter):
    def __init__(self):
        self.result = ""

    def convert_text(self, text):
        self.result += f"\\text{{{text}}}"  # Add TeX formatting

    def get_result(self):
        return self.result

class TextWidgetConverter(TextConverter):
    def __init__(self):
        self.result = []

    def convert_text(self, text):
        self.result.append(f"[Widget: {text}]")  # Simulate a UI widget

    def get_result(self):
        return self.result

# Director (RTFReader)
class RTFReader:
    def __init__(self, builder):
        self.builder = builder

    def parse(self, text):
        for word in text.split():
            self.builder.convert_text(word)

# Client code
rtf_text = "This is a sample RTF text"

# Create an ASCII converter
ascii_builder = ASCIIConverter()
reader = RTFReader(ascii_builder)
reader.parse(rtf_text)
print(ascii_builder.get_result())  # Outputs plain ASCII text

# Create a TeX converter
tex_builder = TeXConverter()
reader = RTFReader(tex_builder)
reader.parse(rtf_text)
print(tex_builder.get_result())  # Outputs TeX-formatted text
```


# Applying the Builder Pattern to the Maze Example

The **Builder** pattern can be applied to our maze example to separate the construction process from the maze's internal representation. 
By using this pattern, we can build different kinds of mazes without changing the maze creation process.
The **Builder** pattern provides flexibility by allowing us to use different builders that can construct different kinds of mazes.


The **MazeBuilder** class defines an interface for building different parts of a maze. It includes methods to:

1. Build the maze itself.
2. Build rooms with specific room numbers.
3. Build doors between two rooms.

Each of these methods does nothing by default, allowing subclasses of `MazeBuilder` to override only the methods they need.

```python
class MazeBuilder:
    def build_maze(self):
        pass

    def build_room(self, room):
        pass

    def build_door(self, room_from, room_to):
        pass

    def get_maze(self):
        return None
```

We can modify the CreateMaze function in the MazeGame class to use a builder object for constructing the maze. The builder handles the construction, while the MazeGame class simply calls the builder's methods to build the maze. The MazeGame acts as a driver here. 

```python
class MazeGame:
    def create_maze(self, builder):
        builder.build_maze()
        builder.build_room(1)
        builder.build_room(2)
        builder.build_door(1, 2)
        return builder.get_maze()
```
This version of CreateMaze is much more flexible because the builder hides the internal representation of the maze (how rooms, doors, and walls are created). This means you can change the maze's representation without modifying the game logic.

The StandardMazeBuilder is a concrete implementation of the MazeBuilder class. 
It builds simple mazes by creating rooms and doors and keeping track of the maze being built.
```python
class StandardMazeBuilder(MazeBuilder):
    def __init__(self):
        self._current_maze = None

    def build_maze(self):
        self._current_maze = Maze()

    def get_maze(self):
        return self._current_maze

    def build_room(self, n):
        if not self._current_maze.room_no(n):
            room = Room(n)
            self._current_maze.add_room(room)
            room.set_side("North", Wall())
            room.set_side("South", Wall())
            room.set_side("East", Wall())
            room.set_side("West", Wall())

    def build_door(self, n1, n2):
        r1 = self._current_maze.room_no(n1)
        r2 = self._current_maze.room_no(n2)
        door = Door(r1, r2)
        r1.set_side(self.common_wall(r1, r2), door)
        r2.set_side(self.common_wall(r2, r1), door)

    def common_wall(self, r1, r2):
        direction = "East" # Logic to determine the common wall direction (could be E, N, S, W)
        return direction
```
The builder keeps track of the maze it's constructing (_current_maze) and provides utility functions like common_wall to handle relationships between rooms.
```python
game = MazeGame()
builder = StandardMazeBuilder()
game.create_maze(builder)
maze = builder.get_maze()
```

We could similarly have builders for building enchanted, or bombed rooms.
```python
class EnchantedRoom(Room):
    def __init__(self, room_number, spell):
        super().__init__(room_number)
        self._spell = spell

class DoorNeedingSpell(Door):
    def __init__(self, room1, room2):
        super().__init__(room1, room2)
        self._spell_required = True

class EnchantedMazeBuilder(MazeBuilder):
    def __init__(self):
        self._current_maze = None

    def build_maze(self):
        self._current_maze = Maze()

    def get_maze(self):
        return self._current_maze

    def build_room(self, n):
        if not self._current_maze.room_no(n):
            room = EnchantedRoom(n, self.cast_spell())
            self._current_maze.add_room(room)
            room.set_side("North", Wall())
            room.set_side("South", Wall())
            room.set_side("East", Wall())
            room.set_side("West", Wall())

    def build_door(self, n1, n2):
        r1 = self._current_maze.room_no(n1)
        r2 = self._current_maze.room_no(n2)
        door = DoorNeedingSpell(r1, r2)
        r1.set_side(self.common_wall(r1, r2), door)
        r2.set_side(self.common_wall(r2, r1), door)

    def common_wall(self, r1, r2):
        # Logic to determine the common wall direction
        return "East"

    def cast_spell(self):
        return "Magic Spell"

game = MazeGame()
builder = EnchantedMazeBuilder()
game.create_maze(builder)
enchanted_maze = builder.get_maze()

```

We can create a more exotic builder like CountingMazeBuilder. This builder doesn’t create a maze but counts the number of rooms and doors that would have been created. It shows how the builder pattern can be used for purposes other than constructing objects.
```python
class CountingMazeBuilder(MazeBuilder):
    def __init__(self):
        self._rooms = 0
        self._doors = 0

    def build_room(self, n):
        self._rooms += 1

    def build_door(self, n1, n2):
        self._doors += 1

    def get_counts(self):
        return self._rooms, self._doors

game = MazeGame()
builder = CountingMazeBuilder()
game.create_maze(builder)
rooms, doors = builder.get_counts()
print(f"The maze has {rooms} rooms and {doors} doors")

```
This builder doesn’t create any objects—it just counts them. It’s a perfect example of how flexible the builder pattern can be.

*(IMPORTANT)*
The primary difference is that the Builder pattern focuses on constructing a complex object step by step. Abstract Factory's emphasis is on families of product objects




