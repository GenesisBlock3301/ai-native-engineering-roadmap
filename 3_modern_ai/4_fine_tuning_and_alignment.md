# Fine-Tuning & Alignment

Fine-tuning means changing the model's weights on your own data, instead of only changing your prompt.

It is the last rung of the ladder in [1_introduction](../1_introduction/README.md#thinking-like-an-ai-system-architect-production-framing). Most teams reach for it far too early, and most of them regret it. This file is mostly about **when not to**, and then about doing it properly when the answer really is yes.

Practice for this file: [4_lora_from_scratch.ipynb](code/4_lora_from_scratch.ipynb).

---

## How a Model Gets Made

Three stages. Knowing which stage does what tells you which one you actually need.

```
1. PRE-TRAINING        predict the next token over trillions of tokens of the internet
                       → the model learns language, facts, code, reasoning ability
                       → costs millions of dollars. You will never do this.
                            ↓
2. SFT                 supervised fine-tuning on (instruction → good answer) pairs
   (instruction        → the model learns to be a helpful assistant instead of an autocomplete
    tuning)            → you CAN do this, with a few hundred to a few thousand examples
                            ↓
3. ALIGNMENT           train on human preferences: "answer A is better than answer B"
   (RLHF / DPO)        → the model learns tone, safety, refusals, and what people actually want
```

Stage 1 gives **capability**. Stages 2 and 3 give **behaviour**. That distinction is the whole point:

> **Fine-tuning is very good at teaching behaviour. It is bad and expensive at teaching facts.**

Facts that change belong in retrieval ([4_applied_ai](../4_applied_ai/README.md)), not in weights. Weights are a bad database: to update one fact you retrain, and you cannot tell where an answer came from.

---

## First: Should You Fine-Tune At All?

Go down this list in order. Stop at the first "yes".

| Question | If yes → |
|---|---|
| Is the prompt just unclear or under-specified? | fix the prompt (free) |
| Does it get the format wrong? | few-shot examples, or forced structured output (cheap) |
| Does it need facts it doesn't have, or facts that change? | **RAG** — fine-tuning cannot fix this |
| Does it need to take actions? | tools / agents |
| Does it need a very specific style or format on *every* call, and the prompt for that is long and costly? | **fine-tune** |
| Is the task closed and machine-checkable (extraction, classification, tool-call accuracy, math, code)? | **fine-tune** — often a small model beats a big prompted one |
| Do you need a small, cheap, fast model to match a big model on one narrow job? | **fine-tune** (distillation) |

**The trap:** "the model doesn't know our internal docs, so let's fine-tune it on them." That mostly does not work. The model picks up the *sound* of your docs and still invents the details. Use RAG. If you need both, do RAG first, and fine-tune later only for style and format.

**Before you start, you need one thing more than data: an eval set.** Fifty to two hundred real cases with a known-good answer, plus a way to score them. Without it you cannot tell whether fine-tuning helped, and "it feels better" is not an engineering claim. Evaluation is covered in [6_production_ai](../6_production_ai/README.md), but you need a rough version of it *before* your first training run.

---

## LoRA: Fine-Tune Without Touching Most of the Model

Full fine-tuning updates every weight. For an 8B model that means storing weights, gradients, and optimizer state — roughly 12–16× the model size in GPU memory. Out of reach on one GPU, and you get a whole new 16 GB model file per task.

**LoRA (Low-Rank Adaptation)** does something clever instead:

```
Frozen original weight W  (say 4096 × 4096 = 16.7M numbers)

     output = W·x  +  B·A·x
               ↑        ↑
            frozen   trainable, tiny

A is 4096 × r      B is r × 4096       with r = 8
→ trainable numbers: 2 × 4096 × 8 = 65,536

65,536 / 16,777,216 = 0.4% of the parameters
```

- **The bet it makes:** the *change* you need is much simpler than the model itself, so it can be written as a low-rank (small `r`) matrix. In practice this holds surprisingly well.
- **What you get:** train on one modest GPU. The saved adapter is a few MB, not a few GB. Keep 20 adapters for 20 tasks and swap them at runtime over one base model.
- **You can merge it** back into W after training (`W + BA`) so inference costs nothing extra — but then it stops being swappable. Pick one.
- **QLoRA** = LoRA on top of a 4-bit quantized frozen base. That is what lets a 70B model be fine-tuned on a single 80 GB GPU. In 2026 QLoRA is the default starting point, and plain LoRA is what you use when you have GPU room to spare.

You will build the `W + BA` math yourself in NumPy in the notebook — 30 lines, and then LoRA stops being a mystery library flag.

**Rank in practice:** `r = 8–16` for style and format work, higher (32–64) when the task is genuinely far from what the model already does. Bigger `r` is not automatically better — it just costs more and overfits sooner.

---

## Alignment: Teaching Preference, Not Answers

SFT teaches "here is a good answer". But for most real questions there is no single good answer — there are better and worse ones. That needs a different training signal.

| Method | How it works | Where it stands in 2026 |
|---|---|---|
| **RLHF (PPO)** | train a separate reward model on human comparisons, then use reinforcement learning against it | powerful, complex, expensive, unstable — mostly frontier labs |
| **DPO** | skip the reward model: train directly on (prompt, better answer, worse answer) triples | **the default for most teams** — much simpler, nearly as good |
| **GRPO / RLVR** | reinforcement learning where the reward is *computed*, not judged — tests pass, JSON parses, the math checks out | the right choice when correctness is machine-verifiable; this is what powers reasoning models |

The rule of thumb that the 2026 stacks converged on: **QLoRA SFT first. Add DPO if you have preference pairs. Move to GRPO only when you have a reward you can compute automatically.**

Two failure modes to know by name:

- **Catastrophic forgetting** — you fine-tuned hard on your narrow task and the model got worse at everything else. Guard with a small held-out set of general questions you check before and after.
- **Reward hacking** — the model finds a way to score well without doing the job (writing empty tests that pass, padding answers because the reward model liked long ones). If you train against a metric, that metric stops being an honest measure.

---

## The Data Is The Job

Everyone asks about method. Method is the easy part. Results come from data.

- **Quality beats quantity.** 500 carefully checked examples routinely beat 50,000 scraped ones. The model copies your data — including its mistakes, its inconsistent formatting, and its bad habits.
- **Be consistent.** If half your examples answer in JSON and half in prose, you have taught the model to be inconsistent.
- **Cover the edges**, including what the model should *refuse* or say "I don't know" to. If none of your examples ever decline, the fine-tuned model never will.
- **Keep a real test split** that never appears in training. Fine-tuning on your eval set is the easiest way to fool yourself.

---

## Quick Summary

| Idea | In one line |
|---|---|
| Pre-training | learns language and world knowledge; not your job |
| SFT | teaches the model to follow instructions in your shape |
| Alignment | teaches which answer people prefer |
| Fine-tuning is for | behaviour, style, format, narrow closed tasks |
| Fine-tuning is not for | facts, especially facts that change — use RAG |
| LoRA | train two small matrices, freeze the rest — ~0.4% of parameters |
| QLoRA | LoRA on a 4-bit base — big models on one GPU |
| DPO | preference tuning without a reward model — today's default |
| GRPO / RLVR | RL with a reward you can compute, not judge |
| Data | quality and consistency beat volume, every time |
| Eval set | build it *before* training, or you cannot claim anything |

## Next

[5_hugging_face.md](5_hugging_face.md) — the toolbox that actually runs everything in this phase.

## Sources

- [Fine-Tuning LLMs in 2026: LoRA, QLoRA, DPO, GRPO Compared](https://futureagi.com/blog/fine-tuning-llms-unlocking-peak-performance/)
- [LLM Fine-Tuning Guide 2026: LoRA, QLoRA, DPO, GRPO, RLHF](https://futureagi.com/blog/llm-fine-tuning-guide-2025/)
- [Red Hat — RAG vs. fine-tuning](https://www.redhat.com/en/topics/ai/rag-vs-fine-tuning)
