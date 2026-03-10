# TPG FCA Model (Full-service Collect Agreement)

## Summary

The **TPG FCA model** captures the end-to-end lifecycle of **tax refund advance loans** (Full-service Collect Agreements). It models funding, repayment, and collection flows, including **principal vs interest** tracking and the **dual waterfall** collection paths used by TPG.

This project emphasizes two things:
1. **Business correctness** (financial logic, waterfalled collections, filing-year scoping)
2. **Operational usability** (clear grains, consistent definitions, report-ready outputs)

## My Role: Built from scratch

I designed and built this FCA model from scratch in dbt:
- Defined grains and canonical keys
- Implemented dual-path collection logic
- Standardized principal vs interest semantics
- Built report-ready outputs for operations and monitoring

## What “FCA” Means (Business Context)

- **FCA**: Full-service Collect Agreement — a loan/advance that is repaid via tax refund flows
- **Principal owed/collected**: The base loan amount and its repayment
- **Interest owed/collected**: Interest amount and repayment
- **Waterfall collections**: Repayment is applied through structured waterfall detail tables

## Key Outputs (At a glance)

| Output | Model | Grain | Purpose |
|--------|-------|-------|---------|
| Loan lifecycle / balances | `int_tpg_fca_waterfall` | `applicationkey` | One row per loan with owed/collected amounts and repay status |
| Daily collections | `int_tpg_fca_collections` | `applicationkey + collection_date` | Daily principal/interest collections for monitoring + rollups |
| Fully collected list | `rpt_fca_info_v2` | `applicationkey` | Report: where principal collected = principal owed |

## Key Design Decisions

1. **Explicit separation of principal vs interest**
   - Principal and interest are modeled independently to avoid mixing financial measures.

2. **Dual waterfall sources**
   - Collections can come from both:
     - Refund waterfall details
     - Disbursement fee waterfall details

3. **Deterministic “collection_date”**
   - Use CashReceiptJournal `ProcessDate` when available, otherwise fall back to `EffectiveDate - 1 day`.

4. **Filing-year scoping**
   - Reports and partner hierarchy dependencies must support configurable `filing_year` (e.g., `var('filing_year', 2025)`).

## Documentation

- [Architecture](architecture.md)
- [Model Catalog](model-catalog.md)
- [Business Rules](business-rules.md)
- [Technical Patterns](technical-patterns.md)
- [LLM Context](llm-context.md)
