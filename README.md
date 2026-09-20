# Braincircle-data-analytics challenge 2026
### QuickCart 2025 Business Performance Review

This is an analyst's read of QuickCart's 2025 order data — where the money is coming from, where it's quietly leaking, and what's worth fixing first. It's built to be defensible: every number here traces back to a cell in the notebook, and every claim about "why" is flagged as a hypothesis unless the data actually proves it.

## What's in this repo

| File | What it is |
|---|---|
| `notebook/quickcart_analysis.ipynb` | The full analysis — cleaning, audit, and all nine required areas, with inline reasoning |
| `data/quickcart_cleaned.csv` | The cleaned dataset, post-audit, used for both the notebook and the dashboard |
| `dashboard/quickcart_dashboard.pbix` | Power BI dashboard — interactive cuts across category, channel, customer, ops, returns *(add once built)* |
| `executive_summary.pdf` | One-page version of the findings and recommendations below *(add once built)* |
| `requirements.txt` | Python dependencies |

## 1. The Problem, As Given

QuickCart grew through 2025 — more products, more sellers, more regions, more ways to acquire customers. But growth on its own doesn't tell leadership anything about health. The real question sitting underneath this whole exercise: **is QuickCart's revenue actually converting into profit, or is some of it being spent right back out through discounts, returns, and delivery failures?**

Four questions were asked, and this analysis answers all four:
1. Where is QuickCart making revenue and profit, and how did that move across the year?
2. Which discounts, products, categories, sellers, channels, or customer groups look economically weak?
3. Where are delivery, returns, and ratings pointing to operational or customer-experience risk?
4. What should change next quarter, based on what the data actually shows?

## 2. How This Was Done

**Step 1 — Data Audit.** Before trusting a single KPI, every calculated field was checked against its own formula: does `gross_sales_inr` actually equal `units × unit_price_inr`? Does `net_sales_inr` reconcile against discount and shipping? Does `profit_inr` reconcile against all five cost components? Nulls were investigated for *why* they were missing before deciding whether to fill them. Full log below.

**Step 2 — KPI Definitions.** Fixed once, used consistently everywhere in this analysis (Section 4).

**Step 3 — Nine analysis areas**, each following the same shape: define the metric, aggregate it, chart it, write down what it actually means for the business — not just what the chart shows.

**Step 4 — Recommendations**, built only from what showed up in Steps 1–3, not from assumptions.

## 3. Data Audit — What Was Found and What Was Done

| Check | Finding | Decision |
|---|---|---|
| Duplicate rows / `order_id`s | Present, confirmed as exact duplicates | Dropped |
| Categorical values (category, seller_tier, channel, etc.) | All clean — no spelling/case variants found | Kept as-is |
| `customer_age` nulls | Present | Filled with median |
| `return_reason` nulls | Null only where `return_flag` = No | Kept as-is — this is expected, not missing data |
| `coupon_code` nulls with `discount_pct` > 0 | ~3,600+ orders discounted with no coupon and no campaign | Investigated — discount amounts for these were small and consistent with the discount formula, so treated as a legitimate non-coupon discount channel, not an error |
| `campaign_id` nulls | Concentrated in Organic Search | Expected — organic traffic isn't campaign-driven. Kept as-is |
| `seller_tier` nulls | Present | Confirmed each `seller_id` maps to exactly one tier; filled via forward/backward-fill within seller_id |
| `rating` nulls | Concentrated in Cancelled/Processing/Shipped orders | Expected — no rating exists yet for undelivered orders. Kept as-is |
| Range checks (units, price, discount %, delivery days, age, rating) | No violations found | No action needed |
| `discount_amount_inr` vs `gross_sales_inr × discount_pct` | Small rounding differences (~₹0.13 avg); 12 rows with larger gaps, max 0.63% of expected value | Treated as consistent — rounding, not a data error |
| `gross_sales_inr` vs `units × unit_price_inr` | A meaningful share of rows didn't reconcile; tested against a "price already reflects discount" hypothesis — didn't fully explain it either | Root cause not conclusively identified. Since `net_sales_inr` and `profit_inr` both reconcile correctly against the *given* `gross_sales_inr`, it was kept as the authoritative field and not recalculated. Flagged as a known limitation |
| `net_sales_inr` formula | Reconciles correctly against gross sales, discount, and shipping | Confirmed valid |
| `profit_inr` formula | Reconciles correctly against net sales and all cost fields | Confirmed valid |
| Negative profit orders | Present | Investigated — driven by high logistics and marketing cost, not calculation errors. Kept as genuine, not cleaned |
| Cancelled orders in revenue fields | Gross sales still populated for cancelled orders | Flagged — revenue KPIs should be read with this in mind |
| Delivery status vs delivery days mismatch | None found | Confirmed consistent |
| `customer_segment` consistency per customer | No inconsistencies | Confirmed stable |
| `customer_state` consistency per customer | No inconsistencies | Confirmed stable |
| `customer_city` → `customer_state` mapping | No inconsistencies | Confirmed stable |
| `product_id` → `category` mapping | No inconsistencies | Confirmed stable |
| `customer_age` consistency per customer | Some customers show age differences too large for one year of data | Cannot be corrected with what's available. Flagged as a limitation for any age-based analysis |
| Daily order volume | Checked for zero-order gaps | *(confirm result from notebook before finalizing this line)* |
| Duplicate customers under different IDs | Cannot be conclusively identified from city/age/device patterns alone | Flagged as a known limitation, not corrected |

