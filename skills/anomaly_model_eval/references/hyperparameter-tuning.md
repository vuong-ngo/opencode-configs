# Hyperparameter Tuning Guide

Strategies for tuning Isolation Forest and One-Class SVM hyperparameters.

---

## Isolation Forest Hyperparameters

### Parameter Reference

| Parameter         | Default     | Range / Options         | Impact                          |
| ----------------- | ----------- | ----------------------- | ------------------------------- |
| `n_estimators`    | 100         | 50–500                  | More trees → more stable        |
| `max_samples`     | "auto"(256) | 64–1024 or float(0–1)   | Subsample size per tree         |
| `contamination`   | "auto"      | 0.001–0.5               | Expected anomaly fraction       |
| `max_features`    | 1.0         | 0.1–1.0                 | Feature fraction per tree       |
| `bootstrap`       | False       | True/False              | Sampling with replacement       |
| `random_state`    | None        | int                     | Reproducibility                 |

### Detailed Parameter Guide

#### `n_estimators` (Number of trees)

- **Default**: 100
- **More trees** = more stable scores, less variance between runs
- **Diminishing returns** after ~200–300 trees
- **Recommendation**: Start with 100, increase if scores are unstable

```python
# Test convergence of anomaly scores
n_estimators_range = [50, 100, 200, 300, 500]
for n in n_estimators_range:
    model = IsolationForest(n_estimators=n, random_state=42)
    model.fit(X_train)
    scores = -model.score_samples(X_test)
    auc = average_precision_score(y_test, scores)
    print(f"n_estimators={n:>4}: AUC-PR={auc:.4f}")
```

#### `max_samples` (Subsample size)

- **Default**: "auto" = min(256, n_samples)
- **Smaller values** → faster training, more diverse trees
- **Larger values** → trees see more data, potentially better boundaries
- **Recommendation**: Keep default for most cases; increase for large datasets
  with subtle anomalies

#### `contamination` (Anomaly fraction)

- **Critical parameter** — directly affects the decision threshold
- Does NOT affect the anomaly scores themselves, only the binary prediction
  cutoff
- **"auto"**: Uses `offset` based on the original paper
- **Best practice**: Set based on domain knowledge or estimate from data

```python
# Search for optimal contamination
contamination_range = [0.01, 0.02, 0.05, 0.1, 0.15, 0.2]
for c in contamination_range:
    model = IsolationForest(contamination=c, random_state=42)
    model.fit(X_train)
    pred = np.where(model.predict(X_test) == -1, 1, 0)
    f1 = f1_score(y_test, pred, zero_division=0)
    print(f"contamination={c:.2f}: F1={f1:.4f}, Predicted anomalies={pred.sum()}")
```

#### `max_features` (Feature subsampling)

- **Default**: 1.0 (use all features)
- **Lower values** → more diverse trees, reduces correlation
- **Recommendation**: Try 0.5–0.8 for high-dimensional data (>50 features)

---

## One-Class SVM Hyperparameters

### Parameter Reference

| Parameter | Default  | Range / Options               | Impact                           |
| --------- | -------- | ----------------------------- | -------------------------------- |
| `kernel`  | "rbf"    | "rbf", "linear", "poly", "sigmoid" | Decision boundary shape    |
| `gamma`   | "scale"  | "scale", "auto", or float     | RBF kernel width                 |
| `nu`      | 0.5      | 0.001–0.5                     | Upper bound on anomaly fraction  |
| `coef0`   | 0.0      | float                         | Independent term (poly/sigmoid)  |
| `degree`  | 3        | 2–5                           | Polynomial kernel degree         |
| `tol`     | 1e-3     | 1e-5–1e-2                     | Convergence tolerance            |

### Detailed Parameter Guide

#### `kernel` (Kernel function)

| Kernel    | Best for                          | Pros                      | Cons                    |
| --------- | --------------------------------- | ------------------------- | ----------------------- |
| `rbf`     | Most cases (default)              | Non-linear, flexible      | Need to tune gamma      |
| `linear`  | High-dimensional, many features   | Fast, interpretable       | Only linear boundaries  |
| `poly`    | Feature interactions matter        | Captures interactions     | Slow for high degree    |
| `sigmoid` | Rarely used                       | —                         | Hard to tune            |

**Recommendation**: Start with `rbf`. Use `linear` for >1000 features.

#### `gamma` (RBF kernel width)

