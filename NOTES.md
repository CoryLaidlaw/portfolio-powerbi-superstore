# Build Notes — Superstore KPI Dashboard

Working notes from the build. Decisions, what I learned, and the things I want to be able to explain cold.

---

## Data model

**Grain:** one row of the fact table = one line item on one order. Named this before splitting anything, because every count downstream depends on it.

**Star schema:** fact table (`Sample - Superstore`) surrounded by five dimensions.

| Table | Key | Rows |
|---|---|---|
| `dim_date` | Date | 1,826 |
| `dim_product` | Product ID | 1,862 |
| `dim_customer` | Customer ID | 793 |
| `dim_location` | location_key | 632 |
| `dim_shipmode` | Ship Mode | 4 |

Order ID stays in the fact table as a **degenerate dimension**. It is a pure identifier with no attributes of its own, so it does not get its own table.

All relationships are many-to-one (fact to dimension) with single-direction cross-filtering. Filtering flows from dimension into fact. Keeping it single-direction is what keeps the star predictable.

### Dimension table creation (Power Query)

1. Load CSV, Transform Data
2. Right-click source query > **Reference**, rename the new query
3. Remove all columns except the ones belonging to that dimension
4. **Remove Duplicates**, then check for overloaded IDs

Used **Remove Duplicates**, not **Group By**. Group By is built for aggregation; a dimension is not aggregating anything, it just needs one row per key with its attributes preserved.

### location_key

Product and Customer had natural keys. Location did not, and Power BI relationships only join on a single column. So I built a composite key in Power Query:

```
Text.Combine({[Country], [Region], [State], [City], Text.From([Postal Code])}, "|")
```

The `"|"` separator prevents collisions between values that would otherwise concatenate identically. **The formula must be identical in both the fact table and the dimension** or the join fails silently. Produced 632 distinct locations.

### dim_date

Created as a DAX calculated table with `CALENDARAUTO()`, then added Year, Quarter, MonthNum, and MonthName as calculated columns. Set MonthName to **Sort by Column = MonthNum** so months order Jan to Dec instead of alphabetically.

**Why generate a calendar instead of using the dates in the data:** time intelligence needs a continuous, gap-free date range, including days with zero orders. Distinct dates pulled from the fact table would have gaps, and gaps quietly break YoY math.

**Mark as Date Table** is required. Skip it and time intelligence misbehaves silently.

**Role-playing dimension:** Order Date and Ship Date both relate to `dim_date[Date]`. Power BI only permits one active relationship between two tables, so Order Date is active and Ship Date is inactive. Ship Date can be activated inside a specific measure with `USERELATIONSHIP` if needed.

---

## Measures

```dax
Total Sales   = SUM('Sample - Superstore'[Sales])
Total Profit  = SUM('Sample - Superstore'[Profit])
Total Orders  = DISTINCTCOUNT('Sample - Superstore'[Order ID])
Profit Margin = DIVIDE([Total Profit], [Total Sales])
Sales PY      = CALCULATE([Total Sales], SAMEPERIODLASTYEAR(dim_date[Date]))
YoY %         = DIVIDE([Total Sales] - [Sales PY], [Sales PY])
YoY Color     = IF([YoY %] < 0, "#F87171", "#22C55E")
```

**Measure vs calculated column:** a calculated column computes once per row at refresh and is stored. A measure computes on the fly inside whatever filter context the user has clicked. If it aggregates, it is a measure.

### DISTINCTCOUNT vs COUNT (the grain trap)

My first instinct was `COUNT([Order ID])`. Wrong, because of the grain.

The fact table is at line-item grain, so one order with seven products is seven rows. `COUNT` counts rows, so it would have reported that as seven orders. Order CA-2014-115812 is exactly this case. `COUNT` would have given roughly 9,994 orders; the true number is **5,009**.

`DISTINCTCOUNT` collapses those rows back to the distinct orders they represent. You cannot write a correct count without knowing your grain.

(Secondary point: DAX `COUNT` is really meant for numeric columns anyway. Order ID is text.)

### DIVIDE vs /

`DIVIDE()` returns blank on divide-by-zero. Plain `/` throws an error and breaks the visual. Always use `DIVIDE` for ratios.

### CALCULATE and SAMEPERIODLASTYEAR

`CALCULATE(<expression>, <context modifier>)` evaluates a measure inside a **modified filter context**. It is the most important function in DAX; nearly everything advanced runs through it.

`SAMEPERIODLASTYEAR(dim_date[Date])` takes whatever dates are currently in context and returns the same span one year earlier. It only works because of the marked date table and the active Order Date relationship.

