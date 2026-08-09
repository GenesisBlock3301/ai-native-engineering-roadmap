# Deciding What to Build

Question one: **should this exist?**

Most teams skip this. They get an idea, it sounds like AI, and the next conversation is already about which vector database to use. Then eight weeks later the feature ships, works 70% of the time, costs $4 per user per month, and quietly gets turned off.

This file is the filter that runs before any of that.

---

## The Ladder, Extended Downwards

[1_introduction](../1_introduction/README.md) gave you a ladder. Each rung costs more, so you only climb when the rung below genuinely fails:

```
Just ask (zero-shot)  →  Few-shot  →  CoT  →  RAG  →  Agent  →  Fine-tune
```

Product engineering adds **two rungs below the bottom**, and they are the ones people forget:

```
0a. A database query
      ↓  (the answer isn't already in your data)
0b. A rule / regex / lookup table
      ↓  (the input is genuinely open-ended)
1.  Just ask (zero-shot)
      ↓  ...the rest of the ladder
```

**Rung 0a — is the answer already in your database?**

"Show me my top customers this quarter" is a `SELECT`. Wrapping it in an LLM gives you a slower, more expensive `SELECT` that is occasionally wrong and can never be unit-tested.

**Rung 0b — is the input actually open-ended?**

Real example shape: a team builds an LLM classifier to route support tickets into 6 categories. It costs $0.002 per ticket, takes 900 ms, and hits 88% accuracy.

Then someone checks the data: 71% of tickets contain one of four exact phrases that map perfectly to a category. A four-line rule handles those at 0 ms and 100% accuracy, and only the remaining 29% go to the model. Same overall quality, ~70% less cost and latency, and now the common path is testable.

The pattern is general: **rules for the head of the distribution, model for the tail.** You have already seen this idea — it's the same shape as model routing in note 5, just applied before the model instead of between models.

---

## Try This First

Before reading on, take a feature you want to build and answer:

> Which rung is the lowest one that could possibly work — and what specific evidence would tell me it doesn't?

If your answer to the second half is "I feel like it won't," you have not tested a rung, you have skipped one. That is the single most common way an AI project becomes expensive.

---

## The Number That Decides the Design: Cost of Being Wrong

Everything about how you build an AI feature follows from one number: **what does a wrong answer cost?**

Not model accuracy. The *consequence*.

| Cost of a wrong answer | Example | What you're allowed to build |
|---|---|---|
| A click | wrong tag suggested on a note | full auto, no confirmation, easy undo |
| A minute | wrong support category, wrong draft reply | auto with visible undo and an edit box |
| An hour | wrong code refactor, wrong data migration | suggest only, human confirms before apply |
| Money | wrong refund, wrong invoice line | human approves every action, full audit log |
| Someone gets hurt | dosage, safety alert, legal filing | model assists a qualified human — never acts |

Read the table twice, because the arrow only points one way: **the consequence sets the automation level.** A better model does not let you move up a row. A 95%-accurate model that issues refunds is worse than a 70%-accurate model that drafts refunds for approval, because 5% of your refund volume is a real number of real dollars.

This is also where the 95% failure statistic gets concrete. Teams pick a high-consequence use case ("AI handles compliance review"), discover halfway through that a human must check every output anyway, and then the feature saves nobody any time.

---

## The One-Page Spec

Write this **before** any feature code. One page, no more. If you can't fill in a line, that gap is the actual first task.

```
FEATURE: ______________________________________________

1. USER + JOB
   Who: ____________  Doing what today: ____________
   Time it takes them now: ____ min, ____ times per week

2. THE NON-AI VERSION
   Lowest rung that could work: ____________
   Why it isn't enough (evidence, not opinion): ____________

3. COST OF BEING WRONG
   A wrong answer costs: ____________
   Therefore automation level: suggest / confirm / auto

4. SUCCESS METRIC
   Task success measured as: ____________
   Baseline today: ____%   Target: ____%
   Eval set size: ____ examples   Who wrote them: ____________

5. BUDGET
   Max cost per task: $______   Max p95 latency: ______ ms

6. FALLBACK
   When the model fails, the user sees: ____________

7. KILL CRITERIA
   I stop this project if, by ____ (date):
     - task success < ____%, OR
     - cost per task > $______, OR
     - fewer than ____% of users keep the output unedited
```

