# Feature Store — Architecture

## Pipeline Overview

The feature store follows a three-layer architecture designed for ML consumption:

```mermaid
graph LR
    subgraph "Layer 1: Account Identification"
        A["dsfs_a_base<br/>Which accounts?"]
    end

    subgraph "Layer 2: Spine Generation"
        B["dsfs_aw_base<br/>Account × Week grid"]
    end

    subgraph "Layer 3: Feature Aggregation"
        C["dsfs_aw_txn_summary"]
        D["dsfs_aw_posted"]
        E["dsfs_aw_disputes"]
        F["dsfs_aw_auth_failure"]
        G["dsfs_aw_ivr"]
        H["dsfs_aw_swipe"]
    end

    A --> B
    B --> C & D & E & F & G & H
```

## Layer 1: Account Identification (`dsfs_a_base`)

**Purpose**: Define the universe of accounts eligible for feature generation.

**Logic**:
- Start with `dim_account` — all accounts in the system
- Inner join to accounts with active transactions in the last 365 days
- Enrich with product/division metadata from multiple dimension tables
- Apply business filters:
  - Status = Normal only
  - Business divisions: BaaS, Direct, Retail
  - Exclude legacy/deprecated brands

**Materialization**: Incremental merge on `acct_uid` — only processes accounts with updates since last run.

## Layer 2: Spine Generation (`dsfs_aw_base`)

**Purpose**: Create a complete account × week grid so every feature domain has consistent time windows.

**Why a spine matters for ML**:
- Without a spine, missing weeks appear as NULLs → requires imputation downstream
- With a spine, missing activity explicitly becomes zeros → cleaner features
- All domains align on the same week boundaries → no temporal misalignment

**Week Key Convention**:
```
week_key = DATEDIFF(week, '2023-12-31', calendar_date)
```

This produces a monotonically increasing integer, making it easy to compute rolling windows:
- 1-week window: `WHERE week_key BETWEEN current_week - 1 AND current_week`
- 5-week window: `WHERE week_key BETWEEN current_week - 5 AND current_week`
- 10-week window: `WHERE week_key BETWEEN current_week - 10 AND current_week`

**Composite Key**:
```
acct_uid_week_key = CAST(acct_uid AS VARCHAR) || '_' || CAST(week_key AS VARCHAR)
```

## Layer 3: Feature Aggregation (`dsfs_aw_*`)

Every feature domain model follows the same CTE pattern:

```mermaid
graph TD
    A["weekly_range CTE<br/>Define week boundaries from dim_date"] --> D
    B["acct_list CTE<br/>Accounts with activity in lookback window"] --> D
    D["all_combos CTE<br/>CROSS JOIN: every account × every week"] --> E
    E["weekly_agg CTE<br/>LEFT JOIN source data, aggregate to 1-week"] --> F
    F["rolling_window CTE<br/>SUM OVER windows: 1w, 5w, 10w"]
```

**Key design decisions:**
- **CROSS JOIN** ensures every account gets a row for every week (even inactive weeks)
- **LEFT JOIN** to source data means inactive weeks get COALESCE'd to 0
- **Window functions** compute rolling aggregations efficiently in a single pass

## Date Parameter System

All feature models use the same centralized macro:

```
get_feature_store_dates() → {
    start_date:           configurable start
    end_date:             configurable end
    look_back_start_date: end_date - 10 weeks (for rolling windows)
}
```

This ensures:
- All domains cover the same date range
- No training/serving skew from misaligned date boundaries
- Easy to reconfigure for backfills or date range changes

## Data Flow Diagram

```mermaid
flowchart TB
    subgraph "EDW Sources"
        S1[fct_posted_transaction]
        S2[fct_dly_acct_txn_summary]
        S3[fct_authorization_transaction]
        S4[fct_service_sale]
        S5[dim_account]
        S6[dim_date]
    end

    subgraph "Other Sources"
        S7[int_disputes__claim_level]
        S8[int_ivr__calls_id]
        S9[bps_account_details]
    end

    subgraph "Base"
        B1[dsfs_a_base]
        B2[dsfs_aw_base]
    end

    subgraph "Features"
        F1[dsfs_aw_txn_summary]
        F2[dsfs_aw_posted]
        F3[dsfs_aw_disputes]
        F4[dsfs_aw_auth_failure]
        F5[dsfs_aw_ivr]
        F6[dsfs_aw_swipe]
    end

    S5 & S9 --> B1
    B1 & S6 --> B2

    B2 & S2 --> F1
    B2 & S1 --> F2
    B2 & S7 --> F3
    B2 & S3 --> F4
    B2 & S8 --> F5
    B2 & S4 --> F6
```
