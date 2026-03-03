# dego-project-team11-TXC
**DEGO 2606 Group Project – Credit Application Governance Analysis**  
MSc Business Analytics | Nova SBE

---

## Team Members & Roles

| Name | Role |
|------|------|
| Inês Monteiro | Data Engineer — data ingestion, cleaning pipeline, repository structure |
| Anh Nguyen | Data Scientist — bias analysis, fairness metrics, statistical testing |
| Estêvão Fernandes | Governance Officer — GDPR mapping, compliance analysis, policy recommendations |
| Jonas Knosp | Product Lead — coordination, README, presentation narrative |

---

## Executive Summary

This report presents the findings of a data governance audit conducted on `raw_credit_applications.json`, a dataset of **502 credit applications** from NovaCred, a fintech company under regulatory scrutiny for potential discriminatory lending practices. The audit was structured around three analytical pillars: **data quality assessment**, **algorithmic bias detection**, and **privacy and GDPR compliance evaluation**.

Key findings are summarised below:

- **31.3% of records (157/502)** have `date_of_birth` stored in a non-standard format (`DD/MM/YYYY` or `YYYY/MM/DD` instead of ISO 8601 `YYYY-MM-DD`), preventing reliable age computation without prior normalisation.
- **22.1% of records** exhibit inconsistent gender encoding, with four distinct string representations (`Male`, `M`, `Female`, `F`) used for a binary categorical field — a critical validity and consistency failure prior to any modelling step.
- A statistically significant gender-based disparity in loan approval rates was identified. Applying the four-fifths rule, the computed **Disparate Impact Ratio of 0.770** falls below the 0.80 threshold, indicating potential unlawful disparate impact on female applicants under EU anti-discrimination frameworks.
- **`zip_code` acts as a proxy for gender** (point-biserial correlation: −0.82), meaning geographic filtering reproduces gender bias even if gender is removed from the model.
- **Multiple categories of PII** — including Social Security Numbers, full names, email addresses, IP addresses, and dates of birth — are stored in plaintext with no evidence of pseudonymisation, encryption, or access controls, in direct conflict with GDPR Articles 5 and 25.
- The dominant rejection mechanism (`algorithm_risk_score`, accounting for **81.7% of denials**) is non-transparent and non-auditable, raising high-risk AI system concerns under the EU AI Act (Annex III, §5(b)).

---

## Repository Structure
```
dego-project-team11-TXC/
├── README.md
├── data/
│   ├── raw_credit_applications.json
│   └── cleaned_credit_applications.csv
├── notebooks/
│   ├── 01-data-quality.ipynb
│   ├── 02-bias-analysis.ipynb
│   └── 03-privacy-demo.ipynb
├── figures/
│   ├── fig1_missing_values.png
│   ├── fig2_distributions.png
│   ├── fig3_date_formats.png
│   ├── fig4_rejection_reasons.png
│   ├── fig5_approval_by_gender.png
│   └── fig6_correlation_heatmap.png
├── src/
│   └── fairness_utils.py
└── presentation/
    └── video_link.md
```

---

## Dataset Overview

**Source file:** `data/raw_credit_applications.json`  
**Format:** Nested JSON (not a flat tabular structure)  
**N:** 502 credit applications  
**Overall approval rate:** 58.2% (292/502)  
**Reference date for age calculations:** 2024-01-15 (inferred from `processing_timestamp`)

### Schema Summary

| Field Path | Type | Description |
|-----------|------|-------------|
| `_id` | String | Application ID |
| `applicant_info.full_name` | String | Applicant full name |
| `applicant_info.email` | String | Email address |
| `applicant_info.ssn` | String | Social Security Number |
| `applicant_info.ip_address` | String | IP address at application time |
| `applicant_info.gender` | String | Gender (inconsistently encoded — see §1.2) |
| `applicant_info.date_of_birth` | String | Date of birth (inconsistent formats — see §1.5) |
| `applicant_info.zip_code` | String | ZIP/postal code |
| `financials.annual_income` | Number | Annual income |
| `financials.credit_history_months` | Integer | Months of credit history |
| `financials.debt_to_income` | Number | Debt-to-income ratio |
| `financials.savings_balance` | Number | Savings balance |
| `spending_behavior[].category` | String | Spending category label |
| `spending_behavior[].amount` | Number | Monthly spending amount |
| `decision.loan_approved` | Boolean | Approval outcome |
| `decision.interest_rate` | Number | Assigned rate (if approved) |
| `decision.approved_amount` | Number | Approved loan amount (if approved) |
| `decision.rejection_reason` | String | Rejection label (if denied) |
| `processing_timestamp` | String | Timestamp of processing (87.6% missing) |
| `loan_purpose` | String | Stated purpose of loan (~90% missing) |

