# Object-Oriented Python for Experienced Developers

You know OOP. This section covers how Python does it differently from Java, C#, or C++,
focusing on the things that will surprise you or trip you up.

## Classes: The Basics

```python
class GraphService:
    def __init__(self, endpoint: str) -> None:
        self._endpoint = endpoint

    def get_graph(self, query: str) -> str:
        return f"Querying {self._endpoint} with {query}"
```

Key differences from Java/C#:

- **`self` is explicit.** Every instance method receives `self` as its first parameter.
  It's the equivalent of `this`, but you must declare it. Python won't do it for you.
- **`__init__` is the constructor** (technically an initializer — the object already exists
  when `__init__` runs, but for practical purposes it's your constructor).
- **No `new` keyword.** You instantiate with `service = GraphService("https://...")`.
- **No access modifiers.** There's no `public`, `private`, or `protected`. See the
  "Visibility" section below.

## Visibility: Convention, Not Enforcement

Python has no access modifiers. Instead, it uses naming conventions:

| Convention      | Meaning                          | Java equivalent     |
|-----------------|----------------------------------|---------------------|
| `name`          | Public                           | `public`            |
| `_name`         | Internal / "please don't touch"  | `private` (by convention) |
| `__name`        | Name-mangled (harder to access)  | `private` (enforced-ish) |

```python
class Example:
    def __init__(self):
        self.public = "anyone can access"
        self._internal = "convention says don't touch from outside"
        self.__mangled = "Python renames this to _Example__mangled"
```

**The single underscore `_` is the most common pattern.** It signals "this is an
implementation detail" without preventing access. Python's philosophy is "we're all
consenting adults" — the convention is respected, but not enforced by the language.

In this project, `self._config` and `self._session` in `GraphService` use this pattern:
they're internal state that callers shouldn't depend on.

## Dataclasses: Python's Answer to Records/DTOs

Writing boilerplate `__init__`, `__eq__`, `__repr__` for simple data-holding classes is
tedious. `@dataclass` generates them from field declarations:

```python
from dataclasses import dataclass

@dataclass
class ConnectionConfig:
    endpoint: str
    username: str
    password: str
```

This single declaration gives you:

- `__init__(self, endpoint, username, password)` — auto-generated constructor
- `__repr__` — readable string representation
- `__eq__` — value-based equality (compares fields, not identity)

### frozen=True: Immutable Dataclasses

```python
@dataclass(frozen=True)
class ConnectionConfig:
    endpoint: str
    username: str
    password: str
```

`frozen=True` prevents field reassignment after construction:

```python
config = ConnectionConfig(endpoint="https://...", username="admin", password="secret")
config.endpoint = "other"  # raises FrozenInstanceError
```

| Language    | Equivalent concept          |
|-------------|-----------------------------|
| Java        | `record`                    |
| Kotlin      | `data class` (with `val`)   |
| C#          | `record` / `readonly struct`|
| TypeScript  | `Readonly<T>` interface     |

### When to Use Dataclasses

- **Use `@dataclass`** for objects that primarily hold data (config, DTOs, value objects).
- **Use regular classes** for objects with complex behavior (services, controllers, etc.).

This is the same distinction you'd make between a Java `record` and a regular `class`.

## Decorators: The `@` Syntax

If you haven't seen it before, `@dataclass` is a **decorator**. Decorators are functions
that wrap other functions or classes. The `@` syntax is just shorthand:

```python
# These two are identical:

@dataclass
class Foo:
    name: str

# is the same as:
class Foo:
    name: str
Foo = dataclass(Foo)
```

You'll see decorators everywhere in Python — `@staticmethod`, `@property`, `@abstractmethod`,
and in frameworks like Flask (`@app.route("/")`) and Dash (`@callback(...)`).

Think of them as Java annotations, except they execute code at class/function definition
time rather than being metadata read by reflection.

## Type Hints: Optional but Recommended

Python is dynamically typed, but supports **type hints** (since Python 3.5):

```python
def get_graph(self, query: str) -> str:
```

Type hints are **not enforced at runtime**. This code runs without error:

```python
def add(a: int, b: int) -> int:
    return a + b

add("hello", "world")  # Returns "helloworld" — no runtime error!
```

They exist for:

- **Documentation** — makes code self-describing
- **IDE support** — autocomplete, refactoring, error highlighting
- **Static analysis** — tools like `mypy` check types before you run the code

| Language    | Type system     | Python equivalent          |
|-------------|-----------------|----------------------------|
| Java        | Static, enforced| Type hints + `mypy`        |
| TypeScript  | Static, compiled| Type hints + `mypy`        |
| JavaScript  | Dynamic, none   | Python without type hints  |

**Best practice:** always write type hints (as we do in this project), and optionally run
`mypy` to catch type errors before runtime.

## Inheritance

Works as you'd expect, with a few syntactic differences:

```python
from abc import ABC, abstractmethod

class BaseService(ABC):
    @abstractmethod
    def execute(self) -> str:
        ...

class ConcreteService(BaseService):
    def execute(self) -> str:
        return "done"
```

- **No `extends` keyword** — the parent goes in parentheses: `class Child(Parent)`
- **Abstract classes** use `ABC` (Abstract Base Class) from the `abc` module
- **Multiple inheritance is supported** — `class Child(Parent1, Parent2)` — unlike Java
- **`super()`** works like Java: `super().__init__()`
- **No interfaces** — use abstract classes or duck typing (if it has the right methods, it works)

## Quick Reference

```python
# --- Regular class ---
class MyService:
    def __init__(self, name: str) -> None:   # constructor
        self._name = name                     # "private" by convention

    def greet(self) -> str:                   # instance method
        return f"Hello, {self._name}"

    @staticmethod
    def utility() -> str:                     # no self, no instance needed
        return "I'm a utility"

# --- Dataclass (for data/config/DTOs) ---
@dataclass(frozen=True)
class Config:
    host: str
    port: int = 8080                          # default value

# --- Instantiation (no 'new' keyword) ---
service = MyService("world")
config = Config(host="localhost")
config_custom = Config(host="localhost", port=3000)
```

---

Previous: [Running Your Application](./04-running-your-application.md) |
Back to [Index](./tutorial.md)
