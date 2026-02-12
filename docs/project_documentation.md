# Feature Categorization & Modeling Scope (PD₀)

This project models **PD₀ (Probability of Default at origination)**:

PD0 = P(loan defaults at any point in the future | information available at issuance `issue_d`)

That means **every predictor must be information that was available at (or before) loan issuance**.  
Any feature that reflects repayment behavior, interventions (hardship/payment plans), recoveries, or "current" credit state **after issuance** introduces **temporal leakage** and is excluded from the PD₀ feature set.

Below is a **complete, exhaustive clustering of all 145 columns** present in the dataset, with a clear decision for each category: **KEEP** (PD₀-safe) or **DISCARD** (leaky or unsuitable).  

All features are listed exactly as they appear in the CSV.

---

**1) Identifiers & platform metadata (DISCARD)**

These fields are used for tracking, URLs, and internal bookkeeping. They are not meaningful predictors of borrower risk.

- `id`
- `member_id`
- `url`
- `policy_code`

**Decision:** **DISCARD** (non-predictive identifiers / metadata).

---

**2) Origination timing anchor (KEEP for splitting, usually DISCARD as predictor)**

`issue_d` is the month the loan was **issued** (the contract becomes active). In PD₀ modeling, it defines the conditioning time $t$ and is essential for **time-based train/test splits** (e.g., train on earlier cohorts, test on later cohorts) to reflect realistic deployment.

- `issue_d`

**Decision:**  
- **KEEP** for splitting data.  
- Usually **DISCARD as a predictor**.

---

**3) Loan contract & pricing at origination (KEEP)**

These are the contractual terms known at listing/issuance and therefore valid PD₀ predictors. They describe the borrower’s repayment burden (e.g., `installment`), the price of risk (`int_rate`), and basic loan purpose/context.

- `term`
- `int_rate`
- `installment`
- `disbursement_method`
- `initial_list_status`
- `application_type` 
- `purpose`
- `title`
- `desc`

**Decision:** 
- **KEEP** (origination-known, PD₀-safe). 
- **USE** `application_type` temporarily to only keep individaul loans and then **DISCARD** it (more details are provided below).

---

**4) Loan amount & funding amounts (MIXED: KEEP one, DISCARD redundant bookkeeping)**

These fields describe loan size. For risk modeling, the “amount actually issued” is the cleanest measure of exposure at origination.

- `funded_amnt` — total amount actually **issued** for the loan (preferred exposure proxy).
- `loan_amnt` — listed/approved amount; often very close to `funded_amnt` but can differ.
- `funded_amnt_inv` — investor-funded portion of `funded_amnt` (operational bookkeeping).

**Decision:**  
- **KEEP:** `funded_amnt`.  
- **DISCARD (recommended):** `loan_amnt`, `funded_amnt_inv`.

---

**5) Borrower profile: income, employment, housing, verification (KEEP)**

Application-time borrower attributes used in underwriting. These are standard PD predictors and do not depend on post-issuance behavior.

- `annual_inc`
- `emp_length`
- `emp_title`
- `home_ownership`
- `verification_status`

**Decision:** **KEEP** (application/origination-known, PD₀-safe).

---

**6) Joint application fields (DISCARDED)**

The following variables are only populated for joint loan applications and describe combined income or verification status of co-borrowers:

- `annual_inc_joint`
- `dti_joint`
- `verification_status_joint`

Joint applications account for approximately 5% of the dataset. To maintain a consistent and interpretable origination-time probability of default (PD₀) framework, the analysis is restricted to individual loan applications. As a result, joint-specific variables are excluded to avoid sparsely populated features and conditional preprocessing logic, while preserving the vast majority of the data and the core risk signal.

---

**7) Debt-to-Income ratio (KEEP)**

DTI is a core affordability metric describing how much of the borrower’s income is already committed to debt obligations at application time. Higher DTI typically implies higher repayment strain and higher PD.

- `dti`

**Decision:** **KEEP** (application-time, PD₀-safe).

---

**8) Geography (Engineered Regional Feature)**

Fine-grained geographic variables such as ZIP code and state can act as proxies for socioeconomic characteristics and introduce unnecessary sparsity and fairness concerns when used directly in credit risk models. To retain high-level regional economic signal while limiting proxy effects and model complexity, raw geographic features are handled selectively.

The ZIP code is discarded, while the state variable is retained only as an intermediate input to engineer a coarse regional indicator. U.S. states will be mapped to one of four Census-style regions (Northeast, Midwest, South, West), capturing broad macroeconomic differences without relying on granular location identifiers.

