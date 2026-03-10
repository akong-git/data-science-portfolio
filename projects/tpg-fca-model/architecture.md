# TPG FCA Model Architecture

## Overview

This is a credit risk feature engineering pipeline built in dbt on Redshift for SBTPG's FCA 2026 Underwriting Model. It extracts ~350 raw features from IRS tax return data, Experian credit bureau responses, application data, and prior-year filing history, then distills them into 12 binned features consumed by an XGBoost model deployed on AWS SageMaker.

I designed the architecture, built all 9 dbt models, wrote the Implementation Programming Specifications, and co-authored the Final Model Documentation with the Head of Tax Refund.

## Business Context

SBTPG provides advance loans (FCA, ERA, RA) to taxpayers before their IRS refund arrives. The core risk: if the refund is smaller than expected, the loan isn't fully repaid.

**Products and template attribute keys:**

| Product | Timing | Principal Key | Interest Key |
|---------|--------|--------------|-------------|
| FCA (Funded Cash Advance) | Post-ACK | 1 | 2 |
| ERA (Early Refund Advance) | Pre-ACK | 142 | 143 |
| RA (Refund Advance) | Variable | 144 | 145 |

**Target variable definition:**
- **1 (Bad/No-IRS-Pay):** PSC 30 or 40 -- expected refund < 40% of actual refund
- **0 (Good/Full-IRS-Pay):** PSC 10 or 12 -- expected refund >= 90% of actual refund
- **Model scope:** PSC in (10, 12, 30, 40) AND denial_cat in (0, 1)

## Pipeline DAG

```mermaid
flowchart TD
    subgraph Sources["Source Systems"]
        direction LR
        S1["tpg.taxreturnheader\ntpg.taxreturndetail\ntpg.taxreturnattribute"]
        S2["tpg.VendorProgramResponse\ngbos_vip.vendorprogramresponse"]
        S3["tpg.application\ntpg.loanapplication\ntpg.applicationadvance"]
        S4["datascience.fca2025_06_psc"]
        S5["tpg.consumerprofile\ntpg.loanapplicantevaluationdetail"]
    end

    subgraph Intermediate["Intermediate Layer (6 models, all TABLE)"]
        APPL["int_tpg__fca_applicants\n──────────────────\nGrain: (identitytoken, filingyear, applicationkey)\nBase universe FY 2023-2025\nPrior-year IRS payment history (F/P/N/U)\nLatest app per identity per year"]
        TAX["int_tpg__tax_return_attributes\n──────────────────\nGrain: (identitytoken, taxyear)\nIncremental materialization\n200+ pivoted IRS form fields\nW-2, 1040, Sched EIC/C/SE"]
        CRED["int_tpg__experian_attributes\n──────────────────\nGrain: identitytoken\n30+ P13 credit attributes\nDual source: tpg + gbos_vip\nCOALESCE best available"]
        KNOCK["int_tpg__knockout_and_scoring\n──────────────────\nGrain: applicationkey\n50 knockout rules\nSHAP features, model scores\nDenial reasons (up to 4)"]
        ACH["int_tpg__ach_and_fees\n──────────────────\nGrain: applicationkey\nFederal ACH, state ACH\nPreparation fees"]
        PSC["int_tpg__psc_file\n──────────────────\nGrain: identitytoken\nhist_2y, in_model_scope\nTarget: 1=loss, 0=non-loss"]
    end

    subgraph Marts["Mart Layer (3 models, all TABLE)"]
        FEAT["mrt_tpg__fca_model_features\n──────────────────\nGrain: (identitytoken, app_filingyear)\n500+ columns: all intermediates joined\nDerived ratios: r_eic2maxeic,\nr_actual2expectedirsrefund, w_ratio\nEIC cap lookup by year × child count"]
        PY["mrt_tpg__fca_model_prior_year_features\n──────────────────\nGrain: identitytoken (current FY)\n3-year cross-year self-joins\nYoY deltas: EIC, wages, prep fees\nRefund coverage, preparer switching"]
        FINAL["mrt_tpg__fca_model_final_features\n──────────────────\nGrain: identitytoken (FY2025, in-scope)\n12 binned features via CASE WHEN\nOrdinal encoding matching trained model\nScope: FY=2025, PSC in (10,12,30,40)"]
    end

    subgraph Scoring["AWS SageMaker"]
        EP["XGBoost Endpoint\nP(no-IRS-pay) → Score 1-999"]
    end

    S1 --> TAX
    S2 --> CRED
    S3 --> APPL
    S3 --> ACH
    S4 --> PSC
    S5 --> KNOCK
    APPL --> FEAT
    TAX --> FEAT
    CRED --> FEAT
    KNOCK --> FEAT
    ACH --> FEAT
    PSC --> FEAT
    FEAT --> PY
    PY --> FINAL
    FINAL --> EP

    style Sources fill:#2d3748,stroke:#4a5568,color:#e2e8f0
    style Intermediate fill:#2c5282,stroke:#2b6cb0,color:#bee3f8
    style Marts fill:#2f855a,stroke:#38a169,color:#c6f6d5
    style Scoring fill:#9b2c2c,stroke:#c53030,color:#fed7d7
```

