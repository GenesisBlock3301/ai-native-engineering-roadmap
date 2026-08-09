# Shipping, Practice & Self-Check

Two halves. First, how the feature actually reaches users without betting the product on your first day of real traffic. Then the drills that turn this phase into something you keep.

---

# Part 1 — Shipping

## The Problem With "Ship It"

A normal deploy has a clean signal. Error rate spikes, you roll back.

An AI feature has no such signal. Quality degrades **silently** (note 3), and the damage is done to trust, which does not show up in a dashboard for weeks. By the time your NPS moves, 40,000 people have read a confidently wrong answer.

So the release process has to manufacture the signal that the system won't give you. Four stages, each one buying evidence more cheaply than the next.

```
1. OFFLINE EVAL   200 examples in CI          → catches regressions, sees no real inputs
        ↓
2. SHADOW MODE    real traffic, hidden output → real inputs, zero blast radius
        ↓
3. STAGED ROLLOUT 1% → 5% → 25% → 100%        → real users, bounded blast radius
        ↓
4. GUARDRAILS     auto-rollback + kill switch → the part that saves you at 2am
```

## Shadow Mode

Run the new feature on real production traffic. Log the output. **Show nobody.** Compare it to whatever happens today.

This is the most underused stage in AI shipping, and it is nearly free evidence. It answers the one thing your eval set structurally cannot: *do my real users' inputs look anything like my 200 curated examples?* They usually don't. Shadow mode is where you discover 30% of real inputs are one-line messages with no context, and your eval set has none of those.

Costs: you pay for the inference twice, and you need somewhere to store traces. Run it for a few days, not a quarter.

## Staged Rollout

Ramp by percentage, with a real gate at each step:

| Stage | Watch for | Typical dwell |
|---|---|---|
| 1% | crashes, timeouts, obviously broken output | 1 day |
| 5% | edit rate, escalation rate, cost per task | 2–3 days |
| 25% | slice regressions (note 3), p95 latency, support tickets | 3–7 days |
| 100% | drift, cost at full volume | forever |

The non-negotiable property: **the flag must be instantly reversible at every stage.** That single requirement is what justifies the whole staged approach — a ramp you can't reverse is just a slow launch.

## Guardrail Metrics and Auto-Rollback

A guardrail metric is one you are *not* trying to improve — you just refuse to let it get worse. Wire them to automatic rollback so a bad release ends without a human in the loop:

| Guardrail | Example trip condition |
|---|---|
| Task success | drops >3 points vs the current version |
| Edit rate | rises >5 points |
| Escalation rate | rises >2 points |
| p95 latency | exceeds the budget from your one-page spec |
| Cost per task | exceeds budget by >20% |
| Refusal / empty-answer rate | rises at all — usually means retrieval broke |

Modern flagging platforms will run sequential statistical tests on these and flag a regression as soon as it's significant, rather than waiting for a fixed sample. Use that if you have it; a threshold and a cron job is fine if you don't.

## Kill Switch and Fallback

**Kill switch:** one flag, instant revert to the previous path. Test it in production on a quiet Tuesday. If reverting requires a deploy, a rebuild, or a cache warm-up, you do not have a kill switch — you have an intention.

**Fallback** is what the user sees when the model times out, refuses, or returns garbage. Decide this before launch, because it is the path your worst day runs through:

```
model fails
   ├─ cached previous answer, if one exists and is still valid
   ├─ a cheaper/simpler model
   ├─ the deterministic old feature (search, template, rule)
   └─ an honest "I couldn't do that" + the escalation path from note 4
```

Never a stack trace, and never a silent empty state — users read an empty state as "the product is broken", which is a worse outcome than "the AI couldn't help."

## Version Everything, Together

When quality moves, you need to know what changed. At minimum, log with every trace:

```
model id + version   prompt version    retrieval index version
eval set version     temperature/params    flag state
```

A prompt change is a code change. It belongs in git, in a PR, with an eval score attached — the same argument [5_data_engineering_infra](../5_data_engineering_infra/README.md) makes about CI. The failure mode this prevents is the worst debugging session in AI products: quality dropped last Thursday and nobody can tell you what shipped, because someone edited a prompt in a web console.

---

# Part 2 — Practice

Reading this phase teaches you nothing on its own. Three kinds of practice, in order of value:

1. **Decide with it** — answer a real design question with numbers you worked out yourself.
2. **Break it** — change one input in the notebook, predict the result, run it, find out where you were wrong.
3. **Explain it** — out loud, no notes, to a person or a rubber duck.

