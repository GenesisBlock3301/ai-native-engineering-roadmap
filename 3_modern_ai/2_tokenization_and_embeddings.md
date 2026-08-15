# Tokenization & Embeddings

## The Whole File in One Table

Two jobs happen before a model can think. Here is **why**, **how**, and **where** for both:

| | **Tokenization** | **Embedding** |
|---|---|---|
| **Why** | A model does math, not text. Words must become numbers. | ID `995` is just a label. A label has no meaning. Numbers need *direction*. |
| **How** | BPE — glue the most common pair of bytes, 50,000 times. | Look up a learned row of numbers for each ID. |
| **Where** | **Outside** the neural network. Plain code, on CPU, before anything is sent. | **Inside** the model, as its very first layer, on GPU. |
| **Costs you** | Your entire bill. Everything is billed per token. | Memory. And your worst search bugs. |

If you only remember one line: **tokenization is a dictionary lookup on a CPU. Embedding is a table lookup on a GPU. Neither is intelligent.**

---

## How To Read This File

Go in order. Each part needs the one before it.

```
Part 1   WHERE it happens        the system map — start here
Part 2   HOW text becomes IDs    tokenization, BPE, box size
Part 3   WHAT it costs you       money, context limits, the strange bugs
Part 4   HOW IDs become meaning  embeddings — and the two kinds people mix up
Part 5   WHAT you should choose  the actual decisions
```

**After this file you should be able to answer four real questions:**

| Question you will be asked at work | Which part answers it |
|---|---|
| "Why is our Bangla version 5× the bill of English?" | Part 2 |
| "Will these 200 documents fit in 128k context?" | Part 3 |
| "Why does search return the passage saying the *opposite*?" | Part 4 |
| "Can we swap the embedding model next sprint?" | Part 5 |

Practice: [1_tokenization_and_embeddings.ipynb](code/1_tokenization_and_embeddings.ipynb) — you build a tokenizer by hand.

---

# Part 1 — WHERE This Actually Happens

Most confusion here comes from not knowing *which machine* is doing *which job*. So look at the map before the details.

### Map A — when you call an LLM

```
   YOUR SIDE (CPU, cheap)              THE MODEL SERVER (GPU, expensive)
   ─────────────────────               ────────────────────────────────

   "Hello world"
        │
        │ ① TOKENIZER
        │   plain code, no AI
        │   ~1 millisecond
        ▼
   [15496, 995] ─────── network ──────► ② EMBEDDING TABLE
                                           the model's first layer
                                           one row per ID
                                              │
                                              ▼
                                           transformer layers
                                           (all the real thinking)
                                              │
                                              ▼
                                        ③ a score for every one
                                           of the 50,257 pieces
                                              │
                                              ▼
                                           pick one ID
        ◄──────────── network ─────────────  995
        │
        │ ④ DETOKENIZER
        │   IDs back to text
        ▼
   " world"
```

Three things this map tells you that nothing else will:

1. **The tokenizer is not part of the neural network.** It is ordinary code. No GPU, no model, no cost. This is why you can count tokens *before* you spend a rupee — run the same tokenizer on your own laptop.
2. **Steps ① and ④ are the same table, used in both directions.** Text → ID going in, ID → text coming out.
3. **Step ③ is not a reverse lookup.** The model does not convert a vector back into an ID. It gives a score to *every piece in the box* and then picks one. That is where temperature and top-p live.

### Map B — when you build search (RAG)

This is a **completely different place** in your system. Same word "embedding", different machine, different time.

```
   INDEX TIME (offline, once)          QUERY TIME (online, every search)
   ──────────────────────────          ────────────────────────────────

   your 10,000 documents               user types "refund policy"
        │                                        │
        ▼                                        ▼
   ┌──────────────────────────────────────────────────┐
   │        THE SAME EMBEDDING MODEL, BOTH SIDES      │  ← must be
   └──────────────────────────────────────────────────┘     identical
        │                                        │
        ▼                                        ▼
   [0.02, -0.31, ...]                    [0.01, -0.40, ...]
        │                                        │
        ▼                                        ▼
   stored in a vector database ◄──── compared ────┘
                                     return top 5
```

**This is the answer to "where".** Token embeddings are a *layer inside a model you rent*. Text embeddings are a *separate service you run twice* — once slowly over everything you own, once quickly per user query.

Confusing them is expensive. Part 4 explains why.

