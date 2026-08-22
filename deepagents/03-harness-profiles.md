# Harness Profiles

A harness profile is configuration you register against a **model**, which Deep Agents applies
automatically whenever that model is selected — without touching your `create_deep_agent` call.

That is the whole idea, and it is also the rule for when to use one:

> Profiles are for adjustments that **depend on which model is selected**. Adjustments that apply
> regardless of model belong on the `create_deep_agent` call site.

If you only ever run one model, you probably do not need profiles at all — pass the arguments
directly. They earn their keep when the same agent code runs against several models that need
different handling, or when you are packaging an integration for other people.

Deep Agents ships built-in harness profiles for OpenAI and Anthropic models, so some of this is
already happening whether or not you register anything.

## Reading the Example

```python
from deepagents import HarnessProfile, register_harness_profile

register_harness_profile(
    "anthropic:claude-sonnet-4-6",
    HarnessProfile(
        excluded_tools=frozenset(
            {"ls", "read_file", "write_file", "edit_file", "delete", "glob", "grep"}
        ),
    ),
)
```

Two arguments: a **key** saying which model this applies to, and a **profile** saying what to
change. Nothing is passed to `create_deep_agent` — registration mutates a global registry, and
resolution happens later, after the chat model is constructed.

The effect here is to hide every filesystem tool, so that on this specific model the agent gets
no `write_file`, no `grep`, and so on. Note what it is *not* doing: it is not removing
`FilesystemMiddleware`. That distinction matters, and is covered under
[Middleware](#excluded_middleware-and-the-stack) below.

## The Fields

`HarnessProfile` has seven fields. Everything a harness profile can do is in this table:

| Field | Type | Does |
|---------------------------|------------------------------|-----------------------------------------------|
| `base_system_prompt` | `str` | Replace the built-in Deep Agents base prompt |
| `system_prompt_suffix` | `str` | Append text at the very end of the assembled prompt |
| `tool_description_overrides` | `Mapping[str, str]` | Rewrite individual tool descriptions, keyed by tool name |
| `excluded_tools` | `frozenset[str]` | Remove tools by name from the tool set |
| `excluded_middleware` | `frozenset[type \| str]` | Strip middleware from the Deep Agents stack |
| `extra_middleware` | `Sequence[AgentMiddleware]` or a callable returning one | Append middleware to every stack this profile applies to |
| `general_purpose_subagent` | `GeneralPurposeSubagentProfile` | Disable, rename, or re-prompt the general-purpose subagent |

```python
from deepagents import (
    GeneralPurposeSubagentProfile,
    HarnessProfile,
    register_harness_profile,
)

register_harness_profile(
    "openai:gpt-5.5",
    HarnessProfile(
        system_prompt_suffix="Respond in under 100 words.",
        excluded_tools={"execute"},
        excluded_middleware={"SummarizationMiddleware"},
        general_purpose_subagent=GeneralPurposeSubagentProfile(enabled=False),
    ),
)
```

### Where the prompt fields land

Your caller-supplied `system_prompt=` always sits at the **front** of the assembled prompt and
`system_prompt_suffix` always sits at the **end** — regardless of which model resolved. In terms
of the assembly order from [Instructing the Agent](./02-instructing-the-agent.md), the profile
can replace item 2 (`base_system_prompt`) and append after everything else
(`system_prompt_suffix`).

The suffix applies to the main agent, to declarative subagents, and to the auto-added
general-purpose subagent. And because **each subagent re-runs profile resolution against its own
model**, a subagent on a different model picks up that model's profile, not the parent's.

### `excluded_tools`

Matched by tool name, applied as a **post-injection filter** — it runs after tools have been
assembled, so it can drop both tools you passed in `tools=` and tools contributed by harness
middleware. That is why the opening example can remove the filesystem tools even though nobody
passed them explicitly.

### `excluded_middleware` and the stack

`create_deep_agent` assembles middleware in a fixed order — skills, filesystem, subagents,
summarization, tool-call patching, async subagents, your own middleware, then profile extras,
excluded-tool filtering, prompt caching, memory, and human-in-the-loop. A profile occupies two designated slots
in that sequence: `extra_middleware` is appended at the profile-extras slot, and `excluded_tools`
is enforced immediately after it.

Entries accept two forms:

- A middleware **class**, matched by exact type, or a plain **string** matching the middleware's
  `.name` — use strings for built-ins and public aliases such as `"SummarizationMiddleware"`.
- A `module:Class` **import ref** like `"my_pkg.middleware:TelemetryMiddleware"`, for targeting an
  exact class from a config file. These resolve lazily, and resolving one imports Python code —
  so only use them with configuration you trust.

**Three middleware cannot be excluded.** Listing `FilesystemMiddleware`, `SubAgentMiddleware`, or
the internal permission middleware raises `ValueError` — they are required scaffolding. To take
their tools away from the model without removing the middleware, use `excluded_tools`. That is
precisely what the opening example does.

Disabling the `task` tool is the other case with a supported path that is not `excluded_middleware`:
set `general_purpose_subagent=GeneralPurposeSubagentProfile(enabled=False)` *and* pass no
synchronous `subagents`. `SubAgentMiddleware` is only attached when at least one synchronous
subagent exists, so it drops out cleanly.

## Registration Keys

The same key format serves both profile types:

| Key | Applies to |
|-------------------------|-------------------------------------------|
| `"openai"` | Every model from that provider |
| `"openai:gpt-5.5"` | Only that specific model |

When both exist they are **merged at resolution time**: unset model-level fields inherit from the
provider-level profile, and explicit model-level values win.

There is **no wildcard key**. To apply something to every model you use — dropping
`SummarizationMiddleware` everywhere, say — register it under each provider key separately. If
that feels wrong, it is the signal that the change is not model-dependent and belongs on the
`create_deep_agent` call site instead.

When you pass a **preconfigured chat model instance** rather than a `provider:model` string, the
harness derives a canonical key from the instance and tries, in order:

1. Exact `provider:identifier`
2. Identifier-only — but only when the identifier already contains `:`
3. Provider-only

## Merge Semantics

Re-registering under an existing key **merges on top of the previous profile**. It does not
replace it. This trips people up: calling `register_harness_profile` twice does not undo the first
call.

| Field | Merge behaviour |
|-------------------------------------|-------------------------------------------------|
| `base_system_prompt`, `system_prompt_suffix` | New value wins when set; otherwise inherits |
| `tool_description_overrides` | Mappings merge per key; new value wins on a shared key |
| `excluded_tools`, `excluded_middleware` | Set **union** — entries can be added but not removed |
| `extra_middleware` | Merged by name: a new instance replaces an existing one at its position; novel entries append |
| `general_purpose_subagent` | Merged field-wise; unset fields inherit |

The set-union rule is the sharp edge. Once a tool is excluded under a key, no later registration
under that key can bring it back.

## Provider Profiles

A narrower, separate API. Where a harness profile changes how the *harness* behaves, a
`ProviderProfile` changes how the *chat model gets constructed*:

```python
from deepagents import ProviderProfile, register_provider_profile

register_provider_profile(
    "openai",
    ProviderProfile(init_kwargs={"temperature": 0}),
)
```

| Field | Type | Does |
|-----------------------|--------------------------------|--------------------------------------------|
| `init_kwargs` | `Mapping[str, Any]` | Static arguments forwarded to `init_chat_model` |
| `pre_init` | `Callable[[str], None]` | Side effects before construction — credential checks, for example |
| `init_kwargs_factory` | `Callable[[], dict[str, Any]]` | Kwargs derived at runtime, such as headers from environment variables |

One significant limitation: a provider profile applies **only when you pass a `provider:model`
string**. Hand `create_deep_agent` a preconfigured model from `init_chat_model` and the profile is
skipped entirely — the model is already built, so there is nothing left to influence.

Most callers do not need these. They are for packaging a provider integration, or for making
`init_chat_model` defaults travel with a provider choice.

## Loading From YAML

`HarnessProfileConfig` mirrors the declarative subset of `HarnessProfile` — prompt text, tool
description overrides, excluded tools and middleware, general-purpose subagent edits — and adds
`to_dict` / `from_dict`. Runtime-only state (middleware instances, factories, class-form
`excluded_middleware`) stays on `HarnessProfile`.

```yaml
# openai.yaml
base_system_prompt: You are helpful.
system_prompt_suffix: Respond briefly.
excluded_tools:
  - execute
  - grep
excluded_middleware:
  - SummarizationMiddleware
  - my_pkg.middleware:TelemetryMiddleware
general_purpose_subagent:
  enabled: false
```

```python
import yaml
from deepagents import HarnessProfileConfig, register_harness_profile

with open("openai.yaml") as f:
    register_harness_profile("openai", HarnessProfileConfig.from_dict(yaml.safe_load(f)))
```

`register_harness_profile` accepts either type, so there is no conversion step.

Going the other way, `HarnessProfileConfig.from_harness_profile(...)` exports a runtime profile
back to the declarative shape — but only when it uses serialisable features. Non-empty
`extra_middleware`, and middleware classes declared in `__main__` or inside a function scope,
cannot be serialised and raise `ValueError`.

## Shipping a Profile as a Plugin

A distributable profile can register itself through `importlib.metadata` entry points, so callers
do not have to call `register_*_profile` by hand:

```toml
[project.entry-points."deepagents.harness_profiles"]
my_provider = "my_pkg.profiles:register_harness"

[project.entry-points."deepagents.provider_profiles"]
my_provider = "my_pkg.profiles:register_provider"
```

```python
def register_harness() -> None:
    register_harness_profile(
        "my_provider",
        HarnessProfile(system_prompt_suffix="Batch independent tool calls in parallel."),
    )
```

Each target is a zero-argument callable, run when `deepagents.profiles` is imported. Load order is
**built-ins first, then entry-point plugins, then direct `register_*_profile` calls in your own
code** — and since all three funnel through the same additive registration, later ones layer on
top of earlier ones under the same key.

## Gotchas

| Symptom | Cause |
|--------------------------------------------|--------------------------------------------------------|
| Registering twice didn't reset anything | Registration merges on top; it never replaces |
| A tool you un-excluded is still missing | `excluded_tools` merges by **union**; exclusions cannot be withdrawn under the same key |
| `ValueError` on `excluded_middleware` | You listed required scaffolding — use `excluded_tools` to hide its tools instead |
| Profile ignored entirely | No key matched: you registered `"anthropic:claude-sonnet-4-6"` but selected a different model, or registered a provider profile and passed a preconfigured model |
| Profile applies to some models, not others | There is no wildcard key — register under every provider you use |
| A subagent behaves differently from the parent | It resolved a profile against **its own** model |
| `from_harness_profile` raises `ValueError` | `extra_middleware` and locally-defined middleware classes are not serialisable |

## Cheat Sheet

```python
from deepagents import (
    GeneralPurposeSubagentProfile, HarnessProfile, ProviderProfile,
    register_harness_profile, register_provider_profile,
)

register_harness_profile(                      # how the HARNESS behaves
    "anthropic",                               # provider key, or "anthropic:claude-sonnet-4-6"
    HarnessProfile(
        base_system_prompt="...",              # replaces the built-in base prompt
        system_prompt_suffix="...",            # always last in the assembled prompt
        tool_description_overrides={"grep": "..."},
        excluded_tools=frozenset({"execute"}), # post-injection filter, by tool name
        excluded_middleware={"SummarizationMiddleware"},
        extra_middleware=[MyMiddleware()],
        general_purpose_subagent=GeneralPurposeSubagentProfile(enabled=False),
    ),
)

register_provider_profile(                     # how the MODEL is constructed
    "anthropic",                               # ignored if you pass a prebuilt model instance
    ProviderProfile(init_kwargs={"temperature": 0}),
)
```

Three things to carry away: **profiles are for model-dependent adjustments — anything else belongs
on the call site**; **registration merges rather than replaces, and exclusions union**; and
**`excluded_tools` hides tools while `excluded_middleware` removes machinery**, which is why the
required middleware must be neutralised with the former.

---

Sources: [Profiles](https://docs.langchain.com/oss/python/deepagents/profiles),
[Customization](https://docs.langchain.com/oss/python/deepagents/customization),
[Subagents](https://docs.langchain.com/oss/python/deepagents/subagents).

Previous: [Instructing the Agent](./02-instructing-the-agent.md) |
Next: [Code Execution](./04-code-execution.md) |
Back to [Deep Agents](./README.md) | [Index](../README.md)
