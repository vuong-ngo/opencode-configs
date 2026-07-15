# Evaluation Code Templates

Ready-to-use Python code templates for evaluating Isolation Forest and
One-Class SVM models using scikit-learn.

---

## 1. Complete Evaluation Pipeline

```python
"""
Complete evaluation pipeline for IForest and One-Class SVM.
Requirements: scikit-learn, numpy, pandas, matplotlib, seaborn
"""
import numpy as np
import pandas as pd
from sklearn.ensemble import IsolationForest
from sklearn.svm import OneClassSVM
from sklearn.preprocessing import StandardScaler
from sklearn.model_selection import train_test_split
from sklearn.metrics import (
    precision_score, recall_score, f1_score,
    roc_auc_score, average_precision_score,
    confusion_matrix, classification_report,
    precision_recall_curve, roc_curve,
    matthews_corrcoef
)


# ============================================================
# Helper Functions
# ============================================================

def convert_labels(y_sklearn):
    """Convert sklearn outlier labels (-1=anomaly, 1=normal) to binary (1=anomaly, 0=normal)."""
    return np.where(y_sklearn == -1, 1, 0)


def get_anomaly_scores(model, X):
    """
    Get anomaly scores from a trained model.
    Returns scores where HIGHER = more anomalous (for consistent comparison).
    """
    if isinstance(model, IsolationForest):
        # score_samples returns negative anomaly scores (lower = more anomalous)
        return -model.score_samples(X)
    elif isinstance(model, OneClassSVM):
        # decision_function returns signed distance (lower = more anomalous)
        return -model.decision_function(X)
    else:
        raise ValueError(f"Unsupported model type: {type(model)}")


def evaluate_model(y_true_binary, y_pred_binary, scores, model_name="Model"):
    """
    Compute all evaluation metrics.
    
    Args:
        y_true_binary: Ground truth (1=anomaly, 0=normal)
        y_pred_binary: Predicted labels (1=anomaly, 0=normal)
        scores: Anomaly scores (higher = more anomalous)
        model_name: Name for display
    
    Returns:
        Dictionary of metrics
    """
    metrics = {
        "model": model_name,
        "precision": precision_score(y_true_binary, y_pred_binary, zero_division=0),
        "recall": recall_score(y_true_binary, y_pred_binary, zero_division=0),
        "f1": f1_score(y_true_binary, y_pred_binary, zero_division=0),
        "mcc": matthews_corrcoef(y_true_binary, y_pred_binary),
        "auc_roc": roc_auc_score(y_true_binary, scores),
        "auc_pr": average_precision_score(y_true_binary, scores),
        "confusion_matrix": confusion_matrix(y_true_binary, y_pred_binary),
    }
    
    # Precision@k and Recall@k
    for k_pct in [1, 5, 10]:
        k = max(1, int(len(scores) * k_pct / 100))
        top_k_idx = np.argsort(scores)[-k:]  # Top-k highest scores
        top_k_pred = np.zeros(len(scores))
        top_k_pred[top_k_idx] = 1
        metrics[f"precision@{k_pct}%"] = precision_score(
            y_true_binary, top_k_pred, zero_division=0
        )
        metrics[f"recall@{k_pct}%"] = recall_score(
            y_true_binary, top_k_pred, zero_division=0
        )
    
    return metrics


def print_evaluation_report(metrics):
    """Print a formatted evaluation report."""
    print(f"\n{'='*60}")
    print(f"  Evaluation Report: {metrics['model']}")
    print(f"{'='*60}")
    print(f"  AUC-PR (primary):    {metrics['auc_pr']:.4f}")
    print(f"  AUC-ROC:             {metrics['auc_roc']:.4f}")
    print(f"  Precision:           {metrics['precision']:.4f}")
    print(f"  Recall:              {metrics['recall']:.4f}")
    print(f"  F1-Score:            {metrics['f1']:.4f}")
    print(f"  MCC:                 {metrics['mcc']:.4f}")
    print(f"\n  Precision@1%:        {metrics['precision@1%']:.4f}")
    print(f"  Precision@5%:        {metrics['precision@5%']:.4f}")
    print(f"  Precision@10%:       {metrics['precision@10%']:.4f}")
    print(f"  Recall@1%:           {metrics['recall@1%']:.4f}")
    print(f"  Recall@5%:           {metrics['recall@5%']:.4f}")
    print(f"  Recall@10%:          {metrics['recall@10%']:.4f}")
    print(f"\n  Confusion Matrix:")
    cm = metrics['confusion_matrix']
    print(f"    TN={cm[0][0]:>6}  FP={cm[0][1]:>6}")
    print(f"    FN={cm[1][0]:>6}  TP={cm[1][1]:>6}")
    print(f"{'='*60}\n")


# ============================================================
# Main Evaluation
# ============================================================

def run_evaluation(X_train, X_test, y_test, contamination=0.1):
    """
    Train and evaluate both IForest and One-Class SVM.
    
    Args:
        X_train: Training features (ideally clean for novelty detection)
        X_test: Test features
        y_test: Test labels in binary format (1=anomaly, 0=normal)
        contamination: Expected proportion of anomalies
    
    Returns:
        Dictionary with metrics for both models
    """
    results = {}
    
    # --- Isolation Forest ---
    iforest = IsolationForest(
        n_estimators=100,
        contamination=contamination,
        random_state=42,
        n_jobs=-1,
    )
    iforest.fit(X_train)
    
    if_pred_sklearn = iforest.predict(X_test)
    if_pred = convert_labels(if_pred_sklearn)
    if_scores = get_anomaly_scores(iforest, X_test)
    
    results["iforest"] = evaluate_model(y_test, if_pred, if_scores, "Isolation Forest")
    results["iforest"]["model_obj"] = iforest
    results["iforest"]["scores"] = if_scores
    
    # --- One-Class SVM ---
    # IMPORTANT: Scale features for OCSVM
    scaler = StandardScaler()
    X_train_scaled = scaler.fit_transform(X_train)
    X_test_scaled = scaler.transform(X_test)
    
    ocsvm = OneClassSVM(
        kernel="rbf",
        gamma="scale",
        nu=contamination,  # Upper bound on fraction of outliers
    )
    ocsvm.fit(X_train_scaled)
    
    svm_pred_sklearn = ocsvm.predict(X_test_scaled)
    svm_pred = convert_labels(svm_pred_sklearn)
    svm_scores = get_anomaly_scores(ocsvm, X_test_scaled)
    
    results["ocsvm"] = evaluate_model(y_test, svm_pred, svm_scores, "One-Class SVM")
    results["ocsvm"]["model_obj"] = ocsvm
    results["ocsvm"]["scores"] = svm_scores
    
    # Print reports
    print_evaluation_report(results["iforest"])
    print_evaluation_report(results["ocsvm"])
    
    # Comparison table
    comparison = pd.DataFrame({
        "Metric": ["AUC-PR", "AUC-ROC", "Precision", "Recall", "F1", "MCC"],
        "IForest": [
            results["iforest"]["auc_pr"],
            results["iforest"]["auc_roc"],
            results["iforest"]["precision"],
            results["iforest"]["recall"],
            results["iforest"]["f1"],
            results["iforest"]["mcc"],
        ],
        "OCSVM": [
            results["ocsvm"]["auc_pr"],
            results["ocsvm"]["auc_roc"],
            results["ocsvm"]["precision"],
            results["ocsvm"]["recall"],
            results["ocsvm"]["f1"],
            results["ocsvm"]["mcc"],
        ],
    })
    comparison["Winner"] = comparison.apply(
        lambda row: "IForest" if row["IForest"] > row["OCSVM"]
        else "OCSVM" if row["OCSVM"] > row["IForest"]
        else "Tie",
        axis=1,
    )
    print("\n📊 Model Comparison:")
    print(comparison.to_string(index=False))
    
    return results
```

