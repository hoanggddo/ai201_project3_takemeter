# TakeMeter — r/smashbros Discourse Classifier

A fine-tuned text classifier that evaluates discourse quality in r/smashbros by distinguishing between analytical posts, opinion posts, and help requests.

---

## Community Choice

r/smashbros is the main Reddit community for the Super Smash Bros. series. It's a strong fit for a classification task because the discourse is genuinely varied: the same subreddit hosts high-level competitive discussion (matchup charts, frame data arguments, tournament breakdowns), casual opinion posts ("Min Min sucks"), and help requests from newer players ("how do I wavedash?"). That range means the labels capture something meaningful — a post like "Yuzha's Pikachu Matchup Chart" and a post like "We gotta stop pretending ult has archetypes" are both text-heavy takes on the game, but they're doing completely different things.

---

## Label Taxonomy

**`analysis`** — Makes a structured argument backed by specific mechanical, statistical, or competitive evidence. The claim is supported by verifiable details such as frame data, matchup numbers, tournament results, or named mechanics — not just asserted.

- Example 1: "Yuzha's Pikachu Matchup Chart (Offline/Online)" — shares a full matchup chart with specific character-by-character assessments grounded in competitive data.
- Example 2: "Advanced Smash: Comparing Roy and Chrom's frame data: Who actually has better aerials." — uses specific frame data to support a comparative claim.

**`opinion`** — States a bold or evaluative position without supporting evidence. The post asserts rather than argues — the claim might be correct, but no real reasoning is given.

- Example 1: "We gotta stop pretending ult has archetypes" — makes a structural claim about the game with no mechanical breakdown.
- Example 2: "Min Min Sucks Lol" — evaluative judgment with no supporting argument.

**`help`** — Asks for or offers practical gameplay advice with no evaluative claim about the game. The goal is to solve a specific problem or improve at a specific skill.

- Example 1: "So I suck at IRAR... can someone explain it to me like the idiot that I am" — explicit request for technique help.
- Example 2: "GC Controllers not working with Project M?" — technical troubleshooting with no opinion attached.

---

## Data Collection

**Source:** r/smashbros posts collected via the `sentence-transformers/reddit-title-body` dataset on HuggingFace, filtered to the smashbros subreddit. Additional analysis posts were collected manually from r/smashbros by searching for "matchup chart", "frame data", and "tier list".

**Labeling process:** Posts were pre-labeled using Groq's `llama-3.1-8b-instant` model with label definitions from planning.md provided as a system prompt. Every pre-assigned label was reviewed and corrected manually. Labels were also corrected using a secondary review pass with Claude. See the AI Usage section for full disclosure.

**Label distribution:**

| Label | Count |
|---|---|
| help | 231 |
| opinion | 95 |
| analysis | 59 |
| **Total** | **385** |

**Difficult-to-label examples:**

1. **"[Brawl] Zero Suit Samus approach options?"** — starts as a help request ("anyone have any better ideas?") but the body provides a detailed breakdown of each approach option with frame data reasoning and hitbox analysis. Labeled `analysis` because the post argues a position with specific evidence rather than just asking for advice. Decision rule: if removing the question framing still leaves a structured mechanical argument, it's analysis.

2. **"Character Discussion #3: Captain Falcon"** — structured as an open community discussion prompt but references tier placement and specific moves with competitive reasoning. Labeled `analysis` because it cites verifiable competitive details even though it invites responses. The deciding factor was whether the post body itself contains evidence, not just solicits it.

3. **"NEC: Your thoughts on trash talking?"** — initially labeled `help` by the pre-labeling model because it asks a question. Corrected to `opinion` because it solicits community opinions rather than practical advice — there is no problem to solve and no skill to improve.

---

## Fine-Tuning Approach

**Base model:** `distilbert-base-uncased` (HuggingFace)

