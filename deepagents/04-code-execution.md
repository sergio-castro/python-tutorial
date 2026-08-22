# Code Execution

An agent that can only call your tools is limited to what you thought of in advance. Letting it
*run code* removes that ceiling — and Deep Agents offers two separate ways to do it, which are
easy to confuse because both end with the agent executing something it wrote.

| | Sandbox backends | Interpreters |
|-------------------|--------------------------------------|----------------------------------------|
| Tool added | `execute` | `eval` |
| Runs | Shell commands | JavaScript |
| Where | An isolated container or VM | A small JavaScript engine, in memory, inside the agent loop |
| Configured as | `backend=` | `middleware=` |
| Reaches | A real OS: files, packages, network | Nothing, unless you open a bridge |
| For | Acting **on an environment** | Composing tools and **transforming data** |
| Status | Stable | Beta |

The distinction that matters: a sandbox is a *place to do work*, an interpreter is a *way to
orchestrate work*. Sandboxes install dependencies, run tests, and edit files. Interpreters loop,
branch, retry, and aggregate — deciding what is worth returning to the model at all.

They are also structurally different. A sandbox **is a backend**, so it slots into the same
`backend=` parameter as `StateBackend` and lives under the rules in
[Backends, State, and Memory](./01-backends-and-memory.md). An interpreter is **middleware** and
is not a backend at all. That is why code execution is its own chapter rather than a section of
the backends one.

## Sandboxes: the `execute` Tool

```python
from deepagents import create_deep_agent
from deepagents.backends import LangSmithSandbox
from langsmith.sandbox import SandboxClient

client = SandboxClient()
ls_sandbox = client.create_sandbox()

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    backend=LangSmithSandbox(sandbox=ls_sandbox),
)
try:
    agent.invoke({"messages": [{"role": "user", "content": "Run the tests"}]}, config)
finally:
    client.delete_sandbox(ls_sandbox.name)
```

### How the tool appears

You never ask for `execute`. The harness checks, **on every model call**, whether the backend
implements the sandbox protocol (`SandboxBackendProtocol` — the overview page calls it
`SandboxBackendProtocolV2`). If it does, `execute` is in the tool set; if not, the tool is
filtered out and the agent never sees it.

So the tool surface is a property of the backend you chose, not a switch you flip. Swap a sandbox
for a `StateBackend` and the agent silently loses the ability to run anything.

### `execute` is the whole implementation

Sandbox backends have a deliberately small architecture: **the only method a provider must
implement is `execute()`**. Everything else — `read_file`, `write_file`, `edit_file`, `delete`,
`ls`, `glob`, `grep` — is built on top of it by the `BaseSandbox` base class, which constructs
scripts and runs them through `execute()`.

Two consequences. Adding a provider is easy, because implementing one method gets you the whole
filesystem surface. And a sandbox's "filesystem" is not a separate storage system at all — it is
shell commands against the container's real disk.

When the agent calls the tool it passes a `command` string and receives combined stdout/stderr,
the exit code, and a truncation notice if the output was too large. You can also call
`backend.execute()` directly from your own application code.

### Lifecycle and scope

Sandboxes are the one backend whose scope you choose rather than inherit:

- **Thread-scoped** (default) — one sandbox per conversation, discarded with it.
- **Assistant-scoped** — one sandbox shared by every thread of an assistant. Files, installed
  packages, and cloned repositories survive across conversations, so in-sandbox state accumulates:
  set a TTL with your provider, snapshot and reset periodically, or clean up explicitly.

They consume resources and bill until shut down. Deleting them is your job — hence the `finally`
in every example.

### Getting files in and out

A sandbox has **two separate planes of file access**, and mixing them up is the usual first
mistake:

| Plane | Who calls it | What it is for |
|--------------------|-------------------|--------------------------------------------|
| Agent tools — `read_file`, `write_file`, `execute`, … | The model, during a run | Doing the work, inside the sandbox |
| `upload_files()` / `download_files()` | Your application code | Moving data across the boundary |

