
# Adapter Pattern</br>

## What Type of Design Pattern Is This?</br>
The **Adapter** pattern is a **structural** design pattern. 
It allows objects with **incompatible interfaces** to work together by wrapping one of the objects in an adapter,
that translates or adapts the interface into something the other object expects.</br>


## Why Use the Adapter Pattern?</br>
1. **Interface Mismatch**: When you have an existing class or service, but its interface doesn't match what a client expects.  
2. **Reusability**: When you want to reuse existing classes in a new system that expects a different interface, without modifying the original classes.  
3. **Simplifying Client Code**: The adapter hides the complexity of conversions or data transformations from the client code.</br>


## Classic Real-World Example: **Media Player**</br>
**Scenario**:</br>
- You have a **simple media player** that can play **MP3** files only.  
- You also have an **advanced media player** that can handle more formats (e.g., **MP4** or **MKV**).  
- The client code is built around a **MediaPlayer** interface that expects a `play_audio(file_name)` method.  
- You want to integrate advanced capabilities (MP4 or MKV playback) into your existing code, but the advanced player has a different interface (e.g., `play_mp4(file)`, `play_mkv(file)`).</br>

**Solution**:</br>
Use an **Adapter** that implements the `MediaPlayer` interface but internally calls methods on the advanced media player, 
bridging the interface gap.</br>


## Code Example in Python</br>

### **1. The Target Interface (MediaPlayer)**</br>
```python
import abc

class MediaPlayer(abc.ABC):
    @abc.abstractmethod
    def play_audio(self, file_name: str):
        pass
```

- `MediaPlayer` is an interface (abstract class in Python) with a single method `play_audio`.  
- Clients only know about this interface and call `play_audio(file_name)`.  
</br>

### **2. The Existing or Simple Implementation**</br>
```python
class Mp3Player(MediaPlayer):
    def play_audio(self, file_name: str):
        # Pretend we can only play MP3 files
        if file_name.lower().endswith(".mp3"):
            print(f"[Mp3Player] Playing MP3 file: {file_name}")
        else:
            print(f"[Mp3Player] Error: Unable to play file {file_name}, not an MP3!")
```

- `Mp3Player` implements `MediaPlayer`.  
- It only knows how to play `.mp3` files, returning an error for other formats.  
</br>

### **3. The Adaptee (AdvancedMediaPlayer)**</br>
```python
class AdvancedMediaPlayer:
    """
    This is a separate interface/class that can play advanced formats.
    But it doesn't follow the MediaPlayer interface.
    """
    def play_mp4(self, file_name: str):
        print(f"[AdvancedMediaPlayer] Playing MP4 file: {file_name}")

    def play_mkv(self, file_name: str):
        print(f"[AdvancedMediaPlayer] Playing MKV file: {file_name}")
```
- `AdvancedMediaPlayer` can play MP4 or MKV, but **has a different interface**.  
- There's no `play_audio` method, so we can't directly use it where a `MediaPlayer` is expected.  
</br>

### **4. The Adapter (MediaAdapter)**</br>
```python
class MediaAdapter(MediaPlayer):
    """
    An adapter that implements the MediaPlayer interface and internally
    uses an AdvancedMediaPlayer to handle non-MP3 formats.
    """
    def __init__(self, advanced_player: AdvancedMediaPlayer):
        self._advanced_player = advanced_player

    def play_audio(self, file_name: str):
        if file_name.lower().endswith(".mp4"):
            self._advanced_player.play_mp4(file_name)
        elif file_name.lower().endswith(".mkv"):
            self._advanced_player.play_mkv(file_name)
        else:
            print(f"[MediaAdapter] Error: Unknown format {file_name}")
```
**Explanation**:</br>
- `MediaAdapter` inherits from `MediaPlayer` (the **target interface**).  
- It holds an instance of `AdvancedMediaPlayer` (the **adaptee**).  
- In `play_audio`, it **checks** the file extension. If `.mp4` or `.mkv`, it delegates the call to the advanced player’s corresponding method.  
</br>

### **5. The Client or Main Application**</br>
We want a convenient way for the client to play any audio file, using either **Mp3Player** or **MediaAdapter** (with an advanced player behind it).</br>

```python
class AudioPlayer:
    """
    High-level class that uses the adapter behind the scenes.
    """
    def __init__(self):
        # By default, we can handle MP3 with a simple mp3 player
        self.mp3_player = Mp3Player()
        # We also have a single advanced player we can wrap in an adapter
        self.advanced_player = AdvancedMediaPlayer()
        self.adapter = MediaAdapter(self.advanced_player)

    def play(self, file_name: str):
        if file_name.lower().endswith(".mp3"):
            self.mp3_player.play_audio(file_name)
        else:
            self.adapter.play_audio(file_name)

# Client Code
def main():
    player = AudioPlayer()
    # Use the same AudioPlayer to handle different formats
    player.play("song.mp3")
    player.play("movie.mp4")
    player.play("trailer.mkv")
    player.play("archive.zip")  # Unknown format
```
**Explanation**:</br>
- `AudioPlayer` decides how to **route** the request. If `.mp3`, use `mp3_player`; otherwise, use `adapter`.  
- The client only calls `player.play(file_name)` and doesn't worry about how the playback is actually implemented.  
</br>

### **Expected Output**</br>
```
[Mp3Player] Playing MP3 file: song.mp3
[AdvancedMediaPlayer] Playing MP4 file: movie.mp4
[AdvancedMediaPlayer] Playing MKV file: trailer.mkv
[MediaAdapter] Error: Unknown format archive.zip
```
- MP3 goes through `Mp3Player`.  
- MP4 and MKV go through `MediaAdapter` + `AdvancedMediaPlayer`.  
- ZIP is unsupported.  
</br>

---

## Pros and Cons of the Adapter Pattern</br>

**Pros**:</br>
1. **Single Interface**: Clients can stick to a single interface while the adapter handles the conversion.  
2. **Reusability**: You don’t need to rewrite or modify the existing advanced classes; you just wrap them.  
3. **Solves Interface Mismatch**: You can integrate third-party libraries or legacy code that doesn’t match your new interface.  
</br>

**Cons**:</br>
1. **Extra Layer**: An adapter adds another layer of indirection, which can affect performance in time-critical applications.  
2. **Complexity**: If you have many different interfaces to adapt, you might end up with multiple adapters.  
</br>

