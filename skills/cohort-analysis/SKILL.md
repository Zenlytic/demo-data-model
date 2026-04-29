---
name: cohort-analysis
description: Definitions, assumptions, and build instructions 
for acquisition cohort analyses at Pure Organics. Trigger this skill whenever the user asks about cohort analysis, retention analysis, LTV by cohort, or customer retention curves. Covers channel cohorts and first-product cohorts.
---

# Pure Organics — Cohort Analysis Skill

## When to Trigger This Skill

Trigger this skill whenever the user asks about:
- Cohort analysis (acquisition, behavioral, channel, product)
- Customer retention rates or retention curves
- LTV by cohort
- Repeat purchase behavior over time
- "How do customers from X channel/product retain?"

---

## Core Definitions & Assumptions

### Cohort Definition
- **Acquisition cohort** = group of customers defined by the calendar month of their **first order**.
- Use `customers.first_order_at` (truncated to month) as the cohort date. This is the authoritative first-order timestamp on the `DEMO_PROD.CUSTOMERS` table.
- Cohort month SQL: `DATE_TRUNC('MONTH', CAST(CAST(CONVERT_TIMEZONE('America/New_York', customers.first_order_at) AS TIMESTAMPNTZ) AS TIMESTAMP))`

### Retention Definition
- A customer is **retained in month N** if they placed at least one order in the calendar month that is N months after their cohort month.
- Retention is measured at the **customer level** (not order level): `COUNT(DISTINCT CUSTOMER_ID)` who ordered in that month.
- **Retention % for month N** = (customers in cohort who ordered in cohort_month + N) / cohort_size x 100
- Month 0 = the cohort month itself (always 100% by definition — every customer placed at least one order to be in the cohort).
- Months 1-12 are the standard reporting window.

### Cohort Size
- Cohort size = `COUNT(DISTINCT CUSTOMER_ID)` whose `first_order_at` falls in that cohort month.

### Channel Cohort Segmentation
- Use `customers.first_channel` (SQL: `customers.first_channel`) as the acquisition channel dimension.
- This is the **first marketing channel** the customer came through before their first order.
- Available channels: Email, Facebook, Organic, Paid Search, TikTok.
- Do NOT use `order_lines.MARKETING_CHANNEL` for cohort segmentation — that reflects the channel on each individual order, not the acquisition channel.

### First Product Type Cohort Segmentation
- Identify each customer's first order by joining `order_lines` to find `MIN(ORDER_AT)` per customer.
- Join that first order to `DEMO_PROD.PRODUCTS` via `PRODUCT_ID` to get the product type.
- If a customer bought multiple product types on their first order, use `MIN(PRODUCT_TYPE)` (alphabetical tiebreak) for a single clean assignment.
- Available product types: Cheek, Eye, Face, Lip.

### Net Revenue vs Gross Revenue
- When computing revenue per cohort, default to **net revenue** (`order_lines.REVENUE - order_lines.DISCOUNT`, excluding AE/AP military postal codes) per the workspace default.
- Use gross revenue only if explicitly requested.

---

## Standard SQL Pattern

```sql
-- Step 1: Customer cohorts with segmentation dimension
WITH customer_cohorts AS (
  SELECT
    c.CUSTOMER_ID,
    DATE_TRUNC('MONTH', CAST(CAST(CONVERT_TIMEZONE('America/New_York', c.first_order_at) AS TIMESTAMPNTZ) AS TIMESTAMP)) AS cohort_month,
    c.first_channel AS acquisition_channel  -- or first_product_type via join
  FROM DEMO_PROD.CUSTOMERS c
  WHERE c.first_order_at IS NOT NULL
    AND c.first_channel IS NOT NULL
),

-- Step 2: All order months per customer
customer_orders AS (
  SELECT DISTINCT
    ol.CUSTOMER_ID,
    DATE_TRUNC('MONTH', CAST(CAST(CONVERT_TIMEZONE('America/New_York', ol.ORDER_AT) AS TIMESTAMPNTZ) AS TIMESTAMP)) AS order_month
  FROM DEMO_PROD.ORDER_LINES ol
),

-- Step 3: Active customers per cohort x month offset
cohort_activity AS (
  SELECT
    cc.acquisition_channel,
    cc.cohort_month,
    DATEDIFF('month', cc.cohort_month, co.order_month) AS months_since_acquisition,
    COUNT(DISTINCT cc.CUSTOMER_ID) AS active_customers
  FROM customer_cohorts cc
  JOIN customer_orders co ON cc.CUSTOMER_ID = co.CUSTOMER_ID
  WHERE DATEDIFF('month', cc.cohort_month, co.order_month) BETWEEN 0 AND 12
  GROUP BY cc.acquisition_channel, cc.cohort_month, months_since_acquisition
),

-- Step 4: Cohort sizes
cohort_sizes AS (
  SELECT
    acquisition_channel,
    cohort_month,
    COUNT(DISTINCT CUSTOMER_ID) AS cohort_size
  FROM customer_cohorts
  GROUP BY acquisition_channel, cohort_month
)

-- Step 5: Final retention %
SELECT
  ca.acquisition_channel,
  ca.cohort_month,
  cs.cohort_size,
  ca.months_since_acquisition,
  ca.active_customers,
  ROUND(100.0 * ca.active_customers / cs.cohort_size, 1) AS retention_pct
FROM cohort_activity ca
JOIN cohort_sizes cs
  ON ca.acquisition_channel = cs.acquisition_channel
  AND ca.cohort_month = cs.cohort_month
ORDER BY ca.acquisition_channel, ca.cohort_month, ca.months_since_acquisition;
```

