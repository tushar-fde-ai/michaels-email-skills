---
name: run
description: Analyzes Michaels SMS performance by fiscal week or individual campaign
---

# SMS Performance Analysis

---

## Step 0: Initial Query + Prompt (ALWAYS run first)

Fetch 6 recent campaigns:

```sql
SELECT DISTINCT campaign_name
FROM mk_src.attentive_general_histunion
WHERE campaign_name IS NOT NULL
  AND (UPPER(campaign_name) LIKE 'MASS%' OR UPPER(campaign_name) LIKE 'PROMO%')
  AND UPPER(campaign_name) NOT LIKE '%WELCOME%'
  AND UPPER(campaign_name) NOT LIKE '%MAKERPLACE%'
  AND UPPER(campaign_name) NOT LIKE '%QUE%'
  AND UPPER(campaign_name) NOT LIKE '%CAN%'
ORDER BY MAX(event_timestamp) DESC
LIMIT 6;
```

Then ask:

> "How would you like to analyze SMS performance?
> - **By fiscal week** — aggregate all campaigns in a given week
> - **By individual campaign** — deep-dive a single campaign"

---

## Path A — Fiscal Week Analysis

### A1: Resolve fiscal week ID

```sql
SELECT
    MIN(CAST(day_idnt AS BIGINT)) AS week_min_id,
    MAX(CAST(day_idnt AS BIGINT)) AS week_max_id,
    MIN(day_dt) AS week_start_date
FROM cdp_unification_mk.bq_date_dim
WHERE wk_idnt = (
    SELECT wk_idnt FROM cdp_unification_mk.bq_date_dim
    WHERE CAST(DAY_DT AS DATE) = DATE '{user_date}'
    LIMIT 1
);
```

### A2: Run two parallel queries

**Q1 — Campaign breakdown (Mass vs Journey/Trigger):**

```sql
SELECT
    CASE
        WHEN UPPER(campaign_name) LIKE 'MASS%' THEN 'Mass'
        ELSE 'Journey/Trigger'
    END AS campaign_type,
    COUNT(DISTINCT CASE WHEN type = 'SENT' THEN message_id END) AS sends,
    COUNT(DISTINCT CASE WHEN type = 'CLICKED' THEN message_id END) AS clicks,
    ROUND(
        100.0 * COUNT(DISTINCT CASE WHEN type = 'CLICKED' THEN message_id END)
        / NULLIF(COUNT(DISTINCT CASE WHEN type = 'SENT' THEN message_id END), 0),
    2) AS click_rate
FROM mk_src.attentive_general_histunion
WHERE CAST(event_timestamp AS DATE) BETWEEN DATE '{week_start_date}' AND DATE '{week_end_date}'
GROUP BY 1
UNION ALL
SELECT 'Total', COUNT(DISTINCT CASE WHEN type='SENT' THEN message_id END),
    COUNT(DISTINCT CASE WHEN type='CLICKED' THEN message_id END),
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN type='CLICKED' THEN message_id END)
        / NULLIF(COUNT(DISTINCT CASE WHEN type='SENT' THEN message_id END),0), 2)
FROM mk_src.attentive_general_histunion
WHERE CAST(event_timestamp AS DATE) BETWEEN DATE '{week_start_date}' AND DATE '{week_end_date}';
```

**Q2 — Subscriber opt-in count up to that week:**

```sql
SELECT COUNT(DISTINCT phone) AS total_subscribers
FROM mk_src.attentive_optstatus
WHERE status = 'SUBSCRIBED'
  AND CAST(updated_at AS DATE) <= DATE '{week_end_date}';
```

### A3: Render fiscal week dashboard

Invoke **`email:dashboard`** with `channel: sms`.


KPI tiles: Total Sends · Total Clicks · Overall Click Rate · Subscriber Count

Segment table: Campaign Type | Sends | Clicks | Click Rate

Bar chart: Mass vs Journey/Trigger sends side-by-side.

---

## Path B — Individual Campaign Analysis

### B1: Confirm campaign name

