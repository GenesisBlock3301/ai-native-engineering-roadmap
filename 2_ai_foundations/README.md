# AI Foundations (Weeks 1–6)

Math intuition → ML intuition → PyTorch concepts.

## What to Learn Deeply

**Mathematics** — not formulas, intuition:
- **Vectors & embeddings** — *problem:* turning meaning into something a matrix multiply can operate on. *How:* direction and distance in high-dimensional space encode similarity. *When:* any time you're asking "how alike are these two things" to a model.
- **Probability** — models output a distribution over next tokens/classes, not a single answer; sampling, uncertainty, and calibration all follow from that one fact.
- **Optimization & gradient intuition** — gradient descent is "which direction reduces error, and by how much," repeated millions of times. Loss landscapes, local minima, and learning rate all fall out of that one idea.

**Machine Learning** — why, not just how:
- **Why XGBoost beats linear regression sometimes** — it captures non-linear interactions and feature splits a linear model can't, at the cost of interpretability and a higher risk of overfitting small or noisy data.
- **Overfitting** — the model memorized the training set's noise instead of the underlying pattern; shows up as a train/validation gap.
- **Bias vs. variance** — bias is a model too simple to capture the pattern; variance is a model too sensitive to the specific training sample. Every model sits somewhere on that trade-off.
- **Feature engineering & evaluation** — a model is only as good as what you feed it and how honestly you measure it (train/val/test split, the right metric for the problem).

**Deep Learning foundations** — the building blocks PyTorch code is made of:
- **Why CNNs work** — convolution shares weights across an image, so the network learns "edge/texture/shape" detectors once instead of per-pixel.
- **Why residual connections matter** — they let gradients skip layers, which is why 100+ layer networks can train at all instead of vanishing.
- **Why LayerNorm exists** — it keeps activations at a stable scale as they pass through many layers, so training doesn't blow up or stall.
- **Why transformers replaced RNNs** — attention looks at all positions at once instead of one at a time, removing the RNN's sequential bottleneck. Full depth on this in [3_modern_ai](../3_modern_ai/README.md).

## How to Study This Phase

Apply the [Learning Loop](../README.md#the-learning-loop): understand → visualize → explain it yourself → read AI-generated PyTorch code (don't write it first) → modify it → debug intentionally → build something small (a gradient descent from scratch, an XGBoost-vs-linear-regression comparison on one dataset) → teach it back.

## Next

[3_modern_ai](../3_modern_ai/README.md) — transformers, attention, and LLM internals build directly on the math and deep-learning foundations above.