---

## 2. Evaluation with Pre-Trained Models

When the user already has trained models:

```python
def evaluate_pretrained(model, X_test, y_test_binary, model_name="Model", scaler=None):
    """
    Evaluate a pre-trained IForest or OCSVM model.
    
    Args:
        model: Trained IsolationForest or OneClassSVM
        X_test: Test features
        y_test_binary: Test labels (1=anomaly, 0=normal)
        model_name: Display name
        scaler: StandardScaler if OCSVM was trained on scaled data
    """
    X_eval = scaler.transform(X_test) if scaler is not None else X_test
    
    # Get predictions and scores
    pred_sklearn = model.predict(X_eval)
    pred_binary = convert_labels(pred_sklearn)
    scores = get_anomaly_scores(model, X_eval)
    
    # Evaluate
    metrics = evaluate_model(y_test_binary, pred_binary, scores, model_name)
    print_evaluation_report(metrics)
    
    return metrics
```

---

## 3. Cross-Validated Evaluation

For more robust evaluation:

```python
from sklearn.model_selection import StratifiedKFold

def cross_validate_anomaly(X, y_binary, model_class, model_params, n_splits=5, scaler=None):
    """
    Cross-validated evaluation for anomaly detection models.
    
    Args:
        X: Features
        y_binary: Labels (1=anomaly, 0=normal)
        model_class: IsolationForest or OneClassSVM
        model_params: Dict of model parameters
        n_splits: Number of CV folds
        scaler: StandardScaler class (pass StandardScaler for OCSVM)
    
    Returns:
        DataFrame with per-fold metrics
    """
    skf = StratifiedKFold(n_splits=n_splits, shuffle=True, random_state=42)
    fold_metrics = []
    
    for fold, (train_idx, test_idx) in enumerate(skf.split(X, y_binary), 1):
        X_train, X_test = X[train_idx], X[test_idx]
        y_test = y_binary[test_idx]
        
        # Scale if needed
        if scaler is not None:
            sc = scaler()
            X_train = sc.fit_transform(X_train)
            X_test = sc.transform(X_test)
        
        # Train and predict
        model = model_class(**model_params)
        model.fit(X_train)
        
        pred = convert_labels(model.predict(X_test))
        scores = get_anomaly_scores(model, X_test)
        
        # Compute metrics
        fold_metrics.append({
            "fold": fold,
            "auc_pr": average_precision_score(y_test, scores),
            "auc_roc": roc_auc_score(y_test, scores),
            "precision": precision_score(y_test, pred, zero_division=0),
            "recall": recall_score(y_test, pred, zero_division=0),
            "f1": f1_score(y_test, pred, zero_division=0),
            "mcc": matthews_corrcoef(y_test, pred),
        })
    
    df = pd.DataFrame(fold_metrics)
    
    # Summary
    print(f"\n{'='*60}")
    print(f"  {n_splits}-Fold Cross-Validation Results: {model_class.__name__}")
    print(f"{'='*60}")
    for col in ["auc_pr", "auc_roc", "precision", "recall", "f1", "mcc"]:
        mean = df[col].mean()
        std = df[col].std()
        print(f"  {col:<15}: {mean:.4f} ± {std:.4f}")
    print(f"{'='*60}\n")
    
    return df


# Usage example:
# cv_if = cross_validate_anomaly(
#     X, y_binary,
#     IsolationForest,
#     {"n_estimators": 100, "contamination": 0.05, "random_state": 42},
# )
# cv_svm = cross_validate_anomaly(
#     X, y_binary,
#     OneClassSVM,
#     {"kernel": "rbf", "gamma": "scale", "nu": 0.05},
#     scaler=StandardScaler,
# )
```

