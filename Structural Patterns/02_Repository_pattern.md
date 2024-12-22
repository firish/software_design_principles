### Understanding the Repository Pattern

**Repository** is not one of the original 23 design patterns from the Gang of Four (GoF). 
Instead, it’s a pattern popularized in **Domain-Driven Design (DDD)** book. 
The **Repository** pattern provides a way to **abstract data access** from the domain or business logic. 

In simpler terms, it acts like an in-memory collection, 
**hiding** the technical details of how data is actually stored or retrieved (database, file, external service, etc.) from the rest of the application. T
his allows your code to remain focused on the domain logic, while the repository handles persistence details.


### **Key Idea**

- The **Repository** pattern acts like a **middleman** between your domain objects and the data layer. 
- It typically provides methods to **add**, **remove**, **update**, and **retrieve** objects.
- The rest of your application **doesn’t** need to know **how** the data is stored—only **that** it can be saved and retrieved.


### **Why Use the Repository Pattern?**

1. **Separation of Concerns**: Keep domain logic separate from data access logic.
2. **Easier Maintenance**: Changing your storage mechanism (database, file system, etc.) doesn’t affect the core domain logic.
3. **Unit Testing (IMP)**: It’s easier to mock or stub repositories in tests, avoiding real database or filesystem calls.
   

### Applying the Repository Pattern

#### Real-World Scenario: Library Management
Consider a library system that tracks books. 
We want to:
- Store book information (title, author, and year).
- Retrieve books from storage (in-memory or database).
- Perform operations on books (searching by title, listing all books, etc.).
  
Domain: 
- A Book object represents a book in the library.
- A LibraryService class handles the operations on books.

Persistence: 
- A BookRepository abstracts how books are stored and retrieved.

Domain Class.
```python
class Book:
    """
    Domain model representing a Book in our library system.
    """
    def __init__(self, book_id, title, author, year):
        self.book_id = book_id      # unique identifier
        self.title = title
        self.author = author
        self.year = year

    def __repr__(self):
        return f"Book(book_id={self.book_id}, title={self.title}, author={self.author}, year={self.year})"
```
Book is our domain entity. Notice it does not know anything about how it’s persisted or retrieved.

Repository Interface.
```python
from abc import ABC, abstractmethod

class BookRepository(ABC):
    @abstractmethod
    def save(self, book: Book):
        """
        Save or update a book in the repository.
        """

    @abstractmethod
    def find_by_id(self, book_id: str) -> Book:
        """
        Retrieve a book by its ID. Returns None if not found.
        """

    @abstractmethod
    def find_all(self) -> list[Book]:
        """
        Retrieve all books in the repository.
        """

    @abstractmethod
    def delete(self, book_id: str):
        """
        Remove a book from the repository by ID.
        """
```
Defines CRUD-like operations: save, find_by_id, find_all, and delete.

Concrete repository
```python
class InMemoryBookRepository(BookRepository):
    def __init__(self):
        self._storage = {}  # dict: book_id -> Book

    def save(self, book: Book):
        self._storage[book.book_id] = book
        print(f"[InMemoryBookRepository] Book '{book.book_id}' saved/updated.")

    def find_by_id(self, book_id: str) -> Book:
        return self._storage.get(book_id)

    def find_all(self) -> list[Book]:
        return list(self._storage.values())

    def delete(self, book_id: str):
        if book_id in self._storage:
            del self._storage[book_id]
            print(f"[InMemoryBookRepository] Book '{book_id}' deleted.")
```
In a real system, you might have a DatabaseBookRepository that uses SQL or NoSQL calls.
In a real system, you might have multiple repository concrete classes, for storing different objects.
Lets say the library has:
1. Millions of books, and hundred thousand transcations per day, and uses Nosql for that (1 repository for that)
2. Thousands of DVDs, with tens of transactions per day, and uses a simple SQL server database (1 repository for that)
3. Weekly auctions for antique books, and uses apache kafka to manage bids in real time (1 repository for that)

Library Service: Domain logic 
```python
class LibraryService:
    """
    High-level application logic that deals with books,
    relying on a BookRepository to handle persistence.
    """
    def __init__(self, repository: BookRepository):
        self.repository = repository

    def add_new_book(self, book_id, title, author, year):
        existing = self.repository.find_by_id(book_id)
        if existing:
            raise ValueError(f"Book with ID '{book_id}' already exists!")
        book = Book(book_id, title, author, year)
        self.repository.save(book)

    def list_all_books(self):
        return self.repository.find_all()

    def get_book_details(self, book_id):
        book = self.repository.find_by_id(book_id)
        if not book:
            print(f"No book found with ID '{book_id}'")
        return book

    def remove_book(self, book_id):
        self.repository.delete(book_id)
```
LibraryService uses the repository to manage books. Notice it does not contain any code about how the books are stored.
This concept would typically be extended to manage the inventory of the books. 


