# TPG Operations Pipeline -- Model Catalog

## Staging Layer (3 models)

All materialized as `view` in `models/staging/tpg/`.

| Model | Source | Description |
|-------|--------|-------------|
| `stg_tpg_applications` | `tpg.application` | Passthrough select |
| `stg_tpg_cash_receipt_journals` | `tpg.cashreceiptjournal` | Column rename/aliasing |
| `stg_tpg_cash_receipt_subledgers` | `tpg.cashreceiptsubledger` | Column rename/aliasing |

Note: Most intermediate models bypass staging and reference `source('tpg', ...)` directly. The staging models exist as a future refactoring target.

---

## Intermediate Layer (20 models)

All materialized as `table` in `models/intermediate/tpg_operations/`.

### Foundation

| Model | Grain | Description |
|-------|-------|-------------|
| `int_tpg_application_product_enhanced` | 1 row per applicationkey | **Universal starting point.** Enriches raw application with: product codes, business line (Pro vs Franchise), bank assignment (Green Dot Bank vs Civista Bank), consumer PII (encrypted SSN), 6-level hierarchy extraction (ERO/SB/Transmitter EFINs), transmitter name classification, Super/Sub Service Bureau extraction, test account flag. |

### Volume & Funding

| Model | Grain | Description |
|-------|-------|-------------|
| `int_tpg_efin_volume_funded` | 1 row per (monitoring_efin, role_level) | Application volume and funding per EFIN. **Key business rule:** ERO volume excludes rejected/error (status not in 4,5); MASTER volume counts ALL applications. Federal and state funded counts from application refund waterfall detail. |

### Fee Pipeline

| Model | Grain | Description |
|-------|-------|-------------|
| `int_tpg_partner_fee_obligations` | 1 row per waterfall detail key | **Dual-source fee normalization.** UNIONs refund waterfall + disbursement fee waterfall into unified fee model. Normalizes payee roles: ERO, MASTER, TRANSMITTER, TPG, FRANCHISE, P-CENTER. |
| `int_tpg_efin_fee_rollup` | 1 row per (monitoring_efin, payee_role_level) | Aggregates fee obligations: total_fees_requested / paid / pending per EFIN. |

### Debt Pipeline

| Model | Grain | Description |
|-------|-------|-------------|
| `int_tpg_partner_debt_obligations` | 1 row per (partner_identity_value, templateattributekey) | Partner-level debts from `partnerrolecollectionwaterfalldetail`. Covers 53 debt categories including JH-specific keys (153-171). Sources from raw tables, not Redshift views (which exclude JH keys). |
| `int_tpg_efin_debt_rollup` | Same as above | Thin rename layer mapping to monitoring pipeline naming conventions. |
| `int_tpg_efin_debt_pivot` | 1 row per monitoring_efin | **159-column pivot.** Pivots long-form debt (53 categories x 3 fields: amount, amountcollected, balancedue) into wide format. Standard categories matched by `LOWER(templateattribute)` text; JH categories by `templateattributekey` integer. Computes grand totals matching legacy output. |

### Monitoring Hub

| Model | Grain | Description |
|-------|-------|-------------|
| `int_tpg_efin_all_monitoring_base` | 1 row per (monitoring_efin, role_level) | **Central hub: 160+ columns.** Assembles 5 data streams: (1) Fees from efin_fee_rollup, (2) Volume from efin_volume_funded, (3) Granular debt from efin_debt_pivot (~159 columns), (4) Partner dimensions (companyname, partner_type, isjh, transmitterefin), (5) PII contacts via `pdr_stg.decryptaes()`. |

### Advance Loan Pipeline

| Model | Grain | Description |
|-------|-------|-------------|
| `int_tpg_fca_waterfall` | 1 row per applicationkey | Consolidates FCA+ERA+RA. Principal/interest owed/collected/balance, product classification, funding date, repay date, disbursement method, daily interest, JH vs Non-JH segment. |
| `int_tpg_fca_collections` | 1 row per (applicationkey, collection_date) | Daily advance loan repayment from dual waterfall sources (UNION ALL). Aggregates principal + interest collections. |
| `int_tpg_fca_disbursements` | 1 row per disbursementkey | "Pattern C" advance disbursements (AR-linked). Tracks funding to taxpayers. |

### Disbursement Pipeline

| Model | Grain | Description |
|-------|-------|-------------|
| `int_tpg_taxpayer_disbursements` | 1 row per disbursementkey | "Pattern A" taxpayer refund disbursements (91% of volume). Tracks reissues, status, method, subsidiary account context. |
| `int_tpg_entity_disbursements` | 1 row per disbursementkey | "Pattern B" partner fee split disbursements (`applicationkey IS NULL`, `subsidiaryaccountbalancekey IS NOT NULL`). |
| `int_tpg_partner_disbursement_status` | 1 row per disbursementkey | Pending partner fee disbursements (ACH Wire, method=6, status=2 Pending with reason). |

