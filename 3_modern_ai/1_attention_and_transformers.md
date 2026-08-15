# Attention & Transformers

## The Whole File in One Table

| | **Attention** | **The Transformer Block** |
|---|---|---|
| **Why** | Old models read one word at a time. By word 500, word 3 was forgotten. And a GPU sat idle waiting. | One attention layer finds one kind of link. Language has many, so you stack blocks. |
| **How** | Every token scores every other token, then mixes them by score. Four lines of math. | attention → feed-forward → repeat 32 times. |
| **Where** | **Inside** the model, on GPU, in every block. The **only** place tokens talk to each other. | The whole middle of the model. This is your GPU bill. |
| **Costs you** | `n × n` work. Double the input, quadruple the cost. | ~80% of the weights sit in feed-forward, but ~100% of your long-context pain comes from attention. |

One line to keep: **attention is the only step where tokens look at each other. Everywhere else, every token is processed alone.**

---

## How To Read This File

```
Part 1   WHERE attention sits       the map — start here
Part 2   WHY it was invented        what RNNs did badly
Part 3   HOW it works               Q, K, V, and four lines of math
Part 4   WHAT it costs you          the n² problem, and GQA
Part 5   WHAT you should choose     the decisions that are actually yours
```

| Question you will be asked at work | Which part |
|---|---|
| "Why can't we just use a longer prompt? It's only 4× longer." | Part 4 |
| "Why did Llama 3 pick GQA over MQA?" | Part 4 |
| "Should we use an MoE model to save money?" | Part 5 |
| "Our model is slow at 32k context. What do we change?" | Part 5 |

Read once for the story. Then run [2_attention_from_scratch.ipynb](code/2_attention_from_scratch.ipynb) and read again. It lands much better the second time.

---

# Part 1 — WHERE Attention Sits

Before the math, know which box it lives in. Llama-3-8B stacks this block **32 times**:

```
   ONE TRANSFORMER BLOCK  (× 32)

   tokens in — one vector per token
        │
        ▼
   ┌──────────────────┐
   │  RMSNorm         │   keep the numbers a sane size
   └──────────────────┘
        │
        ▼
   ┌──────────────────┐
   │   ATTENTION      │  ◄── tokens TALK to each other HERE
   │   (Q, K, V)      │      the only mixing step in the whole model
   └──────────────────┘      ~20% of the weights
        │
        ▼  + shortcut
   ┌──────────────────┐
   │  RMSNorm         │
   └──────────────────┘
        │
        ▼
   ┌──────────────────┐
   │  FEED-FORWARD    │  ◄── each token thinks ALONE
   │  (SwiGLU)        │      no mixing at all
   └──────────────────┘      ~80% of the weights
        │
        ▼  + shortcut
   tokens out — same shape, better numbers
```

Three things this map tells you:

1. **Attention is the only place tokens see each other.** The feed-forward step handles each token completely alone. If you want to know how information moves *between* words, look only at attention.
2. **Most weights are not in attention.** Feed-forward holds about 80%. So "the model is big" is mostly a feed-forward fact.
3. **But attention is where long context hurts.** Feed-forward cost grows *straight* with length. Attention cost grows with **length × length**. Part 4 does that number.

Where this sits in the bigger picture: [2_tokenization_and_embeddings.md](2_tokenization_and_embeddings.md) Map A showed a box labelled *"transformer layers — all the real thinking"*. This file opens that box.

---

# Part 2 — WHY Attention Was Invented

Old models (RNNs, LSTMs) read text like a person looking through a straw — one word at a time, left to right. After each word they updated one small memory vector and moved on.

Two things went wrong.

**Problem 1 — long-distance memory fades.** By word 500, whatever word 3 said is mostly gone. You squeezed 500 words into one small vector.

Feel it with this sentence:

```
"The trophy did not fit in the suitcase because it was too big."

                          What does "it" mean?
```

To answer, the model must link `it` back to `trophy` — 8 words away. Now imagine that link is 8,000 words away. The straw cannot do it.

**Problem 2 — it cannot go fast.** Word 100 needs word 99 finished first. A GPU has thousands of cores sitting idle, but the work must happen in order. You bought a parallel machine and gave it a serial job.

**The fix:** throw away the straw. Let every token look at every other token **directly, in one step**. Distance stops mattering, and every token can be processed at the same time.

---

# Part 3 — HOW Attention Works

Each token makes three vectors. The names are bad; the meanings are simple:

| Name | Plain meaning | The word *it* would say |
|---|---|---|
| **Query (Q)** | what I am looking for | "I am *it*. Who am I pointing at?" |
| **Key (K)** | what I can offer | "I am *trophy*. I am a thing, and a noun." |
| **Value (V)** | what I actually hand over | the real content of *trophy* |

Think of a room. Every word shouts what it needs (Q). Every word wears a badge saying what it is (K). Matching pairs shake hands, and the match strength decides how much content (V) flows across.

