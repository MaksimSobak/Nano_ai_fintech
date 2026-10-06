# Alert escalation prediction — complete hackathon project

This project estimates the probability that a synthetic financial-monitoring alert will be escalated. It includes executed feature engineering, model comparison, validation, final trained models, a verified 6,000-row prediction CSV, a research report, and a public EDA website.

## Start here

- **Submit `submission.csv`**: exactly `signal_id,ehtimollik`, in the supplied template's order. `eskalatsiya` is the training target. The actual attached template, rather than earlier conversations about column names, determines the output header.
- Read `RESEARCH_REPORT.md` for findings, model selection, measured results and limitations.
- The trained model is `artifacts/model.joblib`; validation results are `artifacts/metrics.json`.
- `WEBSITE_URL.txt` contains the public EDA URL. `build_website.py` regenerates the site from reports.
- Raw input files are not duplicated inside this package: use the five original competition files you supplied. Both CSV and Parquet transactions are accepted.

## Results from the supplied files

| Evaluation | ROC-AUC |
|---|---:|
| Development out-of-fold, selected blend | 0.6252 |
| Untouched 2,800-alert holdout | 0.6416 |
| Chronological sensitivity check | 0.6521 |

The holdout bootstrap 95% interval is 0.6138–0.6690. The chronological check is not an independent confirmation because model selection used some of the same period. Hidden-test performance is unknown. Reported holdout results evaluate the selected recipe; the final six-model all-data ensemble has no independent score.

## Setup

Use **Python 3.12** (the tested version), not Python 3.8. Create a virtual environment in the project folder and keep it out of version control.

macOS / Linux:

```bash
python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

Windows PowerShell:

```powershell
py -3.12 -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

The observed environment had about 10 GiB RAM; use at least 8–12 GiB available RAM for the full in-memory pipeline. Feature extraction dominates memory use. Run train and test feature extraction sequentially on smaller machines. macOS may need OpenMP installed for LightGBM; consult its official installation guide if import reports a missing `libomp`.

## Reproduce everything with one command

Place organizer files in `data/` using the names below, then run from this project folder. If your transactions are Parquet, change the two transaction filenames to `.parquet`.

```bash
python run_pipeline.py --train-signals data/train_signals.csv --train-transactions data/train_transactions.csv --test-signals data/test_signals.csv --test-transactions data/test_transactions.csv --sample data/sample_submission.csv --output-dir run_output
```

The pipeline runs `features.py` twice, `train.py`, `predict.py`, `eda.py`, and `build_website.py`. It writes input checksums, engineered tables, validation predictions, models, metrics, charts, `submission.csv`, and a static website under `run_output/`. It does not submit anything to the competition. Publish the generated `website/` directory on any static host if you want a separate URL.

To inspect the regenerated website locally:

```bash
python -m http.server 8000 --directory run_output/website
```

Open `http://localhost:8000`. This local URL is not suitable for organizer submission; use the public URL in `WEBSITE_URL.txt`.

## Predict using the delivered model (no retraining)

```bash
python predict.py --model artifacts/model.joblib --features artifacts/test_features.parquet --signals data/test_signals.csv --sample data/sample_submission.csv --output predictions_again.csv
```

For new test transactions, run the shared feature builder first:

```bash
python features.py --signals data/test_signals.csv --transactions data/test_transactions.csv --output new_test_features.parquet
```

Then pass `new_test_features.parquet` to `predict.py`. Only load model files you trust: joblib/pickle formats can execute code when loading.

## Python file responsibilities

| File | Responsibility |
|---|---|
| `config.py` | Seeds, windows, known directions and transaction types |
| `data_io.py` | CSV/Parquet loading and signal schema checks |
| `features.py` | Shared historical feature builder and transaction audit |
| `validation.py` | Metrics, confidence interval, fixed holdout split |
| `train.py` | Development CV comparison, holdout, temporal check, final refit |
| `predict.py` | Positive-class probabilities and exact submission contract |
| `eda.py` | Aggregate data research and eight chart files |
| `build_website.py` | Complete static EDA site from measured results |
| `run_pipeline.py` | End-to-end command with input hashes |
| `tests/test_pipeline.py` | Five boundary and contract tests |

These scripts replace notebook-state dependencies. You do not need to run Jupyter cells in a particular order. Person 1 can own data/features/EDA, Person 2 validation/training, and Person 3 prediction/site/reproducibility. All must use the same `features.py` and column contract.

## Verification

```bash
python -m unittest discover -s tests -v
python -m compileall -q .
```

Tests check the exact midnight cutoff, trailing-window boundaries, absent history, empty recent windows, invariance to added future transactions, unknown IDs and template order. `predict.py` additionally validates identifier sets, feature schema, finiteness and the [0,1] range. Input hashes are in `reports/input_manifest.json`; versions are pinned in `requirements.txt`.

## Important assumptions

- `signal_sanasi` has only a date: midnight is the conservative availability cutoff. Same-day later transactions are excluded.
- No customer ID is available; customer-disjoint validation cannot be performed.
- `miqdor_indeksi` is a standardized size indicator, not an amount in UZS. Negative values are valid.
- No targets, raw IDs, future-row counts or absolute dates are predictors.
- The data are synthetic; observed associations do not establish real-world criminal behavior.
