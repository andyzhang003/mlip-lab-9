# Lab 9 : Data & Pipeline Versioning — DVC and Roar

In this lab you will run the same ML pipeline through two tools that take fundamentally different approaches to pipeline tracking and reproducibility:

- **DVC (Data Version Control)** — a *declarative* tool where you explicitly define your pipeline stages, dependencies, and outputs in a YAML file. You tell DVC what your pipeline looks like.
- **Roar (Run Observation & Artifact Registration)** — an *implicit* tool that observes file I/O as your code runs and automatically infers the dependency graph. You just run your code; Roar figures out the pipeline.

By using both on the same pipeline, you will understand not just how each tool works, but *why* these different design philosophies exist and when each is appropriate.


## Learning Objectives

1. Version datasets and models using DVC alongside Git
2. Run declarative, reproducible ML pipelines with `dvc.yaml`
3. Run and compare experiments with DVC's experiment tracking
4. Use Roar to implicitly track pipeline lineage without configuration files
5. Inspect and interpret auto-inferred lineage DAGs
6. Articulate when to use declarative (DVC) vs. observational (Roar) pipeline tracking

## The Dataset and Model

This lab uses the **Breast Cancer Wisconsin (Diagnostic)** dataset (569 samples, 30 numeric features computed from digitized images of breast tissue). The target variable is binary: **malignant (0)** or **benign (1)**. The pipeline trains a **Random Forest classifier** to predict the diagnosis from the features.

The dataset is small enough to version in Git for convenience, but in a real project this is exactly the kind of artifact you would manage with DVC.

## Deliverables