**Training setup:** Fine-tuned on Google Colab T4 GPU using the HuggingFace `transformers` library. Dataset split: 70% train (269 examples), 15% validation (58 examples), 15% test (58 examples).

**Hyperparameters:** Default settings — 3 epochs, learning rate 2e-5, batch size 16. No adjustments were made from the starter notebook defaults.

**Hyperparameter decision:** The default learning rate of 2e-5 is standard for DistilBERT fine-tuning and appropriate for a small dataset. With only 269 training examples, a higher learning rate risked overfitting and a lower one would have been too slow to converge in 3 epochs.

---

## Baseline

**Model:** Groq `llama-3.3-70b-versatile` (zero-shot)

**Prompt approach:** The system prompt provided all three label definitions with one example post per label, exactly matching the definitions from planning.md. The model was instructed to output only the label name with no explanation.

**Prompt used:**
```
You are classifying posts from r/smashbros, the Super Smash Bros. subreddit.
Assign each post to exactly one of the following categories.

analysis: Makes a structured argument backed by specific mechanical, statistical,
or competitive evidence such as frame data, matchup numbers, or tournament results.
Example: "Yuzha's Pikachu Matchup Chart — here's my breakdown of every major
matchup with frame data and tournament results to back each rating."

opinion: States a bold or evaluative position WITHOUT supporting evidence.
The post asserts rather than argues.
Example: "We gotta stop pretending ult has archetypes — every character just
does the same neutral with different hitboxes."

help: Asks for or offers practical gameplay advice with no evaluative claim
about the game.
Example: "So I suck at IRAR and I can't get it consistent — can someone
explain it to me like the idiot that I am?"

Respond with ONLY the label name. Do not explain your reasoning.
Valid labels: analysis, opinion, help
```

---

## Evaluation Report

### Overall Accuracy

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Groq llama-3.3-70b) | **91.4%** |
| Fine-tuned DistilBERT | **70.7%** |

Fine-tuning produced a regression of 20.7 percentage points compared to the zero-shot baseline.

### Per-Class Metrics

**Fine-tuned DistilBERT:**

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| analysis | 1.00 | 0.78 | 0.88 | 9 |
| opinion | 0.00 | 0.00 | 0.00 | 15 |
| help | 0.67 | 1.00 | 0.80 | 34 |
| **macro avg** | 0.56 | 0.59 | 0.56 | 58 |

**Zero-shot baseline (Groq):**

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| analysis | 1.00 | 0.89 | 0.94 | 9 |
| opinion | 0.86 | 0.80 | 0.83 | 15 |
| help | 0.92 | 0.97 | 0.94 | 34 |
| **macro avg** | 0.92 | 0.89 | 0.90 | 58 |

### Confusion Matrix (Fine-tuned Model)

|  | Predicted: analysis | Predicted: opinion | Predicted: help |
|---|---|---|---|
| **True: analysis** | 7 | 0 | 2 |
| **True: opinion** | 0 | 0 | 15 |
| **True: help** | 0 | 0 | 34 |

The confusion matrix shows a single catastrophic failure: every `opinion` post in the test set was predicted as `help`. The model learned to predict `analysis` and `help` but completely abandoned `opinion`.

### Wrong Predictions Analysis

**Error 1 — "NEC: Your thoughts on trash talking?"** (True: opinion, Predicted: help, confidence 0.74)

This post asks "how much is too much?" in a question format. The model predicted `help` because the post is phrased as a question asking for community input — structurally similar to genuine help requests. The distinction that matters is whether the post is asking for practical advice (help) or soliciting opinions (opinion), but the model never learned this boundary.

**Error 2 — "Anyone else hate playing with items?" / "What up with the hate towards Brawl"** (True: opinion, Predicted: help)

Multiple opinion posts are phrased as "anyone else...?" or "why does...?" questions. These look like help requests to a model that learned surface-level question structure rather than the intent behind the question. The model has no way to distinguish "how do I wavedash?" (genuine help) from "anyone else think items ruin the game?" (soliciting opinions).

