# Olist E-Commerce Analytics: Sales, Delivery and Customer Satisfaction

An end-to-end analytics project on the Olist Brazilian marketplace dataset. A documented Python cleaning pipeline feeds a multi-page Power BI report that answers one main question: **how does delivery performance affect customer reviews?**

![Overview page](images/overview.png)

## Key findings

All figures are for delivered orders, Jan 2017 to Aug 2018, from the cleaned data. Check them against the live report before you quote them.

- **Scale:** about R$13.2M in product sales from roughly 96,000 delivered orders. Average order value including freight is about R$160. Freight is about 17% of product sales.
- **Concentration:** São Paulo accounts for about 39% of sales, and the top 10% of sellers hold about two thirds of sales.
- **Retention:** only 3.12% of customers placed more than one order.
- **Delivery:** orders take 12.1 days on average, about 11.8 days earlier than the estimate. Estimates are padded, so only 5.94% of deliveries are late.
- **Where late deliveries cluster:** the north and northeast (for example AL, MA, SE), and also Rio de Janeiro, the second-largest market, at about 10%.
- **Reviews:** the average score is 4.09. Late orders average 2.27 stars against 4.29 for on-time orders, and 54% of late-order reviews are 1 star compared with 9% for on-time orders.
- **Caution:** late delivery is strongly associated with bad reviews, but many unhappy reviews come from orders that arrived on time. The data shows association, not cause.

## Business questions

1. How are sales trending, and what drives them (categories, states, seasonality)?
2. Where and when do deliveries run late?
3. Do late deliveries lower review scores?
4. Which sellers and categories need attention?

## Dataset

[Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) (Kaggle): 9 CSV files, about 100,000 orders from 2016 to 2018. The raw data is not stored in this repository. Download it and check the dataset page for its current license terms (it has been published under CC BY-NC-SA 4.0).

## Repository structure

```
├── README.md
├── notebooks/
│   └── datacleaning.ipynb          # profiling + cleaning pipeline
├── data/
│   ├── raw/                        # place the 9 Kaggle CSVs here (not committed)
│   └── cleaned/                    # output of the notebook (not committed)
├── powerbi/
│   ├── olist_report.pbix
│   └── olist_theme_light.json
├── python-visuals/
│   └── seaborn_visual_template.py  # styled Python visual used in Power BI
└── images/                         # report screenshots
```

## Cleaning pipeline (Python)

The notebook profiles every file, then cleans it. Each decision was made by checking what a null or duplicate means, not by applying a blanket rule.

| Finding | Decision |
|---|---|
| Zip codes read as numbers lose leading zeros | Read as text, padded to 5 digits |
| Dates stored as text | Converted to datetimes. Null delivery dates kept, since they mean "not delivered" |
| 8 delivered orders with no delivery date; 6 canceled orders with one | Flagged (`delivered_missing_date`, `canceled_with_delivery`), not removed |
| 775 orders without items; 1 order without a payment | Flagged (`has_items`, `has_payment`) |
| Review IDs repeat across orders (814); 551 orders have several reviews | One review per order, the latest. 99,224 rows became 98,673 |
| 610 products without a category; 2 categories missing a translation; a typo in one translation | `unknown`, manual mapping, typo fixed |
| 4 products with weight 0 | Set to null with a flag. Weight and dimensions are kept |
| Geolocation has about 1,000,000 rows for 19,015 zips | One row per zip with median coordinates. 29 out-of-range coordinates blanked, not dropped |
| 278 customers (157 zips) and 7 sellers have zips absent from the source | Left unmatched and flagged (`has_geo_match`, `has_coordinates`) |
| Hand-typed seller cities (`lages - sc`, state names, an email) | `seller_city_clean` fixes 34 sellers; the original column is kept |
| Carrier dates before purchase or after delivery | 189 orders flagged (`carrier_date_suspect`) |
| 9 zero-value payments, 2 credit-card payments with 0 installments, `not_defined` type | Flagged, and `not_defined` relabeled `unknown` |

Derived fields: `delivery_days`, `delivery_delay_days`, `is_late` (late = delivered at least one full day after the estimate), `order_purchase_date`, `product_volume_cm3`, `review_response_days`.

Row counts exported: customers 99,441, geolocation 19,015, orders 99,441, order_items 112,650, payments 103,886, reviews 98,673, products 32,951, sellers 3,095.

## Data model

