# Backends, State, and Memory

A deep agent sees exactly one storage abstraction: a filesystem. It calls `ls`, `read_file`,
`write_file`, `edit_file`, `delete`, `glob`, and `grep`, and it has no idea what is on the other
side. What *is* on the other side is the **backend**: the implementation those tools run against.

Choosing a backend settles three things.

- **Where files go**, and therefore who can read them and whether they survive. That is most of
  this chapter.
- **Whether the agent can run code.** Sandbox and local-shell backends also provide an `execute`
  tool; the others do not. That is [Code Execution](./04-code-execution.md).
- **What the agent is allowed to touch**, and where that gets enforced — see
  [Access Control](#access-control) below.

So "backend" is a broader idea than "storage": it is the whole surface the agent's tools sit on.

Taking the storage part first, two questions decide everything, and they are answered in two
different places:

| Question | The answers | Chosen by |
|----------------------------------------------------|----------------------------------|--------------------------------------------|
| **Scope** — can another conversation read this file? | Thread-scoped, or cross-thread | The **backend** class |
| **Medium** — where do the bytes physically sit? | RAM, a local file, or a database | The **checkpointer** or the **store** — see below |

Durability follows from the medium rather than being a setting of its own: RAM is lost when the
process restarts, a file or a database is not. "Make this durable" always means "point it at a
medium that isn't RAM."

## The Two Axes

### Scope: chosen by the backend class

"The backend decides scope" means something concrete: the class you pass to
`create_deep_agent(backend=...)` decides where a `write_file` actually lands, and the
destination fixes who can read it back — the **Readable by** column below. That rule is not
configurable; you pick it by picking the class.

Three words show up in that last column before they are properly introduced:

- A **thread** is one conversation, identified by a `thread_id` you pass in at invoke time.
- A **namespace** is a tuple of strings that carves the store into compartments. It behaves like
  a path: `("alice",)` is one compartment and `("alice", "chat-7")` nests inside it. You compose
  it from whatever you want to isolate by — user, agent, conversation, or several of those in the
  order you choose.
- An **assistant** is a named, versioned record on the Agent Server — LangChain's hosting
  platform for deployed agents, part of LangSmith — pairing a deployed agent with one configuration — not a runtime setting you pass at invoke time. Two
  assistants can run the same agent code with different models or prompts, each with its own
  `assistant_id`, and many threads run against one assistant. It exists only once you deploy: in
  a local script there is no assistant at all. In this chapter it shows up only as
  `rt.server_info.assistant_id`, used to scope storage per agent.

Threads and namespaces each get a fuller treatment further down.

| `backend=` | Where the file physically goes | Readable by |
|-----------------------------------|-----------------------------------------------------|-------------------------------------------------|
| `StateBackend()` *(default)* | Into the run's working data — the dict LangGraph carries from step to step, snapshotted by the checkpointer | Only runs with the same `thread_id` |
| `StoreBackend(...)` | Into the LangGraph **store** — a key–value layer you plug a database into | Any thread whose `namespace` matches |
| `FilesystemBackend(...)` | Onto the machine's disk, as ordinary files | Everything on the machine — it is just your disk |
| `LocalShellBackend(...)` | Onto the machine's disk, plus a shell to run commands with | Everything on the machine |
| `ContextHubBackend("my-agent")` | Into a versioned repo hosted in LangSmith, written as commits | Anyone with access to that repo |
| A sandbox backend | Onto the disk of the throwaway container or VM the agent is running inside | Whatever shares that sandbox — one thread, or every thread of an assistant |
| `CompositeBackend(...)` | Wherever the path prefix routes it — any of the above | Per route: whatever the routed backend says |
| Your own `BackendProtocol` impl | Wherever you send it — S3, a database, a remote filesystem | Yours to define |

That is the complete set: two that use a LangGraph storage slot, four that bring their own
storage, one router, and an escape hatch. Each is described in [The Backends](#the-backends)
below.

**Neither of the first two is a "store" in the everyday sense**, which is worth pinning down
before going further:

- **LangGraph state** is not storage at all — it is the working data of a single run: a dict the
  agent carries from one step to the next, held in the process's memory while the run executes.
  That is always true, whatever you configure. It *outlives* the run only because the
  **checkpointer** writes a copy of it somewhere else.

  So there are two copies of a `StateBackend` file, and the question "is it in memory?" has a
  different answer for each. The **live copy** is in RAM, always. The **checkpointed copy** is
  wherever the checkpointer puts it — RAM, a local file, or a database — and that is the medium
  axis below. With no checkpointer at all there is no second copy and no memory between calls:
  the next `invoke` starts from nothing.
- **The LangGraph store** *is* a real key–value database, with `put` / `get` / `search`. But
  "database" here still includes `InMemoryStore`, which is a dict in RAM.

So neither one is durable by nature. Whether a file survives a restart is a separate choice —
the medium — and that is the next section.

### Medium: chosen by what you wire underneath

This axis applies to the two backends that use LangGraph's own storage: `StateBackend`, which
writes into **state**, and `StoreBackend`, which writes into the **store**. Picking the backend
picks which of those two the file goes into — and says nothing about what either one is *made
of*. That is a separate choice, and a different one for each.

#### Checkpointer and store are not two names for one thing

They appear together on this axis because both are pluggable persistence you can point at RAM, a
file, or a database. But they hold different things, for different reasons:

|                | **Checkpointer** | **Store** |
|----------------|--------------------------------------------------|------------------------------------------------|
| Persists | Snapshots of the whole graph state | Application-defined key–value items |
| Keyed by | `thread_id`, plus a checkpoint id per step | A `namespace` tuple, plus a key |
| Scope | A single thread | Across threads |
| Keeps history | Yes — every step is a version you can rewind to | No — `put` overwrites |
| Written | Implicitly, by the graph, as the run proceeds | Explicitly, via `put` / `get` / `search` / `delete` |
| Exists for | Conversation continuity, human-in-the-loop, time travel, fault tolerance | User preferences, facts, shared knowledge |
| Passed as | `create_deep_agent(checkpointer=...)` | `create_deep_agent(store=...)` |

A real agent usually has **both**, and neither substitutes for the other. The checkpointer is
what makes `StateBackend` files survive — along with the message history, and any run that is
sitting paused waiting for a human. The store is what makes `StoreBackend` files survive.

The medium is then chosen independently for each:

|                                                      | **RAM** — volatile | **Local file** — durable | **Database** — durable |
|------------------------------------------------------|--------------------|--------------------------|------------------------|
| **Thread-scoped**<br>`StateBackend` → *checkpointer*  | `InMemorySaver` / `MemorySaver` — the demo default | `SqliteSaver` | `PostgresSaver` |
| **Cross-thread**<br>`StoreBackend` → *store*          | `InMemoryStore` — dev only | `AsyncSqliteStore` † | `PostgresStore`, `RedisStore`, `MongoDBStore`, `UpstashStore` |

† A SQLite store exists, but the docs advise against SQLite for production deployments; the
mainstream stores are networked services.

Note that the cells are not backends — they are the checkpointers and stores from the table
above. Reading *across* a row is the entire point: **moving along a row changes durability and
nothing else.** Swap `InMemorySaver` for `PostgresSaver` and your `StateBackend` files now survive a
restart — and are still invisible to every other thread. This is why "in-memory state" is a
misleading phrase: `StateBackend` is thread-scoped whether or not the bytes happen to be in RAM.

Only `StateBackend` and `StoreBackend` appear in that grid, because only they use a LangGraph
slot. `FilesystemBackend`, `LocalShellBackend`, `ContextHubBackend`, and the sandbox backends
bring their own storage, so no checkpointer or store applies to them — their durability and
scope come from the thing they wrap (your disk, a Hub repo, a running sandbox).

## What the Agent Sees Never Changes

The tool surface is fixed — `ls`, `read_file`, `write_file`, `edit_file`, `delete`, `glob`,
`grep`, plus `execute` for sandbox and local-shell backends. You cannot rename these tools, and
the agent's prompt does not change when you swap storage. That is the point of the abstraction:
you move a file from ephemeral to durable by changing one line of wiring, not by re-teaching the
agent.

`read_file` also handles images (`.png`, `.jpg`, `.jpeg`, `.gif`, `.webp`) on every backend,
returning them as multimodal content blocks.

## The Backends

### `StateBackend` — the default

```python
from deepagents import create_deep_agent

agent = create_deep_agent(model="anthropic:claude-sonnet-4-6")   # StateBackend() implied
```

Files live in LangGraph agent state. They persist across many turns **within one thread** via
checkpoints, and are invisible to any other thread. Best for scratch pads, intermediate results,
and the automatic eviction of large tool outputs that the agent can read back in pieces.

Two behaviours worth knowing:

- **Subagents share it.** A deep agent can hand work to *subagents* — child agents spawned by
  the main one, the *supervisor*. All of them read and write the same state filesystem, and files
  a subagent writes **survive that subagent's exit**, staying available to the supervisor and to
  later subagents. This is the intended hand-off channel between them.
- **It only works inside a graph run.** Calling backend methods directly from outside a run
  won't take effect until the graph executes.

### `FilesystemBackend` — real local disk

```python
from deepagents.backends import FilesystemBackend

backend = FilesystemBackend(root_dir="/abs/path/to/project", virtual_mode=True)
```

Real files, real disk, under a `root_dir` that must be an absolute path. This is durable and
visible to every thread and every process — not because LangGraph shares it, but because it is
simply your filesystem. That is why it isn't in the 2×2: it is outside the scoping model.

Two things that bite:

- **`virtual_mode=True` is not optional in practice.** The default is `False`, which provides
  *no* path restriction even when `root_dir` is set. With it on, `..`, `~`, and absolute paths
  outside the root are blocked.
- **Never in a web server or HTTP API.** The agent can read any accessible file — `.env`,
  credentials, keys — and combined with network tools that becomes an exfiltration path. Use
  `StateBackend`, `StoreBackend`, or a sandbox instead. Local dev CLIs and CI are the legitimate
  cases, ideally with human-in-the-loop review on writes.

### `LocalShellBackend` — disk plus a shell

`FilesystemBackend` extended with an `execute` tool that runs commands via
`subprocess.run(shell=True)` on your host, with no sandboxing. Supports `timeout` (120s default),
`max_output_bytes`, and `env`. Note that `virtual_mode=True` buys you nothing here — a shell can
reach any path regardless. Human-in-the-loop approval is strongly recommended, and this belongs
nowhere near production.

### `StoreBackend` — the LangGraph store

```python
from deepagents.backends import StoreBackend
from langgraph.store.memory import InMemoryStore

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    backend=StoreBackend(namespace=lambda rt: (rt.server_info.user.identity,)),
    store=InMemoryStore(),          # omit on LangSmith Deployment — provisioned for you
)
```

#### What "the store" actually is

The store is not a product, it is an **interface**: `BaseStore`. LangGraph defines five methods,
and anything implementing them can be a store.

| Method | Does |
|--------------------------------------------------|-----------------------------------------------|
| `put(namespace, key, value)` | Write or overwrite one item |
| `get(namespace, key)` | Fetch one item, `None` if missing |
| `delete(namespace, key)` | Remove one item |
| `search(namespace_prefix, *, query, filter, limit, offset)` | List items under a namespace, optionally filtered or semantically ranked |
| `list_namespaces(*, prefix, suffix, max_depth, …)` | Discover which namespaces exist |

An item is `(namespace tuple, key) → value`, where the value is a plain JSON-serialisable dict.
Not a Python object — if it doesn't survive `json.dumps`, it doesn't belong in a store.

#### Key–value, but not necessarily a key–value database

The interface and the engine behind it are separate choices, and they are easy to conflate:

- **The interface is key–value.** There is no join, no SQL, no arbitrary `WHERE` clause, and no
  transaction spanning several items. You address data by namespace and key.
- **The database behind it is whatever you like, relational included.** `PostgresStore` is a
  relational database doing exactly this job. The docs' suggested schema is a single table —
  `namespace TEXT[]`, `key TEXT`, `value JSONB`, primary key `(namespace, key)` — so Postgres
  stores the value as JSONB and indexes the namespace. Redis, MongoDB, Upstash and SQLite
  implementations exist too.

So "relational or not" is an operational choice — reuse the Postgres you already run — and it
changes nothing about how you query. What you give up versus using that Postgres directly is
relational querying; what you get is one interface every LangGraph component already speaks.

The API is slightly richer than pure key–value in two ways: `search` takes a `filter` on fields
inside the value, and, where the backend supports vector search, a `query` string that ranks
results by cosine similarity and adds a `score` to each item. Backends that can't do vectors
raise `NotImplementedError` for `query`.

Files written through `StoreBackend` become items in that store, durable and shared **across
threads**. The store is passed to `create_deep_agent`, not to the backend.

The `namespace` factory is the important part: it is what actually decides isolation. It is a
function the framework calls on every run, handing it the `Runtime`, which returns the tuple to
use for that run. Written inline it is a `lambda`, but a named function is the same thing:

```python
from langgraph.runtime import Runtime

def per_user(rt: Runtime) -> tuple[str, ...]:
    return (rt.context.user_id,)

StoreBackend(namespace=per_user)                       # identical to the lambda form
```

`rt` is not something you define or pass — it is the argument LangGraph hands the function on
each run: a `Runtime` object carrying the context you supplied at invoke time, the execution
identifiers, and (when deployed) server metadata. The `rt` in `lambda rt: ...` is the same
argument, just conventionally abbreviated.

One thing to not misread: `namespace=lambda rt: ...` resembles the **deprecated backend factory**
(`backend=lambda rt: StateBackend(rt)`, covered at the end of this chapter) but has nothing to do
with it. Wrapping the *backend* in a lambda is the deprecated pattern. Passing a function to
`namespace=` is current API.

```python
namespace=lambda rt: (rt.server_info.user.identity,)   # per user
namespace=lambda rt: (rt.server_info.assistant_id,)    # per agent, shared by all users
namespace=lambda rt: (rt.execution_info.thread_id,)    # per conversation
```

Those are three separate choices, not a menu you pick one from. **The tuple is a path, and you
compose it.** One component gives you one level of isolation; more components nest:

```python
namespace=lambda rt: (rt.server_info.user.identity,)                            # this user, all agents
namespace=lambda rt: (rt.server_info.assistant_id,)                             # this agent, all users
namespace=lambda rt: (rt.server_info.assistant_id, rt.server_info.user.identity)  # this user of this agent
namespace=lambda rt: (rt.server_info.user.identity, rt.execution_info.thread_id)  # this user, this conversation
```

Two consequences of it being a path rather than a set of tags:

- **Order matters.** `(assistant_id, user_id)` and `(user_id, assistant_id)` are different
  compartments. Put the level you will want to sweep across on the outside.
- **Search matches by prefix.** Searching `("alice",)` returns everything nested under it —
  `("alice", "memories")`, `("alice", "chat-7")`, and so on. So with `(user_id, thread_id)` you
  get per-conversation isolation *and* the ability to read all of one user's conversations in one
  query. With `(thread_id, user_id)` you get the same isolation and no such sweep.

Components are restricted to alphanumerics, hyphens, underscores, dots, `@`, `+`, colons, and
tildes; wildcards are rejected so nobody can inject a glob.

Note what the `Runtime` gives you, because it differs between local and deployed:

| | Available | Holds |
|--------------------|----------------------------|-----------------------------------------------|
| `rt.server_info` | Only on LangGraph Server | Assistant ID, graph ID, authenticated user |
| `rt.execution_info`| Always | Thread ID, run ID, checkpoint ID |
| `rt.context` | Always | Whatever *you* pass per run via `context_schema` |

So the `rt.server_info` examples above assume a deployment. Running locally there is no
authenticated user and no assistant, and you scope on context you supply yourself:

```python
namespace=lambda rt: (rt.context.user_id,)     # user_id you passed in at invoke time
```

**Set it explicitly.** The legacy default namespaces by `assistant_id`, which means every user of
that agent reads and writes the *same* memory — fine for a single-tenant assistant persona, a
data leak for a multi-user product. The parameter becomes required in v0.5.0.

### `CompositeBackend` — the router, and the pattern you actually want

```python
from deepagents.backends import CompositeBackend, StateBackend, StoreBackend

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    backend=CompositeBackend(
        default=StateBackend(),
        routes={"/memories/": StoreBackend(namespace=lambda rt: (rt.server_info.user.identity,))},
    ),
    store=InMemoryStore(),
)
```

Routes path prefixes to different backends, so one agent gets an ephemeral scratch space *and* a
durable memory in the same filesystem:

- `/workspace/plan.md` → `StateBackend`, thread-scoped, gone tomorrow
- `/memories/user-prefs.md` → `StoreBackend`, durable, readable from any future thread

Rules: **longer prefixes win** (`/memories/projects/` overrides `/memories/`), a path that
matches no route falls to `default`, and `ls`, `glob`, and `grep` aggregate across all routes
while showing the original path prefixes.

**Keep `StateBackend` as the `default`.** Deep Agents write internal data to the default backend —
offloaded tool results under `/large_tool_results/`, conversation history under
`/conversation_history/`. Make `FilesystemBackend` the default and that machinery litters your
project directory; make a persistent store the default and you pay to store it forever.

### `ContextHubBackend` — a LangSmith Hub repo

```python
from deepagents.backends import ContextHubBackend

agent = create_deep_agent(model="anthropic:claude-sonnet-4-6", backend=ContextHubBackend("my-agent"))
```

Durable, LangSmith-native storage without wiring a `BaseStore` yourself. The repo identifier is
`owner/name` or just `name`, and `LANGSMITH_API_KEY` must be set.

The repo layout is the interesting part. An *agent repo* holds top-level instructions and config
(`AGENTS.md`, `tools.json`) and gets mounted at the filesystem root; *skill repos* it links to
appear as subdirectories under `/skills/`. So a skill can be versioned, shared, and reused by
several agents independently, at the cost of your agent's context living across several repos.

Mechanically it pulls the repo tree lazily on first use and serves reads from an in-memory cache;
writes become Hub commits. Writes are optimistic against the last known commit hash, so if
another writer gets there first your push fails and you re-pull and retry. Best when you want
commit history on filesystem changes, or you are already on LangSmith and would rather not run a
store.

### Sandbox backends

An isolated environment — LangSmith, AgentCore, Daytona and others — providing the filesystem
tools *plus* `execute`. This is the production answer when an agent genuinely needs to run code,
and the one to reach for instead of `LocalShellBackend`.

Scope is a deliberate choice here, unlike every other backend:

- **Thread-scoped** (the default) — one sandbox per conversation, discarded with it.
- **Assistant-scoped** — one sandbox shared by every thread of an assistant. Files, installed
  packages, and cloned repos survive across conversations, which also means in-sandbox state
  accumulates: set a TTL, snapshot and reset, or clean up explicitly.

Sandboxes cost money for as long as they are alive, so shutting them down is part of your job.
And never put secrets inside one — anything an agent can read, a context-injected agent can
exfiltrate.

### Custom backends

Implement `BackendProtocol` to point the same filesystem tools at S3, a database, a remote filesystem,
whatever you already run.

## Access Control

The third thing the backend settles. There are three layers, in different places:

**Declarative path rules** are a `create_deep_agent` parameter, not a backend setting, and they
are evaluated *before* the backend is called:

```python
from deepagents import FilesystemPermission

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    backend=CompositeBackend(default=StateBackend(), routes={"/policies/": StoreBackend(...)}),
    permissions=[
        FilesystemPermission(operations=["write"], paths=["/policies/**"], mode="deny"),
    ],
)
```

**Backend-level boundaries** come with the backend you picked.
`FilesystemBackend(virtual_mode=True)` confining paths under `root_dir` is the main one — and in
the other direction, **declarative permissions do not apply to sandbox backends at all**, because
a shell routes around them.

**Policy hooks** cover what path rules cannot express: rate limiting, audit logging, content
inspection. You subclass a backend, or wrap any backend, and enforce the rule in the method:

```python
class GuardedBackend(FilesystemBackend):
    def write(self, file_path: str, content: str) -> WriteResult:
        if file_path.startswith("/protected/"):
            return WriteResult(error=f"Writes are not allowed under {file_path}")
        return super().write(file_path, content)
```

Note that the refusal is **returned, not raised** — the agent sees it as a tool result and can
react to it.

That is why access control belongs in a chapter about backends. The simple rules are configured
beside the backend, but anything stronger is implemented *inside* it, and the backend you chose
decides whether the simple rules apply at all.

## Per-Thread vs Cross-Thread

A **thread** is one conversation, identified by `thread_id`:

```python
config = {"configurable": {"thread_id": "conversation-42"}}
agent.invoke({"messages": [...]}, config)
```

| | Keyed by | Another thread sees it? |
|-----------------|--------------------------|-------------------------|
| State (checkpointer) | `thread_id` | No |
| Store | your `namespace` tuple | Yes, if the namespace matches |

Same `thread_id` continues a conversation; a new one starts fresh. This is the single most common
"my agent forgot everything" bug — the file was written to `StateBackend` under `thread-1` and
read back under `thread-2`. Nothing is broken; that is the contract.

Note that the store is **not** automatically global. `namespace=lambda rt: (rt.execution_info.thread_id,)`
gives you a store that is durable but still per-conversation. Cross-thread visibility is a
property of the namespace you choose, not of the store itself.

## Why the Medium Matters Beyond Not Losing Files

Picking a durable medium is not only about surviving a restart. Because LangGraph checkpoints
state as the run proceeds — after each step, by default — a durable *checkpointer* also buys you:

- **Crash recovery** — a run killed by a failure or timeout resumes from its last checkpoint
  instead of reprocessing. For a long agent that spawned twenty subagents, that is the difference
  between losing minutes and losing hours.
- **Indefinite human-in-the-loop pauses** — an interrupt can wait days and still resume exactly
  where it stopped. A checkpointer is *required* for human-in-the-loop at all, even the in-memory
  one.
- **Time travel** — every step is a snapshot you can rewind to and replay from.
- **An audit trail** — for payments and other irreversible actions, the exact state that led to
  the call is recorded.

### When the checkpoint is written

A durable checkpointer says *where* the copy goes; the `durability` setting says *when* it is
written. It is passed per run, on any graph execution method:

```python
agent.invoke({"messages": [...]}, config, durability="sync")
```

| Mode | Written | Trade-off |
|-----------|-------------------------------------------------|-----------------------------------------------|
| `"exit"` | Only when the run ends — normally, on error, or on a human-in-the-loop pause | Fastest; intermediate state is never saved, so a mid-run crash is unrecoverable |
| `"async"` | After each step, asynchronously, while the next step runs | The **default**; small chance a write is lost if the process dies mid-execution |
| `"sync"` | After each step, before the next one starts | Every checkpoint lands, at some performance cost |

This is why "durable" and "recoverable" are not quite the same thing: with `durability="exit"`
and a `PostgresSaver` you still get a durable record of the finished run, but nothing to resume
from if the process is killed halfway through.

`MemorySaver` and `InMemorySaver` both keep checkpoints in RAM; both lose everything on restart.
For production use `PostgresSaver`, or `SqliteSaver` for a local file. On LangSmith Deployment a
persistent checkpointer and a store are both provisioned automatically.

## Choosing

Six setups, each one complete, ending with the same coding assistant shown twice — once for
local development and once for production. Imports accumulate down the page rather than repeating.

### A scratch pad for one conversation

```python
from deepagents import create_deep_agent

agent = create_deep_agent(model="anthropic:claude-sonnet-4-6")
```

The default. Files live in state, are visible only within one `thread_id`, and vanish when the
process exits. Nothing to install, nothing to run.

### The same, surviving a crash mid-run

```python
from langgraph.checkpoint.postgres import PostgresSaver

checkpointer = PostgresSaver.from_conn_string(DB_URL)
checkpointer.setup()                    # once, at deploy time — creates the tables

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    checkpointer=checkpointer,
)
```

Same backend, same scope. The only change is that state snapshots now land in Postgres instead
of RAM, so a killed run resumes from its last step. Swap `PostgresSaver` for `SqliteSaver` to get
the same thing in a local file.

**Nothing is cleaned up for you.** Ending a run or a conversation deletes nothing — every step of
every thread stays in Postgres until something removes it, and long conversations accumulate
checkpoints that cost storage and slow queries down. Three ways to deal with it:

```python
checkpointer.delete_thread(thread_id)   # drop one conversation's checkpoints and writes
```

- Call `delete_thread` when a conversation is genuinely over.
- Otherwise run a scheduled job that deletes checkpoints older than N days.
- On Agent Server, set a **TTL** on threads instead — a time-to-live, an expiry after which the
  thread is deleted for you. Deleting a thread takes its runs and checkpoints with it.

### Files the agent can read back tomorrow

```python
from deepagents.backends import StoreBackend
from langgraph.runtime import Runtime
from langgraph.store.postgres import PostgresStore

def per_user(rt: Runtime) -> tuple[str, ...]:
    """Called by LangGraph on every run; `rt` is that run's Runtime."""
    return (rt.context.user_id,)

with PostgresStore.from_conn_string(DB_URL) as store:
    store.setup()                      # once, at deploy time — creates the tables

    agent = create_deep_agent(
        model="anthropic:claude-sonnet-4-6",
        backend=StoreBackend(namespace=per_user),
        store=store,
    )
    ...
```

`PostgresStore` is opened as a context manager so its connection is closed when you are done, the
same way `PostgresSaver` is. `setup()` creates the tables and only needs to run once, at deploy
time rather than on every start.

`per_user` is the [namespace factory](#storebackend--the-langgraph-store) — the framework calls it
on each run to pick the store compartment. It is usually written inline as a lambda; a named
function is the same thing and easier to read here.

"File" here means exactly one thing: something the agent created by calling `write_file` or
`edit_file`. It is *not* the agent's reply to the user, and *not* the conversation. Messages live
in graph state and are persisted by the checkpointer no matter which backend you choose — the
backend only ever governs the filesystem tools.

So this configuration sends every file the agent writes to the store, readable from any future
thread for that user.

**Note what does *not* carry over: the conversation.** Tomorrow is a new `thread_id`, so the
agent starts with an empty message history and no recollection of what was said. What it can do
is read the notes it left itself. If you want the dialogue itself to continue, that is not a
store — that is reusing the same `thread_id` with a durable checkpointer.

This setup is also usually more than you want: with `StoreBackend` as the whole backend, scratch
notes and offloaded tool output get permanently stored alongside things worth keeping. Hence the
next one.

### Both: a scratch pad and durable memory

```python
from deepagents.backends import CompositeBackend, StateBackend

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    memory=["/memories/AGENTS.md"],
    backend=CompositeBackend(
        default=StateBackend(),
        routes={
            "/memories/": StoreBackend(namespace=per_user),
        },
    ),
    store=store,
    checkpointer=checkpointer,
)
```

The one to reach for by default. Anything under `/memories/` persists per user across
conversations; everything else — including the agent's internal scratch files — stays thread-scoped
and disposable.

### A coding assistant editing your actual repo

```python
from deepagents.backends import FilesystemBackend

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    backend=CompositeBackend(
        default=StateBackend(),
        routes={
            "/workspace/": FilesystemBackend(root_dir="/abs/path/repo", virtual_mode=True),
        },
    ),
    checkpointer=checkpointer,
    interrupt_on={"write_file": True, "edit_file": True},   # approve writes before they land
)
```

Reads and writes under `/workspace/` hit real files; everything else stays in state, which keeps
`/large_tool_results/` out of your repo. `interrupt_on` pauses for approval before each write —
that needs a checkpointer, which is why one is passed here.

**What `virtual_mode` does.** On its own, `root_dir` is only a starting point for resolving
relative paths — the agent can still walk out of it with `../../etc/passwd`, `~/.aws/credentials`,
or a plain absolute path. The default is `virtual_mode=False`, which means **`root_dir` alone
gives you no protection at all**. Turning it on makes `root_dir` a boundary: paths are normalised
and confined beneath it, and `..`, `~`, and absolute paths pointing outside are rejected. Treat
it as mandatory whenever you set `root_dir`.

**Why not in production.** Not because of a missing feature, but because of what the backend
is — the agent reads and writes your real filesystem with the permissions of the process running
it. Any file that process can reach is in scope, including `.env` files, credentials, and SSH
keys; pair that with a tool that makes network calls and a prompt-injected agent has an
exfiltration route. Writes are immediate and irreversible, with no undo. `virtual_mode` narrows
the blast radius but does not change the nature of it.

That is an acceptable trade on your own machine, where you would have run those commands
yourself, and in CI against a checkout that holds no secrets. It is not acceptable in a web
server or HTTP API, where the input comes from users you do not control — use a sandbox backend
there instead.

### The same coding assistant, in production

Same job — an agent that reads and edits a codebase and runs commands against it — but the
filesystem it touches is a disposable container instead of your machine:

```python
from deepagents.backends import LangSmithSandbox
from langsmith.sandbox import SandboxClient

client = SandboxClient()
ls_sandbox = client.create_sandbox()
backend = LangSmithSandbox(sandbox=ls_sandbox)

backend.upload_files([                              # seed it: the repo is not there yet
    ("/src/index.py", open("src/index.py", "rb").read()),
    ("/pyproject.toml", open("pyproject.toml", "rb").read()),
])

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    system_prompt="You are a Python coding assistant with sandbox access.",
    backend=backend,
    checkpointer=checkpointer,
)
try:
    agent.invoke({"messages": [{"role": "user", "content": "Fix the failing test"}]}, config)
    for f in backend.download_files(["/src/index.py"]):   # collect the results
        if f.content is not None:
            print(f.path, f.content.decode())
finally:
    client.delete_sandbox(ls_sandbox.name)          # sandboxes bill until deleted
```

Lined up against the development version, four things change:

| | Development | Production |
|--------------------|--------------------------------|------------------------------------|
| Backend | `FilesystemBackend` behind a `/workspace/` route | The sandbox, as the whole backend |
| The files | Already there — your repo | Not there — you `upload_files` them in |
| Getting results | Already on disk, nothing to do | `download_files` before you tear it down |
| Blast radius of a mistake | Your actual working copy | A container you were going to delete |
| Shell access | None, unless you use `LocalShellBackend` | The `execute` tool, included |
| Cleanup | None | `delete_sandbox`, or you keep paying |

The important structural difference is the middle two rows. Locally the agent edits your files in
place, so "getting the result" is not a step. In a sandbox there are **two separate planes**: the
agent's own tools (`read_file`, `write_file`, `execute`) operate *inside* the container, while
`upload_files` and `download_files` are provider APIs your application code calls to move data
across the boundary. Forget the download and the work is deleted with the sandbox.

That is also the honest reason the local version cannot simply be promoted: it is not missing a
setting, it is a different shape of program.

### Invoking any of them

```python
config = {"configurable": {"thread_id": "conversation-42"}}
agent.invoke({"messages": [{"role": "user", "content": "..."}]}, config)
```

Reuse a `thread_id` to continue a conversation; generate a new one to start fresh. Omit it and
nothing is remembered between calls.

On LangSmith Deployment, delete the `store=` and `checkpointer=` arguments entirely — the
platform provisions both — and switch `rt.context.user_id` to `rt.server_info.user.identity`.

## Three Things Called "Memory"

The word is overloaded in the docs, so:

- **Short-term memory** — conversation history and scratch files for one thread. State plus a
  checkpointer. Managed for you.
- **Long-term memory** — facts and preferences that outlive the conversation. A store, surfaced
  to the agent as files, by convention under `/memories/`.
- **The `memory=` parameter** — a list of *file paths* the agent should treat as memory, e.g.
  `memory=["/memories/AGENTS.md"]`. It names the files; the backend decides where they live. Its
  sibling `skills=["/skills/"]` does the same for procedural instructions the agent loads on
  demand.

## Gotchas

| Symptom | Cause |
|---------------------------------------------|--------------------------------------------------------|
| File written in one session is gone in the next | `StateBackend` with a new `thread_id` — expected; route it to a store |
| Everything lost on deploy/restart | `InMemorySaver` / `InMemoryStore` in production |
| `StoreBackend` errors or writes nowhere | No `store=` passed to `create_deep_agent` |
| A file under a routed prefix stays ephemeral | Path doesn't match the route — `/memories.txt` is not `/memories/…` |
| Two users see each other's memories | No `namespace` factory; defaulted to `assistant_id` |
| `/large_tool_results/` appearing in your repo | `FilesystemBackend` used as the `default` instead of behind a route |
| Agent escapes `root_dir` | `virtual_mode` left at its `False` default |
| Human-in-the-loop never pauses | No checkpointer passed at all |

One API note: older examples pass a *factory* — `backend=lambda rt: StateBackend(rt)`. That form
is **deprecated since deepagents 0.5.0** (it still runs, with a warning). Backends now resolve
their runtime context internally, so pass instances: `backend=StateBackend()`.

## Cheat Sheet

```python
from deepagents import create_deep_agent
from deepagents.backends import CompositeBackend, StateBackend, StoreBackend
from langgraph.checkpoint.postgres import PostgresSaver     # durable, thread-scoped
from langgraph.store.postgres import PostgresStore          # durable, cross-thread

checkpointer = PostgresSaver.from_conn_string(DB_URL)       # .setup() once, at deploy time
store = PostgresStore.from_conn_string(DB_URL)              # docs open this with `with ... as store:`

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    memory=["/memories/AGENTS.md"],
    backend=CompositeBackend(
        default=StateBackend(),                             # scratch: thread-scoped
        routes={                                            # longest prefix wins
            "/memories/": StoreBackend(
                namespace=lambda rt: (rt.server_info.user.identity,),
            ),
        },
    ),
    checkpointer=checkpointer,   # durability of the state half
    store=store,                 # durability of the store half
)

config = {"configurable": {"thread_id": "conversation-42"}}  # same id = same conversation
agent.invoke({"messages": [{"role": "user", "content": "..."}]}, config)
```

| Scope ↓  Medium → | RAM (volatile) | Local file | Database |
|-------------------------------|-----------------|---------------------|--------------------------------|
| Thread — `StateBackend` | `InMemorySaver` | `SqliteSaver` | `PostgresSaver` |
| Cross-thread — `StoreBackend` | `InMemoryStore` | `AsyncSqliteStore` | `PostgresStore`, `RedisStore`, … |
| Everything — `FilesystemBackend` | — | your disk | — |

Three things to carry away: **the backend picks scope, the checkpointer/store picks the medium
and therefore durability**;
**`/memories/` routed to a `StoreBackend` is the one pattern you will use most**; and **keep
`StateBackend` as the composite default** so internal agent artifacts stay ephemeral.

---

Sources: [Backends](https://docs.langchain.com/oss/python/deepagents/backends),
[Memory](https://docs.langchain.com/oss/python/deepagents/memory),
[Going to production](https://docs.langchain.com/oss/python/deepagents/going-to-production),
[LangGraph persistence](https://docs.langchain.com/oss/python/langgraph/persistence).

Next: [Instructing the Agent](./02-instructing-the-agent.md) |
Back to [Deep Agents](./README.md) | [Index](../README.md)
