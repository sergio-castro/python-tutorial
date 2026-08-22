# Lab: Call Claude with a Custom Env Variable (Python)

Same one-call program as the [main Claude lab](./README.md), with one difference: instead of
letting the SDK auto-read `ANTHROPIC_API_KEY`, this version **reads the key from an
environment variable of your choice** (here `CLAUDE_API_KEY`) and passes it to the client
explicitly in code.

```python
import os
import anthropic

client = anthropic.Anthropic(api_key=os.environ["CLAUDE_API_KEY"])

message = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[{"role": "user", "content": "In one sentence, what is Python?"}],
)

print(message.content[0].text)
```

**Why you'd want this:** keeping `ANTHROPIC_API_KEY` *unset* lets Claude Code stay on your
web-login subscription, while your own programs use a separate key under a different name.
See [API Keys & Billing](./api-keys.md) for the full rationale.

> **Two versions of this lab:** this page reads the key from a custom env variable in code.
> For the standard `ANTHROPIC_API_KEY` approach (SDK reads it automatically), see the
> [main lab](./README.md).

We use **`uv`** (the modern, single-tool package manager — see
[Package Managers](../../02-package-managers.md) for the pip comparison).

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

`uv` installs Python for you — nothing else to install globally.

---

## 2. Create the Project

```bash
uv init claude-hello
cd claude-hello
```

This creates a `pyproject.toml` (the project manifest) and a placeholder `main.py`.

---

## 3. Add the Dependency

```bash
uv add anthropic
```

`uv add` downloads the official Anthropic SDK, records it in `pyproject.toml`, and creates
the virtual environment (`.venv/`) and lock file (`uv.lock`) on first use.

---

## 4. Set Your (Custom-Named) API Key

Pick any variable name you like — this lab uses `CLAUDE_API_KEY`. Get a key from the
[Anthropic Console](https://console.anthropic.com/settings/keys), then export it:

```bash
# macOS / Linux
export CLAUDE_API_KEY="sk-ant-..."
```

```powershell
# Windows (PowerShell)
$env:CLAUDE_API_KEY="sk-ant-..."
```

> The `export`/`$env:` form only lasts for the current terminal session, and you should
> **never commit the key to git**. Because the SDK does *not* look for this name on its own,
> the code in step 5 reads it explicitly — that's the whole point of this variant.
>
> ⚠️ Don't work around it with `export ANTHROPIC_API_KEY="$CLAUDE_API_KEY"` — that re-creates
> the standard name and can pull Claude Code onto API billing (see
> [API Keys & Billing](./api-keys.md)).

---

## 5. Write the Code

Replace the contents of `main.py` with:

```python
import os
import anthropic

# Read the key from YOUR variable and pass it in explicitly.
# (The SDK only auto-detects ANTHROPIC_API_KEY, which we're deliberately not using.)
client = anthropic.Anthropic(api_key=os.environ["CLAUDE_API_KEY"])

message = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[{"role": "user", "content": "In one sentence, what is Python?"}],
)

# The reply is a list of content blocks; the first is the text answer.
print(message.content[0].text)
```

A few things worth knowing:

- **`os.environ["CLAUDE_API_KEY"]`** reads the variable you set in step 4. If it isn't set,
  this raises `KeyError` — set it first. (Use `os.environ.get("CLAUDE_API_KEY")` if you'd
  rather get `None` than an error.)
- **`api_key=...`** is the constructor argument that overrides the SDK's default env-var
  lookup. Any name works — swap `CLAUDE_API_KEY` for whatever you use.
- **`model="claude-opus-4-8"`** is the most capable model. Swap it for `claude-haiku-4-5`
  (fastest/cheapest) or `claude-sonnet-5` (balanced) as needed.
- **`max_tokens`** caps the length of the *reply*. 1024 is plenty for a short answer.

---

## 6. Run It

```bash
uv run main.py
```

`uv run` executes the file inside the project's virtual environment and inherits your
shell's environment (so it sees `CLAUDE_API_KEY`). You should see a one-sentence description
of Python printed back.

---

## Cheat Sheet

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh   # one-time: install uv
uv init claude-hello && cd claude-hello           # create the project
uv add anthropic                                  # add the SDK
export CLAUDE_API_KEY="sk-ant-..."                # your custom-named key
# edit main.py (reads CLAUDE_API_KEY explicitly)
uv run main.py                                    # run it
```

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `KeyError: 'CLAUDE_API_KEY'` | The variable isn't set in the current terminal. Re-run the `export`/`$env:` command (step 4). |
| `AuthenticationError` | The variable is set but the key is wrong/expired. Check it in the [Console](https://console.anthropic.com/settings/keys). |
| `ModuleNotFoundError: No module named 'anthropic'` | You ran `python main.py` instead of `uv run main.py`, so the venv wasn't used. Use `uv run`. |

---

Next: [Build a Deep Agent](../deepagents/README.md), or the [full tutorial](../../tutorial.md).
