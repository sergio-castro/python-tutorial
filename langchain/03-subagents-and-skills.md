# Subagents and Skills

In [Deep Agents](../deepagents/README.md) you get subagents and skills by passing `subagents=[...]`
and `skills=[...]`. The natural question is whether plain LangChain has the same thing.

**Yes — but as patterns you implement, not parameters you set.** `create_agent` has no
`subagents=` or `skills=` argument. What it has is `tools=`, and both ideas are built out of that.

That is the whole relationship between the two layers: LangChain gives you the patterns, Deep
Agents packages two of them with the plumbing already written.

## The Five Patterns

LangChain documents five ways to structure work across specialised components:

| Pattern | How it works |
|--------------------|--------------------------------------------------------------|
| **Subagents** | A main agent coordinates subagents *as tools*. All routing goes through the main agent. |
| **Handoffs** | Behaviour changes with state. A tool call updates a state variable that switches agents, tools, or prompt. |
| **Skills** | Specialised prompts and knowledge loaded on demand. One agent stays in control. |
| **Router** | A classification step directs input to specialised agents; results are synthesised. |
| **Custom workflow** | A bespoke flow in [LangGraph](../langgraph/README.md), mixing deterministic and agentic steps. |

And a comparison to choose between them:

| Pattern | Distributed development | Parallelisation | Multi-hop | Direct user interaction |
|---------------|:---:|:---:|:---:|:---:|
| **Subagents** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐ |
| **Handoffs** | – | – | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Skills** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Router** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | – | ⭐⭐⭐ |

The column that most often decides it is the last one. Subagents report back to the main agent
rather than talking to the user, so if a specialist needs to hold a conversation, you want
handoffs or skills instead.

Before reaching for any of them: a single agent with the right tools and prompt often does the
same job. Multi-agent earns its complexity when one agent has so many tools it chooses badly,
when a task needs extensive specialised context, or when you must gate capabilities behind
conditions.

## Subagents: an Agent Wrapped as a Tool

The entire mechanism:

```python
from langchain.agents import create_agent
from langchain.tools import tool

subagent = create_agent(model="google_genai:gemini-3.6-flash", tools=[...])


@tool("research", description="Research a topic and return findings")
def call_research_agent(query: str):
    result = subagent.invoke({"messages": [{"role": "user", "content": query}]})
    return result["messages"][-1].content


main_agent = create_agent(
    model="google_genai:gemini-3.6-flash",
    tools=[call_research_agent],
)
```

An agent is a callable, so wrapping it in a tool makes it something another agent can invoke. The
`@tool` description is what the main agent reads when deciding to delegate — the same job the
`description` field does on a Deep Agents subagent spec.

What you get, and what you do not:

- **Centralised control.** Every routing decision passes through the main agent.
- **Parallel execution.** The main agent can call several subagent-tools in one turn.
- **No direct user interaction.** Subagents return to the caller, not the user — though an
  `interrupt` inside a subagent can still pause to collect input.
- **Context isolation is yours to arrange.** Returning `result["messages"][-1].content` is what
  keeps the subagent's intermediate work out of the parent's context. Return the whole message
  list and you have lost the main benefit.

## Skills: a Tool That Returns a Prompt

Skills here are prompt-driven specialisation with progressive disclosure — the same idea as
[Agent Skills](https://agentskills.io/), implemented by hand:

```python
from langchain.agents import create_agent
from langchain.tools import tool


@tool
def load_skill(skill_name: str) -> str:
    """Load a specialized skill prompt.

    Available skills:
    - write_sql: SQL query writing expert
    - review_legal_doc: Legal document reviewer

    Returns the skill's prompt and context.
    """
    ...   # load from a file, a database, wherever


agent = create_agent(
    model="gpt-5.5",
    tools=[load_skill],
    system_prompt=(
        "You are a helpful assistant. You have access to two skills: "
        "write_sql and review_legal_doc. Use load_skill to access them."
    ),
)
```

Note where the progressive disclosure lives: the **skill names sit in the system prompt and in
the tool's docstring** (cheap, always present), while the **full prompt arrives only when
`load_skill` is called** (expensive, on demand). That is exactly the two-level split Deep Agents
automates with frontmatter and `SKILL.md`.

The pattern extends naturally: because a tool can update state, loading a skill can also register
tools that only make sense once that skill is active — a `database_admin` skill bringing `backup`,
`restore`, and `migrate` with it.

## What Deep Agents Adds

Same two ideas, with the machinery supplied:

| | LangChain | Deep Agents |
|--------------------|--------------------------------|--------------------------------------|
| Subagents | You write the wrapper tool | `subagents=[...]` builds the `task` tool |
| Skill discovery | You list them in a prompt | Frontmatter scanned into the system prompt |
| Skill loading | Your `load_skill` returns text | `read_file` on `SKILL.md`, three-level disclosure |
| Isolation | Whatever your wrapper returns | Subagent gets its own context by construction |
| Shared filesystem | You build it | Backends, with routing and permissions |

So the decision is not *which framework has subagents*. It is whether you want the packaged
version and its conventions, or the handful of lines above and full control over them.

The patterns also compose in both directions: a subagent architecture can call tools that run
routers or custom workflows, and a subagent can itself use the skills pattern.

---

Previous: [Tools](./02-tools.md) | Next: [PII Detection](./04-pii-detection.md)

Sources: [Multi-agent](https://docs.langchain.com/oss/python/langchain/multi-agent/index),
[Subagents](https://docs.langchain.com/oss/python/langchain/multi-agent/subagents),
[Skills](https://docs.langchain.com/oss/python/langchain/multi-agent/skills).

Back to [LangChain](./README.md) | [Index](../README.md)
