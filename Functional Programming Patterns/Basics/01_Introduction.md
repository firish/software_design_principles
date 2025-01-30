### What is Functional Programming?

1. Mathematical Functions

In FP, functions are modeled as pure mathematical functions: given the same inputs, they always produce the same outputs and do not modify any state outside of their scope.

1.1 First-Class and Higher-Order Functions: 
Functions can be assigned to variables, passed as arguments, and returned from other functions. 
This flexibility leads to a style of programming that promotes code reusability and composition.

1.2 Immutability:
Data is treated as immutable (unchangeable). 
Instead of altering data in-place, FP encourages creating new versions of data structures with the changes. 
This immutability reduces unintended side effects.

1.3 No Side Effects (Purity): 
FP aims to minimize or completely avoid side effects. 
A “pure” function’s output depends solely on its inputs and does not cause observable changes (like modifying a global variable or printing to the screen).

1.4 Examples of Functional Programming Languages
Haskell, Elixir, OCaml, F#, Scala, JavaScript. 


2. How They Differ from Other Paradigms

2.1 Imperative / Procedural:
Imperative code describes how to perform operations step by step. 
Data can be mutated throughout the process.
Functional code describes what to compute using expressions and function composition, avoiding or minimizing direct state changes.

2.2 Object-Oriented Programming (OOP):
OOP organizes code around objects that hold state, exposing behaviors that modify that state.
FP organizes code around functions that transform data, often requiring new copies instead of mutating existing objects.

2.3 Declarative Nature:
Functional programming is considered declarative.
It focuses on the “What” (expressions, transformations, function pipelines) instead of the “How” (managing variable changes).


3. Advantages of Functional Programming

3.1 Modularity and Composability: Functions can be composed to build more complex functionality, making code more reusable and modular.

3.2 Easier Reasoning / Fewer Bugs: Pure functions with no side effects are easier to test and reason about, since the same input always yields the same output.

3.3 Parallelization / Concurrency: Because immutable data structures and pure functions don’t interfere with shared state, it can be easier to introduce parallel or concurrent execution.

3.4 Maintainability: Code tends to be more concise and modular, making it easier to refactor.



### Functional Programming Concepts

Below is a broad, language-agnostic list of functional programming concepts. 
Some are built-in functions (like map, filter, reduce), while others are design techniques (like composition, partial application, pattern matching, etc.). 
Although certain terms may vary across languages, these concepts apply widely in the functional world.

1. Fundamental Function Operations
- Higher-Order Functions (HOFs)
- map
- filter
- reduce / fold
- zip / zipWith

2. Functional Composition and Function Manipulation
- Function Composition
- Partial Application
- Currying
- Point-Free Style

3. Data Structures and Transformations
- Immutable Data Structures
- Pattern Matching
- List / Sequence Comprehensions
- Lazy Evaluation

4. Handling Side Effects
- Pure Functions
- I/O and Effects Management
- Monads, Functors, Applicatives
- IO: For side-effecting operations like file handling or network requests.

5. Advanced Techniques
- Recursion Patterns
- Memoization
- Transducers
- Pipelining / Threading Macros
- Lenses (and Prisms, Traversals)
- Algebraic Data Types (ADTs)
- Type Classes / Protocols
- For Comprehensions


No single language will offer all of these as built-in features, 
but each concept is ubiquitous across functional ecosystems. 
By understanding and combining these building blocks, you can write clean, maintainable, and truly functional code.
