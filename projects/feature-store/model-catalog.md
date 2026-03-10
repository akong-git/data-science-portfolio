# Feature Store — Model Catalog

## Model Inventory

| Model | Layer | Grain | Materialization | Primary Key | Description |
|-------|-------|-------|----------------|-------------|-------------|
| `dsfs_a_base` | Base | Account | Incremental (merge) | `acct_uid` | Active eligible accounts with product/division enrichment |
| `dsfs_aw_base` | Base | Account-Week | Table | `acct_uid_week_key` | Account × week spine with tenure and lifetime metrics |
| `dsfs_a_address` | Feature | Account | Table | `acct_uid` | Geographic features from customer profile |
| `dsfs_aw_txn_summary` | Feature | Account-Week | Incremental (merge) | `acct_uid_week_key` | Weekly transaction summary metrics |
| `dsfs_aw_posted` | Feature | Account-Week | Incremental (merge) | `acct_uid_week_key` | Posted transaction amounts, counts, fees |
| `dsfs_aw_disputes` | Feature | Account-Week | Incremental (merge) | `acct_uid_week_key` | Dispute/chargeback events and resolution |
| `dsfs_aw_auth_failure` | Feature | Account-Week | Incremental (merge) | `acct_uid_week_key` | Authorization failure events and fraud indicators |
| `dsfs_aw_ivr` | Feature | Account-Week | Incremental (merge) | `acct_uid_week_key` | IVR call events by type |
| `dsfs_aw_swipe` | Feature | Account-Week | Incremental (merge) | `acct_uid_week_key` | Swipe fee revenue metrics |

---

## Detailed Model Descriptions

### `dsfs_a_base` — Account Base

| Attribute | Value |
|-----------|-------|
| **Grain** | One row per `acct_uid` |
| **Materialization** | Incremental (merge on `acct_uid`) |
| **Refresh** | Incremental based on `update_dttm` |

**Key Columns:**

| Column | Description |
|--------|-------------|
| `acct_uid` | Account unique identifier (PK) |
| `update_dttm` | Last update timestamp |
| `init_funding_dttm_pt` | Initial funding date (Pacific time) |
| `proc_opened_dttm_pt` | Account opened date (Pacific time) |
| `business_division` | BaaS, Direct, or Retail |
| `division` | Division classification |
| `product_cleanup` | Cleaned product name (e.g., GO2bank, QuickBooks Cash) |

**Filters Applied:**
- `acct_status = 'Normal'`
- `business_division IN ('BaaS', 'Direct', 'Retail')`
- Excludes: Walmart Gift, legacy brands (Uber GoBank, Rush Card, etc.)
- Must have active transactions in last 365 days

---

### `dsfs_aw_base` — Account-Week Spine

| Attribute | Value |
|-----------|-------|
| **Grain** | One row per `acct_uid` × `week_key` |
| **Materialization** | Table (full rebuild) |
| **Primary Key** | `acct_uid_week_key` = `{acct_uid}_{week_key}` |

**Key Columns:**

| Column | Description |
|--------|-------------|
| `acct_uid_week_key` | Composite PK |
| `acct_uid` | Account identifier |
| `week_key` | Numeric week: `DATEDIFF(week, '2023-12-31', calendar_date)` |
| `week_start_date` | Sunday start of week |
| `week_end_date` | Saturday end of week |
| `tenure_init_funding` | Days since initial funding (as of week end) |
| `tenure_opened` | Days since account opened (as of week end) |
| `n_blocked_lifetime` | Lifetime count of Blocked status |
| `n_restricted_lifetime` | Lifetime count of Restricted status |
| `n_pending_lifetime` | Lifetime count of Pending status |
| `n_closed_lifetime` | Lifetime count of Closed status |
| `n_days_from_last_active` | Days since last active txn (capped at 366) |
| `n_days_from_last_purch` | Days since last purchase (capped at 366) |
| `n_days_from_last_fee` | Days since last fee transaction |

---

### `dsfs_aw_txn_summary` — Transaction Summary

