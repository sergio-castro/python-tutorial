# Python for Developers Coming from Other Languages

This guide explains how Python projects are structured, how dependencies work, and how to
run applications. It assumes you have experience with other languages (Java, Node.js, Go, etc.)
but zero Python knowledge.

Each topic is covered in its own file. Read them in order if you're starting from scratch,
or jump to the one you need.

Prefer hands-on mini-walkthroughs? See the [labs](./labs/README.md) — short, self-contained
exercises (start with [Hello World](./labs/hello-world/README.md)).

---

## Topics

1. [Virtual Environments](./01-virtual-environments.md)
   Why Python needs them, what `.venv/` is, and how it compares to `node_modules/` or Go modules.

2. [Package Managers: pip vs uv](./02-package-managers.md)
   The standard tool (pip), the modern alternative (uv), how they relate, and when to use which.

3. [Project Configuration: pyproject.toml](./03-project-configuration.md)
   The project manifest — metadata, dependencies, build system, and CLI entry points.

4. [Running Your Application](./04-running-your-application.md)
   Three ways to run Python code, entry points, and a quick-reference cheat sheet.

5. [Testing and Mocking](./05-testing-and-mocking.md)
   pytest, fixtures, parameterized tests, and modern mocking with `unittest.mock` and `pytest-mock`.

6. [Object-Oriented Python](./06-object-oriented-python.md)
   Classes, visibility, dataclasses, decorators, type hints, and inheritance — what's different from Java/C#.
