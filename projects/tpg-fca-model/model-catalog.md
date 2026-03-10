# TPG FCA Model -- Model Catalog

## Data Science Pipeline

### Intermediate Models (`models/intermediate/tpg/`)

| Model | Materialization | Grain | Description |
|-------|----------------|-------|-------------|
| `int_tpg__fca_applicants` | `table` | 1 row per (identitytoken, filingyear, applicationkey) | **Base universe.** All identity tokens (SSN) that filed in 2023-2025 and had a loan application. Enriches with consumer PII, transmitter name mapping, prior-year IRS payment history (Full/Partial/None/Unknown), pre-ACK indicator, secondary token flag. Deduplicates: latest applicationkey per identity per year, first loanapplicationkey per app. |
| `int_tpg__ach_and_fees` | `table` | 1 row per applicationkey | **IRS refund amounts.** Federal ACH (achcategorykey=1), state ACH (achcategorykey=2), preparation fees (feetypekey=4). Only completed transactions (achtransactionstatuskey 3 or 6). |
| `int_tpg__experian_attributes` | `table` | 1 row per identitytoken | **Credit bureau features.** Pivots Experian vendor response attributes to wide format. Includes: VerificationResultCode, DeceasedResultCode, FormatResultCode, IssueResultCode, MilitaryIndicator, OFACIndicator, plus ~30 P13 credit attributes across ALL, AUA, COL, IQT, REV, STJ, STU, UTI categories. Sources from both `tpg.VendorProgramResponse` and `gbos_vip.vendorprogramresponse`. |
| `int_tpg__knockout_and_scoring` | `table` | 1 row per applicationkey | **Underwriting evaluation.** 50 hardcoded business rules (knockout flags): minimum age, deceased check, prior-year FCA debt, income requirements, forms restrictions (Schedule C limits), multi-bank account limits, withholding ratio thresholds, estimated tax payment limits. Includes SHAP scoring features, raw/calculated model scores, denial reasons (up to 4), risk category, loan eligibility status. |
| `int_tpg__tax_return_attributes` | `incremental` (unique_key: identitytoken + taxyear) | 1 row per (identitytoken, taxyear) | **IRS form field features.** Massive pivot of `taxreturndetail` x `taxreturnattribute` into 200+ columns. W-2 information (wages, SS wages, withholding), 1040 fields (AGI, total income, EIC, Additional CTC), Schedule A/B/C/EIC/8812/8862 details. Computed features: W-2 count, dependent child profiles (son/daughter/grandchild categorization), withholding ratio (`w_ratio`), standard/non-standard child percentages. |
| `int_tpg__psc_file` | `table` | 1 row per identitytoken | **External scoring.** PSC (Performance Scoring Category) file from S3 (`datascience.fca2025_06_psc`). Derives: `hist_2y` (first 2 chars of 3-year payment history), `in_model_scope` (PSC in 10/12/30/40 AND denial_cat in 0/1), `target` (1 for PSC 30/40 = loss, 0 for PSC 10/12 = non-loss, 99 otherwise). |

### Mart Models (`models/marts/tpg/`)

| Model | Materialization | Grain | Description |
|-------|----------------|-------|-------------|
| `mrt_tpg__fca_model_features` | `table` | 1 row per (identitytoken, app_filingyear) | **Wide feature table.** Joins all 6 intermediate models. 500+ columns covering applicant demographics, loan details, tax return attributes, credit bureau response, scoring evaluation, ACH amounts, and PSC category. **Derived features:** `r_eic2maxeic` (EIC / max EIC by child count), `eic_yoy_change` (via LAG window), `r_eic2expectedirsrefund`, `r_wages2totalincome`, `r_eic_ctc_2expectedirsrefund`. EIC cap lookup table (2022-2025) by qualifying children (0-3). Tax return joined with 1-year lag (filingyear - 1 = taxyear). |
| `mrt_tpg__fca_model_prior_year_features` | `table` | 1 row per identitytoken (for current filing year) | **Cross-year feature engineering.** Pivots feature table across 3 filing years per identity token. **Key features:** `has_fca_prev_year_2024/2023` (-1=no RT, 0=RT no FCA, 1=had FCA), `delta_eic_current_vs_prev` / `r_eic_current_vs_prev`, `delta_wages_*` / `r_wages_*`, `delta_prepfees_*` / `r_prepfees_*`, `refund_coverage_2024/2023` (actual_refund - loan_amount), `r_actual2expectedirsrefund_prev`, `refund_coverage_group_prev` ('not_applied'/'smaller_loan'/'larger_loan'), `has_same_transmitter_efin_*` (preparer switching), `has_same_return_filing_status_*`. |
| `mrt_tpg__fca_model_final_features` | `table` | 1 row per identitytoken (filing year 2025, in-scope) | **Production model input.** Selects 12 key features and applies binning/discretization. Scope: `app_filingyear = 2025`, `in_model_scope in ('0','1')`. |

### 12 Binned Features in Final Model

| # | Feature | Bins | What It Captures |
|---|---------|------|-----------------|
| 1 | `risk_f_binned` | 5 | PSC risk factor |
| 2 | `pct_standard_children_binned` | 6 | Standard child relationship percentage |
| 3 | `py_rrec_hist_ordinal_binned` | 3 | Prior-year IRS repayment history |
| 4 | `cnt_irs1040scheduleeic_qualifyingchildssn_binned` | 4 | Qualifying EIC child count |
| 5 | `days_from_tax_season_start_date_binned` | 11 | Filing timing relative to season start |
| 6 | `cnt_irs1040schedulec_binned` | 5 | Self-employment (Schedule C) count |
| 7 | `r_actual2expectedirsrefund_prev_binned` | 6 | Prior-year actual/expected refund ratio |
| 8 | `r_actual2expectedirsrefund_prev2_binned` | 6 | 2-year-ago actual/expected ratio |
| 9 | `w_ratio_binned` | 5 | Tax withholding ratio |
| 10 | `irsw2_socialsecuritywagesamount_binned` | 15 | Social Security wages amount |
| 11 | `r_eic2expectedirsrefund_binned` | 11 | EIC/expected refund ratio |
| 12 | `r_eic2maxeic_binned` | 8 | EIC utilization (actual EIC / statutory maximum) |
