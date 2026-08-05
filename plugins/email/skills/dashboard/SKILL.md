---
name: dashboard
description: Renders the HTML dashboard for Michaels campaign performance reports. Called by both email:run and sms:run. The caller passes a channel context ("email" or "sms") which controls which tabs and metrics are rendered.
---

# Campaign Dashboard — Layout Spec

Single self-contained HTML file. All CSS inline. No external dependencies.

**Channel context:** The caller (email:run or sms:run) specifies `channel: email` or `channel: sms`. This controls Tab 1 metrics and whether the Geography tab is included.

**Display labels:** `purchases` → "Transactions" · `customers` → "Customers" · `clickers` → "Clickers"

**Formatting:** Numbers with commas · Currency `$X,XXX` · Rates <1% show 3 decimals (0.142%), >1% show 1 decimal (17.9%) · bps = (this% − baseline%) × 100 · $ delta = (this−baseline)/baseline × 100

**Design:** Professional, clean, minimal. Neutral color palette (grays, slate, white backgrounds). Color only for meaning — green for favorable/revenue, amber for caution, muted accents for segment identity. No bright/saturated colors.

---

## Banner

Campaign name (monospace badge) · Send date · Attribution window · Channel badge (EMAIL or SMS) · Type

## Tabs by Channel

| Tab | Email | SMS |
|---|---|---|
| Overview | ✓ | ✓ |
| Departments | ✓ | ✓ (optional) |
| Baseline | ✓ | ✓ |
| Geography | ✓ | ✗ (omit) |
| RFM Segments | optional | optional |

**Default tabs — Email:** Overview · Departments · Baseline · Geography *(RFM added on request)*
**Default tabs — SMS:** Overview · Baseline *(Departments + RFM added on request)*

---

## Tab 1: Overview

### Email channel

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

### SMS channel

**Volume** (grid-3): Sends · Unique Clickers · Transactions (+ customers count) · Revenue (green)

*(No Opens, CTOR, or Unsub Rate — SMS channel does not have these signals)*

**Engagement Rates** (grid-2 tiles with baseline):

- Click Rate — baseline median/mean, bps delta, green if above
- Conv Rate (transactions/clickers) — baseline bps delta

**Revenue Metrics** (grid-3):

- Revenue — total click-attributed
- Rev/Clicker — baseline % delta
- AOV — baseline % delta

**Audience Metrics** (grid-2) — both tiles show baseline comparisons:

- Audience Conversion Rate = customers / sends — baseline median/mean, bps delta
- Revenue per Audience = revenue / sends — baseline median/mean, % delta

---

## Tab 2: Departments *(both channels)*

**Summary cards** (grid-4): Total Revenue · Dept count · Theme Revenue (tag related depts) · AOV

**Bar charts** (grid-2): Top 10 Revenue (green bars) · Top 10 Quantity (purple bars)

**Full table:** #, Department, Transactions, Revenue, AOV, % of Rev. Highlight theme depts.

**Footnote:** Dept counts > campaign total (multi-dept baskets).

---

## Tab 3: RFM Segments *(optional — only if user opts in, both channels)*

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

**Reactivated card** (purple): same readout format but percentages are share of total clicker pool. Sourced from lifecycle data.

**Reactivation note:** Reactivated = dormant 364+ days. One short sentence only.

**Stacked bar:** 3 rows (Clickers/Transactions/Revenue) showing % across Core/Aspiring/Developing/Uncommitted

**Funnel table:** Segment · Clickers · Customers · Aud Conv · Transactions · Trips/Customer · Revenue · Rev/Clicker · AOV

**RFM × Lifecycle cross-tab:** Rows = RFM segments, Columns = Existing/Reactivated/New, values = customer counts. Callout below highlighting which segments have highest reactivation.

---

## Tab 4: Baseline *(both channels)*

**Context note:** How baseline was calculated — peer group criteria, count, source table

**Comparison table:** Metric | This Campaign | Baseline Median | Baseline Mean | vs Baseline

**Email metrics:** Open Rate · Click Rate · Unsub Rate · Conv Rate · Rev/Clicker · AOV · Audience Conv Rate · Rev/Audience

**SMS metrics:** Click Rate · Conv Rate · Rev/Clicker · AOV · Audience Conv Rate · Rev/Audience

**Key takeaways:** Bullet points analyzing deltas, what drove them, implications

---

## Tab 5: Geography *(email channel only — omit for SMS)*

**Channel split** (grid-4):

- Total card: text list showing `● In-Store $X · XX%` / `● BOPIS $X · XX%` / `● Online $X · XX%`
- In-Store card (slate top border): revenue, txns, customers, units
- BOPIS card (purple top border): same
- Online card (blue top border): same
- chan\_key: '1'=In-Store, '4'=BOPIS, others=Online

**Region pie chart:** East vs West in-store revenue as SVG donut. East (slate) / West (warm tan). Show $ and % labels.

**In-Store by Market** (top 10+): Horizontal bars, east/west colored, region pill badges (E/W)

**Online by State** (top 10+): Blue gradient bars, `{abbrev} — {full name}` format

**BOPIS by Market** (top 10+): Same format as in-store. Region split summary line above (East $X XX% · West $X XX% · Canada $X XX% if present).

**Methodology footnote:** chan\_key mapping, store\_info join, crafter\_id bridge for online, region note, stores 9283/9284 excluded.

---

## Footer

Attribution method · Window dates · RFM join method · Reactivation fiscal IDs · Tables used · Channel
