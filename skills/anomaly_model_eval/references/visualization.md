# Visualization Templates

Ready-to-use plotting templates for anomaly detection model evaluation.

All templates use matplotlib and seaborn. Import block:

```python
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
from sklearn.metrics import (
    confusion_matrix, roc_curve, precision_recall_curve,
    roc_auc_score, average_precision_score,
    ConfusionMatrixDisplay,
)

# Style configuration
plt.style.use("seaborn-v0_8-whitegrid")
sns.set_palette("husl")
FIGSIZE = (10, 6)
COLORS = {"iforest": "#2196F3", "ocsvm": "#FF5722", "baseline": "#9E9E9E"}
```

---

## 1. ROC Curve Comparison

```python
def plot_roc_comparison(y_true, scores_dict, save_path=None):
    """
    Plot ROC curves for multiple models.
    
    Args:
        y_true: Binary labels (1=anomaly, 0=normal)
        scores_dict: {"Model Name": anomaly_scores, ...}
        save_path: Optional path to save figure
    """
    fig, ax = plt.subplots(figsize=FIGSIZE)
    
    for name, scores in scores_dict.items():
        fpr, tpr, _ = roc_curve(y_true, scores)
        auc = roc_auc_score(y_true, scores)
        ax.plot(fpr, tpr, linewidth=2, label=f"{name} (AUC = {auc:.4f})")
    
    # Random baseline
    ax.plot([0, 1], [0, 1], "k--", linewidth=1, alpha=0.5, label="Random (AUC = 0.5)")
    
    ax.set_xlabel("False Positive Rate", fontsize=12)
    ax.set_ylabel("True Positive Rate", fontsize=12)
    ax.set_title("ROC Curve Comparison", fontsize=14, fontweight="bold")
    ax.legend(fontsize=11, loc="lower right")
    ax.set_xlim([-0.02, 1.02])
    ax.set_ylim([-0.02, 1.02])
    ax.grid(True, alpha=0.3)
    
    plt.tight_layout()
    if save_path:
        plt.savefig(save_path, dpi=150, bbox_inches="tight")
    plt.show()


# Usage:
# plot_roc_comparison(y_test, {
#     "Isolation Forest": if_scores,
#     "One-Class SVM": svm_scores,
# })
```

---

## 2. Precision-Recall Curve Comparison

```python
def plot_pr_comparison(y_true, scores_dict, save_path=None):
    """
    Plot Precision-Recall curves for multiple models.
    More informative than ROC for imbalanced datasets.
    """
    fig, ax = plt.subplots(figsize=FIGSIZE)
    
    baseline = y_true.sum() / len(y_true)  # Anomaly ratio
    
    for name, scores in scores_dict.items():
        precision, recall, _ = precision_recall_curve(y_true, scores)
        ap = average_precision_score(y_true, scores)
        ax.plot(recall, precision, linewidth=2, label=f"{name} (AP = {ap:.4f})")
    
    # Baseline (random classifier)
    ax.axhline(y=baseline, color="k", linestyle="--", linewidth=1, alpha=0.5,
               label=f"Baseline ({baseline:.4f})")
    
    ax.set_xlabel("Recall", fontsize=12)
    ax.set_ylabel("Precision", fontsize=12)
    ax.set_title("Precision-Recall Curve Comparison", fontsize=14, fontweight="bold")
    ax.legend(fontsize=11, loc="upper right")
    ax.set_xlim([-0.02, 1.02])
    ax.set_ylim([-0.02, 1.02])
    ax.grid(True, alpha=0.3)
    
    plt.tight_layout()
    if save_path:
        plt.savefig(save_path, dpi=150, bbox_inches="tight")
    plt.show()
```

---

## 3. Confusion Matrix