**Validation:** put Year, Total Sales, and Sales PY in a table. `Sales PY` for 2015 = $484,247.50, which is exactly 2014's Total Sales. Chain holds for every year. 2014 returns blank because there is no 2013. That check is what proved the date table, the marking, and the active relationship were all correct.

### The 46.88% bug

With the year slicer on "All," the YoY card showed **46.88%**. Plausible-looking and completely meaningless.

`Total Sales` was summing four years (2014-2017, $2,297,200). `Sales PY` shifts back one year and there is no 2013, so it was only summing three (2014-2016, $1,563,985). The card was dividing four years of sales by three years of sales.

**YoY requires a single-year context to mean anything.** Fixed by defaulting the year slicer to a single year so the report cannot be put into a nonsense state. With 2017 selected it reads 20.36%, which is correct.

Takeaway: a number that is wrong but *plausible* is more dangerous than one that is obviously broken.

---

## Data quality decisions

**32 overloaded Product IDs.** Some Product IDs map to two genuinely different products (`FUR-FU-10004848` is both a Howard Miller wall clock and DAX solid wood frames). This is a source data flaw, not something cleanable.

Decision: nothing in the dashboard uses Product Name. Both drill-downs and all KPIs work at Category and Sub-Category, and **zero of the 32 collisions differ in Category or Sub-Category**. So I deduped on Product ID and documented the Product Name ambiguity rather than building a composite key the report never uses. Scope the fix to the question being asked.

To inspect duplicates: Group By Product ID with two aggregations, Count Rows and All Rows, filter Count > 1, expand the nested table. Throwaway query, deleted after.

**Postal code leading zeros.** Power Query typed Postal Code as a whole number on import, stripping leading zeros (01730 became 1730). Changing the type to Text afterward does not restore them; the damage happens at the auto "Changed Type" step. Postal code is not used in any measure or drill-down, and the stripped value is applied identically on both sides of the location key, so the join is unaffected. Left as is, deliberately.

(Fixes if it had mattered: edit the "Changed Type" step to type it as text from the start, or reconstruct with `Text.PadStart(Text.From([Postal Code]), 5, "0")`.)

**Calendar range exceeds sales range.** `CALENDARAUTO()` extends into 2018 because ship dates run to 2018-01-05. Correct behavior. Visuals filtered to years with actual sales.

**Alaska and Hawaii have zero sales.** Only two of the 50 states plus DC are absent from the data entirely. Blank areas are hidden on the map for a cleaner read, but the absence is itself a finding. (Wyoming and Maine *do* have sales, they just render near-white at the bottom of the gradient. Worth not confusing "lowest" with "none.")

---

## Visuals

**Drill-down vs cross-filtering** are two different mechanisms:

- **Drill-down** navigates a hierarchy *inside one visual*. Category and Sub-Category go in the same axis well; the drill controls (single arrow = drill mode, double arrow = expand all, fork = go to next level) sit at the top of the visual.
- **Cross-filtering** is clicking a data point in one visual and having every other visual respond. This is default behavior and is a property of the *visual*, not the chart type.

**Trend chart gotcha:** the monthly line was sorting by sales value, not by month, so it looked like a clean declining trend when it was actually a ranking. Fixed with More options > Sort axis > MonthName, ascending. Requires MonthName to be set to Sort by Column = MonthNum.

**Axis display units:** sales are in tens of thousands, so the default "Millions" units squashed the entire line into one tick. Set display units to Thousands.

**Maps:** Bing map and filled map visuals are disabled by default at the tenant level. Enable in app.powerbi.com > Settings > Admin portal > Tenant settings > Integration > "Map and filled map visuals." Requires being signed into Desktop, and takes a few minutes to propagate.

---

## Theme

Custom theme JSON at `theme/superstore-dark-green.json`. The first version of this dashboard was green on green, everything one hue, and it cost me three things:

- **Hierarchy.** When background, cards, bars, and map are all green, nothing stands out.
- **The color-encoding budget.** The map gradient is doing real work (brighter = more sales). When the whole page is green, that stops reading as an encoding and starts reading as decoration.
- **Semantics and accessibility.** Green means "good" in a business dashboard. Spending it on everything left nowhere to put a negative YoY. A green-only palette is also the worst case for the most common form of color blindness.

Fixed by dropping the background to near-black with a green tint, promoting green to the accent and data color, reserving red and amber for negatives, and letting the map's gradient be the only color ramp on the page.

---

## Concepts to revisit

- `USERELATIONSHIP` for measures that need the inactive Ship Date relationship
- Binned or quantile color scales, to handle the California/New York skew on the map
- Trimming the fact table to keys and measures only
- Time intelligence beyond SAMEPERIODLASTYEAR (rolling 12 months, YTD, `DATEADD`)
