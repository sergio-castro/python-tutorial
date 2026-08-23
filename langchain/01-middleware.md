# Middleware

Middleware is how you get between an agent and the things it does. It is the extension point for
retries, logging, approval gates, guardrails, caching, and dynamic prompts — and it is the
machinery [Deep Agents](../deepagents/README.md) itself is assembled from.

A middleware is an object with **hooks**. You implement the hooks you care about; the agent calls
them at fixed points around its loop.

## The Hooks

There are two kinds, and the difference matters more than the names suggest.

**Node-style hooks** observe and update state at a point in time. They run, they may return a
state update, and the agent continues.

| Hook | Runs |
|-----------------|---------------------------------------------|
| `before_agent` | Before the agent starts, once per invocation |
| `before_model` | Before each model call |
| `after_model` | After each model response |
| `after_agent` | After the agent completes, once per invocation |

**Wrap-style hooks** surround an operation and control whether it happens at all. You are handed
a `handler` and decide to call it **zero times** (short-circuit), **once** (normal), or **many
times** (retry).

| Hook | Wraps |
|-------------------|-----------------|
| `wrap_model_call` | Each model call |
| `wrap_tool_call` | Each tool call |

That zero-times case is what makes wrap hooks a control mechanism rather than a logging one. A
`wrap_tool_call` that declines to call the handler has blocked the tool.

## Writing One

Two equivalent forms. Decorators for a single hook:

```python
from typing import Callable

from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse


@wrap_model_call
def retry_model(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse],
) -> ModelResponse:
    for attempt in range(3):
        try:
            return handler(request)
        except Exception as e:
            if attempt == 2:
                raise
            print(f"Retry {attempt + 1}/3 after error: {e}")
```

A class when you need several hooks or constructor arguments:

```python
from typing import Callable

from langchain.agents.middleware import AgentMiddleware, ToolCallRequest
from langchain_core.messages import ToolMessage
from langgraph.types import Command


class ToolMonitoringMiddleware(AgentMiddleware):
    def wrap_tool_call(
        self,
        request: ToolCallRequest,
        handler: Callable[[ToolCallRequest], ToolMessage | Command],
    ) -> ToolMessage | Command:
        print(f"Executing tool: {request.tool_call['name']}")
        print(f"Arguments: {request.tool_call['args']}")
        try:
            result = handler(request)
            print("Tool completed successfully")
            return result
        except Exception as e:
            print(f"Tool failed: {e}")
            raise
```

`request.tool_call["name"]` and `["args"]` are the two fields you will reach for constantly: they
are how a single middleware can behave differently per tool. `request.override(...)` produces a
modified request to pass on instead.

Pass instances to the agent:

```python
agent = create_agent(model="gpt-5.5", tools=[...], middleware=[ToolMonitoringMiddleware()])
```

## Execution Order

With `middleware=[m1, m2, m3]`:

- **`before_*` hooks run first to last** — `m1`, `m2`, `m3`.
- **`after_*` hooks run last to first** — `m3`, `m2`, `m1`.
- **`wrap_*` hooks nest** — `m1` wraps `m2` wraps `m3` wraps the actual call.

The nesting rule is the one to internalise: the **first** middleware in the list is the
**outermost** wrapper. It sees the request first and the response last, and it can prevent
everything after it from running at all. Put your guardrail first; put your metrics last if you
want them to measure the real work rather than the guardrail's rejection.

## Tracing

Middleware hook spans trace their inputs and outputs by default, which is usually what you want
and occasionally far too much — a hook that only looks at a tool name does not need the whole
message history attached to its span.

`TracePolicy` shapes what gets recorded. It accepts `process_inputs` and `process_outputs`
callables that transform the traced value, and `omit_payload` drops it altogether:

```python
from langchain.agents.middleware import AgentMiddleware, TracePolicy, omit_payload


class MyMiddleware(AgentMiddleware):
    trace_policy = TracePolicy(process_inputs=omit_payload)
```

Set a default for every middleware at once:

```python
from langchain.agents.middleware import configure_trace_policy, TracePolicy, omit_payload

configure_trace_policy(TracePolicy(process_inputs=omit_payload))   # pass None to clear
```

A middleware's own `trace_policy` overrides the global default. This requires
`langchain>=1.3.15`.

## Practical Notes

- **Keep each middleware to one job.** They compose; a single class doing four things does not.
- **Do not let a middleware crash the agent.** An exception in a hook propagates. If your logging
  sink is down, that should not end the run.
- **Order is configuration.** Two correct middlewares in the wrong order can be wrong together —
  see the nesting rule above.

---

Next: how Deep Agents uses this to guard delegation and skills —
[Guardrails and Tracing](../deepagents/06-guardrails-and-tracing.md).

Sources: [Custom middleware](https://docs.langchain.com/oss/python/langchain/middleware/custom),
[Tools](https://docs.langchain.com/oss/python/langchain/tools).

Back to [Index](../README.md)
