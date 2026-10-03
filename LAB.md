# Lab: ReviewSense under version control

CMPT-2500 · Unit 5 · Version control for data and models · about 3 hours

By the end of this lab your copy of ReviewSense has two dataset versions tracked by
DVC, a four-stage pipeline whose every run is recorded in `dvc.lock`, a handful of
experiments you can compare with one command, and an MLflow run table that links each
score to the exact code and the exact bytes that produced it. Every step below shows
the output you should see, taken from a real run on 2 October 2026 (DVC 3.67, MLflow 3.16).
Your hashes for `data/reviews.csv` must match the ones shown; everything downstream
(feature matrices, model, scores) is also deterministic, so your F1 values should match
to the fourth decimal.

Hand in (see the checklist at the end): the URL of your repository and one screenshot
of the MLflow run table sorted by `f1_macro`.

## 0. Set up (10 min)

You need Git, the GitHub CLI (`gh auth login` once) and Python 3.10 or newer. On
Windows use WSL 2 (Ubuntu 24.04) and keep the repository inside the Linux file system
(`~/`, not `/mnt/c/...`): DVC hashes and links files, and that is painfully slow on `/mnt/c`.

```bash
gh repo create reviewsense --template Masorian06/reviewsense --private --clone
cd reviewsense
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
dvc version          # 3.6x, and the "Supports" list includes s3
python src/get_data.py
```

Expected (the download of `All_Beauty.jsonl`, 311 MB, happens once and is cached in
`data/downloads/`; a warning about an unset `HF_TOKEN` is harmless):

```
Month 1: 120,000 reviews from All_Beauty
  701,528 reviews in the file, 120,000 sampled
  wrote data/reviews.csv: 120,000 rows, 32.9 MB, md5 1c08488b0738e4cc97ce4954abcc805b
```

If the Hugging Face Hub is unreachable, `python src/get_data.py --source ucsd` fetches a
90 MB compressed copy from the authors' mirror and produces the same file, byte for byte.

Check yourself: `wc -l data/reviews.csv` prints `120001` (one header line plus 120,000 reviews).

## Part A · Version the dataset (steps 1 to 6, 45 min)

### Step 1 · DVC lives inside a Git repository

```bash
dvc init
git status
git commit -m "Initialize DVC"
```

`dvc init` prints a box about anonymous usage analytics (opt out with
`dvc config core.analytics false` if you like) and `git status` shows three new files
staged for you: `.dvc/.gitignore`, `.dvc/config`, `.dvcignore`. Try `dvc init` in a
folder that is not a Git repository some day: it refuses. DVC needs Git for history.

### Step 2 · Track the dataset, commit the pointer

```bash
dvc add data/reviews.csv
cat data/reviews.csv.dvc
cat data/.gitignore
git add data/reviews.csv.dvc data/.gitignore
git commit -m "Add month-1 reviews (120k rows)"
git tag data-v1
```

```
outs:
- md5: 1c08488b0738e4cc97ce4954abcc805b
  size: 32864605
  hash: md5
  path: reviews.csv
```

Four lines of text now stand for 33 MB of data. The bytes moved to
`.dvc/cache/files/md5/1c/08488b...` and are linked back into `data/`; DVC also wrote
`/reviews.csv` into `data/.gitignore`, so Git never sees them. The tag names this data
version; you will travel back to it in step 6.

### Step 3 · A remote, and the first push

In the lecture demo the remote is an S3 bucket. For the lab a folder on your disk
behaves exactly the same way (DVC only cares that it is a place to copy objects to):

```bash
mkdir -p ~/dvcstore
dvc remote add -d storage ~/dvcstore
cat .dvc/config
git commit -am "Configure local remote"
dvc push
ls ~/dvcstore/files/md5/1c/
git push --tags origin main
```

```
[core]
    remote = storage
['remote "storage"']
    url = /home/<you>/dvcstore
1 file pushed
08488b0738e4cc97ce4954abcc805b
```

Two pushes, two kinds of things: `git push` sends the pointers (and the tag) to GitHub,
`dvc push` sends the bytes to the remote. If you have an AWS account, the S3 version is
`dvc remote add -d storage s3://<your-bucket>/store` with credentials from `aws configure`;
the commands that follow do not change.

