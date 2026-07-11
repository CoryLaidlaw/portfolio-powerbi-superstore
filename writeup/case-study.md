# Case Study: Superstore KPI Dashboard

## The problem

The Sample Superstore dataset is one flat table: 9,994 rows, 21 columns, every row a single line item on an order. It has everything you need to run a sales business and none of the structure you need to actually read it. There are no measures, no relationships, and no way to ask "how did we do against last year" without doing it by hand.

The goal here was to build what a sales leader would actually want on a screen: a handful of KPIs, a trend, and the ability to click into any region or category to see what is driving the number. The harder and more interesting half of that is the modeling underneath it, because a dashboard is only as trustworthy as the model it sits on.

## The data model

The first decision was to not leave the data flat. It would technically work, but a flat table makes every measure fragile and every filter ambiguous, and it does not scale past a toy report.

I reshaped it into a **star schema**: one fact table in the middle, five dimension tables around it.

**Grain first.** Before splitting anything I named what one row of the fact table represents: one line item on one order. Everything else follows from that, and getting it wrong would have made every count wrong downstream. This mattered later, in a specific way I will come back to.

**Fact table** (`Sample - Superstore`) holds the numbers and the keys: Sales, Quantity, Discount, Profit, plus foreign keys out to each dimension. Order ID stays in the fact table as a degenerate dimension, since it is a pure identifier with no descriptive attributes of its own and does not deserve a table.

**Dimensions:**

| Dimension | Key | Attributes | Rows |
|---|---|---|---|
| `dim_date` | Date | Year, Quarter, MonthNum, MonthName | 1,826 |
| `dim_product` | Product ID | Category, Sub-Category, Product Name | 1,862 |
| `dim_customer` | Customer ID | Customer Name, Segment | 793 |
| `dim_location` | location_key | Country, Region, State, City, Postal Code | 632 |
| `dim_shipmode` | Ship Mode | Ship Mode | 4 |

The dimensions were built in Power Query by referencing the source query, keeping only the columns that belong to that dimension, and removing duplicates. Filtering flows one direction, from each dimension into the fact, which is what keeps the drill-downs and slicers behaving predictably.

Full diagram: [data-model.md](data-model.md).

### Building a key where none existed

Product and Customer each came with a natural key. Location did not. There is no "Location ID" in the source, just Country, Region, State, City, and Postal Code, and Power BI relationships can only join on a single column.

So I built one, concatenating the geography columns into a composite key in Power Query:

```
Text.Combine({[Country], [Region], [State], [City], Text.From([Postal Code])}, "|")
```

The pipe separator prevents accidental collisions between values that would otherwise concatenate into the same string. The same formula has to exist identically in both the fact table and the dimension, since that is the column the relationship joins on. If the two formulas drift even slightly, the join fails silently, which is the worst kind of failure. That produced 632 distinct locations.

### The date dimension, and why it is generated

The date table is the one dimension you do not build from your data. I generated it with `CALENDARAUTO()`, which scans every date column in the model and produces a continuous, gap free calendar covering all of them.

The reason matters: time intelligence needs *every* day present, including days with no orders. If I had pulled distinct dates out of the fact table instead, days with zero sales would simply be missing, and those gaps quietly break year over year math. The table then gets formally marked as a date table, which is what lets DAX time intelligence functions trust it.

`CALENDARAUTO()` produced 1,826 rows spanning 2014 through 2018, even though orders stop in 2017. That is correct behavior, not a bug: ship dates run into early January 2018, so the calendar has to cover them. It did create a reporting artifact that I had to handle, which I cover below.

Order Date and Ship Date both point at this one date table. Power BI only allows one active relationship between two tables, so Order Date is the active one and Ship Date is inactive. That is a role playing dimension: one date table serving two roles, with the inactive relationship available to any measure that specifically needs ship timing.

## The measures

Everything is an explicit DAX measure. Measures calculate on the fly inside whatever filter context the user has clicked, which is what makes a dashboard interactive rather than static.

```dax
Total Sales   = SUM('Sample - Superstore'[Sales])
Total Profit  = SUM('Sample - Superstore'[Profit])
Total Orders  = DISTINCTCOUNT('Sample - Superstore'[Order ID])
Profit Margin = DIVIDE([Total Profit], [Total Sales])
Sales PY      = CALCULATE([Total Sales], SAMEPERIODLASTYEAR(dim_date[Date]))
YoY %         = DIVIDE([Total Sales] - [Sales PY], [Sales PY])
```

Three of these are worth explaining, because they are where the thinking is.

### Total Orders uses DISTINCTCOUNT, and that is the whole point

My first instinct was `COUNT([Order ID])`. That is wrong, and it is wrong because of the grain.

The fact table is at line item grain, so one order that contains seven products occupies seven rows. `COUNT` would count that as seven orders. Order CA-2014-115812 in this dataset is exactly that case, seven line items, one order. Counting rows would have inflated "Total Orders" to roughly the row count, 9,994, instead of the true 5,009.

`DISTINCTCOUNT` collapses those seven rows back to the one order they represent. The lesson is that you cannot write a correct count without knowing your grain, and this is the measure that proves the model is understood rather than copied.

### Profit Margin uses DIVIDE, not the division operator

`DIVIDE()` returns blank on a divide by zero instead of throwing an error. If a filter context ever lands on zero sales, plain `/` breaks the visual. `DIVIDE` degrades gracefully. It is a small thing that keeps a dashboard from falling over in front of the person you built it for.

### Year over Year

