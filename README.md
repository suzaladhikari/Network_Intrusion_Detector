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