> ⚠️ The dataset contains intentional data quality issues and bias patterns. All issues identified below are documented and quantified in `notebooks/01-data-quality.ipynb` and `notebooks/02-bias-analysis.ipynb`.

---

## 1. Data Quality Analysis

Full methodology and remediation code: `notebooks/01-data-quality.ipynb`

### 1.1 Completeness

| Field | Missing (n) | Missing (%) |
|-------|------------|------------|
| `loan_purpose` | ~450 | ~89.6% |
| `processing_timestamp` | 440 | 87.6% |
| `email` | 7 | 1.4% |
| `date_of_birth` | 5 | 1.0% |
| `ssn` | 5 | 1.0% |
| `annual_income` | 5 | 1.0% |
| `gender` | 3 | 0.6% |

![Figure 1: Missing value heatmap by field](figures/fig1_missing_values.png)  
*Figure 1: `processing_timestamp` and `loan_purpose` are too sparse for analytical use. The near-total absence of `processing_timestamp` (87.6% missing) means NovaCred cannot demonstrate GDPR storage limitation compliance (Art. 5(1)(e)) or produce audit trails required under the EU AI Act (Art. 12).*

### 1.2 Consistency

| Issue | Affected Records (n) | Affected Records (%) |
|-------|---------------------|---------------------|
| Inconsistent gender encoding (`M`/`Male`, `F`/`Female`) | 111 | 22.1% |
| Inconsistent `date_of_birth` formats (see §1.5) | 157 | 31.3% |
| Duplicate `_id` values | 2 | 0.4% |
| Approved loans missing `interest_rate` field | detected | — |

### 1.3 Validity

| Issue | Affected (n) | Detail |
|-------|-------------|--------|
| Negative `credit_history_months` | 2 | Min observed: −10 months. Logically impossible. |
| Negative `savings_balance` | 1 | Cannot be negative under this schema definition. |
| `debt_to_income` > 1.0 | 1 | Total debt exceeds income — an invalid ratio. |
| `annual_income` = 0 | 1 | Likely a missing value recorded as zero rather than null. |
| Invalid email format (`sarah.smith@`) | 1 | Incomplete email stored in `email` field. |

![Figure 2: Distributions of annual_income, credit_history_months, debt_to_income](figures/fig2_distributions.png)  
*Figure 2: Red dashed lines mark invalid values — zero income, negative credit history months, and DTI above 1.0. All three require flagging before downstream modelling.*

### 1.4 Accuracy

| Issue | Detail |
|-------|--------|
| Duplicate `_id` | 2 records share the same application ID. Resolution: retain the record with the most recent `processing_timestamp`; flag the duplicate for human review. |
| Inconsistent gender encoding | The same attribute is represented as `Male`/`M` or `Female`/`F` — an accuracy failure under GDPR Art. 5(1)(d). |

### 1.5 Date Format Inconsistency

The `date_of_birth` field is stored as a plain string with **no enforced format**. Three distinct formats were detected:

| Format | Count (n) | Share (%) | Example |
|--------|-----------|-----------|---------|
| `YYYY-MM-DD` (ISO 8601 — standard) | 340 | 67.7% | `1985-06-15` |
| `DD/MM/YYYY` | 101 | 20.1% | `15/06/1985` |
| `YYYY/MM/DD` | 56 | 11.2% | `1985/06/15` |
| Empty string | 5 | 1.0% | `""` |

**Total records with non-standard or missing date format: 162 (32.3%)**

