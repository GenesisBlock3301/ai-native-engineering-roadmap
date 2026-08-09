# Product Engineering (Weeks 39–42)

Deciding what to build → Evals as the spec → Designing for wrong answers → Cost per user & pricing → Staged rollout.

By now you can build the thing ([3_modern_ai](../3_modern_ai/README.md), [4_applied_ai](../4_applied_ai/README.md)), feed it ([5_data_engineering_infra](../5_data_engineering_infra/README.md)), and run it ([6_production_ai](../6_production_ai/README.md)). Every one of those phases answers *how*. None of them answers the question that decides whether the work mattered:

> **Should this exist, and does it pay for itself?**

That gap is not theoretical. MIT's 2026 study of enterprise GenAI found **95% of pilots delivered no measurable ROI** — out of $30–40 billion spent. The failures were mostly *not* technical. They were unclear definitions of success, no connection to a real workflow, and no honest measurement. Those are product-engineering failures, made by teams that could all ship a working demo.

This phase is the difference between an engineer who builds what they were told and one who decides what is worth building. In 2026 the second one is the job title people are hiring for.

## What's Here

**Notes** — read these in order:

| File | Covers |
| --- | --- |
| [1_what_is_a_product_engineer.md](1_what_is_a_product_engineer.md) | the role, why it appeared, the demo-to-product gap, the four questions that separate the two |
| [2_deciding_what_to_build.md](2_deciding_what_to_build.md) | the "do we need AI at all?" ladder, cost of being wrong, the one-page AI feature spec, kill criteria |
| [3_evals_as_the_product_spec.md](3_evals_as_the_product_spec.md) | why the eval set *is* the requirements doc, building 200 examples, offline vs online, LLM-as-judge and its failure modes, regression gates |
| [4_designing_for_probabilistic_output.md](4_designing_for_probabilistic_output.md) | designing for the wrong answer, the six trust patterns, confidence thresholds, autonomy dial, what to do when the model has no answer |
| [5_unit_economics_and_pricing.md](5_unit_economics_and_pricing.md) | cost per user from token math, gross margin reality in 2026, caching and routing, per-seat vs usage vs outcome pricing |
| [6_shipping_and_practice.md](6_shipping_and_practice.md) | shadow mode → staged rollout → guardrails → kill switch, the week-by-week plan, 24 self-check questions, the mini-project |

**Code** — one notebook, and it is a spreadsheet you can argue with, not a model:

| Notebook | What you build | Needs |
| --- | --- | --- |
| [code/1_ai_feature_unit_economics.ipynb](code/1_ai_feature_unit_economics.ipynb) | a real cost-per-user model for an AI feature: token math → monthly COGS → gross margin → the price you'd have to charge, plus caching/routing/context-growth scenarios and the point where per-seat pricing goes underwater | NumPy, matplotlib |

### Setup

```bash
pip install -r ../requirements.txt
```

Nothing new to install — the notebook is arithmetic, on purpose. The hardest number in production AI is not a gradient, it's the second decimal place of your cost per user.

## What to Learn Deeply

**The demo-to-product gap** (`1_what_is_a_product_engineer.md`)
- *What problem does it solve?* A demo proves the model *can* do the task once. A product has to do it for strangers, on inputs you never saw, at a price that works, and fail in a way nobody sues you over. Almost all the remaining work lives in that gap, and almost none of it is model work.
- *How does it work conceptually?* You stop asking "does it work?" and start asking four measurable questions: what fraction of real tasks succeed, what does one task cost, what happens on the failures, and would anyone pay more than that cost.
- *When should I use it?* Before you write the feature, and again before you ship it. It's cheap at the start and expensive at the end.

**Deciding what to build** (`2_deciding_what_to_build.md`)
- **Extend the ladder** — [1_introduction](../1_introduction/README.md) has a ladder from zero-shot up to fine-tuning, where each rung costs more. Product engineering adds two rungs *below* the bottom: a database query, and a rule. If `SELECT` answers it, an LLM is a slow, expensive, non-deterministic way to be wrong sometimes.
- **Cost of being wrong** — the single number that decides the whole design. A wrong tag on a support ticket costs a click. A wrong dosage costs a life. That number, not model quality, sets how much automation you're allowed.
- **Kill criteria, written before you start** — the eval score and cost per task at which you stop. Written after, they become negotiable, and the 95% is mostly projects nobody was allowed to kill.

**Evals as the product spec** (`3_evals_as_the_product_spec.md`)
- *What problem does it solve?* "It looks good in my three test prompts" is not a specification, and neither is a PRD paragraph saying "the summary should be high quality." An eval set is the only version of the requirement a machine can check on every commit.
- *How does it work conceptually?* Curate ~200 real examples with known-good outcomes, score every change against them offline, and sample live traffic with an LLM judge online. Track three numbers minimum: task success, latency, cost per task. Alert on **regression**, not on an absolute threshold.
- *When should I use it?* From the first prototype. Teams that keep real evals are reported to ship roughly **5× more model versions per quarter** — the eval set isn't the brake, it's the thing that lets you take your hands off the wall.
- **The rule you already know** — [2_ai_foundations](../2_ai_foundations/README.md) taught you not to peek at the test set. That rule doesn't change because the test set is now prompts. Tune on your dev evals, keep a held-out set you touch rarely, or your number stops meaning anything.

