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

- **22.1% of records** exhibit inconsistent gender encoding, with four distinct string representations used for a binary categorical field, constituting a critical validity and consistency failure prior to any modelling step.
- A statistically significant gender-based disparity in loan approval rates was identified. Applying the four-fifths rule (80% rule), the computed **Disparate Impact Ratio of 0.770** falls below the 0.80 threshold, indicating potential unlawful disparate impact on female applicants under EU anti-discrimination frameworks.
- **Multiple categories of PII** — including Social Security Numbers, full names, email addresses, and IP addresses — are stored in plaintext with no evidence of pseudonymisation, encryption, or access controls, in direct conflict with GDPR Articles 5 and 25.
- The dominant rejection mechanism (`algorithm_risk_score`, accounting for **81.7% of denials**) is non-transparent and non-auditable, raising high-risk AI system concerns under the EU AI Act.

---

## Repository Structure

```
dego-project-team11-TXC/
├── README.md                     ← Executive summary, findings, governance recommendations
├── data/
│   └── raw_credit_applications.json
├── notebooks/
│   ├── 01-data-quality.ipynb     ← Completeness, consistency, validity audit + remediation
│   ├── 02-bias-analysis.ipynb    ← Disparate Impact, proxy discrimination, fairness metrics
│   └── 03-privacy-demo.ipynb     ← PII identification, pseudonymisation demo, GDPR mapping
├── src/
│   └── fairness_utils.py         ← Reusable fairness metric functions (DI ratio, DPD, etc.)
└── presentation/
    └── [video link]
```

---

## Dataset Overview

**Source file:** `raw_credit_applications.json`
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
| `applicant_info.date_of_birth` | String | Date of birth |
| `applicant_info.zip_code` | String | ZIP/postal code |
| `financials.annual_income` | Number | Annual income |
| `financials.credit_history_months` | Integer | Months of credit history |
| `financials.debt_to_income` | Number | Debt-to-income ratio |
| `financials.savings_balance` | Number | Savings balance |
| `spending_behavior[].category` | String | Spending category label |
| `spending_behavior[].amount` | Number | Monthly spend in that category |
| `decision.loan_approved` | Boolean | Approval outcome |
| `decision.interest_rate` | Number | Rate assigned (if approved) |
| `decision.rejection_reason` | String | Rejection label (if denied) |

> ⚠️ The dataset contains intentional data quality issues and bias patterns. All issues identified below are documented and quantified in `notebooks/01-data-quality.ipynb` and `notebooks/02-bias-analysis.ipynb`.

---

## 1. Data Quality Analysis

Full methodology and remediation code: `notebooks/01-data-quality.ipynb`

### 1.1 Completeness

Missing values were assessed across all fields. The table below reports the count and percentage of records where a field is null, empty string, or absent from the JSON object.

| Field | Missing (n) | Missing (%) |
|-------|------------|------------|
| `email` | 7 | 1.4% |
| `date_of_birth` | 5 | 1.0% |
| `ssn` | 5 | 1.0% |
| `annual_income` | 5 | 1.0% |
| `gender` | 3 | 0.6% |

> 📊 *[Figure 1: Missing value heatmap by field — insert from `01-data-quality.ipynb`]*

### 1.2 Consistency

| Issue | Affected Records (n) | Affected Records (%) |
|-------|---------------------|---------------------|
| Inconsistent gender encoding (`M`/`Male`, `F`/`Female`) | 111 | 22.1% |
| Duplicate `_id` values | 2 | 0.4% |
| Approved loans missing `interest_rate` field | detected | — |

**Gender encoding inconsistency** is the most pervasive consistency failure: four string variants are used for a binary field (`Male`, `M`, `Female`, `F`), plus null/empty entries. Prior to any analysis involving gender, all variants must be normalised (e.g., `M` → `Male`, `F` → `Female`).

### 1.3 Validity