## 4. KPI Definitions

| KPI | Formula | Why this definition |
|---|---|---|
| Gross Sales | `sum(gross_sales_inr)` | Top-line, as billed to the customer |
| Net Sales | `sum(net_sales_inr)` | After discount and shipping — the real transaction value |
| Profit | `sum(profit_inr)` | Net sales minus COGS, logistics, payment fees, and allocated marketing |
| Profit Margin % | `profit_inr / net_sales_inr × 100` | The core health check — tells you if revenue is actually worth having |
| AOV | `net_sales_inr / order count` | Average basket size |
| Return Rate % | `mean(return_flag) × 100` | Share of orders returned, by any cut |
| On-Time Delivery % | `mean(delivery_status == 'On Time') × 100` | Operational reliability |
| Marketing Efficiency | `profit_inr / marketing_allocated_inr` | Profit generated per rupee of allocated marketing spend, by channel |

## 5. Findings — By Required Area

### A. Sales & Trend
Monthly order volume, sales, and profit stayed largely stable across 2025 — no sharp structural break, no collapse, no runaway spike. January carried the most order volume; October carried the strongest profit. That gap matters: **more orders moving didn't automatically mean more profit generated** — a small early signal of the volume-vs-margin tension this whole review is built around.

### B. Product & Category Performance
Electronics leads on profit despite lower order volume — high unit price, high margin per order. Grocery moves in much higher volume with a strong margin percentage, but its low price point means smaller profit per order, so its total profit contribution trails Electronics. Neither pattern is a problem on its own — but leadership should know these two categories are winning for structurally different reasons, and shouldn't be graded on the same yardstick.

### C. Discount & Profitability
Profit margin declines gradually as discount bands increase — the 30%+ discount band shows the weakest margins in the data. This is a correlation, not a proven cause; it does not by itself mean discounting *causes* the margin loss, since heavier discounting may cluster with categories or customer types that were already lower-margin. What is defensible: **higher discount bands are consistently associated with weaker unit economics**, and that association held when checked across categories.

The 10–20% discount band carries the most order volume — but the highest discount band does not carry proportionally more orders, which weakens any argument that "more discount = more volume" in a simple, linear way.

### D. Customer Analysis
New customers are the largest single contributor to both revenue and profit. VIP customers have a meaningfully higher AOV per order but a far smaller share of total volume. New customers are also the group receiving the heaviest discounting — over half of new-customer orders in the 20–30% discount band, versus much lower discount exposure for Returning and VIP customers. That raises a real question: **is QuickCart discounting new customers to acquire them, and if so, is that spend converting into Returning/VIP behavior, or just buying one-time volume?** The data can flag this pattern; it can't yet confirm the conversion outcome.

### E. Channel & Campaign Efficiency
Organic Search delivers the highest profit at zero marketing cost — expected, since organic traffic carries no acquisition spend, but a useful reminder not to over-credit paid channels without netting out cost. Direct is close behind with low but nonzero marketing cost. Paid channels (Paid Search, Affiliate, Social) show a heavier discount mix concentrated on new customers, which combined with actual marketing spend puts real pressure on their net economics.

### F. Delivery & Operations
On-time delivery performance is uneven by warehouse — the strongest warehouse ran meaningfully more reliably than the weakest. Platinum-tier sellers deliver on time more consistently than Bronze-tier sellers. Tier 3 cities underperform Tier 1 and Tier 2 on on-time delivery across nearly every warehouse — this isn't isolated to one facility, it's a consistent geography-linked pattern, which points toward last-mile infrastructure in smaller cities rather than any single warehouse being the problem.

### G. Returns & Ratings
Fashion and Electronics carry the highest return rates of any category. "Not as Expected" is the single largest return reason — ahead of late delivery, wrong item, damage, and size/fit combined. That reason concentrates more heavily in orders from Paid Search and Affiliate channels, which is worth flagging directly: it's consistent with a listing-accuracy or marketing-accuracy issue (photos, descriptions, or ad claims not matching the product), though the data can't confirm the exact cause. Return rate is highest among Platinum-tier sellers — a counter-intuitive result worth investigating rather than assuming top-tier sellers are automatically the safest bet.

## 6. Recommendations

*(To be finalized against the numbers above — see notebook Section I)*

1. ...
2. ...
3. ...
4. ...
5. ...

## 7. Limitations & Assumptions

- `gross_sales_inr` does not reconcile with `units × unit_price_inr` for a share of orders; root cause undetermined. Downstream KPIs rely on `net_sales_inr`/`profit_inr`, which reconcile independently, so this does not affect the analysis above.
- Some `customer_id`s show age variation too large for natural one-year aging — treated as a data-collection inconsistency, not corrected.
- Duplicate customers under different `customer_id`s could not be conclusively identified.
- `marketing_allocated_inr` is a modeled/allocated figure, not verified per-order ad spend — channel efficiency numbers should be read as directional, not exact.
- Discount-profit relationship is reported as a correlation. No causal claim is made or should be inferred from this analysis alone.

## 8. Executive Summary

*(Short version for leadership — see `executive_summary.pdf` or Section 6+ above)*
