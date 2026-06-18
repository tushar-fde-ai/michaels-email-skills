---
description: Use this skill when asked to analyze email campaigns from Michaels treasure data instance
---

# Email Performance Analysis Rules

These instructions define how to generate optimized SQL queries for the Email Performance Reports

---

## General Context
* **Database Schema:** All tables are located in the `mk_stg` schema.
* **Campaign Parameter:** Provided by user. Substitute where `{input_campaign}` appears — matched against the `emailname_` column.
* **Attribution Window Parameter:** Integer days (default 7). Substitute where `{input_days}` appears.
* **Dates as Strings:** `eventdate`, `transaction_time`, `send_dt` are stored as strings. Use lexicographical comparison — fast and SARGable.
* **Campaign Theme:** Extract from the content portion of `emailname_` (between slot `_sN_` and the type keyword). E.g., `20260427_s1_seasonaljonathanadler_story_ff_us` → theme = "Seasonal Jonathan Adler". Use this to identify and highlight related departments in Tab 2 (e.g., departments containing "jonathan adler", "home décor", "party" for a JA campaign). If the theme is not clearly identifiable, omit the Campaign-Theme Revenue card.

---

## Core Data Tables & Join Keys
| Table | Notes / Join Key |
|---|---|
| `mk_stg.sfmc_sendlog` | Central campaign table. Pre-filter on `emailname_` first. |
| `mk_stg.sfmc_opens` | Joins to sendlog on `salesforce_id`. |
| `mk_stg.sfmc_clicks` | Joins to sendlog on `salesforce_id`. |
| `mk_stg.sfmc_unsubs` | Joins to sendlog on `subscriberkey` = `email_id`. |
| `mk_stg.transactions_behaviour` | Joins on `email_std`. Does **NOT** have `crafter_id`. |
| `cdp_unification_mk.enrich_transactions_behaviour` | Has `crafter_id`, `email_std`, `day_idnt`. Use whenever `crafter_id` is needed. |
| `cdp_unification_mk.bq_date_dim` | Calendar→fiscal date dimension. Maps `day_dt` → `day_idnt`. Required for reactivation. |
| `cdp_audience_961573.customers` | RFM master. Join via `crafter_id` — **never** `best_email` (drops ~15% revenue). Has `state`, `city`, `zip_code`. |
| `mk_gld.store_info` | Store geography. Join: `transactions_behaviour.store_number = store_info.loc_idnt`. Fields: `loc_mkt_name`, `regn_desc`, `loc_st_or_prvnc_cde`. |

> **Store exclusions:** Always exclude `store_number NOT IN ('9283','9284')` from revenue analysis.

---

## Performance & Correctness Rules
1. **Hybrid approach:** Use pre-computed `mk_stg.email_conversion_agg` for overview metrics (Tab 1) and baselines (Tab 4). Use self-contained Trino queries for breakdowns.
2. **Target Campaign CTE first.** Pre-filter `sfmc_sendlog` before joining activity tables.
3. **Campaign Dates from agg table.** Use `send_dt` from `email_conversion_agg` instead of computing from sendlog.
4. **`best_email` is NOT a join key.** Always bridge `email_std → crafter_id` via `enrich_transactions_behaviour`. Direct `LOWER(best_email) = email_std` drops ~15% of click-attributed revenue.
5. **No CTAS.** TD Trino does not support `CREATE TABLE ... AS`. All queries must be self-contained SELECT statements.
6. **Case-insensitive matching.** Use `LOWER(emailname_) = LOWER('{input_campaign}')` for all campaign name filters.

---

## Phase 1: Engagement Overview + Baseline (instant — from pre-computed agg table)

Source: `mk_stg.email_conversion_agg` (~152 rows). Columns: `emailname_`, `send_dt`, `sends`, `opens`, `clicks`, `unsubs`, `purchases`, `sales`, `qty`, `time`

> **Important:** The agg table uses a broader attribution model. For **revenue, transactions, and customers**, use the live click-attributed §5+6 query instead. The agg table is used only for engagement metrics (sends/opens/clicks/unsubs/rates) and baseline comparisons.

