# Fine-Tuning & Alignment

Everything up to now changed the **text you send**. Fine-tuning changes the **model itself** — the actual numbers inside it.

That difference matters more than it sounds. A prompt is rent: you pay for those tokens on every single call, forever. Fine-tuning is a purchase: you pay GPU hours once, and the behaviour is baked in. The catch is that you can edit a prompt in five seconds, and editing weights means training again.

One warning before anything else, because it is the mistake everyone makes: **fine-tuning is good at teaching behaviour and bad at teaching facts.** Keep reading and you will see exactly why.

Throughout this note I will use one app: **ShoeBox**, a support bot for a shoe shop. Its refund window is **30 days**.

Practice: [4_lora_from_scratch.ipynb](code/4_lora_from_scratch.ipynb) · Last rung of the ladder in [1_introduction](../1_introduction/README.md#thinking-like-an-ai-system-architect-production-framing).

---

## 1. What pre-training leaves missing

Pre-training is simple to describe. The model reads trillions of tokens and, at every position, guesses the next one. Do that for months and you get something that has absorbed grammar, facts, reasoning patterns, code, and the shape of almost every kind of document people write. That is **capability**.

You will often hear the next step explained like this: *"the base model was never taught to answer."* It is a useful handle, but taken literally it is wrong, and believing the literal version will cost you money.

Think about what was in the training data. FAQ pages. Support transcripts. Stack Overflow. Textbooks with worked solutions. The model has read more good answers than any human alive. It knows perfectly well how to answer a question.

What it does not have is a **preference**. All it learned is: given this text, what usually comes next? So when your prompt ends with a customer question, the model asks itself what really follows a customer question on a page like this — and on a shop's help page, the honest answer is often *another customer question*.

```
  after  "Customer: can I return these shoes?"          (shape, not measured)

  next token     what it really is                    share
  ─────────────────────────────────────────────────────────
  "Customer:"    another question in the list           30%   ← the page was a list
  "Agent:"       a reply is starting                    15%
  "Yes"          the actual answer                       5%
```

The answer is in there. It just loses the vote. And notice that every option beating it is a *correct* continuation of a real document. The model is not broken — it is doing exactly the job it was trained for.

There is a second problem, and people forget this one. Even when the model does answer, it keeps going:

```
  you send:    Customer: can I return these shoes?

  it writes:   Agent: Yes, within 30 days of delivery.
               Customer: do you ship to Canada?
               Agent: We ship there in 3-5 days.
               ...
```

It answered correctly, then wrote the customer's next turn, because web pages do not stop after one reply. Knowing *where your turn ends* is a behaviour nobody taught it.

Two facts show this is about preference, not ability. First, put three question-and-answer examples in the prompt and a base model will answer properly — with **zero weights changed**. You cannot prompt a skill into existence, so the skill was already there. Second, when OpenAI tuned a **1.3B** model and compared it against the raw **175B** GPT-3, human raters preferred the small tuned one. Over 100× fewer parameters, and people liked its answers more. Nothing was added to what it knew; only which behaviour came out on top.

> So say it precisely: pre-training taught it **how** to answer. It never taught it to **answer, and then stop**. Fine-tuning does not install a skill — it takes a mode the model already has and makes it the default.

One 2026 caveat. The "base" models you download today are not raw in the old sense — labs now mix instruction-shaped and synthetic question–answer data into pre-training, so many base checkpoints follow a plain instruction out of the box. The gap is smaller than this story suggests. It is not zero: stopping, consistent format, and refusals still come from tuning.

---

## 2. SFT — teaching it to answer, then stop

**SFT** stands for supervised fine-tuning, and it is the simplest thing that could work. You show the model pairs: here is a question, here is the answer it should have given.

```
  {"prompt": "can I return these shoes?",
   "answer": "Yes. You can return any item within 30 days of delivery."}
```

The important detail is that only the **answer** is graded. The model is not being trained to predict your question — it is being trained on what follows one. Do that a few thousand times and answering stops being one option among many. It becomes the default, and so does stopping at the end.

This is cheap by fine-tuning standards: roughly **500 to 5,000 pairs** and a few GPU hours, with one model in memory.

Where do the pairs come from? Your own logs first — real questions your users actually asked, with answers your team actually approved. When you run out, a bigger model can write more for you; that is distillation. But whatever the source, sit down and read **200 rows at random** before you train. Here is what skipping that costs: suppose your best support agents all open with "Hi there!". You never noticed. Now every ShoeBox reply says "Hi there!" and no prompt on earth will remove it, because it is in the weights.

> A falling loss does not mean your data was right. It means your data was **consistent**.

---

## 3. Where SFT runs out

SFT teaches *copy this*. That works until both answers are correct.

```
  A:   "Yes. You can return any item within 30 days of delivery."

  B:   "Pursuant to §4.2 of the Terms, returns may be effected within
        the applicable statutory window."
```

Both are true. Both are on-policy. B is simply worse for a shoe shop — it is cold, and a customer has to read it twice. But SFT has no way to express that. You can show it A and it will learn "write things like A". You can never show it B and say "and avoid this".

So you change the shape of the training data. Instead of one answer, you give it two and say which one won:

```
  {"prompt":   "can I return these shoes?",
   "chosen":   A,
   "rejected": B}
```

That is the whole idea behind every alignment method in the next section. **One answer teaches a copy. Two ranked answers teach a direction.**

---

## 4. Three ways to train on "better"

All three do the same job — push the model toward chosen answers and away from rejected ones. They differ in who does the judging and what that costs you.

**RLHF** was first. You collect around 10,000 human comparisons, train a separate scoring model to imitate those judgements, then use reinforcement learning to push the main model toward high scores. It works, and it is painful: **four models in memory** at once, and training that can quietly collapse.

**DPO** looked at that and asked why the scoring model exists at all. It is a middleman — trained on the pairs, then consulted about the pairs. So DPO throws it away and trains directly on the chosen/rejected data with a plain loss function. **Two models**, no reinforcement learning loop, far more stable. This is today's default.

**GRPO** removes the human instead. It only works when a program can grade the output, but then it works beautifully: the model writes several answers to the same prompt, a checker scores each one, and the good ones get pushed up relative to the bad. If ShoeBox must return valid JSON, you generate 8 answers, run `json.loads()` on each, and the 3 that parse go up while the 5 that fail go down. No labeller involved.

|                  | SFT           | RLHF              | DPO                        | GRPO                   |
|------------------|---------------|-------------------|----------------------------|------------------------|
| Who judges       | your writer   | a scoring model   | the human's pick, directly | a program              |
| Models in memory | 1             | 4                 | 2                          | 2 + checker            |
| Use when         | always, first | you are a big lab | you have ranked pairs      | a program can check it |

**What to actually do:** start with SFT and stop there. Reach for DPO only when you have real ranked pairs and can explain *why* one answer won. Use GRPO only when correctness is something a program can decide. Skip RLHF — you are not going to run four models to beat DPO by a little.

---

## 5. The part people get wrong: it changed HOW, not WHAT

Sooner or later someone will say *"let's fine-tune it on our policy PDF."* This sounds reasonable and it is the most expensive misunderstanding in the field.

Say you do it. You train ShoeBox on 2,000 real refund tickets. Training goes fine, loss drops, the replies sound great. Then a customer asks about returns and the bot says:

```
  "Yes, you can return any item within 21 days of delivery."
```

Right voice. Wrong number. And nothing to check it against.

The reason is that gradient descent had no reason to memorise "30 days". Across 2,000 tickets, what was consistent was the *shape* of a confident refund reply — the tone, the structure, the reassurance. The specific number varied, appeared rarely, and got smeared into an approximation. The model learned the style perfectly and the fact loosely.

|                           | Put it in the weights | Put it in RAG     |
|---------------------------|-----------------------|-------------------|
| Change one fact           | retrain the model     | edit one row      |
| Where did this come from? | nobody can tell you   | show the document |
| Time to make the change   | GPU hours             | seconds           |

**Facts go in RAG. Behaviour goes in the weights.** If the thing you want can change next quarter, it does not belong in a training run.

---

## 6. LoRA and QLoRA — why this is affordable at all

Updating every weight in an 8B model needs roughly **128 GB** of GPU memory, because you are storing gradients and optimiser state for all 8 billion numbers. That is a cluster, and it is why fine-tuning used to be a big-lab activity.

LoRA's insight is that you do not need to move every weight. Freeze the original weight `W` completely, and add a small side path next to it — two thin matrices, `B` and `A`. Only those get trained. `B` starts at **zero**, so on step 0 the side path contributes nothing and the model is exactly the original. From there it learns the small correction your task needs.

QLoRA adds one more trick: squash the frozen base model to 4 bits. It is frozen anyway, so the precision loss costs you little.

| 8B model       | GPU memory | Runs on         |
|----------------|------------|-----------------|
| Full fine-tune | ~128 GB    | a cluster       |
| **LoRA**       | ~18 GB     | one 24 GB GPU   |
| **QLoRA**      | ~6 GB      | a gaming laptop |

The file you end up with is tiny:

```
  rank r = 8

  4 matrices per layer × 32 layers   =   8,400,000 numbers   (0.1% of 8B)
  × 2 bytes per number               =          17 MB        ← the adapter file
```

That 17 MB file is the **adapter**, and its size is the reason this scales. Suppose ShoeBox sells to 20 shops, each wanting its own tone. You do not need 20 models:

```
  1 base model (16 GB, shared) + 20 adapters × 20 MB   =    16.4 GB
  20 separate fine-tuned models                        =   320.0 GB
```

A request comes in with `shop_id = B`, and the server keeps the same base loaded and swaps in adapter B. You can also **merge** an adapter into the base for simpler serving — but then you have one fused model again and you lose the swapping.

One more thing: none of this happens while a user waits. Training is an offline job that produces a file. Serving loads that file.

---

## 7. Two ways it breaks

**Catastrophic forgetting.** The model gets better at your task and worse at everything else. ShoeBox becomes excellent at refunds, then someone asks "do these run small?" and it starts talking about returns anyway. On a general benchmark it fell from 68% to 51% — you did not notice, because you were only measuring refunds. The guard is cheap: keep a small set of general questions and score it before and after every training run.

**Reward hacking.** This one only shows up with GRPO, and it is almost funny. You reward "the JSON parses". The model discovers that `{}` parses. Perfect score, zero value. The rule to remember: the moment you train against a number, that number stops being an honest measure of the thing you cared about.

---

## 8. What you should actually choose

Work down this list and stop at the first yes. Everything above the line is cheaper and faster than everything below it.

| Question                                               | Do this                        | Cost   |
|--------------------------------------------------------|--------------------------------|--------|
| Is the prompt unclear?                                 | fix the prompt                 | free   |
| Is only the *format* wrong?                            | few-shot, or structured output | cheap  |
| Does it need facts, or facts that change?              | **RAG**                        | medium |
| Does it need to take actions?                          | tools / agents                 | medium |
| One exact style on every call, and the prompt is long? | **fine-tune**                  | high   |
| Closed task a program can grade?                       | **fine-tune**                  | high   |
| Must a small model match a big one?                    | **fine-tune** (distillation)   | high   |

Walk ShoeBox down it. Prompt unclear? No. Format wrong? No — few-shot fixed that. Needs facts that change? **Yes**, the refund policy changes. So: RAG, and stop. No training.

Six months later the situation is different. The prompt now carries 40 lines of style rules, and you pay for those tokens on every single call. *Now* fine-tune — for style only. The policy stays in RAG.

And before any of this: **if you have no eval set, stop.** Build 50–200 real cases with known-good answers and keep them out of training. Without that, "it feels better" is the only claim you can make, and you cannot tell improvement from forgetting.

When you do train, the defaults are boring and correct: start with **QLoRA + SFT**, add **DPO** only once you have ranked pairs, use **GRPO** only when a program can grade it. Set rank `r` to **8–16** for style work — bigger is not better, it just costs more and overfits sooner. And 500 examples you have actually read will beat 50,000 you scraped.

---

## Check Yourself

No notes, no AI. Say these out loud.

1. A raw model can already write a correct refund answer. So what two things is SFT actually fixing?
2. Show the A/B example. Why can SFT not teach it?
3. What does DPO remove from RLHF, and what does that save you?
4. Where does the 17 MB come from, and why does `B` start at zero?
5. Someone says "let's fine-tune on our policy PDF". What do you say?

---

## Quick Summary

| Idea              | In one line                                                                     |
|-------------------|---------------------------------------------------------------------------------|
| Raw model         | *can* answer — but answering is one continuation among many, and it never stops |
| SFT               | question→answer pairs; teaches shape, format, tone, stopping                    |
| SFT's limit       | one answer teaches a copy, not a direction                                      |
| RLHF / DPO / GRPO | 4 models / 2 models / a program grades it                                       |
| The big split     | changes **how** it answers, barely **what** it knows                            |
| Not for           | facts, especially facts that change — use RAG                                   |
| LoRA / QLoRA      | 18 GB / 6 GB for an 8B; 0.1% trainable; a 17 MB adapter                         |
| Swapping          | 20 styles on one GPU ≈ 16.4 GB, not 320 GB                                      |
| Forgetting        | 68% → 51% on general tasks; score a general set before and after                |
| Reward hacking    | train against a number and it stops being honest                                |
| Eval set          | build it before training, or you can claim nothing                              |

## Next

[5_hugging_face.md](5_hugging_face.md) — the toolbox that runs everything in this phase.

## Sources

- [Fine-Tuning LLMs in 2026: LoRA, QLoRA, DPO, GRPO Compared](https://futureagi.com/blog/fine-tuning-llms-unlocking-peak-performance/)
- [LLM Fine-Tuning Guide 2026: LoRA, QLoRA, DPO, GRPO, RLHF](https://futureagi.com/blog/llm-fine-tuning-guide-2025/)
- [Red Hat — RAG vs. fine-tuning](https://www.redhat.com/en/topics/ai/rag-vs-fine-tuning)
- [Ouyang et al. — InstructGPT](https://arxiv.org/abs/2203.02155) — the 1.3B vs 175B preference result in section 1
