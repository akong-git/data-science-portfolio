# TPG Operations Pipeline -- Legacy to Modern

## What I Built

I migrated SBTPG's entire operational reporting suite from 19+ monolithic MicroStrategy freeform SQL scripts to a modular dbt pipeline on Redshift. Built everything from scratch: 3 staging models, 20 intermediate models, 19 mart reports, a column-level audit framework, and comprehensive documentation.

This wasn't a lift-and-shift. I decomposed the legacy scripts into reusable intermediate models, identified shared logic that was copy-pasted across scripts, built a proper layered architecture, and added 4 new reports that had no legacy equivalent.

## Scale

- **124+ source tables** from `greendot.tpg`
- **42 dbt models** (3 staging + 20 intermediate + 19 marts)
- **19+ legacy scripts migrated** with column-level audit validation
- **4 new reports** built from scratch (JH Fee Advance detail/summary, EFIN Collections, Return to IRS)
- **160+ column flagship report** (`rpt_efin_all_monitoring`)
- **159-column debt pivot** (53 categories x 3 fields)
- **600-line business context document** written for onboarding

## Architecture

```mermaid
flowchart TD
    subgraph Sources["Sources (124+ tables)"]
        TBL[(greendot.tpg\n124+ tables)]
    end

    subgraph Staging["Staging Layer (3 views)"]
        S1["stg_tpg_application"]
        S2["stg_tpg_partner"]
        S3["stg_tpg_product"]
    end

    subgraph Intermediate["Intermediate Layer (20 tables)"]
        direction LR
        subgraph Core["Core Enrichment"]
            APP_E["Application\nEnrichment"]
            VOL["Volume &\nFunding"]
        end
        subgraph Financial["Financial Pipelines"]
            FEE["Fee Pipeline\n(dual-source)"]
            DEBT["Debt Pipeline\n(53-cat x 3 = 159 cols)"]
        end
        subgraph Advance["Advance Loan"]
            LIFE["Lifecycle\nTracking"]
            DISB["Disbursement\nA / B / C"]
        end
        subgraph Other["Other"]
            LED["Partner\nLedger"]
            RT["RT Clearing\nReconciliation"]
            JH["JH Fee\nAdvance Events"]
        end
    end

    subgraph Marts["Mart Layer (19 reports)"]
        direction LR
        subgraph Flagship["Flagship"]
            EFIN["rpt_efin_all_monitoring\n──────────────────\n160+ columns"]
        end
        subgraph Reports["Report Groups"]
            FCA["FCA Reports (4)\nInfo, Trial Bal,\nDisb, Collections"]
            HOLD["Hold Reports (2)\nERO, Taxpayer"]
            FEES["Fee Reports (3)\nPayee, POS,\nCollections"]
        end
        subgraph Reports2["Report Groups (cont.)"]
            TRANS["Transmitter (3)\nFees, Deals,\nProducts"]
            RTC["RT Clearing"]
            EFINC["EFIN Collections"]
            JHR["JH Fee Advance (2)\nDetail, Summary"]
            RTIRS["Return to IRS"]
        end
    end

    subgraph Consumers["Consumers"]
        TAB["Tableau\nDashboards"]
        MSTR["MicroStrategy\n(Legacy)"]
        REPUB["Republic\nBank"]
    end

    TBL --> Staging
    Staging --> Intermediate
    Intermediate --> Marts
    EFIN --> TAB
    EFIN --> MSTR
    JHR --> REPUB
    Marts --> Consumers

    style Sources fill:#2d3748,stroke:#4a5568,color:#e2e8f0
    style Staging fill:#744210,stroke:#975a16,color:#fefcbf
    style Intermediate fill:#2c5282,stroke:#2b6cb0,color:#bee3f8
    style Marts fill:#2f855a,stroke:#38a169,color:#c6f6d5
    style Consumers fill:#9b2c2c,stroke:#c53030,color:#fed7d7
```

### Flagship Report Dependency Chain

```mermaid
flowchart BT
    FEES_OBL["int_tpg_partner_fee_obligations"]
    DEBT_OBL["int_tpg_partner_debt_obligations"]
    FEE_ROLL["int_tpg_efin_fee_rollup"]
    DEBT_PIV["int_tpg_efin_debt_pivot\n(159 columns)"]
    VOL_FUND["int_tpg_efin_volume_funded"]
    BASE["int_tpg_efin_all_monitoring_base\n──────────────────────────\n160+ columns (table)"]
    RPT["rpt_efin_all_monitoring\n──────────────────────────\n(view)"]

    FEES_OBL --> FEE_ROLL --> BASE
    DEBT_OBL --> DEBT_PIV --> BASE
    VOL_FUND --> BASE
    BASE --> RPT

    style BASE fill:#553c9a,stroke:#6b46c1,color:#e9d8fd
    style RPT fill:#2f855a,stroke:#38a169,color:#c6f6d5
    style FEE_ROLL fill:#2c5282,stroke:#2b6cb0,color:#bee3f8
    style DEBT_PIV fill:#2c5282,stroke:#2b6cb0,color:#bee3f8
    style VOL_FUND fill:#2c5282,stroke:#2b6cb0,color:#bee3f8
    style FEES_OBL fill:#744210,stroke:#975a16,color:#fefcbf
    style DEBT_OBL fill:#744210,stroke:#975a16,color:#fefcbf
```

