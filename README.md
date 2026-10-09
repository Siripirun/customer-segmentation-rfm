# Customer Segmentation Project (RFM Analysis, SQL Validation, Clustering & GenAI Insights)

An end-to-end customer analytics pipeline: transactional data → RFM feature engineering → SQL validation → K-means clustering → LLM-generated segment insights → Responsible AI review.

## Overview

Built a customer segmentation model using RFM (Recency, Frequency, Monetary) analysis on transactional data. The project was then extended to validate the original analysis in SQL, apply K-means clustering, and use an LLM to turn cluster statistics into plain-English business insights — with a documented review of where that LLM output fell short.

## 1. RFM Feature Engineering

- Engineered customer-level features (Recency, Frequency, Monetary) from transactional data
- Applied rule-based segmentation to identify high-value and low-engagement customers

## 2. SQL Validation

The original pandas-based RFM pipeline was rebuilt as SQL queries (joins, aggregate functions, a reference-date CTE) and cross-checked against the original results across all 95,420 customers. This surfaced two real data quality issues:

**Recency reference date was wrong, and here is how it was fixed.** Recency is measured as days since a customer's last order, relative to a single reference date representing "today" (the most recent point in the dataset). The original pipeline computed that reference date from a merged, inner-joined dataset (orders joined to order items), which silently dropped any order with no matching line items. The most recent orders in the full dataset happened to be among those dropped, so the reference date came out ~44 days earlier than it should have been, inflating every customer's Recency by that same constant offset. The fix was to compute the reference date from the complete `orders` table directly, rather than from the inner-joined subset.

**Frequency was inflated for multi-item orders, and here is why.** The original calculation counted rows in the joined (orders × order_items) table. Since a single order can contain several line items, a customer who bought 3 items in one order was counted as Frequency = 3 instead of the correct Frequency = 1. The SQL version fixes this with `COUNT(DISTINCT order_id)`, counting distinct orders rather than line items.

**How the corrected figures were verified.** Both fixes were checked systematically across all 95,420 customers, not just spot-checked on one example:
- The Recency offset introduced by the date bug was consistently close to 44–45 days for every customer (not random noise), confirming it was a systematic bias, not a coincidence.
- Monetary values matched exactly between the original and corrected pipelines for every customer (maximum difference: £0.00), confirming that part of the original logic was already correct.
- Frequency only ever decreased or stayed the same after the fix, for every customer — exactly the expected direction for a fix that removes double-counting, and it never increased, which would have indicated a new bug introduced by the fix itself.

Correcting both moved the measured share of one-time buyers from 87.6% to 96.9%.
