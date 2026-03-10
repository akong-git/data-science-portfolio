# Model Methodology

## 01. Prior Model: Random Forest Regression

Our original breakage model uses an in-house **random forest regression**, which combines multiple decision trees to predict breakage rates based on historical trends. By analyzing similar cohorts (e.g., December cohorts over different years), the model estimates unredeemed balances, assuming spending patterns will remain stable over time.

### COVID-19 Impact on Spending Patterns

During a deep-dive into recent spending, we found that consumer spending on gift cards slowed significantly during the COVID-19 pandemic. Pandemic cohorts (2019-2021) have higher unspent balances compared to pre-pandemic cohorts. This shift affected our model's breakage estimates, particularly for the December 2022 portfolio, which initially indicated a higher-than-expected rate.

Our prior model, which primarily references historical cohorts (e.g., December 2019), didn't fully account for these pandemic-driven shifts, resulting in lower breakage rate predictions than expected.

**Random Forest rate range:** Model output stored in `datascience.breakage_rate_v3_random_forest_monthly_retrain`.

## 02. Alternative Model: Linear Regression

To test another approach, we introduced a **linear regression model**, which considers relationships between breakage rates, spending trends, and other variables. This model provided a wider range of breakage rates (**4.0% to 10.5%**) but occasionally produced rates higher than actual card balances, making it overly aggressive for our use.

**Linear Regression rate range:** Model output stored in `datascience.breakage_rate_v3_linear_regression_monthly_retrain`.

## 03. Combined Modeling Approach

Since neither model was perfect, we adopted a **combined approach**, assigning equal weight (50/50) to both the random forest and linear regression models:

```
combined_rate = (0.5 * linear_regression_rate) + (0.5 * random_forest_rate)
```

This combined model produced breakage rates between **3.8% and 7.4%**, balancing both conservative and aggressive predictions. This approach resulted in a **$10.5 million adjustment** to our P&L as of September 30, 2023.

**Exceptions (manual overrides by Finance):**
- December 2020 cohort (`loaddate = '2020-12-01'`): Rate set to **7.882%** (Linear Regression value)
- December 2019 cohort (`loaddate = '2019-12-01'`): Rate set to **7.2%**
- November 2019 cohort (`loaddate = '2019-11-01'`): Rate set to **4.684%**

Combined rates are stored in `datascience.breakage_rate_v3_as_of_20240101_monthly_retrain_weight` with a `weight` column indicating the quarter (e.g., `'2024Q3'`).

## 04. Terminal Period Adjustment

As part of our review, we extended the breakage terminal period from **36 to 60 months** to reduce the risk of revenue reversals. Key rationale:

- Aligns with the **five-year expiration** typically printed on gift cards
- Provides a more conservative and objective approach
- Especially appropriate given no new gift cards are being issued post-January 2023

### Impact on Revenue Calculation

- **Open period (Month 0 to Month 59):** Breakage revenue is recognized proportionally based on the breakage rate and redemption activity
- **Closed period (Month 60+):** Any redemption in this period triggers `adjustment-reverse` entries that reverse revenue dollar-for-dollar

```sql
-- Open period: revenue = redemption_rate * breakage_to_date * (1 - escheatment_pct)
-- Closed period: revenue = -1 * redemption_amount * (1 - escheatment_pct)
```

## 05. Rate Adjustment Logic (MoM Delta)

Each quarter, after new rates are inserted, a `rate_adjust` is calculated as the month-over-month change:

```
rate_adjust = current_month_rate - prior_month_rate
```

This delta is used in the revenue calculation (Step 02C in the revenue script) to compute the difference between historical revenue-to-date and expected revenue under the latest rates, producing `adjustment-rate` entries.

## 06. Conclusion

Our dual-model approach, combined with a revised terminal period, provides a balanced and comprehensive estimate of breakage rates, factoring in the recent shifts in consumer spending and the impact of pandemic-era behavior. This ensures a more accurate and resilient model for future breakage revenue estimates.
