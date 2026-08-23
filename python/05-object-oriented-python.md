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

For example, `self._endpoint` in `GraphService` above uses this pattern:
it's internal state that callers shouldn't depend on.

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

## Functions Are Values

Before decorators, the idea they rest on: in Python a function is an ordinary object. You can
assign it, store it in a list, pass it to another function, and return it from one.

```python
def double(n: int) -> int:
    return n * 2

f = double          # no parentheses — this is the function itself
print(f(21))        # 42

print(double(21))   # calling it
print(double)       # <function double at 0x...> — the object
```

That distinction is the one to hold on to: **`double` is the function, `double()` runs it.** A
missing or extra pair of parentheses is the most common mistake in this area, and it usually
fails somewhere far away from the line that caused it.

### `lambda`: a function without a name

`lambda` is a **reserved keyword** — one of Python's 35, alongside `def`, `class`, and `return` —
and the expression it introduces creates a function:

```python
lambda n: n * 2
```

Two consequences of it being reserved. You cannot use it as a name: `lambda = 5` is a
`SyntaxError`, not a shadowing warning. And where the word is genuinely the right one — a
wavelength, a decay constant — the convention from PEP 8 is a trailing underscore, `lambda_`.

What it produces is **an ordinary function**, not a distinct kind of object:

```python
f = lambda n: n * 2
def g(n): return n * 2

type(f) is type(g)   # True — both are <class 'function'>
f.__name__           # '<lambda>'
g.__name__           # 'g'
```

There is no "lambda type" to learn. The only trace of how it was written is the name, which is
why a stack trace through one says `<lambda>` and tells you nothing about where it came from.

Read the expression as: *"a function taking `n`, returning `n * 2`"*. Everything before the colon
is the parameter list; the single expression after it is the return value, with no `return`
keyword.

The point is not brevity but **position**: a `lambda` can appear in the middle of an argument
list, where a `def` cannot. So when a library asks you for a function, you can supply one on the
spot instead of naming it first:

```python
names = ["Bruce", "alice", "Carla"]
names.sort(key=lambda name: name.lower())     # sort case-insensitively
```

`key=` here is not receiving a value; it is receiving a function that `sort` will call once per
element. That is the shape you will meet constantly in configuration APIs — you hand over a
function, and the framework decides when to call it and what to pass in.

Limits, by design: a lambda holds **one expression**, so no statements, no `if/else` blocks (the
`a if cond else b` expression is fine), no loops, and no annotations. Anything longer wants a
`def` — which also gets you a real name in the traceback instead of `<lambda>`.

For Java developers this is the same idea as `n -> n * 2`, and a named function passed by
reference is `Foo::bar`. Python simply makes no distinction between the two: both are just the
object.

### Typing one

A parameter that takes a function is annotated `Callable`:

```python
from collections.abc import Callable

def apply_twice(f: Callable[[int], int], value: int) -> int:
    return f(f(value))

apply_twice(lambda n: n * 2, 5)     # 20
```

`Callable[[int], int]` reads as "takes one `int`, returns an `int`" — the equivalent of Java's
`Function<Integer, Integer>`.

## Decorators: The `@` Syntax

Now the payoff. If you haven't seen it before, `@dataclass` is a **decorator**. Decorators are
functions that wrap other functions or classes. The `@` syntax is just shorthand:

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

**Best practice:** always write type hints, and optionally run `mypy` to catch type errors
before runtime.

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

# --- Functions as values ---
f = double                      # the function object; double() would call it
sorted(xs, key=lambda x: x.name)  # inline function, one expression, implicit return

# --- Static method call (no instance needed) ---
MyService.utility()          # "I'm a utility"
```

---

Previous: [Running Your Application](./04-running-your-application.md) |
Next: [Testing and Mocking](./06-testing-and-mocking.md)
