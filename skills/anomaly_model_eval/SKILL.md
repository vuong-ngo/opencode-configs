---
name: anomaly-model-eval
description: >
  Evaluate and benchmark anomaly/novelty detection models, specifically
  Isolation Forest and One-Class SVM. This skill provides evaluation metrics,
  comparison guidelines, Python code templates, visualization patterns, and
  hyperparameter tuning strategies. Activate when the user asks about
  evaluating, comparing, or tuning IForest or OneClassSVM models.
---

# Anomaly Detection Model Evaluation Skill

Comprehensive guide for evaluating and comparing **Isolation Forest (IForest)**
and **One-Class SVM (OCSVM)** models for anomaly and novelty detection tasks.

---

## Scope

| Aspect           | Coverage                                    |
| ---------------- | ------------------------------------------- |
| Models           | Isolation Forest, One-Class SVM             |
| Tasks            | Anomaly Detection, Novelty Detection        |
| Data types       | Any (tabular, time-series, embeddings, etc) |
| Library          | scikit-learn (primary)                      |
| Evaluation focus | Post-training model assessment              |

---

## Workflow

### Step 1: Understand the Task Context

Before evaluating, clarify with the user:

1. **Anomaly vs Novelty Detection**
   - **Anomaly Detection**: Training data contains both normal and anomalous
     samples. Goal: identify the anomalous ones.
   - **Novelty Detection**: Training data contains ONLY normal samples. Goal:
     detect new, unseen anomalies at inference time.
2. **Label availability**
   - **Labeled data** (ground truth available) → use supervised metrics
     (Precision, Recall, F1, AUC)
   - **Unlabeled data** (no ground truth) → use unsupervised evaluation
     strategies (score distributions, silhouette analysis, domain expert review)
3. **Class imbalance ratio** — anomaly detection datasets are typically heavily
   imbalanced (0.1%–10% anomalies)

### Step 2: Select Evaluation Metrics

Refer to [references/metrics.md](references/metrics.md) for detailed metric
descriptions and when to use each one.

**Key metrics for anomaly detection:**

| Metric              | When to Use                                | Priority  |
| ------------------- | ------------------------------------------ | --------- |
| **AUC-PR**          | Imbalanced data (primary metric)           | ⭐⭐⭐   |
| **AUC-ROC**         | General discrimination ability             | ⭐⭐⭐   |
| **Precision@k**     | When top-k anomalies matter                | ⭐⭐     |
| **Recall@k**        | When catching all anomalies is critical    | ⭐⭐     |
| **F1-Score**        | Balance between precision and recall       | ⭐⭐     |
| **Average Precision** | Summary of precision-recall curve        | ⭐⭐⭐   |
| **MCC**             | Best single metric for imbalanced data     | ⭐⭐     |

**Important rules:**

- ❌ Do NOT use **Accuracy** as the primary metric — it is misleading for
  imbalanced anomaly detection datasets
- ✅ Always report **AUC-PR** alongside **AUC-ROC** for imbalanced data
- ✅ Report **Precision, Recall, F1** at the chosen decision threshold
- ✅ Include **confusion matrix** for interpretability

### Step 3: Compare IForest vs One-Class SVM

Refer to
[references/model-comparison.md](references/model-comparison.md) for a detailed
benchmark table and decision guide.

### Step 4: Evaluate Using Code Templates

Refer to [references/evaluation-code.md](references/evaluation-code.md) for
ready-to-use Python evaluation templates.

### Step 5: Visualize Results

Refer to [references/visualization.md](references/visualization.md) for
plotting templates covering ROC curves, PR curves, score distributions,
confusion matrices, and comparison charts.

### Step 6: Tune Hyperparameters

Refer to [references/hyperparameter-tuning.md](references/hyperparameter-tuning.md)
for tuning strategies for both IForest and One-Class SVM.

---

## Quick Decision Guide

```
Is ground truth (labels) available?
├── YES → Use supervised metrics (Step 2)
│   ├── Data heavily imbalanced? → Prioritize AUC-PR, Average Precision
│   └── Balanced? → AUC-ROC, F1-Score are fine
└── NO → Use unsupervised evaluation
    ├── Analyze anomaly score distributions
    ├── Use domain expert feedback on top-k anomalies
    └── Compare model agreement (IForest vs OCSVM consensus)
```

---

## Checklist Before Completion

- [ ] Task type identified (anomaly vs novelty detection)
- [ ] Label availability confirmed
- [ ] Appropriate metrics selected (AUC-PR for imbalanced data)
- [ ] Both models evaluated with same data splits
- [ ] Confusion matrix generated at chosen threshold
- [ ] Visualization of score distributions provided
- [ ] Hyperparameter impact documented
- [ ] Clear recommendation given (which model, which threshold)

---

## Important Notes

- **Always ask the user** about their contamination rate (expected anomaly
  ratio) before setting `contamination` parameter
- **Normalize/scale features** before using One-Class SVM — it is sensitive to
  feature scale. IForest does not require scaling.
- When comparing models, **use the same train/test split and preprocessing** to
  ensure a fair comparison
- For production deployment, recommend **monitoring model drift** and
  periodically re-evaluating performance
