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