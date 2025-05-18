## Wrapper Functions

A wrapper function is a function that provides a layer of abstraction over another function, 
enhancing or modifying its behavior without changing the function itself. 
In Python, this is commonly achieved using decorators.
This is extremely important from the point of view of extensibility of applications. 

Simply:
In Python, the leading @ simply means “take the thing that follows and pass it through a decorator before Python stores it.” 
A decorator is just a function that receives another function or class, adds or changes behavior, and hands back a new (or modified) object.

### Common use cases
- Logging: Automatically log function calls and results.
- Access Control: Enforce user authentication or permissions before executing a function.
- Input Validation: Check arguments before passing them to the target function.
- Caching: Store results of expensive function calls for faster future access.

Consider the access control scenario:
```python
from functools import wraps

# Mock authentication function
def is_authenticated(user):
    return user.get('authenticated', False)

# The decorator function
def login_required(func):
    @wraps(func)  #Preserves the original function's metadata.
    def wrapper(user, *args, **kwargs):
        if not is_authenticated(user):
            raise PermissionError("User must be authenticated to access this function.")
        return func(user, *args, **kwargs)
    return wrapper

# API endpoint functions
# get_user_profile and update_user_settings are decorated and thus wrapped by login_required
@login_required
def get_user_profile(user):
    print(f"Fetching profile for {user['name']}.")

@login_required
def update_user_settings(user, settings):
    print(f"Updating settings for {user['name']} to {settings}.")

# Usage
user = {'name': 'John Doe', 'authenticated': True}
unauthenticated_user = {'name': 'Jane Smith', 'authenticated': False}

# This will work
get_user_profile(user)

# This will raise a PermissionError
try:
    get_user_profile(unauthenticated_user)
except PermissionError as e:
    print(e)

### OUTPUT
Fetching profile for John Doe.
User must be authenticated to access this function.

```

### Benefits
Code Reusability: Write authentication checks once and apply them to multiple functions.
Separation of Concerns: Keeps business logic separate from access control logic.
Enhancements: Easily add new behaviors (like logging) without modifying existing code.

Note:
we can wrap a function using multiple decorators.
```python
@decorator_n
@decorator_n-1
...
@decorator_1
def my_function():
    pass
```