- Controls how far the influence of a single training example reaches
- **High gamma** → tight boundary, overfits to training data
- **Low gamma** → loose boundary, underfits
- **"scale"** (default) = `1 / (n_features * X.var())` — good starting point
- **"auto"** = `1 / n_features`

```python
# Gamma sensitivity analysis
from sklearn.preprocessing import StandardScaler

scaler = StandardScaler()
X_train_s = scaler.fit_transform(X_train)
X_test_s = scaler.transform(X_test)

gamma_range = [0.001, 0.01, 0.05, 0.1, "scale", "auto", 0.5, 1.0]
for g in gamma_range:
    model = OneClassSVM(kernel="rbf", gamma=g, nu=0.05)
    model.fit(X_train_s)
    scores = -model.decision_function(X_test_s)
    try:
        auc = average_precision_score(y_test, scores)
    except:
        auc = float("nan")
    pred_anom = np.where(model.predict(X_test_s) == -1, 1, 0).sum()
    print(f"gamma={str(g):>8}: AUC-PR={auc:.4f}, Predicted anomalies={pred_anom}")
```

#### `nu` (Nu parameter)

- **Dual meaning**:
  1. Upper bound on the fraction of training errors (outliers in training data)
  2. Lower bound on the fraction of support vectors
- **Must be in (0, 1]**
- **Rough guide**: Set `nu ≈ contamination` (expected anomaly ratio)
- **Too low** → too few anomalies detected
- **Too high** → too many false positives

```python
nu_range = [0.01, 0.02, 0.05, 0.1, 0.15, 0.2, 0.3]
for nu in nu_range:
    model = OneClassSVM(kernel="rbf", gamma="scale", nu=nu)
    model.fit(X_train_s)
    pred = np.where(model.predict(X_test_s) == -1, 1, 0)
    f1 = f1_score(y_test, pred, zero_division=0)
    n_sv = model.n_support_
    print(f"nu={nu:.2f}: F1={f1:.4f}, Anomalies={pred.sum()}, Support vectors={n_sv}")
```

---

## Tuning Strategies

### Strategy 1: Grid Search (Small parameter space)

```python
from itertools import product

def grid_search_iforest(X_train, X_test, y_test, param_grid):
    """
    Grid search for Isolation Forest.
    
    Args:
        param_grid: dict of parameter lists, e.g.:
            {
                "n_estimators": [100, 200],
                "max_samples": [256, 512],
                "contamination": [0.05, 0.1],
                "max_features": [0.5, 1.0],
            }
    """
    keys = list(param_grid.keys())
    values = list(param_grid.values())
    
    results = []
    for combo in product(*values):
        params = dict(zip(keys, combo))
        params["random_state"] = 42
        
        model = IsolationForest(**params)
        model.fit(X_train)
        
        scores = -model.score_samples(X_test)
        pred = np.where(model.predict(X_test) == -1, 1, 0)
        
        results.append({
            **params,
            "auc_pr": average_precision_score(y_test, scores),
            "auc_roc": roc_auc_score(y_test, scores),
            "f1": f1_score(y_test, pred, zero_division=0),
            "precision": precision_score(y_test, pred, zero_division=0),
            "recall": recall_score(y_test, pred, zero_division=0),
        })
    
    df = pd.DataFrame(results).sort_values("auc_pr", ascending=False)
    print("Top 5 configurations (by AUC-PR):")
    print(df.head().to_string(index=False))
    
    return df


def grid_search_ocsvm(X_train, X_test, y_test, param_grid):
    """
    Grid search for One-Class SVM.
    IMPORTANT: X_train and X_test must be pre-scaled.
    """
    keys = list(param_grid.keys())
    values = list(param_grid.values())
    
    results = []
    for combo in product(*values):
        params = dict(zip(keys, combo))
        
        model = OneClassSVM(**params)
        model.fit(X_train)
        
        scores = -model.decision_function(X_test)
        pred = np.where(model.predict(X_test) == -1, 1, 0)
        
        results.append({
            **params,
            "auc_pr": average_precision_score(y_test, scores),
            "auc_roc": roc_auc_score(y_test, scores),
            "f1": f1_score(y_test, pred, zero_division=0),
            "n_support_vectors": model.n_support_[0] if hasattr(model, 'n_support_') else 0,
        })
    
    df = pd.DataFrame(results).sort_values("auc_pr", ascending=False)
    print("Top 5 configurations (by AUC-PR):")
    print(df.head().to_string(index=False))
    
    return df
```

