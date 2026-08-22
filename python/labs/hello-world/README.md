# Lab: Hello World in Python

The smallest possible Python program, run the modern way. If you've never set up a Python
project before, start here. For the *why* behind each step, see the
[full tutorial](../../tutorial.md).

We use **`uv`** — the modern, single-tool package manager and Python installer.

---

## 1. Install `uv` (one time)

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh
```

```powershell
# Windows (PowerShell)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Open a **new** terminal and confirm it works:

```bash
uv --version
```

`uv` installs Python for you — you don't need to install it separately.

---

## 2. Create the Project

```bash
uv init hello-world
cd hello-world
```

This creates a `pyproject.toml` (the project manifest) and a placeholder `main.py`.

---

## 3. Write the Code

Replace the contents of `main.py` with:

```python
def main() -> None:
    print("Hello, World!")

if __name__ == "__main__":
    main()
```

The `if __name__ == "__main__":` line means "only run `main()` when this file is executed
directly, not when it's imported by another file." It's Python's equivalent of a
`main()` entry point in Java/Go — see [Running Your Application](../../04-running-your-application.md)
for the full explanation.

(You could also just write `print("Hello, World!")` on its own — the function + guard is
the conventional structure you'll see in real projects.)

---

## 4. Run It

```bash
uv run main.py
```

Output:

```
Hello, World!
```

`uv run` executes the file inside the project's virtual environment automatically — no
"activation" step needed.

---

## Cheat Sheet

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh   # one-time: install uv
uv init hello-world && cd hello-world             # create the project
# edit main.py
uv run main.py                                    # run it
```

---

Next lab: [Run a Claude Agent in Python](../claude/README.md).
