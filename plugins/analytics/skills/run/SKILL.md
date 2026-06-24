---
name: run
description: Use when analyzing Michaels retail customer data, transaction metrics, segmentation (New/Existing/Reactivated customers), YoY waterfall analysis, department performance, income/demographic analysis, campaign calendar lookups, campaign SKU analysis, or any query against cdp_unification_mk or mk_src databases. Triggers on: "analyze customers", "customer trends", "YoY analysis", "waterfall", "department growth", "income analysis", "basket analysis", "Jonathan Adler", "Maker Haul", "Mothers Day", "campaign analysis", "campaign SKUs", "fiscal quarter", "campaign dates", "new vs existing customers", "AOV", "TPC", "AUR", "UPT", "segmentation", "loyalty", "crafter", or any Michaels retail analytics request.
---

# Michaels Analytics Agent

## Role & Objective
Expert retail analysis agent for Michaels. Enforce strict business definitions for "New," "Existing," and "Reactivated" customers. **CRITICAL: "New" ALWAYS means new to Michaels overall. NEVER define "New" as new to a specific brand, department, or campaign.**

---

## Database Schema

**Transactions:** `cdp_unification_mk.enrich_transactions_behaviour` (alias `t`)
- Standard columns: `total_gross_sales`, `transaction_id_number`, `store_number`, `item_quantity`, `DAY_IDNT`, `sku_key`
- Customer IDs: `LOYALTY_ID` (standard analysis) · `CRAFTER_ID` (department/category analysis ONLY)
- Always exclude: `t.store_number NOT IN ('9283', '9284')`

**Products:** `cdp_unification_mk.product_info_ai` (alias `p`) — cols: `dept_desc`, `sku_key`. Join: `t.sku_key = p.sku_key`

**Brand SKUs:**
- Jonathan Adler: `cdp_unification_mk.jonathan_adler_sku_list` (alias `ja`) — join: `CAST(t.sku_key AS BIGINT) = ja.sku_idnt`
- Maker Haul: `cdp_unification_mk.maker_haul_sku_list` (alias `mh`) — join: `CAST(t.sku_key AS BIGINT) = mh.sku_idnt`
- Mothers Day: `cdp_unification_mk.mothers_day_skus` (alias `md`) — join: `t.sku_key = md.sku_idnt`

**Demographics:** `cdp_unification_mk.customer_demo` (alias `cd`) — join on `loyalty_id`

**Fiscal Calendar:** `cdp_unification_mk.bq_date_dim` (alias `d`)
- `DAY_IDNT` (VARCHAR, fiscal day ID — join key)
- `MTH_IDNT` (fiscal month ID) · `QTR_IDNT` (fiscal quarter ID)
- `DAY_DT` (date string VARCHAR — cast: `CAST(DAY_DT AS DATE)`)

**Campaign Calendar:** `mk_src.fy_campaign_calendar_20260211`
- Cols: `primary_event`, `fy` (INTEGER/BIGINT — do not quote), `start_date` (VARCHAR), `end_date` (VARCHAR)
- Match: `LOWER(primary_event) LIKE '%...%'`

---

## Analysis Workflow

### Step 1 — Classify the Request

| Case | Trigger | ID | Filter | Template |
|------|---------|-----|--------|---------|
| **A1** Specific dept | Single dept named | `CRAFTER_ID` | `LOWER(p.dept_desc) = LOWER('{dept}')` | Template B |
| **A2** General depts | Multiple/all depts | `CRAFTER_ID` | `t.crafter_id IS NOT NULL` | Group by `dept_desc`, no segment template |
| **B** Standard | Default loyalty | `LOYALTY_ID` | `LOWER(t.loyalty_id_std) LIKE '%lmr%'` | Template A |
| **C** Demographics | Income/Age/Gender | `LOYALTY_ID` | NO LMR filter | Demographic templates |
| **D** Brand (JA/MH/MD) | Brand mentioned | `LOYALTY_ID` | `LOWER(t.loyalty_id_std) LIKE '%lmr%'` + brand join | Template A |

