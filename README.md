# Lending Club Credit Risk Modeling (PD₀)

Leakage-aware probability of default (PD₀) modeling using Lending Club loan data with chronological train-test splitting and structured preprocessing pipelines.

---

## 📌 Project Objective

The goal of this project is to estimate the **probability of default at origination (PD₀)**:

PD₀ = P(loan defaults at any point in the future | information available at issuance)

All predictors are restricted to information available at or before origination to prevent temporal leakage.

---

## 📂 Dataset

The dataset and feature descriptions can be downloaded from Kaggle:

[Lending Club Loan Data (CSV) – Kaggle](https://www.kaggle.com/datasets/adarshsng/lending-club-loan-data-csv)

---

## 📓 Main Notebook

The full project workflow — including data preparation, leakage control, feature engineering, preprocessing pipelines, and modeling — is implemented in:

**`pd0-credit-risk-model.ipynb`**

This notebook contains the complete end-to-end modeling pipeline.

---

## 📘 Technical Documentation

The full technical documentation of the PD₀ modeling framework is available in:

docs/project_documentation.md

---

## ⚙️ Modeling Framework

### 1. Target Construction
- Default (1): `Charged Off`, `Default`
- Non-default (0): `Fully Paid`
- Non-terminal statuses excluded
- Loans not meeting credit policy excluded

---

### 2. Temporal Validation Strategy

The dataset is split chronologically using `issue_d` (loan issuance date):

- Training set: earlier loan vintages   
- Test set: later loan vintages  

This mirrors real-world deployment and prevents forward-looking bias.

---

### 3. Feature Engineering

- Regional feature engineered from `addr_state`
- Credit history age derived from `earliest_cr_line`
- Joint loans excluded
- Post-issuance and intervention variables removed to prevent leakage

---

### 4. Preprocessing Pipelines

- Ordinal encoding for ordered categorical features
- One-hot encoding for nominal categorical features
- Median imputation with missing indicators for bureau variables
- Log transformation applied to right-skewed features
- Standardization of numerical variables

All transformations are implemented using structured `sklearn` pipelines.

---

## 📊 Current Status

- Data preparation completed
- Chronological train-test split implemented
- Structured preprocessing pipelines implemented
- Ready for baseline and advanced model training