- `orders` is the hub. It connects to `customers` (1:1), `reviews` (1:1, after deduplication), and one-to-many to `order_items` and `payments`.
- `order_items` connects to `products` and `sellers`.
- `geolocation` is used twice (customer zip and seller zip) as a role-playing dimension: one copy per relationship.
- A `Calendar` table is related to `orders[order_purchase_date]` and marked as the date table.
- Sales and categories use `order_items[price]`. Payments are per order and are not split by category.

## Report pages

| Page | Purpose |
|---|---|
| Overview | KPIs, monthly trend, top categories, sales by state, payment mix |
| Delivery Performance | Late rate by state and month, delivery time, delivery buckets |
| Customer Satisfaction | Score by delay bucket, star mix for late vs on-time orders, lowest-scoring categories, state scatter, box plot of delay by score |
| Products and Sellers | Seller concentration, freight vs weight, seller scatter, state scorecard |
| Seller Detail | Drill-through page for one seller, with a comparison against all sellers |
| Insights (AI) | Key influencers and decomposition tree on low review scores |
| Data Notes | Cleaning log, definitions, assumptions |

## Key DAX measures

```DAX
Product Sales      = CALCULATE(SUM(order_items[price]), orders[order_status] = "delivered")
Late Delivery Rate = CALCULATE(AVERAGE(orders[is_late]), orders[order_status] = "delivered")
Avg Score (Late)   = CALCULATE([Avg Review Score], orders[is_late] = 1)
Avg Score (On Time)= CALCULATE([Avg Review Score], orders[is_late] = 0)

Sales MoM % =
VAR Cur  = [Product Sales]
VAR Prev = CALCULATE([Product Sales], DATEADD('Calendar'[Date], -1, MONTH))
RETURN IF(ISBLANK(Cur) || ISBLANK(Prev), BLANK(), DIVIDE(Cur - Prev, Prev))
```

## Power BI techniques used

Update the checklist as you finish each item.

- [x] Data modeling with a role-playing dimension and a marked date table
- [x] DAX measures and calculated columns
- [x] Drill-through page
- [x] Python (seaborn) visual styled to match the report
- [x] Custom page navigation
- [ ] Hierarchies and drill down
- [ ] Report-page tooltips
- [ ] Field and numeric parameters
- [ ] Bookmarks (reset, info panels)
- [ ] AI visuals (key influencers, decomposition tree)
- [ ] Performance analyzer, with before and after timings

## Assumptions and limitations

- **Sales** means item price (GMV), not Olist's income. Freight is reported separately.
- **Subscription revenue**, if shown, uses an assumed R$80 monthly fee. The dataset has no fee data.
- **Late** is measured in whole days, because the estimated date has no time of day. Comparing raw timestamps would give a higher late rate.
- **Date range:** analysis covers Jan 2017 to Aug 2018. 2016 and Sep to Oct 2018 are too sparse to trend.
- **Reviews:** one review per order is kept, so 551 older reviews are excluded.
- **Geography:** 278 customers and 7 sellers have no map coordinates. Slicing uses the complete state and city columns.
- **Associations, not causes:** key influencers and the box plot show relationships in observational data.

## How to reproduce

1. Download the dataset from Kaggle and put the 9 CSVs in `data/raw/`.
2. Install the requirements: Python 3.10 or later, `pandas`, `numpy`, `jupyter`. For the Python visual in Power BI, also `matplotlib` and `seaborn`.
3. Open `notebooks/datacleaning.ipynb`, set the `DATA` path in the first code cell, and run all cells. The cleaned files are written to `data/cleaned/`.
4. Open `powerbi/olist_report.pbix` and update the data source path if Power BI asks (Transform data, Data source settings).
5. Set Power BI's import locale to English (United States) under Options, Current file, Regional settings. With a comma-decimal locale, decimal columns import 10 to 100 times too large.

## Lessons learned

- A card total that looks plausible can still be wrong. Reconciling Power BI against Python caught an import locale bug that inflated sales about 32 times.
- A measure with a status filter can create phantom rows in a table. Group by the order ID.
- Filters do not flow from `order_items` back to `orders` and `reviews` by default, so category-level review scores need an explicit cross-filter.

## Tools

Python (pandas), Jupyter, Power BI Desktop, DAX, seaborn and matplotlib.

## Author

[Your name] | [LinkedIn] | [Email]

Dataset: Olist, via Kaggle. This is an independent portfolio project and is not affiliated with Olist.
