# What a Product Engineer Actually Does

You have spent six phases learning to make the model work. This phase is about the other half of the job, and it starts with an uncomfortable number.

In 2026, MIT looked at enterprise generative-AI pilots. **95% of them produced no measurable business value.** Between $30 and $40 billion was spent. Almost none of those failures were "the model wasn't smart enough."

They failed because nobody defined what success meant, the feature never touched a real workflow, or nobody was allowed to turn it off.

Those are not model problems. They are decision problems. This file is about the person whose job is to make those decisions.

---

## The Demo-to-Product Gap

Here is the thing that surprises backend engineers most when they ship their first AI feature.

A normal function:

```
input  →  function  →  correct output       (every time, or it's a bug)
```

A model:

```
input  →  model  →  a distribution over outputs
                    (mostly good, sometimes confidently wrong)
```

You already know this from [3_modern_ai](../3_modern_ai/README.md) — the model predicts a plausible next token, not a looked-up truth. What you may not have internalised is what it does to *shipping*.

With a normal function, "it works on my machine" is 90% of the way to done. With a model, a working demo is maybe 20%. The rest is:

| Question | Demo answer | Product answer |
|---|---|---|
| Does it work? | "Yes, look" (3 prompts) | "84% task success on 200 held-out real cases" |
| What does it cost? | (nobody asked) | "$0.031 per task, $1.86 per user per month" |
| What happens when it's wrong? | (it wasn't, in the demo) | "Falls back to search, user can undo, 2% escalate to a human" |
| Would anyone pay for it? | "It's really cool" | "Saves 4 min/ticket × 900 tickets/mo, at $1.86/user cost" |

**A product engineer is the person who has all four answers.** Not a product manager who can't read the code, and not an engineer who only has column two.

---

## Why This Role Appeared Now

Three things happened at once.

**1. Implementation got cheap.** AI writes the code. That's the premise of this whole roadmap — see the [2026 Learning Pyramid](../README.md#the-2026-learning-pyramid). When writing the code is no longer the bottleneck, *knowing what to write* becomes the bottleneck.

**2. Outputs became probabilistic.** Traditional QA asks "did it return the right value?" You can't ask that of a summary. Someone has to define what "good" means, in a form a machine can check, before the feature exists. That's a new job and it lands between PM and engineer.

**3. Compute became a line item.** A traditional API endpoint costs fractions of a cent and nobody models it. An LLM call costs real money per request, and it scales with usage, not with headcount. Someone has to own that number.

The job market followed. Roles that barely existed in 2023 — **AI engineer**, **evals engineer**, **forward-deployed engineer**, **AI product manager** — are now normal postings. The common thread isn't a framework. It's owning an *outcome* rather than a ticket.

**A caution on the salary numbers you'll see.** Reported 2026 spreads put senior AI engineers around $180K–$250K+ against $150K–$180K for senior software engineers. Treat that as a weak signal, not a plan. It's US-market, self-reported, and it will compress — the same sources expect the two roles to converge as inference costs fall. Learn this phase because the decisions are the durable part, not because of a salary band.

---

## The Four Questions

Everything in this phase reduces to four questions, asked in this order. The order matters — each one can kill the feature before you spend money on the next.

```
1. Should this exist?        →  2_deciding_what_to_build.md
        ↓  (yes)
2. How will I know it works? →  3_evals_as_the_product_spec.md
        ↓  (I have 200 examples and a score)
3. What happens when it's wrong?  →  4_designing_for_probabilistic_output.md
        ↓  (undo, fallback, escalation designed)
4. Does it pay for itself?   →  5_unit_economics_and_pricing.md
        ↓  (cost per user < what someone will pay)
                             →  6_shipping_and_practice.md
```

Most teams do these in the order 1 → build → ship → 4 → panic. The cost of that ordering is that you discover the economics after the architecture is fixed, when the only lever left is "use a smaller model and hope."

---

## What This Is Not

**Not product management.** A PM decides which problems the company works on. A product engineer decides whether an AI approach can actually solve one, at what quality, at what cost — and then builds it. You need to be able to read the trace and read the P&L line.

**Not "prompt engineering with a business hat."** Note 2's first move is often *don't use a model here*. A product engineer is comfortable killing the AI feature, which is exactly why their surviving AI features work.

**Not a replacement for the previous six phases.** You cannot estimate cost per user without the token math from [3_modern_ai](../3_modern_ai/README.md). You cannot design an eval set without the train/validation/test discipline from [2_ai_foundations](../2_ai_foundations/README.md). You cannot design a fallback without knowing what a timeout in a RAG pipeline actually looks like ([4_applied_ai](../4_applied_ai/README.md)). This phase is the layer on top; it collapses without the layers underneath.

---

## Cost and Trade-off

This discipline is not free.

**It costs time up front.** Writing 200 eval examples and a cost model before you write the feature feels slow, and for a two-day throwaway prototype it *is* slow. Skip it there.

**It creates friction with people who want the demo now.** "Why can't we just ship it?" is a real conversation you will have. The honest answer is that you can — and the 95% number is what shipping-without-this looks like at scale.

**When would you not do this?** Internal tools with fewer than ~20 users, where you are the fallback and the cost is rounding error. Genuine research spikes where the point is to learn whether something is possible at all. One-off scripts. Below a certain blast radius, ceremony costs more than it saves.

Above that line — anything touching customers, money, or a workflow someone depends on — skipping it is how you join the 95%.

---

## Your Turn

Take one AI feature you have already built or seen. Answer the four questions **out loud, with numbers**, no notes:

1. Should it exist? What is the non-AI version, and why is it worse?
2. How would you know it works? Name the metric and the size of the test set.
3. What happens when it's wrong? Describe the exact screen the user sees.
4. Does it pay for itself? Give a cost per user per month, to two decimal places.

If you can't do (4), you're not stuck on business skills — you're missing token math, and the fix is [3_llm_inference.md](../3_modern_ai/3_llm_inference.md) plus this phase's notebook.

---

## Quick Summary

| Idea | In one line |
|---|---|
| The 95% | most enterprise GenAI pilots produce no measurable value |
| Why they fail | unclear success criteria, no real workflow, no honest measurement — not model quality |
| Demo → product | a working demo is ~20% of the work |
| Product engineer | owns all four columns: works / costs / fails / pays |
| Why now | code got cheap, outputs got probabilistic, compute became a line item |
| The four questions | should it exist → how will I know → what if it's wrong → does it pay |
| Wrong order | build → ship → economics → panic |
| When to skip | <20 internal users, research spikes, throwaway scripts |

## Next

[2_deciding_what_to_build.md](2_deciding_what_to_build.md) — question one, and the two rungs that sit *below* the bottom of the prompt-engineering ladder.

## Sources

- [Why 95% of enterprise AI pilots fail — and what the 5% do differently (2026)](https://ibl.ai/blog/why-enterprise-ai-pilots-fail-2026)
- [Forbes — MIT finds 95% of GenAI pilots fail because companies avoid friction](https://www.forbes.com/sites/jasonsnyder/2025/08/26/mit-finds-95-of-genai-pilots-fail-because-companies-avoid-friction/)
- [The AI-native jobs guide — 12 roles that barely existed 3 years ago](https://www.landed.jobs/articles/ai-native-jobs-2026-field-guide)
- [AI engineering vs software engineering in 2026 — an honest breakdown](https://adilshamim8.medium.com/ai-engineering-vs-software-engineering-in-2026-the-complete-honest-breakdown-0a046ec200c6)
- [AI engineer vs software engineer — career paths, skills and salary (2026)](https://myengineeringpath.dev/genai-engineer/ai-vs-software-engineer/)
