# Managing the Context Window

Every chapter so far has been about putting things *into* the model's context: prompts, memory,
skills, tool descriptions, files the agent reads. This one is about what happens when there is
too much of it.

Three mechanisms run **by default in every `create_deep_agent` call**. You do not add middleware
to get them, and you will hit them whether or not you know they exist:

| Mechanism | What it does | Why |
|--------------------|--------------------------------------------------|--------------------|
| **Offloading** | Moves large tool inputs and results to the filesystem, leaving a reference | Keep the context small |
| **Summarization** | Replaces old messages with an LLM-written summary | Keep the context small |
| **Prompt caching** | Marks stable prompt sections as reusable | Keep it fast and cheap |

The first two shrink what is sent. The third does not shrink anything — it makes the unchanged
part cheaper to send again. A fourth mechanism, delegating to a
[subagent](./02-instructing-the-agent.md), avoids the problem instead of managing it, by giving
the heavy work its own context window.

## Offloading

Offloading triggers on **size of a single tool call**: when tool call inputs or results exceed
**20,000 tokens** (the default threshold). There are two cases, and they behave differently.

**Large tool results.** The result is written to the configured backend, and what stays in the
conversation is a file path plus a preview of the first 10 lines. The agent can re-read or `grep`
the full content whenever it actually needs it.

**Large tool inputs.** Writing or editing a big file leaves a tool call in the history containing
the entire file content — which is redundant, since the content is already on the filesystem. So
as the session crosses **85% of the model's context window**, older tool calls are truncated and
replaced with a pointer to the file.

Note the difference: results are offloaded as soon as they are big, inputs are only truncated once
the window starts filling up.

This is the mechanism behind a detail from [Backends, State, and Memory](./01-backends-and-memory.md):
offloaded content goes to the **default backend**, under `/large_tool_results/`. If you make a
persistent store or your real disk the composite default, this machinery quietly writes there
forever. Keep `StateBackend` as the default.

One limitation worth knowing: compression does not resize images or lower their resolution. Large
multimodal inputs are not handled by any of this.

## Summarization

When the context is still too large and **nothing is left that offloading can help with**, the
agent summarizes its own history. Two things happen at once:

- **In-context summary** — an LLM writes a structured summary covering session intent, artifacts
  created, and next steps, and that replaces the full history in working memory.
- **Filesystem preservation** — a text rendering of the original messages is written to the
  filesystem as a canonical record.

So the agent keeps knowing *what it is doing* from the summary, and can recover *exact details*
by searching the file. Summarization is lossy in context but not lossy on disk.

### When it triggers

| | Default |
|-----------------------------|----------------------------------------------|
| Trigger | 85% of the model's `max_input_tokens` |
| Recent context kept | 10% of tokens |
| If the model profile is unavailable | Trigger at 170,000 tokens, keep 6 messages |

A **model profile** here is LangChain's metadata about a model — including its context window —
which is how the harness knows what 85% means. When it cannot look one up, it falls back to the
fixed numbers above.

There is also a safety net: if any model call raises `ContextOverflowError`, the agent immediately
summarizes and retries with the summary plus recent preserved messages, rather than failing.

### What it does not touch

Summarization compresses the **message history**, and only that. The system prompt is reassembled
on every turn from its parts, so everything in it arrives intact no matter how many times the
conversation has been compacted:

| Survives compaction untouched | At risk of being summarized away |
|-------------------------------------|----------------------------------------|
| Your `system_prompt` | Earlier user and assistant messages |
| Memory files (`AGENTS.md`) | Tool calls and their results |
| Skill names and descriptions | The body of an activated skill |
| Tool schemas and descriptions | Anything else in the history |

The right-hand column is one list, not several: the harness has no notion of "important" messages
to protect. Whatever falls outside the recent window is handed to the summarizer together.

### Changing the thresholds

Pass your own `SummarizationMiddleware`; an instance whose name matches a built-in **replaces** it
in the stack rather than adding a second one:

```python
from deepagents import create_deep_agent
from deepagents.backends import StateBackend
from deepagents.middleware import SummarizationMiddleware

model = "anthropic:claude-sonnet-4-6"

agent = create_deep_agent(
    model=model,
    middleware=[
        SummarizationMiddleware(
            model=model,
            backend=StateBackend(),
            trigger=("tokens", 100_000),   # summarize past 100k tokens
            keep=("messages", 20),         # keep the last 20 messages verbatim
        ),
    ],
)
```

`trigger` also accepts `("fraction", ...)` for a share of the context window, and a list of
thresholds is combined with **OR** — whichever fires first wins.

### Compacting on demand

Automatic summarization waits for a threshold. To let the agent compact deliberately — between
tasks, say, rather than mid-task — add `create_summarization_tool_middleware`, which gives it a
`compact_conversation` tool it can call itself.

### It shows up in your stream

The summarizing model call emits tokens like any other, so they appear in a streamed run. Filter
them out by metadata:

```python
for chunk in agent.stream({"messages": [...]}, stream_mode="messages", version="v2"):
    token, metadata = chunk["data"]
    if metadata.get("lc_source") == "summarization":
        continue
```

Without this, users see a summary being written in the middle of their conversation.

