# LLM Inference — What Happens After You Press Enter

## The Whole File in One Table

Generation is **two** different jobs, not one. Almost every production question is really a question about which one you mean.

| | **Prefill** | **Decode** |
|---|---|---|
| **Why** | the model must read your whole prompt before it can say anything | the answer must be written one token at a time — token N+1 needs token N first |
| **How** | all prompt tokens go through the model together, in parallel | one full pass through the model per output token |
| **Where** | on the GPU, **once**, before the first word appears | on the GPU, **repeated** once per output word |
| **Runs out of** | compute — the GPU's math units | memory bandwidth — moving weights around |
| **Costs you** | **TTFT**: how long the user stares at nothing | **TPOT**: how fast text streams — and most of your bill |

One line to keep: **during decode, the GPU reads every weight in the model just to produce one token.** The math is trivial. The reading is the cost. Nearly every trick in this file moves fewer bytes rather than doing less math.

---

## How To Read This File

```
Part 1   WHERE it happens        the serving map and the GPU memory map
Part 2   HOW generation works    prefill vs decode, TTFT vs TPOT
Part 3   WHAT the KV cache costs the one formula to memorise
Part 4   HOW servers go fast     what vLLM actually does
Part 5   WHAT you should choose  the decisions, cheapest first
```

| Question you will be asked at work | Which part |
|---|---|
| "Why is the first word slow but the rest fast?" | Part 2 |
| "How many users fit on one GPU?" | Part 3 |
| "We're burning $40k/month. Give me five levers." | Part 5 |
| "Can we just paste all 500k tokens in?" | Part 5 |

Practice: [3_kv_cache_and_inference_math.ipynb](code/3_kv_cache_and_inference_math.ipynb).

This is the file that separates *"I use LLM APIs"* from *"I can design an LLM system"*.

---

# Part 1 — WHERE This Actually Happens

## Map A — where your request goes

```
  YOUR APP
     │
     │  HTTP, a few ms
     ▼
  LOAD BALANCER
     │
     ▼
  ┌──────────────────────────────────────────────┐
  │  INFERENCE SERVER   (vLLM / SGLang / TRT)    │
  │                                              │
  │   waiting queue  ──►  running batch          │
  │                          │                   │
  │                          │  continuous       │
  │                          │  batching swaps   │
  │                          │  requests here    │
  │                          ▼                   │
  │                   ┌──────────────┐           │
  │                   │     GPU      │           │
  │                   └──────────────┘           │
  └──────────────────────────────────────────────┘
```

The inference server is not a thin wrapper. It is a **scheduler**. It decides who runs now, who waits, and how memory is shared. Most of Part 4 is about that scheduler, not about the model.

## Map B — where the GPU memory goes

This map answers "how many users fit", which is the question you will actually be asked.

```
  ONE 80 GB GPU SERVING Llama-3-8B

  ┌───────────────────────────────────────────────┐  80 GB
  │  MODEL WEIGHTS              16 GB             │  fixed, loaded once
  ├───────────────────────────────────────────────┤
  │  framework overhead         ~2 GB             │  fixed
  ├───────────────────────────────────────────────┤
  │                                               │
  │  KV CACHE            ~62 GB left over         │  ◄── THIS is what
  │                                               │      limits users
  │    at 128 KB per token:                       │
  │       8k context ×  1 user  =   1 GB          │
  │       8k context × 32 users =  32 GB   ✅ fits │
  │       8k context × 62 users =  62 GB   ⚠️ full │
  │                                               │
  └───────────────────────────────────────────────┘
```

**The weights are fixed. The KV cache is not.** It grows with every token *and* every user at the same time. That is why capacity planning is a KV cache question, not a model-size question.

---

# Part 2 — HOW Generation Works

```
YOUR PROMPT (500 tokens)                GENERATED ANSWER (200 tokens)
┌──────────────────────────┐            ┌───┬───┬───┬───┬───┬─────┐
│      P R E F I L L       │    →       │ t │ t │ t │ t │ t │ ... │
│  all 500 at once, once   │            └───┴───┴───┴───┴───┴─────┘
└──────────────────────────┘                    D E C O D E
                                          one token at a time, 200 times
```

**Prefill** reads all 500 prompt tokens together. The GPU loves this — it is one big parallel job. It happens **once**.

**Decode** writes the answer one token at a time. Token 2 cannot start until token 1 exists. It happens **200 times**, and each time the GPU reads all 16 GB of weights to produce a few bytes of output.

## The two numbers to quote

| Metric | Means | Driven by |
|---|---|---|
| **TTFT** — time to first token | how long until the user sees *anything* | prompt length. Long prompt = slow start. |
| **TPOT** — time per output token | how fast text streams after that | model size and memory speed — **not** prompt length |

