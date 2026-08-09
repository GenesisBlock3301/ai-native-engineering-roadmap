# Evals Are the Product Spec

Here is a requirement from a real product doc:

> *"The assistant should produce a high-quality, helpful summary of the support thread."*

Read it as an engineer. What test fails when this breaks? There isn't one. Nobody can implement it, nobody can verify it, and six weeks later two people will disagree about whether it shipped.

Now here is the same requirement written as an eval:

> *200 real support threads. For each, a known-good summary written by a support lead. A change ships only if ≥ 85% of the 200 pass the judge, no slice drops more than 3 points, p95 latency stays under 2.5 s, and cost per summary stays under $0.004.*

Same intent. One of them is a wish. The other is a specification, a test suite, and a release gate at the same time.

**That is the core idea of this file: for an AI feature, the eval set is not a testing artefact. It is the requirements document.** Everything else is prose about it.

---

## Why "Just Test It" Doesn't Work Here

You already know how to test a function: given input X, assert output equals Y. That breaks immediately here, for three reasons.

| Problem | What it breaks |
|---|---|
| **Many right answers** | Two good summaries share almost no tokens. `assert output == expected` fails on a perfect answer. |
| **Non-deterministic** | Same input, run twice, different text. Even at `temperature = 0`, batching and hardware make outputs drift. |
| **Failures are silent** | A wrong answer looks exactly like a right one. There is no stack trace, no 500, no red test. |

That third row is the dangerous one. Every other system you have built tells you when it's broken. This one does not — which is why you have to go and look, on purpose, on a schedule.

---

## Building the Set: Start at 20, Not 200

The number you'll see quoted is around **200 curated examples** for an offline eval set. That's a reasonable steady state. It is a terrible starting point, because you don't yet know what you're measuring.

Do this instead:

**Step 1 — write 20 by hand, before the feature exists.**

Not generated. Written, by you or by whoever does the task today. This is the step people skip and it is the highest-value hour in the whole phase, because writing example 7 is where you discover your requirement was vague. You will hit a thread with two unrelated issues in it and realise nobody ever decided whether the summary covers both.

That discovery is worth more than the eval set.

**Step 2 — run the feature, then read 50 real outputs.**

Read them. All fifty. Write one short note per failure in plain language — not a category, just what went wrong: *"invented a refund amount"*, *"summarised the customer's tone, not the problem"*, *"cut off mid-sentence"*.

**Step 3 — group the notes into failure modes.**

Now you have categories, and they came from data instead of imagination. Typically 5–8 real ones. Now grow the set to ~200 with those categories in mind, making sure each failure mode has 10+ examples.

**Step 4 — split it.**

Same discipline as [2_ai_foundations](../2_ai_foundations/README.md):

```
dev set   (~150)  — you look at these constantly, tune against them
held-out  (~50)   — you touch these rarely, to check the dev score is real
```

If you tune prompts against all 200 and quote all 200 as your score, you have peeked at the test set. The number goes up and the product doesn't. This rule did not stop applying because the test set is now prose.

---

## Three Ways to Score, Cheapest First

Same principle as the ladder in note 2: use the cheapest scorer that actually catches the failure.

| Scorer | Cost per example | Catches | Use for |
|---|---|---|---|
| **Code assertion** | ~$0 , ~1 ms | valid JSON, required field present, length, no PII pattern, cited ID exists in retrieved docs | anything checkable — always start here |
| **LLM-as-judge** | ~$0.001–0.01, ~1 s | "does this answer the question", "is it grounded in the source", "is the tone right" | the open-ended part, at scale |
| **Human review** | ~$1–5, minutes | everything, including what you forgot to ask about | validating the judge, and the tail you don't trust |

Most teams jump straight to LLM-as-judge for everything. That's expensive and, worse, it hides the easy failures inside a fuzzy score. A "did the model cite a document ID that actually exists in the retrieved set?" assertion costs nothing and catches a large share of RAG hallucinations outright.

---

## LLM-as-Judge: Use It, But Know Where It Lies

An LLM judge is a model scoring another model's output. It's the only way to score subjective quality at volume. It also has specific, well-documented biases:

- **Verbosity bias** — longer answers score higher, whether or not they're better.
- **Position bias** — in an A/B comparison, the option shown first wins more often. Fix: run both orders, keep only agreeing verdicts.
- **Self-preference** — a judge tends to prefer text from its own model family.
- **Judge drift** — you upgrade the judge model and every historical score shifts. Your "improvement" is a judge change.

The rule that makes a judge trustworthy: **validate the judge against humans before you trust it.** Take 50 examples, have a human label them pass/fail, run the judge, and measure agreement. Below about 80% agreement, your judge is measuring something other than what you care about — usually because your rubric is vague. Fix the rubric, don't fix the number.

And treat the judge model and its prompt as **pinned, versioned dependencies**. Changing the judge is a change to your measuring instrument. Re-baseline when you do it, and say so.

---

## Offline vs Online

You need both, and they answer different questions.

```
OFFLINE  (~200 curated examples, run on every change)
   "Did this change make the feature better or worse?"
   Runs in CI. Fast, cheap, repeatable. Same inputs every time.

ONLINE   (sample of real production traffic, running continuously)
   "Is it still working, right now, for real users?"
   LLM judge on 1-5% of traces + real user signals.
```

Offline catches regressions before users see them. Online catches the thing offline structurally cannot: **your users' inputs drifting away from your eval set.** That's the same drift concept from [6_production_ai](../6_production_ai/README.md), applied to the eval set instead of the model.

The best online signals are usually free and behavioural, not asked-for:

| Signal | What it tells you |
|---|---|
| edit rate | % of AI outputs the user changed before using — the single best quality proxy |
| acceptance rate | % accepted with no edit |
| retry rate | user re-ran the same request — the answer was useless |
| escalation rate | user went to a human anyway |
| abandonment | user left mid-flow |

Edit rate is worth more than a thumbs-up button, because everyone edits and almost nobody clicks thumbs-down.

---

## Track Three Numbers, and Alert on the Delta

Minimum viable eval dashboard:

1. **Task success** — % passing your rubric
2. **p95 latency** — the slow tail users actually feel, not the mean
3. **Cost per task** — dollars, from note 5

Then the rule that matters: **alert on regression, not on an absolute threshold.**

"Alert if success < 85%" is a bad alarm. If you launched at 91% and drifted to 86%, something broke and nobody was told. "Alert if success dropped more than 3 points from the last release" catches it on day one.

### The failure this hides: averages lie

Here is the one to see for yourself. You change a prompt:

```
Overall task success:   84%  →  87%    ✅ ship it
```

Break it down by slice:

```
slice                    before   after
short threads (n=120)      88%  →  94%   ▲ +6
long threads  (n=50)       82%  →  81%   ▼ -1
non-English   (n=30)       71%  →  58%   ▼ -13   ← shipped, unnoticed
```

The average went up. You broke non-English users by 13 points, and 30 examples out of 200 can't move the headline number enough to notice.

**Always score by slice.** Pick your slices from the things that actually vary: language, input length, customer tier, document type, new vs returning user. A change ships only if no slice regresses beyond your tolerance — that's the "no slice drops more than 3 points" clause in the spec at the top of this file.

---

## Why This Makes You Faster, Not Slower

The natural objection is that this is all overhead. The reported effect is the opposite: teams with real eval suites ship substantially more model and prompt versions per quarter — one 2026 guide puts it at roughly **5×** — than teams without.

The mechanism is obvious once you've felt it. Without evals, every prompt change is a gamble, so changes get batched, reviewed by committee, and shipped monthly. With evals, a change is a pull request with a number attached, and you ship on Tuesday afternoon. The eval set isn't the brake. It's what lets you take your hands off the wall.

The 2026 shorthand for a ship gate: **a versioned eval set, a numerical score, and a regression alarm.** If a change to a prompt can reach production without passing those three, prompts are the least-tested code in your system — and they are code, as [5_data_engineering_infra](../5_data_engineering_infra/README.md) already argued about CI.

---

## Cost and Trade-off

**Money.** 200 examples × an LLM judge at ~$0.005 = about $1 per full run. Run it on every PR, 30 PRs a month: ~$30/month. That's nothing. But if you run a 2,000-example set with a frontier judge on every commit, it's real money and slow CI — which is why you keep the big set for nightly and a 50-example smoke set for PRs.

**Time.** The expensive part is human labelling: 200 examples at ~2 minutes each is about 7 hours, once, plus ongoing curation. Budget it honestly.

**Maintenance.** An eval set rots. Your product changes, your users change, and a set you wrote in January quietly stops representing March. Re-sample from production quarterly.

**When you would not do this:** a prototype with no users. A feature where a code assertion covers 100% of what "correct" means — then you don't need a judge, you need a unit test. And if your feature has fewer than ~50 real inputs in existence, you don't have an eval set problem, you have a "does anyone want this" problem from note 2.

---

## Your Turn

For the feature you specced in note 2:

1. Write **20 eval examples by hand.** Time-box it to one hour. Write down, in one sentence, the requirement ambiguity you discovered while doing it — there will be at least one.
2. Write **one code assertion** that catches a real failure with no model call. Run it.
3. Write an **LLM-as-judge rubric**, then validate it: label 20 examples yourself, run the judge, compute agreement. **Predict the agreement rate before you look.** If you're below 80%, rewrite the rubric — not the threshold.
4. Break your score down by one slice (language, or input length). Look for a subgroup the average is hiding.
5. Now break it on purpose: make the prompt worse in a way that improves the average and hurts one slice. Watch your own dashboard fail to notice.

Step 5 is the one that will stay with you.

---

## Quick Summary

| Idea | In one line |
|---|---|
| Eval set | *is* the requirements doc, not a testing artefact |
| Why normal tests fail | many right answers, non-deterministic, silent failures |
| Start at 20 | writing examples by hand is where vague requirements die |
| Grow to ~200 | from real failure modes you read, not categories you imagined |
| dev / held-out | the train/test rule didn't stop applying because the data is prose |
| Cheapest scorer first | code assertion → LLM judge → human |
| Judge biases | verbosity, position, self-preference, drift |
| Validate the judge | ≥80% agreement with human labels, or fix the rubric |
| Offline vs online | "did this change break it" vs "is it working right now" |
| Best online signal | edit rate — everyone edits, nobody clicks thumbs-down |
| Alert on | regression from last release, not an absolute floor |
| Score by slice | the average hides the subgroup you just broke |
| Ship gate | versioned eval set + a score + a regression alarm |

## Next

[4_designing_for_probabilistic_output.md](4_designing_for_probabilistic_output.md) — your eval says 87% pass. This is about the other 13%, and the fact that they reach a human being with a screen in front of them.

## Sources

- [AI evals for product managers — the complete guide for 2026](https://www.lovelaice.com/resources/ai-evals-for-product-managers-complete-guide-2026)
- [Best AI evaluation tools for product managers in 2026 — eval stack guide](https://www.institutepm.com/knowledge-hub/best-ai-evaluation-tools-2026)
- [AI evals engineer — career guide for 2026's newest discipline](https://jobsbyculture.com/blog/ai-evals-engineer-career-guide-2026)
- [AI deployment in 2026 — CI/CD for LLMs and agents](https://www.harness.io/blog/ai-deployment-in-production-orchestrate-llms-rag-agents)
