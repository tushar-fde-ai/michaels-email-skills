---
name: run
description: Build a YoY coupon and discount analysis HTML dashboard for Michaels retail. Covers promotional discount by department, coupon code/type breakdown, clearance, reward dollars, customer segmentation, and email campaign performance.
action: true
---

# Coupon & Discount YoY Dashboard

Build a self-contained HTML dashboard comparing a TY vs LY 20-day comparable window across five discount buckets: Promotional, Coupon, Clearance, Reward Dollars, and Email Campaigns. Includes customer segmentation (New/Existing/Reactivated).

---

## Step 0 — Establish date windows

Determine the TY and LY DAY_IDNT ranges from `cdp_unification_mk.bq_date_dim`. DAY_IDNT is VARCHAR — always quote it in BETWEEN comparisons. Both windows should span the same fiscal day range (same day-of-week alignment, 20 days each).

```sql
SELECT MIN(day_idnt), MAX(day_idnt), MIN(date), MAX(date)
FROM cdp_unification_mk.bq_date_dim
WHERE date BETWEEN '<TY_START>' AND '<TY_END>'
```

Repeat for LY. Use the resulting `day_idnt` strings as `'<TY_MIN>'`/`'<TY_MAX>'` throughout.

Always apply store exclusions: `store_number NOT IN ('9283', '9284')`.

---

## Step 1 — Headline metrics

**Gross sales and transaction count** from `enrich_transactions_behaviour`:

```sql
SELECT SUM(total_gross_sales) AS gross_sales,
       COUNT(DISTINCT transaction_id_number) AS txns
FROM cdp_unification_mk.enrich_transactions_behaviour
WHERE day_idnt BETWEEN '<MIN>' AND '<MAX>'
  AND store_number NOT IN ('9283','9284')
```

Run separately for TY and LY.

> Column names confirmed: `total_gross_sales` (not `sales_amount`), `transaction_id_number` (not `transaction_id`), `day_idnt` (lowercase), `dept_name` (not `department_description`).

---

## Step 2 — Promotional discount by department

Source: `mk_src.promo_day_store_skus` joined to `cdp_unification_mk.enrich_transactions_behaviour`.

The promo roster uses **calendar date** (`day_dt` column, string `YYYY-MM-DD`), while enrich uses **fiscal DAY_IDNT**. Bridge through `bq_date_dim`.

Note the column name mismatch between the two tables:
- SKU: `promo_day_store_skus.sku_idnt` vs `enrich.sku_key`
- Store: `promo_day_store_skus.store_idnt` vs `enrich.store_number`

Metric: `SUM(-item_discount_amount)`.

Exclude custom framing (`dept_name LIKE '%custom fram%'`) — contract pricing causes discount > gross.

```sql
SELECT e.dept_name AS dept,
       SUM(-e.item_discount_amount) AS promo_disc
FROM cdp_unification_mk.enrich_transactions_behaviour e
JOIN cdp_unification_mk.bq_date_dim d
  ON d.day_idnt = e.day_idnt
JOIN mk_src.promo_day_store_skus p
  ON p.sku_idnt     = e.sku_key
 AND CAST(p.store_idnt AS VARCHAR) = e.store_number
 AND p.day_dt       = d.date
WHERE e.day_idnt BETWEEN '<MIN>' AND '<MAX>'
  AND e.store_number NOT IN ('9283','9284')
  AND e.dept_name NOT LIKE '%custom fram%'
GROUP BY 1
ORDER BY promo_disc DESC
```

---

## Step 3 — Coupon discount by code

Source: `mk_stg.resa_coupon_detail_fact`. Metric: `SUM(-disc_amt)`.

**Critical exclusions:**
- `cpn_sku_idnt NOT IN ('999999','non-coupon')` — removes non-coupon product records
- `pos_cpn_idnt != '999999'` — removes non-coupon POS entries (these can be very large, e.g. $46M+ in LY — they are NOT coupon events)

```sql
SELECT pos_cpn_idnt AS coupon_code,
       cpn_sku_idnt AS sku,
       MAX(cpn_pct)   AS pct,
       SUM(-disc_amt) AS coupon_disc
FROM mk_stg.resa_coupon_detail_fact
WHERE DAY_IDNT BETWEEN '<MIN>' AND '<MAX>'
  AND store_number NOT IN ('9283','9284')
  AND cpn_sku_idnt NOT IN ('999999','non-coupon')
  AND pos_cpn_idnt != '999999'
GROUP BY 1, 2
ORDER BY coupon_disc DESC
```

