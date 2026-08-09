# LLM Inference — What Happens After You Press Enter

This is the file that separates "I use LLM APIs" from "I can design an LLM system". Almost every production question — *why is it slow? why is it expensive? why did it run out of memory? how many users can one GPU hold?* — is answered here.

Practice for this file: [3_kv_cache_and_inference_math.ipynb](code/3_kv_cache_and_inference_math.ipynb).

---

## Two Phases, Not One

Generation has two very different stages. People who miss this cannot debug latency.

```
YOUR PROMPT (500 tokens)                GENERATED ANSWER (200 tokens)
┌──────────────────────────┐            ┌───┬───┬───┬───┬───┬─────┐
│      P R E F I L L       │    →       │ t │ t │ t │ t │ t │ ... │
│  all 500 at once, once   │            └───┴───┴───┴───┴───┴─────┘
└──────────────────────────┘                    D E C O D E
                                          one token at a time, 200 times
```

| | **Prefill** | **Decode** |
|---|---|---|
| What it does | reads your whole prompt | writes the answer, one token per step |
| Runs | once | once **per output token** |
| Parallel? | yes — all tokens together | no — token N+1 needs token N |
| Bottleneck | **compute** (the GPU's math units) | **memory bandwidth** (moving weights around) |
| The metric it sets | **TTFT** — time to first token | **TPOT** — time per output token |

**The key insight:** during decode, the GPU reads *every weight in the model* out of memory just to produce **one** token. The math is trivial; the reading is the cost. That is why decode is called memory-bound, and it explains nearly every optimisation below — they are almost all about moving fewer bytes, not doing less math.

**Two numbers to quote in an interview:**
- **TTFT** — how long until the user sees anything. Driven by prompt length. Long prompt = slow start.
- **TPOT** — how fast the text streams after that. Driven by model size and memory speed, *not* prompt length.

A chat UI needs low TTFT (feels responsive). A batch job that summarises 10,000 documents overnight does not care about TTFT at all — it cares about **throughput** (total tokens per second across everybody). You tune for different things. Saying that out loud in a design interview is a senior signal.

---

## KV Cache — The Most Important Optimisation in Serving

To generate token 501, attention needs the Key and Value vectors of tokens 1–500. Those were already computed while generating token 500. Recomputing them every step would make generation O(n²) work.

So we keep them. That's the KV cache: **trade memory for compute.**

- **Problem it fixes:** without it, every new token re-reads the whole sequence. Generation gets slower and slower as the answer grows.
- **How it works:** after each step, append the new token's K and V to a buffer, and reuse it forever.
- **The catch:** that buffer lives in expensive GPU memory, and it grows with **every token × every user**.

**The formula worth memorising:**

```
KV bytes per token = 2 × layers × kv_heads × head_dim × bytes_per_number
                     ↑
                     one for K, one for V
```

**Llama-3-8B** (32 layers, 8 KV heads, head_dim 128, fp16 = 2 bytes):

```
2 × 32 × 8 × 128 × 2 = 131,072 bytes = 128 KB per token

  8,000 tokens, 1 user   →  1 GB
  8,000 tokens, 32 users →  32 GB   ← more than the model weights themselves (16 GB)
```

Now you understand the real reason GQA exists. The same model without GQA (32 KV heads instead of 8) would use **512 KB per token** — 4× more — and 32 users at 8k context would need 128 GB of cache. That is why long context was unaffordable before GQA.

**This is the interview question:** *you serve a 70B model at 100k context with batch size 32 — what breaks?* The KV cache breaks. Do the arithmetic out loud.

---

## Serving Tricks (What vLLM and SGLang Actually Do)

| Trick | Problem it fixes | How it works |
|---|---|---|
| **Continuous batching** | in a fixed batch, everyone waits for the slowest request to finish | as soon as one request ends, a new one takes its slot immediately |
| **PagedAttention** | reserving max-length cache per user wastes most of the memory | store the cache in small fixed blocks, like OS memory pages — under 4% waste |
| **Prefix caching / RadixAttention** | 20 users send the same long system prompt; you prefill it 20 times | keep the shared prefix's KV cache and reuse it |
| **Chunked prefill** | one huge prompt blocks everyone else's streaming | split the prefill into chunks and interleave with decode steps |
| **Speculative decoding** | decode is memory-bound, so the GPU is mostly idle | a small draft model guesses k tokens ahead, the big model checks them all in one pass; wrong guesses are thrown away, so output quality is unchanged |
| **Quantization** | weights are the thing you keep re-reading | store them in fewer bits (FP8, INT8, INT4) → fewer bytes to move → faster and cheaper |

**Prefix caching deserves your attention as an architect.** Put your long, unchanging system prompt *first* and the user's changing text *last*. Then the cache hits. Reverse that order and you pay full prefill every single call. It is a free win that costs one line of prompt reordering. Providers charge less for cached input tokens too.

**Which engine?** In 2026 the mature open-source choices are **vLLM** (widest model and hardware support — the safe default), **SGLang** (strong on high-concurrency and heavy prefix reuse), **TensorRT-LLM** (highest raw throughput on NVIDIA, hardest to operate), and **llama.cpp** (laptops and small local machines). You meet these again for real in [6_production_ai](../6_production_ai/README.md).

---

## Quantization in Plain Words

Numbers can be stored with different precision. Fewer bits = smaller and faster, with some quality loss.

| Precision | 8B model weights | Typical use |
|---|---|---|
| FP16/BF16 (16-bit) | ~16 GB | training, and the quality baseline |
| FP8 (8-bit) | ~8 GB | the common production choice in 2026 — quality loss is usually tiny |
| INT4 (4-bit) | ~4 GB | laptops, single small GPU; measurable quality loss |

Two rules:
1. **Always measure on your own task**, not on a public benchmark. Loss shows up unevenly — a 4-bit model can chat fine and still get worse at structured JSON or long-chain math.
2. Quantizing the **KV cache** (FP8) is a separate lever from quantizing the **weights**. When long context is your problem, that is often the better lever.

---

## Context Window ≠ Usable Context

The context window is the hard limit on prompt + history + output. In 2026 nearly every frontier model advertises about 1M tokens. That number is a ceiling, not a promise.

What actually goes wrong as inputs get long:

- **Lost in the middle** — models attend well to the beginning and the end of the input. Material in the middle gets much weaker treatment.
- **Attention dilution** — a fixed attention budget spread over more tokens means everything gets less of it.
- **Context rot** — accuracy falls fastest when the distractors look *similar* to the right answer, which is exactly what a real document set looks like.

Public "needle in a haystack" tests hide this, because finding one odd sentence in a pile of unrelated text is far easier than finding the right one of forty similar-looking paragraphs.

**The architect's takeaway:** "just paste everything into the 1M window" is not a design. Retrieving the right 5k tokens usually beats dumping 500k — it is more accurate, much cheaper, and far faster. That is the argument for RAG in [4_applied_ai](../4_applied_ai/README.md), and it is an argument about quality, not just cost.

---

## A Cost Model You Can Do In Your Head

```
cost  ≈ (input tokens × input price) + (output tokens × output price)
```

Output tokens usually cost several times more than input tokens, because each one needs its own full pass over the model. So:

- Long prompt, short answer → cheap-ish, but slow to start (TTFT).
- Short prompt, long answer → expensive and slow to finish (TPOT × many tokens).
- Asking for "think step by step" everywhere → you just multiplied your output tokens. Use it where it pays, as [1_introduction](../1_introduction/README.md#thinking-like-an-ai-system-architect-production-framing) says.

---

## Quick Summary

| Term | In one line |
|---|---|
| Prefill | reads the whole prompt at once; sets TTFT; compute-bound |
| Decode | one token at a time; sets TPOT; memory-bound |
| TTFT / TPOT | time to first token / time per output token |
| KV cache | store past K and V so you don't recompute; memory for compute |
| GQA | fewer KV heads → a much smaller KV cache |
| Continuous batching | free slots get refilled instantly |
| PagedAttention | KV cache in small blocks, almost no waste |
| Prefix caching | reuse the KV cache of a shared prompt prefix |
| Speculative decoding | small model guesses, big model checks in one pass |
| Quantization | fewer bits per number → fewer bytes to move |
| Context rot | long input quality drops, worst with similar-looking distractors |

## Next

[4_fine_tuning_and_alignment.md](4_fine_tuning_and_alignment.md) — when prompting is not enough and you change the model's weights instead.

## Sources

- [Inside vLLM: Anatomy of a High-Throughput LLM Inference System](https://vllm.ai/blog/2025-09-05-anatomy-of-vllm)
- [The State of Open-Source LLM Inference Engines in 2026](https://builderai.tools/blog/state-of-open-source-llm-inference-engines-2026)
- [LLM Serving Optimization: Continuous Batching, PagedAttention, and Chunked Prefill on H100 (2026)](https://www.spheron.network/blog/llm-serving-optimization-continuous-batching-paged-attention/)
- [How KV Caching Slashes LLM Inference Costs at Scale](https://www.digitalocean.com/community/conceptual-articles/how-kv-caching-slashes-llm-inference-costs-at-scale)
- [LLM Context Window Management and Long-Context Strategies 2026](https://zylos.ai/research/2026-01-19-llm-context-management/)