| Issue | Detail |
|-------|--------|
| Negative `credit_history_months` | Min observed: -10 months. Logically impossible; likely a data entry error. |
| `debt_to_income` > 1.0 | 1 record with DTI > 1.0, which is invalid (total debt exceeds income). |
| `annual_income` = 0 | Present in dataset; likely a missing value recorded as zero rather than null. |
| Applicant age range | 21–65 years (median: 37). No underage applicants detected. |
| `credit_history_months` range | -10 to 133 months. Upper bound is plausible; lower bound is invalid. |

> 📊 *[Figure 2: Distribution plots for `annual_income`, `credit_history_months`, `debt_to_income` — insert from `01-data-quality.ipynb`]*

### 1.4 Remediation Steps Demonstrated

The following transformations are implemented in `01-data-quality.ipynb`:

1. **Gender standardisation** — map `M` → `Male`, `F` → `Female`; encode nulls/blanks as `Unknown`
2. **Duplicate resolution** — retain the record with the most recent `processing_timestamp` per `_id`
3. **Invalid value flagging** — records with `credit_history_months < 0` or `debt_to_income > 1.0` are flagged with an `is_invalid` boolean column and excluded from downstream modelling
4. **Zero-income imputation** — zero-value incomes are treated as missing and imputed using field median; original values preserved in a `_raw` column for auditability

---

## 2. Bias Detection & Fairness Analysis

Full methodology and statistical tests: `notebooks/02-bias-analysis.ipynb`

### 2.1 Gender Encoding Normalisation

Before computing approval rates, gender was normalised as described in §1.2. Post-normalisation group sizes and approval rates are as follows:

| Gender | n | Approved | Approval Rate |
|--------|---|----------|---------------|
| Male | 248 | 163 | 65.7% |
| Female | 251 | 127 | 50.6% |
| Unknown/Missing | 3 | 2 | 66.7% |

> 📊 *[Figure 3: Approval rate by gender (bar chart) — insert from `02-bias-analysis.ipynb`]*

### 2.2 Disparate Impact Ratio

The Disparate Impact Ratio (DI) is computed following the four-fifths (80%) rule, where the unprivileged group is Female and the privileged group is Male:

$$DI = \frac{P(\text{approved} \mid \text{Female})}{P(\text{approved} \mid \text{Male})} = \frac{0.506}{0.657} = \mathbf{0.770}$$

> 🔴 **DI = 0.770 < 0.80** — The four-fifths threshold is violated. This result is consistent with potential **unlawful disparate impact** on female applicants, warranting further investigation and regulatory disclosure.

The `fairlearn` library was additionally used to compute the **Demographic Parity Difference (DPD)**:

$$DPD = P(\hat{y}=1 \mid \text{Female}) - P(\hat{y}=1 \mid \text{Male}) = 0.506 - 0.657 = -0.151$$

A DPD of -0.151 confirms that female applicants are 15.1 percentage points less likely to be approved, independent of financial qualifications.

### 2.3 Rejection Reason Distribution

| Rejection Reason | Count (n) | % of Total Rejections |
|----------------|----------|----------------------|
| `algorithm_risk_score` | 170 | 81.7% |
| `insufficient_credit_history` | 23 | 11.1% |
| `high_dti_ratio` | 13 | 6.3% |
| `low_income` | 4 | 1.9% |

The near-exclusive reliance on `algorithm_risk_score` as a rejection rationale is a significant governance concern. The label is opaque — it provides no insight into which features drove the decision, nor whether the scoring model was validated for fairness. Under the **EU AI Act (Art. 6, Annex III)**, automated creditworthiness assessments qualify as high-risk AI systems, requiring conformity assessments and the right to human review.

> 📊 *[Figure 4: Rejection reason distribution (horizontal bar chart) — insert from `02-bias-analysis.ipynb`]*

### 2.4 Proxy Discrimination Analysis

The following non-protected attributes were investigated as potential proxies for protected characteristics:

| Attribute | Potential Proxy For | Analysis |
|-----------|-------------------|---------|
| `zip_code` | Race/ethnicity | Geographic clustering may reproduce redlining patterns |
| `spending_behavior.category` | Lifestyle/religion/culture | Categories such as `Alcohol` or `Gambling` may correlate with demographic groups |
| `credit_history_months` | Age | Younger applicants systematically have shorter credit histories, creating age-correlated disadvantage |
| `annual_income` | Gender | Income gender gaps may cause income-based filtering to act as a gender proxy |

Full correlation matrices and chi-squared independence tests between these attributes and `loan_approved` are documented in `02-bias-analysis.ipynb`.

> 📊 *[Figure 5: Correlation heatmap between financial features and approval outcome — insert from `02-bias-analysis.ipynb`]*

---

## 3. Privacy & GDPR Assessment

Full implementation: `notebooks/03-privacy-demo.ipynb`

> ⚠️ *This section is currently in progress. The PII inventory and GDPR mapping below are complete; pseudonymisation implementation and GDPR article cross-referencing will be finalised in the notebook.*

### 3.1 PII Inventory

| Field | PII Type | GDPR Classification |
|-------|---------|-------------------|
| `full_name` | Direct identifier | Personal data (Art. 4(1)) |
| `email` | Direct identifier | Personal data (Art. 4(1)) |
| `ssn` | Government-issued identifier | Personal data — high sensitivity |
| `ip_address` | Pseudonymous identifier | Personal data when linkable (Recital 30) |
| `date_of_birth` | Quasi-identifier | Personal data (Art. 4(1)) |
| `zip_code` | Quasi-identifier | Personal data when combined |
| `gender` | Quasi-identifier / potential special category | Personal data; may become special category in combination (Art. 9) |

### 3.2 GDPR Compliance Assessment

| Requirement | Article | Status | Finding |
|------------|---------|--------|---------|
| Lawful basis for processing | Art. 6 | ⚠️ Undocumented | No documented legal basis present in the dataset or accompanying metadata |
| Data minimisation | Art. 5(1)(c) | ❌ Violated | `ssn` and `ip_address` appear unnecessary for the credit decision function |
| Storage limitation | Art. 5(1)(e) | ❌ Violated | No retention timestamps or deletion flags present; data appears to be retained indefinitely |
| Right to erasure | Art. 17 | ❌ Not implemented | No mechanism for individual record deletion demonstrated |
| Privacy by design | Art. 25 | ❌ Not implemented | PII stored in plaintext with no pseudonymisation, hashing, or access control |
| High-risk AI classification | EU AI Act, Art. 6 + Annex III | ❌ Not addressed | Automated credit scoring meets the definition of a high-risk AI system; no conformity assessment documented |

### 3.3 Pseudonymisation Demonstration

The following approach is implemented in `03-privacy-demo.ipynb` to replace plaintext SSNs with non-reversible tokens:

```python
import hashlib

def pseudonymise_ssn(ssn: str, salt: str = "novacred_2024") -> str:
    """
    Returns a 16-character SHA-256 hex digest of the salted SSN.
    The salt must be stored securely and separately from the dataset.
    Re-identification requires both the token and the salt — satisfying
    GDPR's pseudonymisation definition under Recital 26.
    """
    return hashlib.sha256(f"{salt}{ssn}".encode()).hexdigest()[:16]
```

The same approach is applied to `full_name` and `email`. The `ip_address` field is truncated to the /24 subnet prefix (last octet zeroed) to remove individual-level precision while preserving geographic signal for fraud detection purposes.

---

## 4. Governance Recommendations

### 4.1 Immediate Controls (0–30 days)

- **Suspend opaque rejection labelling.** The `algorithm_risk_score` rejection reason must be replaced with a structured, feature-level explanation for each denial, fulfilling the right to explanation under GDPR Art. 22 and EU AI Act transparency requirements.
- **Implement PII pseudonymisation at ingestion.** SSN, email, and full name must be hashed before any downstream processing. Raw PII must be stored in a separate, access-controlled vault with a documented legal basis.
- **Enforce data validation at source.** A schema validation layer must reject records with logically invalid values (negative credit history, DTI > 1.0, null mandatory fields) before they enter the processing pipeline.

### 4.2 Short-Term Controls (30–90 days)