These pull in different directions, and different products care about different ones:

| Product | Optimise for | Why |
|---|---|---|
| A chat UI | **TTFT** | the user is watching a blank box |
| Summarising 10,000 docs overnight | **throughput** | nobody is watching; total tokens/sec is all that matters |
| A voice agent | **TPOT** | speech must not stutter mid-sentence |

Saying that distinction out loud in a design interview is a senior signal.

---

# Part 3 — WHAT the KV Cache Costs

To generate token 501, attention needs the Key and Value vectors of tokens 1–500. Those were already computed when generating token 500. Recomputing them every step would make the whole thing `n²` work all over again.

So we keep them. That is the KV cache: **trade memory for compute.**

- **Problem it fixes:** without it, every new token re-reads the whole sequence. Generation gets slower and slower as the answer grows.
- **How:** after each step, append the new token's K and V to a buffer. Reuse forever.
- **The catch:** that buffer lives in expensive GPU memory and grows with **every token × every user**.

## The formula to memorise

```
KV bytes per token = 2 × layers × kv_heads × head_dim × bytes_per_number
                     ↑
                     one for K, one for V
```

**Llama-3-8B** — 32 layers, 8 KV heads, head_dim 128, fp16 (2 bytes):

```
2 × 32 × 8 × 128 × 2  =  131,072 bytes  =  128 KB per token

   8,000 tokens ×  1 user   →   1 GB
   8,000 tokens × 32 users  →  32 GB   ← twice the model weights (16 GB)
```

Now you see the real reason GQA exists. The same model **without** GQA — 32 KV heads instead of 8 — would use **512 KB per token**. Four times more. Those same 32 users would need **128 GB** of cache, which does not fit on any single GPU. Long context was unaffordable before GQA.

**The interview question:** *you serve a 70B model at 100k context, batch 32 — what breaks?* The KV cache breaks. Do the arithmetic out loud.

---

# Part 4 — HOW Servers Go Fast

These all live in the scheduler from Map A. None of them change the model.

| Trick | Problem it fixes | How it works |
|---|---|---|
| **Continuous batching** | in a fixed batch, everyone waits for the slowest request | as soon as one request ends, a new one takes its slot immediately |
| **PagedAttention** | reserving max-length cache per user wastes most of the memory | store the cache in small fixed blocks, like OS memory pages — under 4% waste |
| **Prefix caching** | 20 users send the same long system prompt; you prefill it 20 times | keep the shared prefix's KV cache and reuse it |
| **Chunked prefill** | one huge prompt blocks everyone else's streaming | split the prefill into chunks, interleave with decode steps |
| **Speculative decoding** | decode is memory-bound, so the GPU is mostly idle | a small draft model guesses k tokens ahead; the big model checks them all in one pass. Wrong guesses are thrown away, so **quality is unchanged** |
| **Quantization** | weights are the thing you keep re-reading | store them in fewer bits → fewer bytes to move → faster and cheaper |

## Prefix caching is your free win

Put your long, unchanging system prompt **first** and the user's changing text **last**.

```
✅  [ long fixed system prompt ][ user's question ]     cache hits
❌  [ user's question ][ long fixed system prompt ]     full prefill, every call
```

That is one line of reordering for a real saving. Providers also charge less for cached input tokens.

## Quantization in plain words

Numbers can be stored with different precision. Fewer bits = smaller and faster, with some quality loss.

| Precision | 8B weights | Typical use |
|---|---|---|
| FP16/BF16 (16-bit) | ~16 GB | training, and the quality baseline |
| FP8 (8-bit) | ~8 GB | the common production choice in 2026 — loss usually tiny |
| INT4 (4-bit) | ~4 GB | laptops, one small GPU; measurable quality loss |

Two rules:

1. **Always measure on your own task**, never on a public benchmark. Loss shows up unevenly — a 4-bit model can chat fine and still get worse at structured JSON or long-chain math.
2. Quantizing the **KV cache** is a *separate lever* from quantizing the **weights**. When long context is your problem, the cache is often the better lever.

## Context window ≠ usable context

Nearly every frontier model advertises about 1M tokens in 2026. That number is a ceiling, not a promise.

- **Lost in the middle** — models attend well to the start and the end. The middle gets much weaker treatment.
- **Attention dilution** — a fixed attention budget spread over more tokens means everything gets less.
- **Context rot** — accuracy falls fastest when the distractors *look similar* to the right answer. Which is exactly what a real document set looks like.

Public "needle in a haystack" tests hide this, because finding one odd sentence in a pile of unrelated text is far easier than finding the right one of forty similar paragraphs.

