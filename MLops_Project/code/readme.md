# Hospital readmission MLOps project

Predict which diabetic hospital stays will end in a readmission within 30 days, and manage the
models behind that prediction in a disciplined, repeatable way.

This README explains the whole project from start to end for someone who has not seen it before.
It covers what each script does, which files it reads and creates, and how a change in one place
affects the rest.

Last updated: 2026-10-22.

---

## Contents

1. [What this project is](#1-what-this-project-is)
2. [The big picture](#2-the-big-picture)
3. [Quick start](#3-quick-start)
4. [Folder map](#4-folder-map)
5. [Words you will meet](#5-words-you-will-meet)
6. [Setup (once)](#6-setup-once)
7. [The pipeline, step by step](#7-the-pipeline-step-by-step)
8. [How everything affects everything else](#8-how-everything-affects-everything-else)
9. [What happens in each case](#9-what-happens-in-each-case)
10. [Recipes](#10-recipes)
11. [Current results](#11-current-results)
12. [Limits and rules to respect](#12-limits-and-rules-to-respect)
13. [What is not built yet](#13-what-is-not-built-yet)
14. [Troubleshooting](#14-troubleshooting)
15. [Reference: every file, who writes it, who reads it](#15-reference-every-file-who-writes-it-who-reads-it)

---

## 1. What this project is

**The prediction task.** Each row of data is one hospital stay of a diabetic patient. The label is
whether the patient came back within 30 days (about 11.5% did). A model gives every stay a risk
score so that the highest-risk stays can be flagged.

**The real point: MLOps.** MLOps is the machinery around a model. Training a model once is easy.
The hard part is being able to answer, months later:

- Which model is in use, and why was it chosen?
- Was it tested on data it had never seen, using rules written down beforehand?
- Can a newer model replace it safely, and can we undo that?
- Can someone else rebuild everything and get the same numbers?

This project answers those questions with five scripts, an experiment tracker (MLflow) and a set of
reports and logs.

**An honest note on model quality.** Readmission is hard to predict from hospital records. The
current best model finds readmissions about 1.9 times better than random flagging, with a ROC-AUC
of about 0.67. That is typical for this kind of data, and it is not a sign of a bug. This is not a
clinical tool and has not been validated for use with real patients.

---

## 2. The big picture

```
 raw data
    |   earlier steps: clean, then split BY PATIENT
    v
 data/splits/   train.csv        test.csv         holdout
                   |                |                |
         train.py  |                |                | (not used yet:
                   v                |                |  reserved for the replay)
  models/*.joblib + MLflow runs     |
                   |                |
                   +----> test.py <-+      "the gate": every model is scored
                             |             on data it has never seen
                             v
                    evaluation.json + test reports
                             |
                             v
         register_champion.py --champion X --reason "..."     (YOU decide)
                             |
                             v
        MLflow registry "readmission"  ---->  champion.json
        @champion   @challenger   @baseline
                             ^
                             |   promote.py compares challenger vs champion and
                             |   swaps the aliases ONLY if the challenger clearly wins
                             |   (and can roll back)
                             v
                  serving, monitoring (not built yet)
```

Three ideas hold everything together:

1. **Separate data for separate jobs.** Train data is for learning. Test data is for the final
   exam. The holdout is kept sealed for later. No script is allowed to use the wrong one.
2. **Gate, then register, then promote.** A model must pass a pre-written test before it can be
   registered, and must clearly beat the current champion before it replaces it.
3. **The registry is the source of truth.** The question "which model is champion?" is answered by
   the MLflow registry, not by a file name.

---

## 3. Quick start

If the environment is already set up (see [Setup](#6-setup-once)), the whole flow is:

```powershell
.\.venv\Scripts\Activate.ps1

python train.py                 # 1. train all three models
python test.py                  # 2. score them on the test data (the gate)

# 3. YOU choose the champion and say why (first time only)
python register_champion.py --champion lightgbm --reason "Passed the gate; chosen for credible feature reasoning"

python promote.py --dry-run     # 4. later: would a challenger replace the champion?
```

To look at results in a browser:

```powershell
mlflow ui --backend-store-uri sqlite:///mlflow.db
```

Then open http://127.0.0.1:5000. Use the **Experiments** tab for runs and the **Models** tab for the
registry.

---

## 4. Folder map

```
MLops_Project/
|-- code/                     earlier-step scripts (data cleaning and splitting)
|-- data/
|   |-- splits/               train.csv, test.csv, holdout, split_report.txt
|   |-- cleaning_config.json  record of how the data was cleaned
|   |-- training_report.txt   written by train.py (technical summary)
|   `-- test_report.txt       written by test.py (technical summary)
|-- models/
|   |-- 01_<model>_prauc0.xxx.joblib ...   saved models, ranked by CV score (train.py)
|   |-- best_model.json       which model ranked first by CV only (train.py)
|   |-- model_report.md       plain-language training report (train.py)
|   |-- evaluation.json       gate results, machine-readable (test.py)
|   |-- test_report.md        plain-language gate report (test.py)
|   |-- champion.json         the current champion, machine-readable
|   |-- champion_report.md    the current champion, plain language
|   |-- promotion_log.jsonl   audit trail, one line per decision
|   |-- data_use_log.jsonl    every time any dataset was scored (test.py)
|   `-- promotion_report.md   report on the latest promote.py run
|-- mlflow.db                 MLflow database: run history AND the registry   (DO NOT DELETE)
|-- mlruns/                   MLflow's stored model files                      (DO NOT DELETE)
|-- serving/                  step 5: the API (app.py, export_champion.py, send_sample.py, Dockerfile,
|   |                         requirements-serve.txt, .dockerignore; bundle/ and logs/ are created)
|   `-- ui/                   the web console served at http://localhost:8000/ (index.html, styles.css, app.js)
|-- docker-compose.yml        starts the API container
|-- train.py                  step 1: train and compare models
|-- test.py                   step 2: the evaluation gate
|-- register_champion.py      step 3: put models into the registry
|-- promote.py                step 4: challenger vs champion, promotion, rollback
|-- practice.py               early scratch version of training; NOT part of the pipeline
|-- requirements.txt          the packages the project needs, with exact versions
|-- requirements-lock.txt     every installed package, exact versions (full snapshot)
|-- .venv/                    the virtual environment (disposable, can be rebuilt)
`-- __pycache__/              Python's cache (ignore)
```

Two pairs of files to keep straight:

- `mlflow.db` + `mlruns/` together are the experiment history and the registry. Delete either one
  and the registry is damaged.
- `.venv/` is only installed packages. It is safe to delete and rebuild.

---

## 5. Words you will meet

| Word | Plain meaning |
|---|---|
| **MLOps** | The discipline of managing models like software: tracked, tested, versioned and replaceable. |
| **Train / test / holdout** | Three separate piles of patients. Train: the model learns from it. Test: the final exam, used once to judge models. Holdout: sealed, for a later simulation of real use. |
| **Split by patient** | All stays of one patient go into the same pile, so the model is never tested on a patient it has already seen. |
| **Positive rate** | The share of stays that were readmitted (about 11.5%). Because it is small, plain accuracy is misleading: a model that always says "no" would be 88.5% accurate and useless. |
| **Cross-validation (CV)** | Training on part of the training data and scoring on the rest, repeated 5 times, so every score comes from data the model did not train on. |
| **PR-AUC** | The main score. How well the model puts truly-readmitted stays near the top of its ranking without flagging too many that were not. A model with no skill scores about 0.115 (the positive rate). Higher is better. |
| **ROC-AUC** | The chance that a random readmitted stay gets a higher score than a random non-readmitted one. 0.5 is a coin flip, 1.0 is perfect. |
| **Lift** | PR-AUC divided by the positive rate. 1.0x is no better than chance; 1.9x means about twice as good as chance. |
| **95% range (bootstrap)** | The test data is resampled 200 times (patient by patient) to show how much a score could wobble by luck. If two models' ranges overlap heavily, they are effectively tied. |
| **Paired difference** | The same resampling applied to two models together, giving a range for "how much better is A than B". If the range includes zero, the data cannot tell them apart. |
| **Evaluation gate** | A set of pass/fail rules, written before looking at test results, that a model must meet. |
| **SHAP** | A method that measures how much each input pushed predictions up or down, used to check that a model relies on sensible signals. |
| **Calibration** | Whether a predicted 20% risk really means about 20% of such patients are readmitted. Our scores are not calibrated (see limits). |
| **MLflow** | A tool that records every training run (settings, scores, charts, the model) and holds the model registry. |
| **Experiment vs registry** | The experiment (`readmission-30d`) is the history of every run. The registry (`readmission`) is the short list of models that matter, with labels. |
| **Alias** | A movable label on a registered version: `champion`, `challenger`, `baseline`, `previous_champion`. Code asks for `models:/readmission@champion` and gets whatever currently holds that label. |
| **Champion / challenger / baseline** | Champion: the model in use. Challenger: a candidate that might replace it. Baseline: the simple model every other model must beat. |
| **joblib file** | A saved trained model in a single file. Loading it needs the same package versions used to create it. |
| **Virtual environment (venv)** | A private folder of installed packages for this project, so it does not depend on whatever else is on the computer. |

---

## 6. Setup (once)

**You need:** Windows with PowerShell, Python 3.9.7, and about 5 GB of free disk space.

From the project folder:

```powershell
# create the environment (use your own Python 3.9 path)
& "C:\Users\<you>\AppData\Local\Programs\Python\Python39\python.exe" -m venv .venv

# switch into it; the prompt shows (.venv)
.\.venv\Scripts\Activate.ps1

python -m pip install --upgrade pip
python -m pip install --prefer-binary --no-cache-dir -r requirements.txt
```

If PowerShell refuses to run the activation script:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

For an exact rebuild of every package, use the lock file instead:

```powershell
python -m pip install --prefer-binary --no-cache-dir -r requirements-lock.txt
```

**Verify the setup.** Run `python test.py --no-mlflow`. It should print LightGBM 0.222, XGBoost 0.218
and logistic 0.206. If it does, your environment reproduces the project's results.

**Key package versions** (full list in `requirements.txt`):

| Package | Version | Why it matters |
|---|---|---|
| scikit-learn | 1.5.2 | A newer scikit-learn breaks XGBoost 2.0.3 (see Troubleshooting). |
| xgboost | 2.0.3 | Paired with scikit-learn 1.5.2. |
| lightgbm | 4.6.0 | |
| mlflow | 3.1.4 | Tracking and registry. |
| pandas / numpy | 2.2.3 / 1.26.4 | |
| shap | 0.49.1 | Model explanations. |
| greenlet | 3.2.4 | Pinned because the newest version has no ready-made file for Python 3.9 on Windows. |

**Running from VS Code:** the Run button cannot pass arguments. Scripts that need arguments
(`register_champion.py`) must be run from the terminal.

---

## 7. The pipeline, step by step

Each step below shows what it does, what it reads, what it creates, and how that feeds the next step.

**Where the project plan's 15 steps live.** The plan has three phases (P1, P2 and P4):

| Plan step | What | Where it lives | Status |
|---|---|---|---|
| 1 to 5 (P1) | Look at the data, clean, build the label, leak review, split | `code/` scripts, section "Step 0" below | Done |
| 6 (P1) | Train models | `train.py` | Done |
| 7 (P1) | Track runs | MLflow with a SQLite database (`mlflow.db`) | Done |
| 8 (P2) | Evaluation gate | `test.py` | Done (calibration is reported, not yet a pass/fail rule) |
| 9 (P2) | Register, alias `champion` | `register_champion.py` (plus `promote.py`, added beyond the plan) | Done |
| 10 (P2) | Serve (FastAPI + Docker) | `serving/` folder and `docker-compose.yml`, section 7 step 5 | Built; Docker image not yet built on your machine |
| 11 to 15 (P4) | Replay, API health, data drift, late labels, retrain | not built | Later |

`promote.py` is not one of the 15 plan steps. It adds the champion/challenger rule and rollback, and
it is the manual version of step 15 (retrain, gate, promote).

### Step 0. Data preparation and split (earlier steps)

These are plan steps 1 to 5. In short, they:

1. **Look at the data:** shape, missing share per column, and confirm the full-file numbers.
2. **Clean:** drop patients who died or went to hospice (they cannot be readmitted), group the
   "unknown" ID codes, treat ICD-9 diagnosis codes as strings and group them, drop constant columns.
3. **Build the label:** readmitted within 30 days = 1, anything else = 0. The value `NO` means "none
   observed", which is not proof of no readmission. Check the positive rate.
4. **Review for leakage:** for each column ask "is this known at discharge?", write the reason down,
   and treat a very high AUC as suspect.
5. **Split by patient** (`patient_nbr`): train 60%, test 15%, and "observe" 25%, which is held back.
   The observe part is the **holdout** used for the replay (plan steps 11 to 14). Patient IDs per
   split are saved.

> **TODO for the author:** add the file names of the scripts in `code/` that do this.

The rest of the pipeline only needs this from them:

- `data/splits/train.csv` and `data/splits/test.csv` (and a holdout file), split **by patient**.
- `data/splits/split_report.txt` and `data/cleaning_config.json`, which train.py attaches to each
  MLflow run so the data's origin is recorded.
- Columns: the label `readmitted` (0 or 1), the patient id `patient_nbr`, `encounter_id`,
  `readmitted_raw`, the raw diagnosis codes `diag_1` to `diag_3` (not used as features), 10 numeric
  features, and categorical features. The numeric ones are `time_in_hospital`, `num_lab_procedures`,
  `num_procedures`, `num_medications`, `number_outpatient`, `number_emergency`, `number_inpatient`,
  `number_diagnoses`, `n_drugs_taken` and `n_drugs_changed`.
- The fairness checks in step 2 need the columns `gender`, `race` and `age`.

Sizes at the time of writing: train has 59,753 stays from 42,054 patients (11.49% positive); test has
14,907 stays from 10,483 patients (11.55% positive).

---

### Step 1. `train.py`: train and compare models

**What it does.** Trains three models and measures each one fairly:

| Model | What it is | How it handles the rarity of readmissions |
|---|---|---|
| Logistic regression | Simple weighted-evidence model. The **baseline**. | `class_weight="balanced"` |
| XGBoost | Many small decision trees, each fixing the last one's mistakes. | `scale_pos_weight` = 7.704 |
| LightGBM | Another tree-boosting model, built differently. | `scale_pos_weight` = 7.704 |

Before modelling, the data goes through one shared preprocessing recipe: skewed visit counts are
log-transformed, numbers are standardised, categories are one-hot encoded, and categories with fewer
than 50 rows are merged into one "infrequent" bucket.

Each model is scored with 5-fold cross-validation that is stratified and grouped by patient (seed
42). Then it is refitted on all training data and saved.

**Run it.**

```powershell
python train.py                     # all three models (always do this)
python train.py --cv-folds 3        # quicker check
```

Never run `--models xgboost` on its own: the ranked joblib files are rebuilt from only the models in
that run, so the others would be deleted.

**Reads:** `data/splits/train.csv` only. It never loads test or holdout.

**Creates:**

| File | Contents |
|---|---|
| `models/01_<model>_prauc<cv>.joblib` ... | The saved models, ranked by CV PR-AUC, with the score in the file name. Replaced on every run. |
| `models/best_model.json` | Which model ranked first **by CV only**. It is not the champion. |
| `models/model_report.md` | Plain-language report: results, SHAP and coefficients, limits. |
| `data/training_report.txt` | Technical summary and any failures. |
| MLflow runs in `mlflow.db` / `mlruns/` | Parameters, CV scores per fold, the feature list, SHAP charts, and the full fitted pipeline. |

**If one model crashes,** it is recorded as FAILED and the others still finish.

**How it affects the next step.** `test.py` loads the joblib files. `register_champion.py` later
finds the matching MLflow run by comparing CV scores.

**What to check.** CV PR-AUC should be clearly above 0.115. Current values are about 0.21 to 0.23.
The report's SHAP lists show what each model relies on; sensible models lean on things like prior
inpatient visits rather than tiny rare categories.

---

### Step 2. `test.py`: the evaluation gate

**What it does.** Scores every saved model, unchanged, on the test split, and checks each against
rules that were written down before the first test result was seen. It does not retrain or tune
anything.

**Run it.**

```powershell
python test.py                      # gate every saved model
python test.py --no-mlflow          # same, without logging runs to MLflow
python test.py --reference xgboost  # change which model the others are compared against
```

Without `--no-mlflow`, each run adds three `eval-...` runs to MLflow. They are duplicates if the
numbers did not change.

**The rules** (constants at the top of `test.py`; do not change them after seeing results):

| Check | Rule |
|---|---|
| Lift | at least 1.5x over chance |
| Overfitting | CV score minus test score at most 0.02 |
| Beats the baseline | non-baseline models must score at least as high as logistic on test |
| Fairness (ROC-AUC) | no gender, race or age group more than 0.05 below the overall ROC-AUC |
| Fairness (recall) | when flagging the top 20% by risk, no group's recall more than 0.10 below overall |
| Small groups | groups under 500 stays or 50 readmissions are listed but not judged |

Calibration is reported but not judged, because class weights deliberately inflate the scores.

**The reference model.** For the paired comparison and the fairness table, `test.py` needs one model
to compare the others against. It uses `--reference` if given, otherwise the current champion from
`champion.json`, otherwise the model with the highest test PR-AUC. This is only a comparison anchor.
**The gate never chooses the champion; a person does.**

**Reads:** `data/splits/test.csv`, the joblib files in `models/`, and `champion.json` if it exists.
It checks that the test columns match what each model was trained on and stops with an error if not.

**Creates:**

| File | Contents |
|---|---|
| `models/evaluation.json` | Per-model scores, intervals, gate checks and PASS/FAIL, plus parameters. This is what step 3 reads. |
| `models/test_report.md` | Plain-language gate report. |
| `data/test_report.txt` | Technical report including the full fairness table. |
| MLflow `eval-<model>` runs | Test metrics logged (unless `--no-mlflow`). |

**How it affects the next step.** `register_champion.py` refuses to register any model that did not
PASS here. `promote.py` also imports this file's thresholds and helper functions, so the promotion
rule always uses the same standards as the gate.

**Evaluating on other data (the holdout or new data).** The same rules and the same saved models can be
run on any cleaned file that has the same columns and the `readmitted` outcome:

```powershell
python test.py --data data/splits/holdout.csv --no-mlflow
python test.py --data data/new/new_data.csv --name new_data --no-mlflow
```

- Results go to **separate files named after the data** (`models/evaluation_<name>.json`,
  `models/<name>_report.md`, `data/<name>_report.txt`), so `models/evaluation.json`, which
  `register_champion.py` reads, is never overwritten. The name `test` is reserved for the real test split.
- Every use of any dataset is appended to `models/data_use_log.jsonl`, and a repeat use of the same data
  prints a warning.
- New data must go through the same cleaning as the training data, so the derived columns (such as the
  diagnosis groups) exist. The script stops with a clear message if columns are missing.
- Treat the holdout as single-use: look at it once for a decision made in advance, and do not iterate on it.
- `python promote.py --data <file>` compares the champion and the challenger on the same kind of data.

---

### Step 3. `register_champion.py`: put models in the registry

**What it does.** Records models in the MLflow registry (`readmission`) with labels and the evidence
behind them. It has two modes.

**Champion mode (first-time setup, a human decision):**

```powershell
python register_champion.py --champion lightgbm --reason "Passed the gate; chosen for credible feature reasoning"
```

- Refuses a model that did not pass the gate.
- Finds each model's training run in MLflow, registers one version per model, and sets aliases:
  `champion` (your choice), `baseline` (logistic, always) and `challenger` (other models that
  passed).
- Tags every version with its evidence: gate result, CV and test scores, the training run id, and
  your reason.
- Loads the registered champion and checks that its predictions match the saved joblib exactly.
- Writes `champion.json` and `champion_report.md`.
- Safe to re-run: runs already registered are reused, not duplicated.

**Challenger mode (every later model):**

```powershell
python register_champion.py --challenger xgboost --reason "tried MIN_CATEGORY_ROWS=200"
```

- Registers only that model, as a new version with the alias `challenger`. The champion and baseline
  are untouched.
- Refuses if the model failed the gate, or if it is the exact run that is already champion.
- The previous challenger is not deleted; it is tagged `retired_challenger`.
- Adds a `challenger_registered` line to `promotion_log.jsonl`.

**Safety guard.** After `promote.py` has changed the champion, running champion mode again would
silently reset the labels to the old model, so it refuses unless you add `--force`.

**Reads:** `evaluation.json`, the MLflow runs, and a joblib file (for verification).

**Creates:** registry versions, aliases and tags; `champion.json`; `champion_report.md`.

**How it affects the next step.** `promote.py` and the future serving API read the aliases.

---

### Step 4. `promote.py`: challenger versus champion

**What it does.** Decides whether the challenger should replace the champion, using a rule instead of
a hunch.

```powershell
python promote.py --dry-run             # show the decision, change nothing
python promote.py                       # decide and, if it wins, apply
python promote.py --candidate-version 4 # test a specific registry version
python promote.py --rollback            # put the previous champion back
```

**The rule.** A challenger replaces the champion only if both are true:

1. It passes every gate check (the same ones as `test.py`).
2. Its PR-AUC beats the champion's, and the paired 95% range of the difference is **entirely above
   zero**. A tie is not a reason to switch.

**What it does when the challenger wins:**

- The challenger gets `@champion`; the old champion becomes `@previous_champion`; the `@challenger`
  label is removed.
- Version tags are updated, and `champion.json` and `champion_report.md` are rewritten.
- A line is added to `promotion_log.jsonl`.

**When it does not win:** only a log line and a new `promotion_report.md`. The registry is untouched.

**Rollback.** `--rollback` swaps `@previous_champion` back to `@champion` and updates the files.

**Reads:** the registry (by alias), `test.py`'s rules, and the test data.

**Creates:** `promotion_log.jsonl` (one line per real decision; dry runs are not logged) and
`promotion_report.md`. On a promotion or rollback it also rewrites `champion.json` and
`champion_report.md`.

**Caution.** By default it compares on the test split, which has already been used to choose between
models. Each extra comparison makes results slightly optimistic. Use fresh data (`--data`) or the
holdout for the final check when you have it.

---

### Step 5. `serving/`: the API (plan step 10)

**What it does.** Serves the champion over HTTP. A client sends the cleaned feature columns of one or more
stays; the API returns a risk score, a high-risk flag and the top reasons (SHAP drivers) for each.

**Why a bundle.** The API does not read MLflow. `serving/export_champion.py` loads `models:/readmission@champion`
on your machine and writes `serving/bundle/model.joblib` plus `meta.json` (version, evidence, high-risk cutoff,
package versions, a checksum). The API loads the bundle. This avoids two problems: MLflow stores Windows
paths (`C:\Users\...`) that do not exist inside a Linux container, and SQLite does not behave well on mounted
Windows folders.

```powershell
pip install prometheus-client==0.21.1          # once, in your .venv (only needed to run the API outside Docker)
python serving/export_champion.py              # registry @champion -> serving/bundle
docker compose up -d --build                   # build and start the API on http://localhost:8000
python serving/send_sample.py --n 20           # send 20 test rows and print the answers
```

Open http://localhost:8000/ for the web console, or http://localhost:8000/docs for the plain interactive API documentation. Without Docker:
`python -m uvicorn app:app --app-dir serving --port 8000` from the project folder.

| Endpoint | What it does |
|---|---|
| `GET /` | The web console (see below). |
| `POST /predict` | Score, `high_risk` flag and top SHAP drivers for 1 to 500 records. |
| `GET /example` | A ready-to-send request body. |
| `GET /model-info` | Which model is live, its evidence, the required columns, the flag rule. |
| `GET /health` | 200 if a model is loaded, otherwise 503 (Docker uses this). |
| `GET /metrics` | Prometheus metrics: latency, errors, volume, score distribution, model version, unseen categories. |
| `POST /reload` | Re-read the bundle after a new export. A bad bundle is refused and the old model keeps serving. |

**The web console** (`serving/ui/`, served by the API itself, no extra server) has four views:

| View | What it does |
|---|---|
| Score a stay | A form with dropdowns built from the values the model saw in training. "Load example" fills it in. The result shows the score on a meter with the cutoff marked, a High risk / Not flagged verdict, and the main drivers as bars (right raises risk, left lowers it). The raw request and response are one click away. |
| Score a file | Upload a CSV of cleaned stays. Outcome columns are removed and bad cells are reported with their file row. You get summary cards, a score histogram and a table of the 100 highest scores. A search box finds any stay ID in the whole result, and clicking a row opens that stay in "Score a stay" with its full explanation. "Download results (CSV)" gives every stay: `file_row, id, score, high_risk` (plus three driver columns if "Include top driver" was ticked). By default the first 5,000 rows are scored; raise "Rows to score" for the whole file. |
| Model | The live version, its evidence from the registry (gate, test scores, why it was chosen), the high-risk rule, the required columns, and a Reload button. |
| Monitoring | The `/metrics` numbers as cards: stays scored, share flagged, response time, rejected requests, unseen categories, plus the score distribution and a requests table. It refreshes every 5 seconds. |

It follows the system light or dark setting, and it uses no external fonts or libraries, so it also works offline and inside Docker.

**Health checks.** `GET /health` answers 200 with the live model version when a model is loaded and 503 when it is not. Three things use it: the console's status pill (every 15 seconds), Docker's `HEALTHCHECK` (every 30 seconds; Docker marks the container "unhealthy" but does not restart it by itself), and, later, Prometheus and Grafana (step 12). It tells you the service is up and has a model. It does not tell you the predictions are good: for that, watch the Monitoring tab (flagged share near 20%, no rejected requests, steady response time, few unseen categories) and, from steps 13 and 14, drift and late-label performance.

**Rules the API enforces.** Every feature column must be present, with no nulls; numeric columns must hold
numbers; categories are matched the way the model saw them in training (so `1`, `1.0` and `"1"` mean the same);
a category never seen in training is still scored (it falls into the "infrequent" bucket) and is reported in the
response and in `/metrics`. Extra columns are ignored. An `encounter_id` in a record is returned as `id`.

**The score is a ranking score, not a probability** (class weights inflate it). The high-risk flag means "score
at or above the cutoff", where the cutoff is the score that the top 20% of reference (test) stays reach.
Drivers are in log-odds units: positive raises the predicted risk.

**Prediction log.** Every scored record is appended to `serving/logs/predictions.jsonl` (id, score, flag, model
version, and the features). Steps 13 and 14 (drift, late labels) will read it. It holds patient-level data, so
treat it like the data it came from, or set `LOG_PREDICTIONS: "false"`.

**After a promotion or rollback:** `python serving/export_champion.py`, then `POST http://localhost:8000/reload`
(no rebuild or restart needed, because the bundle is a mounted folder). `promote.py` prints this reminder.

**Reads:** the registry (export only), then the bundle (API). **Creates:** `serving/bundle/`,
`serving/logs/predictions.jsonl`, and Prometheus metrics.

---

## 8. How everything affects everything else

### 8.1 The chain

```
train.py
  writes  models/*.joblib      ---> read by test.py (and by register_champion for verification)
  writes  MLflow runs          ---> found by register_champion (matched on CV score)

test.py
  writes  evaluation.json      ---> read by register_champion (gate result, scores, file names)
  reads   champion.json        ---> to pick its reference model
  defines gate thresholds      ---> imported by promote.py

register_champion.py
  writes  registry aliases     ---> read by promote.py and (later) serving
  writes  champion.json        ---> read by test.py and humans

promote.py
  reads   registry, test.py rules, test data
  writes  aliases, champion.json, logs, reports
```

### 8.2 Which file is the source of truth for what

| Question | Authoritative answer | Not authoritative |
|---|---|---|
| Which model is champion right now? | The registry alias `@champion` (MLflow **Models** tab) | `champion.json` is a human-readable copy the scripts keep in sync. `best_model.json` is only the top of the CV ranking. |
| Which models were trained last? | `models/*.joblib` and the MLflow runs | The registry may still hold older versions. |
| Did a model pass the gate last time? | `evaluation.json` (the last `test.py` run only) | Older reports are overwritten. |
| What decisions were made? | `promotion_log.jsonl` | The reports are overwritten each run. |

### 8.3 If you change X, then rerun Y

| If you change... | Then... |
|---|---|
| The data cleaning or the split | Rerun everything from `train.py`. Old scores are not comparable with new ones, so new models enter as challengers. |
| Model settings or features in `train.py` | `train.py` (all three) -> `test.py` -> `register_champion.py --challenger` -> `promote.py`. |
| Gate thresholds in `test.py` | They change both the gate and `promote.py`, which imports them. Changing them after seeing results defeats the purpose of writing them in advance. |
| The random seed | Every CV split and bootstrap changes, so scores shift slightly. |
| Package versions | Saved models may fail to load or predict differently. Update the requirements files, then retrain and re-verify. |
| `mlflow.db` or `mlruns/` (deleting) | The registry and run history are gone. The joblib files survive, but `register_champion.py` cannot find runs. |
| `models/*.joblib` (deleting) | `test.py` cannot run, and verification in `register_champion.py` fails. The registry itself is unaffected. |
| An alias edited by hand in the MLflow UI | `promote.py` still obeys the registry, but `champion.json` and the reports go stale. |

### 8.4 Hidden couplings worth knowing

1. **`promote.py` imports `test.py` by file path.** They must stay in the same folder. (The file is
   named `test.py`, which clashes with a Python built-in package, so it is loaded by path.)
2. **`test.py` reads `champion.json`** to choose its reference model.
3. **`register_champion.py` finds a model's training run by matching CV scores:** the score from
   `evaluation.json` against each run's recorded score (tolerance 0.0006), taking the latest finished
   run of that model family. If `evaluation.json` is out of date relative to MLflow, no run matches
   and it stops with a clear message.
4. **The registry holds its own copy of each model** (inside `mlruns/`), separate from the joblib
   files. Replacing the joblib files by retraining does not change what the registry serves.
5. **`train.py` renumbers and replaces the joblib files every run.** File names embed the rank and the
   CV score, and `evaluation.json` stores those names, so after a retrain it is stale until
   `test.py` is run again.
6. **Everything is seeded (42),** so a rerun on the same data and packages reproduces the same
   numbers. A different result means something changed.
7. **Test-set reuse.** Every comparison on the test split slightly overfits it. That is why the
   promotion rule demands a clear win, not a tie.

---

## 9. What happens in each case

| Case | What changes | What does not change |
|---|---|---|
| A challenger clearly beats the champion | Aliases swap; `champion.json`, `champion_report.md`, the log and `promotion_report.md` are updated. | The joblib files, `evaluation.json` and `test_report.md`. |
| A challenger does not clearly win | One log line and a new `promotion_report.md`. | The registry. |
| The new champion turns out to be bad | `promote.py --rollback` restores the previous champion and updates the files. | The registered versions. |
| You retrain | `train.py` replaces the joblibs; `test.py` replaces `evaluation.json`. | The registry still holds the old versions until a challenger is promoted. |
| A model fails the gate | It cannot be registered as champion or challenger. The baseline is always registered. | Everything else. |
| One model crashes during training | It is marked FAILED; the others finish. | The other models. |
| `train.py --models xgboost` alone | The other ranked joblib files are deleted. | The registry. |
| `register_champion.py --champion` after a promotion | Refused unless `--force`. | Everything. |
| `register_champion.py --challenger` on the current champion's run | Refused: retrain first. | Everything. |
| Packages change | Saved models may not load or may behave differently. | The registry's records. |

---

## 10. Recipes

### A. First run from scratch

```powershell
.\.venv\Scripts\Activate.ps1
python train.py
python test.py
python register_champion.py --champion lightgbm --reason "your reason"
python promote.py --dry-run
```

### B. Try an improved model

1. Change a setting in `train.py` (for example `MIN_CATEGORY_ROWS`, or `min_child_weight` for XGBoost).
2. `python train.py` (all three models together).
3. `python test.py`
4. `python register_champion.py --challenger xgboost --reason "what you changed"`
5. `python promote.py --dry-run`, then `python promote.py` if you agree.

A new model always enters as a challenger, so it can never skip the promotion rule.

### C. Undo a promotion

```powershell
python promote.py --rollback
```

### D. See who the champion is

- Read `models/champion_report.md`, or
- open the MLflow UI, click **Models**, then **readmission**, and look at the Aliases column, or
- run:

```powershell
python -c "import mlflow; mlflow.set_tracking_uri('sqlite:///mlflow.db'); c=mlflow.MlflowClient(); print('champion is version', c.get_model_version_by_alias('readmission','champion').version)"
```

### E. Rebuild on a new machine

Install Python 3.9.7, copy the project folder (including `mlflow.db`, `mlruns/`, `models/`, `data/`),
create a venv, install from `requirements-lock.txt`, and run `python test.py --no-mlflow`. The numbers
should match section 11.

---

## 11. Current results

As of 2026-10-22. All numbers are on the test split (14,907 stays, no-skill PR-AUC 0.116).

| Model | CV PR-AUC | Test PR-AUC | 95% range | ROC-AUC | Lift | Gate | Registry |
|---|---|---|---|---|---|---|---|
| LightGBM | 0.224 | 0.222 | 0.200 to 0.247 | 0.670 | 1.93x | PASS | v2, `@champion` |
| XGBoost | 0.226 | 0.218 | 0.197 to 0.240 | 0.670 | 1.89x | PASS | v1, `@challenger` |
| Logistic | 0.211 | 0.206 | 0.188 to 0.229 | 0.660 | 1.78x | FAIL (fairness) | v3, `@baseline` |

**How the champion was chosen.** LightGBM and XGBoost are statistically tied (paired difference
-0.005, range -0.010 to +0.001). XGBoost ranked first by CV by 0.002, but its SHAP ranking leaned on
tiny rare categories, while LightGBM relied on sensible signals (prior inpatient visits, discharge
disposition, lab procedures). LightGBM was chosen by a person on that basis, and it also has more
room under the fairness limits. `promote.py` confirms that XGBoost does not clearly beat it.

**Fairness.** The champion's weakest group is patients aged 80 to 90 (ROC-AUC 0.630 against 0.670
overall). Small groups (some races, the youngest and oldest age bands) are too small to judge.

---

## 12. Limits and rules to respect

- **Not a clinical tool.** Not validated for use with real patients.
- **Scores are not probabilities.** Class weights inflate them (mean predicted risk about 0.43
  against an actual rate of 0.116). Use them to rank and flag, not as "a 43% chance", until a
  calibration step is added.
- **The test split is spent for selection.** Do not tune models on it. Use fresh data or the holdout
  for further checks.
- **Never loosen a gate threshold after seeing results.**
- **No dates in the data,** so behaviour over time cannot be tested yet.
- **To confirm:** the plan's cleaning step drops expired and hospice discharges (they cannot be
  readmitted). Check `data/cleaning_config.json` to confirm it was applied.
- **Rare categories** inflate some feature rankings (especially for XGBoost and logistic).
  Increasing `MIN_CATEGORY_ROWS` is a candidate experiment.

---

## 13. What is not built yet

In the order planned:

1. **Serving API.** Loads `models:/readmission@champion`, validates the input columns, returns a risk
   score and a high-risk flag, and reports which model version is live.
2. **Monitoring.** Log predictions; watch input drift, score drift, and real performance once
   30-day outcomes arrive.
3. **Holdout replay (step 11).** Feed the sealed holdout through in batches as if it were live data,
   inject drift, and confirm monitoring and rollback behave.
4. **Automation.** One command to run the flow, tests, Docker, scheduled retraining.
5. **Model improvement.** Calibration and the experiments in section 12, run as challengers through
   the loop above.

---

## 14. Troubleshooting

Errors that have actually come up in this project:

| Message | Meaning and fix |
|---|---|
| `Pipeline should either be a classifier ... Got a regressor with response_method=predict_proba` | scikit-learn is newer than this XGBoost version supports. Use scikit-learn 1.5.2 with xgboost 2.0.3. |
| `UNIQUE constraint failed: metrics...` (MLflow) | A model produced NaN scores and MLflow could not store them. `train.py` is set to raise the real error instead, so read the first error above it. |
| `OSError: [Errno 28] No space left on device` | The disk is full. Free about 5 GB, then install with `--no-cache-dir`. Do not write to MLflow while the disk is full. |
| `Failed building wheel for greenlet` / `Microsoft Visual C++ 14.0 or greater is required` | Pip is trying to compile a package. Install with `--prefer-binary` or from `requirements-lock.txt`. You do not need the C++ build tools. |
| `ModuleNotFoundError: No module named 'joblib'` (or similar) | You are in an empty or wrong environment. Activate `.venv` and install the requirements. |
| `the following arguments are required: --champion, --reason` | `register_champion.py` needs arguments, so run it from the terminal and name the model and the reason. |
| `REFUSED: ... did not pass the evaluation gate` | The model failed the gate in `evaluation.json`. Do not register it. |
| `REFUSED: this exact training run is already the champion` | There is no new model to register. Retrain first. |
| `REFUSED: promote.py has changed the champion ...` | Re-running champion mode would reset the labels. Use `--force` only if you really mean to. |
| `No finished training run for '<model>' matches CV PR-AUC ...` | `evaluation.json` and the MLflow runs are out of step. Run `train.py`, then `test.py`, then try again. |
| `VERIFY FAILED: ... differs from ...joblib` | The registered model and the saved file do not match, which usually means the files are out of sync. Retrain and re-register. |
| `Missing .../evaluation.json. Run test.py first.` | Run `test.py` before registering. |
| The page at `/` shows a banner "No model is loaded yet" | There is no bundle. Run `python serving/export_champion.py`, then press Reload on the Model tab. |
| The page at `/` is unstyled, empty, or shows "Loading the form…" forever | The `serving/ui/` folder is missing or incomplete (it needs `index.html`, `styles.css` and `app.js`), or the browser cached an old copy: press Ctrl+F5. |
| API `/health` returns 503, or `No model is loaded` | There is no bundle yet. Run `python serving/export_champion.py`, then `POST /reload`. |
| `/predict` returns 422 | The message names the column and the problem (missing, null, not numeric). Fix the request; nothing is wrong with the server. |
| `/reload` returns 500 with `still_serving_version` | The new bundle is incomplete or corrupt. The old model keeps serving. Re-run the export. |
| `package_warnings` in `/model-info` | The packages in the container differ from those the bundle was made with. Rebuild the image from `requirements-serve.txt`. |
| MLflow UI port already in use | Add `--port 5001` to the `mlflow ui` command and open that port. |

---

## 15. Reference: every file, who writes it, who reads it

| File | Written by | Read by | Safe to delete? |
|---|---|---|---|
| `data/splits/*.csv`, `split_report.txt`, `data/cleaning_config.json` | earlier steps | `train.py`, `test.py` | No (they are the data) |
| `models/*.joblib` | `train.py` | `test.py`, `register_champion.py` (verification) | Yes, but rerun training |
| `models/best_model.json` | `train.py` | people only | Yes (rewritten) |
| `models/model_report.md`, `data/training_report.txt` | `train.py` | people | Yes (rewritten) |
| `models/evaluation.json` | `test.py` | `register_champion.py` | Yes, rerun `test.py` |
| `models/test_report.md`, `data/test_report.txt` | `test.py` | people | Yes (rewritten) |
| `models/evaluation_<name>.json`, `<name>_report.md`, `data/<name>_report.txt` | `test.py --data` | people | Yes (rewritten) |
| `models/data_use_log.jsonl` | `test.py` (every run) | people (who looked at which data) | Avoid: it is your history |
| `models/champion.json`, `models/champion_report.md` | `register_champion.py`, `promote.py` | `test.py` (reference choice), people | Rewritten on the next change; keep for the record |
| `models/promotion_log.jsonl` | `promote.py`, `register_champion.py --challenger` | people (audit trail) | Avoid: it is your history |
| `serving/ui/index.html`, `styles.css`, `app.js` | you (part of the code) | the browser, via the API | No (the console is these files) |
| `serving/bundle/model.joblib`, `meta.json` | `serving/export_champion.py` | the API | Yes, re-export from the registry |
| `serving/logs/predictions.jsonl` | the API | steps 13 and 14 (later) | Avoid: it is the live-traffic record |
| `models/promotion_report.md` | `promote.py` | people | Yes (rewritten) |
| `mlflow.db`, `mlruns/` | MLflow (all scripts) | all scripts | **No** |
| `requirements.txt`, `requirements-lock.txt` | you | pip | No |
| `.venv/` | `python -m venv` | Python | Yes (rebuild from requirements) |
| `practice.py` | you (scratch) | nothing | Yes |




"""
# 1. switch into the environment (the prompt should show (.venv))
.\.venv\Scripts\Activate.ps1

# 2. one-time install, only if you haven't done it yet
pip install prometheus-client==0.21.1

# 3. export the current champion from the registry into the API's bundle
python serving/export_champion.py

# 4. start the API (this window stays busy; leave it open)
python -m uvicorn app:app --app-dir serving --port 8000
"""