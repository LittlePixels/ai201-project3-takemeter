# Project 3 — TakeMeter — Planning

TakeMeter is an LLM-based classifier that scores the **substance of a single forum
post** in a tech-help community. Given the text of one post, it predicts whether
the post is a request for help, a substantive answer, or low-value noise. The goal
is a classifier that could power a real community tool — e.g. surfacing the genuinely
helpful replies in a long thread and filtering out the noise.

---

## 1. Community

**BleepingComputer — [Network Streaming Devices forum](https://www.bleepingcomputer.com/forums/f/230/network-streaming-devices/)**

I chose this forum because it is a long-running, volunteer-run tech-help community
where the *quality of a contribution* is the thing members actually care about, and
because it is text-heavy and public (no login needed to read), so I can collect
hundreds of real posts.

**Why it's a good fit for classification — the discourse is genuinely varied:**

- **Different post intents coexist in the same threads.** A single thread mixes the
  original problem description, deep troubleshooting replies, bare one-line
  suggestions, off-topic banter, moderator housekeeping, and product promotion. That
  variety is what makes the labels non-trivial to assign.
- **Quality spans the full range.** Some replies are excellent (exact device/firmware
  identification with a manual link; step-by-step instructions with screenshots),
  while others are near-worthless ("Fire Stick or NVIDIA Shield." / "Wow." / "very
  helpful comment, thanks!"). A classifier has to separate these even though both are
  "replies."
- **The community itself polices substance.** Moderators openly call out
  *"promotion of a product without needed explanation,"* which confirms that the
  good-vs-noise distinction is real and recognized by regulars, not something I
  invented.

This is a **substance-based** distinction, not "hot take vs. analysis" — the quality
axis here is whether a post actually moves a problem toward resolution.

---

## 2. Labels

Three labels, assigned along **intent + substance**.

### `help_request`
**Definition:** A post is `help_request` if its purpose is to obtain assistance —
describing a technical problem, asking how to do something, or asking for a product
recommendation — and this includes the original asker's follow-up posts that add
diagnostic detail or clarify while they are still seeking a resolution.
- *Example (id 1):* "WHICH streaming device to replace Roku? ... I need a remote with power/volume/mute and 30-second skip ... Would Chromecast or Fire Stick be better, and do they support commercial skipping on recorded shows?"
- *Example (id 18):* "I'm getting intermittent UPnP streaming problems on Windows 11. I use Foobar/MediaMonkey ... streaming FLAC to a Mano streamer ... One time it works fine for hours, but after the next boot it does not work at all."

### `helpful_answer`
**Definition:** A post is `helpful_answer` if it is a reply that materially helps the
asker by providing specific troubleshooting steps, a technical explanation, a
*justified* recommendation, or relevant firsthand experience with reasoning.
- *Example (id 22):* "That's a MAGNA HIFI MANO ULTRA MKII running piCorePlayer v8.0.0 with Squeezelite v1.9.9-1430-pCP. Here's the manual link for the advanced configuration so you can check the service startup order."
- *Example (id 155):* "I had the exact same problem. I bought a 10/100/1000 switch, hooked both the TV and the PC to it, and the problem went away."

### `noise`
**Definition:** A post is `noise` if it is a reply (or non-help post) that adds little
actionable value — a vague one-liner, a "look it up yourself" deflection, a bare
thanks or agreement, off-topic venting, moderator/thread housekeeping, or unexplained
self-promotion or spam.
- *Example (id 39):* "very helpful comment, i read it, thanks for sharing!"
- *Example (id 37):* "Using express vpn, fast and reliable and also safe, dont go for free vpn." (an unexplained product push; a moderator flagged it as promotion)

**Tiebreaker (the single rule that keeps the labels precise):** for a short reply that
*does* respond to the asker, ask "does it convey specific, actionable information or a
reason?" — if yes it is `helpful_answer`, if no it is `noise`, *regardless of length*.

---

## 3. Hard edge cases

The genuinely ambiguous posts cluster on two boundaries. For each I commit to a rule
now, before annotating, so the calls stay consistent.

**(a) `helpful_answer` ↔ `noise` — the bare recommendation.** A reply that names a
product or action but gives no reasoning (e.g. id 3 "Fire Stick or NVIDIA Shield."
vs. id 5 "I'd say NVIDIA Shield is the best ... bear in mind some of these service
issues follow you regardless of hardware"). This is the hardest and most common edge
case.
**Handling rule:** a name-drop with *no* justification, comparison, or caveat →
`noise`; if it carries even one concrete reason, condition, or specific detail →
`helpful_answer`.

**(b) `help_request` ↔ `noise` — the discussion/venting opener.** A thread-opening
post that airs a grievance or opinion rather than clearly asking for a fix (e.g. id
24 "This is just scummy what Roku did ...").
**Handling rule:** if the post invites a substantive response or contains an implied
question, label `help_request`; if it is a pure rant seeking nothing, label `noise`.

**(c) `help_request` ↔ `helpful_answer` — the asker's own follow-up.** When the
original asker posts new diagnostic info or even their own fix (e.g. id 108, id 204),
it can read like an "answer."
**Handling rule:** classify by author role — posts from the person seeking resolution
stay `help_request` even when informative; only replies *from others that help the
asker* are `helpful_answer`.

A pointer to documentation is resolved by the same substance test as (a): a *specific*
relevant resource that answers the question (id 84, a named article comparing free-VPN
speeds) → `helpful_answer`; a generic "go research it / check the manual" brush-off
(id 2, id 132) → `noise`.

---

## 4. Data collection plan

- **Source:** Public posts and replies from the BleepingComputer Network Streaming
  Devices forum, collected thread-by-thread across the forum index pages.
- **Collected so far:** **208 labeled posts across 31 threads**, stored in
  [data/labeled_dataset.csv](data/labeled_dataset.csv) with columns `id`, `text`,
  `label`.
- **Current distribution:** `helpful_answer` 86 · `help_request` 70 · `noise` 52.
- **Target:** at least 200 labeled posts (met), with **no class below ~40 examples**
  so every class is learnable and the confusion matrix is meaningful. My rough aim is
  roughly 60–90 per class rather than a forced even split, since the natural class
  mix is itself informative.
- **If a label is underrepresented after 200 examples:** `noise` is the rarest class
  and the one at risk. My mitigation, in order: (1) **targeted collection** — revisit
  long threads and very old/locked threads, which concentrate bare one-liners, thanks,
  venting, and moderator housekeeping; (2) sample threads with high reply counts where
  low-value chatter accumulates; (3) only if real examples remain too scarce, **report
  the imbalance explicitly** and rely on macro-averaged metrics (below) rather than
  fabricating synthetic posts, so the evaluation stays honest.

---

## 5. Evaluation metrics

The classes are imbalanced (noise ≈ 25%), so **accuracy alone is misleading** — a model
could score ~75% by being good at the two larger classes while failing on `noise`,
which is the class a real moderation/curation tool most needs to catch. I will report:

- **Per-class precision, recall, and F1** — the primary metrics. They tell me, for
  each label, both how often the model is right when it predicts that label
  (precision) and how many true cases it catches (recall). I especially care about
  **`helpful_answer` recall** (a curation tool must not hide good answers) and
  **`noise` precision** (a filter must not wrongly suppress real help).
- **Macro-averaged F1** — the single headline number, because it weights all three
  classes equally and refuses to let strong performance on the big classes paper over
  weak performance on `noise`.
- **Confusion matrix** ([results/confusion_matrix.png](results/confusion_matrix.png))
  — to see *where* errors go. I expect the `helpful_answer`↔`noise` boundary (edge
  case a) to be the dominant error cell, and the matrix lets me confirm or refute that.
- **Overall accuracy** — reported for context only, not as the success criterion.

Metrics will be exported to
[results/evaluation_results.json](results/evaluation_results.json).

---

## 6. Definition of success

These thresholds are set in advance so I can objectively judge the outcome:

- **Minimum bar (project is working):** macro-F1 **≥ 0.70**, AND every per-class
  F1 **≥ 0.60**, AND overall accuracy **≥ 0.75**.
- **"Good enough to deploy" in a real community curation tool:** macro-F1 **≥ 0.80**,
  with two task-specific safety constraints:
  - **`helpful_answer` recall ≥ 0.85** — the tool must surface at least 85% of genuinely
    helpful replies; hiding good answers is the worst failure for a curation feature.
  - **`noise` precision ≥ 0.80** — when the tool flags a post as noise, it must be
    right at least 80% of the time, so it rarely suppresses real help.
- **Stretch:** macro-F1 ≥ 0.85 with no per-class F1 below 0.78.

If the model clears the minimum bar but misses a deployment constraint, the honest
conclusion is "promising but not deployable as-is," and the failure analysis (§7.3)
should explain which boundary is responsible.

---

## 7. AI Tool Plan

There is no application code to generate in this project, so AI tooling is used at the
three points where it genuinely helps a labeling/evaluation workflow.

### 7.1 Label stress-testing (do this *before* annotating the full set)
I will give an LLM (Claude) my three label definitions plus the edge-case rules in §3
and ask it to generate **5–10 posts that deliberately sit on the `helpful_answer` ↔
`noise` and `help_request` ↔ `noise` boundaries**. If I cannot assign each generated
post to exactly one label using my current rules, the definitions are too loose and I
will tighten them (most likely by sharpening the §2 tiebreaker) *before* finalizing
annotations. These generated posts are for definition-testing only and will **not** be
added to the dataset.

### 7.2 Annotation assistance
**Decision: yes, with disclosure.** The 208 posts were **pre-labeled by an LLM
(Claude) and then reviewed by me**, applying the §2/§3 rules; I make the final call on
every row. Tracking/disclosure plan: the AI usage section of the README will state
that the dataset was AI-pre-labeled then human-reviewed. To make the human review
auditable I will (optionally) add a `prelabel` column recording the model's original
guess alongside the final `label`, so any row where I overrode the model is visible.
The LLM is **never** used to label the held-out test split that produces the final
metrics — that split is human-labeled independently to avoid measuring the model
against its own guesses.

### 7.3 Failure analysis
After running the notebook I will export the list of misclassified posts (true label,
predicted label, text) and give it to an LLM with the prompt: *"group these errors and
identify the dominant failure patterns."* I will specifically look for: (a) whether
errors concentrate on the bare-recommendation boundary (edge case a); (b) whether
short posts are systematically misread; and (c) whether any single class dominates the
errors. **Verification:** I will not take the AI's patterns at face value — for each
claimed pattern I will pull the actual rows from the confusion matrix and confirm the
pattern holds before writing it into the evaluation, discarding any claim the data
doesn't support.

---

## 8. Approach (to be completed)

<!-- TODO: model + prompt strategy, e.g. few-shot prompting Claude with the §2 definitions -->
- Model:
- Prompting / method:
- Notebook:

## 9. Risks & limitations

- **Class imbalance:** `noise` is the smallest class; macro-averaged metrics guard
  against this hiding poor performance.
- **Boundary ambiguity:** the `helpful_answer` ↔ `noise` line (bare vs. justified
  recommendations) is the hardest call and the likeliest source of model confusion.
- **Single annotator + AI pre-labeling:** labels reflect one reviewer's judgment on top
  of an AI's guesses; the human-labeled test split mitigates circularity but not
  individual bias.
- **Single-forum, paraphrased text:** post text is condensed from one community, so
  patterns may not generalize and exact wording is not guaranteed verbatim.

## 10. Milestones

- [x] Choose community and read 30–40 posts
- [x] Define labels, examples, and edge-case rules
- [x] Collect and label ≥ 200 posts
- [ ] Stress-test labels with AI (§7.1) and tighten if needed
- [ ] Build notebook + run model
- [ ] Evaluate, export results, run failure analysis (§7.3)
- [ ] Write up findings + AI usage in README
