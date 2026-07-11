# Superstore KPI Dashboard (Power BI)

An interactive executive dashboard built in Power BI Desktop on the Sample Superstore retail dataset. It answers the questions a sales leader actually asks: how are we tracking against last year, where is the money coming from, and what is driving the number.

![Dashboard](screenshots/dashboard.png)

## The problem

Raw transaction data does not answer business questions. The Superstore file is one flat table of 9,994 order line items with no structure and no measures. A leader looking at it cannot tell whether the business is growing, which regions carry the revenue, or where margin is leaking.

This project turns that flat file into a dimensional model with defined KPIs, then puts a clickable dashboard on top of it so those questions can be answered in a few seconds instead of a few hours.

## What it shows

- **KPI cards:** Total Sales, Total Orders, Profit Margin, and Year over Year growth
- **Monthly sales trend** for the selected year
- **Sales by Category**, with drill-down into Sub-Category
- **Sales by State**, as a US choropleth map
- **Year and Region slicers** that filter the entire page

Clicking any state or category cross-filters every other visual, so you can go from "revenue is up 20%" to "because Technology in the West is up" without writing a query.

## What I found

- Sales grew from $484K in 2014 to $733K in 2017, a 51% increase over the period
- 2015 was the one down year, sales fell 2.83%, but profit margin actually *improved* from 10.23% to 13.10%. Selling less at better margin is a very different story than a simple revenue dip, and it is the kind of thing a single revenue KPI would hide
- 2017 was the strongest year for revenue but margin slipped back to 12.74%, so growth came with some margin compression
- California and New York alone account for roughly a third of all revenue
- Sales are seasonal, with a clear ramp into September and a November peak

## Stack

Power BI Desktop, Power Query (M), DAX. Custom dark theme via theme JSON.

Data: [Sample Superstore](data/) (9,994 rows, 21 columns, US retail orders, 2014 to 2017).

## How to run it

1. Clone the repo.
2. Open `pbix/superstore.pbix` in Power BI Desktop (Windows only).
3. The data source is `data/Sample - Superstore.csv`. If the path does not resolve, go to **Transform data > Data source settings** and repoint it to your local copy.
4. The theme is already applied. To reapply it, use **View > Themes > Browse for themes** and select `theme/superstore-dark-green.json`.

## What's in this repo

| Path | What it is |
|---|---|
| `pbix/superstore.pbix` | The Power BI file |
| `data/` | The source CSV |
| `theme/superstore-dark-green.json` | Custom dark theme |
| `writeup/case-study.md` | Full case study: model, measures, decisions |
| `writeup/data-model.md` | Star schema diagram |
| `NOTES.md` | Build notes, decisions, and what I learned |
| `screenshots/` | Dashboard images |
| `recording/` | Short walkthrough of the dashboard |

## The data model

The flat CSV was reshaped into a star schema: one fact table at order line item grain, surrounded by five dimensions (Date, Product, Customer, Location, Ship Mode). Full diagram and reasoning in [writeup/data-model.md](writeup/data-model.md).

Every measure is explicit DAX. The details, including why Total Orders uses `DISTINCTCOUNT` and how Year over Year is calculated, are in the [case study](writeup/case-study.md).
