# AI Foundations (Weeks 1–6)

The path: get a feel for the math → get a feel for ML → learn PyTorch ideas.

## What to Learn Deeply

**Mathematics** — don't memorize formulas, just build a feel for what's happening:
- **Vectors & embeddings** — *the problem:* how do you turn "meaning" into numbers a computer can multiply? *the answer:* place things in a space with many dimensions, where direction and distance show how similar they are. *when this matters:* any time you ask a model "how similar are these two things?"
- **Probability** — a model doesn't give one fixed answer. It gives a list of possible next words (or classes), each with a probability. Sampling, uncertainty, and how confident a model "should" be all come from this one idea.
- **Optimization & gradient intuition** — gradient descent is just: "which direction lowers the error, and by how much?" — repeated millions of times. Ideas like loss landscapes, local minima, and learning rate all come from this same simple idea.

**Machine Learning** — focus on *why* things work, not just *how* to use them:
- **Why XGBoost sometimes beats linear regression** — it can catch non-linear patterns and feature interactions that a linear model can't. The tradeoff: it's harder to interpret and can overfit more easily on small or noisy data.
- **Overfitting** — the model memorized noise in the training data instead of learning the real pattern. You spot it as a gap between training performance and validation performance.
- **Bias vs. variance** — bias means the model is too simple to capture the pattern. Variance means the model is too sensitive to the exact training data it saw. Every model sits somewhere between these two.
- **Feature engineering & evaluation** — a model is only as good as the data you give it and how honestly you measure its performance (using train/validation/test splits and the right metric for the problem).

**Deep Learning foundations** — the basic building blocks behind PyTorch code:
- **Why CNNs work** — convolution reuses the same weights across an image, so the network learns one "edge/texture/shape" detector instead of a separate one for every pixel.
- **Why residual connections matter** — they let gradients skip over layers. That's why networks with 100+ layers can train at all, instead of the gradient vanishing before it reaches the early layers.
- **Why LayerNorm exists** — it keeps values at a steady scale as they move through many layers, so training doesn't blow up or get stuck.
- **Why transformers replaced RNNs** — attention looks at all positions in a sequence at once, instead of one at a time. This removes the slow, step-by-step bottleneck RNNs had. More detail on this in [3_modern_ai](../3_modern_ai/README.md).

## How to Study This Phase

Follow the [Learning Loop](../README.md#the-learning-loop): understand → picture it in your head → explain it in your own words → read AI-generated PyTorch code (don't write it yourself first) → change it → break it on purpose and fix it → build something small (write gradient descent from scratch, or compare XGBoost vs. linear regression on one dataset) → teach it to someone else.

## Notebooks

- [gradient_descent_from_scratcwhh.ipynb](gradient_descent_from_scratch.ipynb) — builds gradient descent by hand (NumPy only, no autograd): loss, gradients, the training loop, and what happens when the learning rate is too small vs. too large.
- [xgboost_vs_linear_regression.ipynb](xgboost_vs_linear_regression.ipynb) — fits both models on the same non-linear dataset, showing linear regression's underfit, a well-tuned XGBoost, and a deliberately overfit XGBoost, to make bias vs. variance concrete.

Both end with a "Your turn" section — change the code, guess what will happen before you run it, then explain the idea back in your own words.

## Next

[3_modern_ai](../3_modern_ai/README.md) — transformers, attention, and how LLMs work under the hood build directly on the math and deep-learning basics above.
