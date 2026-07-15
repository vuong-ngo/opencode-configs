# IForest vs One-Class SVM: Comparison Guide

Detailed comparison and decision guide for choosing between Isolation Forest and
One-Class SVM.

---

## Algorithm Overview

### Isolation Forest (IForest)

- **Principle**: Anomalies are few and different, so they are easier to isolate
  via random partitioning.
- **How it works**: Builds an ensemble of random trees (Isolation Trees). Each
  tree recursively partitions data with random feature splits. Anomalies require
  fewer splits to isolate → shorter path length → higher anomaly score.
- **Key insight**: Does NOT build a profile of "normal" — instead directly
  isolates anomalies.

### One-Class SVM (OCSVM)

- **Principle**: Learn a decision boundary that encloses the "normal" data in
  feature space.
- **How it works**: Maps data to a high-dimensional space using a kernel
  function, then finds the hyperplane that maximally separates data from the
  origin with the maximum margin.
- **Key insight**: Builds an explicit model of normality — anything outside the
  boundary is anomalous.

---

## Head-to-Head Comparison

| Aspect                      | Isolation Forest                | One-Class SVM                    |
| --------------------------- | ------------------------------- | -------------------------------- |
| **Algorithm type**          | Ensemble (tree-based)           | Kernel-based                     |
| **Training complexity**     | O(n · t · log(ψ))              | O(n² ~ n³)                      |
| **Prediction complexity**   | O(t · log(ψ))                  | O(n_sv · d)                      |
| **Memory usage**            | Low–Medium                      | High (stores support vectors)    |
| **Scalability**             | ✅ Scales well to large data    | ❌ Struggles with n > 10k–50k   |
| **Feature scaling needed**  | ❌ No                           | ✅ Yes (critical)                |
| **Handles mixed features**  | ✅ Naturally handles numeric    | ❌ Requires preprocessing        |
| **High-dimensional data**   | ✅ Good                         | ⚠️ Depends on kernel             |
| **Categorical features**    | ❌ Requires encoding            | ❌ Requires encoding              |
| **Interpretability**        | Medium (feature importance)     | Low (kernel space)               |
| **Stochastic**              | ✅ Yes (random trees)           | ❌ No (deterministic)            |
| **Contamination parameter** | Affects threshold only          | Affects `nu` (training boundary) |
| **Handles local anomalies** | ⚠️ Moderate                    | ⚠️ Moderate                     |
| **Handles global anomalies**| ✅ Excellent                    | ✅ Good                          |
| **Training data purity**    | Works with contaminated data    | Best with clean normal data      |

Where:
- `n` = number of samples
- `t` = number of trees (IForest)
- `ψ` = subsample size (IForest)
- `n_sv` = number of support vectors
- `d` = number of features

---

## Performance Benchmarks (General Guidance)

Based on common benchmark datasets:

| Dataset Characteristics              | IForest | OCSVM | Winner        |
| ------------------------------------ | ------- | ----- | ------------- |
| Large dataset (>100k samples)        | ⭐⭐⭐ | ⭐    | **IForest**   |
| Small dataset (<5k samples)          | ⭐⭐   | ⭐⭐⭐| **OCSVM**     |
| High-dimensional (>50 features)      | ⭐⭐⭐ | ⭐⭐  | **IForest**   |
| Low-dimensional (<10 features)       | ⭐⭐   | ⭐⭐⭐| **OCSVM**     |
| Global/cluster anomalies             | ⭐⭐⭐ | ⭐⭐  | **IForest**   |
| Local/contextual anomalies           | ⭐⭐   | ⭐⭐⭐| **OCSVM**     |
| Clean training data (novelty)        | ⭐⭐   | ⭐⭐⭐| **OCSVM**     |
| Contaminated training data (anomaly) | ⭐⭐⭐ | ⭐    | **IForest**   |
| Real-time inference needed           | ⭐⭐⭐ | ⭐⭐  | **IForest**   |
| Feature interactions matter          | ⭐⭐⭐ | ⭐⭐  | **IForest**   |

---

## Decision Flowchart

```
Start
│
├── Dataset size > 50k samples?
│   ├── YES → Prefer IForest (OCSVM too slow to train)
│   └── NO ─┐
│            │
├── Is training data clean (no anomalies)?
│   ├── YES → Novelty detection → OCSVM may perform better
│   └── NO → Anomaly detection → IForest handles contamination better
│            │
├── Number of features > 50?
│   ├── YES → IForest (tree-based handles high dim well)
│   └── NO ─┐
│            │
├── Need real-time inference?
│   ├── YES → IForest (O(log n) vs O(n_sv))
│   └── NO ─┐
│            │
├── Anomaly type?
│   ├── Global / cluster → IForest
│   └── Local / boundary → OCSVM (with RBF kernel)
│            │
└── When in doubt → Train both, compare with AUC-PR
```

---

## When to Use Each Model

### Choose Isolation Forest when:

- ✅ Dataset is large (>10k samples)
- ✅ High-dimensional feature space
- ✅ Training data may contain some anomalies (contaminated)
- ✅ Fast training and inference are needed
- ✅ No feature scaling available/desired
- ✅ Global or cluster-type anomalies
- ✅ Need for feature importance (rough approximation)

### Choose One-Class SVM when:

- ✅ Dataset is small to medium (<10k samples)
- ✅ Low-dimensional feature space
- ✅ Training data is clean (novelty detection scenario)
- ✅ Local or boundary anomalies
- ✅ Non-linear decision boundaries needed (RBF kernel)
- ✅ Deterministic, reproducible results required

### Use Both (Ensemble) when:

- ✅ Neither model alone achieves satisfactory performance
- ✅ You want higher confidence (consensus voting)
- ✅ Different anomaly types coexist
- ✅ You need a robust production system

---

## Ensemble Strategy

Combine IForest and OCSVM for robust detection:

```python
import numpy as np
from sklearn.preprocessing import MinMaxScaler

def ensemble_predict(iforest_scores, ocsvm_scores, method="average", weights=None):
    """
    Combine anomaly scores from IForest and OCSVM.
    
    Args:
        iforest_scores: Anomaly scores from IForest (lower = more anomalous)
        ocsvm_scores: Decision function from OCSVM (lower = more anomalous)
        method: 'average', 'max', 'voting'
        weights: Weights for weighted average [w_iforest, w_ocsvm]
    
    Returns:
        Combined anomaly scores (lower = more anomalous)
    """
    # Normalize both scores to [0, 1] range
    scaler = MinMaxScaler()
    if_norm = scaler.fit_transform(iforest_scores.reshape(-1, 1)).ravel()
    svm_norm = scaler.fit_transform(ocsvm_scores.reshape(-1, 1)).ravel()
    
    if method == "average":
        if weights is None:
            weights = [0.5, 0.5]
        return weights[0] * if_norm + weights[1] * svm_norm
    
    elif method == "max":
        # Most conservative: flag as anomaly if either model says so
        return np.minimum(if_norm, svm_norm)
    
    elif method == "voting":
        # Binary voting: anomaly only if both agree
        if_pred = (if_norm < np.percentile(if_norm, 10)).astype(int)
        svm_pred = (svm_norm < np.percentile(svm_norm, 10)).astype(int)
        return if_pred + svm_pred  # 2 = both agree, 1 = one model, 0 = neither
    
    else:
        raise ValueError(f"Unknown method: {method}")
```
