# Attention & Transformers

[2_ai_foundations](../2_ai_foundations/README.md) ended with one line: *attention looks at all positions at once*. This file opens that line up.

Read it once for the story. Then run [2_attention_from_scratch.ipynb](code/2_attention_from_scratch.ipynb) and read it again. It lands much better the second time.

---

## The Problem Before Attention

Old models (RNNs, LSTMs) read text like a person reading through a straw — one word at a time, left to right. After each word they updated one small "memory" vector and moved on.

Two things went wrong:

1. **Long-distance memory fades.** By word 500, whatever word 3 said is mostly gone. The model has squeezed 500 words into one small vector.
2. **It can't go fast.** Word 100 needs word 99 first. A GPU has thousands of cores sitting idle, but the work has to happen in order.

**Prompt to feel it:** `The trophy did not fit in the suitcase because it was too big. What does "it" mean?`

To answer, the model must link `it` back to `trophy` — 8 words away. Now imagine that link is 8,000 words away.

---

## The Idea: Let Every Word Look at Every Other Word

Attention throws away the straw. Every token gets to look at every other token **directly**, in one step. Distance stops mattering.

Each token creates three vectors:

| Name | Simple meaning | Human version |
|---|---|---|
| **Query (Q)** | what I am looking for | "I am the word *it*. Who am I pointing at?" |
| **Key (K)** | what I can offer | "I am *trophy*. I am a thing, and I am a noun." |
| **Value (V)** | what I actually pass on | the real content of *trophy* |

The steps:

```
1. score   = Q · Kᵀ          → how well does each query match each key?
2. scale   = score / √d_k    → keep the numbers from getting huge
3. weights = softmax(scale)  → turn scores into percentages that add to 1
4. output  = weights · V     → a mix of all values, weighted by match
```

That's the whole thing. Four lines. Everything else in a transformer is plumbing around these four lines.

**Why divide by √d_k?** With big vectors, the dot products get big. Big numbers into softmax make one weight ≈ 1.0 and the rest ≈ 0.0. The model stops blending and starts picking one word — and the gradients almost vanish, so it learns nothing. Dividing keeps the scores in a sane range. You can watch this break on purpose in the notebook.

---

## Masking: Don't Read the Future

A chat model predicts the next token. During training it sees the whole sentence at once, so you must stop position 5 from peeking at position 6 — otherwise it learns to cheat, and then fails at real inference time when the future does not exist yet.

The fix is a **causal mask**: set every "future" score to −infinity before the softmax, so its weight becomes exactly 0.

```
        the   cat   sat   on
the      ✓     ✗     ✗    ✗
cat      ✓     ✓     ✗    ✗
sat      ✓     ✓     ✓    ✗
on       ✓     ✓     ✓    ✓
```

- **Decoder-only** (GPT, Llama, Claude, Gemini) — masked, generates text. Every frontier chat model today.
- **Encoder-only** (BERT, embedding models) — no mask, sees both sides, does not generate. Still used for embeddings and rerankers, which you meet in [4_applied_ai](../4_applied_ai/README.md).

---

## Multi-Head Attention: Many Small Looks, Not One Big Look

One attention "head" learns one kind of relationship. That's not enough — language has many at once (grammar, subject–verb links, topic, tone).

So the model splits the vector into `h` smaller pieces and runs attention `h` times in parallel, then glues the results back together.

- **Problem it fixes:** one head has to average all relationship types into one blurry answer.
- **How it works:** 12 heads of size 64 instead of 1 head of size 768. Same total compute, many separate views.
- **What to remember:** heads are not assigned jobs by a human. They pick up roles on their own during training.

---

## Where Does Word Order Come From?

Attention as described is a **bag of words** — it has no idea what comes first. `dog bites man` and `man bites dog` would look identical to it.

So position gets added in.