![Figure 3: date_of_birth format inconsistency](figures/fig3_date_formats.png)  
*Figure 3: Three distinct date formats co-exist in the same field. Any age-based calculation performed without prior normalisation will silently produce incorrect results for 31.3% of records. One record (`app_183`) uses US `MM/DD/YYYY` format — ambiguous with `DD/MM/YYYY` and requiring manual resolution.*

### 1.6 Remediation Steps Demonstrated

1. **Gender standardisation** — map `M` → `Male`, `F` → `Female`; encode nulls/blanks as `Unknown`
2. **Date normalisation** — cascading `strptime` parser normalises all three formats to ISO 8601; ambiguous records flagged for human review
3. **Duplicate resolution** — retain the record with the most recent `processing_timestamp` per `_id`
4. **Invalid value flagging** — records with `credit_history_months < 0`, `savings_balance < 0`, or `debt_to_income > 1.0` flagged with `is_invalid = True` and excluded from downstream modelling
5. **Zero-income imputation** — zero and null incomes imputed using field median; originals preserved in `annual_income_raw`
6. **Email validation** — malformed emails flagged via regex; not imputed

---

## 2. Bias Detection & Fairness Analysis

Full methodology and statistical tests: `notebooks/02-bias-analysis.ipynb`

### 2.1 Approval Rates by Gender

| Gender | n | Approved | Approval Rate |
|--------|---|----------|---------------|
| Male | 248 | 163 | 65.7% |
| Female | 251 | 127 | 50.6% |
| Unknown/Missing | 3 | 2 | 66.7% |

![Figure 5: Approval rate by gender](figures/fig5_approval_by_gender.png)  
*Figure 5: Female applicants are approved at 50.6% versus 65.7% for males — a 15.1 percentage point gap.*

### 2.2 Disparate Impact Ratio
```
DI = P(approved | Female) / P(approved | Male)
   = 0.506 / 0.657
   = 0.770
```

> 🔴 **DI = 0.770 < 0.80** — The four-fifths threshold is violated, indicating potential **unlawful disparate impact** on female applicants.

**Demographic Parity Difference (DPD)** computed via `fairlearn`:
```
DPD = P(approved | Female) − P(approved | Male) = −0.151
```

Female applicants are **15.1 percentage points less likely** to be approved.

### 2.3 Age-Based & Intersectional Bias

| Age Group | Male Approval Rate | Female Approval Rate | Female DI |
|-----------|-------------------|---------------------|-----------|
| 18–25 | 61.4% | 55.9% | 0.91 |
| 26–35 | 51.2% | 34.5% | **0.67** 🔴 |
| 36–45 | 67.3% | 54.1% | 0.80 |
| 46–55 | 71.8% | 57.3% | 0.80 |
| 56–65 | 77.8% | 52.4% | **0.67** 🔴 |

The 26–35 and 56–65 groups show **intersectional bias** — DI of 0.67 indicates compounding disadvantage beyond what gender alone explains.

### 2.4 Rejection Reason Distribution

| Rejection Reason | Count (n) | % of Total Rejections |
|----------------|----------|----------------------|
| `algorithm_risk_score` | 170 | 81.7% |
| `insufficient_credit_history` | 23 | 11.1% |
| `high_dti_ratio` | 13 | 6.3% |
| `low_income` | 4 | 1.9% |

![Figure 4: Rejection reason distribution](figures/fig4_rejection_reasons.png)  
*Figure 4: 81.7% of rejections carry only `algorithm_risk_score` — a black-box label providing no actionable information to the applicant or auditor.*

### 2.5 Proxy Discrimination Analysis

| Attribute | Proxy For | Finding |
|-----------|-----------|---------|
| `zip_code` | Gender | Point-biserial correlation **−0.82** with binary gender. ZIP `902xx` → overwhelmingly female; `100xx` → overwhelmingly male. |
| `credit_history_months` | Age | Younger applicants have shorter histories — creates structural age-correlated disadvantage. |
| `annual_income` | Gender | Income gender gaps make income-filtering a partial gender proxy. |
| `spending_behavior` categories | Health/lifestyle/religion | `Healthcare`, `Alcohol`, `Gambling` can reveal information qualifying as special category data under GDPR Art. 9. |

