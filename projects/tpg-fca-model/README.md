# TPG FCA Credit Risk Model

## What I Built

An end-to-end ML feature engineering pipeline for predicting advance loan losses in SBTPG's tax-season financial products. Built entirely from scratch -- from raw IRS tax return data and Experian credit bureau responses to production-ready binned features consumed by a trained credit risk model.

This project spans two systems:
1. **Credit risk feature pipeline** (data science): 9 dbt models producing 500+ features from IRS data, credit bureau, and 3 years of filing history, distilled to 12 binned features for ML scoring
2. **Advance loan operations** (analytics engineering): 7+ dbt models tracking the full lifecycle of advance loans -- disbursement, daily collections, outstanding balances, and exposure monitoring

## Business Context

SBTPG processes ~20M tax returns annually. For a subset of filers, TPG provides advance loans -- cash before the IRS refund arrives. The risk: if the refund is smaller than expected (IRS offset, partial payment, fraud), the advance isn't fully repaid.

I built the feature pipeline to predict which loans will result in losses, enabling the underwriting team to make better approval decisions.

## Feature Engineering Highlights

- **200+ IRS form field features** pivoted from EAV storage (W-2 wages, 1040 fields, Schedule EIC, Schedule C)
- **30+ Experian credit bureau attributes** integrated from two source systems
- **50 knockout business rules** encoded (minimum age, deceased check, prior-year debt, withholding ratio thresholds)
- **3-year cross-year features:** YoY deltas for EIC, wages, prep fees; refund coverage analysis; preparer switching detection
- **Financial ratios:** EIC utilization (`r_eic2maxeic`), refund accuracy (`r_actual2expectedirsrefund`), withholding consistency (`w_ratio`)
- **12 production-ready binned features** with ordinal bins matching the trained model's expected input format

## Architecture

### Credit Risk Feature Pipeline

```mermaid
flowchart TD
    subgraph Sources["Data Sources"]
        IRS["IRS Tax Returns\n(EAV format)"]
        EXP["Experian\nCredit Bureau"]
        APP["Application\nData"]
        PSC["PSC Scoring\n(S3 external)"]
    end

    subgraph Intermediate["6 Intermediate Models"]
        APPL["int_tpg_fca_applicants\n─────────────────\nApplicant universe\nFiling year scoping"]
        TAX["int_tpg_fca_tax_returns\n─────────────────\n200+ IRS fields pivoted\nW-2, 1040, Sched EIC/C"]
        CRED["int_tpg_fca_credit_bureau\n─────────────────\n30+ Experian attributes\nDual source integration"]
        KNOCK["int_tpg_fca_knockout_rules\n─────────────────\n50 business rules\nMin age, deceased, debt"]
        ACH["int_tpg_fca_ach\n─────────────────\nACH routing data"]
        PSCI["int_tpg_fca_psc\n─────────────────\nPSC target variable\nExternal model score"]
    end

    subgraph CrossYear["Prior-Year Features"]
        YOY["3-Year Lookback\n──────────────────\nYoY deltas: EIC, wages, fees\nRefund coverage analysis\nPreparer switching detection"]
    end

    subgraph Assembly["Feature Assembly"]
        FEAT["500+ Raw Features\n──────────────────\nFinancial ratios\nEIC utilization\nWithholding consistency"]
    end

    subgraph Output["Production Output"]
        BIN["12 Binned Features\n──────────────────\nSQL CASE WHEN bins\nOrdinal encoding\nModel-ready format"]
        ML["ML Credit Risk Model\n──────────────────\nAdvance loan\napproval decisions"]
    end

    IRS --> TAX
    EXP --> CRED
    APP --> APPL
    PSC --> PSCI
    APPL --> Assembly
    TAX --> Assembly
    CRED --> Assembly
    KNOCK --> Assembly
    ACH --> Assembly
    PSCI --> Assembly
    TAX --> CrossYear
    Assembly --> FEAT
    CrossYear --> FEAT
    FEAT --> BIN
    BIN --> ML

    style Sources fill:#2d3748,stroke:#4a5568,color:#e2e8f0
    style Intermediate fill:#2c5282,stroke:#2b6cb0,color:#bee3f8
    style CrossYear fill:#744210,stroke:#975a16,color:#fefcbf
    style Assembly fill:#2f855a,stroke:#38a169,color:#c6f6d5
    style Output fill:#9b2c2c,stroke:#c53030,color:#fed7d7
```