For **first product type** cohorts, replace the `customer_cohorts` CTE with:

```sql
WITH first_orders AS (
  SELECT ol.CUSTOMER_ID, MIN(ol.ORDER_AT) AS first_order_at
  FROM DEMO_PROD.ORDER_LINES ol
  GROUP BY ol.CUSTOMER_ID
),
customer_cohorts AS (
  SELECT
    fo.CUSTOMER_ID,
    DATE_TRUNC('MONTH', CAST(CAST(CONVERT_TIMEZONE('America/New_York', fo.first_order_at) AS TIMESTAMPNTZ) AS TIMESTAMP)) AS cohort_month,
    MIN(p.PRODUCT_TYPE) AS first_product_type
  FROM first_orders fo
  JOIN DEMO_PROD.ORDER_LINES ol ON fo.CUSTOMER_ID = ol.CUSTOMER_ID AND fo.first_order_at = ol.ORDER_AT
  JOIN DEMO_PROD.PRODUCTS p ON ol.PRODUCT_ID = p.PRODUCT_ID
  GROUP BY fo.CUSTOMER_ID, cohort_month
)
```

---

## Dashboard Requirements

Every cohort analysis dashboard MUST include the following components:

### 1. KPI Summary Cards
- One card per segment (channel or product type)
- Show M1, M6, and M12 retention rates
- Color-code values: green >= 90%, amber 70-89%, red < 70%
- Cards must be clickable to drill into cohort-level detail

### 2. Average Retention Line Chart
- X-axis: Months since first purchase (M1-M12)
- Y-axis: Retention rate (%)
- One line per segment, using the brand color palette
- Averaged across all cohort months
- Clicking a data point drills into the per-cohort breakdown for that month

### 3. Month-12 Retention Bar Chart
- Horizontal bar chart ranked by M12 retention (highest to lowest)
- Shows which channel or product type retains best at 1 year
- Include data labels on each bar

