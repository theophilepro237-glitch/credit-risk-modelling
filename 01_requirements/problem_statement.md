# Problem Statement – Lending Club Credit Risk

| Field | Value |
|---|---|
| Author | _your name_ |
| Version | 1.0 (draft) |
| Date | 2026-10-13 |
| Status | Draft – pending data profiling (Phase 1B) |

---

## 1. Business context

Lending Club was a US peer-to-peer lender that issued unsecured personal loans of $1,000–$40,000 over 36 or 60 months. Every applicant was given a risk grade (A–G) and an interest rate to match. The lender earns interest when a loan is repaid. When a borrower stops paying, the loan is **charged off** and most of the remaining principal is lost.

Between 2007 and 2018 Lending Club issued about 2.26 million loans. A small improvement in telling good borrowers from bad ones at that scale is worth millions of dollars. The business needs to:

- **Approve** borrowers who will repay and **decline** those who will not.
- **Price** the risk it accepts.
- **Hold enough reserves** for the losses it expects on loans it has already issued. US lenders must do this under the CECL accounting standard.

## 2. Business questions

1. **Who will default?** What is the probability that a new applicant will default (PD), using only information available at the time of application?
2. **How much will we lose?** What is the Expected Loss (EL = PD × LGD × EAD) of the current outstanding portfolio, broken down by grade, term and vintage (the quarter the loans were issued)?
3. **Where should we draw the line?** Which credit-score cutoff maximises expected profit, and what approval rate and bad rate does it imply?
4. **Is risk changing over time?** Do recent vintages default faster or slower than older ones?

## 3. Definition of "default"

The target variable is `is_bad`, derived from `loan_status`:

| `loan_status` | Class | Reason |
|---|---|---|
| Charged Off | **Bad (1)** | Loan written off as a loss |
| Default | **Bad (1)** | Formally in default |
| Does not meet the credit policy. Status:Charged Off | **Bad (1)** | Charged off; issued under an older credit policy |
| Fully Paid | **Good (0)** | Repaid in full |
| Does not meet the credit policy. Status:Fully Paid | **Good (0)** | Repaid in full; issued under an older credit policy |
| Current, In Grace Period, Late (16–30 days), Late (31–120 days) | **Excluded** | Outcome not yet known (*indeterminate*). Labelling them would bias the model. |

> **Note:** status counts will be confirmed in data profiling (Phase 1B). A sensitivity test that treats *Late (31–120 days)* as Bad is planned.

**Outcome-window rule.** A loan only gets a final label once it has had its full term to play out. When the data was extracted (December 2018), many recent loans had finished early only because they *defaulted quickly*. Including them would make recent vintages look riskier than they really are. This is called *survivorship / maturity bias*. Therefore the PD model uses **only loans whose full term ended by December 2018** (see Scope).

## 4. Scope

**In scope**

| Dataset | Population | Purpose |
|---|---|---|
| Development (train) | 36-month loans issued 2007–2014 with a final outcome | Build the PD scorecard |
| Out-of-time test | 36-month loans issued 2015 (matured by Dec 2018) | Check the model on a later period it never saw, as a bank would |
| Live portfolio | Loans still *Current* at Dec 2018 | Score them and calculate Expected Loss on the outstanding book |
| LGD dataset | All charged-off loans | Model how much is lost once a loan defaults |

**Out of scope (v1)**

- **60-month loans** for PD modelling. Too few of them had matured by 2018, which limits the sample. Listed as future work.
- **Rejected applications** (`rejected_2007_to_2018Q4`). We never observe whether these applicants would have repaid. Correcting for this (*reject inference*) is listed as future work.
- **Post-origination fields.** Payment, recovery and last-payment columns are only known *after* the loan is issued. They are **excluded from the PD model** to prevent data leakage. They are used only to build the target, LGD and EAD.
- Macroeconomic stress testing.

## 5. Success metrics

**Model metrics (measured on the out-of-time test set)**

| Metric | Target | What it tells us |
|---|---|---|
| AUC / Gini | AUC ≥ 0.68 (Gini ≥ 0.36) | How well the score ranks bad borrowers above good ones |
| KS statistic | ≥ 25 | Maximum separation between the good and bad score distributions |
| Calibration | Predicted vs. actual bad rate within ±10% (relative) in each score decile | Whether the predicted probabilities can be trusted for Expected Loss |
| PSI (train → test) | < 0.10 | Whether the population is stable over time |
| Benchmark | Beat Lending Club's own `grade` as a predictor | Whether the model adds value over what the lender already had |

**Business metrics**

- Expected profit at the recommended cutoff vs. (a) approving everyone and (b) a grade-based policy.
- Bad rate and approval rate at the recommended cutoff.
- Expected Loss of the live portfolio in dollars and as a percentage of outstanding balance.

## 6. Stakeholders & deliverables

| Stakeholder | Need | Deliverable |
|---|---|---|
| Chief Risk Officer | Portfolio risk at a glance | Power BI dashboard: portfolio overview and Expected Loss |
| Credit Policy team | Where to set the approval cutoff | Scorecard (points table) and cutoff/profit simulator |
| Finance | Loss reserves (CECL) | Expected Loss by segment |
| Model Validation | Evidence the model is sound | Model documentation: metrics, stability, leakage controls |
| Data Engineering | Reproducible data flow | Azure pipeline (Blob → Azure SQL), SQL transformation scripts, data dictionary |

## 7. Assumptions & risks

| # | Assumption / risk | Mitigation |
|---|---|---|
| 1 | Historical relationships still hold for future applicants | Out-of-time validation and PSI monitoring |
| 2 | Accepted loans are not representative of all applicants (*selection bias*) | Documented limitation; reject inference as future work |
| 3 | Post-origination fields leak future information | Explicit list of allowed features; leakage check before modelling |
| 4 | LGD is estimated from recoveries recorded up to Dec 2018 and may understate late recoveries | Documented as a conservative assumption |
| 5 | Credit policy changed over 2007–2018 | Vintage analysis; flag for "does not meet credit policy" loans |

## 8. Plan

| Phase | Output | Folder |
|---|---|---|
| 1 | Problem statement, data profiling | `01_requirements`, `02_data_source_analysis` |
| 2 | Raw data in Azure Blob, then loaded to Azure SQL | `03_raw_data` |
| 3 | Cleaning and transformation (SQL + Python) | `04_data_processing` |
| 4 | PD scorecard, LGD, EAD, Expected Loss | `05_data_model` |
| 5 | Power BI dashboard and cutoff simulator | `06_dashboard_development` |
| 6 | Testing and validation | `07_UAT` |
| 7 | Publish and write up | `08_production`, `09_documentation_governance` |
