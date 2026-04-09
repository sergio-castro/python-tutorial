# Virtual Environments

## The Problem: Python Has No Built-in Package Isolation

In Node.js, `npm install` puts dependencies in a local `node_modules/` folder. In Go, modules
are resolved per project. Python, by default, does neither — installing a package places it
**globally** for that Python interpreter. If Project A needs `requests==2.28` and Project B
needs `requests==2.31`, they collide.

**Python needs an extra mechanism to isolate dependencies per project.** That mechanism is
called a **virtual environment**.

## What Is a Virtual Environment?

A virtual environment (venv) is a self-contained directory that holds:

- A copy (or symlink) of the Python interpreter
- Its own package installer
- A `lib/` directory where packages are installed in isolation

The conventional directory name is `.venv/`, placed at the root of your project:

```
my-project/
├── .venv/                  # The virtual environment
│   ├── bin/                # Python interpreter + installed CLI tools
│   │   ├── python          # The isolated Python binary
│   │   ├── pip             # Package installer
│   │   └── hackathon       # Any CLI entry points you define
│   ├── lib/                # Installed packages live here
│   └── pyvenv.cfg          # Metadata (which Python version, etc.)
├── src/                    # Your source code
└── pyproject.toml          # Project manifest
```

## Key Insight

The `.venv/` folder is **disposable**. It is derived entirely from your project manifest and
lock file. You can delete it and recreate it at any time (`uv sync` or `python -m venv .venv`
+ `pip install`). It should never be committed to git.

## Analogy by Language

| Language    | Dependency isolation         | Python equivalent          |
|-------------|------------------------------|----------------------------|
| Node.js     | `node_modules/`              | `.venv/`                   |
| Java/Kotlin | Gradle/Maven local cache     | `.venv/`                   |
| Go          | `go.sum` + module cache      | `uv.lock` + `.venv/`      |
| Rust        | `target/` + Cargo.lock       | `.venv/` + `uv.lock`      |

## How to Create One

```bash
# With uv (recommended for this project)
uv venv

# With the Python standard library (no extra tools needed)
python -m venv .venv
```

Both create a `.venv/` directory. The difference is what you use next to install packages
into it — see [Package Managers](./02-package-managers.md).

---

Next: [Package Managers: pip vs uv](./02-package-managers.md)
