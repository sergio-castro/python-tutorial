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

[project.optional-dependencies]
excel = ["openpyxl>=3.1"]

[project.scripts]
my-app = "my_app.main:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/my_app"]
```

### [project] — Project Metadata

The core section. Defined by PEP 621 (a Python standard, not tool-specific).

- **`name`** — The package name.
- **`version`** — Semantic version.
- **`requires-python`** — Minimum Python version (like `engines` in package.json).
- **`dependencies`** — Runtime dependencies. When you run `uv add requests`, it adds
  `"requests>=2.31"` here.

### [project.optional-dependencies] — Extras

An **extra** is a named, opt-in bundle of dependencies. The package author declares it; the
user decides at install time whether they want it.

The problem it solves: suppose your library can export reports to Excel, but only some users
need that. Put `openpyxl` in `dependencies` and *everyone* downloads it — including the users
who never touch the Excel code. Leave it out entirely and the Excel users have to somehow
know they need to install it themselves.

An extra is the middle ground. You declare the dependency, give the bundle a name, and let
the user ask for it by name:

```toml
[project]
name = "my-app"
dependencies = ["requests>=2.31"]     # always installed

[project.optional-dependencies]
excel = ["openpyxl>=3.1"]             # installed only on request
pdf   = ["reportlab>=4.0"]
```

Each **key** is the name of an extra. Each **value** is the list of packages pulled in when
someone opts into that name.

You have almost certainly installed an extra already without knowing what it was called —
`uvicorn[standard]`, `pandas[excel]`, and `httpx[http2]` are all this mechanism.

#### Installing an Extra

Name the extra in square brackets after the package:

```bash
uv add "my-app[excel]"
pip install "my-app[excel]"
```

**Always quote it.** In bash and zsh, `[` and `]` are glob characters. Unquoted,
`pip install my-app[excel]` can fail or expand into something you didn't intend.

Extras are **additive**, not mutually exclusive. Ask for several and you get the union of all
of them, on top of the base `dependencies`:

```bash
uv add "my-app[excel,pdf]"            # requests + openpyxl + reportlab
```

#### Composing Extras from Other Extras

An extra can list *its own package with another extra* as a dependency. This looks strange
the first time you see it:

```toml
[project.optional-dependencies]
excel = ["openpyxl>=3.1"]
pdf   = ["reportlab>=4.0"]
all   = ["my-app[excel]", "my-app[pdf]"]
```

`my-app` depends on `my-app`. This is legal and idiomatic — the resolver recognizes it is
already resolving `my-app` and simply folds in the requirements of the named extras. Read
`all = ["my-app[excel]", "my-app[pdf]"]` as **"`all` means everything `excel` means, plus
everything `pdf` means."**

The alternative is copying the package lists into every extra that needs them:

```toml
[project.optional-dependencies]
excel = ["openpyxl>=3.1"]
pdf   = ["reportlab>=4.0"]
all   = ["openpyxl>=3.1", "reportlab>=4.0"]    # duplicated by hand
```

This works — until the first time someone edits it. Say a bug fix in `openpyxl` 3.2 is
needed, so the minimum gets bumped:

```toml
[project.optional-dependencies]
excel = ["openpyxl>=3.2"]                      # bumped
pdf   = ["reportlab>=4.0"]
all   = ["openpyxl>=3.1", "reportlab>=4.0"]    # forgotten — still >=3.1
```

Now `my-app[excel]` and `my-app[all]` disagree. A user who installs `my-app[all]` can end up
with `openpyxl` 3.1 and hit the exact bug the bump was meant to avoid. Nothing errors, no
tool warns, and `all` quietly stops being a superset of the other extras.

With the composed version, there is only one place where `openpyxl` is named. Editing
`excel` fixes `all` automatically, because `all` never had its own copy to forget.

#### Empty Extras

You will also see extras defined as an empty list:

```toml
[project.optional-dependencies]
core = []
```

`my-app[core]` installs exactly the same thing as plain `my-app`. That is not a mistake. An
empty extra earns its place for three reasons:

- **It is a stable name for an intent.** Users write `[core]` today. If the package later
  needs a real dependency for that feature, the maintainer adds it to the list and every user
  picks it up on their next upgrade — without changing their install command.
- **It documents what the package offers.** It sits in the same namespace as its richer
  siblings, so `core`, `core-excel`, and `core-pdf` read as a set of choices.
- **It gives other extras something to build on**, using the composition idiom above.

#### Equivalents in Other Languages

| Language    | Closest equivalent                                            |
|-------------|---------------------------------------------------------------|
| Rust        | Cargo features — `cargo add serde --features derive`          |
| Java/Maven  | `<optional>true</optional>` dependencies, or build profiles   |
| Node.js     | No real equivalent (optional peer deps behave differently)    |
| Go          | No equivalent — build tags are compile-time, not install-time |

Cargo features are the closest match: a named, opt-in, additive set declared by the author
and selected by the consumer.

#### Extras vs. Dependency Groups

Extras are **published**. They are baked into the built package's metadata, so anyone who
installs your package from PyPI can request them.

That makes extras the wrong tool for dependencies only *you* need — `pytest`, `ruff`, `mypy`.
Your users should never be able to install those. Modern projects use `[dependency-groups]`
(PEP 735) instead, which stays local and is never published:

```toml
[dependency-groups]
dev = ["pytest>=8.0", "ruff>=0.6"]
```

`uv sync` installs the `dev` group by default; `uv sync --no-dev` skips it. Older projects
predate PEP 735 and put dev tools in an extra named `dev`, so you will see both in the wild.

**Rule of thumb:** if a *consumer* of your package might want it, make it an extra. If only a
*contributor* to your package needs it, make it a dependency group.

### [project.scripts] — CLI Entry Points

Maps command names to Python functions:

```toml
my-app = "my_app.main:main"
```

This means: "create a CLI command `my-app` that calls the `main()` function from
the module `my_app.main` (i.e., the file `src/my_app/main.py`)."

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
packages = ["src/my_app"]
```

