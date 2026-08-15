# Hugging Face — The Toolbox

## The Whole File in One Table

Up to now "the model" has been something behind an API. Hugging Face is where you open the box.

| | **The Hub** | **`transformers`** |
|---|---|---|
| **Why** | You need weights, configs and tokenizers from somewhere trustworthy. | Weights are just numbers. You need code that turns them into a running model. |
| **How** | Git repos that hold very large files. | `AutoModel` reads the config and builds the right class for you. |
| **Where** | huggingface.co → downloaded **once** to `~/.cache/huggingface` on your disk. | In your Python process — then on your CPU or GPU. |
| **Costs you** | Disk space and download time. An 8B model is ~16 GB. | Nothing to use. The cost is the GPU it needs to run. |

You do not need to memorise this library. You need to **read** code that uses it and know which piece does what.

---

## How To Read This File

```
Part 1   WHERE the files live      download → disk → GPU
Part 2   WHAT the pieces are       eight libraries, and when you touch each
Part 3   HOW to load a model       two doors, and where the logits are
Part 4   WHAT will bite you        security, v5 changes, chat templates
Part 5   WHAT you should choose    API or local, pipeline or AutoModel, which model
```

| Question you will be asked at work | Which part |
|---|---|
| "Why is this old tutorial's code failing?" | Part 4 |
| "Is it safe to download this model?" | Part 4 |
| "Should we self-host or stay on the API?" | Part 5 |
| "Will a 70B fit on our GPU?" | Part 5 |

Practice: [5_hugging_face_and_sampling.ipynb](code/5_hugging_face_and_sampling.ipynb).

---

# Part 1 — WHERE The Files Live

```
   huggingface.co
        │
        │  from_pretrained("meta-llama/Llama-3.1-8B-Instruct")
        │  downloads ONCE, then never again
        ▼
   ~/.cache/huggingface/hub/          ← your disk, ~16 GB
        │   config.json                 what shape is this model?
        │   model.safetensors           the actual numbers
        │   tokenizer.json              the box of pieces from file 2
        │
        │  loaded into
        ▼
   your Python process  ────►  GPU memory (16 GB in fp16)
```

Three things worth knowing:

1. **The download happens once.** The second run is instant. If your disk fills up, that cache is why — it never cleans itself.
2. **`config.json` is worth opening.** It tells you the layer count, hidden size and KV head count — the exact numbers you need for the KV cache formula in [3_llm_inference.md](3_llm_inference.md).
3. **Disk size ≈ GPU size.** A 16 GB download needs ~16 GB of GPU memory, *before* any KV cache.

---

# Part 2 — WHAT The Pieces Are

| Piece | What it is | When you touch it |
|---|---|---|
| **Hub** | huggingface.co — hosts models, datasets, demos | finding and downloading a model |
| **transformers** | loads and runs models | almost always |
| **tokenizers** | fast BPE and friends, written in Rust | comes along with transformers |
| **datasets** | load and stream training data without filling your disk | fine-tuning |
| **accelerate** | same code on CPU / one GPU / many GPUs | training on real hardware |
| **PEFT** | LoRA and QLoRA adapters | fine-tuning cheaply ([file 4](4_fine_tuning_and_alignment.md)) |
| **TRL** | SFT, DPO, GRPO trainers | alignment |
| **safetensors** | the weight file format | the default now — see Part 4 |

---

# Part 3 — HOW To Load a Model

There are two doors in. They differ in **what they hide**.

```
  DOOR 1 — pipeline()
  ┌────────────────────────────────────────────────┐
  │  tokenize → model → sample → detokenize        │   all hidden
  └────────────────────────────────────────────────┘
        you see:  text in ──────────────► text out


  DOOR 2 — AutoTokenizer + AutoModel
        tok(...)          →  you see the IDs
        model(...)        →  you see the LOGITS      ◄── the point
        your own sampling →  you write temperature/top-k yourself
        tok.decode(...)   →  you see the text
```

