# ai201-project3-takemeter

AI201 — Project 3. An LLM-based text classifier ("TakeMeter") with a labeled
dataset, a Colab notebook for training/evaluation, and the resulting metrics.

> The model code lives in a Google Colab notebook. This repository holds
> everything *outside* the notebook: planning, the labeled dataset, and the
> output artifacts downloaded from Colab.

## Repository contents

| File | Description |
| --- | --- |
| [planning.md](planning.md) | Project plan: problem statement, labels, approach, and evaluation strategy. |
| [data/labeled_dataset.csv](data/labeled_dataset.csv) | The hand-labeled dataset used for evaluation. |
| [results/evaluation_results.json](results/evaluation_results.json) | Metrics exported from the Colab notebook. |
| [results/confusion_matrix.png](results/confusion_matrix.png) | Confusion matrix exported from the Colab notebook. |

## Workflow

1. Build and label the dataset → `data/labeled_dataset.csv`.
2. Run the Colab notebook against the dataset.
3. Download `evaluation_results.json` and `confusion_matrix.png` from Colab into `results/`.
4. Commit and push.

## Results

<!-- TODO: paste headline metrics here once the notebook has run, e.g. accuracy / F1 -->
_Pending — fill in after running the notebook._

## Colab notebook

<!-- TODO: add the shareable Colab link -->
_Link: TODO_
