# DAX reference

Every calculated column and measure in the model, with the reasoning where a choice was not obvious.

Tables are named after what they hold rather than after the warehouse objects behind them:
`Order Items` is the fact, `Date` is the generated calendar, and `_Measure` is an empty table that
exists only to hold measures.

---

## Calculated columns

All three sit on the fact table, because each describes an individual order item rather than a
dimension member.

### Delivery Days

```dax
Delivery Days =
VAR Purchased = RELATED('Date'[Date])
VAR Delivered =
    LOOKUPVALUE('Date'[Date], 'Date'[time_day_surr_id], 'Order Items'[delivery_date_surr_id])
RETURN
    IF(NOT ISBLANK(Delivered), DATEDIFF(Purchased, Delivered, DAY))
```

Two dates fetched two different ways, and the reason is the relationship layout.

`RELATED` follows the active relationship, so it reaches the purchase date directly. It cannot reach
the delivery date, because that relationship is inactive and `RELATED` has no way to say which path
to take. `LOOKUPVALUE` ignores relationships entirely, matching a value in one table against a
column in another, so it works regardless of which relationship happens to be active.

The blank test handles orders that have not arrived. Their delivery key points at a default member
filtered out during load, so the lookup finds nothing.

### Delivery Status

```dax
Delivery Status =
VAR Delivered =
    LOOKUPVALUE('Date'[Date], 'Date'[time_day_surr_id], 'Order Items'[delivery_date_surr_id])
VAR Estimated =
    LOOKUPVALUE('Date'[Date], 'Date'[time_day_surr_id], 'Order Items'[estimated_delivery_date_surr_id])
RETURN
    SWITCH(TRUE(),
        ISBLANK(Delivered),    "Not delivered",
        Delivered > Estimated, "Late",
                               "On time")
```

`SWITCH(TRUE())` reads as a list of conditions tested in order, first match wins. The order matters:
the blank test has to come first, or an undelivered order falls through to the comparison and gets
labelled on time, quietly flattering the delivery figures.

### Freight Share

```dax
Freight Share = DIVIDE('Order Items'[Freight], 'Order Items'[Price])
```

`DIVIDE` rather than the division operator, so a product priced at zero returns blank instead of
throwing an error across the whole column.

### Year Month and Year Month Sort

```dax
Year Month      = FORMAT('Date'[Date], "YYYY-MM")
Year Month Sort = 'Date'[Year] * 100 + 'Date'[Month Number]
```

`Year Month` is set to sort by `Year Month Sort`, otherwise the axis sorts alphabetically.

That sort relationship has a consequence worth knowing: **a column sorted by another column drags
its sort column into the filter context**. Any measure that needs to lift the month filter has to
remove both, not just the label. See `Peak Revenue` below.

---

## Base measures

```dax
Revenue      = SUM('Order Items'[Price])
Freight Cost = SUM('Order Items'[Freight])
Net Revenue  = [Revenue] - [Freight Cost]
Items Sold   = COUNTROWS('Order Items')
```

Everything else is expressed in terms of these, so a change to the definition of revenue happens in
one place.

---

## Derived measures

```dax
Avg Order Value        = DIVIDE([Revenue], [Items Sold])
Avg Delivery Days      = AVERAGE('Order Items'[Delivery Days])
Freight Share of Price = DIVIDE([Freight Cost], [Revenue])
```

### Active Customers

```dax
Active Customers = DISTINCTCOUNT(Customers[Customer ID])
```

Counted on the business key rather than the surrogate. The customer dimension is slowly changing
type 2, so one person who has moved city holds several rows with several surrogate keys. Counting
surrogates would report that person more than once.

### Avg Review Score

```dax
Avg Review Score = AVERAGEX('Order Items', RELATED(Reviews[Review Score]))
```

The obvious version, `AVERAGE(Reviews[Review Score])`, is wrong here and it fails silently.

Sellers and Reviews both sit on the one side of a relationship, with the fact table between them on
the many side. Filters travel from the one side to the many side and stop. Selecting a seller
therefore filters the fact and gets no further, and the measure returns the average of every review
in the model no matter what is selected.

