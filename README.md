# QuickCart 2025 Business Performance Review

This is an analyst's read of QuickCart's 2025 order data. Every number in this document comes directly from the notebook. Every interpretation is labeled as such, and nothing here claims causation where the data only shows correlation or association.

## What's in this repo

| File | What it is |
|---|---|
| notebook/quickcart_analysis.ipynb | The full analysis. Cleaning, audit, and all required areas, with inline reasoning and Dashboard |
| data/quickcart_cleaned.csv | The cleaned dataset, post audit |
| requirements.txt | Python dependencies |

## 1. The Problem, As Given

QuickCart grew through 2025 across products, sellers, regions, and acquisition channels. This analysis checks whether that growth is translating into profit, and where discounting, returns, and delivery issues may be creating risk. Four questions were asked.

1. Where is QuickCart making revenue and profit, and how did that move across the year?
2. Which discounts, products, categories, sellers, channels, or customer groups look economically weak?
3. Where are delivery, returns, and ratings pointing to operational or customer experience risk?
4. What should change next quarter, based on what the data shows?

## 2. How This Was Done

**Step 1. Data Audit.** Every calculated field was checked against its own formula before being trusted. Nulls were investigated for why they existed before deciding whether to fill them.

**Step 2. KPI Definitions.** Fixed once, used consistently through the analysis.

**Step 3. Required analysis areas**, each following the same shape: aggregate the metric, chart it, note what the result actually shows.

**Step 4. Recommendations**, built only from what the analysis in Steps 1 to 3 actually found.

## 3. Data Audit: What Was Found and What Was Done

The raw dataset had 30,035 rows and 37 columns.

| Check | Finding | Decision |
|---|---|---|
| Duplicate rows / `order_id`s | 35 exact duplicate rows found | Dropped. Final dataset: 30,000 rows |
| Categorical values (category, seller_tier, channel, device, delivery/order status, return_reason, segment, city_tier, warehouse, state) | All values inspected. No spelling or case variants found | Kept as is |
| `customer_age` nulls | 339 rows | Filled with median |
| `seller_tier` nulls | 119 rows | Each `seller_id` maps to exactly one tier across all its orders, so nulls were filled from the seller's own known tier (forward/backward fill within seller_id). 0 nulls remained after |
| `return_reason` nulls | 28,209 rows | Confirmed these are all orders with `return_flag` = 0 (not returned). Expected, not missing data. Kept as is |
| `coupon_code` nulls | 15,116 rows | Investigated further, see below |
| `campaign_id` nulls | 6,583 rows | Investigated further, see below |
| `rating` nulls | 1,914 rows | Confirmed concentrated in Cancelled (523), Processing (10), Shipped (36), and part of Returned (77) orders, none of which have a rating yet by definition. 1,268 nulls also exist among Delivered orders. Kept as is |
| `coupon_code` null but `discount_pct` > 0 | 14,632 rows | These are legitimate discounts applied without a coupon code |
| `coupon_code` present but `discount_pct` = 0 | 73 rows | Noted, not corrected |
| Of the 14,632 no-coupon discounted orders, how many also have no `campaign_id` | 3,674 rows | Checked these for anomalies. Discount amounts averaged ₹477 on an average 8.8% discount, in line with the rest of the dataset. Treated as a legitimate discount channel outside coupons and campaigns, not an error |
| `campaign_id` null rate by channel | 100% null for Organic Search, 0% null for every other channel | Expected. Organic traffic isn't campaign driven. Kept as is |
| Range checks: units ≤ 0, unit_price ≤ 0, discount_pct outside 0 to 100, actual_delivery_days outside 0 to 60, customer_age outside 10 to 90, rating outside 1 to 5 | 0 violations on every check | No action needed |
| `discount_amount_inr` vs `gross_sales_inr × discount_pct` | Average difference ₹0.14, max ₹7.83 across the full dataset. 12 rows had a difference above ₹5 | Checked these 12 rows as a percentage of their own expected discount value; the largest relative gap was 0.63%. Treated as rounding, not an error |
| `gross_sales_inr` vs `units × unit_price_inr` | 41.68% of rows did not reconcile | Tested against a "unit_price already reflects the discount" hypothesis; the gap between implied price and price-after-discount did not consistently support that either. Root cause not conclusively identified |
| `net_sales_inr` vs `gross_sales_inr − discount_amount_inr + shipping_fee_inr` | 0 mismatches across all 30,000 rows | Confirmed fully consistent |
| `profit_inr` vs `net_sales_inr − cogs_inr − logistics_cost_inr − payment_fee_inr − marketing_allocated_inr` | 0 mismatches across all 30,000 rows | Confirmed fully consistent |
| Decision on `gross_sales_inr` | Since net sales and profit both reconcile correctly using the given `gross_sales_inr`, and the mismatch could not be explained, `gross_sales_inr` was kept as given and not recalculated | Flagged as a known, unresolved limitation |
| Negative profit orders | 35 orders | Concentrated in Grocery/Silver tier (13), Grocery/Gold (8), Grocery/Bronze (4), with a few in Books & Stationery and Health & Wellness. Average of these 35: net sales ₹291.79, costs driven mainly by marketing allocation (₹4,673.60 total) and logistics (₹3,835.48 total) against small order values. Confirmed cost-driven, not a calculation error |
| Cancelled orders with `gross_sales_inr` > 0 | 523 orders | Flagged. Gross sales figures still include cancelled orders |
| Returned orders total net sales / profit | ₹9,874,912 net sales, ₹4,467,340 profit sitting in returned orders | Flagged for awareness. Not excluded from headline KPIs in this version of the analysis |
| Delivery status marked "Delayed" but `actual_delivery_days` ≤ `promised_delivery_days` | 0 rows | Confirmed consistent |
| Delivery status marked "On Time" but `actual_delivery_days` > `promised_delivery_days` | 0 rows | Confirmed consistent |
| `customer_segment` consistency per `customer_id` | 0 customers with more than one segment value | Confirmed stable |
| `customer_state` consistency per `customer_id` | 0 customers with more than one state value | Confirmed stable |
| `customer_city` to `customer_state` mapping | 0 cities mapped to more than one state | Confirmed stable |
| `product_id` to `category` mapping | 0 products mapped to more than one category | Confirmed stable |
| Daily order volume, zero-order days | 0 days with zero orders across the full year | Confirmed no gaps in the order timeline |
| `customer_age` consistency per `customer_id` | 307 customers show more than one age value across their orders; differences range up to 23 years | Cannot be corrected with the data available. Flagged as a limitation for any age-based analysis |
| Duplicate customers under different IDs, checked via city/age/device pattern | No conclusive pattern found; some repeated combinations exist but can't be confirmed as duplicate accounts versus a common customer profile | Flagged as a known limitation, not corrected |
| `marketing_allocated_inr` for Organic Search orders | 0 orders with nonzero marketing cost | Confirmed, consistent with campaign_id findings above |