The transfer APIs use the provider's native file transfer, not shell commands. Use them to seed
the sandbox with source code or data before the run, and to retrieve artifacts after it:

```python
backend.upload_files([("/src/index.py", b"print('hello')\n")])   # absolute paths, bytes
...
for f in backend.download_files(["/src/index.py", "/output.txt"]):
    if f.content is not None:
        print(f.path, f.content.decode())
    else:
        print("failed:", f.path, f.error)
```

Forget the download and the work disappears with the sandbox.

### What isolation does and does not buy

Every provider protects your **host**: the agent cannot read your local files, see your
environment variables, or touch other processes. That is a real and important boundary.

It does not protect against:

- **Context injection.** Whoever controls part of the agent's input can tell it to run arbitrary
  commands. The sandbox is isolated; the agent has full authority *within* it.
- **Network exfiltration.** Unless network access is blocked, an injected agent can send data out
  over HTTP or DNS. Some providers can block it — Modal's `blockNetwork: true`, for instance.

Two more things to know. **Never put secrets inside a sandbox** — API keys, tokens, database
credentials injected via environment variables, mounted files, or a `secrets` option can all be
read and exfiltrated by an injected agent, short-lived or not. And **filesystem permissions do not
apply to sandbox backends**, since arbitrary command execution routes around them; for custom
validation use backend policy hooks instead.

### `LocalShellBackend`, and why it is not this

`LocalShellBackend` also provides `execute`, but on **your host**, via
`subprocess.run(shell=True)`, with no isolation whatsoever. It supports `timeout` (120s default),
`max_output_bytes` (100,000), `env`, and `inherit_env`, and `virtual_mode` buys you nothing —
a shell reaches any path regardless.

It is a local-development convenience. Anywhere that processes input you do not control, the
sandbox is the answer.

## Interpreters: the `eval` Tool

```bash
uv add "deepagents[quickjs]"
```

```python
from deepagents import create_deep_agent
from langchain_quickjs import CodeInterpreterMiddleware

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    middleware=[CodeInterpreterMiddleware()],
)
```

Interpreters are in **beta** — APIs and lifecycle behaviour may change between releases. They
require `langchain-quickjs>=0.2.0` and Python 3.11+.

Note the parameter: `middleware`, not `backend`. An interpreter changes nothing about where files
go; it adds a programmable scratch space inside the agent loop.

### The problem it solves

A model can emit several tool calls in one turn, but that batch is fixed the moment it is sent.
Nothing can loop, branch on a result, retry a failure, or feed one call's output into the next
without another model turn — and every intermediate result lands in the context window. Ask a
model to dispatch work across hundreds of items and it will typically cover a sample rather than
all of them.

An interpreter moves that orchestration into code, so the model reasons about *what* to do rather
than about every step of doing it.

### How it works

The middleware adds an `eval` tool. The agent writes JavaScript, calls `eval`, and the runtime
returns the value of the last expression, having captured `console.log`, `console.warn`, and
`console.error`. You never call the interpreter yourself.

```js
const rows = [
  { team: "alpha", score: 8 },
  { team: "beta", score: 13 },
  { team: "alpha", score: 21 },
];

const totals = rows.reduce((acc, row) => {
  acc[row.team] = (acc[row.team] ?? 0) + row.score;
  return acc;
}, {});

totals;
```

State persists across turns in the same thread by default (`mode="thread"`).

