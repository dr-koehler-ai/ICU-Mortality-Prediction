# ICU Mortality Prediction

**A learning project using first-day ICU data to predict in-hospital mortality.**

This project explores the WiDS Datathon 2020 dataset and compares a dummy baseline, logistic regression, random forest and XGBoost. The focus is on understanding the workflow and explaining model limitations, rather than maximizing a competition score.

## Research question

Can clinical information recorded during the first day of an ICU stay help distinguish patients who die in hospital from those who survive?

The target is `hospital_death`: **0 = survived, 1 = died in hospital**. The outcome is hospital mortality, not death in the ICU. First-day summaries would not be available at admission; this is not an admission-time prediction model.

## Data

- Source: [WiDS Datathon 2020 on Kaggle](https://www.kaggle.com/c/widsdatathon2020/data).
- File: `training_v2.csv`.
- 91,713 encounters and 186 columns, including identifiers and the target.
- 7,915 deaths: approximately 8.6% of encounters.
- 147 hospitals; each patient ID occurs once in this file.
- Selected predictors: demographics, binary clinical indicators and first-day vital signs and laboratory measurements.

Raw data are not included. Obtain the files through Kaggle under the applicable access conditions. IDs, the constant readmission-status column, and precomputed APACHE mortality probabilities are not model inputs. The selected baseline also omits the other APACHE-related fields; these are not all scores.

## Workflow

1. Audit identifiers, target values, duplicates and missingness.
2. Create a stratified 80/20 training/test split with `random_state=42`.
3. Explore training distributions and outcome-group differences.
4. Prepare numeric features with median imputation and scaling; prepare text categories with most-frequent imputation and one-hot encoding.
5. Generate out-of-fold predictions using five-fold stratified cross-validation on the training set. Preprocessing is fitted within each fold.
6. Compare fixed model configurations and explore XGBoost thresholds of 0.5 and 0.3.
7. Evaluate the fixed configurations on the test set and inspect random-forest feature importance.

The split contains 73,370 training encounters and 18,343 test encounters. Initial exploration earlier in development used the full dataset; this limits the independence of the subsequent test evaluation.

## Models

| Model | Main settings |
|---|---|
| Dummy | Predicts the majority class; `strategy="prior"` |
| Logistic regression | Balanced class weights; `max_iter=1000` |
| Random forest | 100 trees; maximum depth 10; minimum leaf size 5; balanced class weights |
| XGBoost | 100 trees; maximum depth 3; learning rate 0.1; no additional class weighting |

No grid search was run for the reported configurations. The models use different weighting settings, so comparisons reflect both the algorithm and its configuration. Scaling is retained in the common preprocessing workflow, although tree models do not require it.

## Results

The following values are recorded from the supplied executed project notebook. The polished notebook has cleared cell outputs and has not been retrained in the editing environment.

### Cross-validation on training data

Precision, recall and F1 below refer to the death class.

| Model | Threshold | ROC-AUC | Average precision | Precision | Recall | F1 |
|---|---:|---:|---:|---:|---:|---:|
| Dummy | 0.5 | 0.500 | 0.086 | 0.000 | 0.000 | 0.000 |
| Logistic regression | 0.5 | 0.841 | 0.424 | 0.245 | 0.732 | 0.367 |
| Random forest | 0.5 | 0.863 | 0.455 | 0.344 | 0.615 | 0.441 |
| XGBoost | 0.5 | 0.872 | 0.497 | 0.716 | 0.240 | 0.359 |
| XGBoost | 0.3 | 0.872 | 0.497 | 0.553 | 0.404 | 0.467 |

### Test data

| Model | Threshold | ROC-AUC | Precision | Recall | F1 |
|---|---:|---:|---:|---:|---:|
| Logistic regression | 0.5 | 0.850 | 0.25 | 0.74 | 0.37 |
| Random forest | 0.5 | 0.867 | 0.34 | 0.64 | 0.44 |
| XGBoost | 0.5 | 0.879 | 0.74 | 0.25 | 0.37 |
| XGBoost | 0.3 | 0.879 | 0.57 | 0.42 | 0.48 |

Test average precision was 0.471 for random forest and 0.527 for XGBoost. 

### Interpretation

XGBoost achieved the highest ROC-AUC and average precision among the evaluated configurations. Logistic regression detected the largest share of deaths at the examined thresholds, but generated more false positives.

Lowering the XGBoost threshold from 0.5 to 0.3 increased cross-validation recall from 0.240 to 0.404 and reduced precision from 0.716 to 0.553. This identified 1,038 additional deaths while producing 1,464 additional false positives. ROC-AUC and average precision did not change because the predicted scores were unchanged.

The random forest assigned its highest impurity-based importance to minimum day-one systolic blood pressure, followed by minimum arterial pH and maximum lactate. This describes how the fitted model used features; it does not establish direction of association or causation.

## Limitations

- Mortality is uncommon, so accuracy alone is misleading: the dummy achieves about 91% accuracy while detecting no deaths.
- Missingness is substantial and differs by outcome. Lactate is missing in approximately 76.9% of survivors and 49.1% of non-survivors in the training subset.
- Statistical comparisons are exploratory and were not adjusted for multiple testing. They were not used as an feature-selection rule.
- Earlier exploration included all data. Neither the random split nor the current cross-validation tests generalization to unseen hospitals.
- First-day event timing and eligibility for a prediction at hour 24 were not reconstructed.
- Threshold 0.3 is an illustration, not an optimized or clinically validated cutoff. Probability calibration, subgroup performance and clinical impact were not evaluated.
- Impurity-based importance can favor features with many possible split points and is affected by correlated variables and preprocessing.
 support. Its scope is intentionally limited to methods explored and discussed during the learning process.
