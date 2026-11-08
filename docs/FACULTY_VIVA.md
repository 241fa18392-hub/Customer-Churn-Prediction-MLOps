# Customer Churn — Faculty Viva Questions and Answers

1. **What does the model predict?**  
   Whether a Telco customer will churn. `Churn` is the binary target; customer identifiers are excluded from model features.

2. **Why use a preprocessing pipeline?**  
   It applies the same categorical encoding and numerical transformations at training and inference, avoiding inconsistent feature representations.

3. **How did you avoid train/test leakage?**  
   The configured preprocessing is fit on the training split, while evaluation uses a fixed held-out test set of 1,409 customers.

4. **Why is recall the quality-gate metric?**  
   Missing a customer who will churn is costly. The configured minimum recall is 0.70; the observed held-out recall was 0.7219.

5. **Does passing the quality gate promote a model?**  
   No. The lifecycle step separately compares candidate recall with the current Production model. Version 2 tied version 1 and was archived.

6. **What does MLflow track?**  
   The project records training runs, parameters, metrics, model artifacts, and registered versions under `Telco_Churn_Production_Model`.

7. **Which model is selected for Production inference?**  
   Registry version 1 remained Production. The inference stage loaded that model and wrote predictions to `outputs/inference_predictions.csv`.

8. **How are duplicate incoming batches prevented?**  
   The loader hashes `new_churn.csv` and checks `dataset_lineage.json`. The current batch was already recorded, so it was validated but not appended.

9. **How does the API handle invalid input?**  
   Pydantic validates typed requests and the API returns structured validation errors; the live health and prediction endpoints were also checked.

10. **Was Docker/Kubernetes deployment completed in this run?**  
    No. The deployment bundle and local API were verified, but Docker and `kubectl` were unavailable. The final pipeline correctly skipped deployment because the candidate was not promoted.

