# Linting and Formatting

In a Java project you probably wire up Spotless for formatting, Checkstyle for style rules, and
SpotBugs or Error Prone for bug patterns — three plugins, three configs. Python's history was
worse: `black` for formatting, `isort` for imports, `flake8` plus a dozen plugins for linting,
`pyupgrade` for modernising syntax, `autoflake` for dead imports. Five tools that had to be
kept in agreement.

**Ruff** replaced all of them with one binary. This chapter covers what it does, the difference
between its two halves, and how to get the `spotless:apply` experience you are used to.

## The Landscape

| Tool | What it is | Should you use it? |
|-----------------------|-------------------------------------------------|-------------------------|
| `black` | The formatter that ended Python's style debates | No — `ruff format` is a drop-in |
| `isort` | Import sorter | No — Ruff's `I` rules |
| `flake8` + plugins | The classic linter, plugin-based | No |
| `pylint` | Deeper, slower linter | Rarely — a few checks Ruff lacks |
| `autoflake`, `pyupgrade` | Single-purpose fixers | No |
| **`ruff`** | Linter **and** formatter, all of the above | **Yes** |
| `mypy`, `pyright`, `ty` | Type checkers | Yes — a different job, see below |

Ruff is written in Rust by Astral, the same team behind `uv`
([Package Managers](./02-package-managers.md)), and it is fast enough — tens of thousands of
files per second — that you stop thinking about when to run it.

The one thing to be clear about from the start: **Ruff is two tools behind one command.**
`ruff format` and `ruff check` are separate, configured separately, and run separately.

## Formatting vs Linting

This distinction trips up people who arrive from ecosystems where one tool did both.

| | `ruff format` | `ruff check` |
|--------------------|--------------------------------------|---------------------------------------|
| Question it answers | How should this code *look*? | Is this code *wrong* or *suspect*? |
| Operates on | Whitespace, line breaks, quotes, parens | Names, imports, control flow, API use |
| Configurable? | Barely — a handful of knobs | ~950 rules, you pick |
| Can it disagree with a colleague? | No — one canonical output | Yes — that's what config is for |
| Java analogue | Spotless / google-java-format | Checkstyle + SpotBugs + Error Prone |

A formatter is a **normaliser**. It parses your file into a syntax tree and prints it back out
using fixed rules, so the output depends only on the code's *structure*, never on how you typed
it. Two developers who format the same file get byte-identical results, which is the entire
point: formatting stops being a code-review topic.

A linter is a **detector**. It looks for patterns that are legal Python but probably a mistake —
an import nobody uses, a mutable default argument, a variable that shadows a builtin. Some of
those it can also fix, which is where the line gets blurry.

Two consequences worth internalising:

- **`ruff format` never changes what your code does.** It only moves characters the parser
  ignores. `ruff check --fix` *does* rewrite code, which is why it distinguishes safe from
  unsafe fixes (below).
- **Import sorting is a lint rule, not formatting.** `ruff format` will not touch the order of
  your imports. Sorting is rule `I001`, and it is applied by `ruff check --fix`. If you came
  from `black` + `isort` and thought of both as "formatting", this is the surprise.

## Setup

```bash
uv add --dev ruff
```

That lands in the contributor-only `[dependency-groups]` section from
[Project Configuration](./03-project-configuration.md) — consumers of your package have no use
for your linter:

```toml
[dependency-groups]
dev = ["pytest>=8.0", "ruff>=0.16"]
```

Then run it through `uv run`, exactly like `pytest`:

```bash
uv run ruff format .
uv run ruff check .
```

You can also install Ruff globally with `uv tool install ruff`, or run a one-off copy with
`uvx ruff check`. Convenient, but keep the project-local dev dependency as the source of truth:
formatter output can shift slightly between releases, and you want every developer and CI to
agree. If you need byte-for-byte reproducibility, pin exactly (`ruff==0.16.1`) rather than with
`>=`.