**CRITICAL for Case D:** Brand join MUST ONLY be in the main execution block. NEVER put brand join inside `all_history` or `existing_history` CTEs. History CTEs must remain unfiltered by brand so "New" means new to Michaels overall.

**Checks:**
- ALWAYS apply business metrics logic (see Business Metrics section)
- ONLY perform Waterfall if explicitly requested
- Use campaign calendar for specific event dates
- Use segmentation templates for Cases A1, B, D

### Step 2 — Construct Execution Plan (Divide & Conquer)

**CRITICAL: Split queries to avoid timeouts. Issue multiple simultaneous `tdx query` calls.**

- **Step 0 (Init):** `SELECT now() AS current_time_col` — confirms connection
- **Step A (Get Fiscal IDs):** Query `bq_date_dim` for `MIN(CAST(DAY_IDNT AS BIGINT))` as `{MIN_ID}`, `MAX(...)` as `{MAX_ID}`, and `MIN(DAY_DT)` as `{start_date}`
  - *Flow 1 (Fiscal):* Filter on `MTH_IDNT` or `QTR_IDNT`
  - *Flow 2 (Campaigns):* Get campaign dates from calendar, then find IDs in `bq_date_dim` via `CAST(DAY_DT AS DATE) BETWEEN...`
  - *Flow 3 (YTD):* Use Step 0 date to find today's `DAY_IDNT`
  - *Flow 4 (Full Year):* Chunk into 4 quarterly queries (`20251`, `20252`, `20253`, `20254`)
- **Step B (Lookback IDs):** NEVER guess dates. Derive from `{start_date}`:
  `WHERE CAST(DAY_DT AS DATE) BETWEEN (DATE '{start_date}' - INTERVAL '364' DAY) AND (DATE '{start_date}' - INTERVAL '1' DAY)`
- **Step C (Split Queries):**
  - Q1 (Aggregates): `WHERE CAST(t.DAY_IDNT AS BIGINT) BETWEEN {MIN_ID} AND {MAX_ID}`
  - Q2/Q3/Q4 (Segments): Inject lookback IDs. Calculate New, Existing, Reactivated in **separate** queries.

### Step 3 — Execute (Parallel)
Issue multiple simultaneous `tdx query` calls. Do not wait for one to complete before issuing the next.

---

## Business Metrics & Formulas

*Aliases (t, h, e) assume segmentation CTEs are applied. TY = This Year, LY = Last Year.*

1. **Total Customers:** `COUNT(DISTINCT t.LOYALTY_ID)`
2. **New Customers:** `COUNT(DISTINCT CASE WHEN h.LOYALTY_ID IS NULL THEN t.LOYALTY_ID END)`
3. **Existing Customers:** `COUNT(DISTINCT CASE WHEN e.LOYALTY_ID IS NOT NULL THEN t.LOYALTY_ID END)`
4. **Reactivated Customers:** `COUNT(DISTINCT CASE WHEN e.LOYALTY_ID IS NULL AND h.LOYALTY_ID IS NOT NULL THEN t.LOYALTY_ID END)`
5. **AOV (Sales per Txn):** `SUM(t.total_gross_sales) / COUNT(DISTINCT t.transaction_id_number)`
6. **TPC (Txns per Cust):** `COUNT(DISTINCT t.transaction_id_number) / CAST(COUNT(DISTINCT t.LOYALTY_ID) AS DOUBLE)`
7. **% of Matched Sales:** `SUM(CASE WHEN LOWER(t.loyalty_id_std) LIKE '%lmr%' THEN t.total_gross_sales END) / SUM(t.total_gross_sales)`
8. **Total Matching Sales:** `SUM(t.total_gross_sales)`
9. **Sales per Cust (New/React):** `SUM(CASE WHEN e.LOYALTY_ID IS NULL THEN t.total_gross_sales END) / COUNT(DISTINCT CASE WHEN e.LOYALTY_ID IS NULL THEN t.LOYALTY_ID END)`
10. **Sales per Cust (Existing):** `SUM(CASE WHEN e.LOYALTY_ID IS NOT NULL THEN t.total_gross_sales END) / COUNT(DISTINCT CASE WHEN e.LOYALTY_ID IS NOT NULL THEN t.LOYALTY_ID END)`
11. **Trans per Cust (New/React):** `COUNT(DISTINCT CASE WHEN e.LOYALTY_ID IS NULL THEN t.transaction_id_number END) / COUNT(DISTINCT CASE WHEN e.LOYALTY_ID IS NULL THEN t.LOYALTY_ID END)`
12. **Trans per Cust (Existing):** `COUNT(DISTINCT CASE WHEN e.LOYALTY_ID IS NOT NULL THEN t.transaction_id_number END) / COUNT(DISTINCT CASE WHEN e.LOYALTY_ID IS NOT NULL THEN t.LOYALTY_ID END)`
13. **AUR (Average Unit Retail):** `SUM(t.total_gross_sales) / NULLIF(SUM(t.item_quantity), 0)` *(CRITICAL: required for Waterfall Level 4)*
14. **UPT (Units Per Transaction):** `SUM(t.item_quantity) / COUNT(DISTINCT t.transaction_id_number)` *(CRITICAL: required for Waterfall Level 4)*

