# Designing for Wrong Answers

Your eval says 87% task success. Good number.

Now look at it from the other side. **13 out of every 100 users get a wrong answer**, delivered in the same confident tone as the right ones, on a screen you designed.

This file is about those 13. It is not a UI-polish chapter — it's the part of the system that decides whether 87% is a useful product or a liability.

The finding worth starting from: through 2026, most AI feature failures reported in the field are **design failures, not model failures.** The model was 90% right and the interface gave nobody a way to catch the other 10%.

---

## The Assumption Your UI Was Built On

Every interface pattern you know assumes a deterministic backend:

```
click  →  the system does the correct thing  →  show the result
```

Errors are exceptional and loud. A 500 is a red toast. A validation failure is a red field.

An AI feature breaks that assumption in a specific way:

```
click  →  the system does a probably-correct thing  →  show the result
                                                       (looks identical either way)
```

There is no red. The wrong answer renders in the same font as the right one. So the interface has to supply the signal the system can't:

> **Design rule: the user must always be able to tell what the system is about to do, see why it did it, and undo it.**

Everything below is a way of paying for that sentence.

---

## The Six Patterns

These six keep appearing in 2026 AI product work. You don't need all six for every feature — note 2's *cost of being wrong* tells you how many you need.

### 1. Intent preview — show the action before you take it

> "I'll cancel order #4471 and refund $89.00 to the card ending 4242."

State the action in the user's terms, before doing it. Not "Processing…". This one pattern kills the worst class of agent failure: doing the right operation to the wrong object.

### 2. Autonomy dial — suggest → confirm → act

Three levels, and the feature should be able to sit at any of them:

| Level | The model | The user | Use when a wrong answer costs |
|---|---|---|---|
| **Suggest** | drafts, does nothing | copies/applies manually | an hour or more |
| **Confirm** | prepares the action | approves each one | money |
| **Act** | does it | reviews after, can undo | a click or a minute |

Ship at **suggest**, earn your way up with eval data. The dial should be per-user or per-account, not a global constant — your power users will want *act*, your enterprise buyer will want *confirm*, and both are right.

### 3. Explainable rationale — cite, don't assert

For anything RAG-shaped ([4_applied_ai](../4_applied_ai/README.md)), show the source chunk inline and make it clickable. Two effects, and the second is the real one:

- The user can check the claim in two seconds instead of trusting it.
- **You can check it too.** "Cited document ID exists in the retrieved set" is a free code assertion from note 3.

Do not fake this. A citation the model generated without reading is worse than no citation, because it converts a visible guess into an invisible one.

### 4. Confidence signal — and the trap inside it

The idea: show high-confidence results plainly, and put low-confidence results behind a verification step.

The trap: **a model's stated confidence is not a probability.** Ask it "how sure are you, 0–100?" and it will happily say 95 on a fabricated answer. It is generating a plausible number, exactly like it generates a plausible fact.

Usable confidence proxies, roughly in order of trustworthiness:

| Proxy | How | Watch out for |
|---|---|---|
| **Retrieval score** | top-chunk similarity from your vector search | low score = "nothing relevant found" — the most honest signal you have |
| **Self-consistency** | sample the answer 3× at temperature > 0; do they agree? | costs 3× tokens; use only on high-stakes paths |
| **Token logprobs** | average log-probability of the generated answer | measures fluency more than truth; weak, but nearly free |
| **Verbal confidence** | ask the model | badly calibrated — do not gate actions on this |

Whichever you pick, **calibrate it**: take 100 outputs, bucket by confidence, and measure actual accuracy per bucket. If your "high confidence" bucket is 71% accurate, your threshold is decoration. That's the same validation move as checking an LLM judge against humans in note 3.

### 5. Action audit & undo — and it must be real

Every AI action needs a log entry (what, when, why, which model version) and a reversal.

The catch: **undo has to actually reverse the action.** Removing a sent email from your UI is not undo. If the action is irreversible — email sent, payment made, message posted, file deleted — you cannot ship at *act* level. You ship at *confirm*, or you build a delay window (the "sending… undo" pattern, 10 seconds of reversibility bought with 10 seconds of latency).

### 6. Escalation path — the exit to a human

Every AI feature needs a visible, one-click way out to a human or to the old manual flow. Users who know they can escape are measurably more willing to try the AI in the first place.

Also: **escalation rate is a top-tier product metric,** not a support cost. It's the honest version of your eval score, reported by the people who actually needed the answer.

---

## Predict This One

Before reading on:

> A team adds a confirmation dialog to an AI action, because a wrong action costs money. Six weeks later, 98% of dialogs are approved within 1.2 seconds. **Is the system safer than full automation?**

Think about it properly before continuing.

---

## Rubber-Stamping: the failure mode nobody designs for

The answer is **no — it is worse than full automation**, in one specific and important way.

At 1.2 seconds, nobody read anything. The human is not reviewing; they are clicking. The error rate is the model's error rate, unchanged. Nothing was caught.

