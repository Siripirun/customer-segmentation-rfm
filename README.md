# Customer Segmentation Project (RFM Analysis, SQL Validation, Clustering & GenAI Insights)

An end-to-end customer analytics pipeline: transactional data → RFM feature engineering → SQL validation → K-means clustering → LLM-generated segment insights → Responsible AI review.

## Overview

Built a customer segmentation model using RFM (Recency, Frequency, Monetary) analysis on transactional data. The project was then extended to validate the original analysis in SQL, apply K-means clustering, and use an LLM to turn cluster statistics into plain-English business insights — with a documented review of where that LLM output fell short.

## Dataset

[Olist Brazilian E-Commerce Public Dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) (Kaggle), 2016–2018. Tables used: orders, customers and order items. Customers are identified by `customer_unique_id` (in Olist, `customer_id` is created per order, so one person can have several). Monetary values are in Brazilian reais (R$).

To run the notebook, download the CSV files from Kaggle into a `data/` folder.

## Tools

Python (pandas, NumPy, Matplotlib), SQL (SQLite), scikit-learn (StandardScaler, KMeans), Anthropic API (Claude), Jupyter Notebook.

## 1. RFM Feature Engineering

- Engineered customer-level features from transactional data:
  - **Recency:** days since the customer's last order
  - **Frequency:** number of distinct orders
  - **Monetary:** total spend
- Applied rule-based segmentation to identify high-value and low-engagement customers

## 2. SQL Validation

The original pandas-based RFM pipeline was rebuilt as SQL queries (joins, aggregate functions, a reference-date CTE) and cross-checked against the original results across all 95,420 customers. This surfaced two real data quality issues:

**Recency reference date was wrong, and here is how it was fixed.** Recency is measured as days since a customer's last order, relative to a single reference date representing "today" (the most recent point in the dataset). The original pipeline computed that reference date from a merged, inner-joined dataset (orders joined to order items), which silently dropped any order with no matching line items. The most recent orders in the full dataset happened to be among those dropped, so the reference date came out ~44 days earlier than it should have been, understating every customer's Recency by that same constant offset (customers looked more recent than they really were). The fix was to compute the reference date from the complete `orders` table directly, rather than from the inner-joined subset.

**Frequency was inflated for multi-item orders, and here is why.** The original calculation counted rows in the joined (orders × order_items) table. Since a single order can contain several line items, a customer who bought 3 items in one order was counted as Frequency = 3 instead of the correct Frequency = 1. The SQL version fixes this with `COUNT(DISTINCT order_id)`, counting distinct orders rather than line items.

**How the corrected figures were verified.** Both fixes were checked systematically across all 95,420 customers, not just spot-checked on one example:
- The Recency offset introduced by the date bug was consistently close to 44–45 days for every customer (not random noise), confirming it was a systematic bias, not a coincidence. The small spread within one day comes from pandas rounding down to whole days while SQL keeps fractions of a day.
- Monetary values matched exactly between the original and corrected pipelines for every customer (maximum difference: 0.00), confirming that part of the original logic was already correct.
- Frequency only ever decreased or stayed the same after the fix, for every customer — exactly the expected direction for a fix that removes double-counting, and it never increased, which would have indicated a new bug introduced by the fix itself.

Correcting the Frequency bug moved the measured share of one-time buyers from 87.6% to 96.9%. The Recency fix shifted every customer by the same constant, so it corrected the reported Recency values without changing customer rankings or the clusters.

## 3. K-means Clustering

K-means (k = 4) was applied to the corrected SQL-based RFM data, after standardising the features with StandardScaler so that Monetary (the largest values) does not dominate the distance calculation.

| Cluster | Share of customers | Profile |
|---|---|---|
| 1 | 54.4% | Recent one-time buyers (avg. recency ~178 days) |
| 0 | 40.2% | Lapsed one-time buyers (avg. recency ~438 days) |
| 3 | 3.0% | Repeat buyers (avg. 2.1 orders) |
| 2 | 2.3% | High-value one-time buyers (avg. spend ~R$1,155) |

## 4. GenAI Layer: Natural-Language Segment Summaries

An LLM (Claude, via the Anthropic API) turns each cluster's aggregate statistics into a short plain-English summary and one retention recommendation that a business stakeholder could act on.

**Responsible AI approach:**
- Only aggregate, anonymised cluster statistics are sent to the LLM — no individual customer records and no personally identifiable information.
- The API key is stored in a local `.env` file, excluded from version control via `.gitignore`, and never hard-coded.
- The output is a **draft for human review**, not an auto-published result, and is not used to trigger any real business decision.

## 5. Responsible AI Review: Where the LLM Output Fell Short

- **Under-differentiated recommendations:** although the segments have very different profiles, the LLM recommended a near-identical discount win-back campaign for all four. A high-value, low-frequency segment would more likely warrant personal outreach or a VIP programme.
- **Unsupported claims:** some statements (for example, a segment being "past the critical reactivation window") are not supported by the statistics provided.

This is why the output is treated as a draft for human review. It mirrors the SQL validation step: automated output, including LLM output, should be checked rather than trusted by default.

## Key Findings

- Almost all customers (96.9%) buy only once, so retention is the main opportunity.
- A small high-value segment (2.3% of customers) spends far more per customer and deserves distinct treatment.
- Cross-checking the pipeline in a second tool caught two silent bugs that a single implementation would have hidden.

## Limitations and Future Work

- Choose k with the elbow method or silhouette scores rather than fixing k = 4.
- Log-transform Frequency and Monetary before clustering, since both are heavily skewed and K-means is sensitive to outliers.
- Filter out cancelled and unavailable orders.
- Recompute the rule-based segments on the corrected data.
- Move the three validation checks into automated pytest tests.
- Improve the LLM step: send all segments in one prompt so the model must contrast them, add an example of a good answer, and evaluate outputs against a simple checklist.

## Repository Structure

- `customer-segmentation-project-rfm-analysis.ipynb` — full analysis notebook
- `README.md` — project documentation
