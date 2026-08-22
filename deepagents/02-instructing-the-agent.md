# Instructing the Agent

The previous chapter was about where an agent's *files* go. This one is about where its
*instructions* come from — every channel through which you tell a deep agent what it is supposed
to do, and when each one reaches the model.

There are five, and they differ on two axes: **when the text enters the context window**, and
**whose context window it is**.

| Channel | Set with | Enters the context | Written by |
|-------------------|--------------------|-------------------------------------------|-----------------------|
| **System prompt** | `system_prompt=` | Always, at the very top | You |
| **Memory** | `memory=[...]` | Always, in full | You; the agent may edit it |
| **Skills** | `skills=[...]` | Name and description always; the body only when invoked | You, or the agent |
| **Tool prompts** | `tools=[...]`, and the harness | Always | You, for your tools; the harness for its own |
| **Subagents** | `subagents=[...]` | Never — each gets a *separate* context | You |

The first four all pile into one system prompt. Subagents are the odd one out: they do not add
instructions to the main agent, they create a second agent with instructions of its own. That is
why they belong here at all — a subagent's `system_prompt` is a real instruction channel — and
also why they are last.

## The Complete System Prompt

Three words appear below before they are properly introduced:

- The **harness** is the machinery Deep Agents wraps around the model — the built-in prompts,
  tools, and plumbing that turn a plain chat model into a deep agent. When something happens that
  you did not write, the harness did it.
- **Middleware** is how that machinery is assembled: small, ordered components, each of which can
  add tools, add text to the system prompt, or act around each model call. The filesystem tools
  come from one, subagents from another. You can add your own, and a
  [harness profile](./03-harness-profiles.md) can add or remove them per model.
- The **`task` tool** is how the agent delegates to a subagent, covered at the end of this
  chapter.

Everything in the first four rows is assembled, in this order, into the single system message the
model sees at the start of a run:

1. Your `system_prompt`, if provided
2. The base agent prompt built into Deep Agents
3. The memory prompt — your `AGENTS.md` contents plus usage guidelines (only if `memory` is set)
4. The skills prompt — skill locations, and each skill's frontmatter (only if `skills` is set)
5. The virtual filesystem prompt — docs for `ls`, `read_file`, `write_file`, and friends
6. The subagent prompt — how to use the `task` tool
7. Prompts contributed by any custom middleware you added
8. The human-in-the-loop prompt (only if `interrupt_on` is set)

Two things follow from that list. Your prompt is **first but not alone** — you are prepending to
a harness prompt, not replacing it, so there is no need to explain the filesystem tools yourself.
And items 3–8 appear **only when you configure the corresponding feature**, so an agent with no
`memory`, no `skills`, and no `interrupt_on` has a markedly smaller baseline prompt.

## 1. `system_prompt` — the role

```python
from deepagents import create_deep_agent

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    system_prompt=(
        "You are a research assistant specializing in scientific literature. "
        "Prefer primary sources, and always cite what you used."
    ),
)
```

Use it for what the agent *is*: role, tone, standing constraints, output conventions. It is the
right place for anything that applies to every single turn and is short enough that paying for it
on every request is fine.

It is the wrong place for a long procedure used once a week — that is a skill — and for project
facts that change independently of your code — that is memory.

## 2. `memory=` — facts that are always true

```python
agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    memory=["/project/AGENTS.md", "~/.deepagents/preferences.md"],
)
```

