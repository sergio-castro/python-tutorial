# Lab: Build a Deep Agent in Python (deepagents)

This lab takes you from **nothing installed** to a running LLM *agent* — a model that can
call tools — using [deepagents](https://pypi.org/project/deepagents/) and
[langchain-anthropic](https://pypi.org/project/langchain-anthropic/). The end goal is to
run this example:

```python
import os
from deepagents import create_deep_agent
from langchain_anthropic import ChatAnthropic

def get_weather(city: str) -> str:
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"

model = ChatAnthropic(
    model="claude-sonnet-4-6",
    api_key=os.environ["CLAUDE_API_KEY"],
)

agent = create_deep_agent(
    model=model,
    tools=[get_weather],
    system_prompt="You are a helpful assistant",
)

agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather in sf"}]}
)
```

> **API key:** like the rest of this project's labs, this one reads the key from a
> **custom-named** variable (`CLAUDE_API_KEY`) in code, rather than the SDK's default
> `ANTHROPIC_API_KEY`. The [Call Claude lab](../claude/README.md) shows *both* styles side by
> side; [API Keys & Billing](../claude/api-keys.md) explains why a distinct name is used.

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

The example reads your key from the `CLAUDE_API_KEY` environment variable (see the note at
the top of this lab for why a custom name). Get a key from the
[Anthropic Console](https://console.anthropic.com/settings/keys), then export it in your
terminal:

```bash
# macOS / Linux
export CLAUDE_API_KEY="sk-ant-..."
```

```powershell
# Windows (PowerShell)
$env:CLAUDE_API_KEY="sk-ant-..."
```

> The `export`/`$env:` form only lasts for the current terminal session, and you should
> **never commit the key to git**. The code in step 5 reads this variable explicitly —
> `langchain-anthropic` would otherwise only look for `ANTHROPIC_API_KEY`.
>
> ⚠️ Don't work around it with `export ANTHROPIC_API_KEY="$CLAUDE_API_KEY"` — that re-creates
> the standard name and can pull Claude Code onto API billing (see
> [API Keys & Billing](../claude/api-keys.md)).

---

## 5. Write the Code

Replace the contents of `main.py` with the agent. This is the example from the top of the
page, plus a small addition at the bottom so running it actually prints the answer:

```python
import os
from deepagents import create_deep_agent
from langchain_anthropic import ChatAnthropic

def get_weather(city: str) -> str:
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"

# Read the key from YOUR variable and pass it to the model explicitly.
model = ChatAnthropic(
    model="claude-sonnet-4-6",
    api_key=os.environ["CLAUDE_API_KEY"],
)

agent = create_deep_agent(
    model=model,
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
- **`ChatAnthropic(model="claude-sonnet-4-6", api_key=...)`** builds the model object
  explicitly. Passing an object (rather than the string `"anthropic:claude-sonnet-4-6"`) is
  what lets us inject a custom-named key. Note the model id here has **no `anthropic:`
  prefix** — that prefix is only for the string form. Swap the id for `claude-opus-4-8`
  (most capable) or `claude-haiku-4-5` (fastest/cheapest) as needed.
- **`api_key=os.environ["CLAUDE_API_KEY"]`** reads your variable and hands it to the client.
  If it isn't set, this raises `KeyError` — set it in step 4 first.

---

## 6. Run It

Use `uv run` — it executes the file inside the project's virtual environment automatically,
so you never have to "activate" anything (it also inherits your shell's environment, so it
sees `CLAUDE_API_KEY`):

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
export CLAUDE_API_KEY="sk-ant-..."      # Provide credentials (custom-named key)
uv run main.py                      # Run your code
```

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `uv: command not found` | Open a new terminal so the updated `PATH` loads, or re-run the install script. |
| `KeyError: 'CLAUDE_API_KEY'` | The variable isn't set in the current terminal. Re-run the `export`/`$env:` command (step 4). |
| `AuthenticationError` | The variable is set but the key is wrong/expired. Check it in the [Console](https://console.anthropic.com/settings/keys). |
| `ModuleNotFoundError: No module named 'deepagents'` | You ran `python main.py` instead of `uv run main.py`, so the venv wasn't used. Use `uv run`. |
| Want a specific Python version | `uv python install 3.12` then `uv venv --python 3.12`. |

---

Next: read the [full tutorial](../../tutorial.md) to understand virtual environments,
package managers, and project configuration in depth.
