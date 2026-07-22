# LLM Sampling Settings

## Temperature

Temperature controls **how creative or how strict** the AI’s answer is.

* **Low temperature** → serious, factual, same answer every time
* **High temperature** → creative, fun, different answers

Think of **temperature as imagination level**.

**Prompt:** `Name one use of water`

* **Low temperature (0.2):**
  → “Water is used for drinking.”
* **High temperature (0.9):**
  → Water gives life, refreshes the soul, and dances in rivers.

---

## Top_p (Nucleus Sampling)

Top_p controls **how many word choices** the AI is allowed to use.

* **Low top_p** → chooses from *few safest words*
* **High top_p** → chooses from *many possible words*

Imagine picking words from a **small box vs a big box**.

**Prompt:** `Describe the sky`

* **Low top_p (0.2):**
  → The sky is blue.

* **High top_p (0.9):**
  → “The sky stretches endlessly with soft clouds floating like dreams.”

---

## Max Length

### What it does (simple)

Max length decides **how long the answer can be**.

* Small max length → short answer
* Large max length → detailed answer

It’s like saying: **Answer in 10 words” or “Answer in 200 words.**

**Prompt:** `Explain what a computer is`

* **Max length = short:**
  → A computer is a machine that processes data.

* **Max length = long:**
  → A computer is an electronic machine that takes input, processes data using programs, stores information, 
and produces output.”

---

## Stop Sequences

### What it does (simple)

Stop sequence tells the AI **when to stop writing**.

It’s like putting a **STOP sign 🚫** in writing.

**Prompt:**
`List fruits: Apple, Banana, Orange, STOP`

* AI stops when it sees **STOP**
* Output:
  → Apple
  → Banana
  → Orange

(No extra words after that)

---

## Frequency Penalty

### What it does (simple)

Frequency penalty stops the AI from **repeating the same word again and again**.

It tells the AI: **“Don’t repeat yourself.”**

**Prompt:** `Write a sentence about a dog`

* **Low-frequency penalty:**
  → “The dog is a dog that likes the dog food.”

* **High-frequency penalty:**
  → “The dog is a loyal animal that enjoys its food.”

---

## Presence Penalty


Presence penalty encourages the AI to **talk about new ideas**, not the same topic again.

It tells the AI: **Say something new.**

**Prompt:** `Talk about school`

* **Low presence penalty:**
  → School has students. Students go to school to study.

* **High presence penalty:**
  → “School includes learning, friendships, teachers, activities, and personal growth.”

---

## Final One-Line Summary for Students

| Setting           | One-Line Meaning                 |
|-------------------|----------------------------------|
| Temperature       | How creative the answer is       |
| Top_p             | How many word choices AI can use |
| Max Length        | How long the answer can be       |
| Stop Sequence     | When the answer must stop        |
| Frequency Penalty | Stop repeating words             |
| Presence Penalty  | Encourage new ideas              |

---