### Step 4 · Clone and pull: a teammate gets the same bytes

Be your own teammate. Clone your repository into a second folder and notice that the
data is missing until you pull it:

```bash
cd ~
gh repo clone <your-github-user>/reviewsense reviewsense-clone
cd reviewsense-clone
ls data/
source ~/reviewsense/.venv/bin/activate      # reuse the environment
dvc pull
md5sum data/reviews.csv
cd ~/reviewsense
```

```
reviews.csv.dvc
A       data/reviews.csv
1 file fetched and 1 file added
1c08488b0738e4cc97ce4954abcc805b  data/reviews.csv
```

The hash matches the pointer file, so the clone holds exactly the same bytes. This is the
sequence a CI job or a training server will run in Unit 6: `git pull`, then `dvc pull`.

### Step 5 · Month 2: a second version of the dataset

```bash
python src/get_data.py --month 2
python src/merge_sources.py
dvc status
dvc add data/reviews.csv
git diff data/reviews.csv.dvc
dvc push
git commit -am "Add 40k music reviews (month 2)"
git tag data-v2
git push --tags origin main
```

```
  wrote data/incoming/music_reviews.csv: 40,000 rows, 17.7 MB, md5 783345e269bfb63dcde66804f0cef833
appended 40,000 rows; data/reviews.csv now has 160,000 rows
data/reviews.csv.dvc:
	changed outs:
		modified:           data/reviews.csv
-- md5: 1c08488b0738e4cc97ce4954abcc805b
-  size: 32864605
+- md5: 6bbc136bf2dc2c692fc27289a2d5864c
+  size: 50562295
1 file pushed
```

Both objects now sit in the cache and in the remote (`ls ~/dvcstore/files/md5/` shows
`1c` and `6b`). Nothing was overwritten: a new version is a new object. The merge
script is committed too, so the change itself is traceable.

### Step 6 · Travel back to month 1

```bash
git checkout data-v1 -- data/reviews.csv.dvc
dvc status
dvc checkout
wc -l data/reviews.csv
git checkout data-v2 -- data/reviews.csv.dvc
dvc checkout
wc -l data/reviews.csv
```

```
M       data/reviews.csv
120001 data/reviews.csv
M       data/reviews.csv
160001 data/reviews.csv
```

The sentence to remember: `git checkout` changes the pointer, `dvc checkout` changes the
bytes. The swap takes well under a second because both objects are already in the cache.

## Part B · The pipeline (steps 7 to 9, 40 min)

### Step 7 · Describe the pipeline as stages

```bash
dvc stage add -n prepare   -p prepare.split,prepare.seed \
    -d src/prepare.py -d data/reviews.csv -o data/prepared  python src/prepare.py
dvc stage add -n featurize -p featurize.max_features,featurize.ngrams \
    -d src/featurize.py -d data/prepared -o data/features    python src/featurize.py
dvc stage add -n train     -p train.C,train.seed \
    -d src/train.py -d data/features -o models/model.pkl     python src/train.py
dvc stage add -n evaluate  -d src/evaluate.py -d models/model.pkl -d data/features \
    -M eval/metrics.json                                     python src/evaluate.py
cat dvc.yaml
git add dvc.yaml data/.gitignore
git commit -m "Define the pipeline"
```

`-d` dependency, `-p` parameter key in `params.yaml`, `-o` cached output, `-M` a small
metrics file kept in Git instead of the cache. `dvc.yaml` is readable YAML you could have
typed by hand. The warning "Could not create .gitignore entry in models/.gitignore" is
expected: the folder does not exist yet; DVC creates the entry on the first run.

### Step 8 · Run it; `dvc.lock` records the lineage

```bash
dvc repro
cat eval/metrics.json
git add dvc.lock models/.gitignore eval/metrics.json
git commit -m "First full run"
dvc push
dvc repro
dvc dag
```

