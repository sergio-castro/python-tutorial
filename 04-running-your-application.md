# Running Your Application

Before looking at the different ways to run Python code, let's understand a pattern
you'll see in almost every Python file.

## The `if __name__ == "__main__"` Guard

Open any Python file in this project and you'll find something like this at the bottom:

```python
def say_hello() -> None:
    print("Hello, World!")

if __name__ == "__main__":
    say_hello()
```

`__name__` is a built-in variable that Python sets automatically for every module:

- **When you run the file directly** (`python hello_world.py`) → `__name__` is `"__main__"`
- **When the file is imported** (`from src.hello_world import say_hello`) → `__name__` is `"src.hello_world"`

So the `if` block means: "only execute this code when the file is run directly, not when
it's imported by something else." Without it, importing a function from the file would
also execute the script — usually not what you want.

**A common misconception:** `"__main__"` is a fixed string constant defined by Python.
It has nothing to do with your function names, file names, or any `main()` convention.
A file called `banana.py` with a function called `whatever()` still uses
`if __name__ == "__main__"` — the string never changes.

For Java developers: this is Python's equivalent of `public static void main(String[] args)`,
but implemented as a runtime check rather than a language-level entry point.

## Three Ways to Run Your Code

Now that you understand the guard, here are the three ways to actually run a Python file.

### Option A: `uv run` (Recommended)

```bash
uv run python src/hello_world.py
```

`uv run` automatically uses the project's virtual environment without requiring activation.
It ensures the correct Python interpreter and all installed packages are available.

**This is the recommended approach** because it eliminates an entire class of errors
("why is my package not found?" — because you forgot to activate the venv).

### Option B: Activate the Venv, Then Use Plain `python`

```bash
# Step 1: Activate the virtual environment
source .venv/bin/activate

# Your shell prompt usually changes to indicate activation:
# (2026-hackathon) $

# Step 2: Now 'python' refers to the venv's Python, not the system one
python src/hello_world.py

# Step 3: When done, deactivate
deactivate
```

**What does activation actually do?** It prepends `.venv/bin/` to your shell's `PATH`
environment variable. That's it. When you type `python`, the shell finds `.venv/bin/python`
before the system Python. When you type `hackathon`, it finds `.venv/bin/hackathon`.

This is equivalent to how `nvm use` works in Node.js — it changes which binary `node`
points to.

### Option C: Call the Venv's Python Directly

```bash
.venv/bin/python src/hello_world.py
```

This bypasses activation entirely by using the full path to the interpreter. It's useful
in environments where `uv` is not available — for example, a minimal Docker image where
the venv was built in a previous stage but `uv` was not copied into the final image.
If `uv` is available, Option A is always preferable, including in CI/CD pipelines.

### Summary

| Method                                  | Requires activation? | Best for                |
|-----------------------------------------|----------------------|-------------------------|
| `uv run python src/hello_world.py`     | No                   | Day-to-day development  |
| `source .venv/bin/activate` + `python` | Yes (once per shell) | Interactive shell work  |
| `.venv/bin/python src/hello_world.py`  | No                   | Scripts, CI, Docker     |

## Entry Points: CLI Commands

All three options above run a specific file. But there's a better way: **entry points**
let you define named commands that call a specific function directly.

As explained in [Project Configuration](./03-project-configuration.md), the `[project.scripts]`
section in `pyproject.toml` maps command names to Python functions:

```toml
[project.scripts]
hackathon = "src.hello_world:say_hello"
```

This means: "create a command called `hackathon` that calls `say_hello()` from
`src/hello_world.py`."

When you run `uv sync`, a small wrapper script is generated at `.venv/bin/hackathon`.
You can then run it with:

```bash
# Via uv (no activation needed)
uv run hackathon

# Or activate first, then just use the command name
source .venv/bin/activate
hackathon
```

You can define as many entry points as you want — each pointing to a different function
in a different file:

```toml
[project.scripts]
hackathon    = "src.hello_world:say_hello"
post-request = "src.post_request:main"
dashboard    = "src.dashboard:main"
```

This is the same mechanism behind tools like `pytest`, `flask`, `black`, and even `pip`
itself — they are all entry point scripts generated during installation.

## Quick Reference

```bash
# --- Project setup (one time) ---
uv venv                        # Create virtual environment
uv sync                        # Install all dependencies + your project

# --- Adding dependencies ---
uv add requests                # Add a package (like npm install --save)
uv add --dev pytest            # Add a dev-only dependency

# --- Running code ---
uv run python src/hello_world.py  # Run a script directly
uv run hackathon                  # Run a CLI entry point
uv run pytest                     # Run tests

# --- Manual venv activation (alternative to uv run) ---
source .venv/bin/activate      # Activate
python src/hello_world.py      # Run with plain python
hackathon                      # Run CLI entry point
deactivate                     # Deactivate when done

# --- Housekeeping ---
rm -rf .venv && uv sync        # Nuclear option: recreate venv from scratch
```

---

Previous: [Project Configuration](./03-project-configuration.md) |
Back to [Index](./tutorial.md)
