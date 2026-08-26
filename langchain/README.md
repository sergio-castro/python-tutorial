# LangChain

The bottom layer of the stack — the agent framework: models, tools, and the agent loop, via
`create_agent`. Provider-agnostic.

## Topics

1. [Middleware](./01-middleware.md)
   The extension point for retries, logging, approval gates, and guardrails: the hook model,
   execution order, and shaping what hooks record into traces.

2. [Tools](./02-tools.md)
   Defining tools with `@tool`, why the docstring is a prompt, passing them to
   `create_agent` — and why the set the model is offered is not always the set you passed.

3. [Subagents and Skills](./03-subagents-and-skills.md)
   Both exist here as patterns rather than parameters — an agent wrapped as a tool, and a tool
   that returns a prompt — plus the other multi-agent patterns and how to choose.

---

Back to [Index](../README.md)