`Sales PY` wraps `[Total Sales]` in `CALCULATE`, which evaluates it inside a modified filter context, and uses `SAMEPERIODLASTYEAR` to shift that context back one year. `YoY %` is then just the change over last year's base.

I validated it rather than assuming it worked. Putting Year, Total Sales, and Sales PY side by side in a table, `Sales PY` for 2015 came back as exactly $484,247.50, which is precisely 2014's Total Sales. The chain holds for every year, and 2014 correctly returns blank because there is no 2013 to compare against. That check is what confirmed the date table, the "Mark as Date Table" step, and the active Order Date relationship were all correct.

### The bug I had to find: a YoY number that meant nothing

With the year slicer set to "All," the YoY card confidently displayed **46.88%**. It looked plausible. It was meaningless.

Here is what was happening. `Total Sales` was summing four years, 2014 through 2017, totaling $2,297,200. `Sales PY` shifts back a year, and since there is no 2013, it was only summing three, 2014 through 2016, totaling $1,563,985. The card was dividing four years of revenue by three years of revenue and reporting the result as growth.

Year over year only means something inside a single year of context. The fix was to default the year slicer to a single year and constrain it so a user cannot put the report into a state that produces a nonsense number. With 2017 selected, the card reads 20.36%, which is real.

I am keeping this in the writeup on purpose. A number that is wrong but plausible is more dangerous than one that is obviously broken, and the habit worth demonstrating is checking whether a KPI can be interpreted at all before trusting it.

## The report

Four KPI cards across the top (Sales, Orders, Margin, YoY), a monthly trend, sales by Category, and a US state choropleth, with Year and Region slicers driving the page.

Two different interactions are doing work here, and they are worth distinguishing:

**Drill-down** navigates a hierarchy inside a single visual. Category and Sub-Category sit in the same axis well on the bar chart, so clicking Furniture expands it into Bookcases, Chairs, Furnishings, and Tables.

**Cross-filtering** is what happens when you click a data point in one visual and every other visual on the page responds. Click California on the map and the cards, the trend, and the category chart all re-filter to California. This is default Power BI behavior, but knowing it is a property of the visual rather than the chart type is what lets you design a page around it.

The YoY card is conditionally formatted, green when growth is positive and red when it is negative, driven by a measure returning a hex value. Flip the slicer to 2015 and it turns red, which is the point of holding a color in reserve rather than using every color for decoration.

## Data quality decisions

Real data is dirty and the interesting part is deciding what actually matters for the question you are answering.

**32 overloaded Product IDs.** Some Product IDs map to two genuinely different products. `FUR-FU-10004848` is both a Howard Miller wall clock and a set of DAX solid wood frames. This is a flaw in the source data, not something to clean up.

The decision came down to one question: does anything in this dashboard use Product Name? It does not. The KPIs and both drill-downs operate at Category and Sub-Category, and I confirmed those attributes are identical across every collision, zero of the 32 differ in Category or Sub-Category. So I deduplicated on Product ID and documented the Product Name ambiguity rather than building a composite key the report never uses. Scoping the fix to the actual question, instead of over-engineering, is the call I would defend.

**Postal code leading zeros.** Power Query typed Postal Code as a whole number on import, so Northeast ZIPs like 01730 lost their leading zero. Postal code is not used in any measure or drill-down, and the value is applied identically on both sides of the location key, so the join is unaffected. Left as is, deliberately, and documented.

**Calendar range exceeds sales range.** As noted, `CALENDARAUTO()` extends into 2018 to cover ship dates. Visuals are filtered to years with actual sales.

**Alaska and Hawaii have zero sales.** Of the 50 states plus DC, only those two are absent from the data entirely. Blank areas are hidden on the map for a cleaner read, but the absence is itself a finding worth naming out loud.

## Design

The theme is a custom JSON ([theme/superstore-dark-green.json](../theme/superstore-dark-green.json)), and the choices behind it were deliberate.

The background is near black with a green tint rather than saturated green, so the cards and charts sit *on* the surface instead of blending into it. Green is spent on the data marks and the KPI values, where it earns attention, rather than on the wallpaper. Red and amber are held in reserve so a negative YoY has somewhere to go, which is the entire reason not to make everything one color. The categorical palette leads with green but includes teal, lime, and blue, so a multi-series visual stays readable, including for the most common form of color blindness. The map's gradient is the only place a color ramp runs, which is what lets it read as a real encoding rather than decoration.

## What I would do differently

**Trim the fact table.** The fact still carries the descriptive columns that now live in the dimensions. It works, but a clean fact table holds only keys and measures. It is model hygiene rather than a functional problem, and worth doing on the next build.

**Handle the skew on the map.** Sales are heavily concentrated in California and New York, so a linear color gradient squashes the other 47 states into a narrow band that is hard to differentiate. A binned or quantile scale would show regional variation far better than a straight linear ramp.

**Add margin to the map.** Right now the map answers "where do we sell the most." The more useful executive question is "where do we sell profitably," and a margin view would surface the states where revenue looks healthy but profit does not.

**Use a proper date range slicer.** A year slicer forces single-year context, which is what keeps YoY honest, but it also means you cannot look at a rolling twelve months. A date range slicer with a YoY measure built to respect it would be more flexible.

## What I took away

The dashboard was the easy half. The parts that took real thinking were choosing the grain, realizing `COUNT` and `DISTINCTCOUNT` answer two different questions, building a key that did not exist, and catching a KPI that was displaying a confident, plausible, completely meaningless number.

The last one is the one I would want to be asked about. Anyone can put a card on a canvas. Knowing whether the number on it can be interpreted at all is the actual job.