*(For brevity the rest of this chapter writes `ruff` instead of `uv run ruff`.)*

## The Commands You Actually Type

```bash
ruff format .                 # rewrite files in place
ruff format --check .         # exit 1 if anything would change; changes nothing
ruff format --diff .          # show the diff it would apply

ruff check .                  # report violations
ruff check --fix .            # report, and apply the safe fixes
ruff check --diff .           # preview what --fix would do
ruff check --fix --show-fixes # list every fix as it is applied
ruff check --watch .          # re-run on save
ruff check --statistics .     # counts per rule — the adoption dashboard
```

Two discovery commands worth remembering, because rule codes are opaque at first:

```bash
ruff rule F401                # what does F401 mean, why, and is it fixable
ruff linter                   # list every rule prefix and the tool it came from
```

## Making It Work Like `spotless:apply`

There is no single "fix everything" subcommand. The idiom is two commands, in this order:

```bash
ruff check --fix .            # 1. lint fixes: sort imports, drop dead imports, modernise
ruff format .                 # 2. then normalise layout
```

**Order matters.** Lint fixes edit code and can leave a line too long or a call awkwardly
broken; running the formatter afterwards tidies that up. Doing it the other way round leaves
the file formatted-then-edited. Ruff's own pre-commit configuration uses this order.

The `spotless:check` equivalent, for CI:

```bash
ruff check .
ruff format --check .
```

Since you will type the pair constantly, wrap it. Unlike npm or Gradle, `uv` has no built-in
task runner, so the two common options are a `Makefile`:

```make
.PHONY: fix check
fix:                          # ≈ mvn spotless:apply
	uv run ruff check --fix .
	uv run ruff format .

check:                        # ≈ mvn spotless:check
	uv run ruff check .
	uv run ruff format --check .
	uv run pytest
```

...or `poethepoet`, which keeps the tasks inside `pyproject.toml`:

```toml
[tool.poe.tasks]
fix = ["_lint_fix", "_format"]
_lint_fix = "ruff check --fix ."
_format = "ruff format ."
```

```bash
uv run poe fix
```

Either way, the real enforcement point is a pre-commit hook and CI — both covered below.

## Configuration

Everything lives under `[tool.ruff]` in `pyproject.toml` — the same `[tool.*]` convention as
`[tool.pytest.ini_options]`. A realistic starting point:

```toml
[tool.ruff]
line-length = 100
src = ["src", "tests"]              # what counts as "your" code, for import sorting
extend-exclude = ["generated"]

[tool.ruff.lint]
select = [
    "E", "W",   # pycodestyle — whitespace and style
    "F",        # Pyflakes — unused imports, undefined names, dead variables
    "I",        # isort — import ordering
    "UP",       # pyupgrade — modernise syntax for your Python version
    "B",        # flake8-bugbear — real bug patterns
    "SIM",      # flake8-simplify — needless complexity
    "C4",       # flake8-comprehensions
    "N",        # pep8-naming
    "TID",      # flake8-tidy-imports
    "PTH",      # flake8-use-pathlib — os.path → pathlib
    "PT",       # flake8-pytest-style
    "RUF",      # Ruff's own rules
]
ignore = ["E501"]                   # line length is the formatter's job

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101"]               # assert is the point in tests
"__init__.py" = ["F401"]            # re-exports look unused

[tool.ruff.lint.isort]
known-first-party = ["my_app"]
combine-as-imports = true

[tool.ruff.lint.flake8-tidy-imports]
ban-relative-imports = "all"

[tool.ruff.format]
docstring-code-format = true        # format code examples inside docstrings too
```

The keys that matter:

- **`select`** — which rules are on. Configure nothing and you still get Ruff's curated default
  set. That default has moved: for years it was a deliberately tiny `E4`, `E7`, `E9`, `F`, while
  Ruff 0.16 enables several hundred rules — including import sorting (`I001`), `B`, `UP`, `SIM`,
  and `RUF` — and deliberately leaves out almost every pycodestyle whitespace rule, because
  those are the formatter's job. Print the resolved list for your version with
  `ruff check --show-settings some_file.py`.
  `select` **replaces** the default set, so an explicit list can quietly turn *off* rules you
  were getting for free — that is the argument for `extend-select`, which adds to the default
  instead. Either is defensible; pinning it explicitly is what stops an upgrade from moving
  your goalposts.
- **`ignore`** — turn off individual rules or whole prefixes that `select` pulled in.
- **`line-length`** — default `88`, inherited from `black`. Used by both halves: the formatter
  wraps at it, `E501` complains past it.
- **`target-version`** — which Python syntax to target. Omit it and Ruff reads
  `requires-python` from `[project]`, which is what you want. It drives `UP`: with `py310+`,
  Ruff will rewrite `Optional[str]` to `str | None`.
- **`src`** — how Ruff decides an import is first-party rather than third-party. Get this wrong
  in a `src/` layout and your own package sorts into the wrong group.
- **`per-file-ignores`** — relax rules by path. Tests and `__init__.py` almost always need it.
- **`extend-exclude`** — add to the default exclusions. Ruff already skips `.venv`,
  `.git`, build directories, and anything in `.gitignore`.

**Do not write `select = ["ALL"]`.** It opts you into rules that don't exist yet, so a Ruff
upgrade turns into a CI outage — and several rule families actively contradict each other.
Start with the list above, then add prefixes deliberately.

If you would rather keep it out of `pyproject.toml`, a standalone `ruff.toml` (or `.ruff.toml`)
takes the same keys without the `tool.ruff` prefix. In a monorepo, subdirectory configs can
inherit with `extend = "../ruff.toml"`.

### A Note on Conflicting Rules

A few lint rules encode formatting opinions and will fight the formatter — the whitespace rules
(`E1xx`, `W191`), the quote rules (`Q`), and the trailing-comma rules (`COM812`, `COM819`).
`ruff format` prints a warning when it sees one selected:

```
warning: The following rule may cause conflicts when used with the formatter: `COM812`.
```

Let the formatter win and ignore the rule.

## Import Hygiene

### Import Ordering

`I001` is in Ruff 0.16's default rule set, so `ruff check --fix` already sorts imports with no
configuration at all. Select `I` anyway — it documents the intent, survives a change of
defaults, and pulls in `I002` as well:

```bash
ruff check --select I --fix .
```

Ruff groups imports the way `isort` did — `__future__`, standard library, third-party,
first-party, then local relative imports — with a blank line between groups and alphabetical
order inside each:

```python
from __future__ import annotations

import json
from pathlib import Path

import pytest
import requests

from my_app.orders import line_total
from my_app.pricing import fetch_rate

from .helpers import build_order
```

That fifth group is local relative imports — shown for completeness, though the config above
bans them via `TID252`. `I001` is the "imports are not sorted" rule, and its fix is safe.
Useful knobs:

```toml
[tool.ruff.lint.isort]
known-first-party = ["my_app"]      # if src = [...] isn't enough
force-single-line = false           # true = one import per line, easier diffs
combine-as-imports = true           # keep `from x import a as b, c` on one line
required-imports = ["from __future__ import annotations"]   # enforced by I002
```

That last one is a nice trick: `I002` will *add* a missing import to every file, which is how
projects roll out `from __future__ import annotations` everywhere in one commit.

### Removing Unused Imports