### 1–4. ENGAGEMENT OVERVIEW (Tab 1 — engagement only)
```sql
SELECT emailname_, send_dt, sends, opens, clicks, unsubs,
    CAST(opens AS DOUBLE) / NULLIF(sends, 0) AS open_rate,
    CAST(clicks AS DOUBLE) / NULLIF(sends, 0) AS click_rate,
    CAST(clicks AS DOUBLE) / NULLIF(opens, 0) AS ctor,
    CAST(unsubs AS DOUBLE) / NULLIF(sends, 0) AS unsub_rate
FROM mk_stg.email_conversion_agg
WHERE LOWER(emailname_) = LOWER('{input_campaign}');
```

> **Revenue metrics** (transactions, revenue, customers, AOV, conv rate, rev/clicker, audience conv rate, rev/audience) come from the live §5+6 query which is strictly click-attributed.

### 11. BASELINE COMPARISON (Tab 4)

**Peer group logic:** Identify the campaign's type from `emailname_`, select the matching peer group. Only one baseline — the type peers. Label it "Baseline" in the dashboard.

**Peer groups (determine from campaign name):**

| Group | Condition (any match) | Label in output |
|---|---|---|
| **Brand Story** | `emailname_` contains `_story_` | `story` |
| **Promo** | `emailname_` contains `_promo_` OR `_wknd_` OR `_circular_` OR `_eow_` OR `_otherpromo_` | `promo` |
| **DCE** | `emailname_` contains `_dce_` | `dce` |
| **Custom Frame** | `emailname_` contains `customframe` | `customframe` |

> **Classification rule:** Check in order: story → dce → customframe → promo (fallback). Use the FIRST match as the campaign's type.

**`baseline` WHERE clause by group:**
* **Brand Story:** `WHERE emailname_ LIKE '%\_story\_%' ESCAPE '\'`
* **Promo:** `WHERE (emailname_ LIKE '%\_promo\_%' ESCAPE '\' OR emailname_ LIKE '%\_wknd\_%' ESCAPE '\' OR emailname_ LIKE '%\_circular\_%' ESCAPE '\' OR emailname_ LIKE '%\_eow\_%' ESCAPE '\' OR emailname_ LIKE '%\_otherpromo\_%' ESCAPE '\')`
* **DCE:** `WHERE emailname_ LIKE '%\_dce\_%' ESCAPE '\'`
* **Custom Frame:** `WHERE emailname_ LIKE '%customframe%'`

```sql
WITH this_campaign AS (
    SELECT *, CAST(opens AS DOUBLE)/NULLIF(sends,0) AS open_rate, CAST(clicks AS DOUBLE)/NULLIF(sends,0) AS click_rate,
        CAST(unsubs AS DOUBLE)/NULLIF(sends,0) AS unsub_rate, CAST(purchases AS DOUBLE)/NULLIF(clicks,0) AS conv_rate,
        sales/NULLIF(clicks,0) AS rev_per_clicker, sales/NULLIF(purchases,0) AS aov
    FROM mk_stg.email_conversion_agg WHERE LOWER(emailname_) = LOWER('{input_campaign}')
),
baseline AS (
    SELECT *, CAST(opens AS DOUBLE)/NULLIF(sends,0) AS open_rate, CAST(clicks AS DOUBLE)/NULLIF(sends,0) AS click_rate,
        CAST(unsubs AS DOUBLE)/NULLIF(sends,0) AS unsub_rate, CAST(purchases AS DOUBLE)/NULLIF(clicks,0) AS conv_rate,
        sales/NULLIF(clicks,0) AS rev_per_clicker, sales/NULLIF(purchases,0) AS aov
    FROM mk_stg.email_conversion_agg
    WHERE {TYPE_PEER_CONDITION} AND LOWER(emailname_) != LOWER('{input_campaign}')
)
SELECT 'this_campaign' AS cohort, open_rate, click_rate, unsub_rate, conv_rate, rev_per_clicker, aov, sends, clicks, purchases, sales FROM this_campaign
UNION ALL SELECT 'baseline_median', APPROX_PERCENTILE(open_rate,0.5), APPROX_PERCENTILE(click_rate,0.5), APPROX_PERCENTILE(unsub_rate,0.5), APPROX_PERCENTILE(conv_rate,0.5), APPROX_PERCENTILE(rev_per_clicker,0.5), APPROX_PERCENTILE(aov,0.5), CAST(NULL AS BIGINT), CAST(NULL AS BIGINT), CAST(NULL AS BIGINT), CAST(NULL AS DOUBLE) FROM baseline
UNION ALL SELECT 'baseline_mean', AVG(open_rate), AVG(click_rate), AVG(unsub_rate), AVG(conv_rate), AVG(rev_per_clicker), AVG(aov), CAST(NULL AS BIGINT), CAST(NULL AS BIGINT), CAST(NULL AS BIGINT), CAST(NULL AS DOUBLE) FROM baseline;
```

