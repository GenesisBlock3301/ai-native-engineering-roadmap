# Fine-Tuning & Alignment

## The Whole File in One Table

Fine-tuning means changing the model's **weights** on your own data, instead of only changing your **prompt**.

| | **Fine-tuning (SFT)** | **Alignment (DPO / RLHF)** |
|---|---|---|
| **Why** | The model answers, but not in your shape, style, or format. Or the prompt that gets it right is long and you pay for it every call. | Most real questions have no single right answer — only better and worse ones. |
| **How** | Train on `(instruction → good answer)` pairs. | Train on `(prompt, better answer, worse answer)` triples. |
| **Where** | **Offline**, on a GPU, once. Produces a **file**. Never at request time. | Same place, after SFT. |
| **Costs you** | GPU hours — but the real cost is building the data and the eval set. | More data, more complexity, more ways to fool yourself. |

One line to keep: **fine-tuning is very good at teaching behaviour. It is bad and expensive at teaching facts.**

---

## How To Read This File

```
Part 1   WHERE it happens        training time vs serving time
Part 2   WHY models need it      the three stages, and capability vs behaviour
Part 3   HOW LoRA works          train 0.4% of the model
Part 4   HOW alignment works     DPO, RLHF, GRPO
Part 5   WHAT you should choose  starting with: should you do this at all?
```

| Question you will be asked at work | Which part |
|---|---|
| "Let's fine-tune on our docs so it knows our product." | Part 5 — the answer is usually no |
| "We need 20 different tones for 20 customers. 20 models?" | Part 1 |
| "Can we fine-tune a 70B on one GPU?" | Part 3 |
| "The model got better at our task but worse at everything else." | Part 4 |

Practice: [4_lora_from_scratch.ipynb](code/4_lora_from_scratch.ipynb).

