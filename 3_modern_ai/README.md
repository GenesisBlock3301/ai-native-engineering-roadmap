# Modern AI (Weeks 7–14)

Transformers → Hugging Face → LLM inference → Fine-tuning concepts.

## What to Learn Deeply

**Attention & Transformers**
- **Why attention was invented** — RNNs process tokens one at a time, so information from early tokens fades before the model reaches later ones (the long-range dependency problem).
- **Why attention solves that** — every token can look at every other token directly, in one step, regardless of distance.
- **Why attention scales** — it's a handful of matrix multiplies (Q·Kᵀ, softmax, ·V), which GPUs parallelize extremely well, unlike an RNN's inherently sequential recurrence.

**LLMs** — know these deeply, not just by name; they're standard interview questions now:
- **Tokenization** — text isn't numbers; a tokenizer maps text to a fixed vocabulary of integer IDs the model can embed.
- **Embeddings** — each token ID maps to a learned vector; nearby vectors mean related meaning.
- **Inference & KV Cache** — generating token N+1 needs the key/value projections of tokens 1..N. Caching them avoids recomputing the whole sequence at every step, which is why context length affects both memory and speed.
- **Context Window** — the hard limit on how many tokens (prompt + history + output) the model can attend to at once.
- **Hallucination** — the model always predicts a plausible next token, not a looked-up truth. It hallucinates when "plausible" and "true" diverge, especially outside its training distribution.
- **Temperature & Sampling** — see [1_introduction](../1_introduction/README.md) for the hands-on version; here, understand *why* sampling from a distribution (instead of always taking the top token) is what makes generation non-deterministic.
- **Reasoning** — longer, structured intermediate generation (chain-of-thought and similar) that trades inference cost for accuracy on multi-step problems.
- **Fine-tuning** — adjusting a pretrained model's weights on task-specific data instead of relying on prompting alone, for when in-context examples aren't enough or a long prompt costs too much.
- **Alignment** — training a model to prefer the responses humans actually want (RLHF/DPO-style), separate from raw next-token-prediction capability.

## Worked Example: Learning Attention the 2026 Way

Old way: watch a 3-hour video, copy the code, forget it.

2026 way:
1. Ask: *why was attention invented?*
2. Then: *what problem does RNN have?*
3. Then: *why does attention solve that?*
4. Ask AI to draw attention visually.
5. Ask: *explain like I'm a backend engineer.*
6. Read the PyTorch / Hugging Face implementation — don't write it, read it.
7. Ask, line by line: *why is this line here?*
8. Repeat until you can explain every line yourself.

## Next

Once you can explain attention and the LLM concepts above without notes, move to [4_applied_ai](../4_applied_ai/README.md) — RAG and agents are built entirely on top of this.
