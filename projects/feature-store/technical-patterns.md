# Feature Store — Technical Patterns

## Core Pattern: Spine-Based Feature Generation

Every `dsfs_aw_*` model follows the same CTE architecture:

### Step 1: Define Week Boundaries
```
weekly_range CTE:
  - Query dim_date for the lookback period
  - Group by week_key to get week_start_date and week_end_date
  - Uses: DATEDIFF(week, '2023-12-31', calendar_date)
```

### Step 2: Identify Active Accounts
```
acct_list CTE:
  - Find accounts with activity in the lookback window
  - INNER JOIN to dsfs_aw_base to limit to eligible accounts
```

### Step 3: Generate Complete Grid
```
all_combos CTE:
  - CROSS JOIN acct_list × weekly_range
  - Creates every possible account-week combination
  - This is what ensures no missing rows
```

### Step 4: Aggregate Source Data
```
weekly_agg CTE:
  - LEFT JOIN source data to the complete grid
  - Aggregate to 1-week granularity
  - COALESCE nulls to 0 (inactive weeks get explicit zeros)
  - Build the composite key: acct_uid_week_key
```

### Step 5: Compute Rolling Windows
```
rolling_window CTE:
  - SUM/AVG OVER (PARTITION BY acct_uid ORDER BY week_key ROWS BETWEEN N PRECEDING AND CURRENT ROW)
  - Produces: *_1w, *_5w, *_10w variants of each feature
```

---

## Centralized Date Parameters

The `get_feature_store_dates()` macro returns:

| Parameter | Description | Purpose |
|-----------|-------------|---------|
| `start_date` | Configurable start | Defines feature generation window |
| `end_date` | Configurable end | Usually current date |
| `look_back_start_date` | end_date - 10 weeks | Lookback for rolling windows |

**Why centralized?** All 8+ domain models reference the same dates. If one model used a different date range, features would be misaligned — a subtle but devastating ML bug.

---

## Incremental Strategy

All feature models use:
```
materialized='incremental'
incremental_strategy='merge'
unique_key='acct_uid_week_key'
```

**How incremental works here:**
1. On first run: builds the full lookback window
2. On subsequent runs: only processes weeks since last run
3. Merge strategy: upserts — if a week's data is reprocessed (late-arriving data), it overwrites cleanly

**Why merge over append?** Late-arriving transactions (posted after the original date) would create duplicates with append. Merge ensures exactly one row per account-week.

---

## Week Key Design

```
week_key = DATEDIFF(week, '2023-12-31', calendar_date)
```

**Why this anchor date?**
- `2023-12-31` is a Sunday — aligns with Sunday-to-Saturday week boundaries
- Produces clean integer keys starting from 1 for the first week of 2024
- Monotonically increasing — makes window functions simple

**Composite key construction:**
```
acct_uid_week_key = CAST(acct_uid AS VARCHAR) || '_' || CAST(week_key AS VARCHAR)
```

This creates human-readable keys like `12345_52` (account 12345, week 52).

---

## Performance Considerations

| Decision | Rationale |
|----------|-----------|
| **Lifetime metrics in separate models** | `dsfs_aw_txn_lifetime` isolated to prevent slow queries |
| **CROSS JOIN size management** | acct_list filters to recently active accounts only |
| **COALESCE to zero** | Avoids NULL handling downstream in ML pipelines |
| **Incremental merge** | Only processes new/changed weeks, not full history |

---

## Complete Example: Rolling Window Feature Model

Below is a sanitized, representative feature domain model showing the full spine-based pattern end-to-end. This is the pattern every `dsfs_aw_*` model follows:

```sql
-- dsfs_aw_disputes (simplified)
-- Grain: one row per account per week
-- Output: dispute counts and amounts at 1w, 5w, 10w rolling windows

{{ config(
    materialized='incremental',
    incremental_strategy='merge',
    unique_key='acct_uid_week_key'
) }}

{% set dates = get_feature_store_dates() %}

-- Step 1: Define week boundaries from date dimension
WITH weekly_range AS (
    SELECT
        DATEDIFF(week, '2023-12-31', calendar_date) AS week_key,
        MIN(calendar_date) AS week_start_date,
        MAX(calendar_date) AS week_end_date
    FROM {{ ref('dim_date') }}
    WHERE calendar_date BETWEEN '{{ dates.look_back_start_date }}'
                              AND '{{ dates.end_date }}'
    GROUP BY 1
),

-- Step 2: Identify accounts with recent activity
acct_list AS (
    SELECT DISTINCT acct_uid
    FROM {{ ref('dsfs_aw_base') }}
    WHERE week_key >= DATEDIFF(week, '2023-12-31', '{{ dates.start_date }}')
),

-- Step 3: Cross join to create complete account × week grid
all_combos AS (
    SELECT a.acct_uid, w.week_key, w.week_start_date, w.week_end_date
    FROM acct_list a
    CROSS JOIN weekly_range w
),

-- Step 4: Aggregate source events to weekly grain
weekly_agg AS (
    SELECT
        ac.acct_uid,
        ac.week_key,
        CAST(ac.acct_uid AS VARCHAR) || '_' || CAST(ac.week_key AS VARCHAR)
            AS acct_uid_week_key,
        COALESCE(COUNT(d.dispute_id), 0)           AS dispute_cnt,
        COALESCE(SUM(d.dispute_amount), 0)         AS dispute_amt,
        COALESCE(SUM(CASE WHEN d.is_fraud THEN 1 ELSE 0 END), 0)
            AS fraud_dispute_cnt,
        COALESCE(SUM(CASE WHEN d.is_granted THEN d.dispute_amount ELSE 0 END), 0)
            AS granted_amt
    FROM all_combos ac
    LEFT JOIN {{ source('edw', 'fct_disputes') }} d
        ON ac.acct_uid = d.acct_uid
       AND d.dispute_date BETWEEN ac.week_start_date AND ac.week_end_date
    GROUP BY 1, 2, 3
),

-- Step 5: Compute rolling windows
rolling_window AS (
    SELECT
        acct_uid_week_key,
        acct_uid,
        week_key,

        -- 1-week (current week only)
        dispute_cnt                                AS dispute_cnt_1w,
        dispute_amt                                AS dispute_amt_1w,
        fraud_dispute_cnt                          AS fraud_dispute_cnt_1w,

        -- 5-week rolling
        SUM(dispute_cnt) OVER (
            PARTITION BY acct_uid ORDER BY week_key
            ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
        ) AS dispute_cnt_5w,
        SUM(dispute_amt) OVER (
            PARTITION BY acct_uid ORDER BY week_key
            ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
        ) AS dispute_amt_5w,

        -- 10-week rolling
        SUM(dispute_cnt) OVER (
            PARTITION BY acct_uid ORDER BY week_key
            ROWS BETWEEN 9 PRECEDING AND CURRENT ROW
        ) AS dispute_cnt_10w,
        SUM(fraud_dispute_cnt) OVER (
            PARTITION BY acct_uid ORDER BY week_key
            ROWS BETWEEN 9 PRECEDING AND CURRENT ROW
        ) AS fraud_dispute_cnt_10w

    FROM weekly_agg
)

SELECT * FROM rolling_window

{% if is_incremental() %}
WHERE week_key >= (SELECT MAX(week_key) - 1 FROM {{ this }})
{% endif %}
```

This pattern ensures: no missing weeks (cross join), no NULLs (COALESCE), consistent date boundaries (centralized macro), and efficient refreshes (incremental merge).