---

# Part 5 — WHAT You Should Choose

### 1. Which metric are you optimising?

Decide this **before** you touch anything. Optimising the wrong one wastes weeks.

| If your product is | Optimise | Ignore |
|---|---|---|
| interactive chat | TTFT | throughput |
| batch/offline processing | throughput | TTFT entirely |
| voice / realtime | TPOT | — |

### 2. Cut cost by 50% — five levers, ranked

Always in this order. Cheapest and safest first.

```
1. Shorten the prompt            free        biggest win, always try first
2. Reorder for prefix caching    free        fixed text first, changing text last
3. Cap max_tokens on output      free        output costs several × input
4. Quantize weights to FP8       low risk    ~half the memory, small quality cost
5. Move to a smaller model       measure     often fine for narrow tasks
```

Only after all five: buy more GPUs.

### 3. Long context or retrieval?

| Situation | Choose |
|---|---|
| The relevant text is small and you can find it | **retrieve** — 5k right tokens beat 500k random ones |
| You genuinely need whole-document reasoning | long context, and measure quality honestly |
| You are unsure | retrieve first. It is cheaper, faster, *and* usually more accurate |

"Just paste everything into the 1M window" is not a design. This is an argument about **quality**, not only cost — that is the case for RAG in [4_applied_ai](../4_applied_ai/README.md).

### 4. Which serving engine?

| Engine | Pick it when |
|---|---|
| **vLLM** | default. Widest model and hardware support, easiest to run |
| **SGLang** | high concurrency, heavy prefix reuse |
| **TensorRT-LLM** | you need maximum NVIDIA throughput and can afford the ops pain |
| **llama.cpp** | laptops and small local machines |

Start with vLLM. Move only when you have measured a reason. You meet these for real in [6_production_ai](../6_production_ai/README.md).

### 5. When to ignore this whole file

If you are under ~10,000 requests a month on an API, none of this pays. Your engineering time costs more than the tokens. Ship first, measure, and come back when the bill is real.

---

## Check Yourself

No notes, no AI. Say it out loud.

1. **Where** does the GPU memory go on a machine serving an 8B model? Name the three parts and which one limits users.
2. What are the two phases of generation, and which resource limits each?
3. Write the KV-cache-per-token formula from memory. Apply it to Llama-3-8B.
4. 32 users at 8k context — how many GB of cache? Is that more or less than the weights?
5. Why does prompt **order** change your bill?
6. Why is a 1M context window not the same as 1M usable tokens?
7. You must cut cost 50%. Name your five levers in order, and say which is free.

---

## Quick Summary

| Term | In one line |
|---|---|
| Where it runs | a scheduler in front of a GPU — the scheduler is most of the magic |
| GPU memory | weights (fixed) + KV cache (grows with tokens × users) |
| Prefill | reads the whole prompt at once; sets TTFT; compute-bound |
| Decode | one token at a time; sets TPOT; memory-bound |
| TTFT / TPOT | time to first token / time per output token |
| KV cache | store past K and V so you don't recompute — memory for compute |
| 128 KB/token | Llama-3-8B. Memorise the formula, not the number |
| GQA | fewer KV heads → 4× smaller cache → long context affordable |
| Continuous batching | free slots refilled instantly |
| PagedAttention | KV cache in small blocks, under 4% waste |
| Prefix caching | fixed text first — a free win |
| Speculative decoding | small model guesses, big model checks in one pass |
| Quantization | fewer bits per number → fewer bytes to move |
| Context rot | long-input quality drops, worst with similar-looking distractors |
| First lever, always | shorten the prompt |

## Next

[4_fine_tuning_and_alignment.md](4_fine_tuning_and_alignment.md) — when prompting is not enough and you change the model's weights instead.

## Sources

- [Inside vLLM: Anatomy of a High-Throughput LLM Inference System](https://vllm.ai/blog/2025-09-05-anatomy-of-vllm)
- [The State of Open-Source LLM Inference Engines in 2026](https://builderai.tools/blog/state-of-open-source-llm-inference-engines-2026)
- [LLM Serving Optimization: Continuous Batching, PagedAttention, and Chunked Prefill on H100 (2026)](https://www.spheron.network/blog/llm-serving-optimization-continuous-batching-paged-attention/)
- [How KV Caching Slashes LLM Inference Costs at Scale](https://www.digitalocean.com/community/conceptual-articles/how-kv-caching-slashes-llm-inference-costs-at-scale)
- [LLM Context Window Management and Long-Context Strategies 2026](https://zylos.ai/research/2026-01-19-llm-context-management/)
