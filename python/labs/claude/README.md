# Lab: Call Claude with a Prompt (Python)

The simplest possible LLM program: **one direct call to Claude** with a prompt and an API
key, using the official [anthropic](https://pypi.org/project/anthropic/) SDK. No agent
framework, no tools — just send text, get text back.

```python
import anthropic

client = anthropic.Anthropic()

message = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[{"role": "user", "content": "In one sentence, what is Python?"}],
)

print(message.content[0].text)
```

This assumes you have experience with other languages but zero Python setup. Once this
works, the [deepagents lab](../deepagents/README.md) builds on it to make an agent that can
call tools. For the *why* behind each step, see the [full tutorial](../../tutorial.md).

> **Two versions of this lab:** this page uses the standard `ANTHROPIC_API_KEY`, which the
> SDK reads automatically. If you'd rather read the key from **your own environment
> variable** in code (e.g. `CLAUDE_API_KEY`), follow the
> [custom env variable variant](./custom-env-key.md) instead.

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

`uv init` is the Python equivalent of `npm init` / `cargo new`:

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

## 4. Set Your API Key

The SDK reads your key from the `ANTHROPIC_API_KEY` environment variable. Get a key from
the [Anthropic Console](https://console.anthropic.com/settings/keys), then export it:

```bash
# macOS / Linux
export ANTHROPIC_API_KEY="sk-ant-..."
```

```powershell
# Windows (PowerShell)
$env:ANTHROPIC_API_KEY="sk-ant-..."
```

> The `export`/`$env:` form only lasts for the current terminal session, and you should
> **never commit the key to git**. To read the key from your own variable instead of
> `ANTHROPIC_API_KEY`, use the [custom env variable variant](./custom-env-key.md). For how
> API-key vs subscription billing works, see [API Keys & Billing](./api-keys.md).

---

## 5. Write the Code

Replace the contents of `main.py` with:

```python
import anthropic

# Reads ANTHROPIC_API_KEY from the environment automatically.
client = anthropic.Anthropic()

message = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[{"role": "user", "content": "In one sentence, what is Python?"}],
)

# The reply is a list of content blocks; the first is the text answer.
print(message.content[0].text)
```

A few things worth knowing:

- **`anthropic.Anthropic()`** picks up `ANTHROPIC_API_KEY` from the environment on its own.
  If your key is in a different variable, pass it explicitly:
  `anthropic.Anthropic(api_key=os.environ["YOUR_VAR"])` (see [API Keys & Billing](./api-keys.md)).
- **`model="claude-opus-4-8"`** is the most capable model. Swap it for
  `claude-haiku-4-5` (fastest/cheapest) or `claude-sonnet-5` (balanced) as needed.
- **`max_tokens`** caps the length of the *reply*. 1024 is plenty for a short answer; raise
  it for longer output.

---

## 6. Run It

```bash
uv run main.py
```

`uv run` executes the file inside the project's virtual environment automatically — no
"activation" step needed. You should see a one-sentence description of Python printed back.

---

## Understanding `messages` and roles

The `messages` list is the conversation, and each entry's `role` tags **who "said"** it:

- **`"user"`** — your input (what you're asking). A single prompt needs just one `user`
  message, which is why this lab has only one.
- **`"assistant"`** — Claude's own replies. You include these to hold a **multi-turn
  conversation**: the API is stateless, so you resend the history each call and add Claude's
  past answers back so it "remembers" them.

```python
messages=[
    {"role": "user", "content": "My name is Alice."},
    {"role": "assistant", "content": "Nice to meet you, Alice!"},  # Claude's earlier reply
    {"role": "user", "content": "What's my name?"},               # your new question
]
```

The list must **start with `user`** and (on current models) **end with `user`**.
Instructions about *how* Claude should behave (persona, tone, rules) don't go here — pass
them in the separate top-level `system=` parameter:

```python
client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    system="You are a terse assistant. Answer in one sentence.",
    messages=[{"role": "user", "content": "What is Python?"}],
)
```

---

## Cheat Sheet

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh   # one-time: install uv
uv init claude-hello && cd claude-hello           # create the project
uv add anthropic                                  # add the SDK
export ANTHROPIC_API_KEY="sk-ant-..."             # provide credentials
# edit main.py
uv run main.py                                    # run it
```

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `uv: command not found` | Open a new terminal so the updated `PATH` loads, or re-run the install script. |
| `AuthenticationError` / missing key | The key isn't set in the current terminal. Re-run the `export`/`$env:` command (step 4), or see [API Keys & Billing](./api-keys.md). |
| `ModuleNotFoundError: No module named 'anthropic'` | You ran `python main.py` instead of `uv run main.py`, so the venv wasn't used. Use `uv run`. |

---

Next: [Build a Deep Agent](../deepagents/README.md) (add tools to your Claude call), or the
[full tutorial](../../tutorial.md).
