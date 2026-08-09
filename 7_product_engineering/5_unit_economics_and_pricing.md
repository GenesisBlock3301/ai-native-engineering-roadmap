# Cost Per User, Margin, and Price

Question four: **does it pay for itself?**

Start with the number that reframes everything. Classic SaaS runs at **80–90% gross margin** — you write the software once and the marginal cost of one more user is nearly zero.

AI-first companies in 2026 do not get that. Reported gross margins cluster at **50–60%**. One 2026 survey put the average at **52%**, up from 41% in 2024 — improving, still 30 points below what SaaS investors were trained to expect. And **84% of SaaS companies report having lost margin to AI costs.**

The reason is structural, not a pricing mistake: every request burns compute. Your cost now scales with *usage*, while your revenue often scales with *headcount*. Those two lines diverge, and the gap is your margin.

Practice for this file: [code/1_ai_feature_unit_economics.ipynb](code/1_ai_feature_unit_economics.ipynb) — you build this whole model with your own numbers.

---

## Cost Per User, From Tokens

You already have every piece of this from [3_modern_ai](../3_modern_ai/README.md). Tokens are the bill. Context length costs money. Thinking tokens are output tokens.

```
cost per task  = (input tokens  × input price)
               + (output tokens × output price)

cost per user  = cost per task × tasks per user per month
```

**Worked example — a support-thread summariser.**

Illustrative 2026 rates (check current pricing before you quote these to anyone):

| Tier | Input / 1M tokens | Output / 1M tokens |
|---|---|---|
| Frontier | $3.00 | $15.00 |
| Mid | $0.80 | $4.00 |
| Small | $0.10 | $0.40 |

One summary: 3,000 input tokens (thread + system prompt + 2 few-shot examples), 250 output tokens.

```
input :  3,000 / 1,000,000 × $3.00  = $0.00900
output:    250 / 1,000,000 × $15.00 = $0.00375
                                      ─────────
cost per summary                      $0.01275
```

An average user does 40 a month:

```
$0.01275 × 40  =  $0.51 per user per month
```

That looks harmless. Two things are about to go wrong with it.

---

## Trap 1: Conversation History Grows Faster Than You Think

Predict before you read on:

> A chat feature. Each turn: ~300 tokens from the user, ~400 tokens back. A conversation runs 20 turns.
> **How many input tokens does the whole conversation cost?**

Most people answer around 10,000 — 500 tokens × 20 turns.

The actual answer is **143,000**, roughly 14× that.

Here's why. The model has no memory ([1_introduction](../1_introduction/README.md) — everything it knows is in front of it). So every turn resends the entire history:

```
turn 1  input:    500
turn 2  input:  1,200   (500 + 700 of turn 1)
turn 3  input:  1,900
...
turn 20 input: 13,800

total input: 143,000 tokens
total output:  8,000 tokens
```

Cost at frontier rates:

```
naive estimate:  10,000 in + 8,000 out  =  $0.15
actual        : 143,000 in + 8,000 out  =  $0.55     ← 3.7× the estimate
```

Input grows with the *square* of the turn count, not linearly. This single effect is the most common reason a cost estimate is out by 3–10× — and you find out in the invoice, not in the demo, because your demo was three turns long.

It also puts a number on something [3_llm_inference.md](../3_modern_ai/3_llm_inference.md) told you qualitatively: context length is a cost driver, not just a memory constraint. Trimming or summarising history is a margin decision, not a nicety.

---

## The Two Big Levers

### Prompt caching — the highest-leverage thing on this page

Both Anthropic and OpenAI offer roughly **90% off cached input tokens**. Anything stable at the front of your prompt — system prompt, few-shot examples, a retrieved document, or the conversation prefix — can be cached and re-read cheaply.

Apply it to the 20-turn conversation above. Each turn's prefix was already sent last turn, so most input is cacheable:

```
uncached input:  13,800 tokens × $3.00/M   = $0.041
cached input  : 129,200 tokens × $0.30/M   = $0.039
output        :   8,000 tokens × $15.00/M  = $0.120
                                             ───────
                                              $0.200   vs $0.549 uncached
```

**64% cheaper**, no quality change, and it usually cuts time-to-first-token too. This is why caching is the first thing to reach for, not the last.

The condition: your prompt prefix must be **stable and identical**. Putting a timestamp, a session ID, or a shuffled retrieval order at the top of your prompt silently destroys every cache hit — the bill snaps straight back to $0.549, **2.7× higher**, and the output is byte-for-byte the same. That's a real bug people ship, and nothing on any dashboard looks wrong.

### Model routing — cheap model first, escalate on hard cases

