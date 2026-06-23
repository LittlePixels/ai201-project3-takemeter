# ai201-project3-takemeter

AI201 — Project 3. **TakeMeter** is a text classifier that scores the *substance of a
single forum post* in a tech-help community, sorting each post into one of three
labels. This repo holds everything outside the Colab notebook: the planning doc, the
labeled dataset, the fine-tuning/evaluation notebook, and the result artifacts.

- **GitHub repository:** https://github.com/LittlePixels/ai201-project3-takemeter
- **Colab notebook:** https://colab.research.google.com/drive/1sELdFd-rB389M6AglXpi6KK_W7tMrn3E?usp=sharing
- **Planning doc (written before data collection, updated before stretch features):** [planning.md](planning.md)
- **Labeled dataset:** [data/labeled_dataset.csv](data/labeled_dataset.csv) — 208 posts
- **Notebook (committed copy):** [Another_copy_of_ai201_project3_takemeter_starter_clean.ipynb](Another_copy_of_ai201_project3_takemeter_starter_clean.ipynb)

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
  identification with a manual link; step-by-step instructions) to near-worthless
  ("Fire Stick or NVIDIA Shield." / "Wow." / "very helpful comment, thanks!").
- **The community self-polices substance** — moderators openly call out *"promotion of
  a product without needed explanation,"* confirming the distinction is recognized by
  regulars.

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
  definitions above, then **reviewed and finalized by me** using the §3 edge-case rules
  in [planning.md](planning.md). (See [AI usage](#ai-usage) for disclosure.)
- **Size:** 208 labeled posts in a single CSV (not pre-split).

### Label distribution

| Label | Meaning | Count | Share |
|---|---|---:|---:|
| `analysis` | substantive helpful answer | 86 | 41.3% |
| `hot_take` | help request | 70 | 33.7% |
| `reaction` | low-value noise | 52 | 25.0% |
| **Total** | | **208** | **100%** |

No class exceeds 70%; `reaction` is the minority class (≈25%), which is why the
evaluation leans on macro-averaged and per-class metrics rather than accuracy.

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

- **Base model:** `distilbert-base-uncased` (DistilBERT) with an
  `AutoModelForSequenceClassification` head sized to 3 labels. Run on a Colab T4 GPU.
- **Task framing:** single-label classification of one post into `analysis` /
  `hot_take` / `reaction`; text tokenized with `max_length=256`, truncation on.
- **Data split:** stratified **70 / 15 / 15** (`random_state=42`) → **train 145,
  validation 31, test 32**. The test set is locked and used only for the final report.
- **Training setup (`TrainingArguments`):** 3 epochs · train batch size 16 · eval batch
  size 32 · learning rate `2e-5` · weight decay `0.01` · `warmup_steps=50` ·
  eval & save each epoch · `load_best_model_at_end=True` selecting on validation
  accuracy.
- **Hyperparameter decision (with rationale):** I kept **`num_train_epochs = 3`**. For a
  dataset this small (145 training rows) more epochs risk memorizing the training set,
  while 1–2 tends to underfit; 3 with a `2e-5` learning rate and 50 warmup steps is the
  standard stable choice for BERT-family fine-tuning at this scale, and
  `load_best_model_at_end` guards against picking a later, overfit checkpoint. **In
  hindsight this was too conservative for the minority class** — see the evaluation and
  reflection below, where `reaction` collapsed to zero recall.

---

## Baseline

- **Method:** zero-shot prompting of Groq **`llama-3.3-70b-versatile`**
  (`temperature=0`, `max_tokens=20`) over the same 32-post test set, parsing the
  model's reply to one of the three label strings.
- **Intended prompt** (the prompt I wrote, defining the task + each label + one example,
  and requiring the model to output only the label name):
  ```
  You are classifying posts from a tech discussion forum.
  Assign each post to exactly one of the following categories.

  analysis:  a reply that materially helps — specific steps, a technical
             explanation, a justified recommendation, or firsthand experience.
  hot_take:  a post seeking help — a problem description or a request for advice.
  reaction:  a low-value reply — vague one-liners, bare thanks, venting,
             housekeeping, or unexplained promotion.

  (one example post per label)
  Respond with ONLY the label name. Do not explain your reasoning.
  Valid labels: analysis, hot_take, reaction
  ```
- **⚠️ Baseline did not produce valid results.** In the run captured in the notebook,
  the Section 5 prompt cell still held the **placeholder** template
  (`<label_1>/<label_2>/<label_3>`), which overwrote the real prompt before the
  classification loop ran. Groq dutifully echoed `<label_2>` / `<label_1>`, none of
  which match a label string, so **all 32 responses were unparseable → baseline
  accuracy = `nan` (0/32 parseable)**. The comparison below therefore has no valid
  baseline yet. **Fix:** put the real prompt above into the Section 5 `SYSTEM_PROMPT`
  cell (so it isn't re-overwritten by the placeholder cell) and re-run Section 5.

---

## Evaluation report

> Fine-tuned numbers below are the real notebook outputs. Baseline is `nan` because it
> did not run (see above). `results/evaluation_results.json` and
> `results/confusion_matrix.png` should be downloaded from Colab and committed to
> [results/](results/) after the baseline is re-run.

### Headline metrics — both models

| Model | Accuracy | Macro-F1 | `analysis` F1 | `hot_take` F1 | `reaction` F1 |
|---|---:|---:|---:|---:|---:|
| Zero-shot baseline (Groq llama-3.3-70b) | `nan`¹ | `nan`¹ | `nan`¹ | `nan`¹ | `nan`¹ |
| Fine-tuned DistilBERT | **0.500** | **0.38** | 0.58 | 0.56 | 0.00 |

¹ 0/32 responses parseable — invalid run, see Baseline section.

### Per-class metrics — fine-tuned model (test set, n=32)

| Label | Precision | Recall | F1 | Support |
|---|---:|---:|---:|---:|
| `analysis` | 0.64 | 0.54 | 0.58 | 13 |
| `hot_take` | 0.43 | 0.82 | 0.56 | 11 |
| `reaction` | 0.00 | 0.00 | 0.00 | 8 |
| **macro avg** | 0.35 | 0.45 | 0.38 | 32 |
| **weighted avg** | 0.41 | 0.50 | 0.43 | 32 |

### Confusion matrix — fine-tuned model (rows = actual, cols = predicted)

| actual ↓ / pred → | `analysis` | `hot_take` | `reaction` |
|---|---:|---:|---:|
| **`analysis`** | 7 | 6 | 0 |
| **`hot_take`** | 2 | 9 | 0 |
| **`reaction`** | 2 | 6 | 0 |

The whole **`reaction` column is zero** — the model never once predicted `reaction`, so
every one of the 8 noise posts was misrouted (recall 0.00). It over-predicts `hot_take`
(21 of 32 predictions). Image version: [results/confusion_matrix.png](results/confusion_matrix.png) _(commit after re-run)_.

### Three wrong predictions, with analysis

1. **"Look at the review articles on CNET and PCMag and figure out your own
   specifications from there."** — actual **`reaction`**, predicted **`analysis`**
   (conf 0.36). This is a deflection ("go research it yourself"), but it name-drops
   tech sources (CNET, PCMag, "specifications"), and the model keyed on those topical
   tokens as if they signalled substance. This is exactly the `analysis`↔`reaction`
   boundary I flagged as hardest in planning.
2. **"There are serious latency issues in 1809 and 1903 that cause these stutters. It
   needs a fix from Microsoft but there's no word on when."** — actual **`analysis`**,
   predicted **`hot_take`** (conf 0.35). A genuinely substantive explanation, misread
   as a help request. The near-chance confidence (0.35) shows the model isn't actually
   distinguishing "explaining a cause" from "asking about a problem."
3. **"Topic closed, spam magnet."** — actual **`reaction`**, predicted **`hot_take`**
   (conf 0.35). Moderator housekeeping with no help intent. Because the model never
   predicts `reaction` at all, short noise like this has nowhere to go and lands in the
   majority-ish `hot_take` bucket.

**Dominant pattern (verified against the confusion matrix):** total `reaction` collapse
(0 predicted, recall 0.00) plus `hot_take` over-prediction. Confidences cluster at
0.35–0.37 across *all* errors — barely above the 1/3 random baseline — indicating the
model underfit rather than learned a sharp boundary.

### Sample classifications (fine-tuned model)

| Post (truncated) | Predicted | Confidence | Correct? |
|---|---|---:|:--:|
| "Look at the review articles on CNET and PCMag …" | `analysis` | 0.36 | ❌ (actual `reaction`) |
| "There are serious latency issues in 1809 and 1903 …" | `hot_take` | 0.35 | ❌ (actual `analysis`) |
| "Topic closed, spam magnet." | `hot_take` | 0.35 | ❌ (actual `reaction`) |
| "For what it's worth I haven't had any real issues with a FireStick Lite …" | `hot_take` | 0.35 | ❌ (actual `analysis`) |
| "Using express vpn, fast and reliable … dont go for free vpn." | `analysis` | 0.37 | ❌ (actual `reaction`) |

**One correct example explained:** the model's one real strength is `hot_take` recall
(0.82 — it caught 9 of 11 help requests), so a clear question like *"WHICH streaming
device to replace Roku?"* is correctly classified `hot_take`: such posts have an
unambiguous interrogative/request shape the model learned well even from 145 examples.
_(Exact per-post confidences for correctly classified items weren't printed in the
notebook run; to populate this row with a real confidence value, add a "correct
predictions" print mirroring the wrong-predictions cell and re-run — see note below.)_

---

## Reflection: what the model learned vs. what I intended

I intended the model to learn the **substance distinction** — that a post is `analysis`
only when it carries specific, actionable information or a justified reason, and
`reaction` when it doesn't.

What it actually learned was much shallower. It effectively learned a near-majority
strategy: predict `hot_take` most of the time (21/32), sometimes `analysis`, and
**never `reaction`**. The minority class collapsed entirely (recall 0.00), and all
error confidences sit at ~0.35 — essentially chance for three classes — so the model
did not internalize a real boundary; it underfit. The errors that *do* occur land
right where planning predicted (the `analysis`↔`reaction` line, e.g. tech-flavored
deflections read as substantive), which validates the taxonomy's hard case but also
shows 145 training rows + 3 epochs were not enough to learn it.

This **misses my pre-registered success bar** (planning §6: macro-F1 ≥ 0.70 minimum;
actual 0.38). Concrete next steps implied by the evidence: address class imbalance
(class weights or oversampling `reaction`), train longer / try a larger model, and
collect more data — the 32-post test set (only 8 `reaction`) is also too small for
stable per-class estimates.

---

## Spec reflection

- **One way the spec helped:** requiring edge-case handling rules *before* annotating
  (Milestone 2) kept the 208 labels consistent — when I hit ambiguous posts like id 5
  and id 24, I already had a rule, so I didn't drift mid-dataset. It also made the
  failure analysis interpretable: the model's errors mapped onto the exact boundary the
  spec had me document in advance.
- **One way implementation diverged, and why:** the notebook shipped a `LABEL_MAP`
  template using `analysis/hot_take/reaction`. Rather than rename the map to my forum
  labels (`helpful_answer/help_request/noise`), I renamed the *labels* to match the
  template so the notebook ran without cell edits. The trade-off: the label names no
  longer describe their meaning, which I mitigate with the naming note at the top of
  this README and the definitions throughout. A second, unintended divergence: the
  baseline cell's placeholder prompt overwrote my real prompt, so the baseline run was
  invalid — caught during this write-up, fix documented above.

---

## AI usage

This project used AI tooling at three points; specific directed instances:

1. **Data gathering + pre-labeling (annotation assistance — disclosed).** I directed
   Claude to fetch posts thread-by-thread from the forum and pre-label each against my
   definitions. **What I revised/overrode:** I reviewed every row and overrode boundary
   cases — e.g. I kept asker self-resolutions (id 108) as `hot_take` rather than
   `analysis`, and demoted bare product name-drops (id 3) to `reaction`. The dataset is
   therefore **AI-pre-labeled, human-reviewed**.
2. **Notebook diagnosis + results write-up.** I directed Claude to read the committed
   `.ipynb`, extract the real metrics, and reconstruct the confusion matrix from the
   per-class precision/recall (the raw matrix array wasn't printed). **What I
   revised/overrode:** I had it *verify* the reconstructed matrix numerically against
   every reported P/R/F1 before trusting it, and I rejected reporting any baseline
   number because the run was invalid (`nan`) — no fabricated comparison.
3. **Repo + docs scaffolding.** I directed Claude to set up the GitHub repo, planning.md
   structure, and this README. **What I revised/overrode:** I chose a public repo and
   chose to rename labels to match the notebook template.

_(Failure-analysis assistance, per [planning.md §7.3](planning.md): the wrong-prediction
list was clustered into the patterns above and each pattern was verified against the
confusion matrix before being written up.)_

---

## Repository contents

| Path | Description |
|---|---|
| [planning.md](planning.md) | Plan: community, labels, edge cases, data plan, metrics, success criteria, AI tool plan. |
| [data/labeled_dataset.csv](data/labeled_dataset.csv) | 208 hand-reviewed labeled posts (`id`, `text`, `label`). |
| [Another_copy_of_ai201_project3_takemeter_starter_clean.ipynb](Another_copy_of_ai201_project3_takemeter_starter_clean.ipynb) | Fine-tuning + baseline notebook (committed copy). |
| [results/evaluation_results.json](results/evaluation_results.json) | Metrics exported from Colab _(pending baseline re-run)_. |
| [results/confusion_matrix.png](results/confusion_matrix.png) | Confusion matrix image from Colab _(pending download)_. |
