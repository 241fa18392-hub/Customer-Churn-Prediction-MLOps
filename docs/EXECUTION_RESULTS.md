# Customer Churn — Execution and Verification Results

## Run scope

- Date: 2026-11-07
- Project: `01_Customer_Churn_Prediction`
- Python: 3.13.15
- Isolated environment: `C:\v\churn`
- Pipeline command: `python src\mlops_pipeline\churn_prediction_pipeline.py --force`
- Final orchestrated result: all ten stages passed.

The existing incoming batch was already recorded in `data/raw/archive/dataset_lineage.json`. The loader validated the schema and batch hash, reported `already_ingested`, and did not append the batch. The `--force` option continued training on the existing current dataset without altering the raw dataset or archive snapshots.

## Dataset and model evaluation

| Item | Observed result |
|---|---|
| Source | `data/raw/churn.csv` |
| Target | `Churn` |
| Rows / columns | 7,344 / 21 |
| Training / test rows | 5,935 / 1,409 |
| Estimator | `RandomForestClassifier` |
| Configured parameters | 150 estimators, max depth 12, random state 42, balanced class weights |
| Accuracy | 0.7672 |
| Precision | 0.5466 |
| Recall | 0.7219 |
| F1 | 0.6221 |
| ROC-AUC | 0.8452 |
| Confusion matrix | `[[811, 224], [104, 270]]` |
| Quality gate | Recall 0.7219 >= 0.70; passed |

## MLflow and lifecycle

- Experiment: `churn_prediction_model_training_lab7`
- Registered model: `Telco_Churn_Production_Model`
- Version 1: Production; run ID `bc9845370a404a36b60420bd7fb25776`.
- Version 2: Archived; run ID `b2cb380b29cf48da9e9c7adccd554246`.
- Version 3: Archived; run ID `e4b9cd2ee70a42c0ac72564e2a3da2e4`.
- Version 4: Archived; run ID `479581f3d988471bad0d1435cd12e046`.
- Version 5: Archived; run ID `907783d83f1049a69311e3299e9a2b0f`.
- Registry verification after all runs confirmed v1 remains Production and v2–v5 are Archived. V5 tied v1 at recall 0.721925, so lifecycle correctly retained v1 and skipped promotion-gated deployment.
- The project-local MLflow store and registry were queried after execution; they are independent of the Fraud project store.

## Legacy Labs 3–6 and final rerun

After the integrated run, the separate Lab 3–6 runners were executed in order:

| Lab | Observed result |
|---|---|
| 3 — Baseline pipeline | Passed; accuracy 0.7842, precision 0.5656, recall 0.7887, F1 0.6588 |
| 4 — MLflow tracking | Passed; F1 0.6588 and ROC-AUC 0.8528; repeated execution produced the same F1 |
| 5 — Production data pipeline | Passed schema validation, preprocessing, and output validation |
| 6 — Registry and lifecycle | Registered candidate v3; recall 0.7139 and F1 0.6480; it was archived because it did not beat Production v1 recall 0.7219 |

Lab 5 initially stopped at a pre-existing syntax error (`report = {4`) in `src/validate_outputs.py`. The stray `4` was removed, after which Lab 5 passed.

The two churn notebooks were also executed using the project-specific Jupyter kernel:

| Notebook | Observed result |
|---|---|
| `notebooks/project_implementation.ipynb` | All 12 non-empty code cells completed with no cell errors. The notebook reported a best Logistic Regression recall of 0.7964. |
| `notebooks/churn_prediction.ipynb` | All 62 code cells completed with no cell errors. Its separate XGBoost model at threshold 0.45 measured ROC-AUC 0.8499 and PR-AUC 0.6628, and wrote the XGBoost pipeline and threshold artifacts. |

