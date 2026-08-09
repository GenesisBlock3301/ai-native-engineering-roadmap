# Hugging Face — The Toolbox

Up to now in this roadmap, "the model" has been something behind an API. Hugging Face is where you open the box: download the weights, read the config, look at the logits, and run it yourself.

You do not need to memorise this library. You need to be able to **read** code that uses it, and know which piece does what.

Practice for this file: [5_hugging_face_and_sampling.ipynb](code/5_hugging_face_and_sampling.ipynb).

---

## The Pieces

| Piece | What it is | When you touch it |
|---|---|---|
| **Hub** | huggingface.co — hosts models, datasets, and demos | finding and downloading a model |
| **transformers** | loads and runs models | almost always |
| **tokenizers** | fast BPE and friends, written in Rust | comes along with transformers |
| **datasets** | load and stream training data without filling your disk | fine-tuning |
| **accelerate** | runs the same code on CPU / one GPU / many GPUs | training on real hardware |
| **PEFT** | LoRA and QLoRA adapters | fine-tuning cheaply |
| **TRL** | SFT, DPO, GRPO trainers | alignment |
| **safetensors** | the weight file format | it's just the default now — see the warning below |

**Why safetensors matters:** the old `.bin` format is Python pickle, and loading a pickle file **runs code from that file**. Downloading a random `.bin` model from the internet is running a stranger's script on your machine. safetensors is plain data — it cannot execute anything. If a repo only offers `.bin`, that is a real security decision, not a formality. Same story for `trust_remote_code=True`: it means "run this repo's custom Python". Read it before you set it.

---

## What Changed in Transformers v5 (2026)

If you read an old tutorial and the code doesn't run, this is usually why:

- **PyTorch only.** Every `TF*` and `Flax*` class is gone. `TFAutoModel`, `TFBertModel` no longer exist.
- **The CLI is `transformers`,** not `transformers-cli`.
- **No more "fast" vs "slow" tokenizers.** There is one tokenizer, on the Rust backend. `use_fast=True` is history.
- **Quantization is first-class** — loading in low precision is part of the normal loading path, not a bolt-on.
- **Simpler model files** — each model file focuses on the forward pass, so reading the source is genuinely doable now. Do that at least once; it is the best way to see attention in real code rather than in a diagram.

---

## Two Ways In: `pipeline` vs `AutoModel`

**`pipeline` — the front door.** One line, sensible defaults, good for a quick check.

```python
from transformers import pipeline
pipe = pipeline("text-generation", model="distilgpt2")
print(pipe("The capital of France is", max_new_tokens=20))
```

**`AutoTokenizer` + `AutoModel` — the real door.** More lines, but you can see and change every step: tokens in, logits out, your own sampling, your own KV cache handling.

```python
from transformers import AutoTokenizer, AutoModelForCausalLM
tok = AutoTokenizer.from_pretrained("distilgpt2")
model = AutoModelForCausalLM.from_pretrained("distilgpt2")

inputs = tok("The capital of France is", return_tensors="pt")
logits = model(**inputs).logits        # shape: (batch, tokens, vocab_size)
```

That `logits` tensor is the whole point of this phase. It holds one score for **every token in the vocabulary**, for every position. `softmax` turns the last row into probabilities, and *how you pick from that row* is temperature, top-k, and top-p — the settings you learned as knobs in [1_introduction](../1_introduction/1_llm_setting.md), now visible as code. You implement them by hand in the notebook.

**Use `pipeline`** for a demo or a smoke test. **Use `AutoModel`** when you need to understand, measure, or control what happens.

---

## Reading a Model Name

`meta-llama/Llama-3.1-8B-Instruct`

```
meta-llama  /  Llama-3.1  -  8B  -  Instruct
    ↑             ↑          ↑         ↑
  who made it   family     size    trained to follow instructions
```

Things worth knowing:

- **Base vs Instruct.** A *base* model only continues text — ask it a question and it may write more questions. An *Instruct* (or *Chat*) model has been through SFT and alignment. For anything conversational, you want Instruct. Picking the base model by accident is a common first-day confusion.
- **Size ≈ memory.** In 16-bit, a model needs roughly `2 × parameters` in bytes: 8B ≈ 16 GB, 70B ≈ 140 GB — *before* the KV cache from [3_llm_inference.md](3_llm_inference.md). At 4-bit, roughly `0.5 ×`: 8B ≈ 4 GB.
- **Gated models** need you to accept a licence on the Hub and log in with a token.
- Suffixes like `GGUF`, `AWQ`, `GPTQ`, `FP8` mean someone already quantized it for a particular runtime.

---

## Chat Templates — Do Not Hand-Roll Them

Every instruct model was trained with its own exact special-token layout. Use the tokenizer's template instead of gluing strings together:

```python
messages = [
    {"role": "system", "content": "You are a terse assistant."},
    {"role": "user", "content": "Why is the sky blue?"},
]
prompt = tok.apply_chat_template(messages, tokenize=False, add_generation_prompt=True)
```

Get this wrong and the model still answers — just noticeably worse, in a way that looks like the model being dumb rather than your string being wrong. Base models have no chat template at all, which is another way to tell base from instruct.

---

## Local Model or API? (The Architect's Call)

| | **API** (OpenAI, Gemini, Bedrock…) | **Local / self-hosted** (transformers, vLLM) |
|---|---|---|
| Time to first working version | minutes | days |
| Cost shape | per token, scales with usage | fixed GPU cost, cheap at high volume |
| Best quality available | yes | close, and closing |
| Data leaves your network | yes | no |
| You control the version | no — models change under you | yes — pin it forever |
| Fine-tuning freedom | limited | complete |
| Ops burden | none | GPUs, memory, batching, monitoring |

**Sensible default:** start on an API. Move to self-hosting when one of four things is true — volume makes per-token pricing hurt, data cannot leave, you need a fine-tuned model, or you need a version that never changes.

Note what your own repo already does: [1_introduction](../1_introduction/basic_prompt.ipynb) uses the Gemini API for prompting practice, and this phase uses a small local model to see the internals. That is the right split — APIs to build with, local models to learn from.

---

## Quick Summary

| Thing | In one line |
|---|---|
| Hub | where models and datasets live |
| transformers v5 | PyTorch-only, simpler tokenizers, quantization built in |
| pipeline | one line, for a quick check |
| AutoModel | full control, and where you can see the logits |
| logits | one score per vocabulary token — sampling happens here |
| Base vs Instruct | continues text vs follows instructions |
| Chat template | the tokenizer builds the special-token layout — never do it yourself |
| safetensors | safe weight format; `.bin` and `trust_remote_code` run code |
| API vs local | speed and quality vs control, privacy, and volume cost |

## Next

[6_practice_and_selfcheck.md](6_practice_and_selfcheck.md) — the drills and questions that prove you actually learned this phase.

## Sources

- [Transformers v5: Simple model definitions powering the AI ecosystem (Hugging Face)](https://huggingface.co/blog/transformers-v5)
- [Migrating to Transformers 5+ Guide](https://medium.com/@vici0549/migrating-to-transformers-5-guide-1e90058a7633)
- [Hugging Face releases](https://github.com/huggingface/transformers/releases)