> **Audience baseline metrics** (Audience Conv Rate = click_rate × conv_rate, Rev/Audience = click_rate × rev_per_clicker) are derived client-side from the baseline_median/mean rows above — not computed in SQL.

---

## Phase 2: Breakdown Queries

### Shared CTE definitions (referenced as `(...)` in subsequent queries)

```sql
target_campaign AS (
    SELECT salesforce_id FROM mk_stg.sfmc_sendlog
    WHERE LOWER(emailname_) = LOWER('{input_campaign}')
)
campaign_dates AS (
    SELECT send_dt AS start_date,
        CAST(DATE_ADD('day', {input_days}, DATE(send_dt)) AS VARCHAR) AS end_date
    FROM mk_stg.email_conversion_agg
    WHERE LOWER(emailname_) = LOWER('{input_campaign}')
)
clickers AS (
    SELECT DISTINCT c.email_std FROM mk_stg.sfmc_clicks c
    INNER JOIN target_campaign tc ON c.salesforce_id = tc.salesforce_id
    WHERE c.email_std IS NOT NULL
)
email_to_crafter AS (
    SELECT email_std, MIN(crafter_id) AS crafter_id
    FROM cdp_unification_mk.enrich_transactions_behaviour
    WHERE email_std IS NOT NULL AND crafter_id IS NOT NULL
    GROUP BY email_std
)
```

---

### 5+6. TRANSACTIONS, REVENUE & DEPARTMENT ATTRIBUTION (single query)

* **Result split:** `GROUPING(dept_name) = 1` → campaign totals. `= 0` → department rows (ignore where `dept_name IS NULL`).
* **Note:** Dept counts sum > campaign total (multi-dept baskets).
* **Display labels:** `purchases` column → display as "Transactions". `customers` column → display as "Customers".

```sql
WITH target_campaign AS (...), campaign_dates AS (...), clickers AS (...)
SELECT
    t.dept_name,
    GROUPING(t.dept_name) AS is_total,
    COUNT(DISTINCT t.transaction_id_number) AS purchases,
    COUNT(DISTINCT t.email_std) AS customers,
    SUM(t.item_quantity) AS quantity,
    ROUND(SUM(t.total_gross_sales), 2) AS revenue
FROM mk_stg.transactions_behaviour t
INNER JOIN clickers cl ON t.email_std = cl.email_std
CROSS JOIN campaign_dates cd
WHERE t.transaction_time >= cd.start_date AND t.transaction_time <= cd.end_date
  AND t.store_number NOT IN ('9283','9284')
GROUP BY GROUPING SETS ((t.dept_name), ())
ORDER BY is_total DESC, revenue DESC;
```

---

### 7. RFM SEGMENT ATTRIBUTION
> **Critical:** Always bridge via `crafter_id`. Never join on `LOWER(best_email) = email_std`.