`F401` (unused import) and `F841` (assigned-but-never-used local) come from Pyflakes and are in
the default rule set — you get them even with no config. `--fix` deletes an unused import
outright; `F841` only reports, because deleting `unused = expensive_call()` would drop the call
(see [Safe vs Unsafe Fixes](#safe-vs-unsafe-fixes)).

The exception is `__init__.py`, where an "unused" import is usually a deliberate re-export.
Three ways to say so, in order of preference:

```python
from my_app.orders import line_total as line_total     # explicit re-export (PEP 484)

__all__ = ["line_total"]                               # or name it in __all__
```

```toml
[tool.ruff.lint.per-file-ignores]
"__init__.py" = ["F401"]                               # or opt the file out entirely
```

The first two communicate intent to type checkers as well, so prefer them over blanket
suppression — and they are what Ruff's own fix suggests when it finds an unused import in an
`__init__.py`. Everywhere else, the `F401` fix is safe and simply deletes the line.

### Banning Star Imports

`from my_app.orders import *` is flagged by two Pyflakes rules:

- **`F403`** — a star import is present.
- **`F405`** — this name *may* be coming from a star import, so nothing can verify it exists.

Both are in the default set. Neither is auto-fixable, and that is not a gap: Ruff cannot know
which of the module's names you actually meant to use. You expand them by hand, or let your
editor's "optimize imports" action do it.

The real cost of a star import is that it defeats static analysis. Once one is present, Ruff
can no longer tell an undefined name from an imported one, so `F821` (undefined name) goes
quiet across the whole file — you lose a genuinely valuable check to save a few keystrokes.
Related rules worth selecting:

```toml
[tool.ruff.lint]
extend-select = [
    "TID252",   # no relative imports (ban-relative-imports)
    "ICN",      # enforce conventional aliases: import numpy as np
    "A",        # no shadowing builtins — `list = [...]`, `id = 3`
]
```

## Other Fixes You Get for Free

Worth knowing about, because several catch mistakes a Java background actively predisposes you
to:

| Rule | Catches | Fix |
|-----------|--------------------------------------------------------------|---------|
| `B006` | Mutable default argument — `def f(items=[])` | Unsafe |
| `B008` | Function call in a default argument | None |
| `UP045` | `Optional[str]` → `str \| None` | Safe |
| `UP007` | `Union[int, str]` → `int \| str` | Safe |
| `UP031` | `"%s" % x` → f-string | Safe |
| `UP006` | `typing.List[int]` → `list[int]` | Safe |
| `SIM108` | `if/else` block that is really a ternary | Safe |
| `C403` | `set([x for x in y])` → a set comprehension | Safe |
| `PTH123` | `open(path)` → `Path(path).open()` | Safe † |
| `RUF100` | A `# noqa` that no longer suppresses anything | Safe |
| `ERA001` | Commented-out code | None |

† Downgraded to unsafe when the rewrite would discard a comment.

The `UP` rewrites are driven by `target-version`: Ruff only offers `str | None` if your
`requires-python` allows 3.10+.

`B006` deserves special attention. Python evaluates default arguments **once**, at function
definition time, so a mutable default is shared across every call — a bug with no Java
equivalent and no obvious symptom until it bites. Note that Ruff's fix is *unsafe*: rewriting
the signature to `items=None` plus an `if items is None:` body is almost certainly what you
meant, but it is a behaviour change, so `--fix` alone will not do it.

## Safe vs Unsafe Fixes

Every fixable rule is marked either **safe** or **unsafe**, and `--fix` applies only the safe
ones. The distinction is:

- **Safe** — preserves behaviour *and* intent. Sorting imports (`I001`). Deleting an unused
  import (`F401`). Rewriting `Optional[str]` to `str | None` (`UP045`).
- **Unsafe** — probably right, but could change behaviour or discard something you meant.
  Deleting an unused local variable (`F841`) is unsafe because `unused = expensive_call()` has a
  side effect the fix would silently remove. Same for `B006`, which restructures a signature, and
  for any fix that would drop a comment.

To get the rest, opt in explicitly — and look before you leap:

```bash
ruff check --diff --unsafe-fixes .     # preview
ruff check --fix --unsafe-fixes .      # apply
```

Ruff's summary line tells you when it is holding fixes back — `1 hidden fix can be enabled with
the --unsafe-fixes option` — so you are never silently missing them. You can also flip it in
config, or restrict fixing to a specific set of rules:

```toml
[tool.ruff]
unsafe-fixes = true

[tool.ruff.lint]
unfixable = ["F401"]          # tell me about dead imports, don't delete them for me
```

Avoid `fix = true` in config. It makes a bare `ruff check` rewrite your files, which is
surprising when you only meant to look.

## Suppressing a Rule

```python
import tomllib  # noqa: F401              — this one line, this one rule
```

```python
# ruff: noqa: F401                        — whole file, one rule (put it at the top)
# ruff: noqa                              — whole file, everything
```

Always name the code. A bare `# noqa` hides future problems too, and `PGH004` exists to flag
exactly that. Pair it with `RUF100`, which reports `noqa` comments that have become
unnecessary, so suppressions get cleaned up instead of accumulating. Think
`@SuppressWarnings("unchecked")`, with the same etiquette: narrow, and justified in a comment
when it isn't obvious.

Adopting Ruff on an existing codebase? `ruff check --add-noqa .` inserts a suppression comment
on every current violation. You get a clean build immediately, and a grep-able list of debt to
work through.

## Editor Integration

Ruff ships a language server (`ruff server`), so editor support is a thin plugin rather than a
reimplementation. This is where the day-to-day experience is won: format and fix on save, and
you will rarely run the CLI by hand.

**PyCharm / IntelliJ** — install the Ruff plugin from Settings → Plugins. Point it at the
project interpreter's binary (`.venv/bin/ruff`) so the IDE and CI use the same version, and
enable "Use ruff format" so ⌥⌘L reformats with Ruff instead of PyCharm's own formatter. The
plugin adds a run-on-save option; PyCharm's built-in Settings → Tools → Actions on Save panel
can also trigger "Reformat code" and "Optimize imports".

**VS Code** — install the Ruff extension, then:

```json
{
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.fixAll.ruff": "explicit",
      "source.organizeImports.ruff": "explicit"
    }
  }
}
```

Note the two separate code actions — `source.fixAll` runs lint fixes, `source.organizeImports`
sorts imports. That mirrors the CLI's split exactly, and it is the closest thing to
`spotless:apply` firing on every ⌘S.

## Enforcement: pre-commit

Editor settings are per-developer and easy to forget. A pre-commit hook is the thing that
actually keeps the repository clean — the equivalent of binding Spotless to a Maven phase.

`.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.16.1                # keep in sync with your dev dependency
    hooks:
      - id: ruff-check          # older configs call this hook `ruff`
        args: [--fix]
      - id: ruff-format
```

```bash
uv add --dev pre-commit
uv run pre-commit install       # wire it into .git/hooks
uv run pre-commit run --all-files   # the initial sweep
uv run pre-commit autoupdate    # bump the pinned revs later
```

Note the hook order — `ruff-check --fix` before `ruff-format`, the same reasoning as the CLI.

One behaviour that confuses everyone once: when a hook **rewrites** a file, the commit is
aborted. That is not a failure to fix, it is the hook telling you it changed something you
haven't staged. Look at the diff, `git add`, commit again.

## Enforcement: CI

```yaml
- run: uv run ruff check --output-format=github .
- run: uv run ruff format --check --diff .
```

`--output-format=github` emits workflow annotations, so violations appear inline on the pull
request diff instead of buried in log output. (`gitlab`, `json`, and `sarif` are also
available.)

**Never run `--fix` in CI.** It would "pass" while the committed code stays broken. CI verifies;
developers fix. And add `.ruff_cache/` to `.gitignore`.

## Adopting Ruff on an Existing Project

The order that avoids a 4,000-line review:

1. **`ruff check --statistics .`** — see what you're dealing with, ranked by rule.
2. **Format in one isolated commit.** `ruff format .` and nothing else. Then add that commit's
   SHA to `.git-blame-ignore-revs` so `git blame` skips past it:
   ```
   # .git-blame-ignore-revs
   a1b2c3d4  # ruff format: whole-repo reformat, no behaviour change
   ```
   Configure it once with `git config blame.ignoreRevsFile .git-blame-ignore-revs`.
3. **Turn on rules in waves.** Start with the defaults plus `I`, fix, commit. Then `UP`, then
   `B`, then the rest.
4. **`ruff check --add-noqa .`** for whatever is left, so the build is green while the debt
   stays visible.

## What Ruff Does *Not* Do

**Type checking.** This is the big one. Ruff parses each file mostly in isolation; it does not
build a project-wide type model, so it cannot tell you that you passed an `int` where a `str`
was expected. That is `mypy` or `pyright` (or `ty`, Astral's much newer type checker). The `ANN`
rules only check that annotations are *present*, never that they are *correct*.

```bash
uv add --dev mypy
uv run mypy src
```

Run both: Ruff for style and bug patterns, a type checker for types. They overlap very little.

**Dependency and vulnerability scanning.** The `S` rules are a port of `bandit` and catch
insecure *code patterns* — `S105` (a password-looking string literal), `S603` (unvalidated input
to `subprocess`), and `S101`, which flags plain `assert` because
[`python -O` strips it](./06-testing-and-mocking.md#assert-is-not-for-production-validation).
That last one is exactly why the config above adds a `tests/**` exception: in tests, `assert`
*is* the API. None of these say anything about vulnerable *dependencies* — that's a separate
class of tool.

## Coming From Another Language

| Concept | Java | JavaScript | Python |
|---------------------|-----------------------------|----------------------------|-------------------------------|
| Formatter | Spotless, google-java-format | Prettier | `ruff format` |
| Linter | Checkstyle, SpotBugs | ESLint | `ruff check` |
| Fix everything | `mvn spotless:apply` | `prettier -w && eslint --fix` | `ruff check --fix && ruff format` |
| Verify in CI | `mvn spotless:check` | `prettier --check` | `ruff format --check && ruff check` |
| Import ordering | IDE / Spotless `importOrder` | `eslint-plugin-import` | Ruff `I` rules |
| Suppress one warning | `@SuppressWarnings("...")` | `// eslint-disable-next-line` | `# noqa: F401` |
| Config location | `pom.xml`, `checkstyle.xml` | `eslint.config.js` | `[tool.ruff]` in `pyproject.toml` |
| Type checking | the compiler | `tsc` | `mypy` / `pyright` — *not* Ruff |

## Cheat Sheet

```bash
uv add --dev ruff                     # install

ruff check --fix . && ruff format .   # ≈ spotless:apply
ruff check . && ruff format --check . # ≈ spotless:check  (use in CI)

ruff check --diff .                   # preview lint fixes
ruff check --fix --unsafe-fixes .     # the riskier fixes, opt-in
ruff check --select I --fix .         # sort imports only
ruff check --statistics .             # violations per rule
ruff check --add-noqa .               # baseline an existing codebase
ruff rule F401                        # explain a rule
ruff linter                           # list rule prefixes
```

```toml
[tool.ruff]
line-length = 100
src = ["src", "tests"]

[tool.ruff.lint]
select = ["E", "W", "F", "I", "UP", "B", "SIM", "C4", "N", "TID", "PTH", "PT", "RUF"]
ignore = ["E501"]                     # length is the formatter's job; never select ALL

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101"]
"__init__.py" = ["F401"]

[tool.ruff.lint.isort]
known-first-party = ["my_app"]
```

Three rules to carry away: **`format` is layout, `check` is correctness — import sorting lives
in `check`**; **run `check --fix` before `format`, never the reverse**; and **Ruff is not a type
checker, so pair it with `mypy`**.

---

Previous: [Testing and Mocking](./06-testing-and-mocking.md) |
Next: [Packaging and Deployment](./08-packaging-and-deployment.md)