- **Deploy a bias monitoring dashboard** computing DI ratio, DPD, and Equalised Odds across protected attributes (gender, age bracket, ZIP cluster) on a rolling 30-day window. Automated alerts should trigger when DI < 0.85 — a conservative buffer above the regulatory 0.80 threshold.
- **Establish an immutable audit trail.** Every credit decision must be logged with: timestamp, application ID, model version, feature vector hash, and outcome. Logs must be retained for a minimum of 5 years.
- **Implement a human review queue** for all `algorithm_risk_score` rejections, fulfilling the EU AI Act Art. 14 requirement for meaningful human oversight of high-risk AI system outputs.
- **Document and enforce a retention policy.** Define maximum storage durations per data category and implement automated deletion workflows triggered by retention expiry.

### 4.3 Long-Term Governance Roadmap

- **Fairness-constrained model retraining.** Retrain the credit scoring model with fairness constraints (e.g., demographic parity or equalised odds) using `fairlearn`. Validate on held-out data stratified by protected attribute before re-deployment.
- **EU AI Act conformity assessment.** Commission a formal conformity assessment for the credit scoring system as a high-risk AI application, including documentation of training data, model card, risk register, and post-market monitoring plan.
- **Proxy discrimination audit.** Conduct a systematic causal analysis of whether non-protected features (zip code, spending categories, income) encode protected characteristics, and remove or transform features confirmed to act as proxies.
- **Establish a Data Governance Board.** Form a cross-functional oversight body with a quarterly mandate to review bias metrics, audit logs, incident reports, and regulatory developments.
- **Consent and purpose tracking.** Implement explicit, granular consent records per applicant, linked to each processing purpose, with documented withdrawal mechanisms.

---

## 5. Reproducibility

### Environment Setup

```bash
git clone https://github.com/[your-org]/dego-project-team11-TXC.git
cd dego-project-team11-TXC
pip install -r requirements.txt
jupyter notebook notebooks/
```

### Dependencies

```
pandas>=2.0
numpy>=1.25
matplotlib>=3.7
seaborn>=0.12
fairlearn>=0.9
```

### Notebook Execution Order

Notebooks must be run in sequence:

1. `01-data-quality.ipynb` — outputs `data/cleaned_applications.json`
2. `02-bias-analysis.ipynb` — reads `data/cleaned_applications.json`
3. `03-privacy-demo.ipynb` — reads `data/raw_credit_applications.json` (operates on raw data by design)

---

## 6. Git Collaboration

### Branch Structure

```
main              ← stable, submission-ready branch
└── dev           ← integration branch for pre-submission review
    ├── feature/data-quality       (Inês)
    ├── feature/bias-analysis      (Anh)
    ├── feature/privacy-gdpr       (Estêvão)
    └── feature/readme-docs        (Jonas)
```

### Commit Convention

Commits follow a tag-prefix convention for traceability:

| Prefix | Scope |
|--------|-------|
| `[data]` | Data ingestion, cleaning, pipeline |
| `[bias]` | Fairness metrics, statistical analysis |
| `[privacy]` | GDPR mapping, pseudonymisation |
| `[docs]` | README, comments, documentation |
| `[fix]` | Bug fixes and corrections |

### Contribution Summary

| Member | Primary Contributions |
|--------|----------------------|
| Inês Monteiro | Data loading, JSON flattening, cleaning pipeline, `01-data-quality.ipynb` |
| Anh Nguyen | DI ratio, DPD computation, proxy analysis, `02-bias-analysis.ipynb`, `fairness_utils.py` |
| Estêvão Fernandes | PII inventory, Pseudonymization ,GDPR mapping, `03-privacy-demo.ipynb` |
| Jonas Knosp | README, project coordination, presentation narrative, PR reviews |

> All team members have committed under their own GitHub accounts. Commit history reflects steady progress across multiple working sessions.

**Video:** `[insert YouTube link]`

---

*DEGO 2606 | MSc Business Analytics | Nova SBE | Team 11 – TXC | February 2026*