`project_implementation.ipynb` had a plotting-section cell incorrectly typed as Python while containing Markdown text; it was changed to a Markdown cell without changing model logic. The notebook environment was explicitly registered as `churn-mlops`, using `C:\v\churn` and the already-declared `xgboost==3.2.0` dependency. The XGBoost notebook's CV worker count was bounded to one for its final run to avoid non-fatal Windows joblib cleanup traces.

The baseline notebook rewrites the shared processed split, so the integrated pipeline was rerun afterward to restore its fixed split and canonical outputs. All ten stages passed; candidate v5 (run ID `907783d83f1049a69311e3299e9a2b0f`) tied Production v1 at recall 0.7219 and was archived. Production remained v1, and the promotion-gated deployment step correctly skipped deployment.

## API, failure handling, and deployment preparation

- Live FastAPI checks passed for `/health`, `/metadata`, and `/predict`.
- The API reported both model and preprocessor loaded. Sample prediction: class `0`, churn probability `0.323844`.
- `src/mlops_pipeline/10_failure_injection.py`: all five scenarios passed in each of three repetitions (15/15 checks), including recovery of the missing model artifact from the Production registry.
- `src/mlops_pipeline/11_prepare_deployment.py`: passed; the model, matching preprocessor, and metadata were reloaded and verified.
- The final pipeline log marks the promotion-gated deployment stage passed because the candidate was correctly not promoted. This is not a Docker build or cluster deployment result.
- Docker and `kubectl` were unavailable; containerization, Docker API testing, and Kubernetes deployment/scaling could not be executed.

## Tests and data integrity

- `C:\v\churn\Scripts\python.exe -m pytest -q`: 29 passed, 2 dependency deprecation warnings.
- The churn CSV SHA-256 remained `6030DE2A1B2EC4B12230B0D5C7F392B3A4A4C36FC138A442B7E3607B8AEB323C`.
- The incoming batch SHA-256 remained `0EB35AE7B184A85716FE0DEE59FB93A1EE8BD119DE898115D767F2B99671A7BC`.
- Existing archive files `churn_v01.csv`, `churn_v02.csv`, and `churn_v03.csv` remained present.

## Issues found and changes made

1. A first dependency installation under the long OneDrive workspace path failed on a deeply nested Jupyter asset because Windows long-path support is disabled. The retry succeeded in the short, separate environment `C:\v\churn`; no dependency manifest change or administrator setting was needed.
2. The first pipeline run reached deployment preparation but raised `UnicodeEncodeError` when the Windows console encoded box-drawing characters using CP1252. The bundle files had been written, but the stage returned failure.
3. `11_prepare_deployment.py` now prints an ASCII tree and creates/updates only its bundle files instead of deleting the entire `deployment/` directory. This prevents deletion of `deployment/deploy_production.py`. The deleted file was restored from the pre-execution backup before rerunning the stage.
4. `pytest` was absent from `requirements.txt`; it was installed only in the churn virtual environment.
5. After the fix, deployment preparation and the complete orchestrated pipeline passed. The original source datasets and archived versions were preserved.
6. The separately documented Lab 5 runner contained a syntax typo in `src/validate_outputs.py`. Removing the stray `4` restored the intended preprocessing-output validation; Lab 5 then passed and the integrated pipeline was rerun.
7. `project_implementation.ipynb` contained Markdown text in a code cell. Converting that cell to Markdown allowed the notebook to execute without changing its model logic. The analysis notebook was run with the project-specific kernel; XGBoost was already pinned in `requirements.txt`, so no dependency-manifest change was needed.

## Generated evidence

- `logs/churn_prediction_pipeline_run_log.json`
- `artifacts/data_load_report.json`
- `artifacts/preprocessing_summary_report.json`
- `reports/evaluation_report.json`
- `reports/quality_gate_report.json`
- `reports/lifecycle_report.json`
- `reports/failure_report.json`
- `outputs/inference_predictions.csv`
- `deployment/model_metadata.json`
- `artifacts/production_model_report.json`
- `models/churn_xgboost_pipeline.pkl`
- `models/churn_decision_threshold.pkl`