Same principle as the ladder in note 2, applied per request. Send everything to the small model; escalate only what it can't handle.

Say 70% of tasks are easy. The same summary on the small tier costs `3,000/1M × $0.10 + 250/1M × $0.40 = $0.00040`:

```
70% × $0.00040 (small)    = $0.00028
30% × $0.01275 (frontier) = $0.00383
                            ─────────
blended cost per task       $0.00411   vs $0.01275 all-frontier  → 68% cheaper
```

The catch, and it's the whole game: **you need a router, and the router can be wrong.** Route a hard task to the small model and you get a bad answer at a 90% discount. That is not a saving.

Which is why routing is a *note 3* decision, not a note 5 decision. You need the eval set to answer: at what confidence do I escalate, and what does the blended eval score look like versus all-frontier? A router that saves 63% and costs 6 points of task success is a trade you may or may not want — but you can only see it if you measure both numbers together.

Together, caching and routing are commonly reported to cut production bills **40–70%**. Two more worth knowing: a **batch API** (~50% off) for anything not user-facing — re-indexing, nightly evals, backfills — and simply **capping output length**, since output tokens usually cost 4–5× input tokens.

---

## Trap 2: The LLM Bill Is Not Your COGS

People model the token cost, get $0.51, and stop. The full picture for that summariser:

| Line item | $/user/month | Where it comes from |
|---|---|---|
| LLM inference | 0.51 | the math above |
| Embeddings + re-indexing | 0.12 | [5_data_engineering_infra](../5_data_engineering_infra/README.md) — the pipeline reruns |
| Vector DB hosting | 0.35 | [4_applied_ai](../4_applied_ai/README.md) — storage + queries |
| Observability / trace storage | 0.18 | [6_production_ai](../6_production_ai/README.md) — you log every trace |
| Eval runs (amortised) | 0.04 | note 3 — CI runs cost money too |
| Human escalations | 0.50 | note 4 — 2% of 40 tasks escalate × 1.5 min × $25/hr |
| **Total COGS** | **$1.70** | |

The LLM is **30% of the total.** Optimising it to zero still leaves you at $1.19. This is the number people miss when they say "inference is getting cheaper, margins will fix themselves."

Note the last line especially: your escalation rate from note 4 is a **cost line**, which means design quality shows up directly in gross margin. That's not a metaphor — halve your escalation rate and you've cut COGS by 15%.

At a $20/seat price, $1.70 COGS is a 91.5% margin. Comfortable. Which brings us to the thing that actually breaks.

---

## Why Flat Per-Seat Pricing Breaks

Usage is never uniform. Split the $1.70 above into what's fixed per user (vector DB, amortised evals: **$0.39**) and what scales per task (LLM $0.01275 + embeddings, traces and escalations $0.02 = **$0.03275**), then look at four cohorts:

| User type | Summaries/mo | LLM cost | Total COGS | Margin at $20/seat |
|---|---|---|---|---|
| Light (60% of users) | 8 | $0.10 | $0.65 | 97% |
| Average (35%) | 40 | $0.51 | $1.70 | 92% |
| Heavy (4%) | 400 | $5.10 | $13.49 | 33% |
| Power (1%) | 2,000 | $25.50 | $65.89 | **−229%** |

Break-even is at **599 summaries a month**. Your best, most engaged customer — the one in your case study — **loses you $46 a month.** And they are the one most likely to renew, expand, and tell their friends.

Under per-seat SaaS, adoption was pure upside. Under AI economics, adoption without metering is how you grow into insolvency. This is the mechanism behind the "84% lost margin" number.

---

## What Pricing Looks Like in 2026

Roughly where the market sits (shares overlap, because hybrids count in more than one bucket):

| Model | Share | Charges for | Breaks when |
|---|---|---|---|
| Subscription / per-seat | ~58% | a login | usage is unbounded — the table above |
| Usage-based | ~35% | tokens, requests, or credits | buyers can't forecast their bill and stall on procurement |
| Outcome-based | ~18% | a resolved ticket, a booked meeting | you can't define "resolved" precisely enough |

Outcome-based pricing went from about **2% in Q2 2025 to 18% in 2026** — the fastest-moving line in the table, and it's what Agentforce, Intercom's Fin, and Now Assist all ship components of.

**The pattern that actually works in 2026:** a **seat or platform floor plus metered usage above a generous included allowance.**

```
$20/user/month  includes 100 summaries
then            $0.05 per summary
```

The buyer gets a predictable bill for normal use. You get protected margin on the 1% who would otherwise cost you $42. Nobody's finance team has to model tokens.

