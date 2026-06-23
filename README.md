# ai201-project3-takemeter

AI201 — Project 3. **TakeMeter** is an LLM-based text classifier that scores the
*substance of a single forum post* in a tech-help community, sorting each post into
one of three labels. This repo holds everything outside the Colab notebook: the
planning doc, the labeled dataset, and the evaluation artifacts downloaded from Colab.

- **GitHub repository:** https://github.com/LittlePixels/ai201-project3-takemeter
- **Colab notebook:** https://colab.research.google.com/drive/1sELdFd-rB389M6AglXpi6KK_W7tMrn3E?usp=sharing
- **Planning doc (written before data collection, updated before stretch features):** [planning.md](planning.md)
- **Labeled dataset:** [data/labeled_dataset.csv](data/labeled_dataset.csv) — 208 posts

> ⚠️ **Label naming note:** the label strings in the dataset are `analysis`,
> `hot_take`, and `reaction` (inherited from the notebook's `LABEL_MAP` template).
> In *this* project they map to forum-substance categories, **not** their everyday
> meanings: `analysis` = a substantive helpful answer, `hot_take` = a help request,
> `reaction` = low-value noise. The definitions below are the source of truth.

---

## Community choice and reasoning

**BleepingComputer — [Network Streaming Devices forum](https://www.bleepingcomputer.com/forums/f/230/network-streaming-devices/)**

I chose this forum because it is a long-running, public, volunteer-run tech-help
community where the *quality of a contribution* is the thing members actually care
about — which makes "substance" a real, labelable signal rather than something I had
to invent.

It's a good fit for classification because the discourse is genuinely varied:

- **Mixed intents in one thread** — a single thread interleaves the original problem,
  deep troubleshooting replies, bare one-line suggestions, off-topic banter,
  moderator housekeeping, and product promotion.
- **Full quality range** — replies span from excellent (exact device/firmware
  identification with a manual link; step-by-step instructions with screenshots) to
  near-worthless ("Fire Stick or NVIDIA Shield." / "Wow." / "very helpful comment,
  thanks!").
- **The community self-polices substance** — moderators openly call out *"promotion of
  a product without needed explanation,"* confirming the good-vs-noise distinction is
  recognized by regulars.

Full reasoning in [planning.md §1](planning.md).

---

## Label taxonomy

Labels are assigned along **intent + substance**. Tiebreaker for a short reply that
*does* respond: "does it convey specific actionable info or a reason?" — yes →
`analysis`, no → `reaction`, regardless of length.

### `analysis` — substantive helpful answer
A reply that materially helps the asker via specific troubleshooting steps, a
technical explanation, a *justified* recommendation, or relevant firsthand experience
with reasoning.
- *Example (id 22):* "That's a MAGNA HIFI MANO ULTRA MKII running piCorePlayer v8.0.0 with Squeezelite v1.9.9-1430-pCP. Here's the manual link for the advanced configuration so you can check the service startup order."
- *Example (id 155):* "I had the exact same problem. I bought a 10/100/1000 switch, hooked both the TV and the PC to it, and the problem went away."

### `hot_take` — help request
A post whose purpose is to obtain assistance: describing a problem, asking how to do
something, or requesting a recommendation (includes the asker's own follow-ups that
add detail while still seeking resolution).
- *Example (id 1):* "WHICH streaming device to replace Roku? ... I need a remote with power/volume/mute and 30-second skip ... do they support commercial skipping on recorded shows?"
- *Example (id 18):* "I'm getting intermittent UPnP streaming problems on Windows 11 ... One time it works fine for hours, but after the next boot it does not work at all."

### `reaction` — low-value noise
A reply (or non-help post) that adds little actionable value: vague one-liners,
"look it up yourself" deflections, bare thanks/agreement, off-topic venting,
moderator/thread housekeeping, or unexplained self-promotion/spam.
- *Example (id 39):* "very helpful comment, i read it, thanks for sharing!"
- *Example (id 37):* "Using express vpn, fast and reliable and also safe, dont go for free vpn." (an unexplained product push; a moderator flagged it as promotion)

---

## Data collection & labeling

- **Source:** Public posts/replies from the BleepingComputer Network Streaming Devices
  forum, collected thread-by-thread across the forum index pages (31 threads).
- **Labeling process:** Each post was **pre-labeled by an LLM (Claude)** against the
  definitions above, then **reviewed and finalized by me**; I made the final call on
  every row using the §3 edge-case rules in [planning.md](planning.md). (See
  [AI usage](#ai-usage) for disclosure.)
- **Size:** 208 labeled posts in a single CSV (not pre-split).

### Label distribution

| Label | Meaning | Count | Share |
|---|---|---:|---:|
| `analysis` | substantive helpful answer | 86 | 41.3% |
| `hot_take` | help request | 70 | 33.7% |
| `reaction` | low-value noise | 52 | 25.0% |
| **Total** | | **208** | **100%** |

No class exceeds 70% of the data; `reaction` is the minority class (≈25%), which is
why the evaluation leans on macro-averaged and per-class metrics rather than accuracy.

### Three difficult-to-label examples and my decisions

1. **id 5** — *"NVIDIA Shield is the best option and the Fire Series would be next.
   Bear in mind some of these service issues can follow you regardless of hardware."*
   → **`analysis`** (vs `reaction`). A bare ranking alone would be `reaction` (cf. id 3,
   "Fire Stick or NVIDIA Shield.", labeled `reaction`), but the caveat is a concrete
   reason, and my tiebreaker says one real reason tips a reply to `analysis`.
2. **id 24** — *"This is just scummy what Roku did ... The timing is suspicious."*
   → **`hot_take`** (vs `reaction`). Reads like venting, but it's a thread opener that
   invites discussion and drew ten substantive replies, so it's treated as a request
   for input rather than noise.
3. **id 108** — *"I solved it — instead of stopping the program ... I change the channel
   and mute. It keeps playing, never sleeps."* → **`hot_take`** (vs `analysis`). The
   content is a working solution, but I classify by author role: this is the original
   asker resolving their own thread, not another member helping them.

(A fourth, id 84, is documented in [planning.md §3](planning.md).)

---

## Fine-tuning approach

<!-- FILL FROM NOTEBOOK: confirm these match what you actually ran -->
- **Base model:** _TBD — e.g. `gpt-4o-mini-2024-07-18` (OpenAI fine-tuning) / DistilBERT (HF)_
- **Task framing:** single-label classification of one post into `analysis` / `hot_take` / `reaction`.
- **Train/validation split:** stratified, 70/30 (`test_size=0.3`), random_state fixed for reproducibility.
- **Training setup:** _TBD — number of epochs, batch size, learning-rate multiplier / LR._
- **Hyperparameter decision (at least one, with rationale):** _TBD — e.g. "Set epochs = 3 rather than the default 1 because the dataset is small (208 rows); 1 epoch underfit the minority `reaction` class in a trial run, while >4 began to overfit (validation loss rose). 3 was the best validation-loss point."_

---

## Baseline

<!-- FILL FROM NOTEBOOK -->
- **Method:** zero-/few-shot prompting of the base model (no fine-tuning) on the same test split.
- **Prompt used:**
  ```
  TBD — paste the exact baseline prompt here, including the label definitions
  and the output format you required (e.g. "respond with exactly one of:
  analysis, hot_take, reaction").
  ```
- **How results were collected:** _TBD — ran each test post through the prompt, parsed
  the model's single-label output, compared to the gold label; same test set as the
  fine-tuned model so the two are directly comparable._

---

## Evaluation report

> **Status: pending notebook run.** The tables below are templated; fill from
> `results/evaluation_results.json` and the confusion matrix once exported. Both files
> should be committed to [results/](results/).

### Headline metrics — both models

| Model | Accuracy | Macro-F1 | `analysis` F1 | `hot_take` F1 | `reaction` F1 |
|---|---:|---:|---:|---:|---:|
| Baseline (prompted) | _TBD_ | _TBD_ | _TBD_ | _TBD_ | _TBD_ |
| Fine-tuned | _TBD_ | _TBD_ | _TBD_ | _TBD_ | _TBD_ |

### Per-class metrics — fine-tuned model

| Label | Precision | Recall | F1 | Support |
|---|---:|---:|---:|---:|
| `analysis` | _TBD_ | _TBD_ | _TBD_ | _TBD_ |
| `hot_take` | _TBD_ | _TBD_ | _TBD_ | _TBD_ |
| `reaction` | _TBD_ | _TBD_ | _TBD_ | _TBD_ |

### Confusion matrix — fine-tuned model (rows = actual, cols = predicted)

| actual ↓ / pred → | `analysis` | `hot_take` | `reaction` |
|---|---:|---:|---:|
| **`analysis`** | _TBD_ | _TBD_ | _TBD_ |
| **`hot_take`** | _TBD_ | _TBD_ | _TBD_ |
| **`reaction`** | _TBD_ | _TBD_ | _TBD_ |

Image version: [results/confusion_matrix.png](results/confusion_matrix.png) _(commit once downloaded)_.

### Three wrong predictions, with analysis

<!-- FILL FROM NOTEBOOK: pick 3 actual misclassifications -->
1. **Post:** _TBD_ — **actual** _X_, **predicted** _Y_. _Why it likely erred (e.g. bare recommendation that sits on the `analysis`/`reaction` boundary)._
2. **Post:** _TBD_ — **actual** _X_, **predicted** _Y_. _Analysis._
3. **Post:** _TBD_ — **actual** _X_, **predicted** _Y_. _Analysis._

### Sample classifications (3–5 posts)

| Post (truncated) | Predicted | Confidence | Correct? |
|---|---|---:|:--:|
| _TBD_ | _TBD_ | _TBD_ | _TBD_ |
| _TBD_ | _TBD_ | _TBD_ | _TBD_ |
| _TBD_ | _TBD_ | _TBD_ | _TBD_ |

**One correct example explained:** _TBD — e.g. "Post id 22 (exact device + manual link)
was predicted `analysis` with 0.97 confidence; the model correctly keyed on the
specific firmware version and actionable link, exactly the signal the label is for."_

---

## Reflection: what the model learned vs. what I intended

<!-- FILL after evaluation; framing to complete with real results -->
I intended the model to learn the **substance distinction** — that a post is `analysis`
only when it carries specific, actionable information or a justified reason. What it
actually learned: _TBD — compare against the confusion matrix. Hypothesis to confirm:
the model leans on surface cues (length, presence of links/product names) and so
confuses bare recommendations (`reaction`) with justified ones (`analysis`), the same
boundary I flagged as hardest in planning._

---

## Spec reflection

- **One way the spec helped:** forcing me to define edge-case handling rules *before*
  annotating (Milestone 2) meant the 208 labels were applied consistently — when I hit
  ambiguous posts like id 5 and id 24, I already had a rule, so I didn't drift
  mid-dataset.
- **One way implementation diverged, and why:** _TBD once notebook is final — e.g. the
  spec's `LABEL_MAP` template used `analysis/hot_take/reaction`; rather than rename the
  notebook map to my forum labels, I renamed the labels to match the template to avoid
  reworking notebook cells, accepting that the names no longer describe their meaning
  (documented at the top of this README)._

---

## AI usage

This project used AI tooling at three points; specific directed instances:

1. **Data gathering + pre-labeling (annotation assistance — disclosed).** I directed
   Claude to fetch posts thread-by-thread from the forum and pre-label each against my
   definitions. **What I revised/overrode:** I reviewed every row and overrode the
   model on boundary cases — e.g. I kept asker self-resolutions (id 108) as `hot_take`
   rather than `analysis`, and demoted bare product name-drops (id 3) to `reaction`.
   The dataset is therefore **AI-pre-labeled, human-reviewed**; the final held-out test
   labels are human-confirmed.
2. **Label stress-testing.** I directed Claude to generate boundary posts between
   `analysis` and `reaction` to test whether my definitions were tight enough. **What I
   changed:** _TBD — note any definition tightening this produced, or "definitions held
   up; no change needed."_
3. **Repo + docs scaffolding.** I directed Claude to set up the GitHub repo, planning.md
   structure, and this README. **What I revised/overrode:** I chose a public repo,
   chose to rename labels to match the notebook template, and corrected the label
   distribution wording.

_(Failure-analysis assistance is planned per [planning.md §7.3](planning.md): the list
of wrong predictions will be handed to an AI to cluster error patterns, with each
pattern verified against the confusion matrix before it's written up above.)_

---

## Repository contents

| Path | Description |
|---|---|
| [planning.md](planning.md) | Plan: community, labels, edge cases, data plan, metrics, success criteria, AI tool plan. |
| [data/labeled_dataset.csv](data/labeled_dataset.csv) | 208 hand-reviewed labeled posts (`id`, `text`, `label`). |
| [results/evaluation_results.json](results/evaluation_results.json) | Metrics exported from Colab _(pending)_. |
| [results/confusion_matrix.png](results/confusion_matrix.png) | Confusion matrix exported from Colab _(pending)_. |