## Materialization Strategy

| Model | Strategy | Rationale |
|-------|----------|-----------|
| All intermediates (except tax returns) | `table` with `grant select on {{ this }} to public` | Pre-compute for wide downstream joins; public access for analytics team |
| `int_tpg__tax_return_attributes` | `incremental` (unique_key: identitytoken + taxyear) | Largest dataset (200+ columns × millions of rows), avoids full recompute |
| Feature assembly marts | `table` | 500+ column joins need materialized intermediate results |
| `mrt_tpg__fca_model_final_features` | `table` | Production model input; must be deterministic and auditable |

## Filing Year Scoping

| Scope | Years | Rationale |
|-------|-------|-----------|
| Applicant universe | FY 2023, 2024, 2025 | 3-year window for prior-year feature lookback |
| Tax return attributes | All years matching applicant universe | Tax return joined with 1-year lag (filingyear - 1 = taxyear) |
| Cross-year features | FY 2023 vs 2024 vs 2025 | YoY deltas require all 3 years |
| Final feature output | FY 2025 only | Production model scored for current season |

Key convention: Filing Year 2026 = Tax Year 2025 (returns filed in early 2026 for income earned in 2025).

## Real-Time Scoring Flow

```mermaid
flowchart LR
    ERO["ERO/Transmitter\nsends tax data"]
    TPG["TPG extracts\n12 features"]
    GW["SageMaker\nAPI Gateway"]
    EP["XGBoost\nEndpoint"]
    PROB["P(no-IRS-pay)"]
    SCORE["Score =\nROUND((1-p)×1000)"]
    UW["Underwriting\nDecision"]

    ERO --> TPG --> GW --> EP --> PROB --> SCORE --> UW

    style ERO fill:#2d3748,stroke:#4a5568,color:#e2e8f0
    style TPG fill:#2c5282,stroke:#2b6cb0,color:#bee3f8
    style GW fill:#553c9a,stroke:#6b46c1,color:#e9d8fd
    style EP fill:#553c9a,stroke:#6b46c1,color:#e9d8fd
    style PROB fill:#744210,stroke:#975a16,color:#fefcbf
    style SCORE fill:#744210,stroke:#975a16,color:#fefcbf
    style UW fill:#2f855a,stroke:#38a169,color:#c6f6d5
```

The score drives two downstream decisions:
1. **Approval/Denial:** Score cutoffs determine which applicants are approved. In simulation, the profitability crossover from denial to approval occurs around score 273.
2. **Loan tier assignment:** Approved applicants are assigned loan amounts ($300-$7,000) based on score band and expected refund range. Higher scores qualify for larger loans.