It looked correct on a card, where the answer happens to be the overall average anyway. It was only
visible on a scatter chart, where every seller landed on exactly the same horizontal line.

`AVERAGEX` iterates the fact table, which is filtered by the seller, and `RELATED` fetches each
row's score across the many-to-one relationship. That direction is allowed.

The alternative fix is setting the relationship to filter in both directions. That creates more than
one possible path between tables and makes later results harder to predict, so changing the measure
was the better trade.

### On Time Rate

```dax
On Time Rate =
DIVIDE(
    CALCULATE([Items Sold], 'Order Items'[Delivery Status] = "On time"),
    CALCULATE([Items Sold], 'Order Items'[Delivery Status] <> "Not delivered")
)
```

The denominator deliberately excludes orders that have not arrived yet. An order in transit is
neither on time nor late, and including it drags the percentage down for a reason that has nothing
to do with delivery performance.

This is why the on-time card reads 92.8 per cent while the delivery donut reads 90.8. Both are
correct; they divide by different things, and each title says which.

---

## Time intelligence

```dax
Revenue YTD = CALCULATE([Revenue], DATESYTD('Date'[Date]))
```

`DATESYTD` only works because the date table is marked as one. The fact joins to it on an integer
surrogate key, so without marking it Power BI has no way to know which column holds the calendar
date, and time intelligence either fails or returns a plausible wrong number.

Marking the table also requires a contiguous date range, which is why the default 1900-01-01 member
had to be filtered out during load.

### Revenue by Delivery Date

```dax
Revenue by Delivery Date =
CALCULATE([Revenue],
    USERELATIONSHIP('Order Items'[delivery_date_surr_id], 'Date'[time_day_surr_id]))
```

This is the measure that makes the five inactive relationships useful. Everything else in the report
reports on when an order was placed. This one reports on when it arrived, by switching the active
relationship for the duration of a single calculation.

---

## Text measures

### Revenue Chart Title

```dax
Revenue Chart Title =
VAR Picked = CONCATENATEX(VALUES(Customers[State]), Customers[State], ", ")
RETURN
    IF(ISFILTERED(Customers[State]),
       "Revenue by month: " & Picked,
       "Revenue by month: all states")
```

Bound to the chart title through the fx button, format style Field value.

`CONCATENATEX` rather than `SELECTEDVALUE`, which returns blank as soon as more than one value is in
context and would empty the title when two states are picked.

`ISFILTERED` asks whether the column is filtered by anything at all, so the title also follows a
selection made by clicking the map rather than only tracking the slicer.

### Detail Header

```dax
Detail Header =
VAR Cat = SELECTEDVALUE(Products[Category])
VAR Sel = SELECTEDVALUE(Sellers[Seller ID])
RETURN
    "Order items for " & COALESCE(Cat, "all categories")
        & IF(NOT ISBLANK(Sel), ", seller " & Sel, "")
```

Names whatever was drilled into on the detail page. The `IF` appends the seller only when one was
actually chosen, so the heading never reads "seller all sellers".

---

## Marking the peak

```dax
Peak Revenue =
VAR CurrentMonth = SELECTEDVALUE('Date'[Year Month])
VAR MonthlyRevenue =
    CALCULATETABLE(
        ADDCOLUMNS(VALUES('Date'[Year Month]), "@Rev", [Revenue]),
        REMOVEFILTERS('Date'[Year Month], 'Date'[Year Month Sort])
    )
VAR TopMonth = MAXX(TOPN(1, MonthlyRevenue, [@Rev], DESC), 'Date'[Year Month])
RETURN
    IF(CurrentMonth = TopMonth, [Revenue])
```

Returns blank for every month except the highest, so adding it as a second series puts a single
marker on the line.

Two traps are handled here.

`REMOVEFILTERS` names both `Year Month` and its sort column. Removing only the label leaves the sort
column filtering, the table collapses to one row, and every month compares equal to itself.

The comparison is on the month label, not the revenue. Comparing two floating point sums computed
down different evaluation paths can fail on all rows including the peak, which shows up as a chart
with no markers at all rather than too many.
