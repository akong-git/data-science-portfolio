# LLM Context: DS Feature Store

## SYSTEM CONTEXT
You are analyzing the DS Feature Store — a production ML feature engineering pipeline built with dbt on Redshift. It serves weekly behavioral features for risk, fraud, and churn models.

## ARCHITECTURE SUMMARY
- **Spine**: `dsfs_aw_base` creates account × week grid (key: `acct_uid_week_key`)
- **Account base**: `dsfs_a_base` identifies eligible active accounts
- **Feature models**: 7 domain-specific models join to spine and aggregate
- **Rolling windows**: Every metric available at 1w, 5w, 10w granularity
- **Materialization**: Incremental merge on `acct_uid_week_key`

## MODEL MAP

| Model | Grain | Key Features |
|-------|-------|-------------|
| `dsfs_a_base` | acct_uid | product_cleanup, business_division, division |
| `dsfs_aw_base` | acct_uid_week_key | tenure, lifetime status counts, days since last activity |
| `dsfs_aw_txn_summary` | acct_uid_week_key | DD amounts, GDV, reload, ACH, active days |
| `dsfs_aw_posted` | acct_uid_week_key | Purchase amounts, ATM, fees, liquid/non-liquid |
| `dsfs_aw_disputes` | acct_uid_week_key | Dispute counts, fraud/non-fraud, granted ratios |
| `dsfs_aw_auth_failure` | acct_uid_week_key | Auth failure counts, suspected fraud, last success |
| `dsfs_aw_ivr` | acct_uid_week_key | IVR call counts by type |
| `dsfs_aw_swipe` | acct_uid_week_key | Swipe fee amounts, revenue |
| `dsfs_a_address` | acct_uid | City, state geographic features |

## KEY CONVENTIONS
- Week key: `DATEDIFF(week, '2023-12-31', calendar_date)`
- Composite key: `{acct_uid}_{week_key}`
- Date parameters: `get_feature_store_dates()` macro
- CTE pattern: weekly_range → acct_list → all_combos → weekly_agg → rolling_window
- All COALESCE'd to 0 (no NULLs in feature output)

## COMMON QUERIES
- "What features are available for fraud detection?" → `dsfs_aw_disputes` (fraud amounts, granted ratios) + `dsfs_aw_auth_failure` (suspected fraud counts)
- "How is churn predicted?" → `dsfs_aw_txn_summary` (declining active days) + `dsfs_aw_ivr` (increasing support calls)
- "What's the grain?" → One row per account per week (`acct_uid_week_key`)
- "How are rolling windows computed?" → SQL window functions: ROWS BETWEEN N PRECEDING AND CURRENT ROW
