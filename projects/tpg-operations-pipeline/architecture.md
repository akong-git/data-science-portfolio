# TPG Operations Pipeline Architecture

## Overview

I migrated 19+ legacy MicroStrategy freeform SQL scripts to a modular dbt pipeline, building everything from scratch. The legacy scripts were monolithic -- each one was a single 500+ line SQL statement with no shared logic, no documentation, and no version control. The dbt pipeline replaces them with a layered architecture where common transformations are computed once and reused across reports.

The pipeline covers the full operational reporting suite for SBTPG: EFIN monitoring, advance loan tracking, fee collection, hold management, transmitter deals, RT clearing reconciliation, and partner ledger activity.

## Three-Layer Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  SOURCES: greendot.tpg (124+ tables)                            │
│                                                                 │
│  Core: application, loanapplication, partnerrole,               │
│        partnerrolehierarchy, applicationrefundwaterfall[detail], │
│        applicationdisbursementfeewaterfall[detail],              │
│        partnerrolecollectionwaterfall[detail],                   │
│        cashreceiptjournal, disbursement,                         │
│        subsidiaryaccounttransactionledger                       │
│                                                                 │
│  Views: vw_partnerrolehierarchy, vw_partnerrolecontact,         │
│         vw_mstr_enrollmentinforeport                            │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGING (3 models, materialized as views)                      │
│                                                                 │
│  stg_tpg_applications         ← Passthrough from application    │
│  stg_tpg_cash_receipt_journals ← Column rename/alias            │
│  stg_tpg_cash_receipt_subledgers ← Column rename/alias          │
│                                                                 │
│  Note: Most intermediates bypass staging and ref source()       │
│  directly. Staging exists as a refactoring target.              │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  INTERMEDIATE (20 models, all materialized as tables)           │
│                                                                 │
│  ┌─ Foundation ───────────────────────────────────────────────┐ │
│  │  int_tpg_application_product_enhanced                      │ │
│  │   (universal starting point: enriched applications)        │ │
│  └────────────┬───────────────────────────────────────────────┘ │
│               │                                                 │
│  ┌────────────┼────────────────────────────────────────────┐    │
│  │            ▼                                            │    │
│  │  ┌─ Volume & Funding ──┐  ┌─ Fee Pipeline ──────────┐  │    │
│  │  │ efin_volume_funded   │  │ partner_fee_obligations  │  │    │
│  │  └──────────┬───────────┘  │     ▼                    │  │    │
│  │             │              │ efin_fee_rollup           │  │    │
│  │             │              └──────────┬───────────────┘  │    │
│  │             │                         │                  │    │
│  │  ┌─ Debt Pipeline ──────────────────────────────────┐    │    │
│  │  │ partner_debt_obligations                          │    │    │
│  │  │     ▼                                             │    │    │
│  │  │ efin_debt_rollup → efin_debt_pivot (159 columns)  │    │    │
│  │  └──────────┬────────────────────────────────────────┘    │    │
│  │             │                                             │    │
│  │  ┌─ Monitoring Hub ────────────────────────────────┐      │    │
│  │  │ int_tpg_efin_all_monitoring_base (160+ columns)  │      │    │
│  │  │  Assembles: fees + volume + debt + dimensions    │      │    │
│  │  └──────────────────────────────────────────────────┘      │    │
│  │                                                            │    │
│  │  ┌─ Advance Loans ─────────┐  ┌─ Disbursements ────────┐  │    │
│  │  │ fca_waterfall            │  │ taxpayer_disbursements  │  │    │
│  │  │ fca_collections          │  │ entity_disbursements    │  │    │
│  │  │ fca_disbursements        │  │ partner_disb_status     │  │    │
│  │  └──────────────────────────┘  └────────────────────────┘  │    │
│  │                                                            │    │
│  │  ┌─ Partner Ledger ────────┐  ┌─ Other ─────────────────┐ │    │
│  │  │ partner_waterfall        │  │ partner_hierarchy        │ │    │
│  │  │ fee_collections          │  │ transmitter_fee_perf     │ │    │
│  │  └──────────────────────────┘  │ rt_clearing_recon        │ │    │
│  │                                │ jh_fee_advance_events    │ │    │
│  └────────────────────────────────└──────────────────────────┘ │    │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  MARTS (19 reports)                                             │
│                                                                 │
│  Views (thin projections):                                      │
│    rpt_efin_all_monitoring, rpt_fca_info_v2,                    │
│    rpt_jh_fee_advance_detail/summary, rpt_efin_collections_v2   │
│                                                                 │
│  Tables (complex assembly):                                     │
│    rpt_rt_clearing (UNION ALL of apps + disbursements),         │
│    rpt_transmitter_deals_pro (multi-year + seed-based matching) │
│                                                                 │
│  Consumers: Tableau dashboards, MicroStrategy (legacy),         │
│             Republic Bank (JH fee advance)                      │
└─────────────────────────────────────────────────────────────────┘
```

## Flagship Report: EFIN All Monitoring

The most complex report in the pipeline. 160+ columns covering every EFIN's operational health:

```
rpt_efin_all_monitoring  [VIEW]
    │
    └── int_tpg_efin_all_monitoring_base  [TABLE]
            │
            ├── int_tpg_efin_fee_rollup
            │       └── int_tpg_partner_fee_obligations
            │               └── int_tpg_application_product_enhanced
            │
            ├── int_tpg_efin_volume_funded
            │       └── int_tpg_application_product_enhanced
            │
            ├── int_tpg_efin_debt_pivot  (159 columns)
            │       └── int_tpg_efin_debt_rollup
            │               └── int_tpg_partner_debt_obligations
            │
            ├── tpg.partnerrole, tpg.partner  (company name)
            ├── tpg.partnerproduct, tpg.partnerrolehierarchy  (type, isjh)
            └── tpg.vw_partnerrolecontact → pdr_stg.decryptaes()  (PII)
