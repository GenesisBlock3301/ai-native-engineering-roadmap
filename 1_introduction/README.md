# Introduction — Prompting Basics, Prompt Engineering & LLM Settings

What's here: `basic_prompt.ipynb` (elements of a prompt), `2_prompt_engineering.md` (prompting techniques — zero-shot through ReAct), and `1_llm_setting.md` (sampling parameters). This is the prerequisite for every later phase — you can't reason about RAG, agents, or fine-tuning if you can't yet reason about what a prompt does to a model's output in the first place.

## What to Learn Deeply

**Elements of a Prompt** (`basic_prompt.ipynb`)
- *What problem does it solve?* A vague ask gets a vague answer. Structuring a prompt into instruction, context, and input data gets a repeatable, steerable one.
- *How does it work conceptually?* The model has no memory between calls — everything it "knows" for this answer is whatever tokens sit in front of it. Instruction says what to do, context narrows the space of correct answers, input data is the actual subject.
- *When should I use it?* Any time an answer is inconsistent or off-target. The fix is usually adding context or input data, not a longer instruction.

**Prompt Engineering Techniques** (`2_prompt_engineering.md`)
- **Zero-shot / Few-shot** — just ask, or show a few examples first. Few-shot fixes bad formatting that zero-shot can't.
- **Chain-of-Thought (CoT) / Self-consistency** — make the model think step by step instead of jumping straight to an answer, then (for the hard cases) ask it several times and go with the most common answer.
- **ReAct (Reason + Act)** — think, act, check the result, repeat. This is the pattern behind almost every tool-using AI agent today, including this assistant. You'll learn it here as a prompting trick. The real version — actual tools, memory, error handling — is in [4_applied_ai](../4_applied_ai/README.md).
- **Prompt chaining** — split a big task into small steps, and feed each answer into the next step.
- **Role/system prompting & structured output** — tell the model who it is once, and ask for answers in a fixed shape (like JSON) when code needs to read them.

**LLM Sampling Settings** (`1_llm_setting.md`)
- **Temperature / Top_p** — how much of the probability distribution over next-tokens the model may draw from. Low = deterministic and safe, high = varied and creative. They interact, so changing both at once makes behavior hard to predict.
- **Max Length** — a hard ceiling on output tokens, not a request the model can reason about. An answer gets truncated mid-thought if it hits the limit.
- **Stop Sequence** — the API halts generation the instant it emits a given string, and drops that string from the output. See `basic_prompt.ipynb`'s Stop Sequence example. Different SDKs expose this differently — a single string or list via `stop` on OpenAI's Chat Completions API, `stop_sequences` inside `GenerateContentConfig` on Gemini — but the underlying behavior is the same.
- **Frequency / Presence Penalty** — discourage repeating the same tokens vs. discourage staying on the same topic. Easy to conflate; they penalize different things.

## Thinking Like an AI System Architect (Production Framing)

Every trick above is a tool, not a rule. On a real job, you don't ask "which trick is the best?" You ask: **"what's the cheapest trick that's good enough for this job?"** You only move up to a more expensive trick when the cheap one really fails — and you check that with real tests, not a guess.

```
Just ask (zero-shot)
      ↓  (wrong format or tone)
Show examples (few-shot)
      ↓  (needs more than one step of thinking)
Think step by step (Chain-of-Thought, + vote if it really matters)
      ↓  (needs facts the model doesn't know, or facts that change a lot)
Look things up (RAG — see 4_applied_ai)
      ↓  (needs to take actions or use tools)
Think + Act (ReAct / agent — see 4_applied_ai)
      ↓  (prompting is too slow or costs too much at scale)
Fine-tuning (see 3_modern_ai, 5_production_ai)
```

Each step down costs more — more time, more money, more moving parts, more ways to break. Jumping straight to "let's fine-tune" or "let's build an agent" when a few examples would have worked is a common, costly mistake. Decide with real tests on real cases, not a feeling.

The same rule applies to settings like temperature. Use `temperature = 0` when you need the same answer every time (tool calls, pulling data out). Use a higher temperature only when different answers are actually the point (brainstorming, creative writing).

This ladder matches the big-picture loop in the [main README](../README.md#the-mindset-shift): Business Problem → Design AI Architecture → Choose Models → Design Retrieval → Design Evaluation. Prompt engineering is where that loop starts — it's the cheapest, fastest thing to try before you touch retrieval, agents, or the model's own weights.

## Next

Once you can predict how instruction/context/input, the prompting techniques above, and the six sampling parameters each change an output, move to [2_ai_foundations](../2_ai_foundations/README.md) — the math and ML intuition behind why the model responds this way at all.
