# Gift Card Breakage

## 01. Overview

Until Q1 2023, Green Dot Corporation managed Walmart's gift card program as the issuing bank. While we stopped issuing new gift cards after January 2023, we continue managing any cards issued before then. As program manager, we earn revenue from "breakage," which is the unused balance left on gift cards that are unlikely to be redeemed.

Revenue from breakage is guided by **ASC 606** and generally recognized based on customer spending patterns. Where redemption is unlikely and unpredictable, revenue is recognized when the likelihood of redemption becomes remote.

## 02. Background

We use a statistical model based on a "random forest" approach to estimate breakage rates. Since Q2 2020, this model has helped us predict unused balances that we may claim as revenue. However, in March 2023, we noted an unusual increase in the breakage rate for the December 2022 gift card cohort, which reached **6.74%** -- higher than the historical range of 4.65% to 5.88% for this period.

Given the significance of December holiday cards and to avoid overstating revenue, we adjusted the breakage rate down to 5.62%, resulting in a **$1.7M revenue constraint** in March. Monitoring continued, and by Q2 2023, an additional adjustment of $1.0M was made, bringing the cumulative constraint to **$2.7M**.

To address these deviations, in August 2023, we began a thorough review of our model's methodology, logic, and assumptions to ensure accuracy.

## 03. Key Concepts

| Term | Definition |
|------|-----------|
| **Breakage** | Unused balance on gift cards unlikely to be redeemed, recognized as revenue |
| **Breakage Rate** | Percentage of initial load amount estimated to remain unredeemed at terminal period |
| **Terminal Period** | Window (60 months) after which remaining balance is considered fully breakage |
| **Escheatment** | State-mandated obligation to remit unclaimed property; reduces recognizable breakage revenue |
| **Load Date Cohort** | Monthly grouping of gift cards by their initial activation/load date |
| **Redemption Curve** | Cumulative percentage of initial load redeemed over elapsed months |

## 04. Documentation

| Document | Description |
|----------|-------------|
| [Model Methodology](docs/model-methodology.md) | Random Forest, Linear Regression, combined approach, and terminal period decisions |
| [Execution Runbook](docs/execution-runbook.md) | Step-by-step monthly/quarterly execution guide (SageMaker, Excel review, SQL) |
| [Data Pipeline](docs/data-pipeline.md) | Stored procedures, key tables, data lineage, and schema details |

## 05. Repository Contents

| File | Purpose |
|------|---------|
| `Breakage_Rate_and_Revenue_60months_v3_inference_monthly_retrain(5).ipynb` | SageMaker notebook: loads transactions, creates features, trains LR + RF models, outputs breakage rates |
| `breakage_create_acct_rev_v5_revised_model_use5_2024q3.sql` | Main revenue calculation script (rate inserts, escheatment adjustments, revenue, rate adjustments) |
| `breakage_monthly_recon_edw_fct_post_to_sagemaker_sp.sql` | Monthly reconciliation between EDW posted transactions and SageMaker pipeline data |
| `monthly_redemption.sql` | Cumulative and monthly redemption analysis by load-date cohort |
| `breakage_load_transdtl_v3.txt` | Stored procedure: load transaction details from EDW |
| `breakage_load_eliminated_v2.txt` | Stored procedure: identify and exclude loss accounts |
| `breakage_load_escheatment_v3.txt` | Stored procedure: classify accounts by state escheatment requirements |
| `breakage_load_transummary_v3.txt` | Stored procedure: create transaction summary (loads + redemptions) |
| `breakage_transform_transummary_v3.txt` | Stored procedure: apply escheatment flags to transaction summary |
| `breakage_create_feature_v3.txt` | Stored procedure: generate ML features (cumulative redemption percentages) |
| `Github - Sagemaker Studio setup.txt` | Guide for cloning this repo into SageMaker Studio |