**Designing for probabilistic output** (`4_designing_for_probabilistic_output.md`)
- *What problem does it solve?* Your UI was designed for functions that return the right answer. A model returns a *distribution*. Reported experience in 2026 is that most AI feature failures are design failures, not model failures — the model was 90% right and the interface offered no way to catch the other 10%.
- *How does it work conceptually?* Six patterns keep showing up: **intent preview** (show what you're about to do), **autonomy dial** (suggest → confirm → act), **explainable rationale** (cite the source), **confidence signal**, **action audit & undo**, **escalation path** to a human.
- *When should I use it?* Every AI feature needs undo and escalation. The rest scale with the cost of being wrong from note 2.

**Unit economics & pricing** (`5_unit_economics_and_pricing.md`)
- **The margin problem** — classic SaaS runs 80–90% gross margin. AI-first companies in 2026 sit around **50–60%**, because every request burns compute. Survey data puts the 2026 average near **52%**, up from 41% in 2024 — improving, still nowhere near SaaS.
- **Cost per user, from tokens** — you already have the pieces from [3_modern_ai](../3_modern_ai/README.md): tokens are the bill, context length drives both memory and price, thinking tokens are output tokens. Multiply by requests per user per month and you have COGS.
- **The two big levers** — prompt caching (roughly **90% off** cached input tokens at both Anthropic and OpenAI) and model routing (cheap model first, escalate only on hard cases). Together these are reported to cut production bills **40–70%**.
- **Pricing shape** — flat per-seat breaks the moment usage is unbounded. The 2026 pattern that works is a seat or platform floor **plus** metered usage above a generous included allowance.

**Shipping** (`6_shipping_and_practice.md`)
- **Shadow mode** — run the feature on real traffic, show nobody, compare to the current behaviour. Free evidence, zero blast radius.
- **Staged rollout** — 1% → 5% → 25% → 100%, with guardrail metrics that auto-roll-back on regression.
- **Kill switch** — a flag that instantly reverts to the old path. If reverting requires a deploy, you don't have one.
- **Fallback** — what the user sees when the model times out, refuses, or returns garbage: a cached answer, a cheaper model, or an honest "I don't know" with a route to a human. Design this *before* launch; it is the path your worst day runs through.

## How to Study This Phase

Follow the [Learning Loop](../README.md#the-learning-loop), with one change: in this phase the "build something small" step is **not** code. It's a decision document you'd be willing to defend in a review.

Concretely: pick one AI feature — ideally one you actually want in [8_portfolio](../8_portfolio/README.md) — and for it, in order:

1. Write the one-page spec from note 2, including the kill criteria.
2. Write 20 eval examples by hand before writing any feature code. Twenty is enough to learn that your requirement was vague.
3. Run the notebook with *your* numbers and find the price you'd have to charge.
4. Design the failure path before the happy path.
5. Write the rollout plan, including what makes you turn it off.

The trap in this phase is the opposite of [3_modern_ai](../3_modern_ai/README.md). There, the risk was collecting vocabulary. Here, the risk is collecting *opinions* — frameworks and pricing takes that sound wise and predict nothing. The test is whether your numbers survive contact with a real feature. If your cost-per-user estimate is out by 10×, you didn't have a view, you had a vibe.

## Next

[8_portfolio](../8_portfolio/README.md) — every project there now starts with this phase's one-page spec and eval set, not with the architecture diagram. "Business Problem" is the first box in that loop for a reason; this phase is how you fill it in.

## Sources

- [MIT / enterprise GenAI pilot failure analysis — why 95% fail and what the 5% do differently](https://ibl.ai/blog/why-enterprise-ai-pilots-fail-2026)
- [Forbes — MIT finds 95% of GenAI pilots fail because companies avoid friction](https://www.forbes.com/sites/jasonsnyder/2025/08/26/mit-finds-95-of-genai-pilots-fail-because-companies-avoid-friction/)
- [AI evals for product managers — the complete guide for 2026](https://www.lovelaice.com/resources/ai-evals-for-product-managers-complete-guide-2026)
- [AI UX design patterns — designing interfaces for AI-powered features](https://www.institutepm.com/knowledge-hub/ai-ux-design-patterns)
- [AI unit economics — pricing and margins framework, 2026](https://www.digitalapplied.com/blog/ai-unit-economics-pricing-margins-services-2026-framework)
- [The AI COGS problem — SaaS gross margin compression in 2026](https://www.saasmag.com/ai-cogs-saas-gross-margin-compression/)
- [AI pricing models 2026 — usage, seats, outcomes and margins](https://www.tldl.io/resources/ai-business-models-pricing)
- [The AI-native jobs guide — roles that barely existed three years ago](https://www.landed.jobs/articles/ai-native-jobs-2026-field-guide)