### Advance Loan Operations Pipeline

```mermaid
flowchart LR
    subgraph Input["Input"]
        APP2["Application\nData"]
    end

    subgraph Waterfall["Waterfall Model"]
        WF["Dual Collection Paths\n───────────────────\nRefund waterfall\nDisbursement fee waterfall"]
    end

    subgraph Lifecycle["Loan Lifecycle"]
        DISB["Disbursements\nA / B / C patterns"]
        COLL["Collections\nUNION both paths"]
        BAL["Outstanding\nBalances"]
    end

    subgraph Reports["Reporting"]
        DAILY["Daily Reports"]
        TRIAL["Trial Balance"]
        REPAID["Fully-Repaid\nTracking"]
    end

    APP2 --> WF
    WF --> DISB
    WF --> COLL
    DISB --> BAL
    COLL --> BAL
    BAL --> DAILY
    BAL --> TRIAL
    BAL --> REPAID

    style Input fill:#2d3748,stroke:#4a5568,color:#e2e8f0
    style Waterfall fill:#553c9a,stroke:#6b46c1,color:#e9d8fd
    style Lifecycle fill:#2c5282,stroke:#2b6cb0,color:#bee3f8
    style Reports fill:#2f855a,stroke:#38a169,color:#c6f6d5
```

See [architecture.md](architecture.md) for detailed DAGs and waterfall mechanics.

## Key Technical Decisions

1. **SQL-native feature engineering:** All 500+ features computed in dbt/SQL rather than Python. This keeps the pipeline in a single system, makes features auditable, and eliminates data movement between platforms.

2. **Incremental tax return processing:** Tax return attributes use `incremental` materialization with compound unique key `(identitytoken, taxyear)`. This handles the 200+ column pivot efficiently -- only new/updated returns are reprocessed.

3. **Dual collection path handling:** Advance loan collections come from two waterfall sources (refund waterfall + disbursement fee waterfall). Missing either path understates collections by 10-20%. I UNION both sources in `int_tpg_fca_collections`.

4. **Binning in SQL, not Python:** The 12 production features are discretized via CASE WHEN in SQL. Fixed bin boundaries match the trained model exactly -- no floating-point drift, no library dependencies, fully auditable.

5. **Three-year lookback design:** The applicant universe spans 3 filing years (2023-2025) to enable prior-year feature computation. Cross-year self-joins produce YoY deltas for EIC, wages, refund coverage, and preparer switching.

## Impact

The feature pipeline feeds SBTPG's advance loan underwriting decisions. The 12 final features -- spanning prior-year IRS payment history, EIC utilization, filing timing, self-employment indicators, withholding ratios, and refund accuracy -- capture the strongest loss predictors identified during model training.

## Technology

- **Platform:** dbt on Amazon Redshift
- **Sources:** `greendot.tpg` (80+ tables), `greendot.datascience`, `gbos_vip` (Experian)
- **Scale:** 3 years of filing data, all loan applications, 200+ pivoted IRS attributes

## Documentation

- [Architecture](architecture.md) -- Three waterfall systems, dual DAGs, filing year scoping
- [Model Catalog](model-catalog.md) -- All 15 models with grain and key logic
- [Business Rules](business-rules.md) -- Waterfall mechanics, 50 knockout rules, PSC scoring, EIC caps
- [Technical Patterns](technical-patterns.md) -- Tax return pivoting, Experian integration, cross-year features, binning
- [LLM Context](llm-context.md) -- AI-readable summary