**Decision:**  
- **KEEP:** temporarily `addr_state`to engineer a new regional feature (4-category region indicator)  
- **DISCARD:** `zip_code`  

---

**9) Bureau history: delinquencies, derogatories, public records (KEEP)**

These features summarize historical credit distress **prior to the application** and are among the most informative drivers of PD in traditional credit-risk modeling.

- `delinq_2yrs`
- `delinq_amnt`
- `mths_since_last_delinq`
- `mths_since_last_major_derog`
- `mths_since_last_record`
- `num_accts_ever_120_pd`
- `num_tl_90g_dpd_24m`
- `pct_tl_nvr_dlq`
- `pub_rec`
- `pub_rec_bankruptcies`
- `tax_liens`

**Decision:** **KEEP** (origination bureau snapshot, PD₀-safe).

---

**10) Bureau inquiry behavior (KEEP)**

Inquiry counts capture recent credit-seeking behavior prior to application and are classic bureau-at-origination predictors.

- `inq_last_6mths`
- `inq_last_12m`
- `inq_fi`

**Decision:** **KEEP** (origination bureau snapshot, PD₀-safe).

---

**11) Credit file age & “oldest account” features (KEEP) vs “recent/recency” features (DISCARD)**

 **A) Credit history depth / age (KEEP)**
 
These describe the age of the borrower’s credit file (anchored to oldest accounts) and are stable, interpretable PD predictors.

- `earliest_cr_line`
- `mo_sin_old_il_acct`
- `mo_sin_old_rev_tl_op`

**Decision:** **KEEP** (credit history depth, PD₀-safe).

 **B) Account recency features (DISCARD)**
 
These measure "months since recent X" or "months since most recent trade". In this dataset, such variables can encode borrower behavior after issuance (and implicitly encode loan age), so these features are excluded.

- `mo_sin_rcnt_rev_tl_op`
- `mo_sin_rcnt_tl`
- `mths_since_rcnt_il`
- `mths_since_recent_bc`
- `mths_since_recent_bc_dlq`
- `mths_since_recent_inq`
- `mths_since_recent_revol_delinq`

**Decision:** **DISCARD** (snapshot/recency ambiguity → potential temporal leakage).

---

**12) Current utilization, balances, and credit state (DISCARD — snapshot-based, leaky)**

These describe **current** revolving/instalment utilization, balances, and limits. They are highly predictive precisely because they reflect the borrower’s condition at the dataset snapshot, which may occur well after issuance and may capture early distress. They are therefore inappropriate for PD₀.

- `all_util`
- `avg_cur_bal`
- `bc_open_to_buy`
- `bc_util`
- `il_util`
- `max_bal_bc`
- `revol_bal`
- `revol_bal_joint`
- `revol_util`
- `tot_cur_bal`
- `tot_hi_cred_lim`
- `total_bal_ex_mort`
- `total_bal_il`
- `total_bc_limit`
- `total_il_high_credit_limit`
- `total_rev_hi_lim`
- `percent_bc_gt_75`

**Decision:** **DISCARD** (post-issuance snapshot state → temporal leakage).

---

**13) Account counts & “open/active/current” activity (DISCARD — snapshot-based, leaky)**

These features describe the borrower’s **currently open or active** accounts, or activity in the "past X months". In this dataset context, they can change after issuance and therefore leak post-origination behavior and implicitly encode time since issuance.

- `open_acc`
- `open_acc_6m`
- `open_act_il`
- `open_il_12m`
- `open_il_24m`
- `open_rv_12m`
- `open_rv_24m`
- `acc_open_past_24mths`
- `acc_now_delinq`
- `num_actv_bc_tl`
- `num_actv_rev_tl`
- `num_bc_sats`
- `num_bc_tl`
- `num_il_tl`
- `num_op_rev_tl`
- `num_rev_accts`
- `num_rev_tl_bal_gt_0`
- `num_sats`
- `num_tl_120dpd_2m`
- `num_tl_30dpd`
- `num_tl_op_past_12m`
- `mort_acc`
- `total_acc`
- `total_cu_tl`
- `collections_12_mths_ex_med`
- `tot_coll_amt`
- `chargeoff_within_12_mths`

**Decision:** **DISCARD** (current/open/recent activity measured after issuance → temporal leakage).

---

**14) Lending Club internal risk ratings (PD₀-safe)**