Note the reordering from earlier phases. In [3_modern_ai](../3_modern_ai/6_practice_and_selfcheck.md), explaining came first, because the goal was understanding. Here the goal is judgement, and judgement is only demonstrated by a decision you defend.

## The Drill Rules

**Rule 1 — predict first.** Write the number down before you compute it. The gap between your guess and reality is the entire lesson.

**Rule 2 — every claim gets a number.** "Caching saves a lot" is not knowledge. "Caching cut a 20-turn conversation from $0.55 to $0.20, 64%, because 129k of the 143k input tokens were a repeated prefix" is knowledge.

**Rule 3 — no AI for the explain step.** Use AI to challenge your reasoning, generate the code, and play devil's advocate. Then close it and say the answer alone.

**Rule 4 — use a feature you actually care about.** This phase collapses into consulting-speak if the example is hypothetical. Pick something you want in [8_portfolio](../8_portfolio/README.md).

## Week-by-Week (Weeks 39–42)

| Week | Read | Do | Prove you got it |
|---|---|---|---|
| 39 | [1_what_is_a_product_engineer.md](1_what_is_a_product_engineer.md), [2_deciding_what_to_build.md](2_deciding_what_to_build.md) | write the one-page spec; **implement rung 0b** (a rule) and test it on 50 real examples | argue *against* using an LLM for a feature someone wants one for — and win on data |
| 40 | [3_evals_as_the_product_spec.md](3_evals_as_the_product_spec.md) | write 20 eval examples by hand; one code assertion; one LLM-judge rubric validated against your own labels | state your feature's requirement as a passing threshold on a versioned set, with no adjectives |
| 41 | [5_unit_economics_and_pricing.md](5_unit_economics_and_pricing.md) | [code/1_ai_feature_unit_economics.ipynb](code/1_ai_feature_unit_economics.ipynb) with your numbers | quote cost per user to two decimals, and name your break-even usage level |
| 42 | [4_designing_for_probabilistic_output.md](4_designing_for_probabilistic_output.md), this file | design the failure path and write the rollout plan | the mini-project below, plus the self-check list |

If a week runs long, drop the reading, not the artefact. The artefacts are the phase.

## Self-Check: Answer These With No Notes

**The role and the decision**
1. Why did 95% of enterprise GenAI pilots produce no value — and which of those causes were technical? → [1](1_what_is_a_product_engineer.md)
2. What are the four questions, in order, and why does the order matter? → [1](1_what_is_a_product_engineer.md)
3. What are the two rungs below the bottom of the prompt-engineering ladder? → [2](2_deciding_what_to_build.md)
4. Why does the *cost of being wrong* set your automation level, rather than model accuracy? → [2](2_deciding_what_to_build.md)
5. Why must kill criteria be written before the project starts? → [2](2_deciding_what_to_build.md)
6. What did the successful 5% have in common? → [2](2_deciding_what_to_build.md)

**Evals**
7. Why can't you use `assert output == expected` on an AI feature? Give all three reasons. → [3](3_evals_as_the_product_spec.md)
8. Why start at 20 examples instead of 200? → [3](3_evals_as_the_product_spec.md)
9. Name three LLM-as-judge biases and the fix for one of them. → [3](3_evals_as_the_product_spec.md)
10. How do you know your judge is trustworthy? What number, and what's the floor? → [3](3_evals_as_the_product_spec.md)
11. Why alert on regression instead of an absolute threshold? → [3](3_evals_as_the_product_spec.md)
12. Overall score went 84% → 87%. Why might that be a bad release? → [3](3_evals_as_the_product_spec.md)
13. Why is edit rate a better quality signal than a thumbs-up button? → [3](3_evals_as_the_product_spec.md)

**Design**
14. Name the six trust patterns, and the two that are never optional. → [4](4_designing_for_probabilistic_output.md)
15. Why is a model's stated confidence not a probability, and what would you use instead? → [4](4_designing_for_probabilistic_output.md)
16. A confirm dialog is approved 98% of the time in 1.2 seconds. Is the system safer? Why not? → [4](4_designing_for_probabilistic_output.md)
17. When can you *not* ship at "act" level, regardless of eval score? → [4](4_designing_for_probabilistic_output.md)

**Economics**
18. Why are AI gross margins ~50–60% instead of ~85%? → [5](5_unit_economics_and_pricing.md)
19. A 20-turn chat: 300 tokens in, 400 out per turn. How many input tokens total, and why isn't it 10,000? → [5](5_unit_economics_and_pricing.md)
20. What single line of code can silently destroy your prompt cache hit rate? → [5](5_unit_economics_and_pricing.md)
21. Why is per-seat pricing dangerous when your best customer is your heaviest user? → [5](5_unit_economics_and_pricing.md)
22. What must you already have before you can sell outcome-based pricing? → [5](5_unit_economics_and_pricing.md)

