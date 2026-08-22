# Deep Agents

Deep Agents is the top layer of the LangChain stack — an agent harness built on LangChain and
LangGraph that ships with planning, a filesystem, subagents, and memory already wired up.

## Topics

1. [Backends, State, and Memory](./01-backends-and-memory.md)
   Where an agent's files actually go: `StateBackend` vs `StoreBackend` vs local disk,
   thread-scoped vs cross-thread, and volatile vs durable.

2. [Instructing the Agent](./02-instructing-the-agent.md)
   The five channels for telling an agent what to do — system prompt, memory, skills, tool
   descriptions, and subagents — and when each one reaches the model.

3. [Harness Profiles](./03-harness-profiles.md)
   Per-model configuration applied automatically when a model is selected: hiding tools,
   adjusting prompts, swapping middleware, and how registration merges.

4. [Code Execution](./04-code-execution.md)
   The two ways an agent runs code — sandbox backends (`execute`) and interpreters (`eval`) —
   what each reaches, and what isolation does and does not buy you.

---

Back to [Index](../README.md)
