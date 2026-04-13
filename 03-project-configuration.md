# Project Configuration: pyproject.toml

## What Is pyproject.toml?

This is the project manifest — the single file that defines your project's metadata,
dependencies, build configuration, and tooling settings. It is the Python equivalent of:

| Language    | Equivalent file     |
|-------------|---------------------|
| Node.js     | `package.json`      |
| Rust        | `Cargo.toml`        |
| Go          | `go.mod`            |
| Java/Gradle | `build.gradle.kts`  |

## Anatomy of a pyproject.toml

```toml
[project]
name = "my-app"
version = "0.1.0"
description = "A short description of your project"
requires-python = ">=3.9"
dependencies = []

[project.scripts]
my-app = "src.main:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src"]
```

### [project] — Project Metadata

The core section. Defined by PEP 621 (a Python standard, not tool-specific).

- **`name`** — The package name.
- **`version`** — Semantic version.
- **`requires-python`** — Minimum Python version (like `engines` in package.json).
- **`dependencies`** — Runtime dependencies. When you run `uv add requests`, it adds
  `"requests>=2.31"` here.

### [project.scripts] — CLI Entry Points

Maps command names to Python functions:

```toml
my-app = "src.main:main"
```

This means: "create a CLI command `my-app` that calls the `main()` function from
the module `src.main` (i.e., the file `src/main.py`)."

When you run `uv sync`, a wrapper script is generated at `.venv/bin/my-app` that
performs this import and function call. This is how tools like `pytest`, `flask`, and
`black` work — they are all entry points.

See [Running Your Application](./04-running-your-application.md) for more on this.

### [build-system] — How to Build the Project

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

This tells Python (and uv) which tool to use to build and install the project.
`hatchling` is one of several build backends. Others include `setuptools` (the old default),
`flit`, and `maturin` (for Rust extensions). You'll rarely need to change this.

### [tool.hatch.build.targets.wheel] — Source Layout

```toml
[tool.hatch.build.targets.wheel]
packages = ["src"]
```

Tells hatchling where to find your Python packages. Without this, it looks for a directory
matching the project name (e.g., `my_app/`), which doesn't exist if you use a `src/`
layout.

## Historical Context

Before `pyproject.toml`, Python projects used some combination of:

- `setup.py` — a Python script that defined metadata imperatively
- `setup.cfg` — a config-file version of setup.py
- `requirements.txt` — a flat list of dependencies (no metadata, no build config)

You'll still see these in older projects and tutorials. Modern Python projects should use
`pyproject.toml` exclusively. If you see a tutorial using `setup.py`, the concepts still
apply — just know that `pyproject.toml` is the current standard (PEP 621, finalized 2021).

---

Previous: [Package Managers](./02-package-managers.md) |
Next: [Running Your Application](./04-running-your-application.md)