```python
def plot_confusion_matrices(y_true, predictions_dict, save_path=None):
    """
    Plot confusion matrices side by side for multiple models.
    
    Args:
        y_true: Binary labels (1=anomaly, 0=normal)
        predictions_dict: {"Model Name": y_pred_binary, ...}
    """
    n_models = len(predictions_dict)
    fig, axes = plt.subplots(1, n_models, figsize=(6 * n_models, 5))
    
    if n_models == 1:
        axes = [axes]
    
    for ax, (name, y_pred) in zip(axes, predictions_dict.items()):
        cm = confusion_matrix(y_true, y_pred)
        
        sns.heatmap(
            cm, annot=True, fmt="d", cmap="Blues",
            xticklabels=["Normal", "Anomaly"],
            yticklabels=["Normal", "Anomaly"],
            ax=ax, cbar=False,
            annot_kws={"size": 14},
        )
        ax.set_xlabel("Predicted", fontsize=12)
        ax.set_ylabel("Actual", fontsize=12)
        ax.set_title(f"{name}", fontsize=13, fontweight="bold")
    
    fig.suptitle("Confusion Matrices", fontsize=15, fontweight="bold", y=1.02)
    plt.tight_layout()
    if save_path:
        plt.savefig(save_path, dpi=150, bbox_inches="tight")
    plt.show()


# Usage:
# plot_confusion_matrices(y_test, {
#     "Isolation Forest": if_pred,
#     "One-Class SVM": svm_pred,
# })
```

---

## 4. Anomaly Score Distribution

```python
def plot_score_distribution(scores_dict, y_true=None, save_path=None):
    """
    Plot anomaly score distributions.
    If y_true is provided, shows separate distributions for normal and anomaly.
    
    Args:
        scores_dict: {"Model Name": scores, ...}
        y_true: Optional binary labels (1=anomaly, 0=normal)
    """
    n_models = len(scores_dict)
    fig, axes = plt.subplots(1, n_models, figsize=(7 * n_models, 5))
    
    if n_models == 1:
        axes = [axes]
    
    for ax, (name, scores) in zip(axes, scores_dict.items()):
        if y_true is not None:
            normal_scores = scores[y_true == 0]
            anomaly_scores = scores[y_true == 1]
            
            ax.hist(normal_scores, bins=50, alpha=0.6, density=True,
                    color="#4CAF50", label=f"Normal (n={len(normal_scores)})")
            ax.hist(anomaly_scores, bins=50, alpha=0.6, density=True,
                    color="#F44336", label=f"Anomaly (n={len(anomaly_scores)})")
            ax.legend(fontsize=10)
        else:
            ax.hist(scores, bins=50, alpha=0.7, density=True, color="#2196F3")
        
        ax.set_xlabel("Anomaly Score", fontsize=12)
        ax.set_ylabel("Density", fontsize=12)
        ax.set_title(f"{name}", fontsize=13, fontweight="bold")
        ax.grid(True, alpha=0.3)
    
    fig.suptitle("Anomaly Score Distributions", fontsize=15, fontweight="bold", y=1.02)
    plt.tight_layout()
    if save_path:
        plt.savefig(save_path, dpi=150, bbox_inches="tight")
    plt.show()
```

---

## 5. Metrics Comparison Bar Chart

