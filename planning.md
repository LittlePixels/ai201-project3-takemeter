# Project 3 — TakeMeter — Planning

## 1. Problem statement

<!-- TODO: What is TakeMeter classifying? One or two sentences. -->
TakeMeter is an LLM-based classifier that labels a piece of text by ...

## 2. Labels / classes

<!-- TODO: list the categories the model predicts -->
| Label | Meaning |
| --- | --- |
| `label_a` | ... |
| `label_b` | ... |

## 3. Dataset

- **Source:** <!-- where the raw text came from -->
- **Size:** <!-- number of labeled examples -->
- **Labeling process:** <!-- how labels were assigned, by whom, any guidelines -->
- **File:** [data/labeled_dataset.csv](data/labeled_dataset.csv)
- **Columns:** `id`, `text`, `label`

## 4. Approach

<!-- TODO: which model / prompt strategy. e.g. few-shot prompting an LLM, fine-tuning, etc. -->
- Model:
- Prompting / method:
- Notebook:

## 5. Evaluation

- **Metrics:** accuracy, precision, recall, F1 (per class + macro)
- **Held-out / test split:** <!-- how the data is split -->
- **Outputs:**
  - [results/evaluation_results.json](results/evaluation_results.json) — numeric metrics
  - [results/confusion_matrix.png](results/confusion_matrix.png) — confusion matrix plot

## 6. Risks & limitations

<!-- TODO: known biases, small dataset, ambiguous labels, etc. -->

## 7. Timeline / milestones

- [ ] Define labels and labeling guidelines
- [ ] Collect and label dataset
- [ ] Build notebook + run model
- [ ] Evaluate and export results
- [ ] Write up findings in README
