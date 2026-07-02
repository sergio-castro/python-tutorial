# Lab: Build a Deep Agent in Python (deepagents)

This lab takes you from **nothing installed** to a running LLM *agent* — a model that can
call tools — using [deepagents](https://pypi.org/project/deepagents/) and
[langchain-anthropic](https://pypi.org/project/langchain-anthropic/). The end goal is to
run this example:

```python
from deepagents import create_deep_agent

def get_weather(city: str) -> str:
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    tools=[get_weather],
    system_prompt="You are a helpful assistant",
)

agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather in sf"}]}
)
```

> If you just want a **single, direct call to Claude** (a prompt and a key, no agent
> framework), start with the simpler [Call Claude](../claude/README.md) lab first.

It assumes you have experience with other languages but zero Python setup. If you want
to understand *why* each step exists, read the [full tutorial](../../tutorial.md) — this
page is the fast path.

We use **`uv`** throughout (the modern, single-tool package manager — see
[Package Managers](../../02-package-managers.md) for the pip comparison).

---

## 1. Install the Toolchain

You need two things: the `uv` tool, and a Python interpreter. `uv` installs Python for
you, so it's the only manual install.

### macOS / Linux

```bash
# Install uv (one line — no admin rights needed)
curl -LsSf https://astral.sh/uv/install.sh | sh
```

On macOS you can alternatively use Homebrew: `brew install uv`.

### Windows

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

### Verify

Open a **new** terminal (so the updated `PATH` takes effect) and check:

```bash
uv --version
```

You do **not** need to install Python separately — `uv` downloads and manages Python
versions on demand. There's nothing else to install globally: dependencies live inside
the project (next steps), not on your system.

---

## 2. Set Up the Project Structure

Create a new project. `uv init` is the Python equivalent of `npm init` / `cargo new`:

```bash
uv init weather-agent
cd weather-agent
```

This generates a minimal, ready-to-run layout:

```
weather-agent/
├── .python-version      # Pins the Python version for this project
├── pyproject.toml       # Project manifest (like package.json / Cargo.toml)
├── README.md
└── main.py              # A placeholder entry-point file
```

`pyproject.toml` is the project manifest — metadata, dependencies, and build config all
in one file. See [Project Configuration](../../03-project-configuration.md) for a full
breakdown.

> **Note:** There is no `.venv/` folder yet. `uv` creates it automatically the first time
> you add a dependency or run code. The `.venv/` is disposable — never commit it (see
> [Virtual Environments](../../01-virtual-environments.md)).

---

## 3. Add Dependencies

Add the two packages the example needs. `uv add` downloads them, records them in
`pyproject.toml`, and creates the virtual environment (`.venv/`) and lock file
(`uv.lock`) on first use:

```bash
uv add deepagents langchain-anthropic
```

That single command does what `python -m venv` + `pip install` + `pip freeze` would do
separately in the older pip workflow.

After it finishes, `pyproject.toml` will list the two packages under `dependencies`, and
your folder gains:

```
weather-agent/
├── .venv/               # The isolated environment (auto-created, do not commit)
├── uv.lock              # Exact locked versions for reproducible installs
└── ...
```

---

## 4. Set Your API Key

The example calls an Anthropic model, so it needs an API key. `langchain-anthropic` reads
it from the `ANTHROPIC_API_KEY` environment variable. Get a key from the
[Anthropic Console](https://console.anthropic.com/settings/keys), then export it in your
terminal:

```bash
# macOS / Linux
export ANTHROPIC_API_KEY="sk-ant-..."
```

```powershell
# Windows (PowerShell)
$env:ANTHROPIC_API_KEY="sk-ant-..."
```

> The `export`/`$env:` form only lasts for the current terminal session. For a persistent
> setup, add the key to a `.env` file or your shell profile — but **never commit the key
> to git**.
>
> If your key lives in a **differently-named** variable (e.g. you renamed it to avoid a
> Claude Code billing collision), or you're unsure how API-key vs subscription billing
> works, see [API Keys & Billing](../claude/api-keys.md).

---

## 5. Write the Code

Replace the contents of `main.py` with the agent. This is the example from the top of the
page, plus a small addition at the bottom so running it actually prints the answer:

```python
from deepagents import create_deep_agent

def get_weather(city: str) -> str:
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    tools=[get_weather],
    system_prompt="You are a helpful assistant",
)

# Run the agent
result = agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather in sf"}]}
)

# Print the agent's final reply so you can see the result
print(result["messages"][-1].content)
```

A few things worth knowing for someone new to Python:

- **The `"""..."""` line under `get_weather`** is a *docstring*. It's not just a comment —
  `deepagents` sends it to the model as the tool's description, so the model knows what the
  tool does. Keep it meaningful.
- **`model="anthropic:claude-sonnet-4-6"`** uses the `provider:model-id` format. You can
  swap the model id (e.g. `anthropic:claude-opus-4-8` for the most capable model, or
  `anthropic:claude-haiku-4-5` for the fastest/cheapest). The `anthropic:` prefix is what
  routes it through `langchain-anthropic`.

---

## 6. Run It

Use `uv run` — it executes the file inside the project's virtual environment automatically,
so you never have to "activate" anything:

```bash
uv run main.py
```

You should see the model call the `get_weather` tool and reply with something like *"It's
always sunny in SF!"*.

That's the whole loop. `uv run` is the recommended way to run code day-to-day — see
[Running Your Application](../../04-running-your-application.md) for the alternatives
(activating the venv, calling the interpreter directly) and how to define named CLI
commands.

---

## Cheat Sheet

```bash
# One-time: install the toolchain
curl -LsSf https://astral.sh/uv/install.sh | sh

# Per project
uv init weather-agent               # Create project structure
cd weather-agent
uv add deepagents langchain-anthropic   # Add dependencies (creates .venv + uv.lock)
export ANTHROPIC_API_KEY="sk-ant-..."   # Provide credentials
uv run main.py                      # Run your code
```

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `uv: command not found` | Open a new terminal so the updated `PATH` loads, or re-run the install script. |
| `ANTHROPIC_API_KEY` error / auth failure | The key isn't set in the current terminal. Re-run the `export`/`$env:` command (step 4), or see [API Keys & Billing](../claude/api-keys.md). |
| `ModuleNotFoundError: No module named 'deepagents'` | You ran `python main.py` instead of `uv run main.py`, so the venv wasn't used. Use `uv run`. |
| Want a specific Python version | `uv python install 3.12` then `uv venv --python 3.12`. |

---

Next: read the [full tutorial](../../tutorial.md) to understand virtual environments,
package managers, and project configuration in depth.
