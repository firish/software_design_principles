# Visitor Pattern: Complete Guide

## What the Visitor Pattern Is

Visitor lets you define a new operation over a family of related objects without editing their classes.

Each object (the element) exposes an `accept(visitor)` method that calls back into the visitor via `visitor.visit_<ElementType>(self)`. That double-dispatch gives the visitor full, type-specific access to every element while the elements remain ignorant of the visitor's purpose.

```kotlin
Directory.accept(v) ──► v.visit_Directory(this)
File.accept(v)      ──► v.visit_File(this)
```

## Why People Use It

### Add Behaviour, Not Properties
When a class hierarchy is stable but you keep inventing new reports, exports, or checks, Visitor avoids sprinkling new methods into every class.

### Cross-cutting Traversals
The visitor can carry context (totals, flags, stacks) across the walk, something a single element method cannot.

### Double Dispatch in Single-dispatch Languages
It resolves "I need code that depends on both this object's type and the visitor's type."

## Where You Meet It in Production

| Domain | Example Visitor |
|--------|----------------|
| **Compilers / linters** | Constant-folding, dead-code elimination, cyclomatic-complexity pass over an AST |
| **Filesystem utilities** | `du`, backup tools, permission auditors walking a directory tree |
| **Serialization** | Write a visitor that turns a scene graph into HTML, Markdown, Protobuf |
| **Finance** | Risk engine walks a portfolio of instruments, computing VAR, Greeks, etc. |

## Pros

- **Open/Closed Principle**: add operations without touching element code
- **Shared housekeeping**: traversal order, memoisation, totals live in one place
- **Multiple passes**: run several visitors in sequence—print, optimise, then execute

## Cons

- **Breaks when you add new element types**: Every visitor needs a new `visit_NewType` method
- **Boilerplate**: `accept()` plus a `visit_*` method for each combination = many lines
- **Harder with optional/union types** in very dynamic domains (JSON blobs, untyped dicts)

## Design Considerations

- Provide a default fallback (`visit_default`) so old visitors don't crash when a new element appears
- Decide between **external traversal** (visitor walks children) or **internal traversal** (each element calls accept on its children)
- In Python, runtimes with `singledispatchmethod` or `match/case` sometimes reduce the need, but Visitor is still handy where keeping operations separate is architecturally cleaner

## Real-World Example: Directory Tree Visitors

**Context:** A backup service must periodically compute (a) total disk usage per directory and (b) disk usage broken down by file owner for billing. We cannot modify File and Directory classes because they're shared across many micro-services.

```python
# file_system_visitor.py
from __future__ import annotations
from dataclasses import dataclass, field
from pathlib import Path
from typing import List, Dict

# ─── element classes (stable, shared library) ──────────────────
class FSNode:
    def accept(self, visitor: "FSVisitor"): ...
    
@dataclass
class File(FSNode):
    path: Path
    size: int
    owner: str
    def accept(self, v: "FSVisitor"): 
        v.visit_file(self)

@dataclass
class Directory(FSNode):
    path: Path
    children: List[FSNode] = field(default_factory=list)
    def accept(self, v: "FSVisitor"):
        v.visit_directory(self)
        for child in self.children:          # internal traversal
            child.accept(v)

# ─── visitor base ──────────────────────────────────────────────
class FSVisitor:
    def visit_file(self, file: File):       
        pass
    def visit_directory(self, dir: Directory): 
        pass

# ─── concrete visitors ────────────────────────────────────────
class DiskUsageVisitor(FSVisitor):
    """Accumulates total bytes under the starting directory."""
    def __init__(self): 
        self.total = 0
    def visit_file(self, file: File):
        self.total += file.size

class OwnerBreakdownVisitor(FSVisitor):
    """Builds owner → bytes map."""
    def __init__(self): 
        self.tally: Dict[str, int] = {}
    def visit_file(self, file: File):
        self.tally[file.owner] = self.tally.get(file.owner, 0) + file.size

# ─── usage example ─────────────────────────────────────────────
tree = Directory(
    Path("/root"),
    [
        File(Path("/root/a.txt"), 1_024, "alice"),
        File(Path("/root/b.txt"), 2_048, "bob"),
        Directory(
            Path("/root/sub"),
            [File(Path("/root/sub/c.log"), 4_096, "alice")]
        ),
    ],
)

du  = DiskUsageVisitor()
own = OwnerBreakdownVisitor()

tree.accept(du)
tree.accept(own)

print(f"Total bytes: {du.total:,}")        # Total bytes: 7,168
print("By owner:", own.tally)               # By owner: {'alice': 5_120, 'bob': 2_048}
```

## Why This Mirrors Production Reality

- **Disk-usage report and billing report** are two brand-new operations added without changing the core File / Directory classes that other services (virus scanner, quota enforcer) already rely on
- **If tomorrow dev-ops introduces Symlink**, only the visitor package needs a new `visit_symlink`; existing backups still compile
- **Internal traversal** (directory loops over children) simplifies visitors—each one focuses solely on per-node logic, not walking
- **The Visitor pattern provides a safe extension point**: you grow the list of analytics and audits over the same immutable hierarchy—exactly what large-scale storage, compiler, and finance platforms do every day


## Updating the Visitor (demo of actual advantage)

