# TPG FCA Model -- Technical Patterns

Deep dive into the engineering patterns I built for the FCA credit risk feature pipeline.

---

## 1. Tax Return Attribute Pivoting

### Problem

IRS tax return data is stored in EAV (Entity-Attribute-Value) format: one row per (tax return, attribute). A single taxpayer's return can have 200+ attribute rows covering W-2 wages, 1040 fields, Schedule EIC children, and more. ML models need this data in wide format: one row per taxpayer with each attribute as a column.

### Solution: `int_tpg__tax_return_attributes`

```sql
-- Join tax return details to attribute definitions
FROM tpg.taxreturnheader trh
JOIN tpg.taxreturndetail trd ON trh.taxreturnheaderkey = trd.taxreturnheaderkey
JOIN tpg.taxreturnattribute tra ON trd.taxreturnattributekey = tra.taxreturnattributekey

-- Pivot via conditional aggregation
SELECT
    identitytoken,
    taxyear,
    MAX(CASE WHEN tra.taxreturnattributekey = 71
             THEN trd.value END) AS irs1040_totalincomeamount,
    MAX(CASE WHEN tra.taxreturnattributekey = 55
             THEN trd.value END) AS irs1040_earnedincomecreditamount,
    SUM(CASE WHEN tra.taxreturnattributekey = 170
             THEN trd.value::numeric END) AS irsw2_socialsecuritywagesamount,
    -- ... 200+ more columns
    COUNT(DISTINCT CASE WHEN tra.taxreturnattributekey = 133
                        THEN trd.value END) AS cnt_irs1040scheduleeic_qualifyingchildssn,
    ...
GROUP BY identitytoken, taxyear
```

### Incremental Strategy

This is the largest model in the pipeline. Full refresh would be prohibitively slow, so I use incremental materialization:

```sql
{{ config(
    materialized='incremental',
    unique_key=['identitytoken', 'taxyear']
) }}
```

New or updated tax returns are merged into the existing table. The compound unique key ensures one row per taxpayer per tax year.

### Dependent Child Profiling

A key innovation for the `pct_standard_children` feature: I pivot `TaxReturnAttributeKey = 66` (DependentRelationshipCode) by `documentsequenceid` 1/2/3 to get the first three dependents, then classify each as STANDARD (son/daughter/grandchild), NON-STAN (other relationships), or NONE (null/blank).

```sql
-- Pivot first 3 dependents
MAX(CASE WHEN taxreturnattributekey = 66 AND documentsequenceid = 1
         THEN taxreturnattributevalue END) AS irs1040_dependentrelationshipcode1,
MAX(CASE WHEN taxreturnattributekey = 66 AND documentsequenceid = 2
         THEN taxreturnattributevalue END) AS irs1040_dependentrelationshipcode2,
MAX(CASE WHEN taxreturnattributekey = 66 AND documentsequenceid = 3
         THEN taxreturnattributevalue END) AS irs1040_dependentrelationshipcode3
```

---

## 2. Experian Credit Bureau Integration

### Challenge

Credit bureau data comes from two source systems (`tpg.VendorProgramResponse` and `gbos_vip.vendorprogramresponse`) with different schemas but overlapping identity tokens. The data arrives as key-value pairs of vendor response attributes.

### Solution: `int_tpg__experian_attributes`

```sql
-- Connect applicant identitytoken to vendor request via external identifier
FROM int_tpg__fca_applicants app
JOIN tpg.consumerprofile cp ON ...
JOIN tpg.vendorprogramrequestexternalidentifier vprei ON ...
JOIN tpg.VendorProgramRequest vpr ON ...
JOIN tpg.VendorProgramResponse vpresp ON ...
JOIN tpg.VendorResponseAttribute vra ON ...

-- Pivot response attributes to columns
SELECT
    identitytoken,
    MAX(CASE WHEN vra.name = 'VerificationResultCode' THEN vra.value END) AS verificationresultcode,
    MAX(CASE WHEN vra.name = 'P13_ALL_Bankcard_BalanceAmount' THEN vra.value END) AS p13_all_bankcard_balanceamount,
    -- ... 30+ P13 attributes across ALL, AUA, COL, IQT, REV, STJ, STU, UTI categories
```

