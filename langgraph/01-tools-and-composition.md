# Tools, Subagents, and Skills in LangGraph

Short answer: **LangGraph has none of them as features, and that is the point.**

There is no `tools=`, no `subagents=`, no `skills=`. LangGraph is the runtime underneath — nodes,
edges, and state — and each of those three ideas is something you assemble from those parts, or
inherit by embedding a LangChain agent inside a node.

You drop to this layer when you need **deterministic control flow**: a fixed sequence, a loop
with a real exit condition, a branch decided by code rather than by a model. If you just want an
agent that calls tools, you are shopping one layer up.

## Two APIs

LangGraph offers two ways to express the same runtime, and they interoperate.

**Graph API** — declarative: nodes, edges, and shared state. Reach for it when you want the
workflow visualised, explicit shared state across many nodes, conditional branching with several
decision points, or parallel paths that merge later.

**Functional API** — procedural: ordinary control flow in ordinary functions. Reach for it when
you want minimal changes to existing procedural code, standard `if`/`else` and loops,
function-scoped state, or a linear workflow with light branching.

Neither is more powerful; they share a runtime and can be mixed in one application.

## Tools

Tools are a LangChain concept, and they arrive here unchanged. Two ways to use them:

**Bind them to a model inside a node** when you want the graph to own the loop:

```python
from langchain.tools import tool


@tool
def search(query: str) -> str:
    """Search the web."""
    ...


model_with_tools = model.bind_tools([search])


def call_model(state):
    return {"messages": [model_with_tools.invoke(state["messages"])]}
```

**Or embed a whole agent as a node**, when you want an agentic step inside a deterministic flow:

```python
from langchain.agents import create_agent

researcher = create_agent(model="gpt-5.5", tools=[search])

graph = (
    StateGraph(State)
    .add_node("research", researcher)     # a compiled agent is just a node
    .add_edge(START, "research")
    .add_edge("research", END)
    .compile()
)
```

That second form is the one to remember, because it is also the answer to the next section.

## Subagents

A subagent at this layer is **just another graph**. There is no special type, because none is
needed — anything invocable can be a node, and `create_agent` returns a compiled graph.

Two shapes, depending on who decides when it runs:

- **As a node** — *you* decide, in the graph structure. The delegation is deterministic.
- **As a tool** — *the model* decides, using the wrapper from
  [Subagents and Skills](../langchain/03-subagents-and-skills.md). The delegation is agentic.

That distinction is the real reason to be at this layer. "Call the reviewer, then the summariser,
always, in that order" is an edge, not a prompt.

Deep Agents bridges the two directions with `CompiledSubAgent`: you hand it a graph you compiled
here, and it becomes something a deep agent can delegate to through its `task` tool. The one
requirement is that your graph's state includes a **`messages`** key, which is the channel the
parent uses to pass the task in and read the result back.

```python
import operator
from typing_extensions import Annotated, TypedDict

from langgraph.graph import START, END, StateGraph


class ReviewState(TypedDict):
    messages: Annotated[list, operator.add]    # required to interoperate as a subagent


review_graph = (
    StateGraph(ReviewState)
    .add_node("lint", lint)
    .add_node("summarise", summarise)
    .add_edge(START, "lint")
    .add_edge("lint", "summarise")
    .add_edge("summarise", END)
    .compile()
)
```

## Skills

Nothing to inherit and nothing to configure — skills are a prompting pattern, not a runtime
feature. At this layer you would implement one as a node that loads text into state, or as a tool
that returns a prompt, exactly as in
[the LangChain pattern](../langchain/03-subagents-and-skills.md#skills-a-tool-that-returns-a-prompt).

If skills are what you actually want, this is the wrong layer to be building at. Use Deep Agents,
which supplies discovery, progressive disclosure, and a filesystem to keep them in.

## Where Each Layer Stands

| | Tools | Subagents | Skills |
|----------------|-------------------------|--------------------------------|--------------------------|
| **LangGraph** | Bind to a model, or embed an agent as a node | Another graph — a node, or wrapped as a tool | Build it yourself |
| **LangChain** | `tools=[...]` on `create_agent` | Pattern: an agent wrapped as a tool | Pattern: a tool that returns a prompt |
| **Deep Agents** | `tools=[...]`, plus built-ins | `subagents=[...]` → the `task` tool | `skills=[...]`, three-level disclosure |

Read it downward and each row is the same idea with less written for you. Read it upward and each
row is the same idea with less control. Nothing is missing at the bottom; it is simply not
prepackaged.

---

Sources: [Choosing between Graph and Functional APIs](https://docs.langchain.com/oss/python/langgraph/choosing-apis),
[Multi-agent](https://docs.langchain.com/oss/python/langchain/multi-agent/index),
[Subagents](https://docs.langchain.com/oss/python/deepagents/subagents).

Back to [LangGraph](./README.md) | [Index](../README.md)
