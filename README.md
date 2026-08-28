# Olist Marketplace Analytics

A Power BI report built on a PostgreSQL data warehouse I designed and loaded myself, covering
25 months of activity on a Brazilian e-commerce marketplace.

The dashboard is the visible half. The part underneath it is a full pipeline: two source systems
landing in a staging area, cleansed into a 3NF core, published as a star schema, and only then
read by Power BI.

> **Portfolio exercise.** Built on the public Olist Brazilian e-commerce dataset from Kaggle.
> Not affiliated with, endorsed by, or produced for Olist. The company name and mark appear only
> because the dataset is theirs and inventing a different name would misrepresent where the data
> came from.

---

## Pipeline

```
source_system_1.csv  (order management)  ─┐
                                          ├─→  SA_OMS / SA_MFS  ─→  BL_3NF  ─→  BL_DM  ─→  Power BI
source_system_2.csv  (fulfilment)        ─┘        staging          3NF core     star
```

| | |
|---|---|
| Fact rows | 583,250 at order-item grain |
| Period | 18 June 2024 to 25 July 2026 |
| Customers | 98,670 (slowly changing, type 2) |
| Sellers | 3,096 |
| Products | 32,952 across 71 categories |
| Reviews | 117,403 |
| Fact partitioning | range, one partition per month |

All ETL lives in the warehouse as stored procedures, with a single master procedure running the
chain in dependency order. The mart layer is what Power BI connects to, in Import mode.

---

## The report

Three pages for the reader, two hidden ones doing work behind them.

### Overview

![Overview](images/01-overview.png)

Monthly revenue leads, because the first question anyone asks is whether the business is growing.
Four cards sit underneath it, chosen so money and service appear side by side. A report showing
only revenue lets a manager conclude everything is fine while delivery performance quietly slips.

The line drops at the final point because the data stops on 25 July 2026, so that month is about
two thirds complete. It is left in rather than trimmed.

### Products and sellers

![Products and sellers](images/02-products-sellers.png)

Revenue and freight are plotted side by side per category, so the freight bite is visible per
category instead of being buried in one total. The scatter puts every seller on revenue against
average review score, with bubble size showing volume.

Ratings converge as revenue grows. The interesting cases are the exceptions: a seller carrying
over a million reais at a 3.3 average is a commercial problem, because the volume means a lot of
customers had that experience.

### Delivery and satisfaction

![Delivery and satisfaction](images/03-delivery-satisfaction.png)

Delivery outcomes, how long delivery actually takes, how reviews are distributed, freight as a
share of price over time, and the ten weakest states for on-time delivery.

The delivery-time histogram keeps its full tail out to 150 days. Cutting the axis at 60 would make
a tidier chart and would hide the fact that those orders exist.

### Drillthrough to order detail

![Order detail](images/04-order-detail.png)

Right-clicking any category or seller opens a hidden detail page carrying that filter, with a
heading that names whatever was drilled into.

---

## Data model

![Data model](images/05-data-model.png)

The interesting part is the six dashed lines into the date dimension.

The fact table records six dated events for every order item: purchase, payment approval, the
shipping limit agreed with the seller, handover to the carrier, delivery, and the delivery date
originally estimated. Each is a separate foreign key into one date dimension.

Power BI allows one active relationship per path, so five of the six are inactive. Purchase date is
active, because that is the date the business thinks in. The others are reached through
`USERELATIONSHIP` inside a measure when a specific question needs them.

That single design fact drives most of the DAX in this model, including why the calculated columns
use `LOOKUPVALUE` rather than `RELATED`. See [dax/measures.md](dax/measures.md).

---

## What the report shows

**Freight is 16.5% of revenue.** R$11.66M of R$70.51M gross goes on shipping, which is why net
revenue sits on the overview page next to gross rather than being left to be inferred.

**Alagoas is the outlier worth acting on.** R$410K of revenue at a 77.8% on-time rate and a 3.59
average review, against 92.8% and 3.88 nationally. A revenue-only report would show a small,
unremarkable state.

**Reviews are J-shaped.** Fives dominate, ones are the second largest group, and the middle is
thin. People who are delighted and people who are angry both write reviews; everyone in between
mostly does not.

**About 12,000 order items never arrived at all.** They are excluded from the on-time rate, because
an order still in transit is neither on time nor late, and counting it would drag the percentage
down for a reason unrelated to delivery performance.

---

## Techniques used

| | |
|---|---|
| Role-playing dimensions | six date keys, one active, `USERELATIONSHIP` for the rest |
| Calculated columns | 5, including delivery duration and on-time classification |
| Measures | 14, in a dedicated `_Measure` table |
| Dynamic titles | chart title follows whatever state filter is applied |
| Drillthrough | hidden detail page, filtered by category or seller |
| Report page tooltip | hover a state on the map for a small dashboard about it |
| Conditional formatting | rules-based thresholds, not a gradient |
| Bookmarks | reset-filters control, Data captured and Display ignored |

![Dynamic title responding to a filter](images/07-dynamic-title.png)

Published to the Power BI Service:

![Published report](images/06-published-service.png)

---

## Repo contents

```
images/               page screenshots
dax/measures.md       every measure and calculated column, with the reasoning
theme/                the report theme as JSON
*.pbix                the Power BI file
```

The `.pbix` is included for completeness. It will not refresh without the PostgreSQL warehouse
behind it, but the model, the DAX and the report layout are all inspectable.

---

## Stack

PostgreSQL 17 · Power Query (M) · DAX · Power BI Desktop and Service

## Credits

Data derived from the [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)
on Kaggle, expanded to a larger volume across two synthetic source systems for the warehouse build.
