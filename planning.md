# Project 3 — TakeMeter — Planning

## 1. Problem statement

TakeMeter is an LLM-based classifier that scores the **substance/helpfulness of a
post** in a tech-help community. Given the text of a single forum post, it predicts
whether the post is a request for help, a substantive answer, or low-value noise.

## 2. Community

**BleepingComputer — [Network Streaming Devices forum](https://www.bleepingcomputer.com/forums/f/230/network-streaming-devices/)**

> BleepingComputer's *Network Streaming Devices* forum is a volunteer tech-help
> community where users troubleshoot streaming hardware (Roku, Fire Stick,
> Chromecast, Android TV) and ask for device/VPN recommendations. Posts fall into
> three recognizable types — requests for help, substantive answers grounded in
> technical knowledge or firsthand experience, and low-value noise (one-line
> deflections, bare thanks, and unexplained product promotion). The distinction
> matters because the forum runs on unpaid expertise: regulars and moderators
> actively reward specific, actionable help and call out the noise that wastes
> their time.

This distinction is **substance-based**, not "hot take vs. analysis" — the quality
axis regulars care about is whether a post actually helps. The community itself
polices it: moderators openly call out *"promotion of a product without needed
explanation."*

## 3. Labels / classes

Three labels, assigned along **intent + substance**. Tiebreaker for a short reply
that does answer: *specific actionable info → `helpful_answer`; otherwise →
`noise`*, regardless of length.

| Label | Definition |
| --- | --- |
| `help_request` | A post that seeks assistance — describing a tech problem, asking how to do something, or requesting a product recommendation (includes the original asker's follow-ups that add diagnostic detail while still seeking resolution). |
| `helpful_answer` | A reply that materially helps — specific troubleshooting steps, a technical explanation, a *justified* recommendation, or relevant firsthand experience with reasoning. |
| `noise` | A reply that adds little actionable value — vague one-liners, "look it up yourself" deflections, bare thanks/agreement, off-topic venting, or unexplained self-promotion/spam. |

### Label examples & boundary cases

**`help_request`**
- *Clear (GracieAllen):* detailed Roku-replacement request listing exact requirements (YouTube TV compatibility, 30-sec skip, remote with volume/mute).
- *Clear (dileo):* "one time it works fine for hours but after next boot... it does not work at all" — describes the full FLAC/UPnP setup and the failure pattern.
- *Uncertain (SuperSapien64, "Roku hacked"):* "This is just scummy what Roku did." Airs a grievance and invites discussion rather than asking for a fix — `help_request` ↔ `noise` boundary. **Rule:** thread-openers that invite substantive response are `help_request`; a pure rant seeking no input is `noise`.

**`helpful_answer`**
- *Clear (cryptodan):* identifies the exact hardware/firmware ("MAGNA HIFI MANO ULTRA MKII running piCorePlayer v8.0.0") and links the configuration manual.
- *Clear (Paperclip):* step-by-step HDMI / Windows "Connect" mirroring instructions with attached screenshots.
- *Uncertain (greg18):* "NVIDIA Shield best, Fire series next, though service issues may persist regardless of hardware." A bare rec with a sliver of reasoning — `helpful_answer` ↔ `noise` boundary. **Rule:** conveys specific actionable info or a reason → `helpful_answer`; a name-drop with no justification → `noise`.

**`noise`**
- *Clear (barkerxavierr):* "very helpful comment, i read it, thanks for sharing!"
- *Clear (Bensasema):* "Using express vpn fast and reliable... dont go for free vpn" — unexplained promo, mod-flagged.
- *Uncertain (Pkshadow):* points the asker to "CNET and PCMag" to determine their own specs — resource pointer or deflection? `noise` ↔ `helpful_answer` boundary. **Rule:** naming a relevant, specific resource that answers the question → `helpful_answer`; a generic "go research it" brush-off → `noise`.

### Mutual-exclusivity check

The three split cleanly: asking (`help_request`) vs. answering-with-substance
(`helpful_answer`) vs. answering-without-substance (`noise`). The one genuine
overlap — a short reply that *does* answer — is resolved by the substance
tiebreaker above. Asker follow-ups stay `help_request` even when they add info,
since intent is still resolution-seeking. On ~60 posts read across 7 threads,
every post assigned to exactly one label without forcing it.

## 4. Dataset

- **Source:** Public posts/replies from the BleepingComputer Network Streaming Devices forum.
- **Size:** target ≥ 200 labeled posts.
- **Labeling process:** Hand-labeled by the project author using the definitions and boundary rules above.
- **File:** [data/labeled_dataset.csv](data/labeled_dataset.csv)
- **Columns:** `id`, `text`, `label` (one of `help_request`, `helpful_answer`, `noise`)

## 5. Approach

<!-- TODO: which model / prompt strategy. e.g. few-shot prompting an LLM, fine-tuning, etc. -->
- Model:
- Prompting / method:
- Notebook:

## 6. Evaluation

- **Metrics:** accuracy, precision, recall, F1 (per class + macro)
- **Held-out / test split:** <!-- how the data is split -->
- **Outputs:**
  - [results/evaluation_results.json](results/evaluation_results.json) — numeric metrics
  - [results/confusion_matrix.png](results/confusion_matrix.png) — confusion matrix plot

## 7. Risks & limitations

- **Class imbalance:** `noise` and especially spam are rarer than help/answers; may need targeted sampling.
- **Boundary ambiguity:** the `helpful_answer` ↔ `noise` line (bare vs. justified recommendations) is the hardest call and the likeliest source of model confusion.
- **Small, single-forum dataset:** patterns may not generalize beyond this community.

## 8. Timeline / milestones

- [x] Define labels and labeling guidelines
- [ ] Collect and label dataset (≥ 200 posts)
- [ ] Build notebook + run model
- [ ] Evaluate and export results
- [ ] Write up findings in README
