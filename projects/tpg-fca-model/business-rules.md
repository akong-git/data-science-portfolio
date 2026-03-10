# TPG FCA Model -- Business Rules

This document captures the critical business rules encoded in the FCA model pipeline. I reverse-engineered many of these from the production database and underwriting team discussions, then formalized them in the Implementation Programming Specifications shared with Model Risk Management.

---

## 1. Model Scope and Target Variable

The FCA 2026 Underwriting Model predicts the probability of "No-IRS-Pay" -- where the IRS refund is insufficient to repay the advance loan.

**PSC (Performance Scoring Category) definition:**

| PSC Code | Meaning | Model Target |
|----------|---------|--------------|
| 10 | Full-IRS-Pay, low risk | 0 (Good) |
| 12 | Full-IRS-Pay, moderate risk | 0 (Good) |
| 30 | No-IRS-Pay, partial recovery | 1 (Bad) |
| 40 | No-IRS-Pay, no recovery | 1 (Bad) |
| Others | Out of scope (partials, rejects, etc.) | Excluded |

**Model scope = PSC in (10, 12, 30, 40) AND denial_cat in (0, 1)**

The 3-year payment history (`hist_3y`) is encoded as a 3-character string where each character represents one year's IRS payment outcome. `hist_2y` = first 2 characters.

---

## 2. Product Classification

Tax-season advance loans are classified by timing relative to the IRS acknowledgment (ACK):

| Product | Timing | Principal Key | Interest Key |
|---------|--------|--------------|-------------|
| FCA (Funded Cash Advance) | Post-ACK | 1 | 2 |
| ERA (Early Refund Advance) | Pre-ACK | 142 | 143 |
| RA (Refund Advance) | Variable | 144 | 145 |

Loan tiers for FCA 2026:
- **Non-JH:** $500, $750, $1,000, $1,500, $2,000, $3,000, $4,000, $5,500, $7,000
- **Jackson Hewitt:** $300, $500, $750, $1,000, $1,500, $2,500, $3,500

Maximum loan typically does not exceed ~75% of the expected federal refund.

---

## 3. Fifty Knockout Rules

The `int_tpg__knockout_and_scoring` model encodes 50 business rules sourced from `loanapplicantevaluationdetail` that immediately disqualify a loan application before model scoring occurs.

### Sample Rules (representative, not exhaustive)

| Rule Category | What It Checks |
|--------------|----------------|
| Minimum Age | Applicant must be 18+ |
| Deceased Check | Experian/SSA death index match |
| Prior Year FCA Debt | Outstanding balance from previous year's advance |
| Income Requirements | Minimum federal anticipated refund amount |
| Schedule C Limits | Self-employment return count thresholds |
| Forms Restrictions | Certain IRS form combinations disqualify |
| Multi-Bank Account | Too many accounts across different banks |
| Withholding Ratio | W-2 withholding / total income must be within bounds |
| Estimated Tax Payments | Excessive estimated payments flag |
| OFAC Match | Treasury Department sanctions list hit |
| Fraud Check | Pattern match against known fraud indicators |
| Military Status | Special handling for military applicants |

If `isknockoutrulehit = 1`, the application is denied regardless of model score. Up to 2 knockout rules are concatenated into a `knockoutrule` field, with up to 4 denial reasons stored separately.

---

## 4. Filing Year Scoping

| Scope | Years | Rationale |
|-------|-------|-----------|
| Applicant universe | FY 2023, 2024, 2025 | 3-year window for prior-year feature lookback |
| Tax return attributes | All years matching applicant universe | Tax return data joined with 1-year lag (filingyear - 1 = taxyear) |
| Final feature output | FY 2025 only | Production model scored for current season |

Key convention: Filing Year 2026 = Tax Year 2025 (returns filed in early 2026 for income earned in 2025).

---

## 5. Prior-Year IRS Payment History

Derived from `applicationadvance` in `int_tpg__fca_applicants`:

