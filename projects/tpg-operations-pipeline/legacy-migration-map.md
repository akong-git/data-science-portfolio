# TPG Operations Pipeline -- Legacy Migration Map

This documents the mapping from legacy MicroStrategy freeform SQL scripts to their dbt replacements. Each legacy script was a monolithic SQL statement that I decomposed into modular, reusable dbt models.

---

## Complete Script-to-Model Mapping

| Legacy Script # | Legacy Name | dbt Model(s) | Migration Notes |
|----------------|-------------|--------------|-----------------|
| Script 01 | EFIN All Monitoring | `rpt_efin_all_monitoring` via `int_tpg_efin_all_monitoring_base` + 5 intermediates | Monolithic 500+ line script decomposed into 6 composable models. Original preserved as `tpg_sourceoftruth_legacy_EFIN_All_Moinitoring.sql` for column-level audit. |
| Script 02 | ERO Holds | `rpt_ero_holds` via `int_tpg_entity_disbursements`, `int_tpg_partner_disbursement_status` | Entity hold report (compliance holds, debt offsets). Uses "Pattern B" disbursement logic. |
| Script 03 | FCA Info V2 | `rpt_fca_info_v2` via `int_tpg_fca_waterfall` | Fully-collected loans. Original preserved as `tpg_sourceoftruth_legacy_fca_info_v2.sql`. |
| Script 05 | FCB Daily Disbursements V2 | `rpt_fcb_daily_disbursements_v2` via `int_tpg_fca_waterfall` | Advance loan funding report with outstanding balance and days-since-disbursement. |
| Script 06 | FCB Trial Balance V2 | `rpt_fcb_trial_balance_v2` via `int_tpg_fca_waterfall` | Loans with outstanding_balance > 0. Financial exposure monitoring. |
| Script 07 | FCB Cumulative Collections | `rpt_fcb_daily_collections_v2` via `int_tpg_fca_collections` | Combined with Script 08 into single daily collections model. Original preserved as `tpg_sourceoftruth_legacy_FCB-Cumulative Dsibursements V2.sql`. |
| Script 08 | FCB Daily Collections | `rpt_fcb_daily_collections_v2` via `int_tpg_fca_collections` | Dual waterfall source UNION. |
| Script 10 | Transmitter Fee Detail | `rpt_transmitter_fee_detail` via `int_tpg_fee_collections` | Granular fee collection detail by transmitter. |
| Script 11 | Transmitter Fee Summary | `rpt_transmitter_fee_summary` via `int_tpg_fee_collections` | Aggregated fee totals by transmitter. |
| Script 12 | Transmitter Deals Pro | `rpt_transmitter_deals_pro` via `int_tpg_transmitter_fee_performance` | Multi-year (3 years) transmitter deal analysis. Added seed-based priority pattern matching (`ref_transmitter_reporting_product_logic`). |
| Script 13 | RT Clearing | `rpt_rt_clearing` via `int_tpg_rt_clearing_reconciliation`, `int_tpg_taxpayer_disbursements`, `int_tpg_fca_disbursements` | Most complex report. UNION ALL of APPS section (clearing issues) + DISB section (pending disbursements). |
| Script 14 | Taxpayer Holds | `rpt_taxpayer_holds` via `int_tpg_taxpayer_disbursements` | "Pattern A" disbursement holds (fraud review, IRS offset). |
| Script 15 | All Fees by Payee Detail | `rpt_all_fees_by_payee_detail` via `int_tpg_partner_fee_obligations` | Fee collection breakdown by payee role (ERO/MASTER/TRANSMITTER/TPG). |
| Script 16 | Fee Disbursed by Payee Detail | `rpt_fee_disbursed_by_payee_detail` | Fee disbursements to partners after collection. |
| Script 17 | POS Fees | `rpt_pos_fee` via `int_tpg_fee_collections` | Point-of-sale fee report (FeeTypeGroupKey=4). |
| Script 19 | Fee Collections (various) | `int_tpg_fee_collections` | Comprehensive fee collection tracking. Handles reversals, franchise/P-Center logic, all fee types. |
| (new) | JH Fee Advance Detail | `rpt_jh_fee_advance_detail` via `int_tpg_jh_fee_advance_events` | **No legacy equivalent.** Built from scratch for Republic Bank requirements. |
| (new) | JH Fee Advance Summary | `rpt_jh_fee_advance_summary` via `int_tpg_jh_fee_advance_events` | **No legacy equivalent.** Aggregated report with BOD/EOD principal, remaining CAP. |
| (new) | EFIN Collections V2 | `rpt_efin_collections_v2` via `int_tpg_partner_waterfall` | **No legacy equivalent.** Daily ledger transaction detail by EFIN/partner account. |
| (new) | Return to IRS | `rpt_return_to_irs` | **No legacy equivalent.** Unclaimed/returned fee amount tracking. |

