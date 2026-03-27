# Hierarchical Network Intrusion Detection System
A two-stage cascade classification pipeline for detecting and classifying network attacks using the KDD Cup 99 dataset

# Overview 

This project implements a hierarchical intrusion detection system (IDS) that works in two sequential stages:

Binary Classifier — Determines whether network traffic is malicious or benign
Severity Classifier — For traffic flagged as malicious, predicts the severity level of the attack (Low / Medium / High)

The two models are chained together in a cascade pipeline, where only traffic identified as an attack in Stage 1 is passed into Stage 2. This mirrors how real-world security operations centers (SOCs) triage threats.


# 📊 Dataset
KDD Cup 1999 — a widely-used benchmark dataset for network intrusion detection research.

Contains labeled network connection records with 41 features
Attack types span four categories: DoS, Probe, R2L, and U2R
Significant class imbalance between normal and rare attack types


---
 
## ⚙️ Pipeline
 
### Stage 0 — Feature Engineering & Preprocessing

All preprocessing was done before model training:
 
- **Target columns created:**
  - `is_malicious` — binary flag (0 = normal, 1 = attack)
  - `Severity_Score` — ordinal severity label (0 = Low, 1 = Medium, 2 = High)
 
- **Log transformation** applied to highly skewed features (`np.log1p`) to reduce the effect of outliers
 
- **Low-variance features dropped** (variance < 0.001): `land`, `is_host_login`, `flag_OTH`, `flag_RSTOS0`, `flag_S2` — these showed near-zero correlation with the target column (< 0.03)

- **Highly correlated features removed** (correlation > 0.95): `srv_serror_rate`, `srv_rerror_rate`, `dst_host_serror_rate`, `flag_S0` — to reduce multicollinearity
 
- **New features engineered:**
  - `bytes_ratio = src_bytes / (dst_bytes + 1)` — ratio of outbound to inbound bytes
  - `total_bytes = src_bytes + dst_bytes` — total data transferred
 
---
 
### Stage 1 — Binary Classifier (Malicious vs. Benign)
 
Six models were trained and evaluated. The goal was maximizing **recall** (catching all attacks) while minimizing **false alarm rate**.

| Model | Precision | Recall | F1-Score | False Alarm Rate | ROC-AUC |
|---|---|---|---|---|---|
| Logistic Regression | 0.9597 | 0.9821 | 0.9708 | 3.81% | 0.9931 |
| Linear SVM | 0.9592 | 0.9887 | 0.9737 | 3.88% | 0.9749 |
| Gaussian Naive Bayes | 0.9010 | 0.9291 | 0.9148 | 9.42% | 0.9555 |
| Decision Tree | 0.9830 | 0.9847 | 0.9838 | 1.57% | 0.9845 |
| Random Forest | 0.9892 | 0.9986 | 0.9939 | 1.01% | 0.9943 |
| **XGBoost** ✅ | **0.9987** | **0.9994** | **0.9991** | **0.12%** | **0.9991** |

> **Selected model: XGBoost Classifier**  
> Chosen for its superior recall and lowest false alarm rate. Saved as `binary_classifier.pkl`.
 
**XGBoost Config:**
```python
XGBClassifier(
    learning_rate=0.1,
    n_estimators=435,
    max_depth=6,
    colsample_bytree=0.6,
    reg_lambda=0.1,
    subsample=0.7,
    min_child_weight=2
)
```
 
---

### Stage 2 — Severity Classifier (Low / Medium / High)
 
Only traffic flagged as malicious by Stage 1 is passed here. Severity levels follow this mapping:
 
| Severity | Score | Example Attacks |
|---|---|---|
| **Low** | 0 | `nmap`, `ipsweep`, `portsweep`, `satan` — Reconnaissance |
| **Medium** | 1 | `neptune`, `smurf`, `teardrop`, `back` — DoS attacks |
| **High** | 2 | `rootkit`, `buffer_overflow`, `sqlattack`, `guess_passwd` — Unauthorized access |

 
Six models were evaluated using **Cohen's Kappa**, **MAE**, and per-class **precision/recall** for rare (High severity) attacks.
 
| Model | MAE | Cohen's Kappa | Rare Attack Precision | Rare Attack Recall | Rare Attack F1 | Misclassification | False Alarm |
|---|---|---|---|---|---|---|---|
| Logistic Regression | 0.0074 | 0.9863 | 0.9757 | 0.9901 | 0.9829 | 0.15% | 0.99% |
| Linear SVM | 0.0071 | 0.9866 | 0.9887 | 0.9741 | 0.9814 | 0.07% | 2.59% |
| Gaussian Naive Bayes | 0.0754 | 0.8458 | 0.8656 | 0.8816 | 0.8735 | 0.82% | 11.84% |
| Decision Tree | 0.0153 | 0.9719 | 0.9509 | 0.9322 | 0.9415 | 0.29% | 6.78% |
| Random Forest | 0.0070 | 0.9889 | 0.9961 | 0.9531 | 0.9742 | 0.02% | 4.69% |
| **XGBoost** ✅ | **0.0012** | **0.9981** | **0.9975** | **0.9938** | **0.9957** | **0.01%** | **0.62%** |


> **Selected model: XGBoost Classifier**  
> Best balance between catching rare high-severity attacks and minimizing misclassification. Saved as `severity_model.pkl`.
 
**XGBoost Config:**
```python
XGBClassifier(
    learning_rate=1,
    n_estimators=43,
    max_depth=6,
    colsample_bytree=0.6,
    objective="multi:softmax",
    reg_lambda=0.1,
    subsample=0.7,
    min_child_weight=2
)
```

 
### Stage 3 — Cascade Pipeline (End-to-End)
 
The two models are chained together in a single inference function:


 