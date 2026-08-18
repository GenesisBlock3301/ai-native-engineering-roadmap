# LLM Inference — What Happens After You Press Enter

## The Whole File in One Table

Making text is **two** different jobs, not one. Almost every production question is really a question about which job you mean.

| | **Prefill** | **Decode** |
|---|---|---|
| **Why** | The model must read your whole prompt first. It cannot say anything before that. | The answer comes out one token at a time. Token 2 needs token 1 first. |
| **How** | All prompt tokens go through the model together, at the same time. | One full pass through the model for each output token. |
| **Where** | On the GPU, **once**, before the first word shows up. | On the GPU, **again and again** — once per output word. |
| **What runs out** | Math power — the GPU's math units. We say it is *compute-bound*. | Memory speed — the time it takes to move weights around. We say it is *memory-bound*. |
| **What it costs you** | **TTFT**: how long the user stares at an empty screen. | **TPOT**: how fast the text flows — and most of your bill. |

New to these two words? Read Part 2 first, then come back to this table. It will make sense the second time.

One line to keep: **during decode, the GPU reads every weight in the model just to make one token.** The math is easy. The reading is the slow part. Almost every trick in this file moves fewer bytes. It does not do less math.

---

## The App We Will Use All The Way Through

Every section below uses the same real request. Same app, same numbers, start to finish.

**ShopBot** — a customer support chatbot for an online shop. It runs **Llama-3-8B** on **one 80 GB H100** GPU.

A real user types this:

```
  "my order #48291 arrived broken, can i get a refund?"
```

Here is what actually gets sent to the model:

```
  ┌─────────────────────────────────────────────┬──────────┐
  │  system prompt                              │   1,800  │  fixed. Refund rules,
  │  (policy, tone, 6 example chats)            │  tokens  │  same on every request.
  ├─────────────────────────────────────────────┼──────────┤
  │  retrieved context                          │     680  │  order #48291 record
  │  (order record + 3 policy chunks)           │  tokens  │  + matching policy
  ├─────────────────────────────────────────────┼──────────┤
  │  the user's actual question                 │      20  │  what they typed
  ├─────────────────────────────────────────────┼──────────┤
  │  TOTAL INPUT                                │   2,500  │
  └─────────────────────────────────────────────┴──────────┘

  ShopBot replies with about 120 tokens:
  "I'm sorry your order arrived damaged. Order #48291 was
   delivered on 12 Aug, so it is inside our 30-day window..."
```

**Notice the shape: 2,500 tokens in, 120 tokens out.** The user typed 20 tokens. The other 2,480 came from you. Hold on to that — it decides which cost lever works in Part 5.

---

## How To Read This File

```
Part 1   WHERE it happens        the serving map and the GPU memory map
Part 2   HOW generation works    prefill vs decode, TTFT vs TPOT
Part 3   WHAT the KV cache costs the one formula to remember
Part 4   HOW servers go fast     what vLLM really does
Part 5   WHAT you should choose  the decisions, cheapest first
```

| Question you will be asked at work | Which part |
|---|---|
| "Why is the first word slow but the rest fast?" | Part 2 |
| "How many users fit on one GPU?" | Part 3 |
| "We spend $40k a month. Give me five ways to cut it." | Part 5 |
| "Can we just paste all 500k tokens in?" | Part 5 |

Practice: [3_kv_cache_and_inference_math.ipynb](code/3_kv_cache_and_inference_math.ipynb).

This file is the difference between *"I use LLM APIs"* and *"I can design an LLM system"*.

---

# Part 1 — WHERE This Actually Happens

## Map A — where your request goes

```
  YOUR APP
     │
     │  HTTP, a few ms
     ▼
  LOAD BALANCER
     │
     ▼
  ┌──────────────────────────────────────────────┐
  │  INFERENCE SERVER   (vLLM / SGLang / TRT)    │
  │                                              │
  │   waiting queue  ──►  running batch          │
  │                          │                   │
  │                          │  continuous       │
  │                          │  batching swaps   │
  │                          │  requests here    │
  │                          ▼                   │
  │                   ┌──────────────┐           │
  │                   │     GPU      │           │
  │                   └──────────────┘           │
  └──────────────────────────────────────────────┘
```