* **RFM column:** `rfm_segment_week`. Values: Core, Aspiring, Developing, Uncommitted.
* **Sanity check:** Bridged RFM revenue should be within ~1% of §5 total.
* **Performance:** Pre-aggregate transactions per email (`clicker_txn_agg`) BEFORE joining to crafter/customer tables. This reduces millions of transaction rows to ~thousands of email-level rows before the expensive lookups.

```sql
WITH target_campaign AS (...), campaign_dates AS (...), clickers AS (...),
email_to_crafter AS (...),
clicker_txn_agg AS (
    SELECT t.email_std,
        COUNT(DISTINCT t.transaction_id_number) AS purchases,
        SUM(t.total_gross_sales) AS revenue
    FROM mk_stg.transactions_behaviour t
    INNER JOIN clickers cl ON t.email_std = cl.email_std
    CROSS JOIN campaign_dates cd
    WHERE t.transaction_time >= cd.start_date AND t.transaction_time <= cd.end_date
      AND t.store_number NOT IN ('9283','9284')
    GROUP BY t.email_std
),
clickers_with_crafter AS (
    SELECT cl.email_std, ec.crafter_id
    FROM clickers cl
    INNER JOIN email_to_crafter ec ON cl.email_std = ec.email_std
),
clicker_with_txns AS (
    SELECT cwc.email_std, cwc.crafter_id, cta.purchases, cta.revenue
    FROM clickers_with_crafter cwc
    LEFT JOIN clicker_txn_agg cta ON cwc.email_std = cta.email_std
)
SELECT
    cust.rfm_segment_week AS rfm_segment,
    COUNT(DISTINCT cwt.crafter_id) AS clickers,
    COUNT(DISTINCT CASE WHEN cwt.purchases > 0 THEN cwt.crafter_id END) AS customers,
    SUM(COALESCE(cwt.purchases, 0)) AS purchases,
    ROUND(SUM(COALESCE(cwt.revenue, 0)), 2) AS revenue
FROM clicker_with_txns cwt
INNER JOIN cdp_audience_961573.customers cust ON cust.crafter_id = cwt.crafter_id
WHERE cust.rfm_segment_week IS NOT NULL
GROUP BY cust.rfm_segment_week ORDER BY revenue DESC;
```

**Calculated:** Conv Rate = `Customers / Clickers` · Rev/Clicker = `Revenue / Clickers` · AOV = `Revenue / Transactions` · Audience Conv Rate (per segment) = `Customers / Clickers` · Revenue per Audience (per segment) = `Revenue / Clickers`

---

### 7b. RFM × LIFECYCLE CROSS-TAB (for Tab 3)

> Produces a cross-tabulation of RFM segment × lifecycle bucket. Each shopper appears in exactly one cell. Requires fiscal IDs from §8a.

```sql
WITH target_campaign AS (...), campaign_dates AS (...), clickers AS (...),
email_to_crafter AS (...),
clicker_txn_agg AS (
    SELECT t.email_std,
        COUNT(DISTINCT t.transaction_id_number) AS purchases,
        SUM(t.total_gross_sales) AS revenue
    FROM mk_stg.transactions_behaviour t
    INNER JOIN clickers cl ON t.email_std = cl.email_std
    CROSS JOIN campaign_dates cd
    WHERE t.transaction_time >= cd.start_date AND t.transaction_time <= cd.end_date
      AND t.store_number NOT IN ('9283','9284')
    GROUP BY t.email_std
),
clicker_with_crafter AS (
    SELECT cta.email_std, ec.crafter_id
    FROM clicker_txn_agg cta
    INNER JOIN email_to_crafter ec ON cta.email_std = ec.email_std
),
existing_history AS (
    SELECT DISTINCT crafter_id FROM cdp_unification_mk.enrich_transactions_behaviour
    WHERE CAST(day_idnt AS BIGINT) BETWEEN {LOOKBACK_MIN_ID} AND {LOOKBACK_MAX_ID}
      AND store_number NOT IN ('9283','9284') AND crafter_id IS NOT NULL
),
all_history AS (
    SELECT DISTINCT crafter_id FROM cdp_unification_mk.enrich_transactions_behaviour
    WHERE CAST(day_idnt AS BIGINT) < {LOOKBACK_MIN_ID}
      AND store_number NOT IN ('9283','9284') AND crafter_id IS NOT NULL
)
SELECT
    cust.rfm_segment_week AS rfm_segment,
    CASE
        WHEN eh.crafter_id IS NOT NULL THEN 'existing'
        WHEN ah.crafter_id IS NOT NULL THEN 'reactivated'
        ELSE 'new'
    END AS lifecycle,
    COUNT(DISTINCT cwc.crafter_id) AS shoppers
FROM clicker_with_crafter cwc
INNER JOIN cdp_audience_961573.customers cust ON cust.crafter_id = cwc.crafter_id
LEFT JOIN existing_history eh ON cwc.crafter_id = eh.crafter_id
LEFT JOIN all_history ah ON cwc.crafter_id = ah.crafter_id
WHERE cust.rfm_segment_week IS NOT NULL
GROUP BY 1, 2 ORDER BY 1, 2;
```

