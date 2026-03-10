# TPG FCA Model -- Model Catalog

## Data Science Pipeline (9 models)

### Intermediate Models (`models/intermediate/tpg/`)

| Model | Materialization | Grain | Description |
|-------|----------------|-------|-------------|
| `int_tpg__fca_applicants` | `table` | 1 row per (identitytoken, filingyear, applicationkey) | **Base universe.** All identity tokens that filed in FY 2023-2025 and had a loan application. Enriches with consumer PII, transmitter name mapping, prior-year IRS payment history (Full/Partial/None/Unknown via `applicationadvance`), pre-ACK indicator, secondary token flag. Deduplicates: latest applicationkey per identity per year, first loanapplicationkey per app. |
| `int_tpg__ach_and_fees` | `table` | 1 row per applicationkey | **IRS refund amounts.** Federal ACH (achcategorykey=1), state ACH (achcategorykey=2), preparation fees (feetypekey=4). Only completed transactions (achtransactionstatuskey 3 or 6). |
| `int_tpg__experian_attributes` | `table` | 1 row per identitytoken | **Credit bureau features.** Pivots Experian vendor response attributes to wide format. Includes: VerificationResultCode, DeceasedResultCode, FormatResultCode, IssueResultCode, MilitaryIndicator, OFACIndicator, plus ~30 P13 credit attributes across ALL, AUA, COL, IQT, REV, STJ, STU, UTI categories. Sources from both `tpg.VendorProgramResponse` and `gbos_vip.vendorprogramresponse`, COALESCEd for best coverage. |
| `int_tpg__knockout_and_scoring` | `table` | 1 row per applicationkey | **Underwriting evaluation.** 50 hardcoded business rules (knockout flags): minimum age, deceased check, prior-year FCA debt, income requirements, forms restrictions (Schedule C limits), multi-bank account limits, withholding ratio thresholds, estimated tax payment limits. Includes SHAP scoring features, raw/calculated model scores, denial reasons (up to 4), risk category, loan eligibility status. |
| `int_tpg__tax_return_attributes` | `incremental` (unique_key: identitytoken + taxyear) | 1 row per (identitytoken, taxyear) | **IRS form field features.** Massive pivot of `taxreturndetail` x `taxreturnattribute` into 200+ columns. W-2 information (wages, SS wages, withholding via key 325/326/327), 1040 fields (AGI, total income via key 71, EIC via key 55, Additional CTC), Schedule A/B/C/EIC/8812/8862 details. Computed features: W-2 count, dependent child profiles (son/daughter/grandchild categorization via key 66 for documentsequenceid 1/2/3), withholding ratio (`w_ratio`), standard/non-standard child percentages. |
| `int_tpg__psc_file` | `table` | 1 row per identitytoken | **External scoring.** PSC (Performance Scoring Category) file from S3 (`datascience.fca2025_06_psc`). Derives: `hist_2y` (first 2 chars of 3-year payment history), `in_model_scope` (PSC in 10/12/30/40 AND denial_cat in 0/1), `target` (1 for PSC 30/40 = No-IRS-Pay, 0 for PSC 10/12 = Full-IRS-Pay, 99 otherwise). |

### Mart Models (`models/marts/tpg/`)

| Model | Materialization | Grain | Description |
|-------|----------------|-------|-------------|
| `mrt_tpg__fca_model_features` | `table` | 1 row per (identitytoken, app_filingyear) | **Wide feature table.** Joins all 6 intermediate models. 500+ columns covering applicant demographics, loan details, tax return attributes, credit bureau response, scoring evaluation, ACH amounts, and PSC category. **Derived features:** `r_eic2maxeic` (EIC / max EIC by child count, with TY2025 caps: $649/0 kids, $4,328/1, $7,152/2, $8,046/3+), `eic_yoy_change` (via LAG window), `r_eic2expectedirsrefund`, `r_wages2totalincome`, `r_eic_ctc_2expectedirsrefund`. Tax return joined with 1-year lag (filingyear - 1 = taxyear). |
| `mrt_tpg__fca_model_prior_year_features` | `table` | 1 row per identitytoken (for current filing year) | **Cross-year feature engineering.** Self-joins feature table across 3 filing years. **Key features:** `has_fca_prev_year_2024/2023`, `delta_eic_current_vs_prev` / `r_eic_current_vs_prev`, `delta_wages_*` / `r_wages_*`, `delta_prepfees_*`, `refund_coverage_2024/2023` (actual_refund - loan_amount), `r_actual2expectedirsrefund_prev` / `_prev2` (with separate JH vs non-JH logic), `refund_coverage_group_prev` ('not_applied'/'smaller_loan'/'larger_loan'), `has_same_transmitter_efin_*` (preparer switching). |
| `mrt_tpg__fca_model_final_features` | `table` | 1 row per identitytoken (FY 2025, in-scope) | **Production model input.** Selects 12 features and applies binning/discretization via CASE WHEN. Bin boundaries are fixed to match the trained XGBoost model's expected input. Scope: `app_filingyear = 2025`, `in_model_scope in ('0','1')`. NULL/missing values get default bin values (see Implementation Specs). Features sent to SageMaker in exact sequence order (1-12). |

### The 12 Binned Features

| Seq | Feature | Default | Min | Max | Source Key / Logic |
|-----|---------|---------|-----|-----|-------------------|
| 1 | `risk_f` | 4 | 1 | 5 | `risk_f_cmb` from 3-year IRS payment history |
| 2 | `pct_standard_children` | 1 | 1 | 6 | TaxReturnAttributeKey=66, docseq 1/2/3; EIC-aware sentinels 1-3 |
| 3 | `py_rrec_hist` | 1 | 1 | 3 | `applicationadvance.prioryearfederal*refundamount` |
| 4 | `cnt_irs1040scheduleeic_qualifyingchildssn` | 1 | 1 | 4 | TaxReturnAttributeKey=133, COUNT DISTINCT |
| 5 | `days_from_tax_season_start_date` | 1 | 1 | 11 | DATEDIFF from Jan 2, 2026 |
| 6 | `cnt_irs1040schedulec` | 1 | 1 | 5 | TaxReturnAttributeKey=249, COUNT DISTINCT docid |
| 7 | `r_actual2expectedirsrefund_prev` | 2 | 1 | 6 | Actual/expected IRS refund TY2024 (JH vs non-JH branching) |
| 8 | `r_actual2expectedirsrefund_prev2` | 2 | 1 | 6 | Same for TY2023 |
| 9 | `w_ratio` | 1 | 1 | 5 | (key325+key326+key327) / MAX(key71, key32, key167, withholding) |
| 10 | `irsw2_socialsecuritywagesamount` | 1 | 1 | 15 | TaxReturnAttributeKey=170, SUM across W-2s |
| 11 | `r_eic2expectedirsrefund` | 1 | 1 | 11 | EIC (key55) / expected refund |
| 12 | `r_eic2maxeic` | 1 | 1 | 8 | EIC / max_eic by qualifying child count (TY2025 caps) |