| Method | Used by | The idea |
|---|---|---|
| Sine/cosine (2017 original) | old transformers | add a fixed wave pattern to each token |
| Learned position vectors | GPT-2, BERT | learn one vector per slot; breaks past the trained length |
| **RoPE (rotary)** | Llama, Qwen, Mistral, most models today | **rotate** Q and K by an angle based on position |

**Why RoPE won:** rotation means attention scores end up depending on the *distance between* two tokens, not their absolute slots. That generalizes to longer inputs much better, and you can stretch it (rope scaling) to extend context without retraining from zero.

---

## The Modern Block (2026)

Every frontier model today is still a decoder-only transformer. But "transformer" in 2026 does not mean the 2017 paper. The standard parts changed:

| 2017 | 2026 | Why it changed |
|---|---|---|
| Multi-Head Attention | **GQA** (Grouped-Query Attention) | many query heads share one K/V pair → far smaller KV cache → long context becomes affordable |
| Sine/cosine position | **RoPE** | works better on long inputs |
| LayerNorm | **RMSNorm** | drops the mean-subtraction step: fewer parameters, less compute, same result |
| ReLU/GELU feed-forward | **SwiGLU** | a gated feed-forward that simply trains better |
| Dense feed-forward | **MoE** (Mixture of Experts) | many "expert" networks, only a few run per token |
| Post-norm | **Pre-norm** | normalize before the sub-layer → deep models train stably |

**GQA in one picture** — 32 query heads, but only 8 K/V heads:

```
MHA  : 32 queries → 32 keys/values   (biggest cache, best quality)
GQA  : 32 queries →  8 keys/values   (4× smaller cache, quality almost the same)  ← today's default
MQA  : 32 queries →  1 key/value     (smallest cache, quality drops a bit)
```

This is a real interview question: *why did Llama 3 pick GQA over MQA?* Answer: MQA saves the most memory but loses quality; GQA keeps almost all the quality for most of the savings. It is a trade-off, not a free win.

**MoE in one line:** a 400B-parameter model where only ~20B parameters run per token. You pay memory for all of them, but compute for a few. That is how models got much bigger without inference cost exploding.

---

## FlashAttention: Same Math, Much Faster

The naive attention above builds the full `n × n` score matrix in GPU memory. At 32k tokens that matrix is over a billion numbers — per head, per layer.

FlashAttention computes attention in **tiles** that fit in the GPU's small, fast on-chip memory, and never writes the big matrix to slow memory at all. Same output, far less memory traffic.

**Why you care as an architect:** it is not a new model idea, it is a memory idea. Most "make the model faster" wins in 2026 are memory-movement wins, not math wins. Keep this lens — it explains most of [3_llm_inference.md](3_llm_inference.md).

---

## Quick Summary

| Piece | In one line |
|---|---|
| Attention | every token looks at every token, in one step |
| Q, K, V | what I want / what I offer / what I pass on |
| √d_k scaling | stops softmax from collapsing onto one token |
| Causal mask | you may not read the future |
| Multi-head | many small relationship-detectors instead of one blurry one |
| RoPE | position by rotation → handles long inputs |
| GQA | share K/V across heads → smaller KV cache |
| MoE | many experts, few active per token |
| FlashAttention | same math, tiled, way less memory traffic |

## Next

[2_tokenization_and_embeddings.md](2_tokenization_and_embeddings.md) — before a token can attend to anything, text has to become numbers. That's the step just below this one.

## Sources

- [Transformer Design Guide (Part 2: Modern Architecture)](https://rohitbandaru.github.io/blog/Transformer-Design-Guide-Pt2/)
- [Modern LLM Architecture Explained: what changed after the original transformer](https://medium.com/@shail251298/from-vanilla-transformers-to-modern-llms-what-changed-after-the-original-transformer-part-1-2e74d8531570)
- [Top 32 LLMs & Transformers Interview Questions (2026)](https://www.datainterview.com/blog/llms-and-transformers-interview-questions)