## 4. KPI Definitions

| KPI | Formula | Notes |
|---|---|---|
| Net Sales | `sum(net_sales_inr)` | Used as the primary revenue figure throughout, since it's fully formula-validated |
| Gross Sales | `sum(gross_sales_inr)` | Used where explicitly noted; carries the unresolved reconciliation gap above |
| Profit | `sum(profit_inr)` | Fully formula-validated |
| Profit Margin % | `profit_inr / net_sales_inr × 100` | |
| AOV | `net_sales_inr / order count` | |
| Return Rate % | `mean(return_flag) × 100` | |
| On Time Delivery % | `mean(delivery_status == 'On Time') × 100` | |
| Marketing Efficiency | `profit_inr / marketing_allocated_inr` | Only meaningful for channels with nonzero marketing spend |

## 5. Findings, By Area

### Sales & Trend
Monthly net sales ranged from a high of ₹13,599,527.85 in January to a low of ₹11,085,158.74 in July, without a sharp structural break in between. Profit margin held close to 46% across the early months checked (45.98% in January, 46.21% in February, 46.07% in March). January had the highest order volume of any month, while October produced a slightly higher profit despite fewer orders moving. Order volume and profit did not move in lockstep every month.

### Product & Category Performance
Electronics is the top category on both net sales (₹63,904,887.56) and profit (₹25,751,804.20), on a comparatively lower order count (2,520 orders) and the highest average sale value per order (₹25,359.08). Its profit margin (40.30%) is actually the lowest among the top categories shown, but the order-level economics are strong enough to make it the top performer overall.