Two separate subqueries handle the two source systems, then COALESCE selects the best available data for each attribute.

---

## 3. Cross-Year Feature Engineering

### Pattern: Year-Over-Year Deltas

The `mrt_tpg__fca_model_prior_year_features` model pivots the feature table across 3 filing years per identity token, then computes deltas:

```sql
-- Self-join across years
current_year AS (SELECT * FROM features WHERE app_filingyear = 2025),
prev_year    AS (SELECT * FROM features WHERE app_filingyear = 2024),
prev_year_2  AS (SELECT * FROM features WHERE app_filingyear = 2023)

SELECT
    curr.identitytoken,
    -- Year-over-year EIC change
    curr.eic_amount - prev.eic_amount AS delta_eic_current_vs_prev,
    CASE WHEN prev.eic_amount > 0
         THEN curr.eic_amount / prev.eic_amount
    END AS r_eic_current_vs_prev,

    -- Preparer switching
    CASE WHEN curr.transmitter_efin = prev.transmitter_efin THEN 1 ELSE 0
    END AS has_same_transmitter_efin_current_vs_prev,

    -- Refund coverage (did prior-year refund cover the advance?)
    prev.actual_refund - prev.loan_amount AS refund_coverage_2024
```

### LAG Window for EIC History

In the main feature table, I use LAG to compute year-over-year EIC change:

```sql
LAG(eic_amount) OVER (
    PARTITION BY identitytoken
    ORDER BY app_filingyear
) AS eic_last_year,

eic_amount - LAG(eic_amount) OVER (...) AS eic_yoy_change
```

### Design Decision

I chose self-join over LAG for the prior-year feature table because the delta computations span different feature sets (wages, prep fees, refund coverage) that each need their own prior-year lookup. A single LAG approach would require separate window functions per metric. The self-join is cleaner for 15+ cross-year features.

---

## 4. Financial Ratio Computation

I engineered several financial ratios as features because raw dollar amounts vary widely across the population, but ratios normalize behavior:

| Ratio | Formula | What It Captures |
|-------|---------|-----------------|
| `r_eic2maxeic` | EIC / max_eic_for_child_count | EIC utilization -- values near 1.0 at high child counts correlate with suspicious returns |
| `r_eic2expectedirsrefund` | EIC / expected_refund | EIC as share of total expected refund |
| `r_wages2totalincome` | W2_wages / total_income | Share of income from employment -- low values (with high refund) indicate potential fraud |
| `r_eic_ctc_2expectedirsrefund` | (EIC + additional_CTC) / expected_refund | Combined credit utilization |
| `r_actual2expectedirsrefund` | actual_irs_refund / expected_refund | Refund accuracy -- prior year values predict current year risk |
| `w_ratio` | total_withholding / max(total_income, wages, withholding) | Withholding consistency |

The EIC cap lookup table is hardcoded by year and child count, since statutory caps change annually (e.g., TY2025: $649/$4,328/$7,152/$8,046 for 0/1/2/3+ children).

---

## 5. Binning / Discretization for ML

### Problem

The trained XGBoost model expects ordinal categorical inputs, not continuous features. The bin boundaries were determined during model training and must be replicated exactly in the production feature pipeline.

### Solution: `mrt_tpg__fca_model_final_features`