But now the audit log says a human approved it. You have not added safety; you have moved liability onto a user who never had a chance to exercise it. The system is exactly as wrong as before and now the wrongness has a signature on it.

This is the most common way AI safety design fails in practice, and the fix is never "add another dialog":

- **Confirm only what's worth confirming.** Route the 90% of high-confidence, low-value actions to auto-with-undo, and reserve the dialog for the 10% that are genuinely risky. A dialog that fires rarely gets read.
- **Make the dialog show the diff, not the intent.** "Refund $890.00" beside "order total $89.00" gets caught. "Confirm refund?" does not.
- **Measure it.** Track approval latency and approval rate. If median approval time is under 2 seconds and approval rate is above 95%, your review step is theatre — say so out loud and redesign it.

---

## Two More Things the Model Gets Wrong

**Latency is a design surface, not just an engineering number.** You know TTFT and TPOT from [3_llm_inference.md](../3_modern_ai/3_llm_inference.md). Product side: streaming the first token in 400 ms feels fast even if the full answer takes 6 seconds, while a spinner for 3 seconds feels broken. Stream when there's text to stream. When there isn't — an agent doing tool calls — show the *steps* ("searching orders… found 3… drafting reply"), because a progress narrative buys far more patience than a spinner.

**"I don't know" is a feature, and it has to be built.** Models don't volunteer it; they produce a plausible answer instead. Retrieval score is your best trigger: if the top chunk is below threshold, do not call the model to "try anyway" — say *"I couldn't find anything about that in your documents,"* and offer the escalation path. Users forgive "I don't know" far more readily than a confident fabrication, and one fabrication costs you trust on every future correct answer.

---

## Cost and Trade-off

**Friction eats the benefit.** Every pattern here adds a step. A feature that saves 4 minutes and adds 90 seconds of confirmation clicking saves 2.5 minutes — and if you add enough, users go back to doing it manually. Measure time-saved end to end, including your safety furniture, not just model latency.

**Confidence UI can backfire.** Showing "72% confident" often makes people trust the 72% answer *more* than a bare answer, because a number reads as rigour. If your confidence isn't calibrated, showing it is actively harmful. Prefer behavioural design (verification step) over displaying a number.

**When you would skip most of this:** zero-consequence, easily-noticed outputs — autocomplete, tag suggestions, search ranking. Undo is free there because ignoring the suggestion *is* the undo. Don't build an approval workflow for a placeholder text generator.

**Non-negotiable in every case:** undo and escalation. If you build nothing else from this file, build those two.

---

## Your Turn

For your feature from notes 2 and 3:

1. Write out the **exact screen** a user sees when the model is wrong. Words, not a description of words.
2. Pick your confidence proxy and **calibrate it**: 100 outputs, bucketed, accuracy per bucket. Predict the accuracy of your top bucket before you measure. Most people are 10–20 points optimistic.
3. Ask the hard question: **is your undo real?** Trace the action to the database and to any external system. If a row is gone or an email left the building, you're at *confirm* level, not *act*.
4. Instrument rubber-stamping: log approval latency. If you already have a confirmation step anywhere, go measure its median right now.
5. Break it on purpose: force your retrieval score to 0 and check what the user sees. If it's a confident answer built from nothing, you have found your next task.

---

## Quick Summary

| Idea | In one line |
|---|---|
| The 13% | your eval's failure rate is a screen a real person reads |
| Most failures | design failures, not model failures |
| The design rule | see what it will do, see why it did it, undo it |
| Intent preview | state the action in the user's words before doing it |
| Autonomy dial | suggest → confirm → act; ship at suggest, earn the rest |
| Rationale | cite the source chunk — it's also a free code assertion |
| Verbal confidence | is a generated number, not a probability |
| Best confidence proxy | retrieval score; calibrate any proxy before gating on it |
| Undo | must reverse the real action, or you can't ship at *act* |
| Escalation rate | a product metric, not a support cost |
| Rubber-stamping | a 1.2-second approval adds liability, not safety |
| Streaming | first token in 400 ms beats a 3-second spinner |
| "I don't know" | has to be built; triggered by low retrieval score |
| Always build | undo and escalation, even when you skip the rest |

## Next

[5_unit_economics_and_pricing.md](5_unit_economics_and_pricing.md) — question four. It works and it fails gracefully. Now find out what it costs, and whether anybody can afford it.

## Sources

- [AI UX design patterns — how to design interfaces for AI-powered features](https://www.institutepm.com/knowledge-hub/ai-ux-design-patterns)
- [39 principles for designing human–AI interaction (UX Collective, 2026)](https://uxdesign.cc/39-principles-for-designing-human-ai-interaction-87be5fabdbbe)
- [Designing for trust — UX patterns for AI features](https://www.designkey.studio/post/designing-for-trust-ux-ai-features)
- [AI uncertainty and trust — a design framework](https://reloadux.com/blog/ai-uncertainty-trust-design-framework/)
- [UX design for AI products (2026) — practical guide](https://www.wavespace.agency/blog/ux-design-for-ai-products)