```sql
CASE
    WHEN prioryearfederalanticipatedrefundamount > 0
         AND prioryearfederalactualrefundamount >= prioryearfederalanticipatedrefundamount
    THEN 'F'  -- Full (IRS paid full expected amount or more)

    WHEN prioryearfederalanticipatedrefundamount > 0
         AND prioryearfederalactualrefundamount > 0
    THEN 'P'  -- Partial (IRS paid something, but less than expected)

    WHEN prioryearfederalanticipatedrefundamount > 0
    THEN 'N'  -- None (IRS paid nothing)

    ELSE 'U'  -- Unknown
END AS py_rrec_hist
```

This is one of the strongest predictive features. An applicant whose prior-year refund was fully paid poses much lower risk than one with partial or zero payment.

The `py_rrec_hist` feature bins this as: Full → 3, Partial → 2, else → 1.

---

## 6. JH vs Non-JH Branching

Several features have different source logic depending on whether the applicant filed through Jackson Hewitt:

**`r_actual2expectedirsrefund_prev` (TY2024):**
- **Non-JH:** Pull actual/expected IRS refund from TPG database for TY2024
- **JH:** Pull `RREC_PY_IRSPAY` / `RREC_PY_REFUND` from Jackson Hewitt data

Both paths share the same binning: sentinel values (-2 for zero/null expected, -1 for null expected), ratio capped at 1.0, then 6 ordinal bins with boundaries at 0.823 and 0.979.

**`r_actual2expectedirsrefund_prev2` (TY2023):**
Same JH vs non-JH split. Different bin boundaries: 0.757 and 0.95.

---

## 7. EIC Cap Lookup

The Earned Income Credit has statutory maximums by number of qualifying children. I maintain a lookup table in `mrt_tpg__fca_model_features`:

| Tax Year | 0 Children | 1 Child | 2 Children | 3+ Children |
|----------|-----------|---------|------------|-------------|
| 2022 | $560 | $3,733 | $6,164 | $6,935 |
| 2023 | $600 | $3,995 | $6,604 | $7,430 |
| 2024 | $632 | $4,213 | $6,960 | $7,830 |
| 2025 | $649 | $4,328 | $7,152 | $8,046 |

`r_eic2maxeic` = actual EIC / max EIC for their child count and tax year. Values close to 1.0 indicate the applicant is claiming near the maximum, which combined with other features can signal suspicious returns.

If max_eic is NULL, default to $649 (0-child cap). EIC > max_eic maps to bin 1 (default).

---

## 8. EIC-Aware Sentinel Values (pct_standard_children)

The `pct_standard_children` feature uses special sentinel bin values for edge cases rather than treating them as missing:

| Condition | Bin Value | Interpretation |
|-----------|-----------|---------------|
| EIC = 0 AND no dependents (depcode1 is null) | 1 | No EIC, no dependents -- baseline |
| EIC = 0 AND has dependents | 2 | Dependents present but no EIC claimed |
| EIC > 0 AND no dependents (depcode1 is null) | 3 | EIC claimed without listed dependents |
| EIC > 0 AND has dependents | Compute ratio, bin 4-6 | Standard relationship % drives bin |

The ratio computation: `standard_cnt / (standard_cnt + non_standard_cnt)` where "NONE" dependents (null/blank) are excluded from the denominator.

---

## 9. Score Interpretation and Usage

The SageMaker endpoint returns P(no-IRS-pay) between 0 and 1. Converted to a 1-999 score:

```
Score = ROUND((1 - probability) * 1000)
```

Higher score = lower risk. Usage:

| Decision | How Score Is Used |
|----------|------------------|
| **Approval/Denial** | Score cutoffs by segment. In simulation, profitability crossover at ~score 273 for $500 minimum loan. |
| **Loan Assignment** | Higher scores qualify for larger loan tiers. Max loan capped at ~75% of expected federal refund. |
| **In-Season Adjustment** | Cutoffs may shift based on observed refund rates, fraud trends, or IRS changes during tax season. |

The worst-scoring 5% had a 17.2% No-IRS-Pay rate vs 1.3% for the best 5% -- a 13x separation factor.

---

## 10. Transmitter Segmentation

```sql
CASE
    WHEN transmitterefin = '68547' THEN 'JH'      -- Jackson Hewitt
    ELSE 'Non-JH'
END AS transmitter_segment
```

JH franchise locations have different risk profiles, fee structures, and loan tiers than independent preparers. This segmentation partitions the portfolio for separate risk analysis and feature computation (see JH vs non-JH branching above).