### 4. Retention Heatmap (REQUIRED — see reference image)
- **This is a mandatory chart type for all cohort analyses.**
- Rows = cohort months (e.g., "Apr 2024", "May 2024", ...)
- Columns = months since acquisition (M1, M2, ... M12)
- Cell values = retention % for that cohort x month combination
- Color scale: red (low retention) -> amber -> green (high retention)
- Reference image: `assets/retention-heatmap-reference.png` — match this layout closely.
  - The reference shows a triangular/diagonal heatmap where newer cohorts have fewer filled columns (because they haven't had time to mature). Replicate this pattern: only fill cells where data exists; leave future months empty.
  - Each cell should display the retention % as a number inside the colored cell.
  - Row labels = cohort month (left side), with cohort size shown next to the label.
  - Column headers = month number (M1, M2, ...).
- Use Highcharts `heatmap` series type.
- Cells must be clickable to show the underlying customer count and retention detail.

### 5. Cohort Retention Curve (Filterable)
- Line chart showing retention curves for each segment
- Include a dropdown to filter to a specific cohort month
- Default to "All Cohorts (Avg)"

### Tab Toggle
- Always provide a toggle between "By Acquisition Channel" and "By First Product Type" views.

---

## Color Palette

Use these colors consistently for segments:

**Channels:**
- Email: `#3D5A47` (Forest Green)
- Facebook: `#C17F59` (Terracotta)
- Organic: `#5A7A62` (Sage)
- Paid Search: `#7A9BA8` (Dusty Blue)
- TikTok: `#E8DDB5` (Butter)

**Product Types:**
- Cheek: `#3D5A47`
- Eye: `#C17F59`
- Face: `#5A7A62`
- Lip: `#7A9BA8`

**Heatmap color scale:**
- Low (<= 40%): `#B85450` (Error Red)
- Mid (70%): `#D4A853` (Warning Amber)
- High (>= 90%): `#4A7C59` (Success Green)
- Max (100%): `#3D5A47` (Forest Green)

---

## Data Notes

- **Data range**: Customer acquisition data runs from approximately April 2024 to September 2024. Cohorts acquired in later months will have fewer mature data points (e.g., a September 2024 cohort only has ~7 months of follow-up as of April 2026).
- **Dips at M4 and M9**: Consistent retention dips appear at months 4 and 9 across all channels and product types. This is a known pattern in the data — flag it to the user and recommend investigation.
- **Timezone**: Always apply `CONVERT_TIMEZONE('America/New_York', ...)` when truncating order timestamps.
- **Military postal exclusions**: Net revenue calculations exclude customers in AE and AP states (military postal codes).

---

## Reference Image — Retention Heatmap

The file `assets/retention-heatmap-reference.png` shows the target layout for the cohort retention heatmap. Key characteristics to replicate:
- Triangular shape: each row has one fewer column than the row above (newer cohorts have less history)
- Cohort month labels on the left, cohort size ("New Users") in a column next to the labels
- Month numbers (1, 2, 3, ...) as column headers
- Retention % displayed as a number inside each colored cell
- Color gradient from red (low) to green (high)
- Empty/white cells for months that have not occurred yet for a given cohort

---

## Brand Design System

Extracted from `assets/fde_discussion.pptx`. Use these when producing any presentation, report, or styled output for Pure Organics.

### Logos

Two logo variants are available in `assets/`:

| File | Usage |
|------|-------|
| `assets/logo-white.png` | White logo — use on dark/green backgrounds (title slides with `#062810` or `#09493D` background) |
| `assets/logo-dark.png` | Dark/black logo — use on light backgrounds (content slides with `#F7F3E8` background) |

- Native dimensions: 5000 x 1405 px (wide horizontal lockup: icon + "zenlytic" wordmark)
- Always place in the top-left corner of slides, sized to approximately 1.4-1.5" wide x 0.35" tall
- Do not recolor the logo — use the white variant on dark backgrounds and the dark variant on light backgrounds

### Slide Color Palette

| Hex | Name | Usage |
|-----|------|-------|
| `#062810` | Deep Forest Black | Title slide background, darkest accent |
| `#09493D` | Dark Forest Green | Headings on light slides, section titles, accent bars |
| `#7CB77F` | Light Green | Subtitle text on dark slides, accent dividers |
| `#F7F3E8` | Warm Cream | Content slide background (light mode) |
| `#454547` | Dark Gray | Body text on light slides |
| `#828287` | Medium Gray | Secondary/caption text, labels |
| `#FFFFFF` | White | Body text on dark slides, logo on dark backgrounds |

### Typography

| Role | Font | Size | Color |
|------|------|------|-------|
| Slide title (dark bg) | Georgia, bold | 32pt | `#FFFFFF` |
| Slide title (light bg) | Georgia, bold | 24pt | `#09493D` |
| Section heading | Georgia, bold | 13-14pt | `#09493D` |
| Subtitle / tagline | Calibri | 14pt | `#7CB77F` (on dark) |
| Body text | Calibri | 12pt | `#454547` |
| Caption / label | Calibri | 9-10pt | `#828287` |

### Slide Layout Patterns

**Title slide (dark):**
- Background: `#062810`
- Logo: `logo-white.png`, top-left
- Title: Georgia 32pt bold, white, left-aligned
- Subtitle: Calibri 14pt, `#7CB77F`, left-aligned below title
- Short horizontal rule (`#7CB77F`) below subtitle

**Content slide (light):**
- Background: `#F7F3E8`
- Logo: `logo-dark.png`, top-left
- Page title: Georgia 24pt bold, `#09493D`, left-aligned
- Short horizontal rule (`#09493D`) below title
- Section headers: Georgia 13pt bold, `#09493D`
- Left accent bar: thin vertical bar in `#09493D` or `#7CB77F` before each section
- Body text: Calibri 12pt, `#454547`

### Reference Slide Images

- `assets/brand-slide-dark.jpg` — Title slide (dark green background, white logo, white title)
- `assets/brand-slide-light.jpg` — Content slide (cream background, dark logo, green headings)