### YoY Waterfall — Growth Driver Decomposition

**CRITICAL:** NEVER calculate TY and LY in the same query. Run separate queries for TY and LY.

**Basis Points (bps) — MANDATORY FOR EVERY LEVEL:**
- Total Lift bps: `((Total_Sales_TY - Total_Sales_LY) / Total_Sales_LY) * 10000`
- Specific Driver bps: `(Driver_Sales_Effect / Total_Sales_LY) * 10000`

**Level 1 — Top Line Splits:**
- New/Reactivated Sales Effect: `(NewReact_Cust_Count_TY - NewReact_Cust_Count_LY) * Avg_Sales_per_Cust_TY`
- Existing Customer Sales Effect: derived from Level 2

**Level 2 — Existing Customer Splits:**
- Customer Count Effect: `(Existing_Cust_Count_TY - Existing_Cust_Count_LY) * Avg_Sales_per_Cust_LY`
- Average Spend Effect: `(Avg_Sales_per_Cust_TY - Avg_Sales_per_Cust_LY) * Existing_Cust_Count_TY`

**Level 3 — Average Spend Splits (Existing only):**
- Trip Frequency (TPC) Effect: `(TPC_TY - TPC_LY) * AOV_LY * Existing_Cust_Count_TY`
- AOV Effect: `(AOV_TY - AOV_LY) * TPC_TY * Existing_Cust_Count_TY`

**Level 4 — AOV Splits (Existing only):**
- AUR Effect: `(AUR_TY - AUR_LY) * UPT_TY * TPC_TY * Existing_Cust_Count_TY`
- UPT Effect: `(UPT_TY - UPT_LY) * AUR_LY * TPC_TY * Existing_Cust_Count_TY`

---

## SQL Segmentation Templates

### Template A — Standard Loyalty Analysis
*Target ID: `LOYALTY_ID` · Filter: `LOWER(t.loyalty_id_std) LIKE '%lmr%'`*

```sql
WITH existing_history AS (
    SELECT DISTINCT LOYALTY_ID
    FROM cdp_unification_mk.enrich_transactions_behaviour t
    WHERE CAST(t.DAY_IDNT AS BIGINT) BETWEEN {LOOKBACK_MIN_ID} AND {LOOKBACK_MAX_ID}
      AND t.store_number NOT IN ('9283', '9284')
      AND LOWER(t.loyalty_id_std) LIKE '%lmr%'
),
all_history AS (
    SELECT DISTINCT LOYALTY_ID
    FROM cdp_unification_mk.enrich_transactions_behaviour t
    WHERE CAST(t.DAY_IDNT AS BIGINT) < {MAIN_PERIOD_MIN_ID}
      AND t.store_number NOT IN ('9283', '9284')
      AND LOWER(t.loyalty_id_std) LIKE '%lmr%'
)
-- EXISTING: INNER JOIN existing_history e ON t.LOYALTY_ID = e.LOYALTY_ID
-- NEW:      LEFT JOIN all_history h ... WHERE h.LOYALTY_ID IS NULL
-- REACTIVATED: INNER JOIN all_history h + LEFT JOIN existing_history e ... WHERE e.LOYALTY_ID IS NULL
```

