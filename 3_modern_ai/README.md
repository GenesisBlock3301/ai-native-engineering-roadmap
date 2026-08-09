# Modern AI (Weeks 7–14)

Transformers → Tokenization → Hugging Face → LLM inference → Fine-tuning.

This is the phase where an LLM stops being a black box. [1_introduction](../1_introduction/README.md) taught you to *use* a model; [2_ai_foundations](../2_ai_foundations/README.md) taught you the math under it. Here you learn what actually happens between "press enter" and "text appears" — and that is the knowledge every production decision after this point rests on.

## What's Here

**Notes** — read these in order:

| File | Covers |
| --- | --- |
| [1_attention_and_transformers.md](1_attention_and_transformers.md) | Q/K/V, scaling, causal masking, multi-head, RoPE, and the 2026 block: GQA, RMSNorm, SwiGLU, MoE, FlashAttention |
| [2_tokenization_and_embeddings.md](2_tokenization_and_embeddings.md) | BPE, why token count is your bill, special tokens, token vs document embeddings, cosine similarity |
| [3_llm_inference.md](3_llm_inference.md) | prefill vs decode, TTFT/TPOT, KV cache math, continuous batching, PagedAttention, prefix caching, speculative decoding, quantization, context rot |
| [4_fine_tuning_and_alignment.md](4_fine_tuning_and_alignment.md) | when *not* to fine-tune, SFT, LoRA/QLoRA, DPO vs RLHF vs GRPO, catastrophic forgetting, data quality |
| [5_hugging_face.md](5_hugging_face.md) | the Hub, transformers v5, `pipeline` vs `AutoModel`, chat templates, safetensors, API vs self-hosting |
| [6_practice_and_selfcheck.md](6_practice_and_selfcheck.md) | the week-by-week plan, 27 self-check questions, interview questions, and the mini-project |

**Code** — every notebook is hands-on practice for one note file, and every one ends with a "Your turn" section:

| Notebook | What you build | Needs |
| --- | --- | --- |
| [code/1_tokenization_and_embeddings.ipynb](code/1_tokenization_and_embeddings.ipynb) | a BPE tokenizer trained by hand, real token-cost comparisons across languages, embeddings + semantic search | `tiktoken`, Gemini API key |
| [code/2_attention_from_scratch.ipynb](code/2_attention_from_scratch.ipynb) | scaled dot-product attention, causal masking, multi-head, and the GQA memory table | NumPy only |
| [code/3_kv_cache_and_inference_math.ipynb](code/3_kv_cache_and_inference_math.ipynb) | a tiny decoder that generates with and without a KV cache, plus GPU sizing and TTFT/TPOT math | NumPy only |
| [code/4_lora_from_scratch.ipynb](code/4_lora_from_scratch.ipynb) | LoRA adapters, rank experiments, merging, catastrophic forgetting, full vs LoRA vs QLoRA memory | NumPy only |
| [code/5_hugging_face_and_sampling.ipynb](code/5_hugging_face_and_sampling.ipynb) | a real model loaded locally: logits, hand-written temperature/top-k/top-p, KV cache timing, real attention maps, chat templates | `torch`, `transformers` |

### Setup

```bash
pip install -r ../requirements.txt
```

Notebooks 2, 3 and 4 need nothing but NumPy and matplotlib — start there if you want to begin immediately. Notebook 5 downloads two small models (a few hundred MB) the first time you run it, and works on a laptop CPU. Notebook 1 uses the same `GEMINI_API_KEY` from `.env` that [1_introduction](../1_introduction/basic_prompt.ipynb) already uses.

## What to Learn Deeply

**Attention & Transformers**
- **Why attention was invented** — RNNs process tokens one at a time, so information from early tokens fades before the model reaches later ones (the long-range dependency problem).
- **Why attention solves that** — every token can look at every other token directly, in one step, regardless of distance.
- **Why attention scales** — it's a handful of matrix multiplies (Q·Kᵀ, softmax, ·V), which GPUs parallelize extremely well, unlike an RNN's inherently sequential recurrence.
- **What changed since 2017** — every frontier model in 2026 is still a decoder-only transformer, but the standard parts are now RoPE, RMSNorm, SwiGLU, GQA and (increasingly) MoE. Know *why* each one replaced what came before; it's a common senior interview question.

