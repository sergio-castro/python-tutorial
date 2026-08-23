# Guardrails and Tracing

You can guard and instrument a tool call: approve it, block it, log it, retry it. The question
this chapter answers is whether the same is possible for the two things Deep Agents adds —
**skills** and **subagents**.

It is, and the reason is worth stating up front, because it makes the whole topic smaller than it
looks:

| What you want to control | What it actually is | So you control it with |
|--------------------------|------------------------------|----------------------------------------|
| A tool call | A tool call | `interrupt_on`, `wrap_tool_call` |
| Delegating to a subagent | A call to the **`task` tool** | The same things |
| Activating a skill | A **`read_file`** on `SKILL.md` | Filesystem permissions, backend routing |

Subagents are the tool case *exactly*: delegation happens through a tool named `task`, so
anything that works on tools works on delegation with no new API. Skills are not a tool case at
all — a skill is a file, activation is a read, and files are governed by permissions and by which
backend the path routes to.

Neither has a bespoke "skill hook" or "subagent hook", and neither needs one.

## Approval Gates: `interrupt_on`

`interrupt_on` maps **tool names** to interrupt configurations, and pauses the run for a human
before those calls execute:

```python
agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    interrupt_on={
        "write_file": True,                                        # approve, edit, reject, respond
        "read_file": False,                                        # no interrupt
        "notify_email": {"allowed_decisions": ["approve", "reject"]},   # no editing
    },
    checkpointer=checkpointer,   # required — the run must be able to pause and resume
)
```

Because it keys on tool names and `task` is a tool, the same mechanism reaches delegation. The
docs do not show a worked `"task"` example, so treat this as following from the mechanism rather
than from a documented recipe — but the mechanism is explicit: `interrupt_on` accepts any tool
name, and `SubAgentMiddleware` contributes `task` to the tool set.

The checkpointer is not optional. Pausing means persisting the run and resuming later, which is
exactly what a checkpointer provides — see
[Managing the Context Window](./05-context-window.md) and
[Backends, State, and Memory](./01-backends-and-memory.md).

## Instrumenting Everything: `wrap_tool_call`

One [middleware](../langchain/01-middleware.md) hook sees every tool call the agent makes —
your tools, the built-in filesystem tools, `execute`, and `task`. That makes it the single place
to log, time, count, or veto anything:

```python
from typing import Callable

from langchain.agents.middleware import AgentMiddleware, ToolCallRequest
from langchain_core.messages import ToolMessage
from langgraph.types import Command


class DelegationAudit(AgentMiddleware):
    def wrap_tool_call(
        self,
        request: ToolCallRequest,
        handler: Callable[[ToolCallRequest], ToolMessage | Command],
    ) -> ToolMessage | Command:
        name = request.tool_call["name"]

        if name == "task":                       # a subagent is about to be dispatched
            log.info("delegating", args=request.tool_call["args"])
        elif name == "read_file" and request.tool_call["args"]["file_path"].endswith("SKILL.md"):
            log.info("skill activated", path=request.tool_call["args"]["file_path"])

        return handler(request)
```

Declining to call `handler` blocks the call outright, which is how the same hook becomes a
guardrail rather than a logger. And since wrap hooks nest with the **first** middleware
outermost, a guardrail belongs at the front of your `middleware` list.

Note the skill branch: there is no "skill activated" event to subscribe to. You infer it from a
read of a `SKILL.md` path, because that is genuinely all activation is.

## Guarding Skills

Skill control splits into three questions, each with a different answer.

**Which skills can this user even see?** Two approaches. Build the `skills` list before creating
the agent and pass different paths for different roles; or route `/skills/` to a `StoreBackend`
with a namespace factory keyed on user or tenant, and populate each namespace with only what that
identity should have. The second is the one that scales.

**Can the agent modify them?** Deny writes with filesystem permissions. Discovery and reading
still work; only your application or an admin workflow updates the store:

```python
from deepagents import FilesystemPermission

permissions=[
    FilesystemPermission(operations=["write"], paths=["/skills/**"], mode="deny"),
]
```

**Should changes need a human?** `mode="interrupt"` instead of `"deny"`, or `interrupt_on` on the
write tools.

### Permission rules

A rule is operations × paths × mode, where mode is `"allow"`, `"deny"`, or `"interrupt"`
(defaulting to `"allow"`). With `"interrupt"`, a matching `write_file`, `edit_file`, or `delete`
raises a human-in-the-loop interrupt instead of running, and a reviewer can approve, edit, or
reject.

**Matching is first-match-wins, so order is the whole game.** Specific rules before broad ones:

```python
# Correct: deny .env, allow the workspace, deny everything else
[
    FilesystemPermission(operations=["read", "write"], paths=["/workspace/.env"], mode="deny"),
    FilesystemPermission(operations=["read", "write"], paths=["/workspace/**"], mode="allow"),
    FilesystemPermission(operations=["read", "write"], paths=["/**"], mode="deny"),
]

# Bug: /workspace/** matches .env first, so the deny is never reached
[
    FilesystemPermission(operations=["read", "write"], paths=["/workspace/**"], mode="allow"),
    FilesystemPermission(operations=["read", "write"], paths=["/workspace/.env"], mode="deny"),
    FilesystemPermission(operations=["read", "write"], paths=["/**"], mode="deny"),
]
```

The broken version looks right and fails silently — the agent reads your `.env` and nothing
reports an error.

### Two limits worth knowing

**Permissions do not apply to sandbox backends.** A shell can reach any path, so path rules are
meaningless there. For a `CompositeBackend` with a sandbox default, every permission path must be
scoped under a known non-sandbox route prefix; a rule like `/**` that covers the sandbox raises
`NotImplementedError` rather than pretending to work. Custom logic beyond paths belongs in a
[backend policy hook](./01-backends-and-memory.md#access-control).

**Subagents inherit permissions, and setting them replaces rather than extends.** A subagent with
its own `permissions` does not get the parent's rules as a base — it gets only what you listed.

## Tracing

Every subagent's `name` is written to the **`lc_agent_name`** metadata key on every run it
produces. That single fact is what makes subagent runs findable.

In the LangSmith UI: open the tracing project, switch to the **Runs** view to see individual
spans, add a **Metadata** filter with key `lc_agent_name` and the subagent's name as the value.
Save it as a named view if you will use it again.

Programmatically:

```python
from langsmith import Client

client = Client()

runs = client.list_runs(                       # one specific subagent
    project_name="<your-project>",
    filter='has(metadata, \'{"lc_agent_name": "research-agent"}\')',
)

runs = client.list_runs(                       # any named subagent, excluding the main agent
    project_name="<your-project>",
    filter="has(metadata, 'lc_agent_name')",
)
```

That second filter is the useful one for monitoring: it isolates all delegated work from the
orchestrator's own.

To control how much your own middleware records into traces, use `TracePolicy` — see
[Middleware](../langchain/01-middleware.md#tracing).

## Summary

| Goal | Mechanism |
|---------------------------------------|-----------------------------------------------|
| Approve a tool call before it runs | `interrupt_on={"tool_name": True}` + checkpointer |
| Approve a delegation | The same — `task` is a tool |
| Approve a file write | `interrupt_on`, or a permission with `mode="interrupt"` |
| Block a path outright | `FilesystemPermission(..., mode="deny")` |
| Make skills read-only | Deny writes under `/skills/**` |
| Show each user different skills | Namespaced `StoreBackend`, or a per-role `skills` list |
| Log or veto anything, uniformly | `wrap_tool_call` middleware, placed first |
| Custom rules beyond paths | Subclass or wrap the backend |
| Find one subagent's runs | Filter on `lc_agent_name` in LangSmith |

## Gotchas

| Symptom | Cause |
|---------------------------------------------|--------------------------------------------------------|
| `interrupt_on` never pauses | No checkpointer — pausing requires persistence |
| A deny rule never fires | An earlier, broader rule matched first — first-match-wins |
| Permissions silently ignored | The backend is a sandbox; path rules do not apply there |
| `NotImplementedError` from a permission | A rule's path covers a sandbox route in a composite backend |
| A subagent can reach more than the parent | Its `permissions` replaced the parent's instead of extending |
| No trace for a delegated run | Filter on `lc_agent_name`; subagent runs are separate spans |
| A guardrail middleware runs too late | Wrap hooks nest — put it first in the list to be outermost |

Three things to carry away: **delegation is a tool call, so tool machinery already covers it**;
**skill activation is a file read, so filesystem permissions already cover it**; and
**`wrap_tool_call` is the one hook that sees all of it**.

---

Sources: [Permissions](https://docs.langchain.com/oss/python/deepagents/permissions),
[Skills](https://docs.langchain.com/oss/python/deepagents/skills),
[Subagents](https://docs.langchain.com/oss/python/deepagents/subagents),
[Human-in-the-loop](https://docs.langchain.com/oss/python/deepagents/human-in-the-loop),
[Custom middleware](https://docs.langchain.com/oss/python/langchain/middleware/custom).

Previous: [Managing the Context Window](./05-context-window.md) |
Back to [Deep Agents](./README.md) | [Index](../README.md)
