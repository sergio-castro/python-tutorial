# Labs

Short, self-contained, hands-on walkthroughs for specific topics. Each lab gets you to a
running result fast; for the concepts and depth behind them, see the
[main tutorial](../tutorial.md).

## Available labs

1. [Hello World](./hello-world/README.md)
   The smallest possible Python program, set up and run with `uv`. Start here if you've
   never set up a Python project before.

2. [Call Claude with a Prompt](./claude/README.md)
   The simplest LLM program: one direct call to Claude with a prompt and an API key, using
   the official `anthropic` SDK. Uses the standard `ANTHROPIC_API_KEY`.
   - [Custom env variable variant](./claude/custom-env-key.md) — same call, but reads the
     key from your own env var (e.g. `CLAUDE_API_KEY`) explicitly in code.
   - [API Keys & Billing](./claude/api-keys.md) — how Anthropic auth/billing works and how
     to supply your key (applies to all the Claude labs).

3. [Build a Deep Agent](./deepagents/README.md)
   A step up: an LLM agent that can call tools, using `deepagents` + `langchain-anthropic`.
