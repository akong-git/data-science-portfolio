# Copilot Instructions — Andy Kong Portfolio

You are assisting with **Andy Kong’s Data Science portfolio** repository.

## Repo goal
Help Andy present his work as a **Data Scientist who builds production-grade data infrastructure**, including:
- ML feature engineering / feature store design
- operational analytics pipelines (dbt)
- LLM analyst design on top of curated reporting surfaces

## Repo structure (important)
- `projects/ods-edw-build/` — ODS & EDW architecture, QA framework, stakeholder communication
- `projects/feature-store/` — Feature store architecture, domains, patterns, and an `llm-context.md`
- `projects/tpg-fca-model/` — FCA loan lifecycle + collections modeling (built from scratch)
- `projects/tpg-operations-pipeline/` — Ops reporting pipeline modernization (freeform SQL → dbt) + LLM analyst vision
- `projects/gdb-giftcard-breakage/` — Gift card breakage revenue modeling (ASC 606)

## Content rules
- Do **not** include proprietary SQL from employer repositories.
- Do **not** include credentials, schema names, connection strings, or internal hostnames.
- Prefer describing:
  - grains, keys, and business definitions
  - model interfaces (“inputs/outputs”)
  - QA/reconciliation patterns
  - lineage and maintainability strategy

## “LLM readiness” conventions
Some files are designed specifically to be used as AI grounding context:
- `llm-context.md` files provide:
  - model maps
  - glossary
  - question → model routing guidance
  - output requirements (show SQL, list assumptions, cite source model)

When Andy asks for help building an LLM analyst workflow, use these context files as the source of truth.

## Domain glossary (quick)
- **EFIN**: Electronic Filing Identification Number
- **ERO**: Electronic Return Originator
- **FCA**: Full-service Collect Agreement (tax refund advance loan)
- **RT clearing**: Operational reconciliation surface for tax product funding/disbursement flows
- **Filing year**: The tax filing year (often parameterized)

## Preferred writing style
- Portfolio-quality: clear, concise, technical but readable
- Use tables for model inventories and grains
- Use mermaid diagrams where it helps explain architecture
- Emphasize Andy’s role:
  - Feature store: co-built
  - FCA model: built from scratch
  - TPG ops pipeline: built from scratch