The inference server is not a thin layer on top of the model. It is a **scheduler**. It decides who runs now, who waits, and how memory is shared. Most of Part 4 is about this scheduler, not about the model.

> **ShopBot on this map.** The user presses Enter. Their 2,500 tokens travel to the load balancer in ~5 ms. The inference server drops the request in the **waiting queue**. Right now 184 other chats are already running. A slot frees up 30 ms later, and ShopBot's request joins the **running batch**. Total time spent before the GPU even starts: ~35 ms.

## Map B — where the GPU memory goes

This map answers "how many users fit". That is the question people will really ask you.

```
  ONE 80 GB GPU SERVING Llama-3-8B

  ┌───────────────────────────────────────────────┐  80 GB
  │  MODEL WEIGHTS              16 GB             │  fixed, loaded once
  ├───────────────────────────────────────────────┤
  │  framework overhead         ~2 GB             │  fixed
  ├───────────────────────────────────────────────┤
  │                                               │
  │  KV CACHE            ~62 GB left over         │  ◄── THIS is what
  │                                               │      limits users
  │    at 128 KB per token:                       │
  │       8k context ×  1 user  =   1 GB          │
  │       8k context × 32 users =  32 GB   ✅ fits │
  │       8k context × 62 users =  62 GB   ⚠️ full │
  │                                               │
  └───────────────────────────────────────────────┘
```

**The weights are fixed. The KV cache is not.** It grows with every token *and* every user at the same time. So when you work out how many users fit, you are really asking a KV cache question — not a model size question.

> **ShopBot on this map.** One full chat is 2,500 in + 120 out = **2,620 tokens**. At 128 KB per token that is **335 MB of KV cache for one user**. The 62 GB of free space divided by 335 MB = **185 chats at the same time on one GPU**. Buying a bigger model does not raise that number. Shrinking the prompt does.

---

# Part 2 — HOW Generation Works

## First — what the model actually does

Give the model a list of tokens. It hands back **one** token: its guess at what comes next.

```
["The", "cat", "sat", "on", "the"]   →   MODEL   →   "mat"
```

That is the whole model. It cannot write a sentence. It cannot write a paragraph.

**One run of the model = one new token.**

## So how do you get 120 tokens?

You run the model 120 times. Each time, you stick the new token onto the end of the list and run it again.

> **ShopBot, run by run.** The list starts at 2,500 tokens.
>
> ```
> run 1    [ 2,500 prompt tokens ]                       →  "I'm"
> run 2    [ 2,500 + "I'm" ]                             →  " sorry"
> run 3    [ 2,500 + "I'm" " sorry" ]                    →  " your"
> run 4    [ 2,500 + "I'm" " sorry" " your" ]            →  " order"
>  ...
> run 120  [ 2,500 + 119 more ]                          →  "."
> ```

**Why not make all 120 at once?** Because run 3 needs the word from run 2 as part of its input. That word does not exist yet. You cannot skip ahead. This is the rule that shapes everything else in this file.

## Now compare run 1 with run 2

They are not the same job at all.

| | Tokens going in | How many are new |
|---|---|---|
| **Run 1** | 2,500 | **all 2,500** |
| **Run 2** | 2,501 | **1** |
| **Run 3** | 2,502 | **1** |

Run 1 has 2,500 fresh tokens to work through. Every run after it has just one.

That gap is why we give the two jobs different names:

> **Prefill = run 1.** The model reads your whole prompt. This happens **once**.
>
> **Decode = runs 2 to 120.** The model adds one token at a time. This happens **119 more times**.

## The plain-words version

- **Prefill is reading.** Someone hands you a letter. Your eyes take in the whole page. You do not read one letter, stop, then read the next letter. You take it all in together.
- **Decode is writing.** Now you write your reply by hand. One word. Then the next word. You cannot write word 5 before you have written word 4.

Reading a page is fast. Writing a page is slow. Same for the GPU, and for the same reason.

## What this costs

```
YOUR PROMPT (2,500 tokens)              GENERATED ANSWER (120 tokens)
┌──────────────────────────┐            ┌───┬───┬───┬───┬───┬─────┐
│      P R E F I L L       │    →       │ t │ t │ t │ t │ t │ ... │
│  all 2,500 at once, once │            └───┴───┴───┴───┴───┴─────┘
└──────────────────────────┘                    D E C O D E
                                          one token at a time, 120 times
```