---

### 8. CLICKER IDENTITY DIAGNOSTIC *(standard — run for every campaign)*

**Purpose:** Split clicker revenue into four mutually exclusive lifecycle buckets. Results feed Tab 3.

> **Prerequisite:** Run Step 8a first to get fiscal IDs, then substitute into Step 8b.

#### Step 8a — Resolve fiscal IDs (instant)
```sql
WITH campaign_dates AS (
    SELECT send_dt AS send_date FROM mk_stg.email_conversion_agg
    WHERE LOWER(emailname_) = LOWER('{input_campaign}')
)
SELECT
    MIN(CASE WHEN day_dt = send_date THEN CAST(day_idnt AS BIGINT) END) AS analysis_min_id,
    MIN(CASE WHEN day_dt = CAST(DATE_ADD('day', {input_days}, DATE(send_date)) AS VARCHAR) THEN CAST(day_idnt AS BIGINT) END) AS analysis_max_id,
    MIN(CASE WHEN day_dt = CAST(DATE_ADD('day', -364, DATE(send_date)) AS VARCHAR) THEN CAST(day_idnt AS BIGINT) END) AS lookback_min_id,
    MIN(CASE WHEN day_dt = CAST(DATE_ADD('day', -1, DATE(send_date)) AS VARCHAR) THEN CAST(day_idnt AS BIGINT) END) AS lookback_max_id
FROM cdp_unification_mk.bq_date_dim, campaign_dates
WHERE day_dt IN (send_date, CAST(DATE_ADD('day',{input_days},DATE(send_date)) AS VARCHAR), CAST(DATE_ADD('day',-364,DATE(send_date)) AS VARCHAR), CAST(DATE_ADD('day',-1,DATE(send_date)) AS VARCHAR));
```

#### Step 8b — Diagnostic split (substitute fiscal IDs from 8a)

* **Performance:** Pre-aggregate transactions per email first, then classify by lifecycle.

| Bucket | Meaning |
|---|---|
| `no_crafter_id` | Email not resolvable to unified identity |
| `active_in_lookback_rfm_eligible` | Purchased in 364-day lookback |
| `strictly_reactivated` | Prior history, dormant for full lookback |
| `truly_new_no_history_ever` | No purchase history before analysis period |

