# YouTube Tags — Module by Module

Tags for every video in this roadmap, organised by phase folder. Each ``` block is
**one video's complete tag set** — copy the whole block into the YouTube tag box and stop.

**Do not combine two blocks.** YouTube gives you **500 characters total for all tags on a
video**, shared across the whole list. Every block below already sits at 220–290 characters
with 10–13 tags, which is the range that performs best; stuffing the full 500 with loosely
related keywords dilutes the relevance signal and hurts ranking.

Tags are a minor signal in 2026. Title, thumbnail, and watch time do the real work. Put the
exact search phrase (e.g. "kv cache explained") in the **title** and in the **first two
lines of the description**, then let these tags support it.

---

# Root — `README.md`

### Channel intro / the 2026 roadmap

```
ai engineer roadmap, how to become an ai engineer, learn ai in 2026, ai engineering roadmap 2026, ai systems engineer, machine learning roadmap, ai for software engineers, self taught ai engineer, ai career path, learn llm from scratch, ai native engineering
```

---

# Module 1 — `1_introduction/`

### 1.1 LLM settings — `1_llm_setting.md`

```
llm settings explained, temperature explained llm, top p explained, top k sampling, max tokens llm, stop sequence llm, frequency penalty, presence penalty, openai api tutorial, gemini api tutorial, llm parameters explained
```

### 1.2 Prompt engineering — `2_prompt_engineering.md`, `basic_prompt.ipynb`

```
prompt engineering, prompt engineering tutorial, chain of thought prompting, few shot prompting, zero shot prompting, react agent prompting, system prompt explained, prompt chaining, structured output json llm, context engineering, prompt engineering 2026
```

---

# Module 2 — `2_ai_foundations/`

### 2.1 Foundations overview — `README.md`, `how-ai-actually-works.pptx`

```
how ai actually works, ai foundations, math for machine learning, vectors and embeddings, probability for machine learning, loss landscape explained, local minima, why transformers replaced rnn, residual connections, layer normalization, vanishing gradient problem
```

### 2.2 Gradient descent from scratch — `gradient_descent_from_scratch.ipynb`

```
gradient descent explained, gradient descent from scratch, machine learning basics, loss function explained, mse loss explained, learning rate explained, linear regression from scratch, numpy machine learning, how neural networks learn, ml math for beginners
```

### 2.3 XGBoost vs linear regression — `xgboost_vs_linear_regression.ipynb`

```
xgboost tutorial, xgboost vs linear regression, gradient boosting explained, overfitting explained, bias variance tradeoff, feature engineering, when to use xgboost, tabular machine learning, scikit learn tutorial, train test split
```

---

# Module 3 — `3_modern_ai/`

### 3.1 Attention & transformers — `1_attention_and_transformers.md`, `code/2_attention_from_scratch.ipynb`

```
attention mechanism explained, transformers explained, self attention explained, query key value attention, multi head attention, causal masking transformer, rope positional encoding, grouped query attention, flash attention explained, mixture of experts, attention from scratch
```

### 3.2 Tokenization & embeddings — `2_tokenization_and_embeddings.md`, `code/1_tokenization_and_embeddings.ipynb`

```
tokenization explained, byte pair encoding, bpe from scratch, tiktoken tutorial, embeddings explained, cosine similarity explained, semantic search tutorial, token cost calculation, context window explained, why llms cant spell, vector embeddings tutorial
```

### 3.3 LLM inference & KV cache — `3_llm_inference.md`, `code/3_kv_cache_and_inference_math.ipynb`

```
kv cache explained, llm inference explained, prefill vs decode, time to first token, llm inference optimization, continuous batching, pagedattention vllm, speculative decoding, llm quantization explained, prompt caching, gpu memory for llm, llm cost optimization
```

### 3.4 Fine-tuning & alignment — `4_fine_tuning_and_alignment.md`, `code/4_lora_from_scratch.ipynb`

```
fine tuning llm, lora explained, lora from scratch, qlora tutorial, peft fine tuning, rlhf explained, dpo vs rlhf, grpo explained, fine tuning vs rag, catastrophic forgetting, instruction tuning, llm alignment explained
```

### 3.5 Hugging Face & sampling — `5_hugging_face.md`, `code/5_hugging_face_and_sampling.ipynb`

```
hugging face tutorial, transformers library tutorial, run llm locally, autotokenizer tutorial, temperature sampling explained, top k sampling, nucleus sampling top p, greedy decoding, logits explained, base vs instruct model, chat template llm
```

### 3.6 Practice & self-check — `6_practice_and_selfcheck.md`

```
llm internals explained, llm concepts quiz, gqa vs mqa, kv cache calculation, llm cost estimation, token cost report, ai engineer practice questions, explain llm simply, llm study guide, learn llm the right way
```

---

# Module 4 — `4_applied_ai/`

### 4.1 RAG — `README.md`

```
rag explained, rag tutorial, retrieval augmented generation, build rag from scratch, chunking strategy rag, vector database explained, reranking in rag, semantic search rag, rag vs fine tuning, embedding model comparison, rag pipeline architecture
```

### 4.2 AI agents & MCP — `README.md`

```
ai agents explained, ai agents tutorial, agentic ai, mcp explained, model context protocol, tool calling llm, agent memory explained, multi agent systems, langgraph tutorial, build ai agent from scratch, agent evaluation, context engineering
```

---

# Module 5 — `5_data_engineering_infra/`

### 5.1 Data pipelines — Spark, Kafka, batch vs streaming

```
data engineering for ai, etl pipeline explained, batch vs streaming, apache spark tutorial, spark partitioning, kafka explained simply, kafka for beginners, feature store explained, data pipeline for llm, duckdb vs spark, data engineering 2026
```

### 5.2 Orchestration & infrastructure — Airflow, Terraform, CI/CD

```
airflow tutorial, airflow dag explained, workflow orchestration, langgraph vs airflow, prefect vs temporal, dagster tutorial, terraform tutorial, infrastructure as code, ci cd for machine learning, gpu node pool, deploy vector database
```

---

# Module 6 — `6_production_ai/`

### 6.1 Deployment & serving

```
llm in production, llm deployment, vllm tutorial, model serving explained, docker for machine learning, kubernetes gpu scheduling, autoscaling inference, aws bedrock tutorial, sagemaker tutorial, gpu out of memory, llm latency optimization, mlops tutorial
```

### 6.2 Evaluation & monitoring

```
llm evaluation, llm as a judge, eval set for llm, ai observability, model drift explained, regression testing llm, monitoring llm in production, latency percentiles, prompt versioning, rollback ml model, llmops
```

---

# Module 7 — `7_product_engineering/`

### 7.1 Deciding what to build — `1_what_is_a_product_engineer.md`, `2_deciding_what_to_build.md`

```
ai product engineering, what is a product engineer, deciding what to build ai, why ai pilots fail, ai roi, cost of being wrong ai, kill criteria, ai feature spec, evals as product spec, human in the loop ai, ai trust patterns
```

### 7.2 Unit economics — `code/1_ai_feature_unit_economics.ipynb`

```
llm unit economics, ai gross margin, token cost calculator, cost per user ai, prompt caching savings, model routing llm, ai pricing strategy, usage based pricing, saas vs ai margins, llm cogs, break even analysis
```

---

# Module 8 — `8_portfolio/`

### 8.1 Portfolio projects

```
ai portfolio projects, machine learning portfolio, ai projects for resume, llm projects, portfolio for ai engineer, end to end ai project, production ai project, github portfolio ai, ai project ideas 2026, build ai project
```

---

# Module 9 — `9_interview_readiness/`

### 9.1 Interview preparation

```
ai engineer interview, llm interview questions, ai system design interview, ml system design, machine learning interview 2026, rag interview questions, kv cache interview question, ai engineer interview prep, ai job preparation, technical interview ai
```

---

# Hashtags

Pick **3 only** — end of the description, or the last one in the title. More than 3 and
YouTube ignores all of them.

| Module | Hashtags |
|---|---|
| Root, 8, 9 | `#AIEngineering #LearnAI #TechCareers` |
| 1 | `#PromptEngineering #LLM #AIEngineering` |
| 2 | `#MachineLearning #DeepLearning #Python` |
| 3 | `#LLM #Transformers #AIEngineering` |
| 4 | `#RAG #AIAgents #LLM` |
| 5, 6 | `#MLOps #DataEngineering #LLM` |
| 7 | `#AIProduct #LLM #AIStrategy` |

---

# Sources

- [YouTube Tags Best Practices 2026 — limits, tag count, SEO impact](https://touhfa.art/blog/seo/youtube-tags-guide/)
- [YouTube Character Limits 2026 — title, description, tags](https://utilhq.com/articles/youtube-character-limits-seo-guide/)
- [AI Search Trends for 2026 — Semrush](https://www.semrush.com/blog/ai-search-trends/)
- [Context Engineering: The 2026 Playbook for AI Agents](https://cruxdigits.nl/blog/context-engineering-ai-agents-2026/)
- [7 Agentic AI Trends to Watch in 2026 — MachineLearningMastery](https://machinelearningmastery.com/7-agentic-ai-trends-to-watch-in-2026/)