---

## 4. Threshold Optimization

Find the optimal decision threshold:

```python
def find_optimal_threshold(y_true, scores, metric="f1"):
    """
    Find the optimal anomaly score threshold.
    
    Args:
        y_true: Binary labels (1=anomaly, 0=normal)
        scores: Anomaly scores (higher = more anomalous)
        metric: 'f1', 'precision', 'recall', 'mcc'
    
    Returns:
        Optimal threshold and corresponding metric value
    """
    thresholds = np.percentile(scores, np.arange(1, 100, 0.5))
    best_score = -1
    best_threshold = None
    
    for t in thresholds:
        y_pred = (scores >= t).astype(int)
        
        if metric == "f1":
            score = f1_score(y_true, y_pred, zero_division=0)
        elif metric == "precision":
            score = precision_score(y_true, y_pred, zero_division=0)
        elif metric == "recall":
            score = recall_score(y_true, y_pred, zero_division=0)
        elif metric == "mcc":
            score = matthews_corrcoef(y_true, y_pred)
        else:
            raise ValueError(f"Unknown metric: {metric}")
        
        if score > best_score:
            best_score = score
            best_threshold = t
    
    print(f"Optimal threshold (max {metric}): {best_threshold:.6f}")
    print(f"  {metric}: {best_score:.4f}")
    
    # Apply optimal threshold
    y_pred_optimal = (scores >= best_threshold).astype(int)
    print(f"  Precision: {precision_score(y_true, y_pred_optimal, zero_division=0):.4f}")
    print(f"  Recall:    {recall_score(y_true, y_pred_optimal, zero_division=0):.4f}")
    print(f"  F1:        {f1_score(y_true, y_pred_optimal, zero_division=0):.4f}")
    
    return best_threshold, best_score
```

---

## 5. Statistical Significance Test

Compare two models statistically:

```python
from scipy import stats

def compare_models_statistical(cv_results_a, cv_results_b, metric="auc_pr", alpha=0.05):
    """
    Paired t-test to determine if model A is significantly better than model B.
    
    Args:
        cv_results_a: CV results DataFrame for model A
        cv_results_b: CV results DataFrame for model B
        metric: Metric column to compare
        alpha: Significance level
    """
    scores_a = cv_results_a[metric].values
    scores_b = cv_results_b[metric].values
    
    # Paired t-test
    t_stat, p_value = stats.ttest_rel(scores_a, scores_b)
    
    mean_a = scores_a.mean()
    mean_b = scores_b.mean()
    diff = mean_a - mean_b
    
    print(f"\n{'='*50}")
    print(f"  Statistical Comparison ({metric})")
    print(f"{'='*50}")
    print(f"  Model A mean: {mean_a:.4f}")
    print(f"  Model B mean: {mean_b:.4f}")
    print(f"  Difference:   {diff:+.4f}")
    print(f"  t-statistic:  {t_stat:.4f}")
    print(f"  p-value:      {p_value:.6f}")
    
    if p_value < alpha:
        winner = "A" if diff > 0 else "B"
        print(f"  Result: Model {winner} is significantly better (p < {alpha})")
    else:
        print(f"  Result: No significant difference (p = {p_value:.4f} > {alpha})")
    print(f"{'='*50}\n")
    
    return t_stat, p_value
```