**Prefill** reads all 2,500 prompt tokens together. The GPU likes this. It is one big job it can split up and do at the same time. It happens **once**.

**Decode** writes the answer one token at a time. Token 2 cannot start until token 1 exists. It happens **120 times**. Each time, the GPU reads all 16 GB of weights just to make a few bytes of output.

> **ShopBot's real clock.**
>
> ```
> prefill   2,500 tokens, one pass          →   ~150 ms
> decode    120 tokens × ~10 ms each        →  ~1,200 ms
>                                              ─────────
> user sees the full answer in                 ~1.35 s
> ```
>
> Prefill did **20× more token-work** than decode (2,500 vs 120) and still took **8× less time**. That is the whole point of the split.

**Why is decode the slow side?** An H100 moves about 3,350 GB of data per second. To make one token, it must read all 16 GB of weights: 16 ÷ 3,350 = **4.8 ms of pure reading**, before any math happens. Real speed lands near 10 ms. The GPU's math units sit mostly idle. It is waiting on memory, not on math.

## The two numbers to quote

| Metric | Means | Driven by |
|---|---|---|
| **TTFT** — time to first token | How long until the user sees *anything*. | Prompt length. Long prompt = slow start. |
| **TPOT** — time per output token | How fast the text flows after that. | Model size and memory speed — **not** prompt length. |

> **ShopBot:** TTFT = **150 ms** (the wait before "I'm" appears). TPOT = **10 ms** (100 words per second flowing after that). Add 2,000 more tokens of retrieved policy and TTFT goes to ~270 ms — but TPOT stays at 10 ms. A longer prompt makes the *start* slower, never the *streaming*.

These two pull in different directions. Different products care about different ones:

| Product | Optimise for | Why | Real example |
|---|---|---|---|
| A chat UI | **TTFT** | The user is watching an empty box. | ShopBot. Above 500 ms, users start clicking again. |
| Batch summarising | **throughput** | Nobody is watching. Total tokens/sec is all that matters. | Summarise 10,000 support tickets overnight. 8 hours or 9 hours — nobody cares. |
| A voice agent | **TPOT** | Speech must not break mid-sentence. | ShopBot on the phone. Speech needs ~25 tokens/sec steady. A 200 ms gap sounds broken. |

Say this difference out loud in a design interview and you sound senior.

---

# Part 3 — WHAT the KV Cache Costs

To make token 2,501, attention needs the Key and Value vectors of tokens 1–2,500. We already worked those out when we made token 2,500. If we work them out again at every step, the whole job becomes `n²` work all over again.

So we keep them. That is the KV cache: **trade memory for compute.** Use more memory so you do less math.

- **Problem it fixes:** without it, every new token reads the whole sequence again. The answer gets slower and slower as it grows.
- **How:** after each step, add the new token's K and V to a buffer. Reuse it forever.
- **The catch:** that buffer sits in expensive GPU memory, and it grows with **every token × every user**.

> **ShopBot without a KV cache.** Token 1 of the reply re-reads 2,500 tokens. Token 120 re-reads 2,619. Added up, that is ~313,000 tokens of repeated work instead of 120. The reply would take about **40 seconds** instead of 1.2. The cache is not an optimisation. It is what makes chat possible at all.

## The formula to remember

```
KV bytes per token = 2 × layers × kv_heads × head_dim × bytes_per_number
                     ↑
                     one for K, one for V
```

**Llama-3-8B** — 32 layers, 8 KV heads, head_dim 128, fp16 (2 bytes):

```
2 × 32 × 8 × 128 × 2  =  131,072 bytes  =  128 KB per token
```

> **ShopBot's numbers, straight from the formula.**
>
> ```
>   2,620 tokens ×  128 KB   =   335 MB   ← one chat
>     335 MB × 185 chats     =    62 GB   ← the GPU is now full
> ```
>
> So the honest answer to "how many customers can one GPU serve at once?" is **185**. Not "lots". Not "it depends". 185 — and you can show the arithmetic.

Now you can see the real reason GQA exists. (GQA = grouped query attention: many query heads share one key/value head.) The same model **without** GQA has 32 KV heads instead of 8. That is **512 KB per token** — four times more.