**One thing to notice about outcome-based pricing:** to charge per *resolved* ticket, you must define "resolved" precisely enough to invoice on it. That definition is your eval rubric from note 3. Outcome pricing is only available to teams who did the eval work — which is a good reason to do the eval work.

---

## Cost and Trade-off

**Cheaper is not free.** Every lever here has a quality cost, and the only honest way to price the trade is to run the eval:

| Lever | Saves | Costs you |
|---|---|---|
| Prompt caching | 40–80% of input cost | nothing — but a fragile prefix silently kills it |
| Model routing | 30–60% | task success on misrouted hard cases |
| Trim history | large on long chats | the model forgets things and users repeat themselves |
| Shorter output | output is 4–5× input price | terse answers, more follow-up turns (which cost input) |
| Batch API | ~50% | latency — non-interactive work only |
| Quantization / self-host | can be large at high volume | ops burden, and a real accuracy drop you must measure |

That last column is the point. **A cost optimisation you haven't run through your eval set is not an optimisation — it's an untested change that happens to be cheaper.**

**When to ignore this whole file:** under roughly $500/month of total spend. At that scale your engineering hours cost more than the tokens, and optimising is a hobby. Revisit at $5,000/month, and treat $50,000/month as a full-time concern.

**The estimate that ages worst:** any cost model built before you have real usage data. Your traffic distribution — how many power users, how long conversations really run — moves cost per user more than model choice does. Model it early, but re-measure within a month of launch.

---

## Your Turn

Run [code/1_ai_feature_unit_economics.ipynb](code/1_ai_feature_unit_economics.ipynb) with your own feature's numbers, then:

1. Compute your cost per user per month. **Write your guess down first.** Most people are 3–10× low, and the conversation-growth trap is usually why.
2. Add the non-LLM COGS lines. What fraction of your total is actually the model? If it's over 60%, you've probably forgotten a line.
3. Find your break-even user: at your price, **how many tasks per month makes a user unprofitable?** That number is your usage allowance.
4. Turn on caching in the model. Then break it on purpose: put a timestamp at the front of the prompt and watch the cache hit rate — and the bill — go to the uncached case.
5. Now the decision drill: your CFO wants 20% off next quarter. Rank the six levers by *cost per point of eval score lost*, not by dollars saved. Defend your top choice in three sentences.

Step 5 is the actual job.

---

## Quick Summary

| Idea | In one line |
|---|---|
| SaaS vs AI margin | 80–90% vs 50–60%; 2026 average ≈52%, up from 41% in 2024 |
| Why | cost scales with usage, revenue often scales with headcount |
| Cost per task | (input × input price) + (output × output price) |
| History trap | 20 turns = 143k input tokens, not 10k — 3.7× the naive estimate |
| Prompt caching | ~90% off cached input; 64% off a 20-turn chat |
| Cache killer | a timestamp or session ID at the front of the prompt |
| Model routing | ~68% cheaper at a 70/30 split, but a misroute is a bad answer at a discount |
| Caching + routing | commonly 40–70% off the total bill |
| LLM ≠ COGS | inference was 30% of a $1.70/user total |
| Escalation rate | a COGS line — design quality shows up in gross margin |
| Per-seat breaks | break-even at 599 tasks; the 1% power user cost $65.89 against a $20 seat |
| 2026 pattern | seat floor + metered usage above a generous allowance |
| Outcome pricing | 2% → 18% in a year; requires an eval-grade definition of "resolved" |
| The rule | an optimisation you haven't eval'd is just a cheaper untested change |

## Next

[6_shipping_and_practice.md](6_shipping_and_practice.md) — it works, it fails safely, it pays for itself. Now get it in front of users without betting the product on your first day of real traffic.

## Sources

- [The AI COGS problem — SaaS gross margin compression 2026](https://www.saasmag.com/ai-cogs-saas-gross-margin-compression/)
- [AI unit economics — pricing and margins framework 2026](https://www.digitalapplied.com/blog/ai-unit-economics-pricing-margins-services-2026-framework)
- [Token economics — why AI pricing is quietly breaking SaaS margins](https://pmnorthstar.in/ai-decoded/ai-pricing-token-economics-saas-2026)
- [AI pricing models 2026 — usage, seats, outcomes and margins](https://www.tldl.io/resources/ai-business-models-pricing)
- [AI SaaS pricing — how to price AI features without killing your margins (2026)](https://conception-labs.com/blog/ai-saas-pricing-how-to-price-ai-features-without-killing-your-margins)
- [SaaS usage-based pricing models — decision matrix 2026](https://www.digitalapplied.com/blog/saas-usage-based-pricing-models-decision-matrix-2026)
