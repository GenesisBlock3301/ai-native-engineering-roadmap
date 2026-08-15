# Modern AI (Weeks 7–14)

Transformers → Tokenization → Hugging Face → LLM inference → Fine-tuning.

This is the phase where an LLM stops being a black box. [1_introduction](../1_introduction/README.md) taught you to *use* a model. [2_ai_foundations](../2_ai_foundations/README.md) taught you the math under it. Here you learn what actually happens between "press enter" and "text appears" — and every production decision after this point rests on that.

---

## How Every Note In This Phase Is Built

All five note files follow the same shape, so you always know where to look:

```
The Whole File in One Table    WHY / HOW / WHERE / what it costs you
How To Read This File          the part map + the work questions it answers

Part 1   WHERE it happens      the system map — which machine does which job
Part 2   HOW it works          the mechanism
Part 3   WHAT it costs you     money, memory, latency — with real numbers
Part 4   (topic-specific)
Part 5   WHAT you should choose   the decisions, cheapest option first

Check Yourself                 questions to answer out loud, no notes
Quick Summary                  one line per idea
Next                           the link onward
```

**Read Part 1 first, always.** Most confusion in this phase comes from not knowing *which machine* is doing *which job* — your CPU or the GPU, online or offline, inside the model or outside it.

**Part 5 is the point.** You are learning to decide, not to code. If you finish a file and cannot say *"in situation X I would pick Y"*, you learned the mechanism and missed the job.

---

## What's Here

**Notes** — read these in order:

| File | Covers | The decision it teaches |
| --- | --- | --- |
| [1_attention_and_transformers.md](1_attention_and_transformers.md) | Q/K/V, √d_k scaling, causal masking, multi-head, RoPE, the n² cost, and the 2026 block: GQA, RMSNorm, SwiGLU, MoE, FlashAttention | when architecture is your problem, and when it isn't |
| [2_tokenization_and_embeddings.md](2_tokenization_and_embeddings.md) | BPE, box size, why token count is your bill, special tokens, token vs document embeddings, cosine similarity | which tokenizer, which embedding model, and when *not* to use embeddings |
| [3_llm_inference.md](3_llm_inference.md) | prefill vs decode, TTFT/TPOT, KV cache math, continuous batching, PagedAttention, prefix caching, speculative decoding, quantization, context rot | which metric to optimise, and five cost levers ranked |
| [4_fine_tuning_and_alignment.md](4_fine_tuning_and_alignment.md) | the three training stages, LoRA/QLoRA, DPO vs RLHF vs GRPO, catastrophic forgetting, data quality | whether to fine-tune at all — usually no |
| [5_hugging_face.md](5_hugging_face.md) | the Hub, transformers v5, `pipeline` vs `AutoModel`, chat templates, safetensors security, model sizing | API or self-host, base or instruct, will it fit |
| [6_practice_and_selfcheck.md](6_practice_and_selfcheck.md) | the week-by-week plan, 38 self-check questions, interview questions, the mini-project | this file *is* the study plan |

**Code** — every notebook is practice for one note, and every one ends with a "Your turn" section:

| Notebook | What you build | Needs |
| --- | --- | --- |
| [code/1_tokenization_and_embeddings.ipynb](code/1_tokenization_and_embeddings.ipynb) | a BPE tokenizer trained by hand, real token-cost comparisons across languages, embeddings + semantic search | `tiktoken`, Gemini API key |
| [code/2_attention_from_scratch.ipynb](code/2_attention_from_scratch.ipynb) | scaled dot-product attention, causal masking, multi-head, the GQA memory table | NumPy only |
| [code/3_kv_cache_and_inference_math.ipynb](code/3_kv_cache_and_inference_math.ipynb) | a tiny decoder generating with and without a KV cache, plus GPU sizing and TTFT/TPOT math | NumPy only |
| [code/4_lora_from_scratch.ipynb](code/4_lora_from_scratch.ipynb) | LoRA adapters, rank experiments, merging, catastrophic forgetting, full vs LoRA vs QLoRA memory | NumPy only |
| [code/5_hugging_face_and_sampling.ipynb](code/5_hugging_face_and_sampling.ipynb) | a real model loaded locally: logits, hand-written temperature/top-k/top-p, KV cache timing, real attention maps, chat templates | `torch`, `transformers` |

### Setup

```bash
pip install -r ../requirements.txt
```