```sql
WITH target_campaign AS (...), campaign_dates AS (...), clickers AS (...),
clicker_txn_agg AS (
    SELECT t.email_std,
        COUNT(DISTINCT t.transaction_id_number) AS purchases,
        SUM(t.total_gross_sales) AS revenue
    FROM mk_stg.transactions_behaviour t
    INNER JOIN clickers cl ON t.email_std = cl.email_std
    CROSS JOIN campaign_dates cd
    WHERE t.transaction_time >= cd.start_date AND t.transaction_time <= cd.end_date
      AND t.store_number NOT IN ('9283','9284')
    GROUP BY t.email_std
),
email_to_crafter AS (...),
existing_history AS (
    SELECT DISTINCT crafter_id FROM cdp_unification_mk.enrich_transactions_behaviour
    WHERE CAST(day_idnt AS BIGINT) BETWEEN {LOOKBACK_MIN_ID} AND {LOOKBACK_MAX_ID}
      AND store_number NOT IN ('9283','9284') AND crafter_id IS NOT NULL
),
all_history AS (
    SELECT DISTINCT crafter_id FROM cdp_unification_mk.enrich_transactions_behaviour
    WHERE CAST(day_idnt AS BIGINT) < {LOOKBACK_MIN_ID}
      AND store_number NOT IN ('9283','9284') AND crafter_id IS NOT NULL
)
SELECT
    CASE
        WHEN ec.crafter_id IS NULL     THEN 'no_crafter_id'
        WHEN eh.crafter_id IS NOT NULL THEN 'active_in_lookback_rfm_eligible'
        WHEN ah.crafter_id IS NOT NULL THEN 'strictly_reactivated'
        ELSE                                'truly_new_no_history_ever'
    END AS customer_type,
    SUM(cta.purchases) AS purchases,
    ROUND(SUM(cta.revenue), 2) AS revenue,
    COUNT(DISTINCT cta.email_std) AS unique_emails,
    COUNT(DISTINCT ec.crafter_id) AS unique_crafter_ids
FROM clicker_txn_agg cta
LEFT JOIN email_to_crafter ec ON cta.email_std = ec.email_std
LEFT JOIN existing_history eh ON ec.crafter_id = eh.crafter_id
LEFT JOIN all_history ah ON ec.crafter_id = ah.crafter_id
GROUP BY 1 ORDER BY revenue DESC;
```

---

### 9. CLICKER LIFECYCLE CLASSIFICATION *(bottom-up — always run)*

| Bucket | Definition |
|---|---|
| `existing` | Has `crafter_id` AND purchased in 364-day lookback |
| `reactivated` | Has `crafter_id` AND history before lookback AND no purchase in lookback |
| `new` | Has `crafter_id` BUT no purchase history before analysis period |
| `unidentified` | No `crafter_id` resolved |

```sql
WITH target_campaign AS (...), clickers AS (...), email_to_crafter AS (...),
clickers_with_crafter AS (
    SELECT cl.email_std, ec.crafter_id FROM clickers cl
    LEFT JOIN email_to_crafter ec ON cl.email_std = ec.email_std
),
existing_history AS (
    SELECT DISTINCT crafter_id FROM cdp_unification_mk.enrich_transactions_behaviour
    WHERE CAST(day_idnt AS BIGINT) BETWEEN {LOOKBACK_MIN_ID} AND {LOOKBACK_MAX_ID}
      AND store_number NOT IN ('9283','9284') AND crafter_id IS NOT NULL
),
all_history AS (
    SELECT DISTINCT crafter_id FROM cdp_unification_mk.enrich_transactions_behaviour
    WHERE CAST(day_idnt AS BIGINT) < {LOOKBACK_MIN_ID}
      AND store_number NOT IN ('9283','9284') AND crafter_id IS NOT NULL
)
SELECT
    CASE
        WHEN cwc.crafter_id IS NULL    THEN 'unidentified'
        WHEN eh.crafter_id IS NOT NULL THEN 'existing'
        WHEN ah.crafter_id IS NOT NULL THEN 'reactivated'
        ELSE                                'new'
    END AS lifecycle,
    COUNT(DISTINCT cwc.email_std) AS clickers,
    COUNT(DISTINCT cwc.crafter_id) AS unique_crafters
FROM clickers_with_crafter cwc
LEFT JOIN existing_history eh ON cwc.crafter_id = eh.crafter_id
LEFT JOIN all_history ah ON cwc.crafter_id = ah.crafter_id
GROUP BY 1 ORDER BY clickers DESC;
```

---

### 10. GEOGRAPHIC ANALYSIS *(click-attributed, dynamic window)*