- **Deliverable 1 (Run DVC & Roar)**: Complete the step-by-step instructions in [INSTRUCTIONS.md](INSTRUCTIONS.md). Show the TA:
  - Your `dvc dag` output and `roar dag` output
  - The registered artifact lineage on [glaas.ai](https://glaas.ai/)

- **Deliverable 2 (Experimentation)**: Run 2–3 experiments with parameters you choose (different from the defaults in `params.yaml`) using **both** DVC and Roar. Show the TA how you compare results across runs in each tool and how you can trace back which data and parameters produced a given model.

- **Deliverable 3 (Reflection)**: Complete the [summary table](#summary-table) below and be prepared to discuss the [reflection questions](#reflection-questions) with the TA.

## How to Use This Repo

| File | What it contains |
|------|-----------------|
| **This README** | Overview, setup, reflection questions, and deliverables |
| [**INSTRUCTIONS.md**](INSTRUCTIONS.md) | Step-by-step commands for DVC (Part 1) and Roar (Part 2) |
| [**TROUBLESHOOTING.md**](TROUBLESHOOTING.md) | Common errors and how to fix them |

---

## Setup

### Prerequisites

- Git installed
- Python 3.10+ (installed via Homebrew, pyenv, or a standard installer — **not** the macOS system Python at `/usr/bin/python3`)
- A terminal environment (see platform notes below)

> **Platform notes**:
> - **macOS**: Both DVC and Roar work natively. Roar requires a non-Apple Python (Homebrew, pyenv, conda, or python.org installer). Run `which python3` — if it shows `/usr/bin/python3`, see [TROUBLESHOOTING.md](TROUBLESHOOTING.md). If you run into any issues on macOS, we recommend that you work on your team's VM.
> - **Windows**: DVC works natively. Roar requires **WSL2** (Windows Subsystem for Linux). Install it with `wsl --install` in PowerShell if you don't have it, then do the entire lab inside WSL. If you run into any issues on Windows, we recommend that you work on your team's VM.
> - **Linux**: Everything works out of the box.

### Installation

1. Clone this repository:

```bash
git clone https://github.com/AshrithaG/mlip-lab-9.git
cd mlip-lab-9
```

2. Create and activate a virtual environment:

```bash
python3 -m venv venv
source venv/bin/activate
```

3. Install dependencies:

```bash
pip install -r requirements.txt
```

4. Verify Roar installed correctly:

```bash
roar --version
```

### Repository Structure

```
mlip-lab-9/
├── data/raw/data.csv          # Breast Cancer Wisconsin dataset (569 samples, 30 features)
├── scripts/
│   ├── preprocess.py          # Train/test split
│   ├── train.py               # Random Forest training
│   ├── evaluate.py            # Model evaluation (accuracy, precision, recall, F1)
│   └── augment_data.py        # Adds synthetic rows to the dataset
├── params.yaml                # Hyperparameters for the pipeline
├── dvc.yaml                   # DVC pipeline definition (3 stages)
└── requirements.txt
```

### Verify the Pipeline

Before using any versioning tools, make sure the pipeline runs end-to-end:

```bash
python3 scripts/preprocess.py
python3 scripts/train.py
python3 scripts/evaluate.py
```

You should see output showing the data split, trained model parameters, and evaluation metrics. Fix any errors before proceeding.

---

**Next step**: Open [INSTRUCTIONS.md](INSTRUCTIONS.md) to begin Part 1 (DVC) and Part 2 (Roar). Once you have completed both parts, return here for Part 3.

---

## Part 3: Compare and Reflect

You have now used both tools on the same pipeline. Fill in the summary table and be ready to discuss the questions below with the TA. Refer to the official documentation as needed:

- [DVC Pipelines Guide](https://dvc.org/doc/user-guide/pipelines)
- [DVC Experiments Guide](https://dvc.org/doc/user-guide/experiment-management)
- [Roar on PyPI](https://pypi.org/project/roar-cli/)
- [GLaaS — Global Lineage as a Service](https://glaas.ai/)

### Reflection Questions

**Q1 — Setup and configuration**:
Compare the setup effort for DVC vs. Roar. What did you have to configure explicitly for DVC that Roar handled automatically? What did DVC give you that Roar did not?

**A1**: DVC required creating a `dvc.yaml` pipeline definition with explicit `deps` and `outs` for each stage, configuring a remote storage backend, and tracking data files with `dvc add`. Roar only needed `roar init` and then prefixing commands with `roar run` — no YAML, no remote config, no explicit dependency declarations. However, DVC gave us smart caching (skipping unchanged stages) and a formal experiment tracking interface (`dvc exp run`, `dvc exp show`) that Roar does not provide out of the box.

**Q2 — The DAGs**:
Look at both DAG outputs side by side. Do they show the same pipeline structure? Are there any differences in what each tool captured? Which was easier to understand?

**A2**: Both DAGs show the same three-stage linear flow: preprocess → train → evaluate. The key difference is scope: DVC's `dvc dag` shows the static pipeline as declared in `dvc.yaml` — clean and easy to read. Roar's `roar dag` shows the full execution history, including all past runs, superseded outputs, and the augment step that was never in the DVC definition. Roar's DAG is more complete but more verbose. DVC's DAG is easier to understand for pipeline design; Roar's is more useful for auditing what actually happened.

**Q3 — Handling changes**:
When you changed a hyperparameter (DVC) or augmented the dataset (Roar), how did each tool respond? With DVC, what determined which stages re-ran? With Roar, who decided what to re-run?

**A3**: With DVC, changing `n_estimators` in `params.yaml` caused only `train` and `evaluate` to re-run — `preprocess` was skipped because none of its declared dependencies changed. DVC determines which stages re-run by comparing content hashes of all declared `deps` against the `dvc.lock`. With Roar, there is no automatic re-run detection — the developer decides which commands to re-run. Roar simply records each run as a new job and links outputs by file hash, marking older outputs as superseded. DVC's model is more powerful for skipping unnecessary work; Roar's model is simpler and more flexible.

**Q4 — Experiment tracking and comparison**:
You ran multiple experiments with both DVC and Roar. How did you find and compare results across runs in each tool? How can you trace back which data version and which parameters produced a specific model? Which tool made this easier?

**A4**: With DVC, `dvc exp show` provides a formatted table comparing all experiments side-by-side: parameters, metrics, and artifact hashes in one view. You can trace back to the exact data and params used in any experiment. With Roar, you use `roar show @N` to inspect individual jobs, but there is no built-in comparison table — you have to inspect each run separately and compare manually. For experiment comparison, DVC is clearly easier. For tracing artifact lineage (what data+code produced this exact model file), Roar is more powerful via GLaaS lookup by content hash.

**Q5 — Team collaboration**:
Imagine a new teammate joins your project next month. With DVC, they can read `dvc.yaml` in the Git repo. With Roar, they can look up artifact lineage on GLaaS. Which gives a clearer picture of the pipeline?

**A5**: For understanding the pipeline structure, DVC's `dvc.yaml` is clearer — it is explicit, human-readable, and versioned in Git alongside the code. A new teammate can immediately see all stages and dependencies. Roar's GLaaS lineage is better for auditing a specific artifact — they can look up a model file's hash and see exactly what data, code, and environment produced it, even months later. Both tools are complementary: you could use DVC to define and run your pipeline reproducibly, while using Roar (or DVC's push to remote) to make specific model artifacts globally discoverable by hash.

**Q6 — Your project**:
For your team's course project, which approach (or combination) would be more useful? Consider your data sources, pipeline complexity, and how your team collaborates.

**A6**: For a course project with a moderate-complexity ML pipeline and a small team, DVC is likely more useful as the primary tool. The explicit `dvc.yaml` makes the pipeline self-documenting, `dvc repro` ensures reproducibility for teammates, and `dvc exp show` makes it easy to compare model runs. Roar would be a valuable complement for artifact lineage — registering final model versions on GLaaS so any stakeholder can trace exactly what data and code produced a given model. If the data is large or changes frequently, DVC's data versioning becomes essential. If the pipeline is exploratory and changes rapidly, Roar's zero-config approach reduces friction during development.

### Summary Table

| | DVC | Roar |
|---|---|---|
| **Philosophy** | Declarative — you explicitly define pipeline stages, dependencies, and outputs in `dvc.yaml` before running | Observational — no config needed; Roar intercepts file I/O at runtime and infers the DAG automatically |
| **Config required** | Yes — `dvc.yaml` (pipeline stages), `params.yaml` (hyperparameters), DVC remote configuration | Minimal — only `roar init`; no pipeline YAML needed; params tracked implicitly via file reads |
| **How is the DAG defined?** | Manually authored in `dvc.yaml` with explicit `deps` and `outs` for each stage | Automatically inferred by observing which files each process reads and writes during execution |
| **What happens on re-run?** | DVC checks content hashes of all dependencies; only stages with changed inputs are re-executed (smart caching) | All stages are re-run by default; Roar records each run as a new job and marks prior outputs as superseded |
| **How are artifacts versioned?** | Files tracked in `.dvc` files (content-addressed cache); `dvc.lock` records exact hashes for reproducibility | Each run's inputs/outputs are hashed and stored in a local SQLite DB; artifacts can be published to GLaaS by hash |
| **Reproducibility mechanism** | `dvc repro` replays the exact pipeline defined in `dvc.yaml` using cached outputs when possible | `roar reproduce <hash>` replays the job chain that produced a given artifact hash |
| **Collaboration model** | Pipeline definition lives in Git (`dvc.yaml`, `dvc.lock`); data/models shared via DVC remote storage (S3, GCS, local) | Lineage published to GLaaS (cloud); teammates look up artifacts by content hash; no shared config files needed |
| **Platform support** | Works natively on macOS, Linux, Windows; broad remote storage support (S3, GCS, Azure, SSH, etc.) | Works on macOS and Linux natively; Windows requires WSL2; cloud lineage via GLaaS |


## Resources

- [DVC Documentation](https://dvc.org/doc)
- [DVC Pipelines Guide](https://dvc.org/doc/user-guide/pipelines)
- [DVC Experiments Guide](https://dvc.org/doc/user-guide/experiment-management)
- [Roar on PyPI](https://pypi.org/project/roar-cli/)
- [GLaaS — Global Lineage as a Service](https://glaas.ai/)
