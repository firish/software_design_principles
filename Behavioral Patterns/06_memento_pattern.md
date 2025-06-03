## What is Memento pattern?

“Memento” packages the internal state of an object inside a snapshot object (the memento) that no other code is allowed to inspect or edit. 
The original object (the originator) can later be told to restore itself from a previously saved memento. 
A third party (the caretaker) merely stores, labels, and hands those mementos back when asked—it never looks inside.

### Why do we bother with it?

By vaulting an object’s private fields into an opaque bundle, we obtain undo/redo, rollback, and time-travel debugging without breaking encapsulation. 
Originators remain the sole authority over their invariants; 
outside code can move them backward or forward in time without learning how they tick.

### Common real-world appearances

- Savegame slots in video games.
- Undo stacks in word processors and IDEs.
- Transactional caches in ORMs (e.g., Hibernate’s “dirty checking” uses snapshots to detect changes).
- Checkpoint/rollback in long-running simulations or workflow engines.

### Upsides, Downsides and Considerations

Upsides:
- Encapsulation is intact. state is copied, never exposed.
- Multiple histories are easy (branching undo stacks, named restore points).
- Caretaker can serialise mementos without knowing their content.

Downsides:
- Memory cost: naïve full snapshots grow linearly with document/game size.
- Performance: large deep-copies are complex unless you delta-encode. But delta-encoding adds code, testing and management complexity.
- Version mismatch: when you evolve the originator’s schema, old mementos may no longer replay cleanly.

Design considerations:
- Decide granularity (entire object vs. incremental diff)
- lifespan (keep N steps or auto-prune),
- safety (clone mutable sub-objects, avoid sharing references).
- if snapshots span processes, add a stable serialisation layer and plan for migration.


### Real World Usecase, code example

Video-game “save slot” (full snapshot)
Imagine a RPG that lets the player quick-save at any moment and quick-load later.
The entire game state—player stats + map position + inventory—is stored in a memento file the engine never peeks inside.

```python
# game_save_memento.py
from dataclasses import dataclass, asdict
import json
import pathlib
from typing import Tuple, List

# ── 1.  Memento object (opaque to outside code) ───────────────────
@dataclass(frozen=True)
class _SaveMemento:
    blob: str                     # JSON string; engine treats as opaque

# ── 2.  Originator: the game world ────────────────────────────────
class Game:
    def __init__(self):
        self.pos: Tuple[int, int] = (0, 0)
        self.hp: int = 100
        self.inventory: List[str] = []

    # gameplay  -----------------------------------------------------
    def move(self, dx, dy):
        self.pos = (self.pos[0] + dx, self.pos[1] + dy)

    def take_damage(self, dmg):
        self.hp = max(0, self.hp - dmg)

    def add_item(self, item):
        self.inventory.append(item)

    # snapshot helpers ---------------------------------------------
    def _create_memento(self) -> _SaveMemento:
        payload = json.dumps(asdict(self))
        return _SaveMemento(payload)

    def _restore(self, m: _SaveMemento):
        state = json.loads(m.blob)
        self.pos = tuple(state["pos"])
        self.hp = state["hp"]
        self.inventory = state["inventory"]

    # convenience
    def __str__(self):
        return f"Pos={self.pos}  HP={self.hp}  Inv={self.inventory}"

# ── 3.  Caretaker: manages numbered save slots ────────────────────
class SaveManager:
    SAVE_DIR = pathlib.Path("saves")
    SAVE_DIR.mkdir(exist_ok=True)

    def __init__(self, game: Game):
        self.game = game

    def save(self, slot: int):
        m = self.game._create_memento()
        (self.SAVE_DIR / f"{slot}.sav").write_text(m.blob)
        print(f" Saved to slot {slot}")

    def load(self, slot: int):
        blob = (self.SAVE_DIR / f"{slot}.sav").read_text()
        self.game._restore(_SaveMemento(blob))
        print(f" Loaded slot {slot}")

# ── 4.  Demo run ─────────────────────────────────────────────────
if __name__ == "__main__":
    g = Game()
    sm = SaveManager(g)

    g.move(3, 4);      g.add_item("sword")
    sm.save(1)         # slot-1 snapshot

    g.move(-2, 1);     g.take_damage(40)
    sm.save(2)         # slot-2 snapshot

    print("Current:", g)    # Pos=(1,5), HP=60, Inv=['sword']
    sm.load(1)
    print("Rewind :", g)    # Pos=(3,4), HP=100, Inv=['sword']
```

Note:
Encapsulation: only Game knows its schema; the .sav file is an opaque blob to the save manager.
Portability: JSON keeps it human-debuggable yet could be swapped for a binary format later.
Versioning: if the schema changes in v2 you can migrate old blobs inside _restore