### B2: Resolve send date range

```sql
SELECT
    MIN(CAST(event_timestamp AS DATE)) AS send_start,
    MAX(CAST(event_timestamp AS DATE)) AS send_end
FROM mk_src.attentive_general_histunion
WHERE LOWER(campaign_name) = LOWER('{campaign_name}')
  AND type = 'SENT';
```

Attribution window = send_start + 7 days.

### B3: Phase 1 — run these 3 in parallel

**Q1 — Engagement metrics:**

```sql
SELECT
    COUNT(DISTINCT CASE WHEN type = 'SENT' THEN message_id END) AS sends,
    COUNT(DISTINCT CASE WHEN type = 'CLICKED' THEN user_id END) AS unique_clickers,
    ROUND(
        100.0 * COUNT(DISTINCT CASE WHEN type = 'CLICKED' THEN user_id END)
        / NULLIF(COUNT(DISTINCT CASE WHEN type = 'SENT' THEN message_id END), 0),
    3) AS click_rate
FROM mk_src.attentive_general_histunion
WHERE LOWER(campaign_name) = LOWER('{campaign_name}');
```

**Q2a — Resolve clicker crafter_ids:**

```sql
SELECT DISTINCT a.user_id AS phone, e.crafter_id
FROM mk_src.attentive_general_histunion a
INNER JOIN cdp_unification_mk.enrich_attentive_optstatus e
    ON a.user_id = e.phone
WHERE LOWER(a.campaign_name) = LOWER('{campaign_name}')
  AND a.type = 'CLICKED'
  AND e.crafter_id IS NOT NULL;
```

**Q3a — Baseline click rate (peer campaigns ±30% send volume, last 90 days):**

```sql
WITH this_campaign AS (
    SELECT COUNT(DISTINCT CASE WHEN type='SENT' THEN message_id END) AS sends
    FROM mk_src.attentive_general_histunion
    WHERE LOWER(campaign_name) = LOWER('{campaign_name}')
)
SELECT
    APPROX_PERCENTILE(click_rate, 0.5) AS baseline_median_click_rate,
    AVG(click_rate) AS baseline_mean_click_rate,
    COUNT(*) AS peer_count
FROM (
    SELECT
        campaign_name,
        ROUND(100.0 * COUNT(DISTINCT CASE WHEN type='CLICKED' THEN user_id END)
            / NULLIF(COUNT(DISTINCT CASE WHEN type='SENT' THEN message_id END),0), 3) AS click_rate,
        COUNT(DISTINCT CASE WHEN type='SENT' THEN message_id END) AS sends
    FROM mk_src.attentive_general_histunion
    WHERE LOWER(campaign_name) != LOWER('{campaign_name}')
      AND CAST(event_timestamp AS DATE) >= CURRENT_DATE - INTERVAL '90' DAY
      AND (UPPER(campaign_name) LIKE 'MASS%' OR UPPER(campaign_name) LIKE 'PROMO%')
    GROUP BY campaign_name
) peers, this_campaign
WHERE peers.sends BETWEEN this_campaign.sends * 0.7 AND this_campaign.sends * 1.3;
```

### B4: Output Phase 1 text summary immediately, then kick off Phase 2

**Q2b — Click-attributed revenue (uses crafter_ids from Q2a):**

```sql
SELECT
    COUNT(DISTINCT t.transaction_id_number) AS transactions,
    COUNT(DISTINCT t.crafter_id) AS customers,
    ROUND(SUM(t.total_gross_sales), 2) AS revenue,
    ROUND(SUM(t.total_gross_sales) / NULLIF(COUNT(DISTINCT t.transaction_id_number), 0), 2) AS aov
FROM cdp_unification_mk.enrich_transactions_behaviour t
WHERE t.crafter_id IN ({crafter_id_list})
  AND CAST(t.DAY_IDNT AS BIGINT) BETWEEN {send_min_id} AND {attribution_max_id}
  AND t.store_number NOT IN ('9283', '9284');
```

**Q3b — Baseline revenue from peer campaigns:**

