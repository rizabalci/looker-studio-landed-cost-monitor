# Landed Cost Monitor (Looker Studio template)

A Looker Studio report that turns a "by product, marketplace and day" profit export into per-unit cost of goods, shipping and fees, by marketplace and fulfilment channel, and flags SKUs whose cost data looks wrong.

## What it shows and why

Margin tools report costs as totals, which hides the per-unit picture and hides gaps. A marketplace that shows zero shipping cost per unit is not cheap to ship to; it usually means the shipping cost was never entered. A SKU with no cost of goods makes its margin look far better than it is. This report answers:

1. What does one unit cost us in goods, shipping and marketplace fees, per marketplace and channel (FBA vs seller-fulfilled)?
2. Which SKUs have suspiciously low or high shipping per unit?
3. Which SKUs have no cost of goods at all?

Be clear with readers about what is in and out. If your export's shipping column does not include outbound parcel postage, this is a cost view, not a full landed cost, and the title text should say so.

## Data source

One Google Sheet per region, each holding a profit export with one row per product per marketplace per day. Many profit tools (for example Sellerboard's "by product, marketplace, day" export) produce this shape. Replace the IDs: `YOUR_EU_SHEET_ID`, `YOUR_US_SHEET_ID`.

If the export tool names the worksheet after the date range or a timestamp, the Looker source breaks every time a new export lands. Copy the export into a fixed tab called `export` (by script or by paste), and point Looker at that tab.

`sample_data.csv` in this folder matches this schema.

| Column | Type | Meaning |
|---|---|---|
| day | Date | Order day. |
| marketplace | Text | Marketplace as exported, for example `Amazon.de`, `Amazon.fr`, `Amazon.com`, `Non-Amazon`. |
| channel | Text | `FBA` or `FBM` (seller-fulfilled). |
| sku | Text | Seller SKU. |
| product | Text | Product name. |
| units | Number | Units sold. |
| sales | Currency | Gross sales. |
| cogs | Currency | Cost of goods, exported as a negative number. |
| shipping | Currency | Seller shipping cost, exported as a negative number. |
| fees | Currency | Marketplace and fulfilment fees, exported as a negative number. |
| currency | Text | EUR, GBP, USD. Export as is; do not convert in Looker. |

## Calculated fields

Add these in both the EU and the US data source.

| Field | Formula | Type |
|---|---|---|
| MP | `REGEXP_REPLACE(marketplace, "^Amazon\\.", "")` | Text |
| Units | `SUM(units)` | Number |
| COGS per unit | `-SUM(cogs) / SUM(units)` | Currency |
| Shipping per unit | `-SUM(shipping) / SUM(units)` | Currency |
| Fees per unit | `-SUM(fees) / SUM(units)` | Currency |
| Has COGS | `CASE WHEN SUM(cogs) = 0 THEN "No COGS" ELSE "OK" END` | Text |

The minus signs flip the export's negative costs into positive per-unit costs. MP shortens `Amazon.de` to `de`; check that non-Amazon values (like `Non-Amazon`) come through untouched, and adjust the pattern if your export uses different labels.

## Page layout

Canvas 1200 wide. Each page opens with a title text box: source, period covered (for example "export, month to date") and currency.

### Page 1: EU by marketplace

- Title text with source, period and the note on what the shipping column includes.
- Channel control (drop-down on `channel`, default `FBM`).
- Table: dimension MP, metrics Units, COGS per unit, Shipping per unit, Fees per unit. Rename the metric headers to `COGS`, `Shipping`, `Fees` and set them to EUR. Sort Units descending. Widen columns so values are not cut.
- Conditional formatting: Shipping per unit equals 0 on a seller-fulfilled row, light yellow (likely missing data, not free shipping).
- Second table "Shipping per unit outliers": dimension sku, product, MP; metrics Units, Shipping per unit (EUR). Two filters as separate tables or a toggle: Shipping per unit under 1, and Shipping per unit 5 or more. Sort Shipping per unit ascending for the low list, descending for the high list.

### Page 2: EU SKUs without COGS

- Title text: "Products with zero cost of goods. Margin is overstated until COGS is entered."
- Table: sku, product, MP, Units, sales. Filter Has COGS = `No COGS`. Sort Units descending. Widen the product column.

### Page 3: US by marketplace and channel

- Title text: source, period, "USD as exported".
- Table: dimensions MP and channel, metrics Units, COGS per unit, Shipping per unit, Fees per unit in USD. Widen the table and resize columns.

### Formatting on every table

- Missing data shown as `-`.
- Currency columns with two decimals and a thousands separator.

## Controls

- Date range control on every page, applied to `day` (default: month to date).
- Channel drop-down on pages 1 and 3.
- Marketplace drop-down (optional).

## Build steps

1. Export the "by product, marketplace, day" report and copy it into a tab named `export` in your Sheet. Or paste `sample_data.csv` to start.
2. In Looker Studio, create a blank report, canvas 1200 wide.
3. Add a Google Sheets data source for the EU Sheet (`YOUR_EU_SHEET_ID`, tab `export`) and one for the US Sheet. First row as headers.
4. Set types: day as Date, units as Number, sales, cogs, shipping and fees as Currency in the Sheet's currency.
5. Add the calculated fields to both sources.
6. Build page 1, then page 2, then page 3.
7. Set Missing data to `-`, check headers and currencies, then open in View mode.
8. When a new export arrives, replace the contents of `export` without renaming or deleting the tab.

## Tips

- Never average the per-unit fields across rows; always divide sums, which the formulas above already do.
- A sudden drop of shipping per unit to zero for one marketplace usually means the cost setting for that marketplace is missing in the export tool.
- Keep EU and US in separate sources so currencies never mix in one table.

## Design rules

Never estimate missing costs; a zero or blank stays visible and gets flagged rather than filled in. Compare all-in costs per unit, never surcharge percentages. Show currencies as exported and do not convert silently.