Do not confuse the two things at play. The **summary text** persists: it replaces the history and
is resent on every subsequent request from then on — that is precisely what it is for. The
**stream tokens** appear exactly once, while the summary is being written. Filtering them is a
display concern, not a cost one; the cost is the summary itself, which is still far smaller than
the history it replaced.

## Prompt Caching

Prompt caching is about **cost and latency, not size**. The static head of your prompt — base
agent instructions, memory, skill content — is identical on every turn, so it can be marked
cache-eligible and not reprocessed each time.

It is **on by default with no configuration**, for Anthropic models and Bedrock models (Claude or
Nova). Both caching middlewares are always registered, and each no-ops on models it does not
support, so nothing breaks when you switch providers — you simply stop getting the benefit. Other
providers have their own caching middleware available separately.

The default cache lifetime is **5 minutes**. For an agent with long gaps between turns, extend it:

```python
from langchain_anthropic.middleware import AnthropicPromptCachingMiddleware

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    middleware=[AnthropicPromptCachingMiddleware(ttl="1h")],   # replaces the default 5m
)
```

### Why ordering matters here

Caching works on a **prefix**: everything up to the cache marker must be byte-identical to the
previous call, or the cache misses. That constrains where the caching middleware can sit, and the
harness places it deliberately:

- Caching runs **after** your middleware and after tool-call patching, so the cached prefix
  matches what is actually sent to the model.
- `MemoryMiddleware` runs **after** caching, so that updating an injected memory file is less
  likely to invalidate the cached prefix.

The practical lesson: anything that rewrites the head of the prompt on every turn destroys the
cache. Content that changes constantly belongs later in the prompt, or in a file the agent reads
rather than in memory that is always loaded.

## How They Fit Together

Ordered from cheapest to most destructive:

1. **Shrink what is there before anything runs.** Unused built-in tools still send their full
   schemas every turn. Dropping them with a [harness profile](./03-harness-profiles.md)'s
   `excluded_tools` reduces the baseline for the whole run and costs nothing.
2. **Offload** — moves bulk out of context but keeps it addressable. Nothing is lost; the agent
   can read it back.
3. **Summarize** — only once offloading has nothing left to give. Detail is lost from context,
   though the transcript survives on disk.
4. **Isolate** — hand the work to a subagent so the intermediate mess never enters this context
   window at all. The one that prevents the problem rather than reacting to it.

Prompt caching sits alongside all of these, reducing the cost of whatever survives.

## Defaults at a Glance

| Setting | Default | Change it with |
|-----------------------------|-------------------------------|------------------------------------|
| Offload threshold | 20,000 tokens per tool call | — |
| Tool-input truncation | At 85% of the context window | — |
| Result preview kept | First 10 lines | — |
| Offload destination | The default backend, `/large_tool_results/` | The composite `default=` backend |
| Summarization trigger | 85% of `max_input_tokens` | `SummarizationMiddleware(trigger=...)` |
| Recent context kept | 10% of tokens | `SummarizationMiddleware(keep=...)` |
| Fallback with no model profile | 170,000 tokens / 6 messages | as above |
| Prompt caching | On for Anthropic and Bedrock | — |
| Cache TTL | 5 minutes | `AnthropicPromptCachingMiddleware(ttl=...)` |

## Gotchas

| Symptom | Cause |
|---------------------------------------------|--------------------------------------------------------|
| `/large_tool_results/` appearing in your repo or store | Offloading writes to the **default** backend; make it `StateBackend` |
| A summary appears mid-conversation in your UI | Summarization tokens stream like any others — filter on `lc_source == "summarization"` |
| The agent "forgot" something it was told | History was summarized; the detail is on disk, and the agent can search for it |
| Cache never seems to hit | Something rewrites the head of the prompt each turn, or more than 5 minutes passed between turns |
| Two summarization middlewares run | Your instance's name did not match the built-in, so it was added instead of replacing it |
| Compression does nothing for huge images | Images are not resized or re-encoded by any of this |
| Thresholds behave oddly on an unknown model | No model profile was found — the 170,000 / 6-message fallback is in effect |

## Cheat Sheet

```python
agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    middleware=[
        SummarizationMiddleware(                      # replaces the built-in instance
            model=model, backend=StateBackend(),
            trigger=("tokens", 100_000),              # or ("fraction", 0.7); a list means OR
            keep=("messages", 20),
        ),
        AnthropicPromptCachingMiddleware(ttl="1h"),   # default is 5m
        create_summarization_tool_middleware(),       # adds `compact_conversation`
    ],
)
```

Three things to carry away: **all of this is already running whether you configured it or not**;
**offloading is lossless and summarization is not, so the order they fire in matters**; and
**caching is a prefix, so anything that rewrites the top of your prompt every turn throws it
away**.

---

Sources: [Context engineering](https://docs.langchain.com/oss/python/deepagents/context-engineering),
[Customization](https://docs.langchain.com/oss/python/deepagents/customization),
[Overview](https://docs.langchain.com/oss/python/deepagents/overview).

Previous: [Code Execution](./04-code-execution.md) |
Back to [Deep Agents](./README.md) | [Index](../README.md)