### Template B — Department / Crafter Analysis
*Target ID: `CRAFTER_ID` · Filter: `t.crafter_id IS NOT NULL` · History CTEs evaluate brand-level status, NOT dept-level.*

```sql
WITH existing_history AS (
    SELECT DISTINCT t.CRAFTER_ID
    FROM cdp_unification_mk.enrich_transactions_behaviour t
    WHERE CAST(t.DAY_IDNT AS BIGINT) BETWEEN {LOOKBACK_MIN_ID} AND {LOOKBACK_MAX_ID}
      AND t.store_number NOT IN ('9283', '9284')
      AND t.crafter_id IS NOT NULL
),
all_history AS (
    SELECT DISTINCT t.CRAFTER_ID
    FROM cdp_unification_mk.enrich_transactions_behaviour t
    WHERE CAST(t.DAY_IDNT AS BIGINT) < {MAIN_PERIOD_MIN_ID}
      AND t.store_number NOT IN ('9283', '9284')
      AND t.crafter_id IS NOT NULL
)
-- Main query: JOIN product_info_ai p, filter LOWER(p.dept_desc) IN ({DEPARTMENT_LIST})
-- EXISTING/NEW/REACTIVATED: same pattern as Template A using CRAFTER_ID
```

---

## Demographic Analysis Rules

**CRITICAL: DO NOT apply LMR filter in any demographic query.**

### Income Analysis (Extrapolation Template)
```sql
WITH period_total AS (
    SELECT SUM(total_gross_sales) AS absolute_total
    FROM cdp_unification_mk.enrich_transactions_behaviour t
    WHERE t.store_number NOT IN ('9283', '9284') AND {DATE_FILTERS}
),
known_incomes AS (
    SELECT
        CASE
            WHEN cd.income_bracket IN ('$0 - $ 14,999','$15,000 - $19,999','$20,000 - $29,999','$30,000 - $39,999','$40,000 - $49,999') THEN '<$50K'
            WHEN cd.income_bracket = '$50,000 - $74,999' THEN '$50K-$75K'
            WHEN cd.income_bracket = '$75,000 - $99,999' THEN '$75K-$100K'
            WHEN cd.income_bracket IN ('$100,000 - $124,999','$125,000 - $149,999','$150,000 - $174,999','$175,000 - 199,999') THEN '$100K-$200K'
            WHEN cd.income_bracket IN ('$200,000 - 249,999','$250,000 or more') THEN '>$200K'
        END AS income_bracket,
        SUM(t.total_gross_sales) AS known_sales
    FROM cdp_unification_mk.enrich_transactions_behaviour t
    JOIN cdp_unification_mk.customer_demo cd ON t.loyalty_id = cd.loyalty_id
    WHERE t.store_number NOT IN ('9283', '9284') AND {DATE_FILTERS} AND cd.income_bracket IS NOT NULL
    GROUP BY 1
)
SELECT k.income_bracket,
    (k.known_sales / SUM(k.known_sales) OVER ()) * p.absolute_total AS extrapolated_sales
FROM known_incomes k CROSS JOIN period_total p
```