Tells hatchling which directories to ship. Each entry is a **path to an importable package**,
and the last component of that path becomes the name people import:

```
my-app/
├── src/
│   └── my_app/          # packages = ["src/my_app"]  →  import my_app
│       ├── __init__.py
│       └── main.py
└── pyproject.toml
```

The `src/` prefix is stripped at build time — it never appears in an import. Without this
setting, hatchling looks for a directory matching the project name at the top level, which
doesn't exist in a `src/` layout.

Note that `[tool.<name>]` is a namespace reserved for one specific tool. This block is read
only by hatchling; other build backends configure the same thing with different keys. The
`[project]` and `[build-system]` sections above are standardized, so they carry over
unchanged if you ever switch backends.

#### Why `src/`?

Directories in your working directory are importable whether or not your project is
installed. With a flat layout (`my_app/` at the top level), `import my_app` silently picks up
the local folder, so your tests can pass against source files that were never actually
packaged. Putting the code under `src/` makes that impossible: nothing is importable until
the package is installed, so you always test what you ship.

#### Distribution Name vs. Import Name

These are two different names, and they're allowed to differ:

| Name              | Value    | Where it appears                        |
|-------------------|----------|-----------------------------------------|
| Distribution name | `my-app` | `[project] name`, `uv add my-app`, PyPI |
| Import name       | `my_app` | `import my_app`, the directory on disk  |

Hyphens are legal in distribution names but illegal in Python identifiers, so the convention
is hyphens on PyPI and underscores in code. They can diverge further — `pip install pillow`
gives you `import PIL`.

#### Why Is `packages` a List?

Because one installed distribution can expose more than one top-level import name. Most
projects have exactly one entry, but the plural case is real: the `attrs` distribution ships
both `import attr` and `import attrs`, and `setuptools` ships both `setuptools` and
`pkg_resources`.

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
