# Interview Readiness (Weeks 49–56)

AI system design, debugging, model trade-offs, performance optimization, and behavioral interviews.

## How to Answer Any "Explain X" Question

For every topic in this roadmap, be ready to answer exactly three things:
1. What problem does it solve?
2. How does it work conceptually?
3. When should I use it, versus the alternative?

If you can answer those three cleanly, out loud, without notes, for every topic from [2_ai_foundations](../2_ai_foundations/README.md) through [7_portfolio](../7_portfolio/README.md), you're ready.

## What's Actually Being Asked Now

These used to be advanced topics; they're baseline interview questions today:
- Tokenization, KV cache, context window, hallucination, fine-tuning vs. alignment — see [3_modern_ai](../3_modern_ai/README.md).
- The RAG pipeline end to end, and agent failure/recovery — see [4_applied_ai](../4_applied_ai/README.md).
- Data pipeline design, batch vs. streaming, and workflow orchestration trade-offs — see [5_data_engineering_infra](../5_data_engineering_infra/README.md).
- Deployment, latency, drift, and cost trade-offs — see [6_production_ai](../6_production_ai/README.md).
- Why you made each architectural decision in your own portfolio projects — see [7_portfolio](../7_portfolio/README.md). The systems you built yourself are the questions you're least likely to be caught off guard by.

## System Design Reflex

For any "design an AI system for X" prompt, walk the same loop you used to build your portfolio:

```
Business Problem → Design AI Architecture → Choose Models → Design Retrieval →
Design Evaluation → (AI Generates Code) → Review → Debug → Optimize → Deploy → Monitor → Iterate
```

Say the trade-off out loud at each arrow — that's the actual signal an interviewer is listening for, not the final answer.