> **ShopBot without GQA.** One chat needs 1.34 GB instead of 335 MB. Your GPU now holds **46 chats, not 185**. Same model, same quality, same hardware — you just serve 4× fewer customers and your cost per chat goes up 4×.

**The interview question:** *you serve a 70B model at 100k context, batch 32 — what breaks?* The KV cache breaks. Do the math out loud.

---

# Part 4 — HOW Servers Go Fast

These all live in the scheduler from Map A. None of them change the model.

| Trick | Problem it fixes | How it works | ShopBot |
|---|---|---|---|
| **Continuous batching** | In a fixed batch, everyone waits for the slowest request. | As soon as one request ends, a new one takes its slot right away. | A "thanks!" reply ends after 8 tokens. A refund explanation runs 300. Without this, that slot idles for 292 steps ≈ **2.9 s of dead GPU**. |
| **PagedAttention** | Saving max-length cache space per user wastes most of the memory. | Keep the cache in small fixed blocks, like an OS uses pages. Under 4% waste. | Reserving the full 8,192-token window per chat costs 1.05 GB each → only **59 chats fit**. Paging it → back to **185**. A 3× win for free. |
| **Prefix caching** | Many users send the same long system prompt. You prefill it every time. | Keep the KV cache of the shared start, and reuse it. | The 1,800-token system prompt is identical on every request. See below. |
| **Chunked prefill** | One huge prompt blocks everyone else's streaming. | Cut the prefill into chunks. Mix them between decode steps. | One user pastes a 60,000-token returns policy. That prefill takes ~3.5 s, and **all 184 other chats freeze** while it runs. |
| **Speculative decoding** | Decode waits on memory, so the GPU sits idle most of the time. | A small draft model guesses k tokens ahead. The big model checks them all in one pass. Wrong guesses are thrown away, so **quality does not change**. | Llama-3-1B drafts 4 tokens, the 8B checks all 4 in one read of its weights. ~3 of 4 accepted → reply time **1.2 s → 0.5 s**. |
| **Quantization** | The weights are the thing you keep re-reading. | Store them in fewer bits → fewer bytes to move → faster and cheaper. | 16 GB → 8 GB. See below. |

## Prefix caching is your free win

Put your long, fixed system prompt **first**. Put the user's changing text **last**.

```
✅  [ long fixed system prompt ][ user's question ]     cache hits
❌  [ user's question ][ long fixed system prompt ]     full prefill, every call
```

> **ShopBot's free 3× speedup.** The first 1,800 tokens never change. With prefix caching on, only the 700 new tokens (order record + question) get prefilled.
>
> ```
> no cache    prefill 2,500 tokens   →  TTFT ~150 ms
> cache hit   prefill   700 tokens   →  TTFT  ~45 ms
> ```
>
> At 100 chats a minute, the broken order wastes 1,800 × 100 = **180,000 tokens of prefill every minute**, forever. The fix is moving one string.

That is one line of reordering for real money saved. Providers also charge less for cached input tokens.

## Quantization in plain words

Numbers can be stored with more or fewer bits. Fewer bits = smaller and faster, but you lose some quality.

| Precision | 8B weights | Typical use |
|---|---|---|
| FP16/BF16 (16-bit) | ~16 GB | Training, and the quality baseline. |
| FP8 (8-bit) | ~8 GB | The common production choice in 2026. Loss is usually tiny. |
| INT4 (4-bit) | ~4 GB | Laptops, one small GPU. You can see the quality drop. |

> **ShopBot on FP8.** Weights drop 16 GB → 8 GB, so two things improve at once:
>
> ```
> free KV space   62 GB  →  70 GB     chats fit:  185  →  209
> bytes per token  16 GB  →   8 GB     TPOT:      10 ms →  ~6 ms
> ```
>
> Both come from the same cause: fewer bytes to move. This is why quantization shows up as both a speed lever and a capacity lever.

Two rules:

1. **Always measure on your own task**, never on a public benchmark. The quality loss is uneven. A 4-bit ShopBot can still chat politely and *still* start getting the 30-day refund window wrong — which is the only thing that mattered.
2. Quantizing the **KV cache** is a *different lever* from quantizing the **weights**. When long context is your problem, the cache is often the better lever to pull.

## Context window ≠ usable context

