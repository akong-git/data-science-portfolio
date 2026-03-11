# Andy Kong — Data Scientist

## 👋 About Me

Data Scientist with deep expertise in building production-grade data infrastructure — from ODS/EDW architecture to ML feature stores to real-time scoring pipelines. I own the full lifecycle: source system discovery, warehouse design, feature engineering, model deployment, and operational reporting. I don't just build models — I build the systems that make data science work at enterprise scale.

## What Sets Me Apart

Most data scientists consume feature stores. **I build them.**  
Most data scientists hand off models. **I productionize them.**  
Most data scientists avoid SQL. **I architect data warehouses and pipelines with it.**  
Most data scientists stay in notebooks. **I lead cross-functional builds and present to executives.**

## 🏗️ Projects

| Project | Role | Stack | Description |
|---------|------|-------|-------------|
| [ODS & EDW Build](projects/ods-edw-build/) | **Technical Lead** | Redshift, dbt | Designed and built the ODS + EDW layers — source discovery, schema design, QA framework, executive reporting |
| [DS Feature Store](projects/feature-store/) | Co-built | dbt, Redshift, Python | Weekly ML feature engineering pipeline — 13 feature domains, rolling window aggregations, account-week spine architecture |
| [TPG FCA Model](projects/tpg-fca-model/) | **Built from scratch** | dbt, Redshift, SageMaker | Tax refund advance underwriting — ~350 features distilled to 12, XGBoost scoring, $1.7B exposure |
| [TPG Operations Pipeline](projects/tpg-operations-pipeline/) | **Built from scratch** | dbt, Redshift | Migrated 19+ legacy MicroStrategy scripts into modular 42-model dbt pipeline with LLM analyst vision |
| [Gift Card Breakage Model](projects/gdb-giftcard-breakage/) | Built | Python, SageMaker, SQL | ASC 606 breakage estimation — Random Forest + Linear Regression, $2.7M revenue constraint, monthly retraining |

## 🤖 AI Vision

Building toward **LLM-powered operational analytics** — where domain experts can ask questions in plain English and get accurate, sourced answers from production data pipelines. Currently designing an LLM analyst layer for tax product operations.

## 🛠️ Core Skills

**Data Architecture:** ODS/EDW Design · Dimensional Modeling · Source-to-Warehouse Mapping · Data Quality Frameworks
**Data Science:** Feature Engineering · Statistical Modeling · ML Pipelines · Time-Series Features · XGBoost
**Analytics Engineering:** dbt (advanced) · SQL · Data Modeling · Redshift · MetricFlow
**AI/LLM:** LLM Analyst Design · RAG Architecture · Prompt Engineering · NL-to-SQL
**DevOps:** CI/CD · dbt Cloud · GitHub Actions · Production Monitoring

## 📁 Repository Structure

```
├── projects/
│   ├── ods-edw-build/          # ODS & EDW architecture — technical lead
│   ├── feature-store/          # ML feature engineering at scale
│   ├── tpg-fca-model/          # Underwriting model — built from scratch
│   ├── tpg-operations-pipeline/# Legacy migration + LLM analyst vision
│   └── gdb-giftcard-breakage/  # Statistical breakage modeling
└── .github/
    └── copilot-instructions.md # AI context for this portfolio
```

## 🔧 How I Work

I treat data infrastructure the same way software engineers treat production systems — with clear architecture, rigorous testing, and documentation that outlives the builder.

**Discovery-first:** Before writing any SQL, I run mapping sessions with source system owners to understand grain, business keys, and data quality issues. I've led dozens of these for ODS/EDW builds and feature store design.

**Reconciliation as a first-class concern:** Every pipeline I build includes validation — row-count checks, aggregate comparisons, and edge-case spot checks between layers. The audit framework in the [TPG Operations Pipeline](projects/tpg-operations-pipeline/) and the QA process in the [ODS/EDW Build](projects/ods-edw-build/) both reflect this.

**Documentation for the next person:** I write architecture docs, model catalogs, and LLM context files not because it's required, but because I've been the person inheriting undocumented pipelines. Every project in this repo includes enough context for someone to understand the system without asking me.

**Stakeholder communication:** I'm comfortable presenting technical work to VP/C-level audiences — translating pipeline status, data quality risks, and model performance into business terms.

## 📬 Contact

- GitHub: [@akong-git](https://github.com/akong-git)