### Strategy 2: Score-Based Threshold Tuning

When `contamination` / `nu` is unknown, use the anomaly scores directly:

```python
def tune_threshold_by_score(model, X_test, y_test, n_thresholds=100):
    """
    Instead of tuning contamination/nu, find the optimal score threshold.
    This decouples model training from threshold selection.
    """
    scores = get_anomaly_scores(model, X_test)
    
    thresholds = np.linspace(scores.min(), scores.max(), n_thresholds)
    best_f1 = 0
    best_threshold = None
    
    for t in thresholds:
        pred = (scores >= t).astype(int)
        f1 = f1_score(y_test, pred, zero_division=0)
        if f1 > best_f1:
            best_f1 = f1
            best_threshold = t
    
    # Final evaluation at optimal threshold
    y_pred_opt = (scores >= best_threshold).astype(int)
    
    print(f"Optimal threshold: {best_threshold:.6f}")
    print(f"  F1:        {f1_score(y_test, y_pred_opt, zero_division=0):.4f}")
    print(f"  Precision: {precision_score(y_test, y_pred_opt, zero_division=0):.4f}")
    print(f"  Recall:    {recall_score(y_test, y_pred_opt, zero_division=0):.4f}")
    
    return best_threshold, y_pred_opt
```

### Strategy 3: Contamination Estimation

Estimate the contamination rate when it's unknown:

```python
def estimate_contamination(X, method="iqr"):
    """
    Estimate contamination rate from data.
    
    Methods:
        - 'iqr': Interquartile range outlier detection per feature
        - 'zscore': Z-score based detection per feature
    """
    if method == "iqr":
        outlier_mask = np.zeros(len(X), dtype=bool)
        for j in range(X.shape[1]):
            q1, q3 = np.percentile(X[:, j], [25, 75])
            iqr = q3 - q1
            lower = q1 - 1.5 * iqr
            upper = q3 + 1.5 * iqr
            outlier_mask |= (X[:, j] < lower) | (X[:, j] > upper)
        estimated = outlier_mask.mean()
    
    elif method == "zscore":
        from scipy import stats
        z = np.abs(stats.zscore(X, axis=0))
        outlier_mask = (z > 3).any(axis=1)
        estimated = outlier_mask.mean()
    
    else:
        raise ValueError(f"Unknown method: {method}")
    
    # Clamp to reasonable range
    estimated = max(0.01, min(estimated, 0.3))
    
    print(f"Estimated contamination ({method}): {estimated:.4f} ({estimated*100:.1f}%)")
    return estimated
```

---

## Recommended Default Configurations

### Isolation Forest — Good Starting Point

```python
IsolationForest(
    n_estimators=200,           # Enough for stability
    max_samples="auto",         # 256 or n_samples (whichever is smaller)
    contamination="auto",       # Or set based on domain knowledge
    max_features=1.0,           # Use all features
    bootstrap=False,            # Standard isolation
    random_state=42,            # Reproducibility
    n_jobs=-1,                  # Use all CPU cores
)
```

### One-Class SVM — Good Starting Point

```python
OneClassSVM(
    kernel="rbf",               # Non-linear boundary
    gamma="scale",              # Adaptive to feature variance
    nu=0.05,                    # Conservative: expect ~5% anomalies
    tol=1e-3,                   # Default convergence tolerance
)
```

### Feature Scaling for OCSVM

```python
from sklearn.preprocessing import StandardScaler, RobustScaler

# StandardScaler: good when data is roughly Gaussian
scaler = StandardScaler()

# RobustScaler: better when data has outliers in training set
# (uses median and IQR instead of mean and std)
scaler = RobustScaler()

X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled = scaler.transform(X_test)
```

---

## Common Pitfalls

| Pitfall                                  | Solution                                          |
| ---------------------------------------- | ------------------------------------------------- |
| Tuning on test set (data leakage)        | Use validation set or cross-validation             |
| Forgetting to scale for OCSVM            | Always use StandardScaler or RobustScaler          |
| Setting contamination too high           | Start low (1-5%), increase if recall is too low    |
| Using accuracy as tuning metric          | Use AUC-PR or F1-score instead                     |
| Not fixing random_state for IForest      | Always set for reproducible results                |
| Training OCSVM on contaminated data      | Filter obvious outliers first, or use IForest      |
| Ignoring computational cost of OCSVM     | Subsample for large datasets, or prefer IForest    |