```python
def plot_metrics_comparison(metrics_dict, save_path=None):
    """
    Bar chart comparing metrics between models.
    
    Args:
        metrics_dict: {
            "Model Name": {"auc_pr": 0.9, "auc_roc": 0.95, ...},
            ...
        }
    """
    metric_names = ["auc_pr", "auc_roc", "precision", "recall", "f1", "mcc"]
    display_names = ["AUC-PR", "AUC-ROC", "Precision", "Recall", "F1", "MCC"]
    
    x = np.arange(len(metric_names))
    width = 0.8 / len(metrics_dict)
    
    fig, ax = plt.subplots(figsize=(12, 6))
    
    for i, (name, metrics) in enumerate(metrics_dict.items()):
        values = [metrics.get(m, 0) for m in metric_names]
        offset = (i - len(metrics_dict) / 2 + 0.5) * width
        bars = ax.bar(x + offset, values, width, label=name, alpha=0.85)
        
        # Value labels on bars
        for bar, val in zip(bars, values):
            ax.text(
                bar.get_x() + bar.get_width() / 2, bar.get_height() + 0.01,
                f"{val:.3f}", ha="center", va="bottom", fontsize=9,
            )
    
    ax.set_xticks(x)
    ax.set_xticklabels(display_names, fontsize=11)
    ax.set_ylabel("Score", fontsize=12)
    ax.set_title("Model Comparison", fontsize=14, fontweight="bold")
    ax.legend(fontsize=11)
    ax.set_ylim(0, 1.15)
    ax.grid(True, axis="y", alpha=0.3)
    
    plt.tight_layout()
    if save_path:
        plt.savefig(save_path, dpi=150, bbox_inches="tight")
    plt.show()


# Usage:
# plot_metrics_comparison({
#     "Isolation Forest": results["iforest"],
#     "One-Class SVM": results["ocsvm"],
# })
```

---

## 6. Threshold vs Metric Curve

```python
def plot_threshold_analysis(y_true, scores, model_name="Model", save_path=None):
    """
    Plot how precision, recall, F1 change with anomaly score threshold.
    Helps choose the optimal operating point.
    """
    thresholds = np.percentile(scores, np.arange(1, 100, 0.5))
    precisions, recalls, f1s = [], [], []
    
    for t in thresholds:
        y_pred = (scores >= t).astype(int)
        precisions.append(precision_score(y_true, y_pred, zero_division=0))
        recalls.append(recall_score(y_true, y_pred, zero_division=0))
        f1s.append(f1_score(y_true, y_pred, zero_division=0))
    
    fig, ax = plt.subplots(figsize=FIGSIZE)
    ax.plot(thresholds, precisions, label="Precision", linewidth=2, color="#4CAF50")
    ax.plot(thresholds, recalls, label="Recall", linewidth=2, color="#F44336")
    ax.plot(thresholds, f1s, label="F1-Score", linewidth=2, color="#2196F3")
    
    # Mark optimal F1 point
    best_idx = np.argmax(f1s)
    ax.axvline(x=thresholds[best_idx], color="gray", linestyle="--", alpha=0.5)
    ax.scatter([thresholds[best_idx]], [f1s[best_idx]], color="#2196F3",
               s=100, zorder=5, label=f"Best F1 = {f1s[best_idx]:.3f}")
    
    ax.set_xlabel("Anomaly Score Threshold", fontsize=12)
    ax.set_ylabel("Metric Value", fontsize=12)
    ax.set_title(f"Threshold Analysis — {model_name}", fontsize=14, fontweight="bold")
    ax.legend(fontsize=11)
    ax.grid(True, alpha=0.3)
    
    plt.tight_layout()
    if save_path:
        plt.savefig(save_path, dpi=150, bbox_inches="tight")
    plt.show()
```

---

## 7. Model Agreement Heatmap