The whole computation is four lines:

```
1. score   = Q · Kᵀ           how well does each query match each key?
2. scale   = score / √d_k     stop the numbers from getting huge
3. weights = softmax(scale)   turn scores into percentages adding to 1
4. output  = weights · V      a blend of all values, weighted by match
```

That is it. Everything else in a transformer is plumbing around those four lines.

## Why divide by √d_k?

This is the step people skip, and it is the one that actually breaks.

With big vectors, dot products get big. Big numbers into softmax make **one** weight ≈ 1.0 and all the rest ≈ 0.0. The model stops blending and starts picking a single word. Worse, the gradients nearly vanish, so it learns almost nothing.

```
scores [2, 4, 6]         → softmax → [0.02, 0.12, 0.87]   healthy blend
scores [20, 40, 60]      → softmax → [0.00, 0.00, 1.00]   collapsed
```

Dividing by `√d_k` keeps scores in the sane range. Delete that line in the notebook and watch it collapse — that is the fastest way to believe it.

## Masking — you may not read the future

A chat model predicts the *next* token. In training it sees the whole sentence at once, so you must stop position 5 from peeking at position 6. Otherwise it learns to cheat, then fails at real generation time when the future does not exist yet.

The fix is a **causal mask**: set every future score to −infinity before the softmax, so its weight becomes exactly 0.

```
        the   cat   sat   on
the      ✓     ✗     ✗    ✗
cat      ✓     ✓     ✗    ✗
sat      ✓     ✓     ✓    ✗
on       ✓     ✓     ✓    ✓
```

- **Decoder-only** (GPT, Llama, Claude, Gemini) — masked, generates text. Every frontier chat model today.
- **Encoder-only** (BERT, embedding models) — no mask, sees both sides, does not generate. Still used for the embedding models and rerankers in [4_applied_ai](../4_applied_ai/README.md).

## Multi-head — many small looks, not one big look

One attention head learns one kind of relationship. Language has many at once: grammar, subject–verb links, topic, tone.

So split the vector into `h` smaller pieces and run attention `h` times side by side, then glue the results back.

```
1 head  × 768 wide     →  one blurry average of every relationship type
12 heads × 64 wide     →  twelve separate views, same total compute
```

Nobody assigns jobs to heads. They pick up roles on their own during training.

## Where does word order come from?

Attention as described is a **bag of words**. It has no idea what came first. `dog bites man` and `man bites dog` look identical to it.

So position gets added in:

| Method | Used by | The idea |
|---|---|---|
| Sine/cosine (2017) | old transformers | add a fixed wave pattern to each token |
| Learned position vectors | GPT-2, BERT | one learned vector per slot — breaks past the trained length |
| **RoPE (rotary)** | Llama, Qwen, Mistral, nearly everything today | **rotate** Q and K by an angle based on position |

**Why RoPE won:** rotation makes the attention score depend on the *distance between* two tokens, not their absolute slot numbers. That generalises to longer inputs far better, and you can stretch it (rope scaling) to extend context without retraining from zero.

---

# Part 4 — WHAT It Costs You

Attention's bill is not like the rest of the model.

Every token scores every other token. So with `n` tokens you compute `n × n` scores. **Double the input and you quadruple the work.**

```
   1,000 tokens →       1,000,000 scores
   2,000 tokens →       4,000,000 scores      (2× input, 4× work)
  32,000 tokens →   1,024,000,000 scores      per head, per layer
```

At 32k tokens that score matrix is over a billion numbers — **per head, per layer**. Llama-3-8B has 32 layers and 32 heads.

This is why "just make the prompt longer" is not free, and why everything in [3_llm_inference.md](3_llm_inference.md) is about not paying it.

## FlashAttention — same math, much less memory traffic

Naive attention builds that whole `n × n` matrix in GPU memory. FlashAttention computes it in **tiles** that fit in the GPU's small fast on-chip memory, and never writes the big matrix to slow memory at all.

Same output. Far fewer bytes moved.

**Keep this lens:** it is not a new model idea, it is a *memory* idea. Most "make the model faster" wins in 2026 are memory-movement wins, not math wins.

## GQA — sharing keys and values

During generation the model stores the K and V of every past token (the KV cache — see [3_llm_inference.md](3_llm_inference.md)). More heads storing K/V means a bigger cache.

```
MHA  : 32 query heads → 32 key/value heads   biggest cache, best quality
GQA  : 32 query heads →  8 key/value heads   4× smaller cache, quality almost same  ← today's default
MQA  : 32 query heads →  1 key/value head    smallest cache, quality drops
```

**The real interview question:** *why did Llama 3 pick GQA over MQA?* MQA saves the most memory but loses quality. GQA keeps almost all the quality for most of the saving. It is a trade-off, not a free win.

## The 2026 block vs the 2017 paper

Every frontier model is still decoder-only. But the standard parts all changed:

| 2017 | 2026 | Why it changed |
|---|---|---|
| Multi-Head Attention | **GQA** | far smaller KV cache → long context becomes affordable |
| Sine/cosine position | **RoPE** | works better on long inputs, and can be stretched |
| LayerNorm | **RMSNorm** | drops the mean-subtraction step: fewer parameters, less compute, same result |
| ReLU/GELU feed-forward | **SwiGLU** | a gated feed-forward that simply trains better |
| Dense feed-forward | **MoE** | many expert networks, only a few run per token |
| Post-norm | **Pre-norm** | normalise *before* the sub-layer → deep models train stably |

**MoE in one line:** a 400B-parameter model where only ~20B run per token. You pay **memory** for all of them and **compute** for a few. That is how models got much bigger without inference cost exploding.

---

# Part 5 — WHAT You Should Choose

Most of this file is not your decision — you do not design attention. But four real choices come out of it.

### 1. Do you need to care about architecture at all?

| Your situation | Care about GQA/MoE/RoPE? |
|---|---|
| Calling an API | **No.** You cannot change it. Pick on price, quality, context length. |
| Self-hosting on your own GPUs | **Yes.** It decides how many users fit in memory. |
| Choosing between two open models | **Yes, a little.** Check KV heads and context method on the model card. |

Do not bring architecture trivia to an API-only decision. It is the wrong conversation.

### 2. If self-hosting — MHA, GQA, or MQA?

**Default: GQA.** It is what Llama 3, Qwen and Mistral ship, and the reason is the trade above. Only look at MQA if memory is so tight that a measured quality drop is worth it. You will rarely choose this yourself — you inherit it with the model.

### 3. Is an MoE model right for you?

| Situation | MoE? | Why |
|---|---|---|
| High volume, GPUs with lots of RAM | ✅ yes | you pay compute for ~5% of the weights |
| Small GPU, or a laptop | ❌ no | you must hold **all** experts in memory, even unused ones |
| Low, spiky traffic | ❌ probably not | you pay for the memory whether requests come or not |

The trap: reading "400B model, 20B active" and budgeting for 20B. **Budget for 400B of memory.**

### 4. Your model is slow at long context. What do you change?

Walk it in this order — cheapest first:

```
1. Shorten the input          free.       Do you really need 32k?
2. Reorder for prefix caching free.       Fixed text first, changing text last.
3. Retrieve instead of paste  cheap.      5k right tokens beat 500k random ones.
4. Turn on FlashAttention     free-ish.   Usually already on in vLLM.
5. Quantize the KV cache      some risk.  FP8 cache halves the memory.
6. Buy a bigger GPU           expensive.  Last.
```

Notice steps 1–3 do not touch the model at all. Because attention cost is `n × n`, **cutting the input in half cuts attention work by 4×.** That is the biggest lever you own, and it is free.

---

## Check Yourself

No notes, no AI. Say it out loud.

1. **Where** in the block do tokens exchange information? What happens to a token everywhere else?
2. Why did RNNs fail at long distance — and separately, why were they slow?
3. Explain Q, K, V without using the words "query", "key", or "value".
4. Why divide by √d_k? Describe what the softmax output looks like when you don't.
5. Your input goes from 4k to 16k tokens. Attention work goes up by how much? Answer with a number.
6. Why did Llama 3 pick GQA over MQA?
7. You are given "400B total, 20B active per token". How much GPU memory do you budget?

---

## Quick Summary

| Piece | In one line |
|---|---|
| Where attention sits | the only step where tokens look at each other |
| Feed-forward | ~80% of the weights, but each token processed alone |
| Attention | every token looks at every token, in one step |
| Q, K, V | what I want / what I offer / what I hand over |
| √d_k scaling | stops softmax collapsing onto one token |
| Causal mask | you may not read the future |
| Multi-head | many small relationship-detectors, not one blurry one |
| RoPE | position by rotation → handles long inputs, can be stretched |
| The n² cost | double the input, quadruple the attention work |
| FlashAttention | same math, tiled, far fewer bytes moved |
| GQA | share K/V across heads → smaller KV cache → today's default |
| MoE | many experts, few active — pay memory for all, compute for a few |
| Biggest lever you own | shorten the input; halving it cuts attention 4× |

## Next

[2_tokenization_and_embeddings.md](2_tokenization_and_embeddings.md) — before a token can attend to anything, text has to become numbers. That is the step just below this one.

## Sources

- [Transformer Design Guide (Part 2: Modern Architecture)](https://rohitbandaru.github.io/blog/Transformer-Design-Guide-Pt2/)
- [Modern LLM Architecture Explained: what changed after the original transformer](https://medium.com/@shail251298/from-vanilla-transformers-to-modern-llms-what-changed-after-the-original-transformer-part-1-2e74d8531570)
- [Top 32 LLMs & Transformers Interview Questions (2026)](https://www.datainterview.com/blog/llms-and-transformers-interview-questions)
