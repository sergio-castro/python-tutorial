# Tools

A tool is a callable with a well-defined input and output that you hand to an agent. The model
decides when to call it and with what arguments; your code runs and returns a result.

This is the foundation the other two chapters build on, because in LangChain a great deal that
sounds like a separate feature turns out to be a tool underneath — including
[subagents and skills](./03-subagents-and-skills.md).

## Defining One

```python
from langchain.tools import tool


@tool
def search_database(query: str, limit: int = 10) -> str:
    """Search the customer database for records matching the query.

    Args:
        query: Search terms to look for
        limit: Maximum number of results to return
    """
    return f"Found {limit} results for '{query}'"
```

Two things are load-bearing here and easy to skip past:

- **Type hints are required.** They *are* the tool's input schema — without them the model has
  nothing to fill in.
- **The docstring becomes the description**, which is what the model reads when deciding whether
  to call this tool at all. It is not internal documentation; it is a prompt.

Then hand it to an agent:

```python
from langchain.agents import create_agent

agent = create_agent(model="gpt-5.5", tools=[search_database])
```

### Name and description

The tool name defaults to the function name, and the description to the docstring. Override
either:

```python
@tool("web_search")
def search(query: str) -> str:
    """Search the web for information."""


@tool("calculator", description="Performs arithmetic calculations. Use this for any math problems.")
def calc(expression: str) -> str:
    ...
```

Prefer `snake_case` names — `web_search`, not `Web Search`. Some providers reject or mishandle
names with spaces and special characters, so alphanumerics, underscores, and hyphens travel best.

### Documenting arguments

`parse_docstring=True` lifts the per-argument descriptions out of the docstring and into the
schema, so the model is told what each argument means rather than inferring it from the name:

```python
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

Note the shape of that docstring. It says **when** to use the tool, not only what it does. A
description that only describes gets called at the wrong moments.

## Tool Descriptions Are Instructions

Everything the model knows about a tool is its name, schema, and description — sent on every
turn, whether or not the tool is used. Two consequences worth planning around:

- Writing a good description is prompt engineering, not documentation. Say when to reach for the
  tool and what each argument means.
- Tools you never remove cost tokens forever. In Deep Agents, unused built-in tools can be
  dropped with a [harness profile](../deepagents/03-harness-profiles.md)'s `excluded_tools`.

## Server-Side Tools

Some providers ship built-in tools — web search, code interpreters — that the provider executes
itself, with no package or API key on your side. You enable them by passing a provider tool dict
rather than a function.

Worth knowing: this saves you *running* the tool, not paying for its output. The invocation and
result come back inside the response message content and then live in the history like anything
else. See [Managing the Context Window](../deepagents/05-context-window.md#server-side-tools-are-not-exempt).

## The Tool List Is Not the Whole Set

`tools=[...]` is the set you *declare*. It is not necessarily the set the model is offered on any
given turn: the real set is assembled at compile time and can be rewritten again before every
model call. Three mechanisms account for the difference, and between them they explain most
"where did that tool come from?" moments.

### Middleware can ship tools

An `AgentMiddleware` subclass can declare a `tools` class attribute, and `create_agent` picks it
up at compile time and merges it in. Once merged, those tools are indistinguishable from the ones
you passed.

This is the normal case, not an exotic one. `TodoListMiddleware` is how an agent acquires
`write_todos`. A Deep Agent gets `ls`, `read_file`, `write_file`, `task` and the rest from
`FilesystemMiddleware` and `SubAgentMiddleware` without your call site naming any of them — which
is precisely why [harness profiles](../deepagents/03-harness-profiles.md) need an
`excluded_tools` field at all.

### `wrap_model_call` can rewrite the set per call

`request.tools` holds the tools for the call about to happen, and `request.override(tools=...)`
hands back a request carrying a different set. Filtering is the common use — expose sensitive
tools only after authentication, drop destructive ones for read-only roles, or cut a long list
down when it is confusing the model:

```python
from langchain.agents.middleware import wrap_model_call


@wrap_model_call
def gated_tools(request, handler):
    if not request.state.get("authenticated", False):
        public = [t for t in request.tools if t.name.startswith("public_")]
        request = request.override(tools=public)
    return handler(request)
```

Adding works too, and it is the path for tools that do not exist when the agent is constructed —
loaded from an MCP server, fetched from a registry, generated per user. Adding takes a second
hook:

```python
from langchain.agents.middleware import AgentMiddleware


class DynamicToolMiddleware(AgentMiddleware):
    def wrap_model_call(self, request, handler):
        return handler(request.override(tools=[*request.tools, calculate_tip]))

    def wrap_tool_call(self, request, handler):
        if request.tool_call["name"] == "calculate_tip":
            return handler(request.override(tool=calculate_tip))
        return handler(request)
```

`wrap_model_call` only changes what the model is *offered*. A tool that was never registered at
compile time has no implementation waiting on the agent's side, so when the model calls it,
`wrap_tool_call` is what supplies one. **Filtering needs the first hook; adding needs both.**

### `response_format` adds one

`ToolStrategy` implements structured output as artificial tool calling — your schema is bound as
a tool, and the model choosing to call it is what produces `result["structured_response"]`.
`ProviderStrategy` uses the provider's native structured output instead, so it does not.

### Seeing the real set

None of this makes `tools=` a reliable inventory. When you need to know what the model actually
received, log it from the innermost position:

```python
@wrap_model_call
def audit_tools(request, handler):
    print([t.name for t in request.tools])
    return handler(request)
```

Middleware nests outermost-first, so put this **last** in the list to see the final set, after
every other middleware has had its turn.

## Intercepting Tool Calls

Every tool call can be wrapped, logged, retried, or blocked with a `wrap_tool_call`
[middleware](./01-middleware.md) — `request.tool_call["name"]` and `["args"]` tell you which call
you are looking at, and declining to call the handler blocks it.

That single hook is also how you guard things that are *implemented* as tools without being
obviously tool-shaped, which is the subject of the next chapter.

---

Previous: [Middleware](./01-middleware.md) |
Next: [Subagents and Skills](./03-subagents-and-skills.md)

Sources: [Tools](https://docs.langchain.com/oss/python/langchain/tools),
[Models](https://docs.langchain.com/oss/python/langchain/models),
[Custom middleware](https://docs.langchain.com/oss/python/langchain/middleware/custom),
[Structured output](https://docs.langchain.com/oss/python/langchain/structured-output).

Back to [LangChain](./README.md) | [Index](../README.md)
