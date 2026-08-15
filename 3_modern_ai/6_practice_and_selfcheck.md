# Practice & Self-Check

## The Whole File in One Table

Reading this phase teaches you nothing on its own. This file turns it into skill you keep.

| | **Explain it** | **Break it** | **Decide with it** |
|---|---|---|---|
| **Why** | If you cannot say it out loud with no notes, you do not know it. | Reading code teaches you nothing about what happens when it goes wrong. | Your job is choosing, not coding. AI writes the code. |
| **How** | Close everything. Talk to a person or a rubber duck. | Change **one** thing, predict, run, find out where you were wrong. | Answer a design question with numbers you worked out yourself. |
| **Where** | Away from the screen. | In the notebooks in `code/`. | On paper, before the meeting. |
| **Costs you** | 5 minutes and your ego. | 20 minutes per experiment. | Nothing — and it is the highest-value one. |

Writing code from scratch is **not** on that list. Read the code, change the code, explain the code. That is the [2026 learning loop](../README.md#the-learning-loop) applied to this phase.

---

## How To Read This File

```
Part 1   The drill rules          how to practise so it sticks
Part 2   Week by week             weeks 7–14, what to read and run
Part 3   Self-check questions     35 questions, no notes allowed
Part 4   Interview questions      what actually gets asked in 2026
Part 5   The mini-project         the thing you keep for your portfolio
```

---

# Part 1 — The Drill Rules

**Rule 1 — predict first.** Before you run any changed cell, say out loud what will happen. Being wrong is the useful part; that gap is exactly where your model of the system is broken.

**Rule 2 — one change at a time.** Two changes at once and you learn nothing from either.

**Rule 3 — no AI for the explain step.** Use AI freely to ask *why*, to compare, to generate code. But when you close the notebook and explain the idea in your own words, do it alone. That is the part that has to live in your head during an interview or a design meeting.

**Rule 4 — write the number down.** *"The KV cache is big"* is not knowledge. *"128 KB per token, so 32 users at 8k context is 32 GB, twice the model weights"* is knowledge.

**Rule 5 — always end on a decision.** After every topic, finish the sentence: *"so in situation X, I would pick Y, because Z."* If you cannot, you learned the mechanism but not the job.

---

# Part 2 — Week by Week (Weeks 7–14)

| Week | Read | Run | Prove you got it |
|---|---|---|---|
| 7 | [1_attention](1_attention_and_transformers.md) Parts 1–3 | [2_attention_from_scratch.ipynb](code/2_attention_from_scratch.ipynb) steps 1–4 | draw Q/K/V on paper, explain the four lines with no notes |
| 8 | [1_attention](1_attention_and_transformers.md) Parts 4–5 | same notebook, steps 5–6 | explain why GQA exists, with the memory arithmetic |
| 9 | [2_tokenization](2_tokenization_and_embeddings.md) all parts | [1_tokenization_and_embeddings.ipynb](code/1_tokenization_and_embeddings.ipynb) | count the tokens in a real prompt from your own work, cost it per million calls |
| 10 | [5_hugging_face](5_hugging_face.md) | [5_hugging_face_and_sampling.ipynb](code/5_hugging_face_and_sampling.ipynb) | show a colleague what a logit is, live, in the notebook |
| 11 | [3_llm_inference](3_llm_inference.md) Parts 1–3 | [3_kv_cache_and_inference_math.ipynb](code/3_kv_cache_and_inference_math.ipynb) steps 1–5 | explain TTFT vs TPOT and which one *your* product cares about |
| 12 | [3_llm_inference](3_llm_inference.md) Parts 4–5 | same notebook, steps 6–7 | size a GPU for a real workload, out loud, in under 5 minutes |
| 13 | [4_fine_tuning](4_fine_tuning_and_alignment.md) | [4_lora_from_scratch.ipynb](code/4_lora_from_scratch.ipynb) | argue *against* fine-tuning for a case where someone wants it |
| 14 | everything again, fast | every "Your turn" you skipped | the mini-project below, plus the self-check list |

If a week runs long, drop the reading, not the notebook.

---

# Part 3 — Self-Check: Explain These With No Notes

Say each answer out loud. If you stumble, that topic is not learned — go back to the file named beside it.

### Where things happen

These are the questions that catch people who memorised mechanisms without a system picture.

1. Where does the tokenizer run — your machine or the GPU server? Why does that make token counting free? → [2](2_tokenization_and_embeddings.md)
2. Where in a transformer block do tokens exchange information? What happens to a token everywhere else? → [1](1_attention_and_transformers.md)
3. Where does GPU memory go on a machine serving an 8B model? Which part limits how many users fit? → [3](3_llm_inference.md)
4. Where does fine-tuning run — at request time or offline? What artefact comes out, and how big is it? → [4](4_fine_tuning_and_alignment.md)
5. Where does a downloaded model live on your disk, and how many times is it downloaded? → [5](5_hugging_face.md)
6. Draw the two places the word "embedding" shows up in a RAG system. Which one do you own? → [2](2_tokenization_and_embeddings.md)

### Attention & transformers → [1](1_attention_and_transformers.md)

7. Why was attention invented — what exactly did RNNs do badly? Give both problems.
8. What are Q, K, and V, without saying "query", "key", or "value"?
9. Why divide by √d_k? Describe what the softmax output looks like when you don't.
10. Why does a causal mask exist, and what would go wrong without one?
11. Why 12 small heads instead of 1 big one?
12. What does RoPE do differently from adding a position vector, and why does that help on long inputs?
13. Your input goes 4k → 16k tokens. Attention work rises by how much? A number, not a word.
14. Name four things that changed between the 2017 transformer and a 2026 one, and why each changed.
15. You are told "400B total, 20B active per token". How much GPU memory do you budget?

### Tokenization & embeddings → [2](2_tokenization_and_embeddings.md)

16. Why subwords instead of words or letters?
17. Describe BPE training in four sentences.
18. Why can a byte-level BPE tokenizer never produce `<UNK>`?
19. Why could GPT-2 not have used a 200k box in 2019? Answer with a number.
20. Why does the same sentence cost more tokens in Bangla than in English, and what does that do to your bill *and* your context budget?
21. Why can't a model reliably count the letters in a word?
22. Why do "the movie was good" and "the movie was not good" score as very similar, and what does that mean for RAG?
23. Name three query types where you should **not** use embeddings, and what to use instead.

### Inference → [3](3_llm_inference.md)

24. What are the two phases of generation, and which resource limits each?
25. Write the KV-cache-per-token formula from memory and apply it to an 8B model.
26. 32 users at 8k context — how many GB? More or less than the weights?
27. Why does continuous batching beat fixed batching?
28. What does PagedAttention fix, and what does it borrow from operating systems?
29. Why does prompt *order* change your bill?
30. Why is a 1M-token context window not the same as 1M usable tokens?
31. Cut cost 50%. Name five levers in order. Which are free?

### Fine-tuning → [4](4_fine_tuning_and_alignment.md)

32. Give three cases where fine-tuning is right, and three where it is the wrong tool.
33. Explain LoRA to a backend engineer in three sentences. Why does `B` start at zero?
34. Twenty customers want twenty tones. How much GPU memory, and why not 20 × 16 GB?
35. What is QLoRA, and which single constraint does it remove?
36. DPO vs RLHF vs GRPO — one line each, and when you'd pick each.
37. What is catastrophic forgetting, and how do you catch it before your users do?
38. Someone says "let's fine-tune on our docs so it knows our product." What do you say?

---

# Part 4 — Interview Questions (2026 Style)

A typical AI-engineer loop today is roughly 40% RAG/evals/agents, 30% production systems, **20% LLM internals**, 10% behavioural. This phase covers that 20%, and much of the 30%.

**Internals**
- Explain attention to someone non-technical. Then explain it to a systems engineer.
- Why did Llama 3 choose GQA over MQA?
- What is in the KV cache, and how big is it for a 70B model at 100k context, batch 32?
- Your model produces good answers but the first token takes 4 seconds. Where do you look?
- Streaming is fast but the first token is slow — which phase, and what fixes it?

**Design / trade-off**
- We are burning $40k/month on an API. Walk me through whether to self-host.
- Our answers are correct in testing and wrong in production on long documents. What do you check?
- The team wants to fine-tune on our docs so the model "knows our product". What do you say?
- We need 20 different tones for 20 customers. One model or twenty?
- We need to cut cost 50% without hurting quality. Five levers, ranked.
- Our Bangla users cost 5× what our English users cost. Why, and what do you change?

**Debugging**
- After adding a KV cache, the output changed slightly. What kind of bug is that?
- The same prompt gives different answers at temperature 0. Name two possible causes.
- Retrieval returns the exact opposite of what the user asked for. Why?

Answer every one in this shape:

```
what I would measure  →  what the numbers would tell me
                      →  what I would change
                      →  what it costs
```

That structure alone puts you ahead of most candidates.

---

# Part 5 — Mini-Project for This Phase

**Build a "token and cost report" for a real prompt you already use.**

Take one prompt from work or from [1_introduction](../1_introduction/basic_prompt.ipynb). Produce a one-page report answering:

1. How many tokens is it? Input and expected output — counted, not guessed.
2. What does it cost per 1,000 calls at your provider's price? Per 1 million?
3. How much of it is a fixed prefix that prefix caching could reuse? Reorder so the fixed part comes first, and re-measure.
4. If you translated it to Bangla, what happens to the token count and cost? Try it on two different tokenizers.
5. If you self-hosted an 8B model instead: how much GPU memory for weights + KV cache at your real concurrency? At what monthly volume does self-hosting become cheaper?
6. One paragraph: your recommendation, and **the single number that decides it**.

That report *is* the job. It is also the best portfolio artefact from this phase, because it shows judgement rather than knowledge. Keep it for [8_portfolio](../8_portfolio/README.md).

---

## Common Mistakes to Avoid

| Mistake | What it looks like | Fix |
|---|---|---|
| Collecting terms | you can say "PagedAttention" but not what it fixes | for every term, say the problem first |
| Learning mechanism, not decision | you can explain GQA but not whether to use MoE | finish every topic with "so I would pick…" |
| Watching, not running | notebooks read but never modified | every "Your turn" section, no skipping |
| Guessing token counts | "that's about 500 tokens I think" | count them; it takes 10 seconds and it is free |
| Fine-tuning reflex | reaching for training before prompting and retrieval | walk the ladder in [1_introduction](../1_introduction/README.md#thinking-like-an-ai-system-architect-production-framing) |
| Trusting benchmarks | "it scores 1M context, so we can paste everything" | test on *your* data, with realistic distractors |
| Skipping the arithmetic | "the GPU should handle it" | do the KV cache math before the meeting, not during |

---

## You're Done With This Phase When…

- [ ] You can explain attention on a whiteboard, Q/K/V to output, with no notes.
- [ ] You can write the KV-cache formula from memory and size a real deployment with it.
- [ ] You can count the tokens and cost of any prompt in under a minute.
- [ ] You can explain LoRA, and argue *both sides* of a fine-tune-or-not decision.
- [ ] You can name where each step runs — CPU or GPU, online or offline — without looking.
- [ ] You have modified and broken every notebook in `code/` at least once.
- [ ] You have taught one of these topics to another person and answered their follow-up question.

## Next

[4_applied_ai](../4_applied_ai/README.md) — RAG, vector databases, agents and MCP. All of it sits directly on top of this phase: retrieval is cosine similarity at scale, agents are ReAct plus tools, and every latency and cost decision there is the inference math you just learned.
