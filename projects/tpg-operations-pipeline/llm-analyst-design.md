# LLM Analyst Design -- Vision Document

## The Opportunity

The TPG Operations pipeline produces 19 reports from 160+ columns of operational data. Today, this data is consumed through static Tableau dashboards and MicroStrategy reports. Operations analysts spend significant time writing ad-hoc SQL to answer questions that the prebuilt reports don't cover.

I envision an **LLM-powered analyst layer** on top of the dbt pipeline that lets TPG Operations users ask questions in natural language and get accurate, sourced answers backed by the production data models.

## What This Would Enable

**Today (manual):**
> "How many Jackson Hewitt P-Centers have outstanding fee advance balances over $50K?"

Analyst opens Redshift, writes a query against `int_tpg_jh_fee_advance_events`, remembers the column names, handles the CAP vs advance vs repayment logic, and returns results via email.

**With LLM Analyst:**
> User asks in natural language. The LLM translates to SQL using the semantic layer definitions, executes against the materialized tables, and returns a formatted answer with the underlying query visible for verification.

---

## Architecture: Semantic Layer + LLM

```
┌─────────────────────────────────────────────────────┐
│  USER INTERFACE                                      │
│  (Slack bot, web app, or CLI)                        │
│                                                      │
│  "Which transmitters have the highest collection     │
│   shortfall this season?"                            │
└───────────────────────┬─────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│  LLM LAYER (Claude / GPT-4)                         │
│                                                      │
│  1. Consult semantic model definitions               │
│  2. Identify relevant metrics and dimensions         │
│  3. Generate SQL against dbt mart models             │
│  4. Execute query                                    │
│  5. Format and explain results                       │
└───────────────────────┬─────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│  SEMANTIC LAYER (MetricFlow / dbt Semantic Models)   │
│                                                      │
│  Metrics:                                            │
│    - total_volume (sum of applications per EFIN)     │
│    - total_funded (federal + state funded count)     │
│    - fees_requested / fees_paid / fees_pending       │
│    - debt_outstanding (balance across 53 categories) │
│    - fca_principal_owed / collected / balance        │
│    - collection_shortfall (owed - collected)         │
│    - jh_fee_advance_cap / advanced / repaid          │
│                                                      │
│  Dimensions:                                         │
│    - efin, company_name, partner_type                │
│    - transmitter_efin, is_jh                         │
│    - filing_year, collection_date                    │
│    - business_line (Pro / Franchise)                 │
│    - product_type (FCA / ERA / RA / RT)              │
│                                                      │
│  Entities:                                           │
│    - efin (primary)                                  │
│    - transmitter                                     │
│    - application                                     │
│    - pcenter                                         │
└───────────────────────┬─────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│  dbt MART MODELS (materialized tables/views)         │
│                                                      │
│  rpt_efin_all_monitoring    (160+ columns)           │
│  int_tpg_fca_waterfall      (loan lifecycle)         │
│  int_tpg_jh_fee_advance_events (P-Center activity)   │
│  rpt_rt_clearing            (reconciliation)         │
│  ... (19 reports)                                    │
└─────────────────────────────────────────────────────┘
```

---

## Semantic Model Definitions

### Proposed MetricFlow Models

**1. EFIN Monitoring Semantic Model**

```yaml
semantic_models:
  - name: efin_monitoring
    model: ref('int_tpg_efin_all_monitoring_base')
    defaults:
      agg_time_dimension: filing_date

    entities:
      - name: efin
        type: primary
        expr: monitoring_efin

    dimensions:
      - name: partner_type
        type: categorical
      - name: is_jh
        type: categorical
      - name: transmitter_efin
        type: categorical
      - name: company_name
        type: categorical

    measures:
      - name: total_volume
        agg: sum
        expr: volume
      - name: total_funded
        agg: sum
        expr: numberfunded
      - name: total_fees_requested
        agg: sum
        expr: totalfeesrequested
      - name: total_debt_outstanding
        agg: sum
        expr: totalerobalancedue
```

**2. Advance Loan Semantic Model**

```yaml
semantic_models:
  - name: advance_loans
    model: ref('int_tpg_fca_waterfall')

    entities:
      - name: application
        type: primary
        expr: applicationkey
      - name: efin
        type: foreign
        expr: ero_efin

    measures:
      - name: total_principal_owed
        agg: sum
        expr: fca_principal_owed
      - name: total_principal_collected
        agg: sum
        expr: fca_principal_collected
      - name: total_outstanding_balance
        agg: sum
        expr: outstanding_balance
      - name: avg_days_since_disbursement
        agg: average
        expr: days_since_disbursement
```

**3. JH Fee Advance Semantic Model**

```yaml
semantic_models:
  - name: jh_fee_advance
    model: ref('int_tpg_jh_fee_advance_events')

    entities:
      - name: pcenter
        type: primary
        expr: pcenter_efin

    dimensions:
      - name: event_date
        type: time

    measures:
      - name: total_cap
        agg: sum
        expr: cap_amount
      - name: total_advanced
        agg: sum
        expr: advance_amount
      - name: total_repaid
        agg: sum
        expr: repayment_amount
```

---

## Example Questions the LLM Could Answer

### Operational Health
- "What's the total outstanding debt across all franchise EFINs?"
- "Which transmitters have funding ratios below 80%?"
- "Show me EFINs with more than $100K in JH B2B principal balance"

### Advance Loan Monitoring
- "How many FCA loans are still outstanding from January?"
- "What's the average days-since-disbursement for unfunded ERA loans?"
- "Compare daily collection totals this week vs last week"

### Fee Advance Tracking
- "How much CAP is remaining across all JH P-Centers?"
- "Which P-Centers have advanced more than 75% of their CAP?"
- "What's the daily repayment trend for the last 30 days?"

### Reconciliation
- "How many applications are stuck in RT clearing with 'RTC < Fees' reason?"
- "Show me the top 10 EFINs by clearing shortfall amount"

---

## Implementation Approach

### Phase 1: Semantic Model Definitions
Define MetricFlow semantic models for the 3 core areas (monitoring, advance loans, fee advance). This is the foundation -- the LLM needs well-defined metrics and dimensions to generate accurate SQL.

### Phase 2: LLM Context Documents
The `llm-context.md` files in each project section provide the LLM with domain knowledge it needs to understand questions in context. These feed into the system prompt or RAG retrieval.

### Phase 3: Query Generation + Guardrails
- LLM generates SQL against semantic layer (not raw tables)
- SQL is validated before execution (read-only, no DDL)
- Results are formatted with column descriptions and data lineage
- The generated SQL is always shown to the user for verification

### Phase 4: Feedback Loop
User corrections ("that's not what funding ratio means") update the semantic model descriptions, improving future accuracy.

---

## Why This Matters

The TPG Operations team generates the same ad-hoc queries repeatedly, with slight variations. An LLM layer doesn't replace analysts -- it eliminates the repetitive SQL writing for common questions and lets analysts focus on investigation and decision-making. The semantic layer ensures the LLM uses the same business definitions that the production reports use, maintaining consistency.

The existing dbt pipeline's table materialization means queries against mart models are fast (seconds, not minutes). The column-level documentation in `_tpg_operations_models.yml` provides the LLM with precise column descriptions for accurate query generation.