![Figure 6: Correlation heatmap](figures/fig6_correlation_heatmap.png)  
*Figure 6: `zip_code` (binary-encoded) shows the strongest correlation with both `gender_binary` and `loan_approved`, confirming its proxy discrimination risk.*

---

## 3. Privacy & GDPR Assessment

Full implementation: `notebooks/03-privacy-demo.ipynb`

### 3.1 PII Inventory

| Field | PII Type | GDPR Classification | Plaintext? |
|-------|---------|-------------------|------------|
| `full_name` | Direct identifier | Art. 4(1) | ✅ Yes |
| `email` | Direct identifier | Art. 4(1) | ✅ Yes |
| `ssn` | Government identifier | High sensitivity | ✅ Yes |
| `ip_address` | Online identifier | Recital 30 | ✅ Yes |
| `date_of_birth` | Quasi-identifier | Art. 4(1) | ✅ Yes |
| `zip_code` | Quasi-identifier | Personal when combined | ✅ Yes |
| `gender` | Quasi-identifier | Art. 4(1) / potential Art. 9 | ✅ Yes |

### 3.2 GDPR Compliance Assessment

| Requirement | Article | Status | Finding |
|------------|---------|--------|---------|
| Lawful basis | Art. 6 | ⚠️ Undocumented | No documented legal basis in dataset or metadata |
| Data minimisation | Art. 5(1)(c) | ❌ Violated | `ip_address` unnecessary; granular spending categories expose sensitive lifestyle data |
| Storage limitation | Art. 5(1)(e) | ❌ Violated | No retention timestamps; `processing_timestamp` 87.6% missing |
| Accuracy | Art. 5(1)(d) | ❌ Violated | Inconsistent gender/date formats, invalid values, duplicates |
| Integrity & confidentiality | Art. 5(1)(f) | ❌ Violated | All direct identifiers in plaintext |
| Right to erasure | Art. 17 | ❌ Not implementable | No mechanism to locate and delete individual records |
| Automated decision-making | Art. 22 | ❌ Violated | No human review pathway; no explanation provided |
| Data Protection by Design | Art. 25 | ❌ Not implemented | No pseudonymisation, no access controls |

### 3.3 Pseudonymisation Demonstration

Salted SHA-256 hashing of `ssn` — demonstrated in `03-privacy-demo.ipynb`:
```python
import hashlib, secrets

SALT = secrets.token_hex(16)  # Stored separately — never in the dataset

def pseudonymise_ssn(ssn: str, salt: str) -> str:
    return hashlib.sha256((salt + ssn).encode()).hexdigest()

df['ssn_pseudonymised'] = df['ssn'].apply(lambda x: pseudonymise_ssn(x, SALT))
```

The notebook demonstrates why **unsalted hashing is insufficient**: SSNs have a finite known format (~1 billion combinations), making them fully reversible via brute-force lookup in minutes without a secret salt.

### 3.4 EU AI Act Classification

NovaCred's lending model is **explicitly High-Risk** under EU AI Act Annex III, §5(b):

| Requirement | Article | Status |
|------------|---------|--------|
| Risk management system | Art. 9 | ❌ No documented risk assessment |
| Data governance | Art. 10 | ❌ Quality issues not remediated at ingestion |
| Record-keeping | Art. 12 | ❌ Timestamps missing for 87.6% of records |
| Human oversight | Art. 14 | ❌ No review fields in schema |
| Explainability | Art. 13 | ❌ 81.7% of rejections give no rationale |
| Conformity assessment | Art. 43 | ❌ No evidence of pre-deployment assessment |

---

## 4. Governance Recommendations

### 1. Pseudonymise All Direct Identifiers at Rest
**Finding:** `ssn`, `full_name`, `email`, `ip_address` stored in plaintext.  
**Action:** Salted SHA-256 hashing of `ssn` (demonstrated). Extend to `full_name` and `email` in all analytical copies. Store salt separately.  
**Addresses:** GDPR Art. 5(1)(f), Art. 25