Fashion, Home & Kitchen, Sports & Fitness, and Beauty & Personal Care all carry higher profit margins (roughly 49 to 51%) than Electronics, but generate far less total profit because their per-order value is much smaller (Fashion averages ₹4,383.48 per order, against Electronics' ₹25,359.08).

At the individual product level, the top performer by profit is P0084 (₹6,105,799.23 profit across 248 orders). The bottom of the product list is a mix of categories, not concentrated in one, and most of the lowest performers by profit are lower-priced, higher-frequency items rather than clear underperformers.

### Discount & Profitability
Discount percentages in the dataset range from 0% to 30%. Profit margin by discount band shows a gradual decline as discount increases (46.81% margin in the 0 to 10% band, 46.33% in the 10 to 20% band). The correlation between `discount_pct` and `profit_inr` across the full dataset is -0.055, which is a weak correlation. This directly means discounting alone is not a major driver of profit outcomes in this data; the gradual margin decline by band exists, but it is not a strong or dominant relationship.

Checking the discount-to-profit pattern separately within each category showed no evidence that any one category's discounting behaves differently from the others; the pattern is consistent across categories.

The 10 to 20% discount band carries the most order volume (51.3% of all orders), but this cannot be read as discounting driving volume, since the highest discount band does not carry proportionally more orders.

### Customer Analysis
New customers are the largest segment by both order count and revenue/profit contribution. VIP customers show a higher average order value than New or Returning, but a smaller share of total order volume. New customers receive the heaviest discount exposure of the three segments, with the highest concentration of orders sitting in the higher discount bands, while Returning and VIP customers see comparatively less discounting.

### Channel & Campaign Efficiency
Organic Search produced the highest profit of any channel (₹15,404,764.20 across 6,583 orders) with zero marketing cost attached, which is expected given no campaign spend is allocated to organic traffic. Direct followed closely (₹14,311,615.87 profit across 5,952 orders) on a small marketing cost (₹104,136.66 total). Paid channels carry actual marketing spend and a higher concentration of discounted orders directed at new customers, which puts real pressure on their net economics compared to Organic and Direct.

### Delivery & Operations
On-time delivery percentage varies by warehouse, from a high of 64.77% (WH-DEL) to a low of 45.35% (WH-JAI). Platinum-tier sellers have the highest on-time rate among seller tiers (60.40%), followed by Gold (59.75%), Silver (58.03%), and Bronze the lowest (51.82%). By city tier, Tier 1 and Tier 2 cities perform similarly (60.09% and 59.73% on-time respectively), while Tier 3 cities lag at 53.06%. Checking warehouse performance broken down by city tier shows Tier 3 delays appear consistently across most warehouses, not isolated to one facility, which points to a geography-linked pattern rather than a single warehouse problem.

### Returns & Ratings
Return rate by category is highest in Fashion (7.91%) and Electronics (7.66%), and lowest in Toys & Kids and similar categories. Return rate by channel is fairly close across all channels, ranging narrowly from 5.54% (Organic Search) to 6.31% (Social), with no channel standing out sharply. Return rate by seller tier is highest for Platinum (6.54%) and lowest for Gold (5.71%), which runs counter to an assumption that higher seller tier means fewer returns.

The most common return reason overall is "Not as Expected" (412 cases), ahead of Late Delivery (352), Wrong Item (269), Damaged (268), Size/Fit Issue (264), and Changed Mind (226). Breaking "Not as Expected" down by channel shows it appears more heavily in Paid Search and Affiliate orders than other channels. This is consistent with a possible listing or marketing accuracy issue, though the data does not confirm the exact cause.

## 6. Recommendations



1. Investigate "Not as Expected" returns as a listing or marketing accuracy problem, starting with Paid Search and Affiliate.
2. Prioritize last-mile delivery investment in Tier 3 cities.
3. Re-examine the assumption that Platinum-tier sellers are the safest bet, using return rate as the trigger.
4. Resolve the gross sales data inconsistency before it's relied on for any pricing or reporting decision.
5. Re-examine the assumption that Platinum-tier sellers are the safest bet, using return rate as the trigger.

## 7. Limitations & Assumptions

- `gross_sales_inr` does not reconcile with `units × unit_price_inr` for 41.68% of orders. Root cause not identified. `net_sales_inr` and `profit_inr` both reconcile independently and correctly, so this does not affect the KPIs used in this analysis, but it is an open question worth raising if asked.
- 307 customers show `customer_age` values that differ by more than what one year of natural aging would explain (up to 23 years difference). Treated as a data collection inconsistency, not corrected.
- Duplicate customers under different `customer_id`s could not be conclusively confirmed or ruled out from the available fields.
- The discount to profit relationship is a weak correlation (-0.055), not a proven cause. No causal claim is made from this analysis.
- Returned orders (₹9,874,912 in net sales, ₹4,467,340 in profit) and cancelled orders (523, still carrying gross sales figures) remain inside the headline revenue and profit numbers used in this version of the analysis, and have not been separately excluded.

## 8. Executive Summary
QuickCart's 2025 order volume and revenue held steady through the year, with no major structural collapse or spike. But steady top-line numbers are hiding two real problems underneath.

First, the business is spending its heaviest discounts on new customers (over half their orders sit in higher discount bands) without yet knowing if that spend is building a repeat customer base or just buying one-time volume. Discounting overall has only a weak relationship with profit margin (correlation of -0.055), so it isn't the dominant driver of profitability, but the new-customer discount pattern specifically is unproven spend until it's tracked against actual repeat behavior.

Second, customer experience has two concrete, traceable weak points. Returns are most often driven by "Not as Expected," the single largest return reason in the data, and it shows up more in Paid Search and Affiliate orders than other channels, a signal worth checking against listing and ad accuracy. Separately, Tier 3 cities run meaningfully behind Tier 1 and Tier 2 on on-time delivery (53% vs roughly 60%), and that gap repeats across nearly every warehouse, pointing to a last-mile infrastructure issue rather than one facility's problem.

Category economics are healthy but uneven: Electronics drives the largest share of total profit on high order value despite a lower margin percentage, while categories like Fashion and Home & Kitchen carry stronger margins but smaller order values, so they contribute less in absolute terms. Neither pattern is a problem, but the two categories shouldn't be judged by the same yardstick.

Five actions are recommended, in priority order: test new-customer discount ROI against actual repeat rate, audit listing and marketing accuracy for Paid Search and Affiliate given the "Not as Expected" pattern, invest in Tier 3 last-mile delivery, investigate why Platinum-tier sellers show the highest return rate of any tier, and resolve a data inconsistency in how gross sales is calculated before it's used for any pricing decision. Full evidence for each is in Section 6.