This is the last rung of the ladder in [1_introduction](../1_introduction/README.md#thinking-like-an-ai-system-architect-production-framing). Most teams reach for it far too early and regret it. So this file is mostly about **when not to**.

---

# Part 1 — WHERE This Actually Happens

The single most useful fact: **fine-tuning never happens while a user is waiting.** It is an offline batch job that produces an artefact. Then that artefact gets loaded at serving time.

```
  TRAINING TIME  (offline, once)        SERVING TIME  (online, every request)
  ────────────────────────────          ──────────────────────────────────

  base model (frozen)                   base model sits in GPU memory
        +                                        │   (16 GB, loaded once)
  your 500 examples                              │
        │                                        │
        ▼                                        ▼
   a few GPU-hours                        ┌─────────────────┐
        │                                 │  base + adapter │ → your style
        ▼                                 └─────────────────┘
   ADAPTER FILE  ~20 MB  ──────────────────────────┘
```

Now the part that changes your architecture:

```
  ONE GPU, ONE BASE MODEL, TWENTY CUSTOMERS

  ┌──────────────────────────────────────────┐
  │  base model            16 GB (shared)    │
  ├──────────────────────────────────────────┤
  │  customer A adapter    20 MB             │
  │  customer B adapter    20 MB             │  swap per request
  │  ... 18 more           360 MB            │
  └──────────────────────────────────────────┘
     total ≈ 16.4 GB — not 20 × 16 GB
```

**That is the answer to "20 tones for 20 customers — one model or twenty?"** One base model, twenty small adapters, swapped at runtime. Twenty full models would need 320 GB and would be absurd.

You lose this if you **merge** the adapter into the base weights (Part 3). Merging makes inference slightly simpler but kills the swapping. Pick one.

---

# Part 2 — WHY Models Need This

Three stages make a model. Knowing which stage does what tells you which one you need.

```
1. PRE-TRAINING     predict the next token over trillions of tokens of internet
                    → learns language, facts, code, reasoning ability
                    → costs millions of dollars. You will never do this.
                          ↓
2. SFT              supervised fine-tuning on (instruction → good answer) pairs
   (instruction     → learns to be a helpful assistant instead of an autocomplete
    tuning)         → you CAN do this, with a few hundred to a few thousand examples
                          ↓
3. ALIGNMENT        train on human preferences: "answer A is better than answer B"
   (RLHF / DPO)     → learns tone, safety, refusals, what people actually want
```

Stage 1 gives **capability**. Stages 2 and 3 give **behaviour**. That split is the whole point:

> Fine-tuning changes **how** the model answers. It barely changes **what** it knows.

Facts that change belong in retrieval ([4_applied_ai](../4_applied_ai/README.md)), not in weights. Weights are a terrible database:

| | Weights | Retrieval |
|---|---|---|
| Update one fact | retrain the model | edit one row |
| Where did this answer come from? | unknowable | show the source document |
| Cost to change | GPU hours | seconds |

---

# Part 3 — HOW LoRA Works

Full fine-tuning updates every weight. Look at what that costs for an 8B model:

| Method | GPU memory needed | Fits on |
|---|---|---|
| **Full fine-tune** | ~128 GB — weights + gradients + optimizer state, roughly 16 bytes per parameter | a multi-GPU cluster |
| **LoRA** | ~18 GB — frozen fp16 base + a tiny trainable piece | one 24 GB GPU |
| **QLoRA** | ~6 GB — frozen **4-bit** base + the same tiny piece | a gaming laptop |

And full fine-tuning gives you a whole new 16 GB model file per task. LoRA gives you a 20 MB adapter.

## The trick

```
Frozen original weight W   (4096 × 4096 = 16.7M numbers)

     output = W·x   +   B·A·x
               ↑          ↑
            frozen   trainable, tiny

  A is 4096 × r      B is r × 4096      with r = 8

  trainable numbers = 2 × 4096 × 8 = 65,536

  65,536 / 16,777,216 = 0.4% of the parameters
```

**The bet it makes:** the *change* you need is much simpler than the model itself, so it can be written as a low-rank (small `r`) matrix. In practice this holds surprisingly well.

**Why does `B` start at zero?** So that at step 0, `B·A·x = 0` and the output is *exactly* the original model. Training starts from "unchanged" and moves away, instead of starting from a random jolt that damages a working model.

**QLoRA** = LoRA on top of a 4-bit quantized frozen base. That is what puts a 70B fine-tune on a single 80 GB GPU. In 2026 QLoRA is the default starting point; plain LoRA is what you use when you have GPU room to spare.

You build the `W + BA` math yourself in NumPy in the notebook — about 30 lines, and then LoRA stops being a mystery library flag.

---

# Part 4 — HOW Alignment Works

SFT teaches "here is a good answer". But for most real questions there is no single good answer — only better and worse ones. That needs a different signal.

| Method | How it works | Where it stands in 2026 |
|---|---|---|
| **RLHF (PPO)** | train a separate reward model on human comparisons, then do reinforcement learning against it | powerful, complex, expensive, unstable — mostly frontier labs |
| **DPO** | skip the reward model — train directly on (prompt, better, worse) triples | **the default for most teams**: much simpler, nearly as good |
| **GRPO / RLVR** | reinforcement learning where the reward is *computed*, not judged — tests pass, JSON parses, the math checks out | the right choice when correctness is machine-checkable; this is what powers reasoning models |

The order the 2026 stacks converged on:

```
  QLoRA SFT   →   add DPO if you have preference pairs   →   GRPO only if
  (start here)                                                you can compute
                                                              the reward
```

## Two failure modes to know by name

**Catastrophic forgetting** — you trained hard on your narrow task and the model got worse at everything else.

> Guard: keep a small held-out set of *general* questions. Score it before and after every run. If general performance dropped, your gain was not free.

**Reward hacking** — the model finds a way to score well without doing the job. Writing empty tests that pass. Padding answers because the reward model liked long ones.

> The rule: the moment you train against a metric, that metric stops being an honest measure of anything.

---

# Part 5 — WHAT You Should Choose

### 1. First gate — should you fine-tune at all?

Go down this list **in order**. Stop at the first "yes". Most teams stop in the first three rows and never needed training.

| Question | If yes → | Cost |
|---|---|---|
| Is the prompt just unclear or under-specified? | fix the prompt | free |
| Does it get the *format* wrong? | few-shot examples, or forced structured output | cheap |
| Does it need facts it lacks, or facts that change? | **RAG** — fine-tuning cannot fix this | medium |
| Does it need to take actions? | tools / agents | medium |
| Does it need a very specific style on *every* call, and that prompt is long and costly? | **fine-tune** | high |
| Is the task closed and machine-checkable (extraction, classification, tool-call accuracy)? | **fine-tune** — a small tuned model often beats a big prompted one | high |
| Do you need a small cheap model to match a big one on one narrow job? | **fine-tune** (distillation) | high |

**The trap you will actually meet:** *"the model doesn't know our internal docs, so let's fine-tune on them."*

That mostly does not work. The model picks up the **sound** of your docs and still invents the details. Use RAG. If you need both, do RAG first and fine-tune later, only for style and format.

### 2. Before you start — do you have an eval set?

If the answer is no, **stop.** You need 50–200 real cases with a known-good answer and a way to score them.

Without it you cannot tell whether fine-tuning helped, and *"it feels better"* is not an engineering claim. A rough eval set built in an afternoon beats a perfect training run you cannot measure. Full treatment in [6_production_ai](../6_production_ai/README.md).

### 3. Which method?

| Your situation | Choose |
|---|---|
| First attempt, any size model | **QLoRA SFT** — cheapest thing that works |
| You have GPU room and want max quality | LoRA SFT |
| You have preference pairs (A better than B) | add **DPO** |
| Correctness is machine-checkable | **GRPO** |
| You are a frontier lab | RLHF/PPO — you are not, so skip it |

### 4. Which rank (`r`)?

| Task | Rank | Note |
|---|---|---|
| Style, tone, output format | **8–16** | almost always enough |
| Genuinely new task, far from the base model | 32–64 | rarely needed |

Bigger `r` is **not** automatically better. It costs more and overfits sooner. Start at 8 and only raise it if the loss curve says the model cannot fit your data.

### 5. Merge the adapter, or keep it separate?

| | Keep separate | Merge into base |
|---|---|---|
| Many customers/tasks on one GPU | ✅ yes | ❌ no |
| Simplest possible inference | ❌ | ✅ |
| Swap behaviour at runtime | ✅ | ❌ |

Default: **keep it separate** unless you have exactly one task forever.

### 6. The data rules — this is where results actually come from

Everyone asks about method. Method is the easy part.

- **Quality beats quantity.** 500 carefully checked examples routinely beat 50,000 scraped ones. The model copies your data, *including* its mistakes, inconsistent formatting, and bad habits.
- **Be consistent.** If half your examples answer in JSON and half in prose, you have taught the model to be inconsistent.
- **Cover the edges** — including what the model should *refuse* or say "I don't know" to. If none of your examples ever decline, your fine-tuned model never will.
- **Keep a real test split** that never appears in training. Training on your eval set is the easiest way to fool yourself.

---

## Check Yourself

No notes, no AI. Say it out loud.

1. **Where** does fine-tuning run — at request time or offline? What artefact does it produce, and how big is it?
2. Twenty customers want twenty tones. How much GPU memory, and why not 20 × 16 GB?
3. Give three cases where fine-tuning is right and three where it is the wrong tool.
4. Explain LoRA to a backend engineer in three sentences.
5. Why does `B` start at zero?
6. What is QLoRA, and which single constraint does it remove?
7. DPO vs RLHF vs GRPO — one line each, and when you'd pick each.
8. What is catastrophic forgetting, and how do you catch it *before* your users do?
9. Someone says "let's fine-tune on our docs so it knows our product." What do you say?

---

## Quick Summary

| Idea | In one line |
|---|---|
| Where it runs | offline, once — never at request time; output is a file |
| Adapter swapping | one base model + many 20 MB adapters = many behaviours, one GPU |
| Pre-training | learns language and world knowledge; not your job |
| SFT | teaches the model to follow instructions in your shape |
| Alignment | teaches which answer people prefer |
| Fine-tuning is for | behaviour, style, format, narrow closed tasks |
| Fine-tuning is not for | facts — especially facts that change. Use RAG |
| LoRA | train two small matrices, freeze the rest — ~0.4% of parameters |
| B starts at zero | so training begins from the unchanged model |
| QLoRA | LoRA on a 4-bit base — 70B on one GPU |
| DPO | preference tuning without a reward model — today's default |
| GRPO / RLVR | RL with a reward you can compute, not judge |
| Rank | 8–16 for style; bigger is not better |
| Data | quality and consistency beat volume, every time |
| Eval set | build it *before* training, or you cannot claim anything |

## Next

[5_hugging_face.md](5_hugging_face.md) — the toolbox that actually runs everything in this phase.

## Sources

- [Fine-Tuning LLMs in 2026: LoRA, QLoRA, DPO, GRPO Compared](https://futureagi.com/blog/fine-tuning-llms-unlocking-peak-performance/)
- [LLM Fine-Tuning Guide 2026: LoRA, QLoRA, DPO, GRPO, RLHF](https://futureagi.com/blog/llm-fine-tuning-guide-2025/)
- [Red Hat — RAG vs. fine-tuning](https://www.redhat.com/en/topics/ai/rag-vs-fine-tuning)