**LLMs** — know these deeply, not just by name:
- **Tokenization** — text isn't numbers; a tokenizer maps text to a fixed vocabulary of integer IDs the model can embed. Token count is what you're billed for and what fills your context window.
- **Embeddings** — each token ID maps to a learned vector; nearby vectors mean related meaning. Separately, a whole document can be embedded as one vector — that's what retrieval runs on.
- **Inference & KV Cache** — generating token N+1 needs the key/value projections of tokens 1..N. Caching them avoids recomputing the whole sequence at every step, which is why context length affects both memory and speed. Learn the memory formula well enough to use it in a meeting.
- **Prefill vs decode** — the two phases have different bottlenecks (compute vs memory bandwidth) and set two different metrics (TTFT vs TPOT). Almost every latency question is really a question about which of these you mean.
- **Context Window** — the hard limit on how many tokens (prompt + history + output) the model can attend to at once. The advertised number is a ceiling, not a promise: quality drops in the middle of long inputs, and drops fastest when distractors look like the answer.
- **Hallucination** — the model always predicts a plausible next token, not a looked-up truth. It hallucinates when "plausible" and "true" diverge, especially outside its training distribution.
- **Temperature & Sampling** — see [1_introduction](../1_introduction/README.md) for the hands-on version; here, understand *why* sampling from a distribution (instead of always taking the top token) is what makes generation non-deterministic. Notebook 5 has you implement the rules yourself.
- **Reasoning** — longer, structured intermediate generation (chain-of-thought and similar) that trades inference cost for accuracy on multi-step problems. Those thinking tokens are output tokens, and you pay for them.
- **Fine-tuning** — adjusting a pretrained model's weights on task-specific data instead of relying on prompting alone, for when in-context examples aren't enough or a long prompt costs too much. LoRA/QLoRA is how it's actually done.
- **Alignment** — training a model to prefer the responses humans actually want (RLHF, DPO, and verifiable-reward RL), separate from raw next-token-prediction capability.

## How to Study This Phase

Follow the [Learning Loop](../README.md#the-learning-loop): understand → picture it in your head → explain it in your own words → read AI-generated code (don't write it first) → change it → break it on purpose and fix it → build something small → teach it to someone else.

Concretely, for this phase: **read the note file, run its notebook, then do every "Your turn" item.** The week-by-week schedule, the self-check questions and the mini-project are in [6_practice_and_selfcheck.md](6_practice_and_selfcheck.md) — that file is the actual study plan.

One warning specific to this phase: it is very easy to *collect vocabulary* here — PagedAttention, GQA, QLoRA, DPO — and mistake that for understanding. The test is not whether you can name it. The test is whether you can say what problem it solves, what it costs, and when you would not use it.

## Worked Example: Learning Attention the 2026 Way

Old way: watch a 3-hour video, copy the code, forget it.

2026 way:
1. Ask: *why was attention invented?*
2. Then: *what problem does RNN have?*
3. Then: *why does attention solve that?*
4. Ask AI to draw attention visually.
5. Ask: *explain like I'm a backend engineer.*
6. Read the PyTorch / Hugging Face implementation — don't write it, read it.
7. Ask, line by line: *why is this line here?*
8. Repeat until you can explain every line yourself.

Then make it concrete: run [code/2_attention_from_scratch.ipynb](code/2_attention_from_scratch.ipynb), delete the `/ √d_k` and watch softmax collapse, flip the causal mask and see why the model would break. Finish by looking at real attention weights from a trained model in [code/5_hugging_face_and_sampling.ipynb](code/5_hugging_face_and_sampling.ipynb).

## Next

Once you can explain attention, the KV cache formula, and the fine-tune-or-not decision without notes, move to [4_applied_ai](../4_applied_ai/README.md) — RAG and agents are built entirely on top of this.
