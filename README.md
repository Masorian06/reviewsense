# ReviewSense: version control for data and models

CMPT-2500 Machine Learning Deployment and Software Development, Unit 5.

ReviewSense classifies Amazon product reviews as positive or negative so a support
team can triage unhappy customers first. This repository is the starting point for
the Unit 5 lab: a small, real ML project whose code lives in Git and whose data,
models and experiments you will put under version control with DVC, then track
with MLflow. Nothing here is versioned yet. That is the lab: open `LAB.md`.

## What is in the box

```
reviewsense/
├── params.yaml            every tunable setting in one file that Git can diff
├── requirements.txt       dvc[s3], scikit-learn, pandas, mlflow, huggingface_hub
├── src/
│   ├── get_data.py        downloads Amazon Reviews 2023 and builds the datasets
│   ├── merge_sources.py   month 2: appends the music reviews to data/reviews.csv
│   ├── prepare.py         stage 1: label (4-5 stars positive, 1-2 negative), clean, split
│   ├── featurize.py       stage 2: TF-IDF features
│   ├── train.py           stage 3: logistic regression, with optional MLflow tracking
│   ├── evaluate.py        stage 4: metrics -> eval/metrics.json
│   └── register_model.py  block 03: MLflow Model Registry and aliases
└── LAB.md                 the lab, step by step
```

## Quick start (WSL 2 Ubuntu 24.04, Linux or macOS)

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python src/get_data.py                 # downloads All_Beauty once (311 MB), writes data/reviews.csv
python src/prepare.py && python src/featurize.py && python src/train.py && python src/evaluate.py
cat eval/metrics.json
```

`get_data.py` pulls the raw file from the Hugging Face Hub (dataset
`McAuley-Lab/Amazon-Reviews-2023`, no account needed; a token in `HF_TOKEN` is used
if present) and falls back to the UCSD mirror (`--source ucsd`, 90 MB compressed).
The download is cached in `data/downloads/`, which Git ignores.

## The data

| Dataset version | Rows | Source category | Built by |
| --- | --- | --- | --- |
| Month 1, `data/reviews.csv` | 120,000 | All_Beauty (701,528 reviews in the full file) | `python src/get_data.py` |
| Month 2, same file after the merge | 160,000 | + 40,000 Digital_Music (130,434 in the full file) | `python src/get_data.py --month 2` then `python src/merge_sources.py` |

The sample is a seeded reservoir sample over the file's line order, so everyone who
runs the script gets the same rows, the same bytes and the same hash. Columns:
`review_id, category, rating, title, text, verified_purchase, helpful_vote, date,
parent_asin, user_id`.

Source: Hou, Y., Li, J., He, Z., Yan, A., Chen, X. and McAuley, J. (2024).
*Bridging Language and Items for Retrieval and Recommendation*, arXiv:2403.03952.
Dataset page: https://amazon-reviews-2023.github.io/ (research and teaching use; cite the paper).

## The pipeline

```
data/reviews.csv ──▶ prepare ──▶ data/prepared/ ──▶ featurize ──▶ data/features/ ──▶ train ──▶ models/model.pkl ──▶ evaluate ──▶ eval/metrics.json
                     split, seed                    max_features, ngrams              C, seed
                     └──────────────────────── params.yaml ────────────────────────┘
```

`train.py` logs each run to MLflow when `MLFLOW_TRACKING_URI` is set, tagging it
with the Git commit and the hash of `data/reviews.csv` so a number in the run table
can always be traced back to exact code and exact bytes.
