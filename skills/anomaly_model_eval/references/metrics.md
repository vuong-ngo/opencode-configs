# Evaluation Metrics for Anomaly Detection

Detailed guide on metrics used to evaluate Isolation Forest and One-Class SVM
models.

---

## Convention: Label Encoding

Throughout this skill, we use the following label convention (scikit-learn
default for outlier detection):

| Label | Meaning  | Description                    |
| ----- | -------- | ------------------------------ |
| `1`   | Normal   | Inlier / normal sample         |
| `-1`  | Anomaly  | Outlier / anomalous sample     |

> **Important**: When computing standard classification metrics (precision,
> recall, F1), you must convert to binary format where `1 = anomaly` (positive
> class) and `0 = normal`. This ensures that "detecting anomalies" is what
> precision/recall refer to.

```python
import numpy as np

def convert_labels(y_sklearn):
    """Convert sklearn outlier labels (-1=anomaly, 1=normal) to binary (1=anomaly, 0=normal)."""
    return np.where(y_sklearn == -1, 1, 0)
```

---

## 1. Threshold-Dependent Metrics

These metrics require a binary decision (anomaly or normal) at a specific
threshold.

### Confusion Matrix

```
                    Predicted
                Normal    Anomaly
Actual  Normal    TN        FP
        Anomaly   FN        TP
```

- **TP (True Positive)**: Correctly detected anomaly
- **FP (False Positive)**: Normal sample falsely flagged as anomaly
- **FN (False Negative)**: Anomaly missed (predicted as normal)
- **TN (True Negative)**: Correctly identified normal sample

### Precision

$$\text{Precision} = \frac{TP}{TP + FP}$$

- Of all samples flagged as anomalies, how many are truly anomalous?
- **High precision** = few false alarms
- Use when **false alarms are costly** (e.g., shutting down production)

### Recall (Sensitivity / True Positive Rate)

$$\text{Recall} = \frac{TP}{TP + FN}$$

- Of all actual anomalies, how many did the model catch?
- **High recall** = catches most anomalies
- Use when **missing anomalies is dangerous** (e.g., fraud detection, security)

### F1-Score

$$F_1 = 2 \cdot \frac{\text{Precision} \times \text{Recall}}{\text{Precision} + \text{Recall}}$$

- Harmonic mean of precision and recall
- Use as a **balanced summary** when both false positives and false negatives
  matter

### F-beta Score

$$F_\beta = (1 + \beta^2) \cdot \frac{\text{Precision} \times \text{Recall}}{\beta^2 \cdot \text{Precision} + \text{Recall}}$$

- `β > 1` → emphasizes recall (catch more anomalies)
- `β < 1` → emphasizes precision (fewer false alarms)
- Common choices: `F2` (recall-focused), `F0.5` (precision-focused)

### Matthews Correlation Coefficient (MCC)

$$MCC = \frac{TP \cdot TN - FP \cdot FN}{\sqrt{(TP+FP)(TP+FN)(TN+FP)(TN+FN)}}$$

- Range: `[-1, +1]`, where `+1` = perfect, `0` = random, `-1` = inverse
- **Best single metric for imbalanced classification**
- Unlike F1, accounts for all four cells of the confusion matrix

---

## 2. Threshold-Independent Metrics

These metrics evaluate the model's ranking/scoring ability across all possible
thresholds.

### AUC-ROC (Area Under ROC Curve)

- Plots **True Positive Rate** vs **False Positive Rate** at all thresholds
- Range: `[0, 1]`, where `0.5` = random, `1.0` = perfect
- **Pros**: Intuitive, widely used
- **Cons**: Can be overly optimistic for highly imbalanced data (because TNs
  dominate the FPR calculation)

### AUC-PR (Area Under Precision-Recall Curve)

- Plots **Precision** vs **Recall** at all thresholds
- Range: `[0, 1]`, baseline = proportion of anomalies in the dataset
- **Pros**: More informative than AUC-ROC for imbalanced datasets
- **Cons**: Harder to interpret; baseline varies with class distribution

### Average Precision (AP)

$$AP = \sum_n (R_n - R_{n-1}) \cdot P_n$$

- Weighted mean of precisions at each recall threshold
- Equivalent to the area under the PR curve (computed via interpolation)
- **Recommended as the primary metric for imbalanced anomaly detection**

---

## 3. Ranking Metrics

### Precision@k

- Precision among the top-k samples ranked by anomaly score
- Useful when you can only investigate a fixed number of alerts

### Recall@k

- Of all true anomalies, what fraction appears in the top-k?
- Useful when you want to know if the top-k captures most anomalies

### Normalized Discounted Cumulative Gain (NDCG@k)

- Measures ranking quality, giving higher weight to anomalies ranked higher
- Less common in anomaly detection but useful for ranking evaluation

---

## 4. Unsupervised Evaluation (No Labels)

When ground truth labels are unavailable:

### Anomaly Score Distribution Analysis

- Plot the distribution of anomaly scores
- Look for **bimodal distribution** (clear separation = good model)
- Compare distributions between IForest and OCSVM

### Stability Analysis

- Run the model multiple times (IForest is stochastic)
- Measure consistency of predictions across runs
- Metric: **Adjusted Rand Index (ARI)** between runs

### Model Agreement

- Compare predictions between IForest and OCSVM
- High agreement on flagged anomalies → higher confidence
- Disagreements → candidates for expert review

### Domain Expert Validation

- Present top-k anomalies to domain experts
- Calculate **Precision@k** based on expert feedback
- Iterative refinement

---

## 5. Metric Selection Guide

```
What is your priority?
├── Minimize false alarms → Precision, F0.5
├── Catch all anomalies → Recall, F2
├── Balance both → F1, MCC
├── Overall ranking quality → AUC-PR, Average Precision
├── Threshold-independent comparison → AUC-ROC (balanced), AUC-PR (imbalanced)
└── No labels available → Score distribution, Model agreement
```

### Recommended Metric Set for Reporting

Always report this minimum set:

1. **AUC-PR** (primary — handles imbalance)
2. **AUC-ROC** (secondary — overall discrimination)
3. **Precision, Recall, F1** at chosen threshold
4. **Confusion Matrix** at chosen threshold
5. **MCC** (single robust metric)