**Channel identification** — `chan_key`: `'1'` = In-Store, `'4'` = BOPIS, all others = Online.
**In-store geography** — `store_number = mk_gld.store_info.loc_idnt` (applies to chan_key '1' and '4')
**Online geography** — Bridge: `email_std → enrich_transactions_behaviour → customers.state` (chan_key != '1' AND != '4')

#### 10a — Channel Split
```sql
WITH target_campaign AS (...), campaign_dates AS (...), clickers AS (...)
SELECT
    CASE
        WHEN t.chan_key = '1' THEN 'In-Store'
        WHEN t.chan_key = '4' THEN 'BOPIS'
        ELSE 'Online'
    END AS channel,
    COUNT(DISTINCT t.transaction_id_number) AS purchases,
    ROUND(SUM(t.total_gross_sales), 2) AS revenue,
    SUM(t.item_quantity) AS quantity,
    COUNT(DISTINCT t.email_std) AS purchasers
FROM mk_stg.transactions_behaviour t
INNER JOIN clickers cl ON t.email_std = cl.email_std
CROSS JOIN campaign_dates cd
WHERE t.transaction_time >= cd.start_date AND t.transaction_time <= cd.end_date
  AND t.store_number NOT IN ('9283','9284')
GROUP BY 1 ORDER BY revenue DESC;
```

#### 10b+c — In-Store by Market + Region totals (GROUPING SETS)

> Rows where `market IS NULL` = region rollup (pie chart). Rows where `market IS NOT NULL` = market detail (bar chart, top 20 by revenue).

```sql
WITH target_campaign AS (...), campaign_dates AS (...), clickers AS (...)
SELECT
    si.regn_desc AS region, si.loc_mkt_name AS market, si.loc_st_or_prvnc_cde AS state,
    COUNT(DISTINCT t.transaction_id_number) AS purchases,
    ROUND(SUM(t.total_gross_sales), 2) AS revenue,
    SUM(t.item_quantity) AS quantity,
    COUNT(DISTINCT t.email_std) AS purchasers
FROM mk_stg.transactions_behaviour t
INNER JOIN clickers cl ON t.email_std = cl.email_std
INNER JOIN mk_gld.store_info si ON t.store_number = si.loc_idnt
CROSS JOIN campaign_dates cd
WHERE t.transaction_time >= cd.start_date AND t.transaction_time <= cd.end_date
  AND t.store_number NOT IN ('9283','9284') AND t.chan_key = '1'
GROUP BY GROUPING SETS (
    (si.regn_desc, si.loc_mkt_name, si.loc_st_or_prvnc_cde),
    (si.regn_desc)
)
ORDER BY revenue DESC;
```

#### 10d — Online by Customer Home State (Top 20)

* **Performance:** Pre-aggregate online transactions per email, then resolve crafter → state on the small set.
* **Online only:** `chan_key != '1' AND chan_key != '4'` (excludes BOPIS)

```sql
WITH target_campaign AS (...), campaign_dates AS (...), clickers AS (...),
online_txn_agg AS (
    SELECT t.email_std,
        COUNT(DISTINCT t.transaction_id_number) AS purchases,
        SUM(t.total_gross_sales) AS revenue,
        SUM(t.item_quantity) AS quantity
    FROM mk_stg.transactions_behaviour t
    INNER JOIN clickers cl ON t.email_std = cl.email_std
    CROSS JOIN campaign_dates cd
    WHERE t.transaction_time >= cd.start_date AND t.transaction_time <= cd.end_date
      AND t.store_number NOT IN ('9283','9284')
      AND t.chan_key != '1' AND t.chan_key != '4'
    GROUP BY t.email_std
),
email_to_crafter AS (...)
SELECT
    UPPER(cust.state) AS state,
    SUM(ota.purchases) AS purchases,
    ROUND(SUM(ota.revenue), 2) AS revenue,
    SUM(ota.quantity) AS quantity
FROM online_txn_agg ota
LEFT JOIN email_to_crafter ec ON ota.email_std = ec.email_std
LEFT JOIN cdp_audience_961573.customers cust ON ec.crafter_id = cust.crafter_id
WHERE cust.state IS NOT NULL AND cust.state NOT IN ('\n','')
GROUP BY 1 ORDER BY revenue DESC LIMIT 20;
```

