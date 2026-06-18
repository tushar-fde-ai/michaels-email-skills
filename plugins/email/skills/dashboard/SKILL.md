---
name: michaels-email-dashboard
description: Renders the HTML dashboard for Michaels email campaign performance reports. Called by michaels-email-skill once all query data is collected. Produces a single self-contained HTML file with 5 tabs.
---

# Email Campaign Dashboard — Layout Spec

Single self-contained HTML file. 5 tabs. All CSS inline. No external dependencies.

**Display labels:** `purchases` → "Transactions" · `customers` → "Customers" · `clickers` → "Clickers"

**Formatting:** Numbers with commas · Currency `$X,XXX` · Rates <1% show 3 decimals (0.142%), >1% show 1 decimal (17.9%) · bps = (this% − baseline%) × 100 · $ delta = (this−baseline)/baseline × 100

**Design:** Professional, clean, minimal. Neutral color palette (grays, slate, white backgrounds). Color only for meaning — green for favorable/revenue, amber for caution, muted accents for segment identity. No bright/saturated colors.

---

## Banner
Campaign name (monospace badge) · Send date · Attribution window · Theme · Type

## Tabs: Overview · Departments · Baseline · Geography (default) · RFM Segments (optional, on request)

> **Default:** Render 4 tabs. The RFM Segments tab is only added if the user opts in after the initial dashboard is rendered.

---

## Tab 1: Overview

**Volume** (grid-3): Sends · Opens · Clicks · Unsubscribes · Transactions (+ customers count) · Revenue (green)

**Engagement Rates** (grid-4 tiles with baseline):
- Open Rate — baseline median/mean, bps delta, green if above
- Click Rate — same
- CTOR — no baseline, value only
- Unsub Rate — baseline, green if BELOW (lower = better)

**Conversion** (grid-4):
- Click → Purchase Rate (customers/clickers) — no baseline
- Conv Rate (transactions/clickers) — baseline bps delta
- Audience Conversion Rate (customers/sends) — no baseline here (shown in Audience Metrics section)
- Revenue per Clicker — baseline % delta + AOV

**Audience Metrics** (grid-2) — both tiles show baseline comparisons:
- Audience Conversion Rate = customers / sends — baseline median/mean, bps delta
- Revenue per Audience = revenue / sends — baseline median/mean, % delta

---

## Tab 2: Departments

**Summary cards** (grid-4): Total Revenue · Dept count · Theme Revenue (tag related depts) · AOV

**Bar charts** (grid-2): Top 10 Revenue (green bars) · Top 10 Quantity (purple bars)

**Full table:** #, Department, Transactions, Revenue, AOV, % of Rev. Highlight theme depts.

**Footnote:** Dept counts > campaign total (multi-dept baskets).

---

## Tab 3: RFM Segments *(optional — only if user opts in)*

**Bridge badge:** RFM revenue vs clicker total (should be within ~1%)

**Segment cards** (grid-5) with colored top borders:
- Core (green) · Aspiring (blue) · Developing (amber) · Uncommitted (red) · **Reactivated (purple)**

Each card uses self-contained readout lines (no label/value split):
- `$X revenue` · `XX.X%`
- `X,XXX transactions` · `XX.X%`
- `X,XXX clickers` · `XX.X%`
- `X,XXX of X,XXX bought` · `XX.X%` ← conversion rate as inline context
- `$XX.XX rev per clicker`
- `$XX.XX per transaction`
- `X.XX trips per customer`

% figures are bold, same font size as the readout text.

Core/Aspiring/Developing/Uncommitted: percentages are share of total RFM pool. **Exclude reactivated customers from these 4 tiles.**

**Reactivated card** (purple): same readout format but percentages are share of total clicker pool. Sourced from §8b/§9 lifecycle data.

**Reactivation note:** Reactivated = dormant 364+ days. One short sentence only.

**Stacked bar:** 3 rows (Clickers/Transactions/Revenue) showing % across Core/Aspiring/Developing/Uncommitted

**Funnel table:** Segment · Clickers · Customers · Aud Conv · Transactions · Trips/Customer · Revenue · Rev/Clicker · AOV

**RFM × Lifecycle cross-tab:** Rows = RFM segments, Columns = Existing/Reactivated/New, values = customer counts. Callout below highlighting which segments have highest reactivation.

---

## Tab 4: Baseline

**Context note:** How baseline was calculated — type peer group, count, source table

**Comparison table:** Metric | This Campaign | Baseline Median | Baseline Mean | vs Baseline

Metrics: Open Rate · Click Rate · Unsub Rate · Conv Rate · Rev/Clicker · AOV · Audience Conv Rate · Rev/Audience

**Key takeaways:** Bullet points analyzing deltas, what drove them, implications

---

## Tab 5: Geography

**Channel split** (grid-4):
- Total card: text list showing `● In-Store $X · XX%` / `● BOPIS $X · XX%` / `● Online $X · XX%` — clearer than a stacked bar when BOPIS is small
- In-Store card (slate top border): revenue, txns, customers, units
- BOPIS card (purple top border): same
- Online card (blue top border): same
- chan_key: '1'=In-Store, '4'=BOPIS, others=Online

**Region pie chart:** East vs West in-store revenue as SVG donut. East (slate) / West (warm tan). Show $ and % labels.

**In-Store by Market** (top 10+): Horizontal bars, east/west colored, region pill badges (E/W)

**Online by State** (top 10+): Blue gradient bars, `{abbrev} — {full name}` format

**BOPIS by Market** (top 10+): Same format as in-store. Region split summary line above (East $X XX% · West $X XX% · Canada $X XX% if present).

**Methodology footnote:** chan_key mapping, store_info join, crafter_id bridge for online, region note, stores 9283/9284 excluded.

---

## Footer
Attribution method · Window dates · RFM join method · Reactivation fiscal IDs · Tables used