```python
def plot_model_agreement(if_pred, svm_pred, y_true=None, save_path=None):
    """
    Visualize agreement/disagreement between IForest and OCSVM predictions.
    """
    agreement = pd.DataFrame({
        "IForest": if_pred,
        "OCSVM": svm_pred,
    })
    
    if y_true is not None:
        agreement["Ground Truth"] = y_true
    
    # Create agreement categories
    categories = []
    for _, row in agreement.iterrows():
        if row["IForest"] == 1 and row["OCSVM"] == 1:
            categories.append("Both: Anomaly")
        elif row["IForest"] == 0 and row["OCSVM"] == 0:
            categories.append("Both: Normal")
        elif row["IForest"] == 1:
            categories.append("Only IForest: Anomaly")
        else:
            categories.append("Only OCSVM: Anomaly")
    
    agreement["Agreement"] = categories
    
    # Count and plot
    counts = agreement["Agreement"].value_counts()
    
    fig, axes = plt.subplots(1, 2, figsize=(14, 5))
    
    # Pie chart
    colors = ["#F44336", "#4CAF50", "#FF9800", "#2196F3"]
    axes[0].pie(counts.values, labels=counts.index, autopct="%1.1f%%",
                colors=colors[:len(counts)], startangle=90)
    axes[0].set_title("Model Agreement Distribution", fontsize=13, fontweight="bold")
    
    # Cross-tabulation heatmap
    ct = pd.crosstab(
        pd.Series(if_pred, name="IForest"),
        pd.Series(svm_pred, name="OCSVM"),
    )
    ct.index = ["Normal", "Anomaly"]
    ct.columns = ["Normal", "Anomaly"]
    
    sns.heatmap(ct, annot=True, fmt="d", cmap="YlOrRd", ax=axes[1],
                cbar=False, annot_kws={"size": 14})
    axes[1].set_title("Prediction Cross-Tab", fontsize=13, fontweight="bold")
    axes[1].set_xlabel("OCSVM Prediction", fontsize=12)
    axes[1].set_ylabel("IForest Prediction", fontsize=12)
    
    plt.tight_layout()
    if save_path:
        plt.savefig(save_path, dpi=150, bbox_inches="tight")
    plt.show()
    
    # Print agreement stats
    total = len(agreement)
    agree = len(agreement[agreement["IForest"] == agreement["OCSVM"]])
    print(f"\nModel Agreement: {agree}/{total} ({agree/total*100:.1f}%)")
    
    if y_true is not None:
        # Accuracy of consensus predictions
        consensus_mask = agreement["IForest"] == agreement["OCSVM"]
        consensus_correct = (
            agreement.loc[consensus_mask, "IForest"] ==
            agreement.loc[consensus_mask, "Ground Truth"]
        ).sum()
        print(f"Consensus accuracy: {consensus_correct}/{consensus_mask.sum()} "
              f"({consensus_correct/consensus_mask.sum()*100:.1f}%)")
```

---

## 8. Complete Visualization Suite

```python
def generate_full_report(y_true, if_scores, svm_scores, if_pred, svm_pred, save_dir=None):
    """
    Generate all visualization plots for a complete evaluation report.
    
    Args:
        y_true: Binary labels (1=anomaly, 0=normal)
        if_scores: IForest anomaly scores (higher = more anomalous)
        svm_scores: OCSVM anomaly scores (higher = more anomalous)
        if_pred: IForest binary predictions
        svm_pred: OCSVM binary predictions
        save_dir: Optional directory to save all plots
    """
    import os
    
    def path(name):
        return os.path.join(save_dir, name) if save_dir else None
    
    if save_dir:
        os.makedirs(save_dir, exist_ok=True)
    
    scores_dict = {"Isolation Forest": if_scores, "One-Class SVM": svm_scores}
    
    print("📊 Generating evaluation report...\n")
    
    # 1. ROC Curves
    plot_roc_comparison(y_true, scores_dict, save_path=path("roc_comparison.png"))
    
    # 2. PR Curves
    plot_pr_comparison(y_true, scores_dict, save_path=path("pr_comparison.png"))
    
    # 3. Confusion Matrices
    plot_confusion_matrices(
        y_true, {"Isolation Forest": if_pred, "One-Class SVM": svm_pred},
        save_path=path("confusion_matrices.png"),
    )
    
    # 4. Score Distributions
    plot_score_distribution(scores_dict, y_true, save_path=path("score_distributions.png"))
    
    # 5. Threshold Analysis
    plot_threshold_analysis(y_true, if_scores, "Isolation Forest",
                           save_path=path("threshold_iforest.png"))
    plot_threshold_analysis(y_true, svm_scores, "One-Class SVM",
                           save_path=path("threshold_ocsvm.png"))
    
    # 6. Model Agreement
    plot_model_agreement(if_pred, svm_pred, y_true,
                        save_path=path("model_agreement.png"))
    
    print("✅ Report generation complete!")
    if save_dir:
        print(f"   Plots saved to: {save_dir}/")
```
