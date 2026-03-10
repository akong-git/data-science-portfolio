# TPG FCA Model -- LLM Context

> Compressed, AI-readable summary. Intended for LLM assistants, RAG pipelines, and automated code review.

## What This Is

A credit risk feature engineering pipeline built in dbt on Redshift for SBTPG (Santa Barbara Tax Products Group), a Green Dot subsidiary. Predicts advance loan losses for tax-season financial products (FCA, ERA, RA). The model is an XGBoost classifier deployed on AWS SageMaker, scoring loan applications in real time. Designed and built by Andy Kong.

## Business Domain

TPG sits between the IRS and tax preparers. When a taxpayer files through a preparer, TPG provides advance loans (cash before the IRS refund arrives). The core risk: if the IRS refund is smaller than expected, the advance loan isn't fully repaid.

**Products:**
| Product | Template Keys | Timing |
|---------|--------------|--------|
| FCA (Funded Cash Advance) | 1 (principal), 2 (interest) | Post-ACK |
| ERA (Early Refund Advance) | 142 (principal), 143 (interest) | Pre-ACK |
| RA (Refund Advance) | 144 (principal), 145 (interest) | Pre/Post-ACK |

**Target variable:** 1 = No-IRS-Pay (PSC 30/40), 0 = Full-IRS-Pay (PSC 10/12). Model scope: PSC in (10,12,30,40) AND denial_cat in (0,1).

**Model performance:** KS=30.4, AUROC=0.702, GINI=0.404. 224,402 training records, 4.5% bad rate. XGBoost with Bayesian hyperparameter tuning.

## Model Inventory (9 dbt models)

**Intermediate (6):**
| Model | Purpose |
|-------|---------|
| `int_tpg__fca_applicants` | Base applicant universe: all identity tokens with loan applications in FY 2023-2025. Deduplicates to latest app per identity per year. Prior-year IRS payment history. |
| `int_tpg__ach_and_fees` | IRS refund amounts (federal/state ACH) and preparation fees per applicant. |
| `int_tpg__experian_attributes` | Credit bureau response: 30+ Experian P13 attributes plus verification/deceased/military/OFAC flags. Dual source (tpg + gbos_vip) with COALESCE. |
| `int_tpg__knockout_and_scoring` | 50 business rules (knockout flags), SHAP scoring features, model scores, denial reasons (up to 4). |
| `int_tpg__tax_return_attributes` | 200+ IRS form field pivots from taxreturndetail (W-2, 1040, Schedule EIC/C/SE). Incremental on identitytoken+taxyear. |
| `int_tpg__psc_file` | PSC scoring file from S3. Derives model scope and target variable. |

**Marts (3):**
| Model | Purpose |
|-------|---------|
| `mrt_tpg__fca_model_features` | Wide feature table joining all 6 intermediates. 500+ columns including derived financial ratios (EIC/max EIC, actual/expected refund, wages/income, withholding ratio). |
| `mrt_tpg__fca_model_prior_year_features` | Cross-year features: YoY deltas for EIC, wages, prep fees, refund coverage. Preparer switching detection. 3-year self-join. |
| `mrt_tpg__fca_model_final_features` | Production model input: 12 binned features with ordinal bins matching trained XGBoost expectations. FY2025, in-scope only. |

## The 12 Production Features (in scoring order)

1. `risk_f` (5 bins) -- 3-year IRS payment history risk factor
2. `pct_standard_children` (6 bins) -- % standard child relationships; EIC-aware sentinels
3. `py_rrec_hist` (3 bins) -- prior-year IRS repayment: Full/Partial/None
4. `cnt_irs1040scheduleeic_qualifyingchildssn` (4 bins) -- qualifying child SSN count
5. `days_from_tax_season_start_date` (11 bins) -- days from Jan 2, 2026
6. `cnt_irs1040schedulec` (5 bins) -- Schedule C count (self-employment)
7. `r_actual2expectedirsrefund_prev` (6 bins) -- actual/expected refund TY2024 (JH vs non-JH branching)
8. `r_actual2expectedirsrefund_prev2` (6 bins) -- same for TY2023
9. `w_ratio` (5 bins) -- withholding ratio
10. `irsw2_socialsecuritywagesamount` (15 bins) -- W-2 SS wages
11. `r_eic2expectedirsrefund` (11 bins) -- EIC/expected refund ratio
12. `r_eic2maxeic` (8 bins) -- EIC utilization vs statutory max

## Key Technical Patterns

1. **Tax return pivoting:** Massive pivot of `taxreturndetail` x `taxreturnattribute` into 200+ columnar features. Uses `incremental` with `unique_key=['identitytoken','taxyear']`.
2. **Cross-year features:** Self-join across 3 filing years (not LAG) for 15+ cross-year deltas because each metric needs its own prior-year lookup.
3. **50 knockout rules:** Hardcoded business rules from `loanapplicantevaluationdetail` (minimum age, deceased check, prior year debt, forms restrictions, withholding ratio, OFAC).
4. **Binning in SQL:** 12 features discretized into ordinal bins with explicit CASE WHEN boundaries matching trained XGBoost model's expected input.
5. **JH vs Non-JH branching:** Features 7 and 8 use different source data for Jackson Hewitt vs non-JH transmitters.
6. **EIC-aware sentinels:** `pct_standard_children` uses sentinel bin values 1-3 for edge cases (EIC=0/no deps, EIC=0/has deps, EIC>0/no deps) rather than NULL handling.
7. **Score conversion:** SageMaker returns P(no-IRS-pay) 0-1; converted to 1-999 score via `ROUND((1-p)*1000)`. Higher scores = lower risk.

## Source Systems

- `tpg.application`, `tpg.loanapplication`, `tpg.applicationadvance`
- `tpg.taxreturnheader`, `tpg.taxreturndetail`, `tpg.taxreturnattribute`
- `tpg.VendorProgramResponse`, `tpg.VendorResponseAttribute` (Experian)
- `gbos_vip.vendorprogramresponse` (additional Experian source)
- `tpg.loanapplicantevaluationdetail` (knockout rules)
- `datascience.fca2025_06_psc` (PSC file from S3)
- `tpg.consumerprofile` (PII linkage)