```sql
-- Example: Social Security wages (15 bins)
CASE
    WHEN irsw2_socialsecuritywagesamount IN (-1)           THEN 1
    WHEN irsw2_socialsecuritywagesamount <= 4438.0         THEN 2
    WHEN irsw2_socialsecuritywagesamount <= 9031.8         THEN 3
    WHEN irsw2_socialsecuritywagesamount <= 10647.0        THEN 4
    ...
    WHEN irsw2_socialsecuritywagesamount > 71784.833       THEN 15
    ELSE 1  -- default
END AS irsw2_socialsecuritywagesamount_binned

-- Example: Withholding ratio (5 bins)
CASE
    WHEN w_ratio <= 0.01                          THEN 1
    WHEN w_ratio > 0.01 AND w_ratio <= 0.05       THEN 2
    WHEN w_ratio > 0.05 AND w_ratio <= 0.08       THEN 3
    WHEN w_ratio > 0.08 AND w_ratio <= 0.1        THEN 4
    WHEN w_ratio > 0.1                             THEN 5
    ELSE 1
END AS w_ratio_binned
```

Each of the 12 features has its own bin structure (ranging from 3 to 15 bins). NULL values get a designated default bin value. The bin boundaries are fixed -- they don't change with new data because they match the trained model's expectations.

### Default Values

Every feature has a documented default value for NULL/missing cases:

```sql
-- From Implementation Programming Specifications
-- risk_f default = 4, pct_standard_children default = 1, ...
COALESCE(risk_f_binned, 4) AS risk_f_binned,
COALESCE(pct_standard_children_binned, 1) AS pct_standard_children_binned
```

### Design Decision

I could have implemented binning in Python or a post-processing step. I chose SQL-native CASE WHEN for three reasons:
1. **No additional infrastructure:** Runs inside the existing dbt pipeline
2. **Auditable:** Every bin boundary is visible in the SQL, matching the Implementation Specs document shared with Model Risk Management
3. **Deterministic:** No floating-point issues or library version dependencies

---

## 6. Deduplication Patterns

### Latest Application Per Identity Per Year

```sql
ROW_NUMBER() OVER (
    PARTITION BY identitytoken, app_filingyear
    ORDER BY applicationkey DESC
) AS rn
-- Keep rn = 1
```

A taxpayer can file multiple times in the same year (amendments, refilings). I keep the latest application.

### First Loan Per Application

```sql
ROW_NUMBER() OVER (
    PARTITION BY applicationkey
    ORDER BY loanapplicationkey ASC
) AS loan_rn
-- Keep loan_rn = 1
```

An application can have multiple loan applications (if the first was denied and they reapplied). I keep the first loan to capture the initial underwriting decision.

---

## 7. JH vs Non-JH Feature Branching

Several features require different source data depending on whether the applicant filed through Jackson Hewitt (transmitterefin = '68547').

### Pattern: `r_actual2expectedirsrefund_prev`

```sql
-- Non-JH: use TPG database
CASE WHEN transmitter_segment = 'Non-JH' THEN
    CASE
        WHEN expected_refund_ty2024 IS NULL THEN -1
        WHEN expected_refund_ty2024 = 0 THEN -2
        WHEN actual_refund_ty2024 IS NULL THEN -2
        ELSE ROUND(actual_refund_ty2024 / expected_refund_ty2024, 5)
    END
-- JH: use transmitter-provided data
WHEN transmitter_segment = 'JH' THEN
    CASE
        WHEN rrec_py_refund IS NULL THEN -1
        WHEN rrec_py_refund = 0 THEN -2
        WHEN rrec_py_irspay IS NULL THEN -2
        ELSE ROUND(rrec_py_irspay / rrec_py_refund, 5)
    END
END

-- Cap ratio at 1.0
LEAST(ratio, 1.0)

-- Both branches share identical binning
CASE
    WHEN ratio = -2         THEN 1
    WHEN ratio = -1         THEN 2
    WHEN ratio = 1          THEN 6
    WHEN ratio <= 0.823     THEN 3
    WHEN ratio <= 0.979     THEN 4
    WHEN ratio < 1.0        THEN 5
    ELSE 2
END
```

This branching exists because JH provides their own prior-year refund data (`RREC_PY_IRSPAY/REFUND`) while non-JH applications rely on TPG's database records for the same information.
