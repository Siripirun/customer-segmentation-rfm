# Customer Segmentation Project (RFM Analysis, SQL Validation, Clustering & GenAI Insights)

An end-to-end customer analytics pipeline: transactional data → RFM feature engineering → SQL validation → K-means clustering → LLM-generated segment insights → Responsible AI review.

## Overview

Built a customer segmentation model using RFM (Recency, Frequency, Monetary) analysis on transactional data. The project was then extended to validate the original analysis in SQL, apply K-means clustering, and use an LLM to turn cluster statistics into plain-English business insights — with a documented review of where that LLM output fell short.

## 1. RFM Feature Engineering

- Engineered customer-level features (Recency, Frequency, Monetary) from transactional data
- Applied rule-based segmentation to identify high-value and low-engagement customers

## 2. SQL Validation

The original pandas-based RFM pipeline was rebuilt as SQL queries (joins, aggregate functions, a reference-date CTE) and cross-checked against the original results across all 95,420 customers. This surfaced two real data quality issues:

- **Recency reference date was biased by an implicit inner join.** The original pipeline computed the most recent order date from a merged, inner-joined dataset, which silently dropped orders with no matching line items — inflating every customer's Recency by a constant ~44-day offset.
- **Frequency was inflated for multi-item orders.** The original calculation counted line items, not distinct orders, overstating how often customers actually purchased.

Correcting both moved the measured share of one-time buyers from 87.6% to 96.9%.

## 3. K-means Clustering

Applied K-means clustering to the corrected data to uncover data-driven customer groups based on behavioural patterns, beyond the original rule-based segments.

## 4. GenAI Layer

Used the Anthropic API to generate plain-English, stakeholder-facing summaries and retention recommendations for each customer segment from its aggregate statistics — no individual customer data or PII sent to the LLM.

## 5. Responsible AI Review

The LLM's recommendations came out nearly identical across all four segments, despite genuinely different underlying statistics — a real, documented limitation, not a success story. This is why the output is treated as a draft for human review, not an auto-published result: no LLM output here is used to trigger a real business decision without a human checking it first.

## Key Takeaway

Analysed customer behaviour to generate actionable, data-driven recommendations for retention and targeted marketing strategies — while also demonstrating that both the data pipeline and the AI layer need to be checked, not trusted by default.

## Tools

Python, Pandas, NumPy, Scikit-learn, Matplotlib/Seaborn, SQL (SQLite), Anthropic API

## Project History

Originally developed and published on [Kaggle](https://www.kaggle.com/code/siripiruntans/customer-segmentation-project-rfm-analysis). The SQL validation, K-means clustering, GenAI layer and Responsible AI review were added as a second phase of the project, documented in full in the notebook in this repository.