See [architecture.md](architecture.md) for detailed diagrams and dependency chains.

## Key Technical Achievements

### 1. Decomposed Monolithic Scripts into Composable Models

The legacy EFIN All Monitoring script was 500+ lines of SQL with no shared logic. I decomposed it into 6 models:

```mermaid
flowchart BT
    FEE_OBL["int_tpg_partner_fee_obligations"]
    DEBT_OBL["int_tpg_partner_debt_obligations"]
    FEE_ROLL["int_tpg_efin_fee_rollup"]
    DEBT_PIV["int_tpg_efin_debt_pivot\n(159 columns)"]
    VOL_FUND["int_tpg_efin_volume_funded"]
    BASE["int_tpg_efin_all_monitoring_base\n(table, 160+ columns)"]
    RPT["rpt_efin_all_monitoring\n(view)"]

    FEE_OBL --> FEE_ROLL --> BASE
    DEBT_OBL --> DEBT_PIV --> BASE
    VOL_FUND --> BASE
    BASE --> RPT
```

Each intermediate model is independently testable and reused by multiple mart reports.

### 2. Column-Level Audit Framework

I built a Jinja-based audit model (`audit_rpt_efin_all_monitoring.sql`) that compares ~130+ columns between the legacy source-of-truth and the new dbt output. It produces `match_pct` per column, sorted worst-first, so mismatches surface immediately.

### 3. Dual Fee Source Normalization

Fee data comes from two waterfall families (refund waterfall + disbursement fee waterfall). The legacy scripts handled this inconsistently. I normalized both into `int_tpg_partner_fee_obligations` with standardized payee roles, ensuring all downstream reports use the same fee logic.

### 4. 159-Column Debt Pivot

Partner debts span 53 categories, each with 3 fields (amount, amountcollected, balancedue). I pivot this from long format to 159 wide columns in `int_tpg_efin_debt_pivot`, matching the exact column names the legacy MicroStrategy report expects.

### 5. Seed-Based Transmitter Classification

Transmitter deals require matching combinations of transmitter name, product code, sub-product, and disbursement method to reporting categories. I externalized the ~35 matching rules into a CSV seed file (`ref_transmitter_reporting_product_logic`), so business users can update classifications without touching SQL.

### 6. JH Fee Advance (New Capability)

Built entirely new reporting for Jackson Hewitt P-Center fee advances, combining 3 data sources (CAP limits, advance disbursements, collection waterfall repayments) into daily running balance reports consumed by Republic Bank. Includes a 641-line reference document with raw SQL, validation queries, domain glossary, and risk log.

## Legacy Migration Map

| Script | Legacy Name | dbt Replacement |
|--------|------------|-----------------|
| 01 | EFIN All Monitoring | `rpt_efin_all_monitoring` |
| 02 | ERO Holds | `rpt_ero_holds` |
| 03 | FCA Info V2 | `rpt_fca_info_v2` |
| 05-08 | FCA Trial Balance, Disbursements, Collections | `rpt_fcb_*` (4 reports) |
| 10-12 | Transmitter Fees, Deals | `rpt_transmitter_*` (3 reports) |
| 13 | RT Clearing | `rpt_rt_clearing` |
| 14 | Taxpayer Holds | `rpt_taxpayer_holds` |
| 15-17 | Fee by Payee, POS | `rpt_*_fee*` (3 reports) |
| 19 | Fee Collections | `int_tpg_fee_collections` |

See [legacy-migration-map.md](legacy-migration-map.md) for the full mapping with migration notes.

## Technology

- **Platform:** dbt on Amazon Redshift
- **Source system:** `greendot.tpg` (SBTPG production database, 124+ tables)
- **Consumers:** Tableau dashboards, MicroStrategy (legacy), Republic Bank
- **PII:** Encrypted contact info decrypted via `pdr_stg.decryptaes()` in mart reports

## Documentation

- [Architecture](architecture.md) -- Three-layer design, flagship report dependency chain
- [Model Catalog](model-catalog.md) -- All 42 models with grain and descriptions
- [Legacy Migration Map](legacy-migration-map.md) -- Script-to-model mapping with migration notes
- [LLM Analyst Design](llm-analyst-design.md) -- Vision for natural language querying of TPG ops data
- [LLM Context](llm-context.md) -- AI-readable summary