### 2. Remove `ip_address`; Aggregate `spending_behavior`
**Finding:** `ip_address` serves no credit-decisioning purpose; granular categories expose special-category data.  
**Action:** Remove `ip_address`. Replace per-category spending with `total_monthly_spending`.  
**Addresses:** GDPR Art. 5(1)(b), Art. 5(1)(c)

### 3. Enforce Input Validation at Ingestion
**Finding:** Three date formats, four gender representations, invalid numeric values — all passed ingestion unchecked.  
**Action:** Schema validator enforcing ISO 8601 dates, controlled gender vocabulary, non-negative numeric fields, valid email format. Log rejected submissions.  
**Addresses:** GDPR Art. 5(1)(d); EU AI Act Art. 10

### 4. Implement Bias Monitoring with Defined Thresholds
**Finding:** DI = 0.770 overall; DI = 0.67 for women aged 26–35 and 56–65. `zip_code` correlation −0.82 with gender.  
**Action:** Compute DI by gender, age group, and intersection on every retrain. Trigger mandatory human review when DI < 0.80. Flag `zip_code` as a proxy variable.  
**Addresses:** GDPR Art. 5(1)(a); EU AI Act Art. 9, Art. 10

### 5. Replace `algorithm_risk_score` with Structured Rejection Reasons
**Finding:** 81.7% of rejections provide no actionable explanation.  
**Action:** Add `rejection_factors` field with top 3 contributing variables and direction.  
**Addresses:** GDPR Art. 22; EU AI Act Art. 13

### 6. Introduce Human-in-the-Loop Review
**Finding:** No human review fields exist; all rejections are fully automated.  
**Action:** Add `reviewed_by`, `review_timestamp`, `override_decision`. Mandate human review before communicating any rejection.  
**Addresses:** GDPR Art. 22; EU AI Act Art. 14

### 7. Implement a Data Retention Policy
**Finding:** No retention fields exist; data appears stored indefinitely.  
**Action:** Add `retention_until` at ingestion. Monthly automated archival/deletion job. Document in ROPA.  
**Addresses:** GDPR Art. 5(1)(e), Art. 30

### 8. Conduct a Data Protection Impact Assessment (DPIA)
**Finding:** High-risk automated system processing sensitive PII at scale; no DPIA evidence present.  
**Action:** Conduct a DPIA under GDPR Art. 35 before further deployment or retraining.  
**Addresses:** GDPR Art. 35; EU AI Act Art. 9, Art. 43

---

## 5. How to Reproduce
```bash
pip install pandas numpy matplotlib seaborn fairlearn jupyter
```

Run notebooks in order:

1. `notebooks/01-data-quality.ipynb` → outputs `data/cleaned_credit_applications.csv` + all figures to `figures/`
2. `notebooks/02-bias-analysis.ipynb` → reads cleaned CSV
3. `notebooks/03-privacy-demo.ipynb` → reads raw JSON (by design)

---

## 6. Git Collaboration

### Branch Structure
```
main
└── dev
    ├── feature/data-quality       (Inês)
    ├── feature/bias-analysis      (Anh)
    ├── feature/privacy-gdpr       (Estêvão)
    └── feature/readme-docs        (Jonas)
```

### Commit Prefixes
`[data]` · `[bias]` · `[privacy]` · `[docs]` · `[fix]`

### Contributions

| Member | Primary Work |
|--------|-------------|
| Inês Monteiro | Data loading, JSON flattening, date normalisation, cleaning pipeline, `01-data-quality.ipynb` |
| Anh Nguyen | DI ratio, DPD, age-group intersectional analysis, proxy analysis, `02-bias-analysis.ipynb`, `fairness_utils.py` |
| Estêvão Fernandes | PII inventory, salted pseudonymisation, GDPR mapping, EU AI Act classification, `03-privacy-demo.ipynb` |
| Jonas Knosp | README, governance recommendations, presentation narrative, PR reviews |

**Video:** `[INSERT YOUTUBE LINK HERE]`

---

*DEGO 2606 | MSc Business Analytics | Nova SBE | Team 11 – TXC | February 2026*