Compare that to Map A in [2_tokenization_and_embeddings.md](2_tokenization_and_embeddings.md). Door 2 is that map, with every arrow exposed.

**Door 1 — the front door.** One line, sensible defaults, good for a quick check.

```python
from transformers import pipeline
pipe = pipeline("text-generation", model="distilgpt2")
print(pipe("The capital of France is", max_new_tokens=20))
```

**Door 2 — the real door.**

```python
from transformers import AutoTokenizer, AutoModelForCausalLM
tok = AutoTokenizer.from_pretrained("distilgpt2")
model = AutoModelForCausalLM.from_pretrained("distilgpt2")

inputs = tok("The capital of France is", return_tensors="pt")
logits = model(**inputs).logits        # shape: (batch, tokens, vocab_size)
```

That `logits` tensor is the whole point of this phase. It holds **one score for every token in the vocabulary**, at every position — the 50,257 scores from file 2's Map A, now a real object you can print.

`softmax` turns the last row into probabilities. *How you pick from that row* is temperature, top-k and top-p — the knobs from [1_introduction](../1_introduction/1_llm_setting.md), now visible as code. You implement them by hand in the notebook.

## Reading a model name

`meta-llama/Llama-3.1-8B-Instruct`

```
meta-llama  /  Llama-3.1  -  8B  -  Instruct
    ↑              ↑         ↑         ↑
 who made it    family     size    trained to follow instructions
```

- **Base vs Instruct.** A *base* model only continues text — ask it a question and it may write more questions. An *Instruct* model has been through SFT and alignment ([file 4](4_fine_tuning_and_alignment.md)). For anything conversational you want Instruct. Grabbing the base model by accident is the classic first-day mistake.
- **Gated models** need you to accept a licence on the Hub and log in with a token.
- Suffixes like `GGUF`, `AWQ`, `GPTQ`, `FP8` mean someone already quantized it for a particular runtime.

---

# Part 4 — WHAT Will Bite You

## Security: `.bin` and `trust_remote_code` run code

The old `.bin` format is Python pickle, and **loading a pickle file runs code from that file**. Downloading a random `.bin` model is running a stranger's script on your machine.

`safetensors` is plain data. It cannot execute anything.

```
model.safetensors   →  just numbers            ✅ safe
model.bin           →  numbers + runnable code ⚠️  it runs on load
trust_remote_code=True  →  "run this repo's custom Python"  ⚠️  read it first
```

If a repo offers only `.bin`, that is a real security decision, not a formality.

## Transformers v5 (2026) — why old tutorials break

- **PyTorch only.** Every `TF*` and `Flax*` class is gone. `TFAutoModel`, `TFBertModel` no longer exist.
- **The CLI is `transformers`**, not `transformers-cli`.
- **No more "fast" vs "slow" tokenizers.** There is one tokenizer, on the Rust backend. `use_fast=True` is history.
- **Quantization is first-class** — loading in low precision is part of the normal path, not a bolt-on.
- **Simpler model files** — each model file focuses on the forward pass. Reading the source is genuinely doable now. Do it once; it is the best way to see attention in real code instead of in a diagram.

## Chat templates — do not hand-roll them

Every instruct model was trained with its own exact special-token layout ([file 2, Part 3](2_tokenization_and_embeddings.md)). Use the tokenizer's template:

```python
messages = [
    {"role": "system", "content": "You are a terse assistant."},
    {"role": "user", "content": "Why is the sky blue?"},
]
prompt = tok.apply_chat_template(messages, tokenize=False, add_generation_prompt=True)
```

Get it wrong and the model still answers — just noticeably worse, in a way that looks like the model being dumb rather than your string being wrong. Base models have no chat template at all, which is another way to tell base from instruct.

---

# Part 5 — WHAT You Should Choose

### 1. API or self-hosted?

