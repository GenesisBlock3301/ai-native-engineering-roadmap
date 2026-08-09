# Practice & Self-Check

Reading this phase teaches you nothing on its own. This file is how you turn it into skill you keep.

Three kinds of practice, in order of value:

1. **Explain it** — out loud, with no notes, to a person or a rubber duck.
2. **Break it** — change one thing in a notebook, predict the result, run it, and find out where you were wrong.
3. **Decide with it** — answer a design question with numbers you worked out yourself.

Writing code from scratch is *not* on that list. Read the code, change the code, explain the code. That is the [2026 learning loop](../README.md#the-learning-loop) applied to this phase.

---

## The Drill Rules

**Rule 1 — predict first.** Before you run any changed cell, say out loud what will happen. Being wrong is the useful part; that gap is exactly where your model of the system is broken.

**Rule 2 — one change at a time.** Two changes at once and you learn nothing from either.

**Rule 3 — no AI for the explain step.** Use AI freely to ask *why*, to compare things, and to generate code. But when you close the notebook and explain the idea in your own words, do it alone. That is the part that has to live in your head during an interview or a design meeting.

**Rule 4 — write the number down.** "The KV cache is big" is not knowledge. "128 KB per token, so 32 users at 8k context is 33 GB, which is twice the model weights" is knowledge.

---

## Week-by-Week (Weeks 7–14)

| Week | Read | Run | Prove you got it |
|---|---|---|---|
| 7 | [1_attention_and_transformers.md](1_attention_and_transformers.md) | [code/2_attention_from_scratch.ipynb](code/2_attention_from_scratch.ipynb) steps 1–4 | draw Q/K/V on paper and explain the four lines with no notes |
| 8 | same, plus the "modern block" table | same notebook, steps 5–6 | explain why GQA exists, with the memory arithmetic |
| 9 | [2_tokenization_and_embeddings.md](2_tokenization_and_embeddings.md) | [code/1_tokenization_and_embeddings.ipynb](code/1_tokenization_and_embeddings.ipynb) | count the tokens in a real prompt from your own work and cost it per million calls |
| 10 | [5_hugging_face.md](5_hugging_face.md) | [code/5_hugging_face_and_sampling.ipynb](code/5_hugging_face_and_sampling.ipynb) | show a colleague what a logit is, live, in the notebook |
| 11 | [3_llm_inference.md](3_llm_inference.md) prefill/decode + KV cache | [code/3_kv_cache_and_inference_math.ipynb](code/3_kv_cache_and_inference_math.ipynb) steps 1–5 | explain TTFT vs TPOT and which one your product cares about |
| 12 | [3_llm_inference.md](3_llm_inference.md) serving tricks + context rot | same notebook, steps 6–7 | size a GPU for a real workload, out loud, in under 5 minutes |
| 13 | [4_fine_tuning_and_alignment.md](4_fine_tuning_and_alignment.md) | [code/4_lora_from_scratch.ipynb](code/4_lora_from_scratch.ipynb) | argue *against* fine-tuning for a case where someone wants it |
| 14 | everything again, fast | every "Your turn" you skipped | the mini-project below, plus the self-check list |

If a week runs long, drop the reading, not the notebook.

---

## Self-Check: Explain These With No Notes

Say each answer out loud. If you stumble, that topic is not learned yet — go back to the file named next to it.

**Attention & transformers**
1. Why was attention invented — what exactly did RNNs do badly? → [1](1_attention_and_transformers.md)
2. What are Q, K, and V, in your own words, without saying "query, key, value"?
3. Why divide by √d_k? What breaks if you don't?
4. Why does a causal mask exist, and what would go wrong without one?
5. Why 12 small heads instead of 1 big one?
6. What does RoPE do differently from adding a position vector, and why does that help on long inputs?
7. Name four things that changed between the 2017 transformer and a 2026 one, and why each changed.

**Tokenization & embeddings**
8. Why subwords instead of words or letters? → [2](2_tokenization_and_embeddings.md)
9. Describe BPE training in four sentences.
10. Why does the same sentence cost more tokens in Bangla than in English, and what does that do to your bill and your context budget?
11. Why can't a model reliably count the letters in a word?
12. What is the difference between a token embedding and a document embedding?
13. Why do "the movie was good" and "the movie was not good" score as very similar, and what does that mean for RAG?

**Inference**
14. What are the two phases of generation, and which bottleneck limits each? → [3](3_llm_inference.md)
15. What does the KV cache store, and what does it cost?
16. Write the KV-cache-per-token formula from memory and apply it to an 8B model.
17. Why does continuous batching beat fixed batching?
18. What does PagedAttention fix, and what does the idea borrow from operating systems?
19. Why does prompt *order* change your bill?
20. Why is a 1M-token context window not the same as 1M usable tokens?

**Fine-tuning**
21. Give three cases where fine-tuning is right, and three where it is the wrong tool. → [4](4_fine_tuning_and_alignment.md)
22. Explain LoRA to a backend engineer in three sentences.
23. Why does `B` start at zero?
24. How do you choose the rank, and what does a too-small rank look like in the loss curve?
25. What is QLoRA, and which single constraint does it remove?
26. DPO vs RLHF vs GRPO — one line each, and when you'd pick each.
27. What is catastrophic forgetting, and how do you catch it before your users do?

---

## Interview Questions (2026 Style)

A typical AI-engineer loop today is roughly 40% RAG/evals/agents, 30% production systems, **20% LLM internals**, 10% behavioural. This phase covers that 20%, and a lot of the 30%. These are asked in the real thing:

**Internals**
- Explain attention to someone non-technical. Then explain it to a systems engineer.
- Why did Llama 3 choose GQA over MQA?
- What is in the KV cache, and how big is it for a 70B model at 100k context, batch 32?
- Your model produces good answers but the first token takes 4 seconds. Where do you look?
- Streaming is fast but the first token is slow — which phase is the problem, and what fixes it?

**Design / trade-off**
- We are burning $40k/month on an API. Walk me through whether to self-host.
- Our answers are correct in testing and wrong in production on long documents. What do you check?
- The team wants to fine-tune on our docs so the model "knows our product". What do you say?
- We need 20 different tones for 20 customers. One model or twenty?
- We need to cut cost by 50% without hurting quality. Give me five levers, ranked.

**Debugging**
- After adding a KV cache, the output changed slightly. What kind of bug is that?
- The same prompt gives different answers at temperature 0. Name two possible causes.
- Retrieval returns the exact opposite of what the user asked for. Why might that be?

For each one, practise answering in this shape: **what I would measure → what the numbers would tell me → what I would change → what it costs.** That structure alone puts you ahead of most candidates.

---

## Mini-Project for This Phase

**Build a "token and cost report" for a real prompt you already use.**

Take one prompt from work or from [1_introduction](../1_introduction/basic_prompt.ipynb). Then produce a one-page report answering:

1. How many tokens is it? Input and expected output, counted — not guessed.
2. What does it cost per 1,000 calls at your provider's price? What about 1 million?
3. How much of it is a fixed prefix that prefix caching could reuse? Reorder it so the fixed part comes first, and re-measure.
4. If you translated it, what happens to the token count and the cost?
5. If you self-hosted an 8B model instead: how much GPU memory for weights + KV cache at your real concurrency? At what monthly volume does self-hosting become cheaper?
6. One paragraph: your recommendation, and the number that decides it.

That report *is* the job. It is also the single best portfolio artefact from this phase — it shows judgement, not just knowledge. Keep it for [8_portfolio](../8_portfolio/README.md).

---

## Common Mistakes to Avoid

| Mistake | What it looks like | Fix |
|---|---|---|
| Collecting terms | you can say "PagedAttention" but not what it fixes | for every term, say the problem first |
| Watching, not running | notebooks read but never modified | every "Your turn" section, no skipping |
| Guessing token counts | "that's about 500 tokens I think" | count them; it takes 10 seconds |
| Fine-tuning reflex | reaching for training before prompting and retrieval | walk the ladder in [1_introduction](../1_introduction/README.md#thinking-like-an-ai-system-architect-production-framing) |
| Trusting benchmarks | "it scores 1M context, so we can paste everything" | test on *your* data, with realistic distractors |
| Skipping the arithmetic | "the GPU should handle it" | do the KV cache math before the meeting, not during |

---

## You're Done With This Phase When…

- [ ] You can explain attention on a whiteboard, from Q/K/V to output, with no notes.
- [ ] You can write the KV-cache formula from memory and size a real deployment with it.
- [ ] You can count the tokens and cost of any prompt in under a minute.
- [ ] You can explain LoRA, and argue *both sides* of a fine-tune-or-not decision.
- [ ] You have modified and broken every notebook in `code/` at least once.
- [ ] You have taught one of these topics to another person and answered their follow-up question.

## Next

[4_applied_ai](../4_applied_ai/README.md) — RAG, vector databases, agents and MCP. All of it sits directly on top of what you just learned: retrieval is cosine similarity at scale, agents are ReAct plus tools, and every latency and cost decision there is the inference math from this phase.
