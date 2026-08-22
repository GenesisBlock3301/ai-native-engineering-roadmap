# Fine-Tuning & Alignment

Fine-tuning changes the **weights** — the numbers inside the model. A prompt only changes the text you send. You pay for a prompt on every call. You change weights once.

**Warning:** it is good at teaching **behaviour**. It is bad at teaching **facts**.

**Story used everywhere below:** **ShoeBox**, a support bot for a shoe shop. Refund window: **30 days**.

Practice: [4_lora_from_scratch.ipynb](code/4_lora_from_scratch.ipynb) · Last rung of the ladder in [1_introduction](../1_introduction/README.md#thinking-like-an-ai-system-architect-production-framing).

---

## 1. A raw model only continues text

Pre-training = read trillions of tokens, guess the next one. That gives **capability**. It never taught the model to *answer*.

**Example**

```
you send:   Customer: can I return these shoes?
it writes:  Customer: do you ship to Canada?
            Customer: is there a wide fit?
```

It knows the rule. But support pages are full of question lists, so more questions came next.

→ **Fix: SFT.**

---

## 2. SFT — teach it to answer

Show it question–answer pairs. Only the **answer** is graded, not the question. So it learns to answer and stop.

**Example — 2 training rows**

```
{"prompt": "can I return these shoes?",
 "answer": "Yes. You can return any item within 30 days of delivery."}

{"prompt": "I lost the box",
 "answer": "That is fine. The box is not needed for a return."}
```

Same shape every time. The model copies the shape.

- **Cost:** 500–5,000 pairs, a few GPU-hours, 1 model in memory.
- **Good for:** answering at all, format, tone, one narrow task. **Bad for:** facts.
- **Where pairs come from:** your own logs first. Then a big model writes the rest (distillation). A human writes the spec and the eval set — and reads 200 rows at random.

**Example — what skipping that read costs**

Your best 4 agents all open with "Hi there!". You never checked. Now every ShoeBox reply says "Hi there!" and no prompt can remove it. It is in the weights.

> A falling loss does not mean the data was right. It means the data was **consistent**.

---

## 3. SFT's limit — better vs worse

One answer teaches *copy this*. That fails when both answers are correct.

**Example**

```
A:  "Yes. You can return any item within 30 days of delivery."
B:  "Pursuant to §4.2 of the Terms, returns may be effected within
     the applicable statutory window."
```

B is **correct and worse**. SFT can say "A is good". It can never say "avoid B".

So you rank them:

```
{"prompt": "can I return these shoes?", "chosen": A, "rejected": B}
```

**One answer teaches a copy. Two ranked answers teach a direction.**

---

## 4. Three ways to train on "better"

**RLHF** — train a scoring model from 10,000 human picks, then push the model toward high scores. **4 models in memory.** Breaks often. Big labs only.

**DPO** — that scoring model is a middleman. Drop it, train straight on the pairs. **2 models.** Never writes during training, so it stays calm. **Today's default.**

**GRPO** — no human. A program grades the answer.

**Example — GRPO on ShoeBox JSON output**

```
one prompt → model writes 8 answers → run json.loads() on each
             0 0 1 0 1 1 0 0   (average 0.375)
             the 3 that parsed → UP     the 5 that failed → DOWN
```

| | SFT | RLHF | DPO | GRPO |
|---|---|---|---|---|
| Judge | your writer | scoring model | the human, direct | a program |
| Models in memory | 1 | 4 | 2 | 2 + checker |
| Use when | always, first | you are a big lab | you have ranked pairs | a program can check it |

**Default: SFT, and stop.** Add DPO only when you have real pairs and can say why one won. Skip RLHF.

---

## 5. It changed HOW, not WHAT

Pre-training gave capability. SFT and DPO gave behaviour.

> Fine-tuning changes **how** it answers. It barely changes **what** it knows.

| | Weights | RAG |
|---|---|---|
| Change one fact | retrain | edit one row |
| Where did this come from? | unknown | show the document |
| Time to change | GPU hours | seconds |

**Example — the trap you will meet**

Someone says *"fine-tune it on our policy PDF"*. You train on 2,000 refund tickets. The model does not store "30 days" — it stores **the shape of a confident refund answer**. Then it tells a customer:

```
"Yes, you can return any item within 21 days of delivery."
```

Right voice. Wrong number. No source to check. **Facts → RAG.**

---

## 6. LoRA and QLoRA

Training every weight is too expensive. For an 8B model:

| Method | GPU memory | Fits on |
|---|---|---|
| Full fine-tune | ~128 GB | a cluster |
| **LoRA** | ~18 GB | one 24 GB GPU |
| **QLoRA** | ~6 GB | a gaming laptop |

Freeze the big weight `W`. Add a tiny side path `B·A`. Train only that. `B` starts at **zero**, so step 0 is exactly the old model.

**QLoRA** = same, but the frozen base is squashed to 4 bits. Puts a 70B fine-tune on one GPU. Default first try in 2026.

**Example — where the file size comes from**

```
one matrix, r=8      2 × 4096 × 8        =     65,536 numbers
4 per layer × 32 layers                  =  8,388,608 trainable  (0.1% of 8B)
at 2 bytes each                          =      17 MB
```

That 17 MB file is the **adapter**.

---

## 7. Where it runs

Never while a user waits. It is an offline job that makes a file. At serve time: base model + adapter.

**Example — ShoeBox sells to 20 shops**

```
base model            16 GB   (shared)
20 adapters × 20 MB  400 MB
                   ─────────
                   ≈ 16.4 GB      not 20 × 16 GB = 320 GB
```

Request comes in with `shop_id = B` → same base, swap adapter B. You can **merge** the adapter into the base for simpler serving, but then you lose the swapping.

---

## 8. Two ways it breaks

**Catastrophic forgetting** — better at your task, worse at everything else.

> **Example:** ShoeBox nails refunds. Ask "do these run small?" and it talks about returns. General score fell 68% → 51%.
> **Guard:** score a small set of general questions before and after every run.

**Reward hacking** — it scores well without doing the job.

> **Example:** GRPO rewards "the JSON parses". The model learns to return `{}`. Perfect score. Zero value.
> **Rule:** train against a number and that number stops being honest.

---

## 9. What to choose

Stop at the first yes.

| Question | Do this | Cost |
|---|---|---|
| Prompt unclear? | fix the prompt | free |
| Wrong *format*? | few-shot, or forced structured output | cheap |
| Needs facts, or facts that change? | **RAG** | medium |
| Needs to take actions? | tools / agents | medium |
| Needs one exact style every call, and that prompt is long? | **fine-tune** | high |
| Closed task a program can check? | **fine-tune** | high |
| Small cheap model must match a big one? | **fine-tune** (distillation) | high |

**Example — walk it for ShoeBox**

```
prompt unclear?   no      format wrong?   no, few-shot fixed it
needs facts?      YES, the policy changes  →  RAG. Stop here.
```

Six months on, the prompt holds 40 lines of style rules you pay for every call. *Now* fine-tune — style only. Policy stays in RAG.

**No eval set → stop.** 50–200 real cases with known-good answers, kept out of training. Otherwise "it feels better" is your only claim.

| Situation | Choose |
|---|---|
| First attempt | **QLoRA + SFT** |
| You have ranked pairs | add **DPO** |
| A program can grade it | **GRPO** |

Rank `r`: **8–16** for style. Bigger is not better — it costs more and overfits sooner.

**Data rules:** 500 checked examples beat 50,000 scraped ones · keep the format consistent · include what it should refuse · keep a test split it never sees.

---

## Check Yourself

No notes, no AI. Say it out loud.

1. Why does a raw model reply with more questions? Does it know the answer?
2. Show the A/B example. Why can SFT not teach it?
3. What does DPO remove from RLHF, and what does that save?
4. Where does the 17 MB come from? Why does `B` start at zero?
5. Someone says "fine-tune on our policy PDF". What do you say?

---

## Quick Summary

| Idea | In one line |
|---|---|
| Raw model | continues text; never taught to answer |
| SFT | question–answer pairs; teaches shape, format, tone |
| SFT's limit | one answer teaches a copy, not a direction |
| Ranked pairs | (prompt, chosen, rejected) — teaches what to avoid |
| RLHF / DPO / GRPO | 4 models / 2 models / a program grades it |
| The big split | changes **how** it answers, barely **what** it knows |
| Not for | facts, especially facts that change — use RAG |
| LoRA / QLoRA | 18 GB / 6 GB for an 8B; 0.1% trainable; 17 MB adapter |
| Swapping | 20 styles on one GPU = 16.4 GB, not 320 GB |
| Forgetting | score a general set before and after |
| Reward hacking | train against a number and it stops being honest |
| Eval set | build it before training, or you can claim nothing |

## Next

[5_hugging_face.md](5_hugging_face.md) — the toolbox that runs everything in this phase.

## Sources

- [Fine-Tuning LLMs in 2026: LoRA, QLoRA, DPO, GRPO Compared](https://futureagi.com/blog/fine-tuning-llms-unlocking-peak-performance/)
- [LLM Fine-Tuning Guide 2026: LoRA, QLoRA, DPO, GRPO, RLHF](https://futureagi.com/blog/llm-fine-tuning-guide-2025/)
- [Red Hat — RAG vs. fine-tuning](https://www.redhat.com/en/topics/ai/rag-vs-fine-tuning)
