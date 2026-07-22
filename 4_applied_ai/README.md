# Applied AI (Weeks 15–22)

RAG → Vector databases → Agent systems → MCP.

## What to Learn Deeply

**RAG** — don't memorize LangChain/LlamaIndex APIs, understand the pipeline:

```
Documents → Chunking → Embedding → Vector Search → Retrieval → Re-ranking → Prompt → LLM → Answer
```

- *What problem does it solve?* The model's knowledge is frozen at training time and its context window is finite — RAG lets it answer from your (current, much larger) documents instead.
- *How does it work conceptually?* Chunk documents small enough to embed meaningfully, embed both chunks and the query into the same vector space, retrieve the nearest chunks, optionally re-rank them for relevance, then put the survivors into the prompt as context.
- *When should I use it?* When answers need to reflect information the model wasn't trained on, or information that changes too often to fine-tune for.

Once you understand that pipeline, any specific framework is just an API for the same eight boxes.

**Agents** — framework syntax (LangChain, LlamaIndex, a custom loop) changes constantly; these concepts don't:
- **Planning** — breaking a goal into steps before acting.
- **Memory** — what the agent carries between steps or sessions (short-term scratchpad vs. long-term store).
- **Tool calling** — how the model requests an external function/API and gets its result back into context.
- **MCP (Model Context Protocol)** — a standard interface for exposing tools/data to a model, instead of a bespoke integration per tool.
- **Reflection** — the agent critiques its own output or plan before finalizing.
- **State** — what the agent tracks about the world/task across a multi-step run.
- **Failures & recovery** — what happens when a tool call errors, a step contradicts an earlier one, or the agent loops: does it retry, escalate, or fail gracefully?

## How to Study This Phase

Build a minimal RAG pipeline over a handful of your own documents by hand first (manual chunking, one embedding call, cosine similarity, no framework) before reaching for a library — a framework's abstractions only make sense once you've felt what they're abstracting away.

## Next

[5_production_ai](../5_production_ai/README.md) — a RAG pipeline or agent that only works in a notebook isn't shippable; that's what production AI covers.