---

## Migration Approach

### Phase 1: Source-of-Truth Preservation
I started by converting each legacy SQL script verbatim into a dbt model (e.g., `tpg_sourceoftruth_legacy_EFIN_All_Moinitoring.sql`). These serve as regression baselines, materialized as tables, that the new models can be compared against.

### Phase 2: Decomposition into Intermediate Models
I identified shared logic across scripts:
- **Application enrichment** (hierarchy extraction, product codes, bank assignment) appeared in every script → `int_tpg_application_product_enhanced`
- **Fee calculations** appeared in Scripts 01, 10, 11, 15, 16, 17 → `int_tpg_partner_fee_obligations` + `int_tpg_efin_fee_rollup`
- **Advance loan waterfall** appeared in Scripts 03, 05, 06, 07, 08 → `int_tpg_fca_waterfall` + `int_tpg_fca_collections`
- **Disbursement patterns** appeared in Scripts 02, 13, 14 → `int_tpg_taxpayer_disbursements`, `int_tpg_entity_disbursements`, `int_tpg_fca_disbursements`

### Phase 3: Column-Level Audit
Built a Jinja audit framework (`audit_rpt_efin_all_monitoring.sql`) that joins legacy output to dbt output by EFIN and compares ~130+ columns. Produces `match_pct` per column, sorted worst-first. Every mismatch is investigated -- most were improvements (better NULL handling, corrected join logic).

### Phase 4: New Reports
With the intermediate layer in place, I built 4 new reports that had no legacy equivalent: JH Fee Advance detail/summary (for Republic Bank), EFIN Collections V2, and Return to IRS.

---

## What Changed vs. Legacy

| Dimension | Legacy (MSTR) | dbt Pipeline |
|-----------|--------------|--------------|
| **Modularity** | Each script is monolithic, 500+ lines | 20 reusable intermediate models |
| **Shared logic** | Copy-pasted across scripts | Computed once, referenced by many |
| **Documentation** | None | Column-level YML (559 lines), business context doc (600 lines) |
| **Version control** | None (pasted into MSTR UI) | Git-tracked with PR review |
| **Testing** | Manual spot checks | Column-level audit framework, dbt tests |
| **Parameterization** | Hardcoded filing year | `var('filing_year')` flows through all models |
| **New capabilities** | N/A | JH Fee Advance reports, EFIN Collections V2, Return to IRS |

---

## Evidence in Source Code

Legacy script references found in comments throughout the codebase:

```sql
-- int_tpg_fca_waterfall.sql, line 39:
-- "Matches MSTR 'loanproduct' CTE"

-- rpt_efin_all_monitoring.sql, line 5:
-- "Replication of the legacy tpg_sourceoftruth_legacy_EFIN_All_Moinitoring.sql"

-- int_tpg_efin_debt_pivot.sql, line 7:
-- "matching the legacy tpg_sourceoftruth_legacy_EFIN_All_Moinitoring.sql output"

-- jh_fee_advance_reference.md, line 7:
-- "Phase 1: Raw SQL for MicroStrategy freeform SQL (build first)"
```

RPT ticket numbers (RPT-8674, RPT-8581, RPT-8656) reference the internal project tracker for each migration milestone.