Three lines carry most of the weight.

**Line 2 (the non-AI version).** This is the rung check. "Evidence, not opinion" means you tried it on 50 real examples and it got 61% — not that it felt weak.

**Line 4 (baseline).** A target with no baseline is meaningless. "90% accuracy" sounds good until you learn the humans doing it today hit 97%.

**Line 7 (kill criteria).** Written *before* you start, this is a rule. Written after, it's a negotiation you always lose, because by then someone's quarter depends on the feature shipping. Most of the 95% are projects that no one was ever allowed to kill.

---

## What the 5% Do Differently

Of the enterprise pilots that did produce value, the reported pattern is not a smarter model. It's **connection to real institutional data and a real workflow** — the system reaches into the tools people already work in, rather than being a chat box bolted on beside them. Reported return for that group is around $3.70 per $1 spent.

Translated into your one-page spec: line 1 is the one that predicts success. If you cannot name the person, the task, and the minutes it takes them today, you are building a demo, whatever the architecture diagram says.

There is a related trap worth knowing. MIT's work on AI in manufacturing found productivity often *drops* first — in some cases sharply — and takes years to recover, because tools get bolted onto legacy workflows instead of triggering the redesign that would let the gains compound. Adding AI to a broken process usually produces a faster broken process.

---

## Cost and Trade-off

**What this costs:** roughly one to three days per feature — half a day writing the spec, one to two days actually testing the lower rung with real data. That's real time you spend before anyone sees anything.

**When you would skip it:** prototypes meant to be thrown away; internal tools where you are the only user; genuine exploration where the question *is* "can a model do this at all?" Below a certain blast radius, the process costs more than the mistake.

**The honest failure mode of this file:** spec paralysis. A one-page spec is a filter, not a deliverable. If you've spent a week on it, you've turned a cheap filter into an expensive one — go build the 20-example eval set from note 3 and let reality answer the open questions.

---

## Your Turn

Take a feature idea you like. Do this in order, and do not skip step 2:

1. Write the one-page spec. Time-box it to 40 minutes.
2. **Actually implement rung 0b** — a rule or lookup for the most common input pattern. Run it on 50 real examples. Write down the accuracy.
3. Now write the kill criteria, using that number as the baseline.
4. Ask yourself the question that matters: *given the rule got X%, what does the model have to hit to be worth $0.002 and 900 ms per call?*

Then predict, before you build anything: **what fraction of your inputs do you think the rule alone will handle?** Write your guess down. Check it against step 2. The size of that gap is the size of the gap in your intuition, and it is the most useful thing you will learn this week.

---

## Quick Summary

| Idea | In one line |
|---|---|
| Rung 0a | if a `SELECT` answers it, an LLM is a slow wrong `SELECT` |
| Rung 0b | rules for the head of the distribution, model for the tail |
| Cost of being wrong | the consequence, not the accuracy, sets the automation level |
| A better model | does not let you automate a higher-consequence action |
| Baseline | a target without one is meaningless |
| Kill criteria | must be written before you start, or they're negotiable |
| The 5% | real data, real workflow — not a smarter model |
| Bolted-on AI | a faster broken process is still a broken process |
| Spec paralysis | the one-pager is a filter, not a deliverable |

## Next

[3_evals_as_the_product_spec.md](3_evals_as_the_product_spec.md) — question two. Line 4 of the spec said "eval set size: ____". That line is the whole requirements document, and this is how you write it.

## Sources

- [Why 95% of enterprise AI pilots fail — and what the 5% do differently (2026)](https://ibl.ai/blog/why-enterprise-ai-pilots-fail-2026)
- [MIT says 95% of GenAI pilots fail — how to beat the odds](https://www.dataiku.com/blog/moving-past-genai-pilots)
- [6 hard truths behind MIT's finding that 95% of AI pilots fail](https://www.cloudfactory.com/blog/6-hard-truths-behind-mits-ai-finding)
- [AI project failure statistics 2026](https://www.pertamapartners.com/insights/ai-project-failure-statistics-2026)