```
Running stage 'prepare':
> python src/prepare.py
prepare: 148,493 labelled reviews -> train 118,794 / test 29,699, positive share 81.3%
Running stage 'featurize':
featurize: 118,794 x 5,000 train matrix, ngrams 1-1
Running stage 'train':
train: logistic regression, C=1.0, 118,794 documents
Running stage 'evaluate':
evaluate: {"f1_macro": 0.9183, "accuracy": 0.9521, "precision_negative": 0.9074, "recall_negative": 0.8279, "n_test": 29699}
```

The whole run takes about 20 seconds. Open `dvc.lock`: for every stage, the hash and size
of each dependency and output and the value of each parameter. That is the lineage chain
from the lecture, written for you. The second `dvc repro` runs nothing ("Data and
pipelines are up to date"), and `dvc push` reports `8 files pushed`: the prepared splits,
the feature matrices and the model are now in the remote too.

### Step 9 · Change a parameter; only what depends on it reruns

Edit `params.yaml`: `max_features: 10000` and `ngrams: 2`. Then:

```bash
dvc repro
dvc params diff
dvc metrics diff
git commit -am "Bigrams, 10k features: F1 0.9251"
dvc push
```

```
Stage 'prepare' didn't change, skipping
Running stage 'featurize': ...
Path         Param                   HEAD    workspace
params.yaml  featurize.max_features  5000    10000
params.yaml  featurize.ngrams        1       2
Path               Metric              HEAD    workspace    Change
eval/metrics.json  accuracy            0.9521  0.9563       0.0042
eval/metrics.json  f1_macro            0.9183  0.9251       0.0068
eval/metrics.json  precision_negative  0.9074  0.9247       0.0173
eval/metrics.json  recall_negative     0.8279  0.8336       0.0057
```

Now put the two parameters back to 5000 and 1 and run `dvc repro` again: every stage says
"is cached - skipping run, checking out outputs". DVC's run cache remembers that these
inputs already produced these outputs. Restore the bigram values with
`git checkout params.yaml` and run `dvc repro` once more (cached again).

## Part C · Experiments (step 10, 25 min)

```bash
dvc exp run -S train.C=0.5
dvc exp run --queue -S train.C=0.1
dvc exp run --queue -S train.C=2 -S featurize.ngrams=3
dvc exp run --run-all
dvc exp show --only-changed
```

```
  Experiment                 f1_macro   accuracy   precision_negative   recall_negative   featurize.ngrams   train.C
  workspace                    0.9186     0.9529                0.926            0.8127   2                  0.5
  main                         0.9251     0.9563               0.9247            0.8336   2                  1
  ├── e67ad56 [areal-mome]     0.9298     0.9586               0.9201             0.852   3                  2
  ├── 2881c82 [vocal-caps]     0.8812     0.9354                0.943             0.696   2                  0.1
  └── 1501ed3 [burnt-acne]     0.9186     0.9529                0.926            0.8127   2                  0.5
```

Your experiment names will differ (DVC invents them), the numbers will not. Experiments
are Git references, not commits: `git log` stays clean. Bring the winner into the
workspace and commit it as a normal change:

```bash
dvc exp apply <name-of-the-best-experiment>
cat params.yaml
git commit -am "C=2, trigrams: F1 0.9298"
dvc push
git push origin main
```

Run at least two experiments of your own (`-S` any key of `params.yaml`) before moving on,
and keep the one you would ship.

## Part D · MLflow: the run table and the registry (40 min)

Open a second terminal in the repository, activate the environment and start a tracking
server backed by SQLite (the Model Registry needs a database):

```bash
mlflow server --backend-store-uri sqlite:///mlflow.db --host 127.0.0.1 --port 5000
```

Back in the first terminal, point the pipeline at it. `train.py` logs a run whenever this
variable is set, tagging it with the Git commit, the data hash from `data/reviews.csv.dvc`
and the DVC experiment name:

```bash
export MLFLOW_TRACKING_URI=http://127.0.0.1:5000
git checkout data-v1 -- data/reviews.csv.dvc && dvc checkout
dvc exp run -n m1-base    -S featurize.max_features=5000  -S featurize.ngrams=1 -S train.C=1.0
git checkout data-v2 -- data/reviews.csv.dvc && dvc checkout
dvc exp run -n m2-base    -S featurize.max_features=5000  -S featurize.ngrams=1 -S train.C=1.0
dvc exp run -n m2-c01     -S featurize.max_features=5000  -S featurize.ngrams=1 -S train.C=0.1
dvc exp run -n m2-bigrams -S featurize.max_features=10000 -S featurize.ngrams=2 -S train.C=1.0
```

Open http://127.0.0.1:5000, experiment `reviewsense`, and add the columns `git_commit`,
`data_md5` and `dvc_exp` (Columns menu, Tags). Sort by `f1_macro`. You should see:

| run | data_md5 | ngrams | max_features | C | f1_macro | reading |
| --- | --- | --- | --- | --- | --- | --- |
| m1-base | 1c0848 (v1) | 1 | 5000 | 1.0 | 0.9222 | month 1 |
| m2-base | 6bbc13 (v2) | 1 | 5000 | 1.0 | 0.9183 | new data alone: −0.004 |
| m2-c01 | 6bbc13 (v2) | 1 | 5000 | 0.1 | 0.8814 | C = 0.1: −0.037 |
| m2-bigrams | 6bbc13 (v2) | 2 | 10000 | 1.0 | 0.9251 | candidate |

Take the screenshot now. Then register models and move aliases:

```bash
python src/register_model.py --run-id <run id of m1-base>                      # version 1
python src/register_model.py --alias champion   --run-id <run id of m2-c01>    # version 2
python src/register_model.py --alias challenger --run-id <run id of m2-bigrams> # version 3
python src/list_models.py
python src/register_model.py --alias champion --version 3                      # promotion
python src/list_models.py
```

Run ids are the first column of the UI, or `mlflow.search_runs(...)` in Python.
Promotion is moving an alias; nothing is copied and the code that loads
`models:/reviewsense@champion` never changes. Finally, prove the registry works:

```bash
python - <<'PY'
import mlflow, pickle
from mlflow import MlflowClient
model = mlflow.pyfunc.load_model("models:/reviewsense@champion")
v = MlflowClient().get_model_version_by_alias("reviewsense", "champion")
vec = pickle.load(open(mlflow.artifacts.download_artifacts(run_id=v.run_id, artifact_path="vectorizer.pkl"), "rb"))
texts = ["Great product, exactly as described", "Broke after two days, waste of money"]
print(model.predict(vec.transform(texts)))
PY
```

Leave the workspace clean before you finish: `git checkout -- params.yaml dvc.lock eval/metrics.json && dvc checkout`.

## Hand in

Add a section `## Lab results` to your `README.md` with the two data hashes, the
parameters and F1 of the model you would ship, and the path of your screenshot (commit it
under `docs/`). Then push and submit the repository URL. The instructor checks:

- [ ] tags `data-v1` and `data-v2` exist and point at different `data/reviews.csv.dvc` contents
- [ ] `dvc.yaml` has the four stages; `dvc.lock` and `eval/metrics.json` are committed
- [ ] `dvc exp show` lists at least five experiments, two of them yours
- [ ] the MLflow screenshot shows `git_commit` and `data_md5` on every run
- [ ] `python src/list_models.py` shows three versions and the `champion` alias
- [ ] the `## Lab results` section, and a clean `git status`

## Troubleshooting

- `ERROR: failed to initiate DVC - /... is not tracked by any supported SCM tool`: you ran `dvc init` outside a Git repository.
- `dvc push` says "Everything is up to date" when you expected a push: the object is already in the remote (same bytes, same hash). That is the point.
- `dvc exp run --run-all` seems stuck: open another terminal and run `dvc queue status`, then `dvc queue logs <name>`.
- Changed nothing but want to log a run to MLflow: the run cache skips the stage. Use `dvc exp run -f`, or change a parameter.
- `Address already in use` from `mlflow server`: another server is on port 5000; pick `--port 5001` and update `MLFLOW_TRACKING_URI`.
- "Failed to resolve installed pip version" while logging a model: harmless, it happens in environments created by `uv`.
- Slow `dvc add` / `dvc checkout` on WSL: the repository is under `/mnt/c`. Move it to `~/`.
- `zsh: no matches found: dvc[s3]`: quote it, `pip install "dvc[s3]"`, or just use `requirements.txt`.