**Shipping**
23. What does shadow mode tell you that an offline eval set structurally cannot? → this file
24. What single property makes a staged rollout worth doing at all? → this file

## Interview Questions (2026 Style)

These are increasingly the *first* questions in an AI engineering loop, not the last ones.

1. "Walk me through how you'd decide whether a feature should use an LLM at all."
2. "How do you know your AI feature works? Be specific about the number and the sample size."
3. "Your eval score improved and users complained. What happened?" *(slice regression, or eval set drifted from real inputs)*
4. "Estimate the monthly cost per user for a RAG chatbot. Talk me through your assumptions." *(they are watching for the conversation-growth trap)*
5. "Your inference bill tripled with no traffic change. Name three causes." *(cache-prefix broken, context/history growth, model or routing change, retry storm)*
6. "How would you price this feature?"
7. "The model is 92% accurate. Should we automate the action?" *(the answer is a question: what does a wrong action cost?)*
8. "How do you roll out a prompt change safely?"
9. "Tell me about an AI feature you decided *not* to build." — the strongest answer in the whole loop, and almost nobody has one.

## Mini-Project for This Phase

**Not** an app. A decision packet for one feature, in five pages or fewer, that you would defend in a design review:

1. **The one-page spec** from note 2, kill criteria included.
2. **The rung-0b result** — your rule, tested on 50 real examples, with its accuracy. Include it even if it embarrasses the AI version.
3. **20 hand-written eval examples**, a code assertion, a judge rubric, and the judge-vs-human agreement rate.
4. **The cost model** — cost per user, full COGS table, break-even usage, and the two levers you'd pull first with their eval cost.
5. **The failure design** — the exact screen for a wrong answer, the fallback chain, and the rollout plan with guardrail trip conditions.

Then the part that makes it real: **give it to someone and have them attack it.** Not review it — attack it. The two questions they should ask are *"where did that number come from?"* and *"what would make you kill this?"* If you can answer both for every page, you have the skill this phase is for.

Keep the packet. It's a stronger portfolio artefact than most demos, because it shows judgement, and demos are cheap now.

## Common Mistakes to Avoid

| Mistake | Why it hurts |
|---|---|
| Building before writing 20 eval examples | you encode a vague requirement into code and discover it three weeks later |
| Quoting a score from the set you tuned on | it's not a measurement, it's an echo |
| Optimising cost without re-running evals | you shipped a cheaper, worse product and called it a win |
| Treating the LLM bill as COGS | it was 30% of the total in note 5's example |
| Adding a confirmation dialog and calling it safety | 1.2-second approvals launder liability, they don't reduce error |
| Shipping without a tested kill switch | an untested revert path is discovered during the incident |
| Editing prompts in a web console | quality moves and nothing in git explains why |
| A "framework" with no numbers in it | this phase's specific failure mode — opinions that predict nothing |

## You're Done With This Phase When…

- You can talk someone **out of** an AI feature, using their own data.
- You can state a product requirement as a threshold on a versioned eval set, with no adjectives in it.
- You can estimate cost per user to two decimal places, in a meeting, without a spreadsheet — and you know which assumption is shakiest.
- You can name the screen a user sees when the model is wrong, before you've written the happy path.
- You have one feature you decided not to build, and you can say exactly which number killed it.

## Next

[8_portfolio](../8_portfolio/README.md) — every project there now opens with this packet, not with an architecture diagram. "Business Problem" is the first box in that loop; this phase is how you fill it in with numbers.

## Sources

- [How to safely ship AI features that can hallucinate](https://www.growthbook.io/insights/safely-ship-ai-features-can-hallucinate)
- [Feature flagging for AI models — how to safely roll out changes](https://www.growthbook.io/insights/feature-flagging-ai-models-safely-roll-out-changes)
- [The complete AI experimentation guide — test, compare, validate and ship safely](https://launchdarkly.com/blog/ai-experimentation/)
- [AI deployment in 2026 — CI/CD for LLMs, RAG and agents](https://www.harness.io/blog/ai-deployment-in-production-orchestrate-llms-rag-agents)
- [The complete AI guardrails implementation guide for 2026](https://www.getmaxim.ai/articles/the-complete-ai-guardrails-implementation-guide-for-2026/)