Lending Club assigns loan grades at origination based on borrower characteristics, loan terms, and credit bureau information available at underwriting. These grades provide a coarse summary of credit risk and are fixed at issuance, making them PD₀-safe from a temporal standpoint. At the same time, they reflect Lending Club’s internal risk assessment and pricing logic rather than raw borrower attributes.

To balance realism and interpretability, the coarse loan grade is retained as a high-level risk indicator, while the finer-grained sub-grade is excluded to avoid redundancy and over-weighting of the platform’s internal scoring.

**Decision:**  
- **KEEP:** `grade`  
- **DISCARD:** `sub_grade`


---

**15) Post-issuance performance & cashflow (DISCARD as features; `loan_status` used to build target)**

These fields describe repayment, remaining principal, and recoveries. They are not available at origination and would directly leak the outcome.

- `loan_status` *(used to construct the target; never a feature)*
- `out_prncp`
- `out_prncp_inv`
- `total_pymnt`
- `total_pymnt_inv`
- `total_rec_prncp`
- `total_rec_int`
- `total_rec_late_fee`
- `last_pymnt_amnt`
- `last_pymnt_d`
- `next_pymnt_d`
- `last_credit_pull_d`
- `recoveries`
- `collection_recovery_fee`

**Decision:**  
- **KEEP AS LABEL SOURCE:** `loan_status`  
- **DISCARD as features:** all variables in this section

---

**16) Payment plans & interventions (DISCARD — hard leakage)**
These reflect post-issuance interventions (hardship/payment plan initiation), which typically occur after financial distress begins. They are strong predictors but invalid for origination PD.

- `pymnt_plan`
- `payment_plan_start_date`
- `deferral_term`

**Decision:** **DISCARD** (post-issuance interventions → temporal leakage).

---

**17) Hardship program fields (DISCARD — hard leakage)**
Hardship variables exist only when a borrower has entered a hardship program. They are downstream of distress and cannot be known at origination.

- `hardship_flag`
- `hardship_type`
- `hardship_reason`
- `hardship_status`
- `hardship_loan_status`
- `hardship_amount`
- `hardship_length`
- `hardship_dpd`
- `hardship_start_date`
- `hardship_end_date`
- `hardship_last_payment_amount`
- `hardship_payoff_balance_amount`

**Decision:** **DISCARD** (post-issuance hardship → temporal leakage).

---

**18) Settlement / debt resolution fields (DISCARD — hard leakage)**
Settlement and debt resolution variables occur after severe delinquency/charge-off. They are purely downstream of default dynamics.

- `debt_settlement_flag`
- `debt_settlement_flag_date`
- `settlement_amount`
- `settlement_date`
- `settlement_percentage`
- `settlement_status`
- `settlement_term`
- `orig_projected_additional_accrued_interest`

**Decision:** **DISCARD** (post-charge-off recovery workflow → temporal leakage).

---

**19) Secondary applicant bureau fields (DISCARDED)**

The following variables describe credit bureau attributes of the secondary applicant in joint loan applications:

- `sec_app_earliest_cr_line`
- `sec_app_inq_last_6mths`
- `sec_app_mort_acc`
- `sec_app_mths_since_last_major_derog`
- `sec_app_num_rev_accts`
- `sec_app_open_acc`
- `sec_app_open_act_il`
- `sec_app_revol_util`
- `sec_app_chargeoff_within_12_mths`
- `sec_app_collections_12_mths_ex_med`

Since the analysis is restricted to **individual loan applications**, all secondary-applicant features are excluded. 

**Decision:**  
- **DISCARD:** all `sec_app_*` variables

---

**Additional Modeling Choice: Free-Text Fields**

Free-text variables (`title`, `desc`, `emp_title`) are excluded from the baseline model. While these fields may contain additional information, they require dedicated text preprocessing and NLP techniques to be used responsibly. Here I will focus only on tabular features. Text-based features are considered a potential extension.

---

**Target construction from `loan_status`** 

For PD₀, we need a **terminal outcome**. `loan_status` is multi-state at snapshot time. A standard approach is:

- **Default (1):** loans with terminal bad outcomes (e.g., `Charged Off`, and `Default` if present)
- **Non-default (0):** `Fully Paid`
- **Exclude (censored):** active or in-progress statuses where the eventual outcome is unknown at snapshot (`Current`, `In Grace Period`, `Late (16-30 days)`, `Late (31-120 days)`, etc.)

This avoids treating “still active” loans as non-default, which would bias the target.

Loans labeled as "Does not meet the credit policy" will be **excluded** from the analysis. These loans were issued under earlier underwriting standards that differ from the credit policy governing the majority of the dataset. Excluding them ensures a more homogeneous origination regime.