```python
from __future__ import annotations
from dataclasses import dataclass, field
from pathlib import Path
from typing import Dict, List

# ───────── Element hierarchy ─────────
class FSNode:
    def accept(self, visitor: "FSVisitor"): ...

@dataclass
class File(FSNode):
    path: Path
    size: int
    owner: str
    def accept(self, v: "FSVisitor"): v.visit_file(self)

@dataclass
class Directory(FSNode):
    path: Path
    children: List[FSNode] = field(default_factory=list)
    def accept(self, v: "FSVisitor"):
        v.visit_directory(self)              # node-specific work
        for ch in self.children:             # INTERNAL traversal
            ch.accept(v)

@dataclass
class Symlink(FSNode):
    path: Path
    target: Path
    def accept(self, v: "FSVisitor"): v.visit_symlink(self)

# ───────── Visitor base ─────────
class FSVisitor:
    # default no-op handlers (fallback for new element types)
    def visit_file(self, f: File):        pass
    def visit_directory(self, d: Directory):  pass
    def visit_symlink(self, s: Symlink):  pass

# ───────── Concrete visitors ─────
class DiskUsageVisitor(FSVisitor):
    def __init__(self): self.total = 0
    def visit_file(self, f: File): self.total += f.size

class OwnerBreakdownVisitor(FSVisitor):
    def __init__(self): self.tally: Dict[str, int] = {}
    def visit_file(self, f: File):
        self.tally[f.owner] = self.tally.get(f.owner, 0) + f.size

class BrokenLinkFinder(FSVisitor):
    """
    EXTERNAL traversal: we perform our own DFS so we can early-exit
    after finding `limit` broken links (no need to visit the rest).
    """
    def __init__(self, limit: int = 10):
        self.limit, self.broken: List[Path] = limit, []
    
    # override directory to *avoid* delegation and run custom walk
    def visit_directory(self, d: Directory):
        stack = [d]
        while stack and len(self.broken) < self.limit:
            node = stack.pop()
            if isinstance(node, Symlink):
                self.visit_symlink(node)
            elif isinstance(node, File):
                self.visit_file(node)  # nothing
            elif isinstance(node, Directory):
                stack.extend(node.children)
    
    def visit_symlink(self, s: Symlink):
        if not s.target.exists():      # pretend check
            self.broken.append(s.path)

# ───────── Demo tree & run ───────
if __name__ == "__main__":
    tree = Directory(
        Path("/root"),
        [
            File(Path("/root/a.txt"), 1024, "alice"),
            Symlink(Path("/root/broken"), Path("/no/such/file")),
            Directory(
                Path("/root/sub"),
                [
                    File(Path("/root/sub/c.log"), 4_096, "alice"),
                    Symlink(Path("/root/sub/ok"), Path("/etc/hosts")),
                ],
            ),
        ],
    )
    
    du  = DiskUsageVisitor()
    own = OwnerBreakdownVisitor()
    bad = BrokenLinkFinder(limit=5)
    
    for v in (du, own, bad):
        tree.accept(v)
    
    print(f"Total bytes: {du.total:,}")     # Total bytes: 5,120
    print("By owner:", own.tally)            # By owner: {'alice': 5120}
    print("Broken links:", bad.broken)       # Broken links: [PosixPath('/root/broken')]
```

# Visitor Pattern: Key Concepts and Design Patterns

## Key Patterns Demonstrated

### Internal vs External Traversal

**Internal Traversal (Standard Approach)**
- **DiskUsageVisitor and OwnerBreakdownVisitor** use **internal traversal**
- `Directory.accept()` handles walking children automatically
- Visitors just process individual nodes - simple and clean
- Good for: operations that need to visit every node, straightforward processing

**External Traversal (Custom Control)**
- **BrokenLinkFinder** uses **external traversal**
- Overrides `visit_directory()` to implement custom DFS
- Provides early exit when limit is reached
- Good for: performance optimization, conditional traversal, complex walking patterns

### Extensibility Without Breaking Changes

**Backward Compatibility**
- **Adding Symlink** required no changes to existing File/Directory classes
- Existing code continues to work unchanged
- New functionality added through new visitor methods only

**Defensive Programming**
- **Default fallback methods** in FSVisitor base class mean old visitors don't crash when new element types appear
- `visit_symlink()` defaults to no-op, so older visitors still function
- Graceful degradation instead of runtime errors

**Parallel Operations**
- **Multiple operations** can run over the same tree structure without interference
- Each visitor maintains its own state independently
- Same data structure supports unlimited new operations

## Why This Mirrors Production Reality

### Real-World Scalability
- **Multiple analytics operations** (disk usage, billing, broken link detection) added without changing core filesystem classes that other services rely on
- Core data structures remain stable while business logic evolves
- Microservices can share common data models without tight coupling

### Architectural Benefits
- **Safe extension point**: you grow the list of analytics and audits over the same immutable hierarchy
- New requirements don't require refactoring existing, tested code
- Operations can be developed and deployed independently

### Performance Considerations
- **Performance optimization**: BrokenLinkFinder can stop early instead of visiting entire tree
- Different visitors can have different performance characteristics
- Traversal strategy can be optimized per use case

### Clean Architecture
- **Separation of concerns**: traversal logic, business logic, and data structure remain decoupled
- Data structure classes focus on structure, not operations
- Business logic concentrated in visitors, not scattered across element classes
- Testing becomes easier - test structure separately from operations

### Production Examples
This pattern is exactly what large-scale systems use:

**Storage Systems**
- File indexing, quota calculation, backup selection, virus scanning
- Same filesystem tree, multiple independent operations

**Compilers**
- AST analysis: type checking, optimization passes, code generation
- Same syntax tree, different compilation phases

**Finance**
- Portfolio analysis: risk calculation, regulatory reporting, performance metrics
- Same instrument hierarchy, different analytical needs

**Content Management**
- Document processing: validation, transformation, publishing, archiving
- Same content tree, multiple workflows
