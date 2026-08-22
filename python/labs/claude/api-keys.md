# API Keys & Billing (Anthropic)

Companion note for the Claude labs — [Call Claude](./README.md) and
[deepagents](../deepagents/README.md). It explains how Anthropic authentication and billing
work, and — for this machine specifically — why the API key is stored under `CLAUDE_API_KEY`
instead of the usual `ANTHROPIC_API_KEY`.

## Can I skip the API key and just log in?

**No.** This is the misconception worth nailing down:

- A web/OAuth **login never sets `ANTHROPIC_API_KEY`** (or any env var). The browser only
  performs the authorization handshake; the resulting token is stored **locally** — in the
  macOS Keychain, or `~/.claude/.credentials.json` on Linux — for the Claude apps / Claude
  Code to use. It is not an API key, it is short-lived and rotates, and `langchain-anthropic`
  won't read it.
- Your own programs authenticate with an **API key**, not a login session. There's no
  logged-in session for the SDK to borrow.
- The **subscription** (Pro / Max) behind a login powers the Claude *tools*; it is **not** a
  general-purpose API credential. Programmatic API calls are billed per-token against an
  API key, independent of any subscription.

So the split is inherent: **login/subscription for the tools, API key for your code.** A
program like these labs will always need an API key.

> There *is* a separate OAuth mechanism for the official Anthropic SDK (`ant auth login`,
> which stores a profile a bare `Anthropic()` client can use without an env var). But it's a
> different login from a Claude Code subscription, it still isn't a Pro/Max plan powering API
> calls, and `langchain-anthropic` expects an explicit API key rather than picking up that
> profile. Not worth the complexity — use the API key.

## Two ways to pay Anthropic (the billing axis is the *credential*, not the app)

| Credential | How you pay | How you authenticate |
|---|---|---|
| **Subscription** (Claude Pro / Max) | Flat monthly fee; usage bounded by plan limits; no per-token charge | Web / OAuth login |
| **API key** (pay-as-you-go) | Metered per token, billed to Console payment method; no monthly fee | `ANTHROPIC_API_KEY` (an `sk-ant-...` key) |

## Which tool uses which credential

- **Claude desktop app / claude.ai (web)** → always the **subscription**. No API key involved.
- **Your own programs** (the labs / any SDK code) → always an **API key** (pay-per-token). Cannot use a subscription login.
- **Claude Code (CLI)** → **either**: API key if `ANTHROPIC_API_KEY` is set, otherwise the web-login subscription. This is the only tool that straddles both — hence the collision below.

## Where credentials come from

- **API key** (`sk-ant-...`): Anthropic Console → Settings → API Keys →
  <https://console.anthropic.com/settings/keys>. This is the value that goes in an env var.
- **Web / OAuth login**: run `/login` inside Claude Code (opens a browser to authenticate the
  account at <https://console.anthropic.com> or claude.ai). Stored as a token, **not** an env
  var. Web login does *not* produce an `ANTHROPIC_API_KEY`.
- Check the current Claude Code session's auth/plan with `/status`.

## Why this machine uses `CLAUDE_API_KEY`

The env var name `ANTHROPIC_API_KEY` is checked by **both** Claude Code and the SDKs. So if
you set it for your own programs **and** also use Claude Code with a web/OAuth login, Claude
Code detects the env var and uses the **API key (pay-as-you-go billing)** instead of your
web-login subscription. That is the collision:

- If `ANTHROPIC_API_KEY` is set, **Claude Code uses it** → metered pay-per-token API billing.
- If it is **not** set, Claude Code falls back to the **web/OAuth login** → flat-fee subscription.

To keep Claude Code on the subscription while still having an API key available for programs,
the programmatic key is stored under **`CLAUDE_API_KEY`** (a nonstandard name Claude Code
ignores). Downside: libraries default to `ANTHROPIC_API_KEY`, so programs must read
`CLAUDE_API_KEY` explicitly (below). Confirm which credential Claude Code is currently using
with `/status`.

⚠️ Do **not** `export ANTHROPIC_API_KEY="$CLAUDE_API_KEY"` in the shell profile — that
re-introduces the exact collision this rename avoids.

## How a program consumes `CLAUDE_API_KEY`

Read it explicitly and pass it in — no new env var, no `export`.

With the official SDK (the [Call Claude](./README.md) lab):

```python
import os
import anthropic

client = anthropic.Anthropic(api_key=os.environ["CLAUDE_API_KEY"])
```

With `langchain-anthropic` / deepagents (the [deepagents](../deepagents/README.md) lab), pass
it to the model object instead — note the object form drops the `anthropic:` prefix:

```python
import os
from langchain_anthropic import ChatAnthropic

model = ChatAnthropic(
    model="claude-sonnet-4-6",
    api_key=os.environ["CLAUDE_API_KEY"],
)
```

Run from a shell where `CLAUDE_API_KEY` is set; `uv run` inherits the environment.

## Current state of this machine (as observed)

- `ANTHROPIC_API_KEY`: not set → Claude Code runs on the **web-login subscription**.
- `CLAUDE_API_KEY`: set → used by my own programs (**pay-per-token** API billing).
