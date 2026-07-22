# Prompt Engineering Techniques

`1_llm_setting.md` is about **how** the model picks its next words (temperature, top_p, and so on). This file is about **what you ask it to do first**. Same model. Same settings. But if you ask in a smarter way, you get a much better answer.

Every trick below costs something — more time, more money, or more setup work. Your job is to find the cheapest trick that still gets the job done.

---

## Zero-Shot Prompting

Just ask. No examples.

**Prompt:** `Classify this review as positive or negative: "The battery life is terrible."`
**Output:** → Negative

* **Problem it fixes:** it's the fastest way to ask for something.
* **When it works:** the task is common and easy — sentiment, translation, simple questions.
* **When it fails:** the task is unusual, or you need a specific format. The model just guesses, and its guesses won't always match.

---

## Few-Shot Prompting

Show 2–5 examples before you ask your real question, so the model copies the pattern.

**Prompt:**
```
Review: "Fast shipping, loved it!" → Sentiment: POS
Review: "Broke after one day." → Sentiment: NEG
Review: "The battery life is terrible." → Sentiment:
```
**Output:** → NEG

* **Problem it fixes:** zero-shot often gets the right idea but the wrong shape — wrong label, wrong structure, wrong tone. Examples show it exactly what shape you want.
* **When to use it:** the exact format matters (labels, JSON keys, tone), but training a whole new model is too much for the job.
* **Cost:** you pay extra tokens — extra time and money — every single time you ask.

---

## Chain-of-Thought (CoT)

Ask the model to think step by step, instead of jumping straight to an answer.

**Prompt (no CoT, often wrong):** `A store had 23 apples, sold 15, then got 8 more. How many now?`
→ "16" *(wrong)*

**Prompt (with CoT):** `A store had 23 apples, sold 15, then got 8 more. Think step by step, then give the final answer.`
→ "23 − 15 = 8. 8 + 8 = 16." *(right — and now you can see how it got there)*

* **Problem it fixes:** the model writes one word at a time, with no scratch paper. On a hard problem, jumping straight to the answer skips the real thinking. Writing it out step by step gives it a place to work things out first.
* **When to use it:** math, logic, anything with more than one step.
* **Cost:** more words means more time and money. Newer "thinking" models already do a version of this on their own.

---

## Self-Consistency

Ask the same step-by-step question several times, then go with the answer most of the tries agree on.

* **Problem it fixes:** any single attempt can go down the wrong path and still sound completely sure of itself.
* **How:** ask N times, then pick the most common final answer.
* **When to use it:** when getting it right matters more than speed or cost.

---

## ReAct (Reason + Act)

Think out loud, take an action, look at what happened, then think again — on repeat until you're done.

**Prompt pattern:**
```
Question: What's the population of the capital of the country the Eiffel Tower is in?

Thought: I need to know which country has the Eiffel Tower.
Action: search("Eiffel Tower country")
Observation: France
Thought: Now I need the capital of France.
Action: search("capital of France")
Observation: Paris
Thought: Now I need the population of Paris.
Action: search("population of Paris")
Observation: about 2.1 million
Thought: I have everything I need.
Final Answer: about 2.1 million (Paris, France)
```

* **Problem it fixes:** plain step-by-step thinking can't check real facts, so it can guess wrong and still sound confident. Plain tool-calling can check facts, but it doesn't explain *why* it's calling a tool, so it can pick the wrong one or misread the result. ReAct does both at once: think, act, check, think again.
* **When to use it:** this is the pattern behind almost every AI "agent" that uses tools today — including the assistant helping you right now.
* **Where the full version lives:** here, you're learning it as a way of prompting. The real version — actual tools, memory across steps, and what to do when a tool fails — is covered in [4_applied_ai](../4_applied_ai/README.md). Come back and re-read this section once you've built one; it'll click differently.

---

## Prompt Chaining (Breaking a Task Into Steps)

Split one big task into a few small prompts, and feed each answer into the next one.

**Example:** `Pull the key facts out of this article` → `Turn those facts into 3 bullet points` → `Rewrite the bullets in a formal tone`.

* **Problem it fixes:** one giant prompt trying to do everything at once usually does each part worse. Small mistakes in the middle get buried, and you can't fix just one step.
* **When to use it:** any job with more than one stage — draft, then check, then fix; or pull out data, then sum it up, then format it. This is the first step toward a real pipeline — see [5_production_ai](../5_production_ai/README.md).

---

## Role / System Prompting

Tell the model who it is, before you ask your real question.

**Example:** `You are a senior security engineer reviewing code for bugs.`

* **Problem it fixes:** the same question gets a flat, generic answer without a role, and a sharper, more useful one with one.
* **When to use it:** almost always, in a real app. You set this once, and it shapes every answer that follows.

---

## Structured Output Prompting

Ask for the answer in a fixed shape — like JSON — instead of a paragraph.

* **Problem it fixes:** free-form text is hard for code to read reliably. Real systems need an answer they can plug straight into code, not a paragraph to search through.
* **How:** describe the exact shape you want in the prompt. Many providers can also force the model to follow that shape, instead of just asking nicely.
* **When to use it:** any time the model's answer will be read by code next, not by a person.

---

## Quick Summary

| Technique | In One Line | Main Cost |
|---|---|---|
| Zero-shot | Just ask | Can fail on odd or unclear tasks |
| Few-shot | Show a few examples first | Costs tokens on every call |
| Chain-of-Thought | Think step by step | More output tokens |
| Self-consistency | Vote across several tries | Many more calls |
| ReAct | Think, act, check, repeat | Multiple round trips, needs tools |
| Prompt chaining | Split into small steps | More calls, more setup |
| Role/system prompting | Set the role once, up front | Almost free — big win |
| Structured output | Force a fixed, parseable shape | A bit less freedom, much more reliable |

## Next

See [1_introduction/README.md](README.md#thinking-like-an-ai-system-architect-production-framing) for how to pick between these when you're building something real.