Almost every top model says it has about 1M tokens in 2026. That number is a ceiling, not a promise.

- **Lost in the middle** — models pay good attention to the start and the end. The middle gets much less.
- **Attention dilution** — the attention budget is fixed. Spread it over more tokens and every token gets less.
- **Context rot** — accuracy drops fastest when the wrong bits *look like* the right one. That is exactly what a real set of documents looks like.

> **ShopBot's failure case.** Paste the full 400-page policy manual (~300,000 tokens) instead of retrieving 3 chunks. The manual holds **40 refund clauses** that all read almost the same — electronics, furniture, sale items, gift cards, damaged-in-transit. The right one for order #48291 is clause 31, sitting in the middle. The model quotes clause 12 instead and promises a refund the company will not honour.
>
> A "needle in a haystack" test would score 99% here, because a made-up sentence like *"the secret code is banana"* looks nothing like the rest. Forty near-identical refund clauses are the real test.

Public "needle in a haystack" tests hide this. Finding one odd sentence in a pile of unrelated text is much easier than finding the right one out of forty similar paragraphs.

---

# Part 5 — WHAT You Should Choose

### 1. Which metric are you optimising?

Decide this **before** you change anything. If you optimise the wrong one, you waste weeks.

| If your product is | Optimise | Ignore |
|---|---|---|
| Interactive chat | TTFT | Throughput |
| Batch / offline work | Throughput | TTFT, completely |
| Voice / realtime | TPOT | — |

> **ShopBot: TTFT.** A customer with a broken order is angry already. 150 ms feels instant; 2 s feels broken. Nobody has ever complained that a chatbot replied at 100 tokens/sec instead of 130.

### 2. Cut cost by 50% — five levers, in order

Always in this order. Cheapest and safest first.

```
1. Shorten the prompt            free        biggest win, always try first
2. Reorder for prefix caching    free        fixed text first, changing text last
3. Cap max_tokens on output      free        output costs several × input
4. Quantize weights to FP8       low risk    ~half the memory, small quality cost
5. Move to a smaller model       measure     often fine for narrow tasks
```

Only after all five: buy more GPUs.

> **ShopBot's bill, and why the order flips.** 2 million chats a month, at an example price of $0.50 per million input tokens and $1.50 per million output.
>
> ```
> input    2M chats × 2,500 tokens  =  5,000M  ×  $0.50/M  =  $2,500
> output   2M chats ×   120 tokens  =    240M  ×  $1.50/M  =    $360
>                                                            ───────
>                                                             $2,860
> ```
>
> **Input is 87% of the bill.** So lever 1 is huge here: cutting the system prompt from 1,800 to 900 tokens saves ~$900 a month, or 31%. And **lever 3 is nearly useless** — capping output could save at most $360 even if you deleted every reply.
>
> Now flip the app. A **code generator** takes 200 tokens in and writes 2,000 out. Its bill is ~$3,100 of output against $200 of input. There, lever 3 is the top lever and lever 1 barely registers.
>
> **The real rule: check your in/out ratio before you pick a lever.** The order above is a default, not a law.

### 3. Long context or retrieval?

| Situation | Choose |
|---|---|
| The text you need is small and you can find it. | **Retrieve** — 5k right tokens beat 500k random ones. |
| You really do need to reason over a whole document. | Long context — and measure quality honestly. |
| You are not sure. | Retrieve first. It is cheaper, faster, *and* usually more accurate. |

> **ShopBot: "why not just paste the whole policy manual?"** Because 300,000 tokens instead of 680 costs you all three things at once:
>
> ```
> TTFT        45 ms   →   ~18 s      the customer is gone
> KV cache   335 MB   →   39 GB      per user
> capacity   185 chats →   1 chat    the 80 GB GPU now serves one person
> quality    right clause → clause 12   see the failure case above
> ```
>
> Retrieval is not the cheap compromise here. It is faster, 440× cheaper, **and** more accurate.

"Just paste everything into the 1M window" is not a design. This is an argument about **quality**, not only cost — that is the case for RAG in [4_applied_ai](../4_applied_ai/README.md).

### 4. Which serving engine?

| Engine | Pick it when |
|---|---|
| **vLLM** | Default. Works with the most models and hardware, and is the easiest to run. |
| **SGLang** | Many users at once, with lots of shared prompt text. |
| **TensorRT-LLM** | You need the most speed from NVIDIA hardware and can handle the extra ops work. |
| **llama.cpp** | Laptops and small local machines. |

