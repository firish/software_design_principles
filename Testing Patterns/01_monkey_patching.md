# Monkey Patching

## 1. What Is Monkey Patching?

**Monkey patching** refers to **dynamically (at runtime) modifying, adding, or replacing** the behavior of classes, functions, or modules **without altering their original source code**.

### Why Is It Called “Monkey” Patching?
- One explanation is the idea of “keeping a ‘monkey’ on your back” to fix a problem when you can’t modify the original code directly.
- Another is simply that it’s a playful name that stuck in the Python community (and other dynamic language communities).

---

## 2. Typical Use Cases

- **Testing**: Replace real dependencies (like database calls, network requests, or hardware sensors) with mock or stub implementations.
- **Temporary Bug Fixes**: Patch a third-party library (e.g., if there’s a bug) until you can upgrade to a newer, fixed version.
- **Prototyping**: Quickly modify behavior in a running system or interactive environment to see how a new approach might work.

### Why Use It in Testing?
- **Isolation**: You can isolate the piece of code you want to test from external dependencies (APIs, databases, etc.).
- **Control**: You can force certain code paths—like error conditions or timeouts—that might be difficult or slow to replicate in real life.
- **Speed**: You avoid slow operations or setups by faking them at runtime, so tests run faster.

---

## 3. Pros and Cons of Monkey Patching

**Pros**:
- Very flexible; leverages Python’s dynamic nature.
- Allows quick fixes or test modifications without changing original source.
- Great for mocking external dependencies in testing scenarios.

**Cons**:
- Can lead to hard-to-track bugs if patches affect unexpected parts of the system.
- May cause confusion if multiple monkey patches stack up or conflict.
- Not always the cleanest design; consider alternatives like dependency injection if feasible.

---

## 4. Simple Python Example

Suppose you have a class `DataFetcher` that calls an external API:

```python
# data_fetcher.py
import requests

class DataFetcher:
    def get_data(self, url):
        """
        Fetches JSON data from a specified URL.
        """
        response = requests.get(url)
        response.raise_for_status()
        return response.json()
```

For testing, you might not want to make an actual network call. Instead, you can “monkey patch” `requests.get` (or `DataFetcher.get_data`) so your test doesn’t rely on an external service.

### Monkey Patching in a Test

```python
# test_data_fetcher.py
import unittest
from unittest.mock import patch
from data_fetcher import DataFetcher

def fake_get(*args, **kwargs):
    """Fake replacement for requests.get"""
    class FakeResponse:
        def raise_for_status(self):
            pass
        def json(self):
            return {"name": "Monkey", "type": "Animal"}
    return FakeResponse()

class TestDataFetcher(unittest.TestCase):
    @patch('data_fetcher.requests.get', side_effect=fake_get)
    def test_get_data(self, mock_get):
        fetcher = DataFetcher()
        data = fetcher.get_data("http://fakeurl.com")
        self.assertEqual(data["name"], "Monkey")
        self.assertEqual(data["type"], "Animal")

if __name__ == "__main__":
    unittest.main()
```

**Explanation**:
- `@patch('data_fetcher.requests.get', side_effect=fake_get)` overrides (`patches`) the `requests.get` function **within** the `data_fetcher` module.
- Whenever `DataFetcher.get_data` calls `requests.get`, our `fake_get` function is invoked instead.
- We can now test logic without making actual network calls.

---

## 5. A Real-World Detailed Example

Imagine you have a larger application that regularly interacts with an external mapping service to get geographic data. For instance:

```python
# location_service.py
import requests

class LocationService:
    def get_user_location(self, user_id):
        # Suppose we build a request URL using some ID
        url = f"https://api.maps.example.com/userlocation/{user_id}"
        response = requests.get(url)
        response.raise_for_status()
        return response.json()

    def process_user_location(self, user_id):
        location_data = self.get_user_location(user_id)
        # Some more processing...
        return {
            "latitude": location_data.get("lat"),
            "longitude": location_data.get("lon"),
            "city": location_data.get("city"),
        }
```

### Challenge: During Testing
You want to check scenarios like:
1. Valid location.
2. API call fails (throws an exception).
3. Returned JSON is missing expected fields (e.g., missing `"lat"`).

### Step-by-Step Testing with Monkey Patching

1. **Create Fake Responses**  
2. **Test Successful Response**  
3. **Test Failure Response**  

In these tests:
- We replaced (monkey patched) the `requests.get` call with a fake function.
- By returning controlled results, we can test how `LocationService` behaves in different scenarios **without** actually calling a third-party API.

### Example Test Code

```python
# test_location_service.py
import unittest
from unittest.mock import patch
import requests
from location_service import LocationService

def fake_requests_get_success(*args, **kwargs):
    class FakeResponse:
        def raise_for_status(self):
            pass  # no exception
        def json(self):
            return {
                "lat": 123.45,
                "lon": 67.89,
                "city": "ExampleCity"
            }
    return FakeResponse()

def fake_requests_get_failure(*args, **kwargs):
    class FakeResponse:
        def raise_for_status(self):
            raise requests.exceptions.HTTPError("Not Found")
        def json(self):
            return {}
    return FakeResponse()

class TestLocationService(unittest.TestCase):
    @patch('location_service.requests.get', side_effect=fake_requests_get_success)
    def test_process_user_location_success(self, mock_get):
        service = LocationService()
        result = service.process_user_location("user123")
        self.assertEqual(result["latitude"], 123.45)
        self.assertEqual(result["longitude"], 67.89)
        self.assertEqual(result["city"], "ExampleCity")

    @patch('location_service.requests.get', side_effect=fake_requests_get_failure)
    def test_process_user_location_failure(self, mock_get):
        service = LocationService()
        with self.assertRaises(Exception):
            # Because raise_for_status() will raise an HTTPError
            service.process_user_location("userNotFound")

if __name__ == "__main__":
    unittest.main()
```

---

## 6. Key Takeaways

1. **Runtime Modification**  
   Monkey patching leverages Python’s dynamic nature—classes and functions can be rebound at any time.
2. **Useful for Testing**  
   - Simulate any scenario (network errors, missing fields, etc.) on demand.
   - Avoid real I/O, speeding up tests.
3. **Use with Caution**  
   - Global patching means **all** calls in that module use the patched version.
   - `unittest.mock.patch` is helpful because it handles setup/teardown automatically.
4. **Better Alternatives Sometimes**  
   - **Dependency injection**: pass in mock objects directly, so you don’t need to patch them at runtime.
   - Still, monkey patching remains powerful for quick or legacy scenarios.