### Partner Ledger & Fee Collections

| Model | Grain | Description |
|-------|-------|-------------|
| `int_tpg_partner_waterfall` | 1 row per subsidiary account transaction ledger key | Double-entry bookkeeping for partner accounts. Normalizes ledger entries joined with collection waterfall detail. Computes `amount_collected_component` and `is_fully_collected`. |
| `int_tpg_fee_collections` | 1 row per (applicationkey, fee detail) | Comprehensive fee collection tracking. Handles reversals (key 77), franchise/P-Center logic (template keys 24/25), all fee types. |

### Other Intermediate Models

| Model | Grain | Description |
|-------|-------|-------------|
| `int_tpg_partner_hierarchy` | 1 row per (applicationkey, hierarchy node) | ERO/SB/Transmitter EFIN extraction from 6-level hierarchy. |
| `int_tpg_rt_clearing_reconciliation` | 1 row per applicationkey | Compares IRS deposit vs fees owed vs paid vs pending. Assigns reason codes ("RTC < Fees", "No ERO Record", "Ready"). |
| `int_tpg_transmitter_fee_performance` | 1 row per (transmitter, product, year) | Multi-year fee collection and funding metrics. Uses seed-based priority pattern matching via `ref_transmitter_reporting_product_logic`. |
| `int_tpg_jh_fee_advance_events` | 1 row per (pcenter_efin, event_date) | Daily JH fee advance activity combining 3 sources: CAP (loanproductkey=14), Advances (disbursements), Repayments (collection waterfall key 154). |

---

## Mart Layer (19 reports)

All in `models/marts/tpg_operations/`.

### Flagship Monitoring

| Report | Materialization | Description |
|--------|----------------|-------------|
| `rpt_efin_all_monitoring` | `view` | **Primary daily operational report.** 160+ columns: volume, funding, fees, 53 debt categories, partner ID, decrypted PII. Consumed by TPG Operations + Tableau. |

### Advance Loan Reports

| Report | Materialization | Description |
|--------|----------------|-------------|
| `rpt_fca_info_v2` | `view` | Fully-collected advance loans (principal_collected = principal_owed) |
| `rpt_fcb_daily_collections_v2` | report | Daily principal/interest collection amounts |
| `rpt_fcb_daily_disbursements_v2` | report | Funded loans with outstanding balance |
| `rpt_fcb_trial_balance_v2` | report | Loans with outstanding_balance > 0 |

### Hold Reports

| Report | Materialization | Description |
|--------|----------------|-------------|
| `rpt_ero_holds` | report | ERO/entity holds (compliance, debt offsets) |
| `rpt_taxpayer_holds` | report | Taxpayer refund holds (fraud review, IRS offset) |

### Fee Reports

| Report | Materialization | Description |
|--------|----------------|-------------|
| `rpt_all_fees_by_payee_detail` | report | Fee collection breakdown by payee role |
| `rpt_fee_disbursed_by_payee_detail` | report | Fee disbursements to partners |
| `rpt_pos_fee` | report | Point-of-sale fee report (FeeTypeGroupKey=4) |

### Transmitter Reports

| Report | Materialization | Description |
|--------|----------------|-------------|
| `rpt_transmitter_deals_pro` | `table` | Pro-line transmitter deal performance across 3 years. Seed-based product classification. |
| `rpt_transmitter_fee_detail` | report | Granular fee collection by transmitter |
| `rpt_transmitter_fee_summary` | report | Aggregated fee totals by transmitter |

### Reconciliation & Ledger

| Report | Materialization | Description |
|--------|----------------|-------------|
| `rpt_rt_clearing` | `table` | UNION ALL of APPS section + DISB section for clearing reconciliation |
| `rpt_efin_collections_v2` | report | Daily ledger transaction detail by EFIN |
| `rpt_return_to_irs` | report | Unclaimed/returned fee amounts |

### JH Fee Advance

| Report | Materialization | Description |
|--------|----------------|-------------|
| `rpt_jh_fee_advance_detail` | `view` | P-Center + Date grain with running cumulative balance (window function) |
| `rpt_jh_fee_advance_summary` | `view` | All P-Centers aggregated: BOD/EOD principal, remaining CAP |

---

## Seed File

| Seed | Description |
|------|-------------|
| `ref_transmitter_reporting_product_logic` | 35 priority-ordered pattern-matching rules to classify transmitter name + product code + sub-product + disbursement method + EFIN into `clean_transmitter_name` and `reporting_product_group`. Covers: Advanced Tax Solutions, TaxWise CCH, CrossLink, Drake, Intuit, Online Taxes, ProTaxPro, TaxACT, TaxLync, TaxSlayer, TaxHawk, EZ Tax, Taxware Systems, TenForty. |
