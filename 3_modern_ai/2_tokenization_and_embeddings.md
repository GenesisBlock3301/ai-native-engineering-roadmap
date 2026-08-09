# Tokenization & Embeddings

A model cannot read text. It can only multiply numbers. This file is about the two steps that turn your words into numbers:

```
"Hello world"  →  [15496, 995]  →  [[0.12, -0.7, ...], [0.4, 0.02, ...]]
    text            token IDs              vectors
                 (tokenizer)              (embeddings)
```

Practice for this file: [1_tokenization_and_embeddings.ipynb](code/1_tokenization_and_embeddings.ipynb) — you build a tokenizer by hand.

---

## Why Not Just Use Words? Or Letters?

Two easy ideas, both bad:

| Idea | The problem |
|---|---|
| One ID per **word** | The vocabulary never ends (names, typos, `kubectl`, emoji). Any unseen word becomes `<UNK>` — meaning lost. |
| One ID per **letter** | Vocabulary is tiny, but now "hello" costs 5 steps instead of 1. Sequences get very long, and long means slow and expensive. |

So we sit in the middle: **subwords**. Common words stay whole. Rare words split into pieces.

```
"unhappiness"  →  ["un", "happi", "ness"]
"the"          →  ["the"]
```

Nothing is ever unknown, because in the worst case a word falls apart into single bytes.

---

## BPE (Byte-Pair Encoding)

BPE is the algorithm behind almost every model you will use — GPT, Llama, Gemma, Qwen.

**How it learns (training the tokenizer, done once):**

1. Start with a vocabulary of single bytes (256 of them).
2. Look at a huge pile of text. Find the **most frequent pair** of neighbours.
3. Merge that pair into one new token. Save the merge rule.
4. Repeat until you hit your target vocabulary size.

```
Round 1: "l" + "o" is the most common pair  →  new token "lo"
Round 2: "lo" + "w"                          →  new token "low"
Round 3: "e" + "r"                           →  new token "er"
```

**How it encodes (every time you send a prompt):** split the text into bytes, then apply the saved merge rules in the order they were learned.

GPT-2's vocabulary is 50,257 = 256 base bytes + 50,000 learned merges + 1 special end-of-text token. Modern models use bigger ones (100k–200k) because a bigger vocabulary means fewer tokens per sentence.

You build this whole loop yourself in the notebook. It is about 40 lines of plain Python.

---

## Why This Costs You Money

Every API bill and every context limit is counted in **tokens**, not words or letters.

Rough English rule: **1 token ≈ 4 characters ≈ ¾ of a word.** So 1,000 tokens ≈ 750 English words.

But that rule is only for English. Tokenizers are trained mostly on English text, so other languages break into more pieces:

| Text | Roughly |
|---|---|
| English sentence | 1× tokens (baseline) |
| Same meaning in Bangla / Hindi / Thai | 2–4× more tokens |
| JSON with long key names | more tokens than you expect |
| A UUID or a hash | almost one token per character |

**Real consequences you will hit at work:**

- The same product, translated, can cost several times more per request.
- Sending raw JSON to a model burns tokens on `{`, `"`, and repeated key names. A compact format (CSV-like, or shorter keys) can cut cost with zero quality loss.
- "Count the letters in *strawberry*" fails on many models — the model never saw letters, it saw a few subword chunks. This is not stupidity, it is tokenization.
- A prompt that "fits in the context window" in English may not fit in another language.

**Architect habit:** before optimising a prompt for quality, count its tokens. It is the cheapest measurement in the whole stack.

---

## Special Tokens

The tokenizer also inserts control markers the model was trained to obey:

```
<|begin_of_text|>  <|start_header_id|>user<|end_header_id|>  ...  <|eot_id|>
```

This is what a "chat template" really is: your neat `{"role": "user", "content": ...}` list gets flattened into one long string with these markers in the right places. If you build the string yourself and get the markers wrong, the model behaves oddly for reasons that look like magic. Always use the tokenizer's chat template — you will see it in the Hugging Face notebook.

---

## Embeddings: Two Kinds, Don't Mix Them Up

The word "embedding" is used for two related but different things. Interviews test this.

**1. Token embeddings (inside the model)**

A lookup table with one row per vocabulary entry. Token ID 15496 means "go get row 15496". These vectors are learned during training and live inside the model. You almost never touch them directly.

**2. Sentence/document embeddings (outside the model)**

One vector for a whole piece of text, made by a separate embedding model. Similar meaning → vectors point in a similar direction. This is what powers search, RAG, clustering, and deduplication in [4_applied_ai](../4_applied_ai/README.md).

**How similarity is measured:** cosine similarity — the angle between two vectors, ignoring their length.

```
cos(a, b) = (a · b) / (|a| × |b|)

 1.0  = same direction (same meaning)
 0.0  = unrelated
-1.0  = opposite direction
```

Real embedding models rarely give you negative numbers on normal text, so don't expect "opposite" to appear often. What matters is the *ranking*: which candidate scores highest, not the raw number.

**Things that surprise people:**

- "The movie was good" and "The movie was not good" score very high together. Embeddings capture topic much more than logic. This is a real source of RAG bugs.
- You cannot compare vectors from two different embedding models. Different models, different spaces. If you change your embedding model, you must re-index everything.
- Bigger vectors are not automatically better. They cost more storage and more search time. Many models now let you cut the vector short (Matryoshka-style) and lose very little quality.

---

## Quick Summary

| Idea | In one line |
|---|---|
| Tokenizer | turns text into IDs from a fixed vocabulary |
| Subword | common words whole, rare words in pieces, nothing unknown |
| BPE | repeatedly merge the most frequent pair |
| 1 token | ≈ 4 English characters ≈ ¾ word |
| Non-English | costs 2–4× more tokens for the same meaning |
| Special tokens | the real shape of a "chat message" |
| Token embedding | one vector per token, inside the model |
| Text embedding | one vector per document, outside the model |
| Cosine similarity | the angle between two vectors = how similar |

## Next

[3_llm_inference.md](3_llm_inference.md) — now that text is numbers, what actually happens when you press enter, and why the first word takes longer than the rest.

## Sources

- [Hugging Face — Tokenization algorithms summary](https://huggingface.co/docs/transformers/en/tokenizer_summary)
- [Hugging Face LLM Course — Byte-Pair Encoding tokenization](https://huggingface.co/learn/llm-course/chapter6/5)
- [Sebastian Raschka — Implementing a BPE tokenizer from scratch](https://sebastianraschka.com/blog/2025/bpe-from-scratch.html)