```sql
-- Same peer group as Q3a; compute median/mean of revenue per campaign
SELECT
    APPROX_PERCENTILE(revenue, 0.5) AS baseline_median_revenue,
    AVG(revenue) AS baseline_mean_revenue
FROM (
    SELECT campaign_name, SUM(total_gross_sales) AS revenue
    FROM ... -- peer campaign transactions
    GROUP BY campaign_name
) peers;
```

### B5: Render initial dashboard

Sends · Unique Clickers · Click Rate (vs baseline bps) · Revenue · AOV · Rev/Clicker

Invoke **`email:dashboard`** with `channel: sms`. Renders Overview + Baseline tabs.

Then offer:
> "Want me to add RFM segment and department breakdowns? (~5-10 min)"

### B6 (optional): Department + RFM breakdown

**Q4 — Department revenue (GROUPING SETS):**

```sql
SELECT
    t.dept_name,
    GROUPING(t.dept_name) AS is_total,
    COUNT(DISTINCT t.transaction_id_number) AS transactions,
    ROUND(SUM(t.total_gross_sales), 2) AS revenue
FROM cdp_unification_mk.enrich_transactions_behaviour t
WHERE t.crafter_id IN ({crafter_id_list})
  AND CAST(t.DAY_IDNT AS BIGINT) BETWEEN {send_min_id} AND {attribution_max_id}
  AND t.store_number NOT IN ('9283', '9284')
GROUP BY GROUPING SETS ((t.dept_name), ())
ORDER BY is_total DESC, revenue DESC;
```

**Q5 — RFM segment breakdown:**

```sql
SELECT
    c.rfm_segment_week AS rfm_segment,
    COUNT(DISTINCT t.crafter_id) AS clickers,
    COUNT(DISTINCT CASE WHEN t.total_gross_sales > 0 THEN t.crafter_id END) AS customers,
    COUNT(DISTINCT t.transaction_id_number) AS transactions,
    ROUND(SUM(t.total_gross_sales), 2) AS revenue,
    ROUND(SUM(t.total_gross_sales) / NULLIF(COUNT(DISTINCT t.crafter_id), 0), 2) AS rev_per_clicker,
    ROUND(SUM(t.total_gross_sales) / NULLIF(COUNT(DISTINCT t.transaction_id_number), 0), 2) AS aov
FROM cdp_unification_mk.enrich_transactions_behaviour t
INNER JOIN cdp_audience_961573.customers c ON t.crafter_id = c.crafter_id
WHERE t.crafter_id IN ({crafter_id_list})
  AND CAST(t.DAY_IDNT AS BIGINT) BETWEEN {send_min_id} AND {attribution_max_id}
  AND t.store_number NOT IN ('9283', '9284')
  AND c.rfm_segment_week IS NOT NULL
GROUP BY 1 ORDER BY revenue DESC;
```

### B7: Re-render full dashboard

Invoke **`email:dashboard`** with `channel: sms`. Renders Overview · Departments · RFM Segments · Baseline tabs.

---

## Calculated Metrics

| Metric | Formula |
|---|---|
| Click Rate | Unique Clickers / Sends |
| Conv Rate | Customers / Unique Clickers |
| Rev/Clicker | Revenue / Unique Clickers |
| AOV | Revenue / Transactions |
| bps delta | `(campaign% − baseline%) × 100` |
| % delta | `(campaign − baseline) / baseline × 100` |

---

## Data Tables

| Table | Purpose |
|---|---|
| `mk_src.attentive_general_histunion` | Raw SMS events (type: SENT, CLICKED) |
| `mk_src.attentive_optstatus` | Subscriber opt-in status |
| `cdp_unification_mk.enrich_attentive_optstatus` | Phone → crafter_id bridge |
| `cdp_unification_mk.enrich_transactions_behaviour` | Transaction records with crafter_id |
| `cdp_unification_mk.bq_date_dim` | Fiscal calendar dimension |
| `cdp_audience_961573.customers` | RFM master (rfm_segment_week) |
