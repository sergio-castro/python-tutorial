# PII Detection

`PIIMiddleware` scans message content for sensitive values and rewrites them before they reach
the model, the user, or your logs. One instance handles **one PII type**; you stack instances to
cover several.

```python
from langchain.agents import create_agent
from langchain.agents.middleware import PIIMiddleware

agent = create_agent(
    model="gpt-5.5",
    tools=[...],
    middleware=[
        PIIMiddleware("email", strategy="redact"),
        PIIMiddleware("credit_card", strategy="mask"),
        PIIMiddleware("api_key", detector=r"sk-[a-zA-Z0-9]{32}", strategy="block"),
    ],
)
```

## Detector and Strategy Are Independent

This is the whole mental model, and it answers most questions about custom types before they come
up:

| Half | Question it answers | Produces |
|------------|------------------------|-----------------------------------------------|
| `detector` | *Where is the PII?* | A list of matches — `type`, `value`, `start`, `end` |
| `strategy` | *What do we do with it?* | Rewritten content, or an exception |

The detector never decides how a value is rewritten, and the strategy never decides what counts
as PII. They meet only through the match list. So **a custom detector works with every strategy
automatically** — there is no registration step, and nothing about `redact`, `hash`, or `block`
knows or cares whether `"api_key"` is built in.

Built-in types (`email`, `credit_card`, `ip`, `mac_address`, `url`) are just detectors that ship
with the library. Naming any other `pii_type` without a `detector` raises `ValueError`.

## The Four Strategies

`strategy` defaults to `"redact"`. There are exactly four values — the type is
`Literal["block", "redact", "mask", "hash"]`, and anything else raises `ValueError`.

| Strategy | Effect | Output for a custom `api_key` match |
|----------|--------------------------------------|--------------------------------------|
| `block` | Raises `PIIDetectionError`, aborting the run | — (exception) |
| `redact` | Replaces the value entirely | `[REDACTED_API_KEY]` |
| `mask` | Replaces it partially, keeping a tail | `****3456` |
| `hash` | Replaces it with a stable SHA-256 prefix | `<api_key_hash:82be8a4d>` |

`redact` and `hash` build their replacement from `match["type"]` — uppercased for `redact`,
verbatim for `hash` — so a custom type name flows straight through into the output. `block`
ignores the value entirely. All three are fully type-agnostic.

**`redact` vs `mask` is the "completely or partially" distinction.** `redact` removes the value
and leaves a labelled placeholder; `mask` leaves enough of the original to stay recognisable to a
human. `hash` sits between them: the value is gone, but the same input always produces the same
digest, so you can still correlate occurrences without storing the secret.

## `mask` Is the One Type-Aware Strategy

`mask` is format-aware, and its formats are hard-coded per built-in type:

| Type | Input | Masked |
|---------------|-------------------------|-------------------------|
| `credit_card` | `4532-0151-1283-0366` | `****-****-****-0366` |
| `email` | `bob.smith@example.com` | `bob.smith@****.com` |
| `ip` | `192.168.1.42` | `*.*.*.42` |
| `mac_address` | `00:1B:44:11:3A:B7` | `**:**:**:**:**:B7` |
| `url` | `https://x.com/a?b=1` | `[MASKED_URL]` |

A custom type matches none of those branches and falls through to a generic rule:

- longer than 4 characters → `****` plus the **last 4** characters
- 4 characters or fewer → `****`, with nothing preserved

So `mask` still works with a custom detector, but you get the generic tail rather than anything
shaped like your data. If that matters, there is a trick: **the match's `type` selects the mask
format, and a custom callable detector chooses its own `type`.** Return `"credit_card"` from a
detector registered under a different `pii_type` and you inherit credit-card masking:

```python
def detect_internal_card(content: str) -> list[dict]:
    # ... your own matching and validation ...
    return [{"type": "credit_card", "value": value, "start": start, "end": end}]


PIIMiddleware("internal_card", detector=detect_internal_card, strategy="mask")
# pay 4532-0151-1283-0366 now  ->  pay ****-****-****-0366 now
```

A regex-string detector cannot do this — it always stamps matches with the `pii_type` you passed.

## Writing a Detector

Two forms are supported.

**A regex string.** The middleware compiles it and wraps `finditer`:

```python
PIIMiddleware("api_key", detector=r"sk-[a-zA-Z0-9]{32}", strategy="block")
```

**A callable** taking the content and returning matches, for anything needing validation —
checksums, allowlists, context checks:

```python
import re


def detect_ssn(content: str) -> list[dict]:
    matches = []
    for m in re.finditer(r"\d{3}-\d{2}-\d{4}", content):
        first_three = int(m.group(0)[:3])
        if first_three not in (0, 666) and not (900 <= first_three <= 999):
            matches.append({"text": m.group(0), "start": m.start(), "end": m.end()})
    return matches


PIIMiddleware("ssn", detector=detect_ssn, strategy="hash")
```

Only `start` and `end` are required. The value key may be `value` or `text`, and `type` defaults
to the middleware's `pii_type` — the returned dicts are normalised into `PIIMatch` before any
strategy sees them.

> **A compiled pattern is not a valid detector.** The official docs list `re.compile(...)` as a
> third form, but `resolve_detector` only special-cases `str` and otherwise calls the object.
> `re.Pattern` is not callable, so construction succeeds and the first scan raises
> `TypeError: 're.Pattern' object is not callable`. For flags, use an inline group in the string
> instead: `detector=r"(?i)sk-[a-z0-9]{32}"`.

## Where It Applies

Three independent switches decide which content is scanned:

| Flag | Default | Scans |
|-------------------------|---------|-----------------------------------------------|
| `apply_to_input` | `True` | The last `HumanMessage`, in `before_model` |
| `apply_to_output` | `False` | The last `AIMessage`, in `after_model` |
| `apply_to_tool_results` | `False` | `ToolMessage`s after the last `AIMessage`, in `before_model` |

Two details behind that table are easy to miss. Each hook looks at the **last** message of its
kind, not the whole history — earlier turns are not rescanned, so a middleware added mid-thread
does not retroactively clean what is already in state. And content is coerced with `str(...)`
before matching, so multimodal content blocks are scanned in their stringified form.

With `apply_to_output=True` the middleware also registers a stream transformer, so text deltas,
tool-call arguments, tool outputs, and `values` state snapshots are redacted on the wire too, not
just in final state. That part needs `langchain>=1.3.2`.

## Stacking Rules

Middleware order applies normally — `before_model` hooks run first to last, `after_model` last to
first. Each instance rewrites the message in place, so a later rule scans text an earlier one
already modified. Distinct patterns are independent in practice; overlapping ones are not, and a
`block` rule that sits behind a `redact` rule matching the same span will never fire.

`block` raises `PIIDetectionError` out of the agent invocation rather than returning a message, so
it aborts the run. Catch it at the call site if you want to answer the user instead of failing:

```python
from langchain.agents.middleware import PIIDetectionError

try:
    result = agent.invoke({"messages": [{"role": "user", "content": user_text}]})
except PIIDetectionError as e:
    result = f"Blocked: message contained {len(e.matches)} instance(s) of {e.pii_type}."
```

## Gotchas

| Symptom | Cause |
|-----------------------------------------------|--------------------------------------------------------------|
| `ValueError: Unknown PII type` | Custom `pii_type` with no `detector` |
| `TypeError: 're.Pattern' object is not callable` | A compiled regex passed as `detector` — pass the string |
| Masked output is `****1234`, not your format | `mask` only knows the five built-in formats; everything else gets the generic tail |
| Masked output is just `****` | The matched value is 4 characters or shorter |
| Model output still leaks PII | `apply_to_output` defaults to `False` |
| Tool results still leak PII | `apply_to_tool_results` defaults to `False` |
| PII earlier in the thread survives | Only the last message of each kind is scanned |
| A `block` rule never triggers | An earlier middleware already rewrote the matching span |

---

Previous: [Subagents and Skills](./03-subagents-and-skills.md)

Sources: [Built-in middleware](https://docs.langchain.com/oss/python/langchain/middleware/built-in#pii-detection),
[`PIIMiddleware` API reference](https://reference.langchain.com/python/langchain/agents/middleware/pii/PIIMiddleware).
Strategy outputs and the compiled-pattern behaviour verified against `langchain` 1.3.11.

Back to [LangChain](./README.md) | [Index](../README.md)