> **ShopBot: vLLM.** Later, if the system prompt grows to 5,000 shared tokens and traffic triples, SGLang's stronger prefix reuse becomes worth testing — but only after you have measured that prefix work is really where the time goes.

### 5. When to ignore this whole file

If you send under about 10,000 requests a month to an API, none of this pays off. Your time costs more than the tokens. Ship first, measure, then come back when the bill is real.

> **ShopBot at launch week.** 3,000 chats in month one = about $4 of tokens. Spending three days on prefix caching to save $1.30 is a bad trade. Ship it. Come back at 500,000 chats.

---

## Check Yourself

No notes, no AI. Say it out loud. Use ShopBot's numbers.

1. How many tokens does one run of the model produce? So how many runs does ShopBot's 120-token reply take? Why can't you do them all at once?
2. **Where** does the GPU memory go on a machine serving an 8B model? Name the three parts, and which one limits how many users fit.
3. What are the two phases of generation, and what runs out in each? Which one took 150 ms and which took 1,200 ms?
4. Write the KV-cache-per-token formula from memory. Apply it to Llama-3-8B.
5. One ShopBot chat is 2,620 tokens. How much KV cache is that, and how many chats fit in 62 GB?
6. Why does moving the 1,800-token system prompt to the front change your bill?
7. Pasting the 300,000-token manual breaks three things at once. Name all three.
8. ShopBot's input is 87% of its cost. Which of the five levers matters most, and which one is nearly useless?

---

## Quick Summary

| Term | In one line |
|---|---|
| Where it runs | A scheduler in front of a GPU. The scheduler is most of the magic. |
| GPU memory | Weights (fixed) + KV cache (grows with tokens × users). |
| One model run | Takes a list of tokens, gives back **one** token. That is all it does. |
| Prefill | **Run 1.** Reads the whole prompt at once. Sets TTFT. Runs out of math power. Like reading a page. |
| Decode | **Runs 2 to N.** One token at a time. Sets TPOT. Runs out of memory speed. Like writing a page by hand. |
| TTFT / TPOT | Time to first token / time per output token. ShopBot: 150 ms / 10 ms. |
| KV cache | Keep past K and V so you do not work them out again. Memory for compute. |
| 128 KB/token | Llama-3-8B. Remember the formula, not the number. |
| One ShopBot chat | 2,620 tokens = 335 MB of cache. 185 chats fill an 80 GB GPU. |
| GQA | Fewer KV heads → 4× smaller cache → 185 chats instead of 46. |
| Continuous batching | Free slots get refilled right away. |
| PagedAttention | KV cache in small blocks. Under 4% waste. 59 chats → 185. |
| Prefix caching | Fixed text first. TTFT 150 ms → 45 ms, for free. |
| Speculative decoding | Small model guesses, big model checks in one pass. 1.2 s → 0.5 s. |
| Quantization | Fewer bits per number → fewer bytes to move. FP8: 185 chats → 209. |
| Context rot | Quality drops on long input. Worst when the wrong bits look right. |
| First lever, always | Shorten the prompt — *if* input dominates your bill. Check the ratio first. |

## Next

[4_fine_tuning_and_alignment.md](4_fine_tuning_and_alignment.md) — when prompting is not enough and you change the model's weights instead.

## Sources

- [Inside vLLM: Anatomy of a High-Throughput LLM Inference System](https://vllm.ai/blog/2025-09-05-anatomy-of-vllm)
- [The State of Open-Source LLM Inference Engines in 2026](https://builderai.tools/blog/state-of-open-source-llm-inference-engines-2026)
- [LLM Serving Optimization: Continuous Batching, PagedAttention, and Chunked Prefill on H100 (2026)](https://www.spheron.network/blog/llm-serving-optimization-continuous-batching-paged-attention/)
- [How KV Caching Slashes LLM Inference Costs at Scale](https://www.digitalocean.com/community/conceptual-articles/how-kv-caching-slashes-llm-inference-costs-at-scale)
- [LLM Context Window Management and Long-Context Strategies 2026](https://zylos.ai/research/2026-01-19-llm-context-management/)