### General Demographic Analysis (Age, Gender, Kids_Flag)
Do NOT use CASE WHEN — use raw values directly.
```sql
WITH period_total AS (
    SELECT SUM(total_gross_sales) AS absolute_total
    FROM cdp_unification_mk.enrich_transactions_behaviour t
    WHERE t.store_number NOT IN ('9283', '9284') AND {DATE_FILTERS}
),
known_demographics AS (
    SELECT cd.{DEMOGRAPHIC_COLUMN} AS demographic_value,
        SUM(t.total_gross_sales) AS known_sales
    FROM cdp_unification_mk.enrich_transactions_behaviour t
    JOIN cdp_unification_mk.customer_demo cd ON t.loyalty_id = cd.loyalty_id
    WHERE t.store_number NOT IN ('9283', '9284') AND {DATE_FILTERS}
      AND cd.{DEMOGRAPHIC_COLUMN} IS NOT NULL
    GROUP BY 1
)
SELECT k.demographic_value,
    (k.known_sales / SUM(k.known_sales) OVER ()) * p.absolute_total AS extrapolated_sales
FROM known_demographics k CROSS JOIN period_total p
```

---

## Critical Performance Rules

### 1. Fiscal Calendar Mapping (NO MATH GUESSING)
Retail fiscal year starts in **February**. Always query `bq_date_dim` — never compute dates manually.

**Month mapping (`MTH_IDNT`):**
- February = `01` of given year (e.g., Feb 2026 → `202601`)
- January = `12` of PREVIOUS year (e.g., Jan 2026 → `202512`)
- Mar=`02`, Apr=`03`, May=`04`, Jun=`05`, Jul=`06`, Aug=`07`, Sep=`08`, Oct=`09`, Nov=`10`, Dec=`11`

**Quarter mapping (`QTR_IDNT`):** Strictly ONE digit, NEVER zero-pad.
- Q4 2025 → `20254` (NOT `202504`)

### 2. No Date Joins
NEVER join `bq_date_dim` to the transactions table. Pre-query fiscal IDs, then inject directly into WHERE on `t.DAY_IDNT`.

### 3. Query Isolation — NO MEGA-QUERIES
- NEVER query a full 12-month year in one execution — split into 4 quarterly queries
- NEVER calculate all metrics in a single SQL statement
- NEVER combine New, Existing, and Reactivated in one query
- For large date ranges (> 3 months): prefer `APPROX_DISTINCT()` over `COUNT(DISTINCT)`

---

## Visualization & Output

### Standard Output
- Present results as **Markdown tables**
- Include key takeaways as bullet points above the table

### Waterfall Flow Chart (Tailwind CSS — NOT Plotly)
When user requests "Waterfall", "Driver Analysis", or "Growth Tree", render a hierarchical flow chart using Tailwind CSS:

**Visual Hierarchy:**
- **Level 0 (Top):** Total Lift ($ and bps)
- **Level 1:** New/Reactivated Sales vs. Existing Customer Sales
- **Level 2:** Customer Count Growth vs. Avg. Spend per Customer
- **Level 3:** Trip Frequency (TPC) vs. AOV Growth
- **Level 4:** AUR (Price/Mix) vs. UPT (Units/Txn) — if units data missing, render at 60% opacity with "DATA NOT AVAILABLE"

**Styling:**
- Cards: `bg-gradient-to-br`, `rounded-lg`, `shadow-xl`, borders
- Connectors: `<div className="w-1 h-12 bg-slate-400"></div>`
- Primary driver: darker gradient (`from-teal-400 to-teal-500`), "⭐ PRIMARY" badge
- Secondary: lighter gradient (`from-teal-200 to-teal-300`)
- Quote ALL inline style values: `style={{ opacity: '0.9', zIndex: '100' }}`

**Below the tree:**
- Legend & Key Insights (colors + Top 3 Drivers)
- Breakdown Summary table: Level (0–4), Component, $ Impact, BPS, % of Total, Status (✅ ⭐ ⚠️)

### Dashboard Layout (when full dashboard requested)
Single-column vertical stack:
1. Analysis Context (time period, filters)
2. Executive Summary (bullet takeaways)
3. Headline KPIs (responsive grid)
4. Visual Trend (one chart — line or bar)
5. Customer Segmentation Breakdown (one chart)
6. Waterfall Flow Chart (if applicable)
7. Detailed Metrics Table
