# Production AI (Weeks 23–30)

Evaluation → MLOps → AWS Bedrock/SageMaker → vLLM → Observability.

## What to Learn Deeply

**MLOps** — many engineers can build a demo; fewer can run it reliably:
- **Deployment** — packaging a model and serving code behind an API that survives restarts and rolling updates.
- **Latency** — time-to-first-token and tokens/second; what drives each (model size, batching, hardware) and where users actually feel it.
- **GPU memory** — why context length, batch size, and model size all compete for the same finite VRAM, and what happens when they don't fit (OOM, forced eviction).
- **Monitoring** — is it still working right now: error rates, latency percentiles, token throughput.
- **Drift** — is it still working right *for the same reasons*: input distribution or output quality quietly changing after launch.
- **Versioning** — which model/prompt/config produced a given output, and how to roll back one without rolling back the others.
- **Scaling** — handling more traffic than one instance or GPU can serve.
- **Cost** — dollars per request, and the levers that move it (model choice, caching, batching, quantization).
- **Docker** — packages a model, its dependencies, and runtime into one reproducible container, so "works on my machine" becomes "works everywhere."
- **Kubernetes** — orchestrates those containers in production: autoscaling inference pods with traffic, scheduling GPU resources, rolling out new model versions without downtime, and self-healing when a pod crashes.

**Evaluation** — the step most demos skip:
- *What problem does it solve?* "It looks good in my three test prompts" doesn't mean it works. Evaluation is how you find out before your users do.
- *How?* A fixed test set plus a metric (exact match, LLM-as-judge, human review), run automatically whenever the model, prompt, or retrieval pipeline changes.
- *When?* Before every deploy, not just once at the start of the project.

## Next

[6_portfolio](../6_portfolio/README.md) — apply evaluation, deployment, and monitoring to systems you design end to end.