---

# Part 2 — HOW Text Becomes IDs

## The problem: how do you cut text into pieces?

Two obvious ways. Both fail.

**Try 1 — one number per word.** The list of words never ends: names, typos, `kubectl`, emoji, other languages. Any word you did not store becomes `<UNK>` and its meaning is gone.

**Try 2 — one number per letter.** Nothing is unknown now. But `"hello"` costs 5 pieces instead of 1. The model works **per piece**, so you pay 5× for the same page.

**The choice everyone made: subwords.** Common words stay whole. Rare words break into parts.

```
"the"          →  ["the"]                     common → 1 piece
"unhappiness"  →  ["un", "happi", "ness"]     rare   → 3 pieces
```

Nothing is ever unknown, because in the worst case a word falls apart into single bytes. And common text stays short.

## BPE — how the pieces are chosen

BPE means **Byte-Pair Encoding**. One sentence: **glue the pair that appears most often, over and over.**

Start with 256 pieces (one per byte). Then repeat on a huge pile of text:

> Find the most common pair of neighbours. Glue it into one new piece. Save the rule.

```
start      l o w      l o w e r      l o w e s t
           ─┬─        ─┬─            ─┬─
            └─ "l"+"o" appears 3 times — winner

merge 1    lo w       lo w e r       lo w e s t
           ──┬──      ──┬──          ──┬──
              └─ "lo"+"w" appears 3 times — winner

merge 2    low        low e r        low e s t
```

Do it 50,000 times and you have GPT-2:

```
   256 bytes   +   50,000 merges   +   1 special   =   50,257
  (the fallback)  (the learned bits) (<|endoftext|>)
```

Those 256 byte pieces are the safety net. Feed GPT-2 an emoji, Bengali, or random binary — it can never fail, because it can always spell it out one byte at a time. There is no `<UNK>` at all.

To use it: split text into bytes, replay the saved rules in the order they were learned. No thinking, just rules.

## Box size — what it means

**Box size = how many different pieces the tokenizer knows.** That is all it means.

| Model | Year | Pieces it knows |
|---|---|---|
| GPT-2 | 2019 | 50,257 |
| GPT-4 (`cl100k`) | 2023 | 100,277 |
| Llama 3 | 2024 | 128,256 |
| GPT-4o (`o200k`) | 2024 | 200,019 |

Remember how the box fills: **most common first, rarest last.** When it is full, everything left over gets no piece of its own.

GPT-2 learned almost only English, so its 50,000 slots went to English fragments. Watch what that does — measured, not guessed:

```
আমি   ("I")   =  3 letters  =  9 bytes

GPT-2     ▓▓▓▓▓▓     6 tokens   [48071, 228, 48071, 106, 48071, 123]
GPT-4o    ▓          1 token    [154271]
```

Look at the GPT-2 IDs. `48071` appears three times. Decode it and you get the bytes `E0 A6` — the two bytes that begin *every* Bengali letter. So GPT-2 learned exactly **one** Bengali piece. That single piece cut 9 bytes to 6 tokens. Everything else is still raw bytes.

A whole sentence, same meaning both sides:

```
"I speak in Bangla"        "আমি বাংলায় কথা বলি"

GPT-2      5 tokens         35 tokens      ← 7× penalty
GPT-4o     5 tokens          6 tokens      ← gone
```

English costs the same on both. Only Bengali moved.

## Why bigger models use bigger boxes

If a big box is better, why did GPT-2 not just use 200,000?

**Because the box is not free.** Every piece needs its own row of numbers stored inside the model. The row is as wide as the model. GPT-2 is 768 numbers wide.

```
 50,257 pieces × 768  =   38.6 million numbers
200,019 pieces × 768  =  153.6 million numbers
```

Now the punchline: **GPT-2 small is only 124 million numbers in total.**

A 200k box would weigh 153M — **more than the entire model**. The dictionary would be heavier than the brain. Impossible in 2019.

Watch it stop mattering as models grow:

| Model | Whole model | Box | Box weighs | Share of model |
|---|---|---|---|---|
| GPT-2 small | 124M | 50,257 | 38.6M | **31%** |
| GPT-2 small *with a 200k box* | 124M | 200,019 | 153.6M | **124% — impossible** |
| Llama 3 8B | 8B | 128,256 | 525M | 6.6% |
| Llama 3 70B | 70B | 128,256 | 1.05B | **1.5%** |