**Error 3 — "Character Discussion #3: Captain Falcon"** (True: analysis, Predicted: help, confidence 0.71)

This post is labeled `analysis` because it references tier placement and specific moves with competitive reasoning. The model predicted `help` — likely because the post ends with open questions ("What is his worst move? What makes him bad?") which pattern-match to help requests. The model is reacting to the question marks rather than the analytical content in the body.

### Sample Classifications

| Post (truncated) | Predicted Label | Confidence | Notes |
|---|---|---|---|
| "sssr's ROB matchup chart (Best Solo ROB in Japan)" | analysis | 0.96 | Correct — title pattern is unambiguous |
| "Can't perform short hops, any tips?" | help | 0.94 | Correct — clear technique question |
| "I strongly dislike whoever put pratfalling in Brawl." | help | 0.61 | Wrong — this is opinion; model confused by complaint phrasing |
| "Zomba's R.O.B. Matchup Chart" | analysis | 0.97 | Correct — matchup chart titles learned reliably |
| "Why is Jigglypuff in the top tier in SSBM?" | help | 0.78 | Correct by our label — phrased as question asking for explanation |

---

## What the Model Learned vs. What I Intended

The model learned two things reliably: matchup chart titles map to `analysis`, and technical gameplay questions map to `help`. It learned almost nothing about `opinion`.

What I intended was for the model to distinguish posts based on whether they make evidence-backed arguments, assert positions without evidence, or seek practical advice. What the model actually learned was a shallower heuristic: question marks and casual phrasing → `help`, structured titles with player names → `analysis`, everything else → also `help`.

The core problem is that `opinion` and `help` share a surface feature — both are often phrased as questions. "How do I beat camping Sonics?" and "Anyone else think Brawl's online is unplayable?" look similar at the token level. With 95 opinion examples and 231 help examples, the model learned to bet on `help` whenever it was uncertain, which turned out to be correct often enough to achieve 70% accuracy while completely ignoring `opinion`.

To fix this, the label definitions would need to be operationalized with more training signal — specifically, more opinion examples that are phrased as questions, so the model can learn that "anyone else think X?" and "how do I do X?" are different despite similar structure.

---

## Spec Reflection

**One way the spec helped:** The requirement to define a hard edge case before annotating forced me to think about the `opinion` vs `analysis` boundary before touching any data. The decision rule — "if removing the opinion framing still leaves a structured mechanical argument, it's analysis" — turned out to be the most-used rule during annotation and saved a lot of reclassification.

**One way implementation diverged:** The spec assumes fine-tuning will outperform the zero-shot baseline. In this project it didn't — the baseline was 20.7 points better. This happened because the dataset skews heavily toward `help` (60%) and `opinion` posts are often phrased as questions, making the boundary hard to learn from 95 examples. In hindsight, I would have collected more opinion examples that are clearly distinct in structure from help posts before training.

---

## AI Usage

**Instance 1 — Pre-labeling with Groq:** Used Groq's `llama-3.1-8b-instant` model to pre-label all 400 posts in the dataset. The system prompt included all three label definitions and a `delete` category for non-text posts. Every pre-assigned label was reviewed manually and corrected where wrong. The most common correction was `opinion` posts mislabeled as `help` due to question phrasing — approximately 15–20 labels were changed during review. The `pre_labeled` column in the dataset CSV tracks which rows were auto-labeled.

**Instance 2 — Label review and correction with Claude:** Used Claude to review the pre-labeled dataset and identify systematic mislabelings. Claude flagged that the ZSS approach options post should be `analysis` rather than `help`, that several "Character Discussion" posts should be `analysis`, and that community survey posts should be deleted rather than labeled. These corrections were applied before training. Claude also helped identify the pattern in wrong predictions — that `opinion` posts phrased as questions were being systematically misclassified as `help`.