#### 10e — BOPIS by Market + Region totals (GROUPING SETS)

> Same structure as §10b+c but filtered to `chan_key = '4'`. Rows where `market IS NULL` = region rollup. Rows where `market IS NOT NULL` = market detail.

```sql
WITH target_campaign AS (...), campaign_dates AS (...), clickers AS (...)
SELECT
    si.regn_desc AS region, si.loc_mkt_name AS market, si.loc_st_or_prvnc_cde AS state,
    COUNT(DISTINCT t.transaction_id_number) AS purchases,
    ROUND(SUM(t.total_gross_sales), 2) AS revenue,
    SUM(t.item_quantity) AS quantity,
    COUNT(DISTINCT t.email_std) AS purchasers
FROM mk_stg.transactions_behaviour t
INNER JOIN clickers cl ON t.email_std = cl.email_std
INNER JOIN mk_gld.store_info si ON t.store_number = si.loc_idnt
CROSS JOIN campaign_dates cd
WHERE t.transaction_time >= cd.start_date AND t.transaction_time <= cd.end_date
  AND t.store_number NOT IN ('9283','9284') AND t.chan_key = '4'
GROUP BY GROUPING SETS (
    (si.regn_desc, si.loc_mkt_name, si.loc_st_or_prvnc_cde),
    (si.regn_desc)
)
ORDER BY revenue DESC;
```

---

## Calculated Metric Rules
* **Open Rate:** `Opens / Sends` · **Click Rate:** `Clicks / Sends` · **CTOR:** `Clicks / Opens`
* **Unsub Rate:** `Unsubscribes / Sends` · **bps delta:** `(campaign% − baseline%) × 100`
* **$ delta:** `(campaign − baseline) / baseline × 100`
* **Audience Conversion Rate (Overview):** `Customers / Sends` — send-attributed, measures what % of the total audience converted
* **Revenue per Audience (Overview):** `Revenue / Sends` — revenue generated per subscriber reached
* **Audience Conversion Rate (RFM):** `Segment Customers / Segment Clickers` — conversion efficiency within each segment
* **Revenue per Audience (RFM):** `Segment Revenue / Segment Clickers` — value generated per clicker in each segment

---

## Execution Flow

### Default run (fast — ~30-60 min total):
1. Phase 1: Engagement + Baseline (instant, from agg table)
2. §5+6: Transactions + Departments
3. §8a: Fiscal IDs (instant)
4. §10a: Channel split
5. §10b+c: In-Store markets
6. §10d: Online states
7. §10e: BOPIS markets

Render the dashboard with 4 tabs: **Overview · Departments · Baseline · Geography** (no RFM tab).

### After dashboard is rendered, ask the user:

> "Would you like to add the **RFM Segments** tab? This adds segment-level performance breakdowns (Core/Aspiring/Developing/Uncommitted/Reactivated), lifecycle classification, and a cross-tab showing where reactivated customers land. It requires 4 additional heavy queries against the transaction history table and typically takes **30-60 minutes** to complete. Want me to run it?"

### If user says yes — run RFM queries:
1. §7: RFM segment attribution
2. §7b: RFM × Lifecycle cross-tab (uses fiscal IDs from §8a)
3. §8b: Clicker identity diagnostic (uses fiscal IDs from §8a)
4. §9: Clicker lifecycle classification

Then re-render the dashboard with all 5 tabs: **Overview · Departments · RFM Segments · Baseline · Geography**

---

## Dashboard Rendering

Once all query data is collected, invoke the **`michaels-email-dashboard`** skill to render the HTML output. That skill defines the complete 5-tab dashboard structure, CSS classes, formatting rules, and layout specifications.
