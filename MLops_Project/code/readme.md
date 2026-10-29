# Hospital readmission MLOps project

Predict which diabetic hospital stays will end in a readmission within 30 days, and manage the
models behind that prediction in a disciplined, repeatable way: train, gate, register, promote,
serve over an API in Docker, and monitor with Prometheus and Grafana.

This README explains the whole project from start to end for someone who has not seen it before. It covers
how to run it from a fresh clone, what every file does, what the code inside each script does, and how a
change in one place affects the rest.

Last updated: 2026-10-28.

> **Not a clinical tool.** The model has not been validated for use with real patients.
> The data is the public UCI "Diabetes 130-US Hospitals (1999-2008)" dataset (CC BY 4.0, https://doi.org/10.24432/C5230J).

---

## Contents

1. [What this project is](#1-what-this-project-is)
2. [The big picture](#2-the-big-picture)
3. [Run it from a fresh clone](#3-run-it-from-a-fresh-clone)
4. [Folder map](#4-folder-map)
5. [Words you will meet](#5-words-you-will-meet)
6. [Setup details](#6-setup-details)
7. [Docker and Docker Desktop](#7-docker-and-docker-desktop)
8. [The pipeline, step by step](#8-the-pipeline-step-by-step)
9. [Monitoring (Prometheus and Grafana)](#9-monitoring-prometheus-and-grafana)
10. [How everything affects everything else](#10-how-everything-affects-everything-else)
11. [What happens in each case](#11-what-happens-in-each-case)
12. [Recipes](#12-recipes)
13. [Current results](#13-current-results)
14. [Limits and rules to respect](#14-limits-and-rules-to-respect)
15. [Status and what is not built yet](#15-status-and-what-is-not-built-yet)
16. [Troubleshooting](#16-troubleshooting)
17. [File-by-file guide (what each file does, in detail)](#17-file-by-file-guide)
18. [Function-level walkthrough of the code](#18-function-level-walkthrough-of-the-code)
19. [Publishing to GitHub](#19-publishing-to-github)

---

## 1. What this project is

**The prediction task.** Each row of data is one hospital stay of a diabetic patient. The label is whether the
patient came back within 30 days (about 11.5% did). A model gives every stay a risk score so that the
highest-risk stays can be flagged.

**The real point: MLOps.** MLOps is the machinery around a model. Training a model once is easy. The hard part
is being able to answer, months later:

- Which model is in use, and why was it chosen?
- Was it tested on data it had never seen, using rules written down beforehand?
- Can a newer model replace it safely, and can we undo that?
- Can someone else rebuild everything and get the same numbers?
- Is the live service healthy, and are its predictions still behaving normally?

This project answers those questions with scripts for each stage, an experiment tracker and model registry
(MLflow), an HTTP API with a web console (FastAPI, in Docker), and a monitoring stack (Prometheus, Grafana).

**An honest note on model quality.** Readmission is hard to predict from hospital records. The current best model
finds readmissions about 1.9 times better than random flagging, with a ROC-AUC of about 0.67. That is typical for
this kind of data and is not a sign of a bug.

---

## 2. The big picture

```
 raw data (data/diabetic_data.csv, from UCI)
    |  code/data_cleaning.py  ->  code/leak_review.py (checks)  ->  code/data_split.py   (split BY PATIENT)
    v
 data/splits/   train.csv        test.csv         holdout.csv
                   |                |                |
         train.py  |                |                | (only used by
                   v                |                |  serving/replay.py, step 11)
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
                  ^          |
                  |          |  serving/export_champion.py: registry @champion -> serving/bundle/
                  |          v
   promote.py compares   Docker: readmission-api (FastAPI + web console)   <--- scraped every 15 s by
   challenger vs champion   |  writes serving/logs/predictions.jsonl               Prometheus ---> Grafana
   and swaps aliases ONLY   |  exposes /predict /health /metrics /reload
   if the challenger clearly wins (and can roll back)
```

Three ideas hold everything together:

1. **Separate data for separate jobs.** Train data is for learning. Test data is for the final exam. The holdout
   is kept sealed for later. No script is allowed to use the wrong one.
2. **Gate, then register, then promote.** A model must pass a pre-written test before it can be registered, and
   must clearly beat the current champion before it replaces it.
3. **The registry is the source of truth.** "Which model is champion?" is answered by the MLflow registry, not by
   a file name. The API serves an exported copy (a "bundle") of whatever is `@champion`.

---

## 3. Run it from a fresh clone

The repository does **not** contain the data, the trained models, or the MLflow database (see
[section 19](#19-publishing-to-github)). You rebuild them in a few commands. Everything is seeded (42), so a
rebuild reproduces the numbers in [section 13](#13-current-results).

### 3.1 What you need

| Need | Why |
|---|---|
| Python 3.9 (the project was built on 3.9.7) | The pinned packages are matched to it |
| About 5 GB free disk | Packages, MLflow artifacts, Docker images |
| Docker (Docker Desktop on Windows or macOS, Docker Engine on Linux) | Only for the API and monitoring (steps 3.6 onwards) |
| Git | To clone |

### 3.2 Clone and create the environment

Windows (PowerShell):

```powershell
git clone <your-repo-url> MLops_Project
cd MLops_Project
python -m venv .venv                       # use a Python 3.9 interpreter
.\.venv\Scripts\Activate.ps1               # prompt shows (.venv)
python -m pip install --upgrade pip
python -m pip install --prefer-binary --no-cache-dir -r requirements.txt
```

macOS / Linux:

```bash
git clone <your-repo-url> MLops_Project
cd MLops_Project
python3.9 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install --prefer-binary --no-cache-dir -r requirements.txt
```

If PowerShell refuses to run the activation script: `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass`.
`requirements-lock.txt` is a full snapshot of the original Windows environment (it includes Windows-only packages
such as `pywin32`), so use it only on Windows. On macOS or Linux use `requirements.txt`.

### 3.3 Get the data

Download the UCI dataset (https://doi.org/10.24432/C5230J, "Diabetes 130-US Hospitals for years 1999-2008") and
put two files in `data/`:

```
data/diabetic_data.csv
data/IDS_mapping.csv
```

### 3.4 Prepare the data (earlier steps; run from the project root)

```bash
python code/data_cleaning.py     # raw -> data/diabetic_data_cleaned.csv (+ cleaning_config.json, cleaning_report.txt)
python code/leak_review.py       # optional: leak checks -> data/leak_review_report.txt
python code/data_split.py        # -> data/splits/train.csv, test.csv, holdout.csv, split_assignments.csv, split_report.txt
```

Expected sizes: train 59,753 stays / 42,054 patients, test 14,907 stays / 10,483 patients.

### 3.5 Train, gate, register, compare

```bash
python train.py                                  # 1. train all three models (always all three)
python test.py                                   # 2. the gate: score them on the test split
python register_champion.py --champion lightgbm --reason "Passed the gate; chosen for credible feature reasoning"
python promote.py --dry-run                      # 4. would the challenger (XGBoost) replace the champion?
```

`python test.py --no-mlflow` should show LightGBM 0.222, XGBoost 0.218 and logistic 0.206 test PR-AUC. If it does,
your environment reproduces the project.

To browse runs and the registry:

```bash
mlflow ui --backend-store-uri sqlite:///mlflow.db     # open http://127.0.0.1:5000 (Experiments and Models tabs)
```

### 3.6 Serve the champion (Docker)

```bash
python serving/export_champion.py        # registry @champion -> serving/bundle/ (model.joblib + meta.json)
docker compose up -d --build             # builds the API image, starts API + Prometheus + Grafana
python serving/send_sample.py --n 20     # send 20 test rows, print the answers
```

Start Docker Desktop first on Windows or macOS (see [section 7](#7-docker-and-docker-desktop)).

| Thing | Address | Login |
|---|---|---|
| Web console | http://localhost:8000/ | none |
| API docs (Swagger) | http://localhost:8000/docs | none |
| Health | http://localhost:8000/health | none |
| Prometheus | http://localhost:9090 (Status > Targets, Alerts) | none |
| Grafana | http://localhost:3000 | admin / admin (change it; see section 9) |

Stop everything: `docker compose down` (keeps metric history) or `docker compose down -v` (deletes it).

### 3.7 Serve without Docker (optional)

```bash
pip install -r serving/requirements-serve.txt
python serving/export_champion.py
python -m uvicorn app:app --app-dir serving --port 8000
```

### 3.8 Replay the holdout (step 11)

With the stack up and a bundle exported (3.6):

```bash
python serving/replay.py --preset demo --dry-run                           # show the 12-week plan, send nothing
python serving/replay.py --preset demo --run-id demo1 --seconds-per-week 120   # about 24 minutes
```

Watch Grafana (http://localhost:3000) and http://localhost:9090/alerts while it runs. Results are written to
`data/replay/demo1/` (see [Step 6](#step-6-servingreplaypy-the-holdout-replay-plan-step-11)). To rehearse a bad model
and the rollback under traffic, start `python serving/replay.py --run-id drill3 --weeks 8 --seconds-per-week 90`, then in a second
window run `python serving/rollback_drill.py --start` as week 3 begins and `--finish` as week 7 begins.
To see real before/after rows for every test without sending anything: `python serving/replay.py --preset demo --show-examples`.

### 3.9 Drift report (step 13)

After a replay (3.8), from the project root:

```bash
python monitoring/drift_report.py --run-id demo1 --baseline-weeks 1-4
```

Open `data/drift/demo1/drift_report.md` and `drift_heatmap.png` (see [Step 7](#step-7-monitoringdrift_reportpy-the-drift-report-plan-step-13)).

### 3.10 After a promotion or rollback

```bash
python serving/export_champion.py        # refresh the bundle from the registry
curl -X POST http://localhost:8000/reload    # or press "Reload model" on the console's Model tab
```

No rebuild is needed, because `serving/bundle/` is a mounted folder.

---

## 4. Folder map

```
MLops_Project/
|-- code/                       earlier-step scripts (data preparation)
|   |-- data_cleaning.py        raw CSV -> cleaned CSV              (+ data_cleaning.md: explanation)
|   |-- leak_review.py          checks every column for leakage     (+ leak_review.md: explanation)
|   `-- data_split.py           cleaned CSV -> train / test / holdout, split by patient
|-- data/                       (not in git) data and data reports
|   |-- diabetic_data.csv, IDS_mapping.csv    raw UCI files you download
|   |-- diabetic_data_cleaned.csv             output of data_cleaning.py
|   |-- replay/<run-id>/        written by serving/replay.py: plan.json, labels.csv, summary.json, summary.md
|   |-- drift/<run-id>/         written by monitoring/drift_report.py: drift_report.md, psi_by_week.csv, drift_summary.json, drift_heatmap.png
|   |-- splits/                 train.csv, test.csv, holdout.csv, split_assignments.csv, split_report.txt
|   |-- cleaning_config.json    record of how the data was cleaned
|   |-- cleaning_report.txt, leak_review_report.txt, leak_review_columns.csv
|   |-- training_report.txt     written by train.py (technical summary)
|   `-- test_report.txt         written by test.py (technical summary)
|-- models/                     (not in git) everything the pipeline produces
|   |-- 01_<model>_prauc0.xxx.joblib ...   saved models, ranked by CV score (train.py)
|   |-- best_model.json         which model ranked first by CV only (train.py)
|   |-- model_report.md         plain-language training report (train.py)
|   |-- evaluation.json         gate results, machine-readable (test.py)
|   |-- test_report.md          plain-language gate report (test.py)
|   |-- champion.json           the current champion, machine-readable
|   |-- champion_report.md      the current champion, plain language
|   |-- promotion_log.jsonl     audit trail, one line per decision
|   |-- data_use_log.jsonl      every time any dataset was scored (test.py, export_champion.py)
|   `-- promotion_report.md     report on the latest promote.py run
|-- mlflow.db                   (not in git) MLflow database: run history AND the registry   (DO NOT DELETE)
|-- mlruns/                     (not in git) MLflow's stored model files                      (DO NOT DELETE)
|-- serving/                    step 10: the API
|   |-- app.py                  the FastAPI application
|   |-- export_champion.py      registry @champion -> bundle/
|   |-- send_sample.py          sends CSV rows to the running API
|   |-- replay.py               step 11: replays the holdout week by week, with drift scenarios
|   |-- rollback_drill.py       step 11: puts a worse model live on purpose, then rolls back
|   |-- Dockerfile              how the API image is built
|   |-- requirements-serve.txt  pinned packages inside the container
|   |-- .dockerignore           files kept out of the image
|   |-- ui/                     the web console: index.html, styles.css, app.js
|   |-- bundle/                 (created, not in git) model.joblib + meta.json
|   `-- logs/                   (created, not in git) predictions.jsonl
|-- monitoring/                 steps 12 and 13
|   |-- README.md
|   |-- drift_report.py         step 13: per-week drift report (PSI) from the prediction log
|   |-- prometheus/             prometheus.yml, alerts.yml (5 real-traffic rules), alerts_replay.yml (4 replay rules)
|   `-- grafana/                provisioning/datasources/prometheus.yml, starter_dashboard.json
|-- docker-compose.yml          starts API + Prometheus + Grafana with one command
|-- train.py                    step 1: train and compare models
|-- test.py                     step 2: the evaluation gate
|-- register_champion.py        step 3: put models into the registry
|-- promote.py                  step 4: challenger vs champion, promotion, rollback
|-- requirements.txt            packages needed for the pipeline, exact versions
|-- requirements-lock.txt       full Windows snapshot of every installed package
|-- .gitignore                  keeps data, models, MLflow files and logs out of git
|-- .venv/                      (not in git) the virtual environment (disposable)
`-- __pycache__/                (not in git) Python's cache
```

Two pairs of files to keep straight:

- `mlflow.db` + `mlruns/` together are the experiment history and the registry. Delete either one and the
  registry is damaged.
- `.venv/` is only installed packages. It is safe to delete and rebuild.

---

## 5. Words you will meet

| Word | Plain meaning |
|---|---|
| **MLOps** | The discipline of managing models like software: tracked, tested, versioned and replaceable. |
| **Train / test / holdout** | Three separate piles of patients. Train: the model learns from it. Test: the final exam, used to judge models. Holdout: sealed, for a later simulation of real use. |
| **Split by patient** | All stays of one patient go into the same pile, so the model is never tested on a patient it has already seen. |
| **Positive rate** | The share of stays that were readmitted (about 11.5%). Because it is small, plain accuracy is misleading: a model that always says "no" would be 88.5% accurate and useless. |
| **Cross-validation (CV)** | Training on part of the training data and scoring on the rest, repeated 5 times, so every score comes from data the model did not train on. |
| **PR-AUC** | The main score. How well the model puts truly-readmitted stays near the top of its ranking without flagging too many that were not. No skill scores about 0.115 (the positive rate). Higher is better. |
| **ROC-AUC** | The chance that a random readmitted stay gets a higher score than a random non-readmitted one. 0.5 is a coin flip, 1.0 is perfect. |
| **Lift** | PR-AUC divided by the positive rate. 1.0x is no better than chance; 1.9x means about twice as good as chance. |
| **95% range (bootstrap)** | The test data is resampled 200 times (patient by patient) to show how much a score could wobble by luck. |
| **Paired difference** | The same resampling applied to two models together, giving a range for "how much better is A than B". If the range includes zero, the data cannot tell them apart. |
| **Evaluation gate** | A set of pass/fail rules, written before looking at test results, that a model must meet. |
| **SHAP** | A method that measures how much each input pushed a prediction up or down. |
| **Calibration** | Whether a predicted 20% risk really means about 20% of such patients are readmitted. Our scores are not calibrated (see limits). |
| **MLflow** | A tool that records every training run (settings, scores, charts, the model) and holds the model registry. |
| **Experiment vs registry** | The experiment (`readmission-30d`) is the history of every run. The registry (`readmission`) is the short list of models that matter, with labels. |
| **Alias** | A movable label on a registered version: `champion`, `challenger`, `baseline`, `previous_champion`. Code asks for `models:/readmission@champion` and gets whatever currently holds that label. |
| **Champion / challenger / baseline** | Champion: the model in use. Challenger: a candidate that might replace it. Baseline: the simple model every other model must beat. |
| **joblib file** | A saved trained model in a single file. Loading it needs the same package versions used to create it. |
| **Bundle** | `serving/bundle/`: the exported champion (`model.joblib` plus `meta.json`) that the API loads. The API never talks to MLflow. |
| **Image / container / compose** | An image is a packaged recipe-result (OS + Python + packages + code). A container is a running copy of an image. Compose starts several containers together from one file. |
| **Prometheus / Grafana** | Prometheus scrapes the API's `/metrics` every 15 s and stores the history. Grafana draws charts from it. |
| **Virtual environment (venv)** | A private folder of installed packages for this project. |

---

## 6. Setup details

**Key package versions** (full list in `requirements.txt`):

| Package | Version | Why it matters |
|---|---|---|
| scikit-learn | 1.5.2 | A newer scikit-learn breaks XGBoost 2.0.3 (see Troubleshooting). |
| xgboost | 2.0.3 | Paired with scikit-learn 1.5.2. |
| lightgbm | 4.6.0 | |
| mlflow | 3.1.4 | Tracking and registry. |
| pandas / numpy | 2.2.3 / 1.26.4 | |
| shap | 0.49.1 | Model explanations. |
| prometheus-client | 0.21.1 | The API's `/metrics`. |
| greenlet | 3.2.4 | Pinned because the newest version has no ready-made file for Python 3.9 on Windows. |

The container installs its own pinned set from `serving/requirements-serve.txt` (same versions for the model
packages, plus FastAPI, Starlette, Pydantic and Uvicorn), so the API runs with the same numerics as training.

**Running from VS Code:** the Run button cannot pass arguments. Scripts that need arguments
(`register_champion.py`) must be run from the terminal.

---

## 7. Docker and Docker Desktop

### 7.1 What is the difference?

| | **Docker (Docker Engine)** | **Docker Desktop** |
|---|---|---|
| What it is | The technology itself: a background service (the daemon, `dockerd`) that builds images and runs containers, plus the `docker` command and the `docker compose` plugin. | A desktop application that packages Docker Engine, the CLI, Compose and a graphical dashboard into one installer for Windows and macOS. |
| Where it runs | Natively on Linux. | On Windows and macOS, inside a small hidden Linux virtual machine (on Windows, WSL 2). |
| Why it exists | Containers use the Linux kernel. | Windows and macOS do not have a Linux kernel, so Docker Desktop supplies one in a VM and wires it to your terminal and file system. |
| Interface | Command line. | Command line plus a GUI (list of containers, logs, volumes, settings, resource limits). |
| Cost | Free and open source. | Free for personal use, education and small businesses; larger companies need a paid subscription (check Docker's current terms). |

Short version: **Docker is the engine; Docker Desktop is the easiest way to get that engine on a Windows or Mac
laptop.** The commands you type (`docker compose up`) are identical. On this project you use Docker Desktop
because you are on Windows. On a Linux server you would install Docker Engine and use the same compose file.

### 7.2 Why does this project use Docker at all?

Training runs in the virtual environment. Docker is used for **serving and monitoring**:

1. **Reproducible runtime.** The image fixes the OS (Debian slim), Python 3.9 and exact package versions
   (`requirements-serve.txt`). "It works on my machine" becomes "it works in the image".
2. **Deployment realism.** A model API is shipped as a container. This is what a cloud or Kubernetes deployment
   would run, so the project demonstrates the real deployment shape.
3. **One command for a multi-service system.** `docker compose up` starts the API, Prometheus and Grafana, wired
   together on a private network where they find each other by service name (`readmission-api:8000`).
4. **Isolation.** The API runs as a non-root user (`appuser`) in its own filesystem and only sees the folders we
   mount into it.
5. **Health checks and restarts.** The container is probed at `/health`, and `restart: unless-stopped` brings
   services back after a reboot.
6. **Swappable model without rebuilding.** The model is not baked into the image. It is a read-only mounted folder
   (`serving/bundle`), so a promotion only needs a new export and `POST /reload`.

### 7.3 Docker words, using this project's files

| Term | Meaning here |
|---|---|
| **Dockerfile** (`serving/Dockerfile`) | The recipe for the API image: start from `python:3.9-slim`, install `libgomp1` (needed by LightGBM), `pip install` the pinned requirements, copy `app.py` and `ui/`, create a non-root user, define the health check and the start command. The requirements are copied and installed **before** the code, so Docker's layer cache skips the slow install when only code changes. |
| **Image** | The built result of the Dockerfile, tagged `readmission-api:local`. |
| **Container** | A running instance of an image (`readmission-api`). |
| **docker-compose.yml** (project root) | Describes three services and how they connect (ports, volumes, environment, start order). |
| **Volume mount** | A host folder made visible inside a container. `./serving/bundle:/app/bundle:ro` gives the API the model read-only; `./serving/logs:/app/logs` lets it write the prediction log back to your disk. |
| **Named volume** | Docker-managed storage that survives restarts: `prometheus_data` (metric history) and `grafana_data` (dashboards). |
| **`.dockerignore`** | Keeps `bundle/`, `logs/`, `export_champion.py` and `send_sample.py` out of the image (they run on the host or are mounted). |

### 7.4 Why a "bundle" instead of reading MLflow inside the container?

MLflow stores absolute Windows paths (`C:\Users\...\mlruns`) in `mlflow.db`, which do not exist inside a Linux
container, and SQLite does not behave well on mounted Windows folders. So `export_champion.py` runs on the host,
loads `models:/readmission@champion`, and writes a self-contained `model.joblib` plus `meta.json` (with a SHA-256
checksum). The container only needs joblib and the pinned packages.

### 7.5 Everyday commands

```bash
docker compose up -d --build     # build the image if needed, start all services in the background
docker compose ps                # what is running, and is it healthy?
docker compose logs -f readmission-api    # follow the API's log
docker compose restart readmission-api
docker compose down              # stop and remove containers (named volumes, and so metric history, are kept)
docker compose down -v           # ALSO delete the named volumes (Prometheus history, Grafana settings)
```

If `docker` says it cannot connect to the daemon, Docker Desktop is not running: start it and wait for the whale
icon to say "running".

---

## 8. The pipeline, step by step

Each step below shows what it does, what it reads, what it creates, and how that feeds the next step.

**Where the project plan's 15 steps live:**

| Plan step | What | Where it lives | Status |
|---|---|---|---|
| 1 to 5 (P1) | Look at the data, clean, build the label, leak review, split | `code/` scripts, "Step 0" below | Done |
| 6 (P1) | Train models | `train.py` | Done |
| 7 (P1) | Track runs | MLflow with a SQLite database (`mlflow.db`) | Done |
| 8 (P2) | Evaluation gate | `test.py` | Done (calibration is reported, not yet a pass/fail rule) |
| 9 (P2) | Register, alias `champion` | `register_champion.py` (plus `promote.py`, added beyond the plan) | Done |
| 10 (P2) | Serve (FastAPI + Docker) | `serving/`, `docker-compose.yml` | Done |
| 12 (P4) | Monitoring (Prometheus + Grafana) | `monitoring/`, `docker-compose.yml`, metrics in `serving/app.py` | Built (starter dashboard, 5 alert rules, no notifications) |
| 11 (P4) | Holdout replay, drift injection, rollback drill | `serving/replay.py`, `serving/rollback_drill.py`, `monitoring/prometheus/alerts_replay.yml` | Built |
| 13 (P4) | Data and score drift report | `monitoring/drift_report.py` | Built |
| 14, 15 (P4) | Late labels, retrain | not built | Later |

`promote.py` is not one of the 15 plan steps. It adds the champion/challenger rule and rollback, and it is the
manual version of step 15 (retrain, gate, promote).

### Step 0. Data preparation and split (`code/`)

1. **`data_cleaning.py`:** drops patients who died or went to hospice (they cannot be readmitted) and 3 rows with
   invalid gender; groups the "unknown" ID codes; fills missing values with explicit categories; builds the label
   (`<30` = 1, `>30` and `NO` = 0, keeping the original as `readmitted_raw`); adds `n_drugs_taken`,
   `n_drugs_changed` and `diag_*_group`; drops `weight` (97% missing) and constant columns. Result: 99,340 rows,
   69,987 patients, 11.39% positive.
2. **`leak_review.py`:** a hand-written table decides which columns are allowed ("known at discharge?"); the
   statistics (single-column AUC, identifier AUC, out-of-fold categorical AUC, a split-leak demo and a
   whole-model sanity check) only flag suspects. Result: no leak flagged (whole-model AUC 0.669).
3. **`data_split.py`:** `StratifiedGroupKFold` with 20 folds grouped by `patient_nbr`, folds assigned 12 / 3 / 5 to
   train (60%) / test (15%) / holdout (25%). Asserts zero patient overlap.

Full explanations: `code/data_cleaning.md` and `code/leak_review.md`.

What the rest of the pipeline needs from them: `data/splits/train.csv` and `test.csv` (and `holdout.csv`),
`split_report.txt` and `cleaning_config.json` (which `train.py` attaches to each MLflow run), and these columns:
the label `readmitted`, `patient_nbr`, `encounter_id`, `readmitted_raw`, the raw `diag_1` to `diag_3` (not used as
features), 10 numeric features, and categorical features. The fairness checks need `gender`, `race` and `age`.

Sizes: train has 59,753 stays from 42,054 patients (11.49% positive); test has 14,907 stays from 10,483 patients
(11.55% positive).

---

### Step 1. `train.py`: train and compare models

**What it does.** Trains three models and measures each one fairly:

| Model | What it is | How it handles the rarity of readmissions |
|---|---|---|
| Logistic regression | Simple weighted-evidence model. The **baseline**. | `class_weight="balanced"` |
| XGBoost | Many small decision trees, each fixing the last one's mistakes. | `scale_pos_weight` = 7.704 |
| LightGBM | Another tree-boosting model, built differently. | `scale_pos_weight` = 7.704 |

(7.704 is the number of non-readmitted stays divided by the number of readmitted stays in train.)

All models share one preprocessing recipe: skewed visit counts are log-transformed (`log1p`) then standardised, the
other numbers are standardised, categories are one-hot encoded, and categories with fewer than 50 rows (or unseen
later) go into one "infrequent" bucket. Preprocessing and model live in one scikit-learn `Pipeline`, so the exact
same steps run in training, in the gate and in the API.

Each model is scored with 5-fold cross-validation that is stratified and grouped by patient (seed 42). Then it is
refitted on all training data and saved.

```bash
python train.py                     # all three models (always do this)
python train.py --cv-folds 3        # quicker check
```

Never run `--models xgboost` on its own: the ranked joblib files are rebuilt from only the models in that run, so
the others would be deleted.

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

**What to check.** CV PR-AUC should be clearly above 0.115. Current values are about 0.21 to 0.23. Sensible
models lean on things like prior inpatient visits rather than tiny rare categories.

---

### Step 2. `test.py`: the evaluation gate

**What it does.** Scores every saved model, unchanged, on the test split, and checks each against rules that were
written down before the first test result was seen. It does not retrain or tune anything.

```bash
python test.py                      # gate every saved model
python test.py --no-mlflow          # same, without logging runs to MLflow
python test.py --reference xgboost  # change which model the others are compared against
```

Without `--no-mlflow`, each run adds three `eval-...` runs to MLflow. They are duplicates if the numbers did not
change.

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

**The reference model.** For the paired comparison and the fairness table, `test.py` needs one model to compare
the others against. It uses `--reference` if given, otherwise the current champion from `champion.json`, otherwise
the model with the highest test PR-AUC. This is only a comparison anchor. **The gate never chooses the champion;
a person does.**

**Creates:** `models/evaluation.json` (read by step 3), `models/test_report.md`, `data/test_report.txt`, and MLflow
`eval-<model>` runs.

**Evaluating on other data (the holdout or new data):**

```bash
python test.py --data data/splits/holdout.csv --no-mlflow
python test.py --data data/new/new_data.csv --name new_data --no-mlflow
```

- Results go to separate files named after the data (`models/evaluation_<name>.json`, `models/<name>_report.md`,
  `data/<name>_report.txt`), so `models/evaluation.json` is never overwritten. The name `test` is reserved.
- Every use of any dataset is appended to `models/data_use_log.jsonl`, and a repeat use prints a warning.
- Treat the holdout as single-use: look at it once for a decision made in advance, and do not iterate on it.

---

### Step 3. `register_champion.py`: put models in the registry

**Champion mode (first-time setup, a human decision):**

```bash
python register_champion.py --champion lightgbm --reason "Passed the gate; chosen for credible feature reasoning"
```

- Refuses a model that did not pass the gate.
- Finds each model's training run in MLflow, registers one version per model, and sets aliases: `champion` (your
  choice), `baseline` (logistic, always) and `challenger` (other models that passed).
- Tags every version with its evidence: gate result, CV and test scores, the training run id, and your reason.
- Loads the registered champion and checks that its predictions match the saved joblib exactly.
- Writes `champion.json` and `champion_report.md`. Safe to re-run: runs already registered are reused.

**Challenger mode (every later model):**

```bash
python register_champion.py --challenger xgboost --reason "tried MIN_CATEGORY_ROWS=200"
```

- Registers only that model, as a new version with the alias `challenger`. The champion and baseline are untouched.
- Refuses if the model failed the gate, or if it is the exact run that is already champion.
- The previous challenger is tagged `retired_challenger`, not deleted. A `challenger_registered` line is logged.

**Safety guard.** After `promote.py` has changed the champion, running champion mode again would silently reset the
labels to the old model, so it refuses unless you add `--force`.

---

### Step 4. `promote.py`: challenger versus champion

```bash
python promote.py --dry-run             # show the decision, change nothing
python promote.py                       # decide and, if it wins, apply
python promote.py --candidate-version 4 # test a specific registry version
python promote.py --rollback            # put the previous champion back
```

**The rule.** A challenger replaces the champion only if both are true:

1. It passes every gate check (the same thresholds as `test.py`, imported from it).
2. Its PR-AUC beats the champion's, and the paired 95% range of the difference is **entirely above zero**. A tie is
   not a reason to switch.

**When the challenger wins:** it gets `@champion`; the old champion becomes `@previous_champion`; `@challenger` is
removed; version tags, `champion.json` and `champion_report.md` are rewritten; a line is added to
`promotion_log.jsonl`. **When it does not win:** only a log line and a new `promotion_report.md`.
**Rollback** swaps `@previous_champion` back to `@champion` and updates the files.

**Caution.** By default it compares on the test split, which has already been used to choose between models. Each
extra comparison makes results slightly optimistic. Use fresh data (`--data`) or the holdout for the final check.

---

### Step 5. `serving/`: the API (plan step 10)

**What it does.** Serves the champion over HTTP. A client sends the cleaned feature columns of one or more stays;
the API returns a risk score, a high-risk flag and the top reasons (SHAP drivers) for each.

| Endpoint | What it does |
|---|---|
| `GET /` | The web console. |
| `POST /predict` | Score, `high_risk` flag and top SHAP drivers for 1 to 500 records. |
| `GET /example` | A ready-to-send request body. |
| `GET /model-info` | Which model is live, its evidence, the required columns, the allowed category values, the flag rule. |
| `GET /health` | 200 if a model is loaded, otherwise 503 (Docker and the console use this). |
| `GET /metrics` | Prometheus metrics: latency, errors, volume, score distribution, model version, unseen categories. |
| `POST /reload` | Re-read the bundle after a new export. A bad bundle is refused and the old model keeps serving. Needs header `X-Admin-Token` if `ADMIN_TOKEN` is set. |

**The web console** (`serving/ui/`, served by the API itself, no extra server, no external fonts or libraries) has
four views:

| View | What it does |
|---|---|
| Score a stay | A form built from `/model-info` with dropdowns of the values seen in training. The result shows the score on a meter with the cutoff marked, a High risk / Not flagged verdict, and the main drivers as bars. |
| Score a file | Upload a CSV of cleaned stays. Outcome columns are removed, requests are sent in chunks of 500, bad cells are reported with their file row. Shows summary cards, a histogram, the top 100 scores, a search box, click-through to a stay's explanation, and CSV download. |
| Model | Live version, registry evidence, the high-risk rule, required columns, package-mismatch warnings and a Reload button. |
| Monitoring | Parses `/metrics` in the browser and shows stays scored, share flagged, response times, rejected requests, unseen categories and the score distribution. Refreshes every 5 s. |

**Rules the API enforces.** Every feature column must be present with no nulls; numeric columns must hold numbers;
categories are matched the way the model saw them in training (so `1`, `1.0` and `"1"` mean the same); a category
never seen in training is still scored (it falls into the "infrequent" bucket) and is reported in the response and
in `/metrics`. Extra columns are ignored. An `encounter_id` in a record is returned as `id`.

**The score is a ranking score, not a probability.** The high-risk flag means "score at or above the cutoff",
where the cutoff is the score that the top 20% of reference (test) stays reach (scores only, no labels).
Drivers are in log-odds units: positive raises the predicted risk.

**Prediction log.** Every scored record is appended to `serving/logs/predictions.jsonl` (time, request id, model
version, id, score, flag, replay headers and the features). Steps 13 and 14 (drift, late labels) will read it. It
holds patient-level data, so treat it like the data it came from, or set `LOG_PREDICTIONS: "false"` in compose.

**Replay headers (optional).** `X-Replay-Run` and `X-Replay-Week` let a future replay tag its traffic; they are
logged and the week is exposed as the gauge `readmission_replay_week`.

**Health checks.** `/health` says the service is up and has a model; it does not say the predictions are good. For
that, watch flagged share (near 20%), rejected requests, response time and unseen categories, and later drift and
late-label performance.

---

### Step 6. `serving/replay.py`: the holdout replay (plan step 11)

**What it is.** A rehearsal of the live life of the service. The sealed holdout (25% of the patients, never used for
training, model selection or the gate) is cut into simulated weeks and sent, week by week, to the running API as if
hospitals were submitting discharges. From chosen weeks onward the data is deliberately damaged, one kind of damage at
a time, and we check whether the monitoring notices and whether a bad model can be rolled back under traffic.

**Why.** The test split says how the model behaved on a static file. It cannot say what the live service looks like to
the people running it: is the flagged share stable, do bad requests and unfamiliar codes show up, do alerts fire, and can
a worse model be put live and taken out again without dropping requests.

**Official runs used in this README:** `demo1` (12 weeks, four drift tests) and `drill3` (8 weeks, rollback drill under traffic).

```bash
python serving/replay.py --preset demo --dry-run             # print the 12-week plan, send nothing
python serving/replay.py --preset demo --show-examples       # print REAL before/after rows for each test, send nothing
python serving/replay.py --preset demo --run-id demo1 --seconds-per-week 120      # about 24 minutes

# rollback drill under traffic: window 1, then window 2 at the start of week 3 and again at the start of week 7
python serving/replay.py --run-id drill3 --weeks 8 --seconds-per-week 90
python serving/rollback_drill.py --start          # window 2, as week 3 begins
python serving/rollback_drill.py --finish         # window 2, as week 7 begins

python serving/replay.py --run-id demo1 --start-week 7       # resume a stopped run
```

Other options: `--inject NAME[:FIRST[-LAST]][:SEVERITY]` (repeatable, to build your own timeline), `--batch-size`,
`--explain` (adds SHAP drivers, much slower), `--seed`, `--out-dir`.

#### 6.1 Words used in this step

| Term | Meaning |
|---|---|
| **Replay** | Sending already-known data through the live API in a controlled order, to rehearse production. |
| **Holdout** | `data/splits/holdout.csv`: 24,680 stays from 17,450 patients that nothing else has used. |
| **Simulated week** | A slice of the holdout. The data has no dates, so weeks are made by shuffling whole patients (see 6.2). |
| **Clean week** | A week sent unchanged. Clean weeks are the baseline everything else is compared with. |
| **Drift** | The live data (or the model) behaving differently from what the model was built and tested on. |
| **Covariate shift** | The inputs change (for example patients arrive with more prior inpatient stays) while the task is the same. |
| **Population shift** | The mix of patients changes (for example far fewer young patients). |
| **Unseen category** | A code or label value the model never saw in training (for example a new discharge code). The API still scores it, in the "infrequent" bucket, and counts it. |
| **Bad data** | A malformed request: a null, text in a numeric field, or a missing column. The API rejects it with HTTP 422. |
| **High-risk cutoff** | The score the top 20% of reference stays reach (0.5632 for LightGBM v2). A stay at or above it is flagged. |
| **Alert band** | 12% to 28% flagged share. Outside it for long enough, the flagged-share alerts fire. |
| **Label-free monitoring** | Watching inputs, scores and errors without knowing the true outcome. It can show that something moved, not that predictions got worse. |
| **Late labels** | The true outcomes, which in real life arrive 30 days after discharge. The replay withholds them from the API and saves them in `labels.csv` for step 14. |
| **Rollback drill** | Deliberately putting a worse model live, then recovering with the real rollback command. |
| **Run id** | A name for one replay (`demo1`). It is sent to the API as `X-Replay-Run`, logged, and names the output folder. |

#### 6.2 How the holdout became weeks

1. List the unique patients and shuffle them with a fixed seed (42).
2. Walk down the shuffled list, giving each patient's stays to the next week, so each week gets about the same number of stays.
3. A patient therefore never appears in two weeks, and the same seed always gives the same weeks.

The 12-week plan of `demo1` (real output of `--dry-run`):

| Week | Stays | Patients | Test applied |
|---|---|---|---|
| 1 to 4 | 2,057 / 2,057 / 2,057 / 2,056 | 1,487 / 1,451 / 1,423 / 1,372 | none (clean baseline) |
| 5, 6 | 2,057 each | 1,518 / 1,429 | covariate shift, severity 0.50 |
| 7 | 2,057 | 1,439 | none |
| 8, 9 | 2,056 each | 1,477 / 1,465 | new categories, severity 0.05 |
| 10 | 2,057 | 1,416 | none |
| 11 | 2,063 | 1,492 | bad data, severity 0.25 |
| 12 | 2,050 | 1,481 | population shift, severity 0.60 |

"Week 1" is only a random slice. A run with a different number of weeks (the 8 weeks of `drill3`, about 3,085 stays each)
groups patients differently. The holdout file is never edited: each week is copied in memory, damaged there if a test is
active, and sent.

#### 6.3 What the data looks like at each stage

**(a) Two real rows of `holdout.csv`** (53 columns; a selection shown):

| Column | Row 0 | Row 1 |
|---|---|---|
| `encounter_id` | 2278392 | 63768 |
| `patient_nbr` | 8222157 | 114882984 |
| `race`, `gender`, `age` | Caucasian, Female, [0-10) | Caucasian, Male, [70-80) |
| `admission_type_id`, `discharge_disposition_id` | unknown, unknown | 1, 1 |
| `time_in_hospital` | 1 | 5 |
| `medical_specialty` | Pediatrics-Endocrinology | missing |
| `num_lab_procedures`, `num_medications` | 41, 1 | 73, 12 |
| `number_outpatient`, `number_emergency`, `number_inpatient` | 0, 0, 0 | 0, 0, 0 |
| `diag_1`, `diag_2`, `diag_3` (raw ICD-9) | 250.83, missing, missing | 428, 492, 250 |
| `diag_1_group` ... `diag_3_group` | diabetes, missing, missing | circulatory, respiratory, diabetes |
| drug columns | all `No` | `glyburide` = Steady, others `No` |
| `n_drugs_taken`, `n_drugs_changed` | 0, 0 | 1, 0 |
| **`readmitted`**, `readmitted_raw` | **0**, NO | **0**, >30 |

**(b) What is sent to the API.** The replay asks `/model-info` which columns the live model needs (46) and sends only those
plus `encounter_id`. Removed: `patient_nbr`, the raw `diag_1` to `diag_3`, `readmitted` and `readmitted_raw` (53 - 7 = 46,
plus the id). Row 0 becomes (shortened):

```json
{"encounter_id": "2278392", "race": "Caucasian", "gender": "Female", "age": "[0-10)",
 "admission_type_id": "unknown", "discharge_disposition_id": "unknown", "time_in_hospital": 1,
 "medical_specialty": "Pediatrics-Endocrinology", "num_lab_procedures": 41, "num_medications": 1,
 "number_inpatient": 0, "diag_1_group": "diabetes", "n_drugs_taken": 0, "...": "..."}
```

Requests go in batches of 100 stays with headers `X-Replay-Run: demo1` and `X-Replay-Week: 5`.

**(c) What the API logs** (`serving/logs/predictions.jsonl`, one line per scored stay; a real line from the console test):

```json
{"ts": "2026-10-26T11:29:36+00:00", "request_id": "2c4b24ed9add", "model_version": 2, "run_id": null, "replay_week": null,
 "id": "16680", "score": 0.62967, "high_risk": true,
 "features": {"race": "Caucasian", "gender": "Male", "age": "[40-50)", "time_in_hospital": 1, "number_inpatient": 0, "...": "..."}}
```

`high_risk` is true because 0.62967 is at or above the cutoff 0.5632. That line came from ordinary traffic, so `run_id` and
`replay_week` are null. Replay rows have them filled (`"drill3"`, `3`), and `model_version` is what shows which model scored
the stay. The true outcome is not in the log.

**(d) What is saved for later** (`data/replay/<run-id>/labels.csv`; real rows):

```
encounter_id,patient_nbr,readmitted,week,scenario
2790720,48573,1,1,none
1119858,68661747,0,1,none
```

Step 14 joins these outcomes to the logged scores on `encounter_id = id`. The `scenario` column marks which weeks were
altered, so performance can be computed on clean weeks only (in damaged weeks the features were changed but the outcomes
were not, so those weeks are not valid for accuracy).

#### 6.4 The tests we ran

Each test below is applied to the in-memory copy of a week. The "real sample" rows come from `--show-examples` on this
holdout.

**Test 0. Clean baseline (weeks 1 to 4, 7, 10)**

- *Definition:* unchanged weeks, to learn what normal looks like.
- *Observed:* flagged share 18.7%, 19.6%, 20.8%, 24.0%, 20.2%, 21.0%, mean 20.7%; mean score about 0.42 to 0.44; unseen
  values 0 to 2 per week; no rejections. The cutoff is set to flag about 20%, so the live service matches testing, and
  week-to-week noise is about plus or minus 3 points, well inside the 12% to 28% band.

**Test 1. Covariate shift (weeks 5 and 6): "patients look sicker"**

- *What we changed:* for a random half of the stays, `number_inpatient` + 3 and `time_in_hospital` + 3 (capped at 14).
  1,035 of 2,057 stays changed in week 5.
- *Real sample:*

  | encounter | time_in_hospital | number_inpatient |
  |---|---|---|
  | 792402 | 2 -> 5 | 0 -> 3 |
  | 2524134 | 6 -> 9 | 0 -> 3 |
  | 2730936 | 1 -> 4 | 2 -> 5 |

- *Why these columns:* prior inpatient stays is the strongest feature in LightGBM, so shifting it moves the scores clearly.
- *Expected:* flagged share and mean score rise.
- *Observed:* flagged share **48.5% and 49.7%** (about 2.4 times normal), mean score 0.524 and 0.536 (normally about 0.43).
  `ReplayFlaggedShareOutOfRange` fired. **Detected.**

**Test 2. New categories (weeks 8 and 9): "unfamiliar codes appear"**

- *What we changed:* 5% of stays had `discharge_disposition_id` and `medical_specialty` replaced by a value never seen in training
  (`REPLAY_NEW_VALUE`). In week 8: 108 stays in the first column, 96 in the second.
- *Real sample:*

  | encounter | change |
  |---|---|
  | 4322028 | medical_specialty: Psychiatry -> REPLAY_NEW_VALUE |
  | 8397792 | medical_specialty: Pediatrics-Endocrinology -> REPLAY_NEW_VALUE |
  | 10126866 | discharge_disposition_id: 3 -> REPLAY_NEW_VALUE |

- *Expected:* the stay is still scored (the encoder files it under "infrequent"), and the unseen-category counter rises.
- *Observed:* the API's counter rose by **204 and 198** (108 + 96 = 204 matches exactly). Scores barely moved (flagged
  23.2% and 23.0%). `ReplayUnseenCategories` and the slow `UnseenCategoriesRising` both fired. **Detected.**

**Test 3. Bad data (week 11): "an upstream system sends junk"**

- *What we changed:* a quarter of the requests (batches) contained one broken record. Three kinds rotate. After the rejection the
  replay resends the clean batch (an upstream bug fixed by a retry).
- *Real sample:*

  | Kind | Change | API answer |
  |---|---|---|
  | null | record 1, `number_outpatient`: 0 -> null | 422 `nulls` |
  | not numeric | record 1, `number_emergency`: 0 -> "abc" | 422 `not_numeric` |
  | missing column | `diag_3_group` removed from every record in the batch | 422 `missing_columns` |

- *Expected:* rejections, an error-share alert, and no bad record ever scored.
- *Observed:* **7 of 28 requests rejected** (25%, as injected), all 2,063 stays still scored after the clean resend.
  `ReplayErrorShare` fired. **Detected.**

**Test 4. Population shift (week 12): "the patient mix changes"**

- *What we changed:* 60% of the stays of patients under 60 were dropped, so the week is older and smaller (2,050 -> 1,676 stays,
  374 removed).
- *Real result (age mix before and after):*

  | Age | Before | After | Share before | Share after |
  |---|---|---|---|---|
  | [10-20) | 16 | 4 | 0.8% | 0.2% |
  | [20-30) | 21 | 10 | 1.0% | 0.6% |
  | [30-40) | 66 | 30 | 3.2% | 1.8% |
  | [40-50) | 187 | 86 | 9.1% | 5.1% |
  | [50-60) | 345 | 131 | 16.8% | 7.8% |
  | [60-70) | 457 | 457 | 22.3% | 27.3% |
  | [70-80) | 557 | 557 | 27.2% | 33.2% |
  | [80-90) | 347 | 347 | 16.9% | 20.7% |
  | [90-100) | 54 | 54 | 2.6% | 3.2% |

- *Expected:* flagged share and scores might move.
- *Observed:* flagged share **19.1%**, mean score 0.430, both normal. No alert fired. **Not detected.** Under-60s fell from
  31.0% to 15.6% of the week, and nothing in score-level monitoring showed it, because the model's scores did not move.

**Test 5. Rollback drill under traffic (`drill3`)**

- *Why a drill:* the gate and `promote.py` exist so that a bad model can never be promoted by the normal route, so recovering
  from one has to be rehearsed by putting it in deliberately. `rollback_drill.py --start` makes the logistic baseline
  (a weaker model that failed the gate) the champion, writes `drill_swap` to the audit log, exports the bundle and reloads
  the API. `--finish` runs the real `promote.py --rollback`, restores tags, exports and reloads.
- *What we did:* 8 clean weeks, `--start` as week 3 began, `--finish` as week 7 began.
- *Observed (from `predictions.jsonl`, grouped by week and model version):*

  | Week | Model version | Stays scored | Mean score | Flagged share |
  |---|---|---|---|---|
  | 1 | 2 (LightGBM) | 3,085 | 0.422 | 19.5% |
  | 2 | 2 | 3,086 | 0.428 | 19.9% |
  | 3 | **3 (logistic)** | 3,084 | 0.467 | 20.8% |
  | 4 | 3 | 3,086 | 0.471 | 21.7% |
  | 5 | 3 | 3,084 | 0.466 | 19.6% |
  | 6 | 3 | 3,085 | 0.461 | 19.6% |
  | 7 | 2 | 3,085 | 0.432 | 20.1% |
  | 8 | 2 | 3,085 | 0.422 | 19.3% |

- *What it proved:* the swap and the rollback both happened mid-run with **no failed requests** (every week scored every stay,
  0 rejected); the bad model scored about 12,300 live stays; and the model-version panel and `ReplayModelChanged` alert
  reacted.
- *The lesson:* the flagged share stayed at about 20% (20.8, 21.7, 19.6, 19.6) while the weaker model was live, because the
  export recomputes the cutoff (0.5867 for logistic) for whatever model it exports. The only label-free hint was a small
  rise in mean score (about 0.466 against 0.426, matching the models' mean predicted risk on the test split, 0.466 and 0.428).
  Nothing in score-level monitoring says "worse". Only labels can, which is step 14.

#### 6.5 Results of `demo1` (real `summary.md`)

Clean-week flagged share 20.7%; alert band 12% to 28%.

| Week | Injected | Stays scored | Requests | Rejected (422) | Flagged | Mean score | Unseen values (API counter change) |
|---|---|---|---|---|---|---|---|
| 1 | none | 2,057 | 21 | 0 | 18.7% | 0.419 | 0 |
| 2 | none | 2,057 | 21 | 0 | 19.6% | 0.428 | 0 |
| 3 | none | 2,057 | 21 | 0 | 20.8% | 0.427 | 2 |
| 4 | none | 2,056 | 21 | 0 | 24.0% | 0.439 | 0 |
| 5 | covariate_shift 0.50 | 2,057 | 21 | 0 | 48.5% (outside band) | 0.524 | 0 |
| 6 | covariate_shift 0.50 | 2,057 | 21 | 0 | 49.7% (outside band) | 0.536 | 1 |
| 7 | none | 2,057 | 21 | 0 | 20.2% | 0.429 | 1 |
| 8 | new_categories 0.05 | 2,056 | 21 | 0 | 23.2% | 0.434 | 204 |
| 9 | new_categories 0.05 | 2,056 | 21 | 0 | 23.0% | 0.436 | 198 |
| 10 | none | 2,057 | 21 | 0 | 21.0% | 0.436 | 1 |
| 11 | bad_data 0.25 | 2,063 | 28 | 7 | 19.7% | 0.426 | 1 |
| 12 | population_shift 0.60 | 1,676 | 17 | 0 | 19.1% | 0.430 | 3 |

**Alerts that fired:** ReplayFlaggedShareOutOfRange, ReplayUnseenCategories, UnseenCategoriesRising, ReplayErrorShare,
ReplayModelChanged (the drill). The slow real-traffic rules for flagged share, error share and latency stayed quiet, which is
correct: their 5- to 15-minute windows are too long for a 2-minute scenario. They are built for real traffic, and the replay
rules are the fast versions.

#### 6.6 What we found

1. **The live service matches testing on clean data.** Flagged share 20.7% on average against a design value of 20%.
2. **Three of the four damage types were caught**, each by the signal you would expect: scores and flagged share
   (covariate shift), the unseen-value counter (new categories), the rejection counter (bad data).
3. **One was missed: population shift.** A large change in who the patients are (under-60s halved) did not move scores or flagged
   share. The gap is real, and it is fixed by checking input features directly each week (step 13), for example the age mix.
4. **A bad model can be swapped in and out under traffic with zero failed requests,** thanks to the atomic reload and the
   checksummed bundle.
5. **Label-free monitoring cannot see a worse model.** The flagged share stayed healthy throughout the drill. Detecting it needs real
   outcomes (step 14). The labels are already saved in `labels.csv`.
6. **Replay alerts need windows that match the replay speed.** At 90 to 120 seconds per week, 3-minute windows work; the hour-scale
   rules would not fire in a short demo.

#### 6.7 Files created by this step

| File | Significance |
|---|---|
| `data/replay/<run-id>/plan.json` | The recipe: arguments plus each week's size, patients, positive rate and tests. With seed 42 the run can be repeated exactly. |
| `data/replay/<run-id>/labels.csv` | The withheld true outcomes for every stay actually scored, with week and scenario. Input for step 14. |
| `data/replay/<run-id>/summary.json` | Per-week results in machine-readable form (reject reasons, request times, unseen counts). Input for step 13 and automation. |
| `data/replay/<run-id>/summary.md` | The human-readable table shown in 6.5, plus what to look at. |
| `serving/logs/predictions.jsonl` | Every scored stay with `run_id`, `replay_week`, `model_version`, score, flag and features. Filter by `run_id`. It holds patient-level rows: never commit it. |
| `models/data_use_log.jsonl` | One line per replay: who looked at the holdout and when. |
| `models/promotion_log.jsonl` | `drill_swap` and rollback entries (the drill's audit trail). |
| `models/champion.json`, `champion_report.md` | Rewritten by the drill and again by the rollback; they describe the model currently registered. |
| `serving/bundle/` | Overwritten by each export; what the API serves. |
| Prometheus and Grafana data | Stored in Docker volumes: the history behind the alerts and charts. |

The code behind it: `serving/replay.py`, `serving/rollback_drill.py`, `monitoring/prometheus/alerts_replay.yml` and the
`readmission_replay_week` gauge and replay headers already in `serving/app.py`.

**Alert rules for the replay.** `monitoring/prometheus/alerts_replay.yml` has four rules with 3-minute windows
(ReplayFlaggedShareOutOfRange, ReplayUnseenCategories, ReplayErrorShare, ReplayModelChanged). They are loaded by
`prometheus.yml`; after editing run `docker compose restart prometheus`. Use 90 to 120 s per simulated week.

#### 6.8 Limits

- Weeks are simulated by shuffling whole patients; the holdout has no dates, so time effects are untested.
- Damaged weeks keep the original outcomes, so they are not valid for measuring accuracy. Use clean weeks for step 14.
- The holdout is single-use for decisions. Replaying it with different scenarios is fine because nothing is tuned on it, but
  every run is logged in `data_use_log.jsonl`.
- Thresholds (12% to 28%, 25 unseen values, 5% errors) were carried over from the production rules, not tuned on replay data.
- The logistic baseline differs from LightGBM by only about 0.016 PR-AUC, so a real label-based comparison in step 14 needs
  enough stays per segment (the drill scored about 12,300) to separate the difference from noise.


---

### Step 7. `monitoring/drift_report.py`: the drift report (plan step 13)

**What it is.** A script that compares what the model **received** each week (every input feature) and what it **output**
(the scores) with a baseline of clean weeks, and says which features drifted and by how much.

**Why.** The replay (step 6) found a gap: when the patient mix changed, no alert fired, because the scores barely moved. Alerts
watch the output. This step watches the inputs, feature by feature, so it can say "age changed" even when the scores look
normal. It is a diagnostic report, not a live alert (a live version is a possible later addition).

**It needs nothing new.** It reads what the API already logged (`serving/logs/predictions.jsonl`: week, model version, score,
flag and the input features of every scored stay). It does not call the API, the registry or any model, and it changes no
existing file.

```bash
python monitoring/drift_report.py --run-id demo1                         # baseline = weeks 1-4
python monitoring/drift_report.py --run-id demo1 --baseline-weeks 1-4 --top 8
python monitoring/drift_report.py --run-id drill3 --baseline-weeks 1-2
```

#### 7.1 Words used in this step

| Term | Meaning |
|---|---|
| **Baseline** | The clean weeks that define "normal" (weeks 1 to 4 of `demo1`; 8,227 stays in a typical run). |
| **PSI (population stability index)** | A measure of how much one distribution moved from another. 0 means identical. |
| **Bins** | For a numeric feature, the baseline's deciles; each stay falls in one bin. Features with 12 or fewer values, and categories, are compared value by value. |
| **`__rare__` and `__unseen__` buckets** | For categories: `__rare__` = seen in the baseline but under 0.5%; `__unseen__` = never seen in the baseline. A brand-new code lands in `__unseen__` on its own. |
| **Stable / moderate / major** | The usual rule of thumb: PSI below 0.10, 0.10 to 0.25, above 0.25. A convention, not a law. |
| **Noise floor** | The PSI of a clean sample of typical week size against the rest of the baseline. Only values clearly above it mean something. |
| **Score PSI** | The same measure applied to the model's scores. It jumps when the model version changes (a different model, different scores); that is not input drift. |

#### 7.2 How PSI works, with a small example

For each bin, compare the share of stays in the week (a) with the share in the baseline (e):

`PSI = sum over bins of (a - e) * ln(a / e)`

Using the real age counts of week 12 in `demo1` (before and after 60% of the under-60 stays were dropped), the age mix goes
from 31.0% to 15.6% under 60, and the PSI of `age` comes to about **0.14**: a moderate drift that no score-based alert saw.
For the drastic young version (80% of the 60+ stays dropped, see 6.4) it comes to about **0.62**: major.

#### 7.3 What it does

1. Loads the logged stays of one run id, drops duplicates (a resumed week is logged twice) and keeps `replay_week`, `model_version`, score and features.
2. Builds bins and category lists from the baseline weeks only.
3. Measures the noise floor: a clean sample of one typical week against the rest of the baseline.
4. For every week and every feature (and the score), computes PSI and a one-line description of the change
   (for example `mean 1.05 -> 2.54` or `__unseen__ 0.0% -> 5.3%`).
5. If `data/replay/<run-id>/plan.json` exists, builds a "did we catch it?" table: each injected test is judged by the feature it
   should move (covariate shift: `number_inpatient` and `time_in_hospital`; new categories: `discharge_disposition_id` and
   `medical_specialty`; population shifts: `age`). Clean weeks are checked for false alarms. `bad_data` is reported as
   not visible here, because rejected requests are never logged (the error-share alert covers it).
6. Writes the files below.

**Creates** `data/drift/<run-id>/`:

| File | Significance |
|---|---|
| `drift_report.md` | The report: weekly summary, the "did we catch it" table, and details of the weeks that drifted. |
| `psi_by_week.csv` | PSI of every feature and of the score, every week. For your own analysis. |
| `drift_summary.json` | The same in machine-readable form (input for automation or a later live exporter). |
| `drift_heatmap.png` | Weeks across, the 25 most drifted features down, green / amber / red. The picture to show. |

#### 7.4 What we expect for `demo1`

| Week | Injected test | Expected in the drift report |
|---|---|---|
| 1 to 4 | none (baseline) | PSI near the noise floor |
| 5, 6 | covariate shift | `number_inpatient` and `time_in_hospital` major; score PSI up |
| 7, 10 | none | nothing major (a false alarm would be a finding) |
| 8, 9 | new categories | `discharge_disposition_id` and `medical_specialty` major, driven by the `__unseen__` bucket |
| 11 | bad data | nothing, which is correct: rejected requests are never logged |
| 12 | population shift | `age` moderate (about 0.14): the gap found in step 6 is closed |

On the synthetic end-to-end test of the script (a mock API and a generated holdout with your real age mix) every row above
came out as listed: the noise floor was 0.005, covariate shift was caught (PSI 1.4 to 1.5), new categories were caught (0.3),
the age shift was caught at 0.16, and the clean weeks stayed quiet. The real numbers come from your own run; run the command
above and add the result here.

The week 11 row is worth remembering: different detectors catch different failures. The error-share alert catches bad data,
the drift report catches changed data, and neither replaces the other.

#### 7.5 What to do when a feature drifts

Drift is a symptom, not a verdict. Retraining is only one possible response:

| What the report says | What it means | What you do |
|---|---|---|
| A feature drifted and the data is wrong (a broken feed, a renamed code) | The model is fine, the input is damaged | Fix the data or the mapping. Do not retrain. |
| Inputs drifted and real performance is unchanged (step 14) | The model still works on the new mix | Keep the model and keep watching. |
| Inputs drifted and real performance dropped | The model is stale | Retrain, then follow the normal path: gate, challenger, promotion rule. A retrained model never goes straight live. |

#### 7.6 Limits

- PSI depends on binning and sample size. With about 2,000 stays per week against a baseline of about 8,000 it is stable, but
  small weeks are noisy (weeks under 300 stays are marked "too small").
- With 46 features, read the top few, not all of them. Tiny categories are pooled into `__rare__` to avoid noise.
- The baseline here is clean replay weeks. In production it would be the training or reference data.
- Drift shows that inputs moved, not that predictions got worse. That needs the late labels (step 14).
- It compares the logged features, so any request rejected before logging is invisible to it.


---

## 9. Monitoring (Prometheus and Grafana)

`docker compose up` also starts:

- **Prometheus** (`prom/prometheus:v2.55.1`, port 9090). Scrapes `readmission-api:8000/metrics` every 15 s and keeps
  30 days of history in the named volume `prometheus_data`.
- **Grafana** (`grafana/grafana:11.3.0`, port 3000). Its Prometheus data source is provisioned automatically from
  `monitoring/grafana/provisioning/`. Log in with admin / admin and change the password, or put
  `GRAFANA_PASSWORD=...` in a `.env` file next to `docker-compose.yml` (do not commit it).
- A starter dashboard: Grafana > Dashboards > New > Import > upload `monitoring/grafana/starter_dashboard.json`.
  It shows API up, live model version, replay week, share flagged, request rate, stays scored per minute, `/predict`
  latency p50/p95, error share, score-distribution heatmap, unseen categories and rejected requests.

**Alert rules** (`monitoring/prometheus/alerts.yml`; visible at http://localhost:9090/alerts; no notifications are
sent anywhere yet):

| Alert | Fires when |
|---|---|
| ApiDown | Prometheus cannot scrape the API for 1 minute |
| FlaggedShareOutOfRange | Share flagged high risk is outside 12% to 28% for 15 minutes (expected about 20%) |
| PredictLatencyHigh | 95th percentile of `/predict` is over 1 s for 5 minutes |
| ErrorShareHigh | More than 2% of requests are non-2xx for 5 minutes |
| UnseenCategoriesRising | More than 25 never-seen category values in an hour |

**Replay alert rules** (`monitoring/prometheus/alerts_replay.yml`, 3-minute windows, for the replay and drills):
ReplayFlaggedShareOutOfRange (outside 12% to 28%, with more than 200 stays in the window), ReplayUnseenCategories
(more than 25 in 3 minutes), ReplayErrorShare (more than 5% non-2xx), ReplayModelChanged (a model was loaded in the
last 10 minutes).

API counters reset when the API restarts, so always chart `rate()` or `increase()`, never raw totals.
`docker compose down -v` deletes the metric history.

---

## 10. How everything affects everything else

### 10.1 The chain

```
code/data_*.py  writes data/splits/*.csv, cleaning_config.json, split_report.txt

train.py
  reads   data/splits/train.csv
  writes  models/*.joblib      ---> read by test.py (and by register_champion for verification)
  writes  MLflow runs          ---> found by register_champion (matched on CV score)

test.py
  writes  evaluation.json      ---> read by register_champion (gate result, scores, file names)
  reads   champion.json        ---> to pick its reference model
  defines gate thresholds      ---> imported by promote.py

register_champion.py
  writes  registry aliases     ---> read by promote.py and serving/export_champion.py
  writes  champion.json        ---> read by test.py and humans

promote.py
  reads   registry, test.py rules, test data
  writes  aliases, champion.json, logs, reports

serving/export_champion.py
  reads   registry @champion + data/splits/test.csv (scores only, for the flag cutoff)
  writes  serving/bundle/      ---> read by the API (app.py)
  appends models/data_use_log.jsonl

serving/replay.py
  reads   data/splits/holdout.csv, the API's /model-info
  sends   weekly batches to /predict with X-Replay-Run / X-Replay-Week
  writes  data/replay/<run-id>/ (plan, labels.csv, summary), models/data_use_log.jsonl

monitoring/drift_report.py
  reads   serving/logs/predictions.jsonl, data/replay/<run-id>/plan.json (optional)
  writes  data/drift/<run-id>/ (report, PSI table, heatmap)

serving/rollback_drill.py
  reads   registry; imports promote.py's functions
  writes  aliases, champion.json, promotion_log.jsonl ("drill_swap"), then export + /reload

serving/app.py (in Docker)
  reads   serving/bundle/
  writes  serving/logs/predictions.jsonl, /metrics  ---> scraped by Prometheus ---> Grafana
```

### 10.2 Which file is the source of truth for what

| Question | Authoritative answer | Not authoritative |
|---|---|---|
| Which model is champion right now? | The registry alias `@champion` (MLflow **Models** tab) | `champion.json` is a copy the scripts keep in sync. `best_model.json` is only the top of the CV ranking. |
| Which model is the API serving? | `GET /model-info` or `serving/bundle/meta.json` | The registry may already have moved on until you export and reload. |
| Which models were trained last? | `models/*.joblib` and the MLflow runs | The registry may still hold older versions. |
| Did a model pass the gate last time? | `evaluation.json` (the last `test.py` run only) | Older reports are overwritten. |
| What decisions were made? | `promotion_log.jsonl` | The reports are overwritten each run. |

### 10.3 If you change X, then rerun Y

| If you change... | Then... |
|---|---|
| The data cleaning or the split | Rerun everything from `train.py`. Old scores are not comparable, so new models enter as challengers. |
| Model settings or features in `train.py` | `train.py` (all three) -> `test.py` -> `register_champion.py --challenger` -> `promote.py`. |
| Gate thresholds in `test.py` | They change both the gate and `promote.py`. Changing them after seeing results defeats the purpose of writing them in advance. |
| The random seed | Every CV split and bootstrap changes, so scores shift slightly. |
| Package versions | Saved models may fail to load or predict differently. Update the requirements files, retrain, re-verify, rebuild the image. |
| The champion (promotion or rollback) | `python serving/export_champion.py`, then `POST /reload`. |
| `serving/app.py` or the Dockerfile | `docker compose up -d --build`. |
| `serving/ui/*` | Just refresh the browser (the folder is mounted). |
| `serving/replay.py` scenarios, or the replay alert rules | Rerun with a new `--run-id`; after editing `alerts_replay.yml` run `docker compose restart prometheus`. |
| `monitoring/prometheus/*.yml` | `docker compose restart prometheus`. |
| `mlflow.db` or `mlruns/` (deleting) | The registry and run history are gone. The joblib files survive, but `register_champion.py` cannot find runs. |
| `models/*.joblib` (deleting) | `test.py` cannot run and verification fails. The registry is unaffected. |
| An alias edited by hand in the MLflow UI | `promote.py` still obeys the registry, but `champion.json` and the reports go stale. |

### 10.4 Hidden couplings worth knowing

1. **`promote.py` imports `test.py` by file path.** They must stay in the same folder. (A file named `test.py`
   clashes with Python's built-in `test` package, so it is loaded with `importlib` by path.)
2. **`test.py` reads `champion.json`** to choose its reference model.
3. **`register_champion.py` finds a model's training run by matching CV scores:** the score in `evaluation.json`
   against each run's recorded score (tolerance 0.0006), taking the latest finished training run of that family.
   If `evaluation.json` is out of date relative to MLflow, no run matches and it stops with a clear message.
4. **The registry holds its own copy of each model** (inside `mlruns/`), separate from the joblib files. Retraining
   replaces the joblibs but does not change what the registry serves.
5. **`train.py` renumbers and replaces the joblib files every run.** File names embed the rank and CV score, and
   `evaluation.json` stores those names, so after a retrain it is stale until `test.py` is run again.
6. **Everything is seeded (42),** so a rerun on the same data and packages reproduces the same numbers.
7. **Test-set reuse.** Every comparison on the test split slightly overfits it. That is why the promotion rule
   demands a clear win, not a tie.
8. **The flag cutoff comes from the test split's scores.** `export_champion.py` uses scores only (no labels), and
   logs that use in `data_use_log.jsonl`, but the cutoff does assume live traffic looks like test data.
9. **Folder name case matters on Linux.** The API, the Dockerfile and compose all use `serving/ui` (lowercase) and
   `serving/Dockerfile`.

---

## 11. What happens in each case

| Case | What changes | What does not change |
|---|---|---|
| A challenger clearly beats the champion | Aliases swap; `champion.json`, `champion_report.md`, the log and `promotion_report.md` are updated. | The joblib files, `evaluation.json`, `test_report.md`, and **the API (until you export and reload)**. |
| A challenger does not clearly win | One log line and a new `promotion_report.md`. | The registry. |
| The new champion turns out to be bad | `promote.py --rollback` restores the previous champion; export and reload to serve it. | The registered versions. |
| You retrain | `train.py` replaces the joblibs; `test.py` replaces `evaluation.json`. | The registry still holds the old versions until a challenger is promoted. |
| A model fails the gate | It cannot be registered as champion or challenger. The baseline is always registered. | Everything else. |
| One model crashes during training | It is marked FAILED; the others finish. | The other models. |
| `train.py --models xgboost` alone | The other ranked joblib files are deleted. | The registry. |
| `register_champion.py --champion` after a promotion | Refused unless `--force`. | Everything. |
| `register_champion.py --challenger` on the current champion's run | Refused: retrain first. | Everything. |
| `/reload` gets a corrupt bundle | HTTP 500 with `still_serving_version`. | The old model keeps serving. |
| The API starts with no bundle | `/health` returns 503, the console shows a banner. | Nothing crashes. |
| Packages change | Saved models may not load or may behave differently. | The registry's records. |

---

## 12. Recipes

### A. First run from scratch

See [section 3](#3-run-it-from-a-fresh-clone).

### B. Try an improved model

1. Change a setting in `train.py` (for example `MIN_CATEGORY_ROWS`, or `min_child_weight` for XGBoost).
2. `python train.py` (all three models together).
3. `python test.py`
4. `python register_champion.py --challenger xgboost --reason "what you changed"`
5. `python promote.py --dry-run`, then `python promote.py` if you agree.
6. If promoted: `python serving/export_champion.py`, then reload the API.

A new model always enters as a challenger, so it can never skip the promotion rule.

### C. Undo a promotion

```bash
python promote.py --rollback
python serving/export_champion.py
curl -X POST http://localhost:8000/reload
```

### D. See who the champion is

- Read `models/champion_report.md`, or
- open the MLflow UI, click **Models**, then **readmission**, and look at the Aliases column, or
- run: `python -c "import mlflow; mlflow.set_tracking_uri('sqlite:///mlflow.db'); c=mlflow.MlflowClient(); print('champion is version', c.get_model_version_by_alias('readmission','champion').version)"`

### E. Check what the API is serving

Open http://localhost:8000/model-info, or the Model tab of the console.

---

## 13. Current results

As of 2026-10-22. All numbers are on the test split (14,907 stays, no-skill PR-AUC 0.116).

| Model | CV PR-AUC | Test PR-AUC | 95% range | ROC-AUC | Lift | Gate | Registry |
|---|---|---|---|---|---|---|---|
| LightGBM | 0.224 | 0.222 | 0.200 to 0.247 | 0.670 | 1.93x | PASS | v2, `@champion` |
| XGBoost | 0.226 | 0.218 | 0.197 to 0.240 | 0.670 | 1.89x | PASS | v1, `@challenger` |
| Logistic | 0.211 | 0.206 | 0.188 to 0.229 | 0.660 | 1.78x | FAIL (fairness) | v3, `@baseline` |

**How the champion was chosen.** LightGBM and XGBoost are statistically tied (paired difference -0.005, range
-0.010 to +0.001). XGBoost ranked first by CV by 0.002, but its SHAP ranking leaned on tiny rare categories (for
example `medical_specialty_Otolaryngology`), while LightGBM relied on sensible signals (prior inpatient visits,
discharge disposition, lab procedures). LightGBM was chosen by a person on that basis, and it also has more room
under the fairness limits (worst AUC gap 0.040 against 0.048 for XGBoost, limit 0.05). `promote.py` confirms that
XGBoost does not clearly beat it.

**Logistic regression fails the gate** on fairness: worst ROC-AUC gap 0.0504 (limit 0.05) and worst recall gap
0.126 (limit 0.10). It is still registered as the baseline.

**Fairness.** The champion's weakest group is patients aged 80 to 90 (ROC-AUC 0.630 against 0.670 overall). Small
groups (some races, the youngest and oldest age bands) are too small to judge.

**Calibration.** Mean predicted risk for LightGBM is 0.428 against an actual rate of 0.116 (ECE 0.313), because of
the class weights. Scores are for ranking only.

---

## 14. Limits and rules to respect

- **Not a clinical tool.** Not validated for use with real patients.
- **Scores are not probabilities.** Class weights inflate them. Use them to rank and flag, not as "a 43% chance",
  until a calibration step is added.
- **The test split is spent for selection.** Do not tune models on it. Use fresh data or the holdout.
- **Never loosen a gate threshold after seeing results.**
- **No dates in the data,** so behaviour over time cannot be tested yet.
- **The high-risk cutoff is derived from test-split scores** and assumes live traffic resembles them.
- **The holdout is used by the replay.** It is logged in `data_use_log.jsonl`; use it for decisions made in advance, not for tuning.
- **Rare categories** inflate some feature rankings (especially for XGBoost and logistic). Increasing
  `MIN_CATEGORY_ROWS` is a candidate experiment.
- **The prediction log holds patient-level rows.** Do not commit it or share it.
- **Default Grafana password is admin / admin.** Change it before exposing the port beyond your machine.

---

## 15. Status and what is not built yet

Built: data preparation, training, gate, registry, promotion and rollback, API and web console in Docker,
Prometheus and Grafana with a starter dashboard and alert rules, the holdout replay with a rollback drill, and the per-week drift report.

Not built yet, in the order planned:

1. **Late-label performance (step 14):** join `data/replay/<run-id>/labels.csv` back by `encounter_id`, week by week,
   with a realistic delay, to compute real PR-AUC and recall in the flagged group.
2. **Automated retrain, gate and promote (step 15);** one-command orchestration and CI (tests, lint, image build).
3. **Live drift alerts:** export PSI to Prometheus (`drift_exporter.py`, with a reference profile saved in the bundle) so drift pages
   someone instead of waiting for a report.
4. **Alert notifications** (email or chat) from Prometheus alerts.
5. **Model improvement:** a calibration step and the experiments in section 14, run as challengers.

---

## 16. Troubleshooting

Errors that have actually come up in this project, plus the Docker ones to expect.

| Message | Meaning and fix |
|---|---|
| `Pipeline should either be a classifier ... Got a regressor with response_method=predict_proba` | scikit-learn is newer than this XGBoost version supports. Use scikit-learn 1.5.2 with xgboost 2.0.3. |
| `UNIQUE constraint failed: metrics...` (MLflow) | A model produced NaN scores and MLflow could not store them. `train.py` raises the real error instead, so read the first error above it. |
| `OSError: [Errno 28] No space left on device` | The disk is full. Free about 5 GB, then install with `--no-cache-dir`. |
| `Failed building wheel for greenlet` / `Microsoft Visual C++ 14.0 or greater is required` | Pip is trying to compile a package. Install with `--prefer-binary`. |
| `ERROR: No matching distribution found for pywin32` (macOS/Linux) | You used `requirements-lock.txt`, which is a Windows snapshot. Use `requirements.txt`. |
| `ModuleNotFoundError: No module named 'joblib'` (or similar) | Wrong or empty environment. Activate `.venv` and install the requirements. |
| `the following arguments are required: --champion, --reason` | Run `register_champion.py` from the terminal and name the model and the reason. |
| `REFUSED: ... did not pass the evaluation gate` | The model failed the gate in `evaluation.json`. Do not register it. |
| `REFUSED: this exact training run is already the champion` | There is no new model to register. Retrain first. |
| `REFUSED: promote.py has changed the champion ...` | Re-running champion mode would reset the labels. Use `--force` only if you mean it. |
| `No finished training run for '<model>' matches CV PR-AUC ...` | `evaluation.json` and the MLflow runs are out of step. Run `train.py`, then `test.py`, then try again. |
| `VERIFY FAILED: ... differs from ...joblib` | The registered model and the saved file do not match. Retrain and re-register. |
| `Missing .../evaluation.json. Run test.py first.` | Run `test.py` before registering. |
| `Missing .../diabetic_data_cleaned.csv` / `Missing .../train.csv` | Run `code/data_cleaning.py`, then `code/data_split.py` (some messages say `split_data.py`; the file is `code/data_split.py`). |
| MLflow UI port already in use | Add `--port 5001` to the `mlflow ui` command. |
| `error during connect` / `Cannot connect to the Docker daemon` | Docker Desktop is not running. Start it and wait until it says running. |
| `port is already allocated` (8000, 9090 or 3000) | Another program uses that port. Stop it or change the left side of the port mapping in `docker-compose.yml`. |
| `failed to read dockerfile` / `unable to prepare context` on Linux | The file must be named `serving/Dockerfile` (capital D). On Linux the name is case-sensitive. |
| Console unstyled, empty, or "Loading the form…" forever; `/ui/...` returns 404 | `serving/ui/` is missing or is named `UI`. It must be lowercase `ui` with `index.html`, `styles.css`, `app.js`. Hard-refresh with Ctrl+F5. |
| The page at `/` shows "No model is loaded yet", or `/health` returns 503 | There is no bundle. Run `python serving/export_champion.py`, then press Reload (or restart the API). |
| `/predict` returns 422 | The message names the column and the problem (missing, null, not numeric). Fix the request. |
| `/reload` returns 500 with `still_serving_version` | The new bundle is incomplete or corrupt. The old model keeps serving. Re-run the export. |
| `/reload` returns 403 | `ADMIN_TOKEN` is set. Send the header `X-Admin-Token`. |
| `package_warnings` in `/model-info` | Packages in the container differ from those the bundle was made with. Rebuild the image. |
| `Could not write the prediction log` in the API log (Linux) | The container user (uid 1000) cannot write `serving/logs`. Run `mkdir -p serving/logs && sudo chown 1000 serving/logs`. Requests are still served. |
| Container shows "unhealthy" | `docker compose logs readmission-api`; usually no bundle or a bad bundle. Docker marks it unhealthy but does not restart it by itself. |
| `replay.py`: `Cannot reach the API` / `The API is not healthy` | Start the stack (`docker compose up -d`) and export a bundle (`python serving/export_champion.py`, then reload). |
| `replay.py`: `... already exists. Choose another --run-id` | Run ids name the output folder and are never overwritten. Pick a new one, or use `--start-week N` with the same id to resume. |
| `replay.py`: `The holdout lacks columns the live model needs` | The holdout was cleaned differently from the training data. Rerun `code/data_cleaning.py` and `code/data_split.py`. |
| `replay.py` stops with `HTTP 422` on a clean week | A real data problem, not an injected one. The message names the column; fix the data or the scenario. |
| Replay alerts never fire | Weeks are too short for the 3-minute windows (use `--seconds-per-week 120`), or Prometheus was not restarted after editing `alerts_replay.yml`. Check http://localhost:9090/alerts. |
| `rollback_drill.py --finish`: `no @previous_champion` | The drill was not started, or it was already finished. Check the Models tab in MLflow. |
| `drift_report.py`: `No logged stays with run_id ...` | The run id is wrong, or the log was deleted or the replay ran against another API. Filter check: `python -c "import pandas as pd; d=pd.read_json('serving/logs/predictions.jsonl', lines=True); print(d.run_id.value_counts())"`. |
| `drift_report.py`: `The baseline weeks [...] hold only N stays` | `--baseline-weeks` names weeks that are not in the run. Use weeks that exist, for example `1-4`. |
| Every week shows moderate drift, even clean ones | The baseline is too small or not clean. Use more clean weeks, and compare with the noise floor printed in the report. |
| Score PSI is high but no feature drifted | The model version changed during the run (a drill or a reload); a different model gives different scores. |
| Prometheus target `readmission-api` is DOWN | The API container is not running or not healthy. Check `docker compose ps`. |
| Grafana panels say "No data" | Make sure the data source is Prometheus, send some traffic (`python serving/send_sample.py --n 200`) and widen the time range. |

---

## 17. File-by-file guide

How to read this section: **Significance** is why the file exists; **What it does** is the code in plain words.
Section 18 goes function by function.

### 17.1 Data preparation (`code/`)

| File | Significance | What it does |
|---|---|---|
| `code/data_cleaning.py` | Turns the raw UCI file into a clean, readable dataset and records exactly how. Plan steps 1 to 3. | Reads every column as a string (so IDs and ICD-9 codes keep their exact values; only `?` is missing). Drops expired / hospice discharges (IDs 11, 13, 14, 19, 20, 21) and invalid gender. Replaces `None` in the lab columns with `not_measured` and other missing values with `missing`. Merges "unknown / not mapped" admission and discharge IDs into `unknown`. Maps the label (`<30` to 1, else 0) and keeps `readmitted_raw`. Validates the drug columns (only No / Down / Steady / Up). Adds `n_drugs_taken`, `n_drugs_changed` and `diag_1/2/3_group` (via `icd9_group`). Drops `weight`, `examide`, `citoglipton` and any constant column. Writes the cleaned CSV, `cleaning_config.json` and `cleaning_report.txt`. Never touches the raw file and does not encode categories (encoding belongs in the training pipeline, fit on train only). |
| `code/data_cleaning.md` | Human explanation of the cleaning step. | Run results, row counts per step, the table of treatments, switches (`GROUP_UNKNOWN_IDS`, `DROP_RARE_DRUGS`, ...). |
| `code/leak_review.py` | Plan step 4: proves no feature gives away the answer. | A hand-written table (`build_review`) gives every column EXCLUDE / KEEP / KEEP-WATCH / TARGET with a reason, and the script stops if a column has no verdict. Then seven checks: single-column AUC for numeric columns, AUC of the ID columns (should be about 0.5), out-of-fold target-encoding AUC for categoricals (grouped by patient), readmission rates for key categories, a demo of split leakage (41.5% of rows share a patient under a random split vs 0 under the grouped split), and a whole-model `HistGradientBoostingClassifier` run three ways (allowed features, without discharge disposition, plus `encounter_id` as a deliberate leak demo). Statistics only flag; the table decides. |
| `code/leak_review.md` | Human explanation and Q&A for the leak review. | Results (highest single AUC 0.607, model AUC 0.669, no leak flagged), open decisions, how unseen categories should be handled, how leak review differs from VIF/correlation. |
| `code/data_split.py` | Plan step 5: the three piles. | `StratifiedGroupKFold` with 20 folds, shuffled, seed 42, grouped by `patient_nbr` and stratified on the label. Folds 12 / 3 / 5 become train / test / holdout (60 / 15 / 25%). Asserts every encounter is assigned, `encounter_id` is unique, and no patient appears in two splits. Writes the three CSVs, `split_assignments.csv` (source of truth) and `split_report.txt` (sizes, positive rates, unseen categories in test/holdout). |

### 17.2 Pipeline scripts (project root)

| File | Significance | What it does |
|---|---|---|
| `train.py` | Step 1. Produces the candidate models and their evidence. | Loads train only. Builds one `Pipeline(prep, model)` per model family. Scores with 5-fold `StratifiedGroupKFold` cross-validation (PR-AUC and ROC-AUC). Fits on all of train. Logs params, per-fold metrics, the feature list, data context files, SHAP or coefficients and the pipeline to MLflow. Saves ranked joblib files, `best_model.json`, `model_report.md` and `training_report.txt`. A crash in one model does not stop the others. |
| `test.py` | Step 2. The evaluation gate. | Loads every saved joblib, scores them once on the test split (or `--data`), computes PR-AUC, ROC-AUC, lift, a patient-level bootstrap (200 resamples) for 95% ranges and paired differences, Brier and ECE calibration, and fairness by gender / race / age. Applies the fixed gate rules, writes `evaluation.json`, reports and `eval-*` MLflow runs, and logs every dataset use. Never chooses the champion. |
| `register_champion.py` | Step 3. Puts models in the MLflow registry with aliases and evidence. | Reads `evaluation.json`, refuses failed models, matches each model to its training run by CV score, registers versions, sets `champion` / `challenger` / `baseline` aliases, tags versions with evidence, verifies the registry copy against the joblib, and writes `champion.json` / `champion_report.md`. Challenger mode registers one new version and leaves the champion alone. |
| `promote.py` | Step 4. The promotion rule and rollback. | Loads the champion and challenger from the registry, scores both on the same data, recomputes the gate checks for the challenger plus a paired bootstrap of PR-AUC gain versus the champion, promotes only if everything passes and the gain's 95% range is above zero. Swaps aliases, rewrites champion files and logs the decision; `--rollback` reverses. |
| `requirements.txt` | The exact packages for the pipeline. | Pinned versions for pandas, numpy, scikit-learn, xgboost, lightgbm, shap, matplotlib, mlflow, joblib, greenlet, prometheus-client. |
| `requirements-lock.txt` | A full snapshot of the original Windows environment. | Every installed package with exact versions (including Windows-only ones). For exact rebuilds on Windows. |
| `docker-compose.yml` | Starts the whole runtime stack. | Defines `readmission-api` (built from `./serving`), `prometheus` and `grafana`, their ports, mounts, environment, start order, restart policy and the two named volumes. |
| `.gitignore` | Keeps data, models, MLflow files and logs out of git. | See section 19. |

### 17.3 Serving (`serving/`)

| File | Significance | What it does |
|---|---|---|
| `serving/export_champion.py` | The bridge from registry to API. | Loads `models:/readmission@champion`, computes the high-risk cutoff (top 20% of test-split scores, no labels), writes `model.joblib` atomically (temp file then `os.replace`), then writes `meta.json` last (checksum, cutoff, example record, registry evidence tags, package versions, timestamps). Logs its use of test data to `data_use_log.jsonl`. |
| `serving/app.py` | The API. | FastAPI app that loads the bundle, validates input, scores, explains with SHAP, flags, logs predictions, exposes Prometheus metrics, supports hot reload and serves the console. Detailed in section 18. |
| `serving/send_sample.py` | Smoke test and seed for the replay. | Reads a CSV, drops outcome columns, posts rows in batches to `/predict`, prints scored count, flagged count and the first record. Standard library plus pandas. |
| `serving/replay.py` | Step 11: the holdout replay. | Builds simulated weeks (whole patients per week, seed 42), asks `/model-info` which columns the model needs, applies the scenarios of the active weeks, sends batches to `/predict` with the replay headers, pauses to pace each week, and writes `plan.json`, `labels.csv`, `summary.json` and `summary.md`. Never sends outcome columns. Stops cleanly on error or Ctrl+C and can resume with `--start-week`. `--show-examples` prints real before/after rows for each scenario and sends nothing. |
| `serving/rollback_drill.py` | Step 11: rehearse a bad model and the rollback. | `--start`: swaps the champion alias to a worse model using `promote.py`'s functions, logs `drill_swap`, exports and reloads. `--finish`: runs `promote.py --rollback`, restores the baseline tag, exports and reloads. |
| `serving/Dockerfile` | Recipe for the API image. | `python:3.9-slim`, installs `libgomp1`, installs pinned requirements first (layer cache), copies `app.py` and `ui/`, runs as non-root `appuser`, adds a `HEALTHCHECK` on `/health`, starts `uvicorn` with one worker (Prometheus counters live in-process). |
| `serving/requirements-serve.txt` | Packages inside the container. | Same model-related pins as training plus FastAPI, Starlette, Pydantic, Uvicorn and prometheus-client. |
| `serving/.dockerignore` | Keeps the image small and model-free. | Excludes `bundle/`, `logs/`, caches, `export_champion.py`, `send_sample.py`. |
| `serving/ui/index.html` | The console's page shell. | Header with status pill and tabs, and four sections (score, batch, model, monitor) filled by `app.js`. |
| `serving/ui/styles.css` | The console's look. | CSS variables for light and dark mode (`prefers-color-scheme`), cards, form grid, meter, driver bars, histogram, tables, toasts. |
| `serving/ui/app.js` | The console's behaviour. | Talks only to its own API. Builds the form from `/model-info`, scores one stay, uploads and chunks CSVs, renders results and the CSV download, shows model evidence, parses `/metrics` text for the monitoring tab. Detailed in section 18. |

### 17.4 Monitoring (`monitoring/`)

| File | Significance | What it does |
|---|---|---|
| `monitoring/prometheus/prometheus.yml` | Tells Prometheus what to scrape. | 15 s scrape and evaluation interval; scrapes `readmission-api:8000/metrics` (the compose service name) and Prometheus itself; loads `alerts.yml`. |
| `monitoring/prometheus/alerts.yml` | The five alert rules. |
| `monitoring/prometheus/alerts_replay.yml` | Four replay alert rules. | Same ideas with 3-minute windows: flagged share outside 12% to 28% (needs over 200 stays), more than 25 unseen values, over 5% non-2xx, and a model load in the last 10 minutes. | PromQL for ApiDown, FlaggedShareOutOfRange, PredictLatencyHigh, ErrorShareHigh, UnseenCategoriesRising (section 9). |
| `monitoring/grafana/provisioning/datasources/prometheus.yml` | Connects Grafana to Prometheus automatically. | Declares a data source with uid `prometheus` at `http://prometheus:9090`. |
| `monitoring/grafana/starter_dashboard.json` | A ready dashboard. | 13 panels over the API's metrics; import it in Grafana. |
| `monitoring/drift_report.py` | Step 13: per-week drift report. | Reads the prediction log for one run id, builds bins and category lists from the baseline weeks, measures a noise floor, computes PSI of every feature and of the score for every week, describes each change, checks each injected test against the feature it should move, and writes `drift_report.md`, `psi_by_week.csv`, `drift_summary.json` and `drift_heatmap.png`. |
| `monitoring/README.md` | Short monitoring notes. | Addresses, logins, file list, and the "use rate() not raw totals" rule. |

### 17.5 Generated files (not in git)

| File | Written by | Read by | Safe to delete? |
|---|---|---|---|
| `data/splits/*.csv`, `split_report.txt`, `data/cleaning_config.json` | `code/` scripts | `train.py`, `test.py`, export | Regenerate with the `code/` scripts |
| `models/*.joblib` | `train.py` | `test.py`, `register_champion.py` (verification) | Yes, but rerun training |
| `models/best_model.json`, `model_report.md`, `data/training_report.txt` | `train.py` | people | Yes (rewritten) |
| `models/evaluation.json` | `test.py` | `register_champion.py` | Yes, rerun `test.py` |
| `models/test_report.md`, `data/test_report.txt` | `test.py` | people | Yes (rewritten) |
| `models/evaluation_<name>.json`, `<name>_report.md` | `test.py --data` | people | Yes (rewritten) |
| `models/data_use_log.jsonl` | `test.py`, `export_champion.py` | people | Avoid: it is your history |
| `models/champion.json`, `champion_report.md` | `register_champion.py`, `promote.py` | `test.py`, people | Rewritten on next change |
| `models/promotion_log.jsonl`, `promotion_report.md` | `promote.py`, `register_champion.py --challenger` | people | Log: avoid (audit trail). Report: yes |
| `serving/bundle/model.joblib`, `meta.json` | `export_champion.py` | the API | Yes, re-export |
| `data/replay/<run-id>/*` | `replay.py` | people, step 14 (labels) | Avoid deleting `labels.csv` of a run you still want to analyse |
| `data/drift/<run-id>/*` | `drift_report.py` | people | Yes (regenerate from the log) |
| `serving/logs/predictions.jsonl` | the API | steps 13 and 14 (later) | Avoid: it is the live-traffic record |
| `mlflow.db`, `mlruns/` | MLflow (all scripts) | all scripts | **No** |
| `.venv/` | `python -m venv` | Python | Yes (rebuild) |

---

## 18. Function-level walkthrough of the code

### 18.1 `train.py`

| Function / block | What it does |
|---|---|
| `find_project_root()` | Walks up from the script's folder to the first folder that contains `data/`, so the script works from the root, `code/` or `data/`. |
| Constants | `NOT_FEATURES` (IDs, raw target, raw diagnosis codes), `NUMERIC` (10 numeric features), `LOG_COUNTS` (3 skewed visit counts), `MODEL_PARAMS`, `MIN_CATEGORY_ROWS = 50`, `SEED = 42`. Everything not numeric and not excluded is categorical. |
| `build_preprocessor(categorical)` | A `ColumnTransformer`: log counts -> `log1p` -> `StandardScaler`; other numerics -> `StandardScaler`; categoricals -> `OneHotEncoder(handle_unknown="infrequent_if_exist", min_frequency=50)`. Rare and unseen categories share one `_infrequent_sklearn` column, which is how the API can score never-seen values. |
| `build_model(name, pos_weight)` | Returns the classifier and its parameters. XGBoost and LightGBM get `scale_pos_weight` (negatives / positives = 7.704); logistic uses `class_weight="balanced"`. |
| `log_explanations(...)` | Logistic: saves coefficients sorted by size. Trees: takes a 2,000-row sample, transforms it, runs `shap.TreeExplainer`, handles the different SHAP output shapes, saves mean absolute SHAP to CSV and a bar chart, logs both to MLflow, and returns the ranking. |
| `log_pipeline(...)` | Logs the fitted pipeline with `mlflow.sklearn.log_model` using cloudpickle (the default skops format rejects XGBoost and LightGBM), with an input example; falls back to the MLflow 2 argument name. |
| `export_ranked_models(ok)` | Deletes old ranked joblibs (only files matching its own pattern), sorts by CV PR-AUC, writes `01_...`, `02_...`, and `best_model.json`. |
| `original_column`, `write_model_report` | Maps one-hot feature names back to source columns and writes the plain-language report (results table, verdict, SHAP lists, glossary). |
| `main()` | For each model: start an MLflow run, log params, run `cross_validate` with `StratifiedGroupKFold` (`error_score="raise"` so real errors are not hidden as NaN), check scores are finite, log mean/std/per-fold metrics, `fit` on all train, log artifacts, collect the result. Failures are caught per model. Then ranks, exports and writes reports. |

### 18.2 `test.py`

| Function / block | What it does |
|---|---|
| Gate constants | `MIN_LIFT 1.5`, `MAX_CV_DROP 0.02`, `MUST_BEAT_BASELINE`, `MAX_GROUP_AUC_GAP 0.05`, `MAX_RECALL_GAP 0.10`, `MIN_GROUP_ROWS 500`, `MIN_GROUP_POSITIVES 50`, `FLAG_FRACTION 0.20`, `N_BOOT 200`. Fixed before seeing results. |
| `load_models()` | Finds `NN_<model>_prauc<cv>.joblib` with a regex, loads each pipeline, and reads the CV score from the file name. |
| `ece(y, p)` | Expected calibration error with ten equal-count bins: average gap between mean predicted and actual rate. |
| `patient_bootstrap(...)` | Groups row indices by patient, resamples whole patients with replacement 200 times, and computes PR-AUC per model on **the same resample**, so differences are paired. |
| `fairness(...)` | For each level of gender / race / age: size, positives, ROC-AUC, recall at the top-20% cutoff, flag rate, and the gap to overall. Groups under 500 rows or 50 positives are listed but not gated. |
| `choose_reference(...)` | `--reference`, else the family in `champion.json`, else the best test PR-AUC. |
| `relabel(text, label)` | Rewords the report text when a dataset other than the test split is evaluated. |
| `write_plain_report(...)` | Writes `test_report.md`. |
| `main()` | Loads data, checks every model's expected columns exist, scores, bootstraps, builds per-model metrics, applies the checks (booleans converted to plain `bool` so JSON works), writes reports and `evaluation.json`, appends to `data_use_log.jsonl`, logs `eval-*` MLflow runs. |

### 18.3 `register_champion.py`

| Function | What it does |
|---|---|
| `find_training_run(...)` | Searches finished runs tagged with the model family, skips evaluation runs, returns the latest whose `cv_pr_auc_mean` is within 0.0006 of the value in `evaluation.json`. |
| `model_uri_for(...)` | MLflow 3 stores models as "logged models"; finds the one named `readmission_<family>` created by that run, else falls back to `runs:/<id>/readmission_<family>`. |
| `register_version(...)` | Reuses an existing version for the same run, otherwise creates the registered model (if needed) and registers a new version. |
| `verify_against_joblib(...)` | Loads the registry model and the saved joblib and requires identical predictions (`allclose`, 1e-9) on 500 test rows. |
| `register_challenger(...)` | One new version, alias `challenger`, evidence tags, previous challenger tagged retired, log line. Refuses the current champion's run. |
| `main()` | Argument parsing (`--champion` or `--challenger`, `--reason` required, `--force`, `--no-verify`), gate and safety-guard checks, registers all versions in champion mode, verifies, writes files. |
| `write_champion_files(...)` | Writes `champion.json` and `champion_report.md`. |

### 18.4 `promote.py`

| Function | What it does |
|---|---|
| `load_gate()` | Imports `test.py` by file path with `importlib` so thresholds and helpers (`fairness`, `patient_bootstrap`) are shared, never copied. |
| `get_version`, `score`, `family`, `cv_score` | Resolve a version number or alias, load that registry model and score a dataframe, read tags and the training run's CV metric. |
| `refresh_champion_files(...)` | Rewrites `champion.json` and `champion_report.md` from registry tags (removing stale fields). |
| `swap_aliases(...)` | Sets `previous_champion` to the old version and `champion` to the new one, updates `role` tags, removes `challenger` if it pointed at the new champion. |
| `do_rollback(...)` | Calls `swap_aliases` in reverse and logs it. |
| `main()` | Scores champion and candidate, computes PR-AUC, lift, fairness gaps, the paired bootstrap difference, builds the checks dict (including "gain's 95% lower bound above zero"), prints the decision, logs (not for dry runs), writes `promotion_report.md`, and applies the swap if it wins. |

### 18.5 `serving/export_champion.py`

| Function | What it does |
|---|---|
| `sha256(path)` | Streams the file in 1 MB chunks and returns its hash. |
| `package_versions()` | Records the versions of the key packages at export time. |
| `write_bundle(...)` | Reads the required columns from the pipeline, scores the reference data, sets the cutoff at the 80th percentile of scores, writes the model atomically, then `meta.json` last with the checksum. A half-finished export therefore fails the API's checksum test instead of loading. |
| `main()` | Resolves the alias, loads `models:/readmission/<version>`, builds metadata (registry tags such as gate, scores, selection reason), writes the bundle, logs the data use and prints a summary. |

### 18.6 `serving/app.py`

| Function / block | What it does |
|---|---|
| Metrics objects | Counters, histograms and gauges: requests by method/path/status, latency, predictions by outcome, score distribution (20 buckets), batch size, validation errors by reason, unseen categories by column, model info, load time, reload outcomes, replay week. |
| `ModelState` | A dataclass holding everything about the live model (pipeline, meta, columns, categories, explainer, cutoff, load time, warnings). The module-level `_state` is swapped as one object. |
| `inspect_pipeline(pipe)` | Reads the input contract **from the fitted pipeline itself**: required columns, which are numeric, the categories each `OneHotEncoder` learned (and whether they were strings or numbers), and a map from each one-hot feature name back to its source column. |
| `load_state(dir)` | Checks both bundle files exist, verifies the SHA-256 against `meta.json`, loads the pipeline, checks the step names `prep` and `model`, builds a SHAP `TreeExplainer` (or marks logistic as linear), and compares package versions with the export. |
| `activate`, `try_reload` | Under a lock, load a new state and only then swap it in and update the model-info gauge. If loading fails, the old model keeps serving and a "failed" reload is counted. |
| `lifespan` | Tries to load at startup; if there is no bundle it logs an error and still starts (health returns 503). |
| `metrics_middleware` | Times every request and counts it by **route template** (not raw URL, so labels stay bounded); skips `/metrics` itself. |
| `reject(...)` | Counts a validation error and raises HTTP 422 with a structured list of problems (column, issue, first bad rows). |
| `parse_replay_headers` | Validates `X-Replay-Run` and `X-Replay-Week`. |
| `build_frame(...)` | Validation in order: missing columns, nulls, numeric conversion, categorical normalisation (a float like `1.0` becomes `"1"`), unseen category detection (counted and returned, not rejected). Returns the frame, ignored columns, unseen values and ids. |
| `explain(...)` | Transforms the frame, computes SHAP values (or coefficient times value for logistic), and returns the top-k drivers per record in log-odds with the original input value. |
| `log_predictions(...)` | Appends one JSON line per scored record under a lock; a logging failure never fails the request. |
| `predict` | Validates, scores, flags (`score >= cutoff`), optionally explains, updates metrics, logs, and returns scores, flag rule, model identity and notes. |
| `example`, `model_info`, `health`, `metrics`, `reload_model`, `console`, `favicon` | The remaining endpoints. `/model-info` also returns the allowed categories, which is how the console builds its dropdowns. |
| `NoCacheStaticFiles` | Serves `/ui/*` with `Cache-Control: no-cache`, so edits show immediately. |

### 18.7 `serving/ui/app.js`

| Function / block | What it does |
|---|---|
| `el`, `put`, `$`, `$$` | Tiny DOM helpers; all text is set with `textContent`/text nodes, so server data is never injected as HTML. |
| `request(path, opts)` | `fetch` wrapper that parses JSON when possible and throws `{status, body}` on errors. |
| `LABELS`, `GROUPS` | Friendly names and the grouping of form fields (Patient, Admission, Care, Prior use, Diagnoses, Medications). |
| `makeField`, `buildForm`, `fillForm`, `collect` | Build one input per required column (number box or dropdown plus an "Other" box), fill from an example, and read the form back into a record. |
| `score`, `renderResult`, `meter`, `drivers`, `renderError` | POST one record, draw the score, the cutoff meter and the driver bars, and map 422 problems onto the offending form fields. |
| `parseCSV`, `loadCSV`, `rowToRecord`, `runBatch`, `renderBatch`, `downloadResults` | Parse a CSV in the browser (quoted fields handled), check required columns, convert rows to records, send 500-row chunks with a progress bar and cancel, then show stats, histogram, sortable/searchable table and CSV export. |
| `renderModel`, `reloadModel` | Show registry evidence and call `/reload` (asks for an admin token on 403). |
| `parseProm`, `quantileLE`, `loadMonitor` | Parse Prometheus text format, compute totals and an approximate p95 from histogram buckets, and draw the monitoring cards. |
| `init`, `showView`, `pollHealth` | Wiring, hash-based tab routing, and the status pill (polls `/health` every 15 s). |

### 18.8 `code/` scripts, key functions

| Function | What it does |
|---|---|
| `data_cleaning.icd9_group(code)` | Maps a diagnosis code to a broad group: V and E codes, then numeric ranges (diabetes 250.x, circulatory 390-459 and 785, respiratory 460-519 and 786, and so on). |
| `leak_review.build_review()` | The hand-written verdict table. |
| `leak_review.two_sided_auc`, `oof_target_encoding_auc`, `grouped_model_auc` | Direction-free AUC for a single column; out-of-fold mean-target AUC for a category column with patient-grouped folds; a patient-grouped gradient boosting AUC with ordinal-encoded categories. |
| `data_split.main()` | Fold assignment, fold-to-split mapping, hard checks, file writing and the report. |

### 18.9 `serving/replay.py`

| Function / block | What it does |
|---|---|
| Constants | `FLAG_LOW/HIGH` (0.12 and 0.28, same band as the alerts), `DEFAULT_SEVERITY`, `PRESETS` (the 12-week `demo` timeline), `EXPECT` (what each scenario should trigger). |
| `call(...)`, `as_json`, `api_counters` | Standard-library HTTP helpers. `api_counters` reads the API's unseen-value and rejected-request totals from `/metrics`, so each week can report its own change. |
| `parse_inject(spec, n_weeks)` | Parses `name:first-last:severity` and validates the name, the week range and the severity. |
| `assign_weeks(df, n_weeks, seed)` | Shuffles the patient list with a fixed seed, then walks through it cumulatively so weeks get about equal numbers of stays and a patient never spans two weeks. |
| `covariate_shift`, `population_shift`, `new_categories` | Data scenarios applied to a week's feature frame. They keep the original index, so labels still line up after rows are dropped. `new_categories` picks a value that is certainly not in the model's known categories (a big integer for numeric-coded columns, a string otherwise). |
| `corrupt_batch(records, k, info)` | The `bad_data` scenario: rotates between a null in a numeric column, text in a numeric column and a removed required column, so all three rejection reasons appear. |
| `post_predict`, `reject_reason` | One `/predict` call with the replay headers and timing; extracts the API's rejection reason from a 422. |
| `info_from_holdout`, `show_examples` | `--show-examples`: runs the same scenario functions on the first active week of each test and prints real before/after rows (changed values, age mix, broken-record kinds). Uses `/model-info` if the API is up, otherwise infers the columns from the holdout. Sends nothing. |
| `write_summary(...)` | Writes `summary.json` and `summary.md`, including the clean-week flagged share and which weeks fell outside the alert band. |
| `main()` | Validates arguments, builds and prints the plan (`--dry-run` stops here), checks `/health` and `/model-info`, checks the holdout has the needed columns, writes `plan.json` and the data-use log line, then loops over weeks: apply scenarios, send in batches (sending a corrupted copy first when `bad_data` is active, then the clean batch), collect scores and flags, append labels for the stays actually scored, print one line per week, and sleep out the rest of the week. On resume it drops old rows of the resumed week from `labels.csv`. |

### 18.10 `serving/rollback_drill.py`

| Function / block | What it does |
|---|---|
| `export_and_reload(...)` | Runs `export_champion.py` as a subprocess (it needs the same environment), then `POST /reload`, and prints the version change. A refused reload is reported and the old model keeps serving. |
| `main()` with `--start` | Resolves the current champion and the target (`--to`, default `@baseline`), asks for confirmation, calls `promote.swap_aliases`, `promote.refresh_champion_files` and `promote.append_log` (decision `drill_swap`), then exports and reloads. |
| `main()` with `--finish` | Checks `@previous_champion` exists, runs `promote.py --rollback` as the real tool, restores the baseline version's `role` tag (which `swap_aliases` changes), then exports and reloads. |

### 18.11 `monitoring/drift_report.py`

| Function / block | What it does |
|---|---|
| Constants | `EPS` (floor for empty bins), `RARE_SHARE` 0.5%, `MAX_DISCRETE` 12, `N_BINS` 10, `MODERATE` 0.10, `MAJOR` 0.25, `DETECT_FEATURES` (which features should react to which injected test). |
| `parse_weeks("1-4")` | Turns `1-4` or `1,2,3` into a list of weeks. |
| `load_log(path, run_id)` | Reads the JSON-lines log, keeps only rows of this run with a replay week, flattens the `features` dict into columns, and drops duplicate `(id, week)` rows (a resumed week is logged twice). |
| `make_profile(series)` | Learns the bins from the baseline: deciles for a numeric feature with many values; otherwise a category list, with categories under 0.5% pooled into `__rare__`. |
| `shares(profile, series)` | Share of stays per bin or category. Values not in the baseline go to `__unseen__`, so a new code is visible on its own. |
| `psi(expected, actual)` | `sum((a - e) * ln(a / e))` with empty bins floored at 0.0001. |
| `level(v)` | Stable, moderate or major. |
| `describe(...)` | One sentence on what changed (a mean shift, or the three biggest share changes). |
| `main()` | Loads the log, builds baseline profiles, computes the noise floor (a clean sample of typical week size against the rest of the baseline), computes PSI for every week and feature, builds the weekly summary and the "did we catch it" table from `plan.json`, draws the heatmap and writes the four output files. |

---

## 19. Publishing to GitHub

### 19.1 What to commit

Commit **code, configs and documentation**. Do not commit data, trained models, the MLflow store or logs:

- The data is a public dataset others can download, the CSVs are large, and the prediction log holds patient-level rows.
- `mlflow.db` stores absolute paths from your machine (for example `C:\Users\<you>\...`), so it would not work for anyone else. A clone rebuilds it in minutes (section 3).
- `promotion_log.jsonl` and `data_use_log.jsonl` also contain local file paths.

Suggested `.gitignore`:

```
# environment and caches
.venv/
__pycache__/
*.pyc
.env

# data (download from UCI, see README section 3.3)
data/*.csv
data/splits/
data/replay/
data/drift/
data/*_report.txt
data/leak_review_columns.csv

# pipeline outputs (rebuilt by running the scripts)
models/
mlflow.db
mlruns/

# serving runtime files
serving/bundle/
serving/logs/
```

### 19.2 Fix before you push

1. **Folder and file name case.** Docker on Linux is case-sensitive. The code uses `serving/ui` and
   `serving/Dockerfile`. On a case-insensitive Windows disk Git can keep the wrong case (`serving/UI`,
   `serving/dockerfile`). Rename in two steps so Git records the change:
   ```bash
   git mv serving/UI serving/ui_tmp && git mv serving/ui_tmp serving/ui
   git mv serving/dockerfile serving/Dockerfile_tmp && git mv serving/Dockerfile_tmp serving/Dockerfile
   ```
2. **Delete the stale `serving/docker-compose.yml`.** The real one is the root `docker-compose.yml` (it also has
   Prometheus and Grafana, and its paths are relative to the root).
3. **Delete `code/readme.md`** (an old copy of this README) and the empty `practice.py`.
4. **Add a `LICENSE`** for your code, and keep the UCI citation (CC BY 4.0).
5. **Replace the placeholder** `<your-repo-url>` in section 3.2.
6. If you already pushed files that should be ignored, stop tracking them:
   ```bash
   git rm -r --cached data models mlruns mlflow.db serving/bundle serving/logs
   git commit -m "Stop tracking data, models and local artifacts"
   ```
   They remain in earlier commits. If any contained private paths or patient-level rows, remove them from history
   (for example with `git filter-repo`) or recreate the repository.

### 19.3 Optional next steps for the repo

A GitHub Actions workflow that builds the image and runs a smoke test, a `Makefile` or script that runs section 3.5
in one command, and a few unit tests for `build_frame` and `icd9_group`.