The box costs a **fixed** amount. Models grew about 500×. The box only grew 4×. So it went from *"a third of my whole model"* to *"rounding error"*.

**Bigger models use bigger boxes because they can finally afford to** — not because bigger is smarter.

Two costs remain even today:

1. **Slower each step.** To pick its next word the model scores *every* piece in the box (step ③ on Map A). 50k scores vs 200k scores.
2. **Rare pieces are badly learned.** A piece seen only 50 times in all of training never learned its meaning.

---

# Part 3 — WHAT It Costs You

Every bill and every context limit is counted in **tokens**. Not words. Not letters.

Rough English rule: **1 token ≈ 4 characters ≈ ¾ of a word.** So 1,000 tokens ≈ 750 English words.

That rule is English-only. And the penalty for other languages is **not fixed** — it depends on the box you picked. Three Bengali sentences vs their English translations:

| Tokenizer | Box | Bengali costs |
|---|---|---|
| GPT-2 | 50,257 | **6.7×** English |
| cl100k (GPT-4) | 100,277 | **4.7×** English |
| o200k (GPT-4o) | 200,019 | **1.2×** English |

Read that as a bill. If your users write Bengali, moving from a cl100k-era model to an o200k-era one cuts your input tokens ~4× before you touch a single prompt.

**Where this bites at work:**

- Raw JSON burns tokens on `{`, `"`, and the same key names repeated every row. A flatter format is cheaper with zero quality loss.
- *"How many r's in strawberry?"* fails on many models. The model never saw letters — it saw 3 chunks. Not stupidity. Tokenization.
- A prompt that fits 128k in English may not fit in Bangla.

> **Architect habit:** before tuning a prompt for quality, count its tokens. Look again at Map A — the tokenizer runs on *your* CPU, so counting is free and instant. Cheapest measurement in the whole stack.

## Special tokens — what a "chat message" really is

Your neat Python list:

```python
[{"role": "user", "content": "hi"}]
```

is flattened into one long string before the model sees it:

```
<|begin_of_text|><|start_header_id|>user<|end_header_id|>

hi<|eot_id|><|start_header_id|>assistant<|end_header_id|>

└─ start of text  └─ "a user is speaking"      └─ "this turn ended"
```

There is no such thing as a "message" inside the model. Only one long string with markers in it. This flattening happens at step ① on Map A — before the network call.

Put a marker in the wrong place and the model behaves strangely in ways that look like magic. **Always use the tokenizer's built-in chat template.** Never build the string yourself.

---

# Part 4 — HOW IDs Become Meaning

An ID is just a label. `995` is not bigger or better than `994`. So the model looks up a **row of numbers** for each ID — and *those* numbers carry meaning, because they were learned.

Now the trap. The word "embedding" names two different things, in two different places. Look back at Map A and Map B:

```
1. TOKEN EMBEDDING                  2. TEXT EMBEDDING
   Map A — inside the model            Map B — a separate service

   one row per piece                   one vector per document
   ┌─────────────────┐                 "The cat sat on the mat"
   │ id 0    → [...] │                           │
   │ id 995  → [...] │ ← " world"                ▼
   │ id 50256→ [...] │                 a separate small model
   └─────────────────┘                           │
                                                 ▼
   learned during training           [0.02, -0.31, 0.88, ...]
   runs on GPU, inside the model     runs as your own service
   you never touch it                you run it twice: index + query
```

Type 2 is what powers everything in [4_applied_ai](../4_applied_ai/README.md).

**Similarity is measured by cosine similarity** — the angle between two arrows. Length ignored, only direction counts.

```
cos(a, b) = (a · b) / (|a| × |b|)

  1.0  = same direction  (same meaning)
  0.0  = unrelated
 -1.0  = opposite
```

You will almost never see negative scores on real text. What matters is the **ranking**, not the raw number.

**The trap that will cost you two days of debugging:**

```
"The movie was good"      ─┐
                           ├─ cosine ≈ 0.95  ← nearly identical!
"The movie was not good"  ─┘
```

Embeddings capture **topic**, not **logic**. Both sentences are about reviewing a movie. The word "not" barely moves the arrow.

---

# Part 5 — WHAT You Should Choose

Every problem above has a decision attached. Here they are in one place.

### 1. Which model — for cost?

Look at what language your users write in.

