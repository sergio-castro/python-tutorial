# Package Managers: pip vs uv

## pip — The Standard Tool

`pip` is Python's official package installer. It ships with Python itself and is what most
tutorials, Stack Overflow answers, and documentation reference. **pip is not deprecated.**
It is actively maintained and remains the default tool in the Python ecosystem.

```bash
# Typical pip workflow (for reference — most modern projects use uv instead)
python -m venv .venv                  # Create a virtual environment
source .venv/bin/activate             # Activate it
pip install requests                  # Install a package
pip install -r requirements.txt       # Install from a requirements file
pip freeze > requirements.txt         # Snapshot current packages
```

### pip's Limitations

pip works fine for simple projects, but has friction points at scale:

- **No lock file by default.** `requirements.txt` lists packages but doesn't lock transitive
  dependencies unless you use extra tools like `pip-tools`.
- **Slow.** Dependency resolution and downloads are sequential.
- **Doesn't manage virtual environments.** You create them separately with `python -m venv`,
  then activate, then use pip inside them. Three separate steps.
- **Doesn't manage Python versions.** You need another tool (`pyenv`, system packages) to
  install different Python versions.

## uv — The Modern Alternative

`uv` is a third-party tool (by Astral, the team behind `ruff`) written in Rust. It is
significantly faster than pip and combines several tools into one:

| What you want to do                          | pip ecosystem (multiple tools)   | uv (single tool)     |
|----------------------------------------------|----------------------------------|----------------------|
| Create an empty virtual environment          | `python -m venv .venv`           | `uv venv`            |
| Add a dependency to your project             | `pip install` + edit requirements| `uv add`             |
| Lock all dependency versions                 | `pip-tools` / `pip-compile`      | `uv lock`            |
| Install all dependencies (creates venv too)  | `pip install -e .`               | `uv sync`            |
| Run a command inside the venv                | `source .venv/bin/activate` first| `uv run`             |
| Manage Python versions                       | `pyenv`                          | `uv python install`  |

### Does uv Use pip Behind the Curtains?

No. uv is a **complete reimplementation from scratch** in Rust — its own resolver, downloader,
and installer. It never shells out to pip.

The `uv pip` subcommand exists only for **CLI compatibility**: it accepts the same arguments
as pip (e.g., `uv pip install -r requirements.txt`) so you can migrate existing scripts
without rewriting them. But internally, it's uv's Rust code doing all the work.

### What Tools Does uv Replace?

The Python ecosystem historically required a separate tool for each concern. uv replaces
all of them with a single binary:

| Tool           | Purpose                                | Status            |
|----------------|----------------------------------------|-------------------|
| `pip`          | Install packages                       | Active, standard  |
| `pip-tools`    | Lock dependency versions               | Active            |
| `venv`         | Create virtual environments            | Built into Python |
| `pyenv`        | Install/manage Python versions         | Active            |
| `pipx`         | Install CLI tools in isolated envs     | Active            |
| `poetry`       | All-in-one project management          | Active            |

All of these still work and are maintained. uv is not the "official" replacement for any of
them — it's a faster, unified alternative. The ecosystem hasn't converged on a single tool
(yet), which is why you'll see different projects using different combinations.

### Why Use uv

uv is the recommended choice for new projects because:

- **Single tool** — no need to remember `venv` + `pip` + `pip-tools` separately
- **`uv run`** — runs commands in the venv without activation (fewer mistakes)
- **`uv sync`** — one command to create the venv, install dependencies, and install the project
- **Fast** — dependency resolution and installs are 10-100x faster than pip
- **Lock file** — `uv.lock` is generated automatically, ensuring reproducible builds

### Can I Still Use pip?

Yes. uv creates a standard `.venv/` directory. You can activate it and use pip inside it
if you want — they are fully compatible:

```bash
source .venv/bin/activate
pip install some-package      # Works fine inside a uv-managed venv
```

However, mixing tools can cause the lock file (`uv.lock`) and `pyproject.toml` to get out
of sync with what's actually installed. **Pick one and stick with it.**

## Analogy

Think of it like JavaScript's package managers:

| Python     | JavaScript equivalent |
|------------|-----------------------|
| `pip`      | `npm`                 |
| `uv`       | `pnpm`                |

Both work. `npm` is the default that ships with the runtime. `pnpm` is faster and stricter.
Neither is deprecated. Most new projects prefer the modern option.

---

Previous: [Virtual Environments](./01-virtual-environments.md) |
Next: [Project Configuration](./03-project-configuration.md)