| Attribute | Value |
|-----------|-------|
| **Grain** | One row per `acct_uid_week_key` |
| **Materialization** | Incremental (merge) |

**Features (× 3 windows: 1w, 5w, 10w):**

| Feature | Description |
|---------|-------------|
| `n_days_active` | Days with any active transaction |
| `amt_dd` | Direct deposit amount |
| `amt_gdv` | GDV amount |
| `amt_reload` | Reload transaction amount |
| `amt_ach` | ACH transfer amount |
| `amt_savings` | Savings transaction amount |

---

### `dsfs_aw_posted` — Posted Transactions

| Attribute | Value |
|-----------|-------|
| **Grain** | One row per `acct_uid_week_key` |
| **Materialization** | Incremental (merge) |

**Features (× 3 windows: 1w, 5w, 10w):**

| Feature | Description |
|---------|-------------|
| `amt_funds_out` | Funds out (net of reversals) |
| `n_purchase` / `amt_purchase` | Purchase count and amount |
| `n_atm` / `amt_atm` | ATM withdrawal count and amount |
| `amt_fee` | Fee amount |
| `n_purchase_liquid` | Liquid purchase count |
| `n_purchase_non_liquid` | Non-liquid purchase count |
| `n_purchase_crypto` | Crypto purchase count |

---

### `dsfs_aw_disputes` — Dispute Events

**Features:**

| Feature | Window | Description |
|---------|--------|-------------|
| `n_dispute_claim` | 1w, 5w, 10w | Distinct dispute claim count |
| `n_dispute` | 1w, 5w, 10w | Total dispute volume |
| `amt_dispute` | 1w, 5w, 10w | Total dispute dollar amount |
| `amt_dispute_fraud` | 1w, 5w, 10w | Fraud-classified amount |
| `amt_dispute_non_fraud` | 1w, 5w, 10w | Non-fraud amount |
| `n_dispute_granted` | 1w, 5w, 10w | Granted count |
| `ratio_dispute_granted` | 1w, 5w, 10w | Granted / total ratio |
| `n_dispute_claim_lifetime` | Lifetime | All-time claims |
| `amt_dispute_lifetime` | Lifetime | All-time amount |

---

### `dsfs_aw_auth_failure` — Authorization Failures

**Features (× 3 windows: 1w, 5w, 10w):**

| Feature | Description |
|---------|-------------|
| `n_auth_failure` | Auth failures (reason ≠ SUCCESS) |
| `n_suspected_fraud` | Reason = SUSPECTED_FRAUD |
| `ind_last_auth_successful` | Was last auth in the week successful? (boolean) |

---

### `dsfs_aw_ivr` — IVR Call Events

**Features (× 3 windows: 1w, 5w, 10w):**

| Feature | Description |
|---------|-------------|
| `n_ivr_calls` | Total IVR call count |
| `n_ivr_calls_authorized` | Authorized calls |
| `n_ivr_calls_registration` | Registration calls |
| `n_ivr_calls_support` | Support calls |
| `n_ivr_calls_giftcard` | Gift card calls |
| `n_ivr_calls_other` | Other calls |

---

### `dsfs_aw_swipe` — Swipe Fee Revenue

**Features (× 3 windows: 1w, 5w, 10w):**

| Feature | Description |
|---------|-------------|
| `amt_swipe_fee` | Total swipe fee amount |
| `amt_revenue_swipe_fee` | Revenue portion |

---

### `dsfs_a_address` — Geographic Features

| Column | Description |
|--------|-------------|
| `acct_uid` | Account identifier (PK) |
| `sor_uid` | System of record ID |
| `consumer_profile_uid` | Consumer profile ID |
| `city` | City from address |
| `state_province` | State/province from address |

---

## Schema Testing

All models have schema-level tests:
- **Primary keys**: `unique` + `not_null` on `acct_uid_week_key` (or `acct_uid`)
- **Required columns**: `not_null` on `week_key`, `week_start_date`, `week_end_date`