| Your users write | Choose | Why |
|---|---|---|
| English only | ignore box size entirely | Box size changes English cost ~10%. Not a decision. Pick on price and quality. |
| Bangla / Hindi / Arabic / Thai | a 200k-class box (GPT-4o, Llama 3, Gemma) | 4–6× cheaper on the same text. The biggest single cost win available to you. |
| Mostly code | 100k-class or bigger | Code has long rare identifiers that small boxes shatter. |

**Measure first:** take 20 real user messages, encode with both, compare totals. Five minutes, on your own CPU, free.

### 2. How to send structured data?

| Payload | Choose |
|---|---|
| under ~1,000 tokens | raw JSON — do not optimise, your time costs more |
| large or repeated tables | short keys, or CSV-style rows |

The trap is optimising a 200-token prompt. Saving 30 tokens is worth nothing. Measure before you rewrite.

### 3. Which embedding model, and how big?

- **Start small.** 256–768 dimensions, a good multilingual model.
- Go bigger only when you have *measured* bad recall. Bigger costs storage and search time forever.
- Many models let you cut the vector short (Matryoshka style — keep the first 256 of 1,536) and lose very little.
- **Pick once.** Look at Map B: the same model must run on both sides. Change it and every stored vector becomes garbage — you must re-embed all 10,000 documents. That is a migration with downtime, not a config change.

### 4. When NOT to use embeddings

The one people get wrong. Embeddings match topic. They do not do logic.

| Query | Embeddings? | Use instead |
|---|---|---|
| "documents about refund policy" | ✅ yes | — |
| "invoice #A-4471" | ❌ no | exact match / keyword |
| "orders NOT shipped" | ❌ no | a filter in the database |
| "price under 500" | ❌ no | a `WHERE` clause |

**Default for a real product: hybrid.** Keyword search catches exactly what embeddings miss.

### 5. The cheapest thing that works

Do not build a vector database on day one. Climb the ladder:

```
1. Count your tokens          free, 5 minutes
2. Try keyword search         cheap, often enough
3. Add embeddings             only when you can NAME what keyword missed
4. Add reranking              only when you can NAME what embeddings missed
```

Each step costs more money and more complexity. Do not skip to step 3 because it sounds impressive.

---

## Check Yourself

No notes, no AI. Say it out loud.

1. **Where** does the tokenizer run — your machine or the GPU server? Why does that make token counting free?
2. Why can a byte-level BPE tokenizer never produce `<UNK>`?
3. Why could GPT-2 not have used a 200k box in 2019? Answer with a number.
4. A big box means fewer tokens per sentence, but each step scores 200k pieces instead of 50k. **Does generation come out faster or slower?**
5. Your search returns the passage saying the *opposite* of the question. What is happening, and what do you do about it?

---

## Quick Summary

| Idea | In one line |
|---|---|
| Tokenizer | text → IDs, from a fixed box of pieces |
| Where it runs | your CPU, outside the model — so counting tokens is free |
| Subword | common words whole, rare words in pieces, nothing unknown |
| BPE | glue the most frequent pair, repeat 50,000 times |
| 256 byte pieces | the safety net — why `<UNK>` cannot happen |
| Box size | how many pieces the tokenizer knows |
| Why boxes grew | the box costs a fixed amount; models got big enough to afford it |
| 1 token | ≈ 4 English characters ≈ ¾ word |
| Non-English | 1.2×–7× more tokens, depending on the box you picked |
| Special tokens | the real shape of a "chat message" |
| Token embedding | one vector per token, inside the model, on GPU |
| Text embedding | one vector per document, your own service, run at index time and query time |
| Cosine similarity | angle between two arrows — matches topic, not logic |
| Default retrieval | hybrid (keyword + vector), not vector alone |

## Next

[3_llm_inference.md](3_llm_inference.md) — Map A showed the arrows. Next file opens the box on the right: what actually happens on the GPU when you press enter, and why the first word takes longer than all the rest.

## Sources

- [Hugging Face — Tokenization algorithms summary](https://huggingface.co/docs/transformers/en/tokenizer_summary)
- [Hugging Face LLM Course — Byte-Pair Encoding tokenization](https://huggingface.co/learn/llm-course/chapter6/5)
- [Sebastian Raschka — Implementing a BPE tokenizer from scratch](https://sebastianraschka.com/blog/2025/bpe-from-scratch.html)