Notebooks 2, 3 and 4 need nothing but NumPy and matplotlib — start there if you want to begin right now. Notebook 5 downloads two small models (a few hundred MB) the first time, and works on a laptop CPU. Notebook 1 uses the same `GEMINI_API_KEY` from `.env` that [1_introduction](../1_introduction/basic_prompt.ipynb) already uses.

---

## What to Learn Deeply

**Attention & transformers**
- **Why attention was invented** — RNNs read one token at a time, so early tokens fade before the model reaches later ones. And the work could not be parallelised.
- **Why attention solves it** — every token looks at every other token directly, in one step, whatever the distance.
- **What it costs** — `n × n` work. Double the input, quadruple the attention cost. This is the number behind every long-context problem.
- **What changed since 2017** — every frontier model in 2026 is still decoder-only, but the standard parts are now RoPE, RMSNorm, SwiGLU, GQA and increasingly MoE. Know *why* each replaced what came before.

**LLMs** — know these deeply, not just by name:
- **Tokenization** — a tokenizer maps text to a fixed vocabulary of integer IDs. Token count is what you're billed for and what fills your context window. It runs on your CPU, outside the model, so counting is free.
- **Embeddings** — each token ID maps to a learned vector. Separately, a whole document can be embedded as one vector — that is what retrieval runs on. These are two different things in two different places.
- **Inference & KV cache** — generating token N+1 needs the K/V of tokens 1..N. Caching them avoids recomputing everything each step, which is why context length costs both memory and speed. Learn the formula well enough to use it in a meeting.
- **Prefill vs decode** — different bottlenecks (compute vs memory bandwidth), different metrics (TTFT vs TPOT). Almost every latency question is really asking which one you mean.
- **Context window** — the hard limit on prompt + history + output. The advertised number is a ceiling, not a promise: quality drops in the middle of long inputs, and drops fastest when distractors look like the answer.
- **Hallucination** — the model predicts a plausible next token, not a looked-up truth. It hallucinates when "plausible" and "true" come apart.
- **Temperature & sampling** — why picking from a distribution instead of always taking the top token is what makes generation non-deterministic. Notebook 5 has you implement the rules by hand.
- **Reasoning** — longer structured intermediate generation that trades inference cost for accuracy. Those thinking tokens are output tokens, and you pay for them.
- **Fine-tuning** — changing weights on task data instead of relying on prompting. LoRA/QLoRA is how it is actually done, and it happens offline, never at request time.
- **Alignment** — training a model to prefer the responses humans actually want (RLHF, DPO, verifiable-reward RL), separate from raw capability.

---

## How to Study This Phase

Follow the [Learning Loop](../README.md#the-learning-loop): understand → picture it → explain it in your own words → read AI-generated code (don't write it first) → change it → break it on purpose and fix it → build something small → teach it to someone else.

Concretely: **read the note file, run its notebook, then do every "Your turn" item.** The week-by-week schedule, the self-check questions and the mini-project are in [6_practice_and_selfcheck.md](6_practice_and_selfcheck.md) — that file is the actual study plan.

One warning specific to this phase: it is very easy to *collect vocabulary* here — PagedAttention, GQA, QLoRA, DPO — and mistake that for understanding. The test is not whether you can name it. The test is whether you can say **what problem it solves, what it costs, and when you would not use it.**

## Worked Example: Learning Attention the 2026 Way

Old way: watch a 3-hour video, copy the code, forget it.

2026 way:
1. Ask: *why was attention invented?*
2. Then: *what problem does an RNN have?*
3. Then: *why does attention solve that?*
4. Ask AI to draw attention visually.
5. Ask: *explain like I'm a backend engineer.*
6. Read the PyTorch / Hugging Face implementation — don't write it, read it.
7. Ask, line by line: *why is this line here?*
8. Repeat until you can explain every line yourself.

Then make it concrete: run [code/2_attention_from_scratch.ipynb](code/2_attention_from_scratch.ipynb), delete the `/ √d_k` and watch softmax collapse, flip the causal mask and see why the model would break. Finish by looking at real attention weights from a trained model in [code/5_hugging_face_and_sampling.ipynb](code/5_hugging_face_and_sampling.ipynb).

## Next

Once you can explain attention, the KV cache formula, and the fine-tune-or-not decision without notes, move to [4_applied_ai](../4_applied_ai/README.md) — RAG and agents are built entirely on top of this.
