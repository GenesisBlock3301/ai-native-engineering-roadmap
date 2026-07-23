# Data Engineering & Infra (Weeks 23–30)

Data pipelines → Spark & Kafka → Airflow (workflow orchestration) → Terraform → CI/CD.

## What to Learn Deeply

**Data Pipelines** — every RAG index, fine-tuning dataset, and feature store starts here:
- *What problem does it solve?* Models and retrieval systems need clean, current data in a specific shape; raw data almost never arrives that way.
- *How does it work conceptually?* Extract from a source → transform (clean, dedupe, chunk, embed) → load into wherever it's consumed (a vector DB, a training set, a feature store). Batch runs on a schedule; streaming runs continuously as new data arrives.
- *When should I use it?* Any time your RAG/agent/fine-tuning system's quality depends on documents or events that change after you first built it — which is almost always.

**Spark** — distributed batch processing:
- **Problem it fixes:** some transform jobs (re-embedding millions of documents, deduping a large dataset) don't fit in memory, or finish fast enough, on one machine.
- **How it works:** it splits the data and the job across a cluster, and handles what a single-node script doesn't: partitioning, retries, fault tolerance when a worker dies mid-job.
- **When to use it:** the data genuinely doesn't fit on one machine. If pandas or DuckDB on a single node still finishes in reasonable time, you don't need Spark yet — it's a heavier tool than most pipelines actually require.

**Kafka** — the streaming/event backbone:
- **Problem it fixes:** some data isn't a batch you can schedule — it's continuous (user events, sensor data, logs) — and downstream systems need it close to real time.
- **How it works:** producers publish events to a topic; consumers read from that topic independently, at their own pace. The topic is a durable buffer between them, so producers and consumers never need to be online at the same moment.
- **When to use it:** a RAG index or agent needs to react to new data within seconds or minutes, not on the next scheduled batch run.

**Airflow** — the classic workflow orchestrator for data pipelines:
- **Problem it fixes:** a real pipeline is many steps with dependencies (extract → chunk → embed → index) that need to run on a schedule, retry on failure, and surface *which* step broke without someone grepping logs.
- **How it works:** you define the steps and their dependencies as a DAG; Airflow schedules runs, tracks what succeeded, retries what failed, and gives you one place to see the whole pipeline's health.
- **When to use it:** more than a couple of pipeline steps, run on a schedule, where "did it run, and if not, why" needs a real answer.

**Workflow Orchestration (the general idea)** — Airflow orchestrates *data* jobs; the same idea shows up one layer up, orchestrating *agent* steps (LangGraph, Prefect, Temporal, a custom state machine):
- *What problem does it solve?* Both cases need the same things: run steps in the right order, retry the ones that fail, and know the state of a long-running process without babysitting it.
- *How does it work conceptually?* Model the work as a graph of steps with explicit dependencies and state, instead of one long imperative script — so any single step can fail, retry, or resume without restarting the whole run.
- *When should I use it?* Data pipelines → Airflow-style tools. Multi-step agent workflows → the agent-orchestration concepts (planning, state, failure recovery) already covered in [4_applied_ai](../4_applied_ai/README.md) — same underlying problem, different domain.

**Terraform** — infrastructure as code:
- **Problem it fixes:** infrastructure clicked together by hand in a console is unreproducible and undocumented — nobody can tell you exactly what's running, or rebuild it once it's gone.
- **How it works:** you declare the infrastructure you want (a GPU node pool, a vector DB instance, an S3 bucket) in config files; Terraform diffs that against what actually exists and makes the minimal set of changes to match.
- **When to use it:** any infra you'll need to recreate, review, or roll back — which, in production, is all of it.

**CI/CD** — automated build, test, and deploy:
- **Problem it fixes:** running tests and deploying by hand doesn't scale past one person, and it's exactly the step people skip under deadline pressure — right when it matters most.
- **How it works:** every push triggers an automated pipeline — run tests → build → deploy — with each stage gating the next. For AI systems specifically, "tests" should include the evaluation suite from [6_production_ai](../6_production_ai/README.md): a prompt or retrieval change is a code change, and should fail a build the same way a broken unit test would.
- **When to use it:** any project more than one person touches, or any model/prompt change that ships to real users.

## How to Study This Phase

Stand up one pipeline end to end before reaching for the heavier tools: pull a handful of documents, chunk and embed them, load them into a vector DB, on a schedule, orchestrated by Airflow (or a lighter alternative like Prefect/Dagster). Add Spark only once that pipeline's data volume genuinely doesn't fit on one machine; add Kafka only once you have a real streaming source, not a hypothetical one. Then wrap the whole thing in Terraform and a CI/CD pipeline, so standing it back up doesn't depend on remembering what you clicked.

## Next

[6_production_ai](../6_production_ai/README.md) — the pipeline and infra above are what a production AI system runs on top of; that phase covers deploying, evaluating, and monitoring the model or agent that sits on top of it.