Run for TY and LY. Merge results on `pos_cpn_idnt` to build TY/LY columns (codes present in one year but not the other get 0 for the missing year).

### Getting human-readable coupon names

`offer_type_description` in `enrich_transactions_behaviour` provides names for some codes. Join using `coupon_cd` (a `long` — equivalent to the last 12 digits of `pos_cpn_idnt` cast to integer, which strips leading zeros):

```sql
SELECT c.pos_cpn_idnt,
       MAX(e.offer_type_description) AS name
FROM mk_stg.resa_coupon_detail_fact c
JOIN cdp_unification_mk.enrich_transactions_behaviour e
  ON e.coupon_cd = TRY(CAST(SUBSTR(c.pos_cpn_idnt, LENGTH(c.pos_cpn_idnt)-11) AS BIGINT))
WHERE c.DAY_IDNT BETWEEN '<MIN>' AND '<MAX>'
  AND e.offer_type_description NOT LIKE 'coupon upc%'
  AND e.offer_type_description != 'non-coupon'
GROUP BY 1
```

Not all codes have readable names — `null` is fine; display a dash in the dashboard.

### Grouping by % off type

After collecting all codes, group them by `cpn_sku_idnt` (same SKU = same promotional event family). Assign a conceptual type label like `~30% off entire purchase`, `~40% off single item`, etc. This becomes the "By % Off / Type" view.

---

## Step 4 — Clearance

Source: `mk_src.clearance_day_store_skus` (item_type IN ('Basic','Seasonal')) joined to `enrich_transactions_behaviour`.

Same join pattern as promo: bridge through `bq_date_dim` to match calendar date in the clearance roster to fiscal `day_idnt` in enrich. Use `sku_idnt`/`store_idnt` from clearance vs `sku_key`/`store_number` in enrich (same mismatch as promo).

Metrics: clearance gross sales (`SUM(total_gross_sales)`), clearance discount (`SUM(-item_discount_amount)`), clearance transaction count.

Note: clearance discount can exceed clearance gross (items priced well below original retail). Present the disc/gross ratio as the "markdown depth" metric.

---

## Step 5 — Reward dollars redeemed

Source: `mk_stg.bq_rewards_redeem_mrptd` joined to `mk_stg.resa_rec15_fact` (voucher records, `rec_pattern LIKE '%voucher%'`) and `mk_stg.bq_earned_rewards` (card status).

Metric: `SUM(debit pos_approved_amount - credit pos_approved_amount) / 100` (stored in cents). Filter on redemption `day_idnt` range. No channel split — instore + online combined.

---

## Step 6 — Customer segmentation

Source: `cdp_unification_mk.enrich_transactions_behaviour`, loyalty members only (`loyalty_id_std LIKE '%lmr%'`).

364-day lookback window ending the day before each period. Derive lookback ranges from `bq_date_dim`:

```sql
-- TY lookback: ends the day before TY period start
SELECT MIN(day_idnt), MAX(day_idnt) FROM cdp_unification_mk.bq_date_dim
WHERE date BETWEEN DATE_ADD('day', -364, DATE '<TY_START>') AND DATE_ADD('day', -1, DATE '<TY_START>')
```

Segment definitions (per `loyalty_id_std`):
- **New**: no prior Michaels history at all (not in `all_history`)
- **Existing**: purchased within the 364-day lookback (present in `lookback`)
- **Reactivated**: has prior history before lookback, but zero transactions within lookback

Metrics: distinct customers, total sales, total transactions, AOV (sales/txns), TPC (txns/customers).

```sql
WITH lookback AS (
  SELECT loyalty_id_std
  FROM cdp_unification_mk.enrich_transactions_behaviour
  WHERE day_idnt BETWEEN '<LOOKBACK_MIN>' AND '<LOOKBACK_MAX>'
    AND loyalty_id_std LIKE '%lmr%'
    AND store_number NOT IN ('9283','9284')
  GROUP BY 1
),
all_history AS (
  SELECT loyalty_id_std
  FROM cdp_unification_mk.enrich_transactions_behaviour
  WHERE day_idnt < '<PERIOD_MIN>'
    AND loyalty_id_std LIKE '%lmr%'
  GROUP BY 1
),
period AS (
  SELECT loyalty_id_std,
         SUM(total_gross_sales)           AS sales,
         COUNT(DISTINCT transaction_id_number) AS txns
  FROM cdp_unification_mk.enrich_transactions_behaviour
  WHERE day_idnt BETWEEN '<PERIOD_MIN>' AND '<PERIOD_MAX>'
    AND loyalty_id_std LIKE '%lmr%'
    AND store_number NOT IN ('9283','9284')
  GROUP BY 1
)
SELECT
  CASE
    WHEN h.loyalty_id_std IS NULL THEN 'New'
    WHEN l.loyalty_id_std IS NOT NULL THEN 'Existing'
    ELSE 'Reactivated'
  END AS segment,
  COUNT(DISTINCT p.loyalty_id_std) AS customers,
  SUM(p.sales) AS sales,
  SUM(p.txns)  AS txns
FROM period p
LEFT JOIN lookback l ON p.loyalty_id_std = l.loyalty_id_std
LEFT JOIN all_history h ON p.loyalty_id_std = h.loyalty_id_std
GROUP BY 1
```