```

**What it replaced:** A single 500+ line legacy SQL script (Script 01) that had no reusable components, no tests, and no documentation. Now the same output is produced by 6 composable models, each independently testable and documented.

## Materialization Strategy

| Layer | Strategy | Rationale |
|-------|----------|-----------|
| Staging | `view` | Thin passthrough, no compute cost |
| All 20 intermediates | `table` | Heavy joins and aggregations; pre-computed once, reused by many marts |
| Most mart reports | `view` | Thin SELECT over pre-computed intermediate tables |
| RT Clearing, Transmitter Deals | `table` | Multi-step UNION ALL or complex aggregations too heavy for views |

## Column-Level Audit Framework

I built a Jinja-based audit framework to validate the migration:

```sql
-- audit_rpt_efin_all_monitoring.sql
-- Compares ~130+ columns between legacy and dbt output
-- Produces match_pct per column, sorted worst-first
```

This catches subtle differences: rounding, NULL handling, join fan-out, and business logic mismatches. Every column that doesn't match 100% gets investigated.

## Partner Hierarchy Extraction

The `partnerrolehierarchy` table has 6 levels. I scan each level to extract role identities:

```sql
-- In int_tpg_application_product_enhanced:
MAX(CASE WHEN partnerroletypekeylevel1 = 1 THEN partnerroleidentityvaluelevel1
         WHEN partnerroletypekeylevel2 = 1 THEN partnerroleidentityvaluelevel2
         WHEN partnerroletypekeylevel3 = 1 THEN partnerroleidentityvaluelevel3
         WHEN partnerroletypekeylevel4 = 1 THEN partnerroleidentityvaluelevel4
         WHEN partnerroletypekeylevel5 = 1 THEN partnerroleidentityvaluelevel5
         WHEN partnerroletypekeylevel6 = 1 THEN partnerroleidentityvaluelevel6
    END) AS ero_efin

-- Same pattern for: SB (type=3), Transmitter (type=5), SuperSB (type=4), SubSB (type=9)
```

This pattern appears in the foundation model and is reused by every downstream model that needs EFIN hierarchy.

## Filing Year Parameterization

```sql
{{ config(materialized='table', tags=['tpg','partner','waterfall']) }}

-- Parameterized filing year
WHERE filing_year = {{ var('filing_year', 2026) }}
```

The `var('filing_year')` default advances each tax season. All models inherit the same parameter, ensuring the pipeline produces a consistent snapshot.