---

# Origination-Time Dataset Construction (PD₀)

To transform the raw dataset into a PD₀-consistent modeling table anchored at loan origination, the following steps are applied:

1. **Target construction**
   - A binary default label (`target`) is constructed from `loan_status`.
   - Terminal bad outcomes (`Charged Off`, `Default`) are labeled as default.
   - Terminal good outcomes (`Fully Paid`) are labeled as non-default.
   - All non-terminal or in-progress loan statuses are excluded.
   - Loans issued under outdated credit policies are removed.

2. **Restriction to individual applications**
   - Joint loan applications are excluded using `application_type` (removed after filtering).
   - The dataset is restricted to individual borrowers only.

3. **Geographic feature engineering**
   - The borrower's state (`addr_state`) is mapped to a coarse regional
     indicator (Northeast, Midwest, South, West).
   - The raw state identifier is removed after transformation.

4. **Credit history feature engineering**
   - Origination date (`issue_d`) and earliest credit line date
     (`earliest_cr_line`) are parsed as calendar dates.
   - Credit history length at origination is computed in months as
     the difference between issuance date and earliest credit line.
   - The raw earliest credit line date is removed.

5. **Temporal anchor retention**
   - The issuance date (`issue_d`) is retained solely for time-based
     train/test splitting in a later stage and is not used as a predictor.

---

# Data Preprocessing

## Categorical Feature Handling

The dataset contains **9 categorical features**:

- `term`
- `disbursement_method`
- `initial_list_status`
- `purpose`
- `emp_length`
- `home_ownership`
- `verification_status`
- `grade`
- `region`

Among these, **three variables are ordered categorical features** with a natural ranking that is meaningful for credit risk modeling:

- `term` (36 months < 60 months)
- `grade` (A < B < C < D < E < F < G)
- `emp_length` (shorter employment length < longer length)

These variables are therefore encoded using an **ordinal encoding** that preserves their ordering.

The remaining **six variables are nominal categorical features** with no intrinsic order:

- `disbursement_method`
- `initial_list_status`
- `purpose`
- `home_ownership`
- `verification_status`
- `region`

These variables are encoded using **one-hot encoding**.

#### Missing values in categorical features

Among the categorical variables, only `emp_length` contains missing values.  
Missing employment length does not automatically imply zero employment length; rather, it might reflect non-reporting, self-employment, or irregular employment situations, which may themselves carry risk-relevant information.

Accordingly, missing values in `emp_length` are imputed as a separate **“Missing” category**, allowing the model to learn any associated risk signal instead of discarding or distorting this information.

## Numerical Feature Handling

The dataset contains multiple numerical features derived from loan terms and
credit bureau data.

#### Missing values in numerical features

The following numerical features contain missing values:

- `mths_since_last_delinq`
- `mths_since_last_major_derog`
- `mths_since_last_record`
- `num_accts_ever_120_pd`
- `num_tl_90g_dpd_24m`
- `pct_tl_nvr_dlq`
- `pub_rec_bankruptcies`
- `tax_liens`
- `inq_last_6mths`
- `inq_last_12m`
- `inq_fi`
- `mo_sin_old_il_acct`
- `mo_sin_old_rev_tl_op`

For many bureau-derived variables, missing values typically indicate the
absence of the underlying event (e.g. no prior delinquency, no public record,
or no inquiries recorded under a given reporting schema), rather than an
unknown or erroneous measurement.

Accordingly, missing values in numerical features are handled using **median
imputation**. For features with substantial missingness, **explicit
missing-value indicators** are added to allow the model to learn
risk-relevant information associated with the absence of reported events.
For numerical variables with only negligible missing rates (specifically
`pub_rec_bankruptcies`, `tax_liens`, and `inq_last_6mths`), median imputation
is applied **without** adding a missing-value indicator to avoid introducing
degenerate or uninformative features.

#### Distributional transformations

An inspection of feature distributions reveals that the following numerical
variables are strongly right-skewed and exhibit long right tails:

- `installment`
- `inq_last_12m`
- `inq_fi`
- `mo_sin_old_rev_tl_op`
- `credit_history_months`

To reduce skewness and stabilize scale, a **logarithmic transformation
(`log1p`)** is applied to these variables after imputation and before
standardization.

### Feature scaling

All numerical features are standardized after imputation and (where
applicable) log transformed. Standardization improves numerical stability
and is particularly important for linear models used as baselines in the PD₀
modeling framework.

