# Feature Store — Feature Domains

## Overview

Each feature domain focuses on a distinct behavioral signal. All domains share the same account-week spine and produce features at the `acct_uid_week_key` grain with consistent rolling windows.

## Rolling Window Strategy

| Window | Lookback | ML Purpose |
|--------|----------|------------|
| **1w** | Current week | Recency signals — what happened this week? |
| **5w** | ~1 month | Short-term trends — recent behavioral shifts |
| **10w** | ~2.5 months | Medium-term patterns — sustained behavior |
| **Lifetime** | All history | Long-term baseline — overall account profile |

---

## Domain 1: Transaction Summary (`dsfs_aw_txn_summary`)

**Source**: `fct_dly_acct_txn_summary` — daily pre-aggregated transaction summaries

**ML Signals:**
- **Engagement**: `n_days_active` — how many days was the account active?
- **Income proxy**: `amt_dd` (direct deposit) — indicates regular income
- **Spending velocity**: GDV + reload + ACH amounts show fund movement
- **Savings behavior**: `amt_savings` indicates financial planning

**Why it matters for ML**: Transaction frequency and amount patterns are the strongest predictors of churn and account health. A drop in `n_days_active` from 5w to 1w is a classic churn signal.

---

## Domain 2: Posted Transactions (`dsfs_aw_posted`)

**Source**: `fct_posted_transaction` + `dim_txn_type_hierarchy` + `dim_mcc`

**ML Signals:**
- **Spending categories**: Liquid vs non-liquid purchases, crypto transactions
- **Cash behavior**: ATM withdrawal frequency and amounts
- **Fee sensitivity**: Fee amounts indicate cost burden on the account
- **Transaction mix**: Ratio of purchases to withdrawals to fees

**Categorization Logic:**
- Liquid purchases identified by MCC codes (convenience stores, gas stations)
- Crypto purchases identified by merchant name patterns
- Reversals netted against positive transactions for accurate totals

---

## Domain 3: Disputes (`dsfs_aw_disputes`)

**Source**: `int_disputes__claim_level` — processed dispute claims

**ML Signals:**
- **Risk indicator**: High dispute counts = higher fraud risk
- **Fraud classification**: `amt_dispute_fraud` vs `amt_dispute_non_fraud`
- **Resolution patterns**: `ratio_dispute_granted` — high granted ratio may indicate legitimate victim vs abuser
- **Lifetime accumulation**: Lifetime dispute metrics capture long-term risk profile

**Unique feature**: This is the only domain with **lifetime aggregations** alongside rolling windows — critical because dispute history has long-term predictive power.

---

## Domain 4: Authorization Failures (`dsfs_aw_auth_failure`)

**Source**: `fct_authorization_transaction` + `dim_auth_reason`

**ML Signals:**
- **Fraud proxy**: `n_suspected_fraud` — direct fraud signal from auth system
- **Account health**: High `n_auth_failure` rate may indicate compromised card
- **Recency**: `ind_last_auth_successful` — was the most recent auth attempt successful?

**Why it matters**: Auth failures are leading indicators. They happen before disputes and before customer calls. A spike in auth failures often precedes a fraud event by 1-2 weeks.

---

## Domain 5: IVR Calls (`dsfs_aw_ivr`)

**Source**: `int_ivr__calls_id` — IVR call events with account mapping

**ML Signals:**
- **Support need**: Total call volume indicates account friction
- **Call type mix**: Registration vs support vs gift card calls reveal intent
- **Authorization status**: Whether the caller was authenticated

**Why it matters**: IVR calls are a proxy for customer friction. Accounts with increasing support calls are more likely to churn. Registration calls after a period of inactivity may indicate re-engagement.

---

## Domain 6: Swipe Fees (`dsfs_aw_swipe`)

**Source**: `fct_service_sale` — service sale transactions

**ML Signals:**
- **Revenue contribution**: `amt_swipe_fee` — total fee revenue from this account
- **Company revenue**: `amt_revenue_swipe_fee` — the company's portion

**Why it matters**: Swipe fee activity is a proxy for card usage at point-of-sale. Declining swipe fees can indicate reduced card usage before formal churn.

---

## Domain 7: Address / Geographic (`dsfs_a_address`)

**Source**: Customer profile data

**ML Signals:**
- **Geographic risk**: State-level risk scoring
- **Urban/rural proxy**: City classification for segmentation
- **Identity verification**: Address consistency checks

**Note**: This is an **account-level** (not account-week) feature. It enriches the feature vector with static demographic information.

---

## Feature Cross-Domain Patterns

The real power of the feature store comes from **combining features across domains**:

| Pattern | Domains Combined | ML Signal |
|---------|-----------------|-----------|
| Dispute spike + Auth failures | Disputes + Auth Failure | Strong fraud indicator |
| Declining active days + Increasing IVR calls | Txn Summary + IVR | Churn warning |
| High DD amount + Low purchase activity | Txn Summary + Posted | Dormant account risk |
| Geographic outlier + Suspected fraud | Address + Auth Failure | Location-based fraud |