| | **API** (OpenAI, Gemini, Bedrock…) | **Local / self-hosted** (transformers, vLLM) |
|---|---|---|
| Time to first working version | minutes | days |
| Cost shape | per token, scales with usage | fixed GPU cost, cheap at high volume |
| Best quality available | yes | close, and closing |
| Data leaves your network | yes | no |
| You control the version | no — models change under you | yes — pin it forever |
| Fine-tuning freedom | limited | complete |
| Ops burden | none | GPUs, memory, batching, monitoring |

**Default: start on an API.** Move to self-hosting when one of exactly four things is true:

```
1. Volume makes per-token pricing hurt
2. Data cannot leave your network
3. You need a fine-tuned model
4. You need a version that never changes under you
```

If none of those are true, self-hosting is a hobby, not a decision.

### 2. `pipeline` or `AutoModel`?

| You want to | Use |
|---|---|
| demo, smoke test, "does this model work at all" | `pipeline` |
| understand, measure, or control what happens | `AutoModel` |
| see logits, write your own sampling, time the KV cache | `AutoModel` |

### 3. Will this model fit on my GPU?

Rough sizing, in 16-bit: **bytes ≈ 2 × parameters.**

| Model | fp16 weights | 4-bit weights | + KV cache? |
|---|---|---|---|
| 8B | ~16 GB | ~4 GB | yes — see [file 3](3_llm_inference.md) |
| 70B | ~140 GB | ~35 GB | yes |

**The mistake:** budgeting only for weights. A 24 GB GPU holds an 8B model's 16 GB and then has 8 GB left for *every user's* KV cache. Do that arithmetic before you buy anything.

### 4. Base or Instruct?

| You are | Pick |
|---|---|
| building anything conversational | **Instruct** |
| fine-tuning from scratch on your own SFT data | Base |
| unsure | **Instruct** |

### 5. Which format to download?

| Situation | Pick |
|---|---|
| Any normal case | `safetensors`, fp16 |
| Laptop / small GPU | `GGUF` (llama.cpp) or 4-bit `AWQ`/`GPTQ` |
| Production serving on NVIDIA | `FP8` if the engine supports it |
| Only `.bin` is offered | treat as a security review, not a download |

---

## Check Yourself

No notes, no AI. Say it out loud.

1. **Where** does a downloaded model live on your machine, and how many times is it downloaded?
2. What is in `config.json` that you need for the KV cache formula?
3. What does `pipeline` hide that `AutoModel` shows you — and why do you care?
4. What is a logit, and how many are there per position?
5. Why is loading a `.bin` file a security decision?
6. Name the four reasons to leave an API for self-hosting.
7. You have a 24 GB GPU and an 8B model. How much is left for the KV cache?

---

## Quick Summary

| Thing | In one line |
|---|---|
| Where files live | downloaded once to `~/.cache/huggingface`, then loaded to GPU |
| `config.json` | layer count and KV heads — the numbers for your cache math |
| Hub | where models and datasets live |
| transformers v5 | PyTorch-only, one tokenizer, quantization built in |
| pipeline | one line, hides everything, for a quick check |
| AutoModel | full control, and where you can see the logits |
| logits | one score per vocabulary token — sampling happens here |
| Base vs Instruct | continues text vs follows instructions |
| Chat template | the tokenizer builds the layout — never do it yourself |
| safetensors | safe; `.bin` and `trust_remote_code` execute code |
| Size rule | fp16 bytes ≈ 2 × parameters, before any KV cache |
| API vs local | four reasons to switch — volume, privacy, fine-tuning, version pinning |

## Next

[6_practice_and_selfcheck.md](6_practice_and_selfcheck.md) — the drills and questions that prove you actually learned this phase.

## Sources

- [Transformers v5: Simple model definitions powering the AI ecosystem (Hugging Face)](https://huggingface.co/blog/transformers-v5)
- [Migrating to Transformers 5+ Guide](https://medium.com/@vici0549/migrating-to-transformers-5-guide-1e90058a7633)
- [Hugging Face releases](https://github.com/huggingface/transformers/releases)