The engine is [QuickJS](https://github.com/quickjs-ng/quickjs) — a small, embeddable JavaScript
runtime. It runs in your own process, not a container, which is why it is cheap to use and also
why it is fenced off so tightly.

**The sandbox is a real machine; the interpreter is deliberately not.** By default, interpreter
code has no access to the host filesystem, network, shell, package manager, or even the clock. It
can compute, hold state, and log. Nothing else crosses the QuickJS boundary.

### The two bridges

Exactly two things extend that reach, and both are explicit:

**Programmatic tool calling (PTC)** — off until you enable it with an allowlist:

```python
middleware=[CodeInterpreterMiddleware(ptc=["web_search"])]
```

Allowlisted tools appear inside the interpreter as async functions under a `tools` namespace,
with names converted to camelCase while the argument object still follows the tool's schema:

```js
const result = await tools.webSearch({ query: "deepagents interpreters" });
```

The model then sees only the final interpreter output, not every intermediate value — which is
the point. It is implemented by middleware, so it is model-agnostic rather than depending on a
provider's tool-calling API.

**Dynamic subagents** — when the agent has subagents configured, the interpreter exposes a
`task()` global for dispatching them from code, enabling fan-out, verification passes, and
recursive workflows over large inputs. Unlike PTC this is **on by default**, and can be turned
off.

## Choosing

| You need | Use |
|--------------------------------------------------------|--------------------------------|
| One or two straightforward external calls | Ordinary tool calling |
| Loops, branches, retries, or data transforms in memory | An interpreter |
| Many external tool calls orchestrated from code | An interpreter with PTC |
| Fan-out, multiple perspectives, recursion over big inputs | An interpreter with dynamic subagents |
| Shell commands, package installs, tests, OS filesystem | A sandbox backend |
| Any of the above, on your own laptop, quickly | `LocalShellBackend` — and nowhere else |

The two are complementary, not alternatives: a sandbox backend and interpreter middleware can be
configured on the same agent, since one is `backend=` and the other is `middleware=`.

## Gotchas

| Symptom | Cause |
|--------------------------------------------|--------------------------------------------------------|
| The agent has no `execute` tool | The backend does not implement the sandbox protocol — the harness filters the tool out |
| Work vanished after the run | You deleted the sandbox without `download_files` |
| Sandbox costs keep accruing | Nothing shuts them down automatically; delete them, or set a provider TTL |
| Assistant-scoped sandbox grows without bound | State accumulates across threads by design — snapshot, reset, or set a TTL |
| Filesystem permissions are ignored | They do not apply to sandbox backends; use backend policy hooks |
| Interpreter code cannot reach a tool | PTC is off by default — pass an explicit `ptc=[...]` allowlist |
| Interpreter code cannot fetch a URL or read a file | By design: no network, filesystem, shell, or clock. Bridge through PTC or use a sandbox |
| A tool name is not found inside `eval` | Names are camelCased: `web_search` becomes `tools.webSearch` |

## Cheat Sheet

```python
# Acting on an environment — a place to run things
agent = create_deep_agent(
    model=M,
    backend=LangSmithSandbox(sandbox=client.create_sandbox()),   # adds `execute`
)

# Orchestrating work — a way to compose things
agent = create_deep_agent(
    model=M,
    middleware=[CodeInterpreterMiddleware(ptc=["web_search"])],  # adds `eval`
)
```

| | Sandbox | Interpreter |
|--------------|-----------------------------------|------------------------------------|
| Parameter | `backend=` | `middleware=` |
| Tool | `execute` | `eval` |
| Isolation | Container or VM, away from your host | QuickJS: no FS, network, shell, or clock |
| Escape hatch | It is already a real machine | PTC allowlist; `task()` for subagents |
| Cleanup | Yours — delete the sandbox | None |

Three things to carry away: **`execute` appears because of the backend you chose, not a flag you
set**; **a sandbox protects your host but not against an injected agent inside it**; and **an
interpreter reaches nothing until you open a bridge**.

---

Sources: [Sandboxes](https://docs.langchain.com/oss/python/deepagents/sandboxes),
[Interpreters](https://docs.langchain.com/oss/python/deepagents/interpreters),
[Backends](https://docs.langchain.com/oss/python/deepagents/backends),
[Overview](https://docs.langchain.com/oss/python/deepagents/overview).

Previous: [Harness Profiles](./03-harness-profiles.md) |
Next: [Managing the Context Window](./05-context-window.md) |
Back to [Deep Agents](./README.md) | [Index](../README.md)