`memory` takes a list of **file paths**, not text. The files are read through the backend, and
their contents are loaded into the system prompt in full, on every run. The convention is
[`AGENTS.md`](https://agents.md/).

### What "always loaded" means

Not "appended to the conversation". Memory goes into the **system message**, which is rebuilt and
re-sent on every request. It never becomes part of the message history.

Three things follow, and they are the practical payoff of that choice:

- **It is re-sent, not accumulated.** Each request carries the file's *current* contents. Edit
  the file and the next request carries the new version, with no trace of the old one anywhere.
  A correction posted into the history would leave the original sitting there, still being read.
- **It cannot be summarized away.** Compaction rewrites history; the system message is assembled
  fresh each turn. See [Managing the Context Window](./05-context-window.md).
- **You pay for it every single turn**, relevant or not.

Where in the system message? In the assembly order from
[The Complete System Prompt](#the-complete-system-prompt), memory is **item 3** — after your own
`system_prompt` and the built-in base prompt, and before the skills, filesystem, and subagent
sections. So: near the front, but not the very front.

One caveat before you build anything on that exact position. The harness deliberately runs its
memory middleware **late** in the middleware stack, after the prompt-caching middleware,
specifically so that editing a memory file is less likely to invalidate the cached prompt prefix.
The composition order tells you the reading order the model sees; it is not a promise about byte
offsets relative to the cache boundary. Rely on "memory is near the top of the system message and
cheap to change", not on anything finer.

### Why a file, and not just text?

The content ends up inline in the system prompt anyway, so the indirection looks pointless until
you ask what the alternatives cannot do. A literal string in your source is fixed forever. A
message posted into the history dies with the thread and can be summarized away. A file avoids
both, and the reason is simple: **memory has to outlive the conversation, and the message history
*is* the conversation.**

Four things follow from it being a file:

- **The backend decides persistence and scope.** `/memories/AGENTS.md` routed to a `StoreBackend`
  survives across conversations and can be scoped per user; the same path on a plain
  `StateBackend` does not survive at all. That is the whole subject of
  [Backends, State, and Memory](./01-backends-and-memory.md).
- **The agent can change it.** `edit_file` works on memory files like any other, which is how an
  agent records a preference and still has it next week. Whether it *may* is a permissions
  question, not a memory one.
- **It survives compaction.** Being re-injected into the system prompt every turn, it is never at
  risk of being summarized away. Real and useful, though a consequence of the design rather than
  the motivation for it.
- **Other agents can read it.** Subagents, later threads, and anything else sharing the store
  namespace see the same file.

The cost of all this is that **always loaded means always paid for**: memory is billed on every
request whether or not it is relevant. Keep it to things that genuinely apply everywhere —
project conventions, user preferences, guidelines you want honoured unconditionally.

## 3. `skills=` — procedures loaded on demand

```python
agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    skills=["/skills/research/", "/skills/web-search/"],
)
```

A skill is a directory containing a `SKILL.md`. Skills exist to solve the problem memory has:
detailed multi-step instructions that you want available but not resident in context.

### Progressive disclosure

Skills load in three levels, and only the first is unconditional:

| Level | What loads | When |
|-------------------|-----------------------------------------------|--------------------------------------|
| 1. Metadata | `name` and `description` from the frontmatter | At startup, for *every* configured skill |
| 2. Instructions | The full `SKILL.md` body | When the agent decides the skill applies |
| 3. Resources | Files under `scripts/`, `references/`, `assets/` | Later still, if the body references them |

`SkillsMiddleware` handles levels 1 and 2 — it scans the configured paths at startup, parses each
frontmatter, and injects name and description into the system prompt; when the agent invokes a
skill it reads the body with `read_file`. Level 3 is just the model following instructions and
reading more files.

So the cost of *having* a skill is its frontmatter. The cost of *using* one is its body. You can
attach many skills to an agent cheaply, which is the whole design.

### Activation is one-way

Nothing unloads a skill once it has been read. Level 2 happens through `read_file`, so the body
arrives as an ordinary **tool result in the message history** — and stays there for the rest of
the thread, resent with every subsequent request, exactly like any other tool result.

Progressive disclosure therefore saves you the cost of skills you *never* use. It does not save
you the cost of a skill after you have used it. Two things follow:

- The body size is a **recurring** cost from activation onward, not a one-off read. That is the
  real force behind the ~5,000-token guidance.
- What eventually clears it is context compression, not the skill system:
  [summarization](./05-context-window.md) once the window fills, or offloading if the body
  somehow exceeded the 20,000-token threshold.

If a procedure is genuinely enormous, keep `SKILL.md` thin and push the bulk into
`references/`, which the agent reads only if the instructions send it there — and which it can
read a piece at a time.

**And when summarization does eventually fire?** The body is neither preserved nor deliberately
dropped: no skill-specific handling is documented, and none is implied by the mechanism. The body
is a tool result, so it is treated exactly like every other tool result. If it falls inside the
recent window that compaction keeps, it survives verbatim; if it falls outside, it goes to the
summarizer with everything else, and what remains is whatever a summary of session intent,
artifacts, and next steps happens to retain. Step-by-step procedural detail is not guaranteed to
survive that.

**Nothing reloads it for you.** There is no re-activation step: the harness does not notice that a
skill was compacted away, and the agent carries on with whatever the summary left behind. If it
needs the procedure again, the full text comes back only when the agent calls `read_file` on the
skill itself — which it can always do, since the file is untouched on the backend and its
description is still in the system prompt.

Whether it *will* is a judgement the model makes. That is the risk worth naming: after compaction
the agent may hold a confident-sounding paraphrase of a detailed procedure and act on that rather
than re-reading. Nothing marks the summary as lossy.

Four ways to keep a complex skill safe, roughly in order of how much they help:

- **Compact between tasks, not during one.** `create_summarization_tool_middleware` gives the
  agent a `compact_conversation` tool so it can compress at a natural boundary instead of
  mid-procedure.
- **Raise `keep`** so a recently activated skill stays inside the verbatim window
  ([Managing the Context Window](./05-context-window.md)).
- **Keep `SKILL.md` short and push detail into `references/`.** A short body is cheap to re-read,
  and a one-line instruction to consult a reference file is far more likely to survive
  summarization than the reference content itself.
- **If something must never be lost, it is not a skill.** Put it in memory, which lives in the
  system prompt and is immune to compaction.

### Anatomy

```
skills/
└── data-pipeline/
    ├── SKILL.md          # name must match the directory: data-pipeline
    ├── scripts/
    ├── references/
    └── assets/
```

```md
---
name: langgraph-docs
description: Use this skill for requests related to LangGraph in order to fetch relevant
  documentation to provide accurate, up-to-date guidance.
license: MIT
compatibility: Requires internet access for fetching documentation URLs
metadata:
  author: langchain
  version: "1.0"
allowed-tools: fetch_url
---

# langgraph-docs

Use the fetch_url tool to read https://docs.langchain.com/llms.txt, then fetch relevant pages.
```

| Field | Required | Notes |
|-----------------|----------|--------------------------------------------------------------|
| `name` | Yes | Lowercase alphanumeric with hyphens, 1–64 chars. **Must match the parent directory name.** |
| `description` | Yes | What it does *and when to use it*. Max 1,024 characters. |
| `license` | No | Name, or a reference to a bundled licence file |
| `compatibility` | No | Environment requirements. Max 500 characters. |
| `metadata` | No | Arbitrary key–value pairs |
| `allowed-tools` | No | Space-separated pre-approved tools. Experimental. |

### The description is the whole interface

At startup the description is the *only* thing the agent knows about a skill. It is not
documentation, it is a matching rule, so it has to say both what the skill does and when to reach
for it:

```yaml
# Good: specific about what and when
description: >-
  Extract text and tables from PDF files, fill PDF forms, and merge
  multiple PDFs. Use when working with PDF documents or when the user
  mentions PDFs, forms, or document extraction.

# Poor: too vague for reliable matching
description: Helps with PDFs.
```

Overlapping descriptions across skills make the agent pick the wrong one or dither between them.
If two skills sound alike, merge them.

Keep the frontmatter short and the `SKILL.md` body under roughly 5,000 tokens — the Agent Skills
specification suggests under 500 lines. Longer reference material belongs in `references/`, cited
from the body, where it stays at level 3.

## 4. Tool prompts — the channel that is easy to forget

Every tool the model can see contributes text to its context: a name, a schema, and a
description. That is an instruction channel whether you treat it as one or not.

**The harness contributes its own.** Middleware that adds capabilities also appends usage guidance
to the system prompt — the filesystem prompt documenting `ls`/`read_file`/`write_file`/`edit_file`/
`delete`/`glob`/`grep` (plus `execute` on a sandbox backend), the subagent prompt covering the
`task` tool, the human-in-the-loop prompt when `interrupt_on` is set, and a local-context prompt
in the CLI.

**Your tools contribute their descriptions.** Write them as instructions — say when to use the
tool, not only what it does, and describe every argument:

```python
from langchain.tools import tool


@tool(parse_docstring=True)
def search_orders(user_id: str, status: str, limit: int = 10) -> str:
    """Search for user orders by status.

    Use this when the user asks about order history or wants to check
    order status. Always filter by the provided status.

    Args:
        user_id: Unique identifier for the user
        status: Order status: 'pending', 'shipped', or 'delivered'
        limit: Maximum number of results to return
    """
```

Two levers when this gets expensive. Unused built-in tools still send their full schemas every
turn, so `excluded_tools` removes ones the agent should never call — `write_file` or `execute` on
a read-only agent — and shrinks the baseline prompt for the whole run. And a [harness profile](./03-harness-profiles.md)'s
`tool_description_overrides`, keyed by tool name, rewrites a description for a specific model or
provider.

## 5. Subagents — instructions in a separate context

A subagent is not more instruction for your agent. It is a second agent, with its own system
prompt and its own context window, that the main agent calls through the `task` tool and gets a
single result back from.

The reason is context bloat. Tools with large outputs — web search, file reads, database
queries — fill the main context with intermediate junk. A subagent absorbs all of that and returns
only the answer.

Worth it for multi-step work that would clutter the main context, specialised domains needing
their own instructions or tools, and tasks that want a different model. Not worth it for
single-step tasks, or when the main agent actually needs the intermediate detail.

### Defining one

```python
agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    subagents=[
        {
            "name": "code-reviewer",
            "description": "Reviews a diff for correctness and style. Use after code changes.",
            "system_prompt": "You are a meticulous reviewer. Report findings as a numbered list.",
            "tools": [read_diff],
            "model": "anthropic:claude-sonnet-4-6",
        }
    ],
)
```

`name`, `description`, and `system_prompt` are required. The `description` matters more than it
looks: it is what the main agent reads when deciding whether to delegate, so write it
action-oriented and specific, exactly like a skill description.

### What a subagent inherits

This table is the part that causes surprises:

| Field | Inherits from the main agent? |
|-------------------|------------------------------------------------------------|
| `system_prompt` | **No** — required, every custom subagent writes its own |
| `tools` | Yes, by default. Specifying it replaces the inherited set entirely |
| `model` | Yes, by default |
| `interrupt_on` | Yes, by default; the subagent's value wins |
| `permissions` | Yes, by default; setting it **replaces** the parent's rules |
| `middleware` | **No** |
| `skills` | **No** — only the `general-purpose` subagent inherits them |

Skill state is fully isolated: a subagent's loaded skills are invisible to the parent and vice
versa. A subagent with its own `skills` runs its own `SkillsMiddleware`.

Also available: `response_format`, which makes the subagent return JSON matching a schema instead
of free text — the parent then gets structured data rather than prose to re-parse.

### `CompiledSubAgent`

When the delegate is a whole LangGraph graph rather than a prompt-and-tools agent:

| Field | Type | Notes |
|---------------|------------|--------------------------------------------|
| `name` | `str` | Required |
| `description` | `str` | Required |
| `runnable` | `Runnable` | Required — a graph you have already `.compile()`d |

Note the dictionary form has no `runnable` field and this one has no `system_prompt` — the graph
*is* the behaviour, so there is nothing left for the harness to prompt.

**The common route** is LangChain's `create_agent`, which returns a compiled graph already:

```python
from deepagents import CompiledSubAgent, create_deep_agent
from langchain.agents import create_agent

custom_graph = create_agent(
    model="anthropic:claude-sonnet-4-6",
    tools=specialized_tools,
    system_prompt="You are a specialized agent for data analysis...",
)

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    system_prompt="You are a research coordinator.",
    subagents=[
        CompiledSubAgent(
            name="data-analyzer",
            description="Specialized agent for complex data analysis tasks",
            runnable=custom_graph,
        )
    ],
)
```

**The other route** is a graph you wrote yourself, when the delegate needs deterministic control
flow — a fixed sequence, a loop, a branch — rather than a model deciding what to do next:

```python
import operator
from typing_extensions import Annotated, TypedDict

from langgraph.graph import START, END, StateGraph


class ReviewState(TypedDict):
    messages: Annotated[list, operator.add]      # required — see below


def lint(state: ReviewState) -> dict: ...
def summarise(state: ReviewState) -> dict: ...

review_graph = (
    StateGraph(ReviewState)
    .add_node("lint", lint)
    .add_node("summarise", summarise)
    .add_edge(START, "lint")
    .add_edge("lint", "summarise")
    .add_edge("summarise", END)
    .compile()                                   # required — pass a compiled graph
)

reviewer = CompiledSubAgent(
    name="code-reviewer",
    description="Lints a diff and summarises the findings. Use after code changes.",
    runnable=review_graph,
)
```

One hard requirement for a hand-written graph: **its state must have a `messages` key.** That is
the channel the parent uses to hand the task in and read the result back out. A graph without it
will not work as a subagent.

### The `general-purpose` subagent

Deep Agents adds a synchronous `general-purpose` subagent automatically unless you supply one
with that name. It has the filesystem tools, and it is the only subagent that inherits the main
agent's skills. Pass your own subagent named `general-purpose` to replace it.

To run with no `task` tool at all, two steps: set
`general_purpose_subagent=GeneralPurposeSubagentProfile(enabled=False)` on the active
[harness profile](./03-harness-profiles.md), and pass no synchronous subagents. `SubAgentMiddleware` is only attached when at least
one synchronous subagent exists. Do not try to remove it with `excluded_middleware` — it is
required scaffolding, and listing it raises `ValueError`.

## Choosing a Channel

| The instruction is… | Put it in |
|-----------------------------------------------------|--------------------------|
| Who the agent is; rules for every turn | `system_prompt` |
| A project or user fact that outlives your code | a `memory` file |
| A long procedure used occasionally | a skill |
| When and how to call one specific tool | that tool's description |
| Work that would flood the context with intermediates | a subagent |
| Something the agent should learn and keep | a memory file it can edit |

The reliable instinct: **if it is not needed on every turn, do not put it in the system prompt.**
Memory and `system_prompt` are always-on and always-billed; skills and subagents are the two ways
to keep instructions available without keeping them resident.

## Gotchas

| Symptom | Cause |
|--------------------------------------------|--------------------------------------------------------|
| A skill is never picked up | Its `description` says what it does but not *when* to use it, or it overlaps another skill |
| Context keeps growing after a skill ran | Activation is one-way — the body stays in history until compression removes it |
| Skill fails to load at all | `name` in the frontmatter does not match the directory name |
| Subagent ignores your main instructions | `system_prompt` does not inherit — it only knows what you gave it |
| Subagent can't use a skill | Skills do not inherit; pass `skills` on the subagent (only `general-purpose` inherits) |
| `CompiledSubAgent` never returns anything usable | Its graph state has no `messages` key |
| Subagent lost its tools | Setting `tools` replaces the inherited set rather than adding to it |
| Memory file is empty at startup | The path is not present in the configured backend |
| Prompt is huge before you write anything | Unused built-in tools ship their schemas each turn — see `excluded_tools` |
| `excluded_middleware` raises `ValueError` | You tried to remove `SubAgentMiddleware`; use the profile knob instead |

## Cheat Sheet

```python
agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    system_prompt="Who the agent is. Always loaded, always billed.",
    memory=["/memories/AGENTS.md"],        # file paths; always loaded in full
    skills=["/skills/"],                   # frontmatter always; body on demand
    tools=[search_orders],                 # descriptions are instructions
    subagents=[{                           # separate context, separate prompt
        "name": "code-reviewer",
        "description": "Reviews a diff. Use after code changes.",
        "system_prompt": "You are a meticulous reviewer.",
    }],
)
```

| | Cost at startup | Cost when used |
|-----------------|-------------------------|-----------------------------|
| `system_prompt` | Full text, every turn | — |
| `memory` | Full file, every turn | — |
| Skills | Frontmatter only | Body, then any resources |
| Subagents | `name` + `description` | Runs in its own context; only the result returns |

Three things to carry away: **the system prompt is assembled from many sources, and yours is
merely first**; **skills exist so instructions can be available without being resident**; and
**a subagent inherits tools and model but never a system prompt**.

---

Sources: [Context engineering](https://docs.langchain.com/oss/python/deepagents/context-engineering),
[Skills](https://docs.langchain.com/oss/python/deepagents/skills),
[Subagents](https://docs.langchain.com/oss/python/deepagents/subagents),
[Memory](https://docs.langchain.com/oss/python/deepagents/memory).

Previous: [Backends, State, and Memory](./01-backends-and-memory.md) |
Next: [Harness Profiles](./03-harness-profiles.md) |
Back to [Deep Agents](./README.md) | [Index](../README.md)