---

## Step 7 — Email campaigns

Source: `mk_stg.email_conversion_agg` (pre-computed, 7-day click attribution).

> **Important**: This table only covers dates up to approximately 3–4 weeks before the current date. For an August analysis period, use the most recent available 3-week window (e.g. Jun 13–Jul 4). Check the max available `send_dt` first:
> ```sql
> SELECT MAX(send_dt) FROM mk_stg.email_conversion_agg
> ```

The sales column is named **`sales`** (not `revenue`). Aggregate by `emailname_`:

```sql
SELECT emailname_,
       MIN(send_dt) AS send_dt,
       SUM(sends) AS sends, SUM(opens) AS opens,
       SUM(clicks) AS clicks, SUM(unsubs) AS unsubs,
       SUM(purchases) AS purchases,
       ROUND(SUM(sales), 2) AS revenue,
       ROUND(SUM(opens)*100.0/NULLIF(SUM(sends),0), 2) AS open_pct,
       ROUND(SUM(clicks)*100.0/NULLIF(SUM(sends),0), 4) AS click_pct,
       ROUND(SUM(sales)/NULLIF(SUM(purchases),0), 2) AS aov
FROM mk_stg.email_conversion_agg
WHERE send_dt BETWEEN '<START>' AND '<END>'
GROUP BY 1
ORDER BY SUM(sales) DESC
```

**Type classification** from `emailname_` pattern:
- contains `customframe` → Custom Frame (high open rates due to inbox preview renders — note this caveat)
- contains `_dce_` → DCE (daily/triggered)
- contains `_story_` → Story
- default → Promo / Weekend

Note: `emailname_` values are truncated in CLI display at ~52 chars. Use `SUBSTR(emailname_, 50)` to verify the tail of long names.

---

## Dashboard structure (HTML)

Build a single-file `<filename>-yoy-<period>.html` with:

1. **Headline KPIs** — gross sales, promo disc, coupon disc, reward dollars, clearance disc, transactions (TY vs LY with delta)
2. **Top-N callouts** — 5–6 key insight bullets with direction indicator and $ delta
3. **Section 1: Promo by department** — sortable table (by dept / TY / LY / delta / delta% / abs delta); diverging bar chart direction column
4. **Section 2: Coupon analysis** — dual view tabs (By Coupon Code / By % Off Type) with department filter dropdown; sortable columns
5. **Section 3: Clearance** — KPI tiles (gross, disc, txns, disc/gross ratio)
6. **Section 4: Reward dollars** — KPI tiles (txns, dollars, avg per redemption)
7. **Section 5: Customer segmentation** — loyalty members KPI tiles + segment breakdown table
8. **Section 6: Email campaigns** — aggregate KPI tiles + by-type summary table + sortable campaign table with type filter dropdown
9. **Method & caveats** — bullet list of all data sources, SQL logic, and overlap notes

Design: light mode only, no theme toggle. CSS variables for colors. Blue (`#2a78d6`) for TY, Orange (`#eb6834`) for LY. Diverging bars for directional comparison. Pills for LY-only / TY-only / both.

---

## Key caveats to include in the dashboard

- **Promo + coupon overlap**: a clearance item can also appear on `promo_day_store_skus`. Metrics are not additive to a clean total.
- **Custom framing excluded from promo dept table**: contract pricing causes discount > gross.
- **pos_cpn_idnt = 999999 exclusion**: these are non-coupon POS records; LY had $46.9M+ in excluded rows.
- **Loyalty sales vs total sales**: segmentation covers loyalty-linked transactions only — always a subset of total gross.
- **Email data lag**: `email_conversion_agg` does not update in real time; use the most recent available window and note the period explicitly.
- **Open rate on Custom Frame**: inflated by inbox preview renders, not genuine opens.
