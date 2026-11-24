# Online Retail II: Data Cleaning, Cohort Analysis and BI Dashboards

End-to-end portfolio project: download a public retail dataset, clean it in Python (pandas), document every decision with evidence from the data, and visualize it in Looker Studio and Tableau Public.

**Dashboards**
- Looker Studio: _link to be added_
- Tableau Public: _link to be added_

## Goals

- Build a clean, documented data source that two BI tools can use without tool-specific formulas.
- Answer business questions on revenue, cancellations and customer retention.
- Show that unusual rows (cancellations, stock adjustments, fees) are classified and kept visible, not silently deleted.

## Data source

UCI Machine Learning Repository, **Online Retail II**: transactions of a UK-based online retailer, 2009-12-01 to 2011-12-09. The workbook has two sheets (525,461 and 541,910 rows) with 8 columns: Invoice, StockCode, Description, Quantity, InvoiceDate, Price, Customer ID, Country. Check the license and citation requirements on the UCI dataset page before reuse.

## Cleaning decisions (with evidence)

Every rule below was tested on the data before it was applied.

| # | Finding | Evidence | Decision |
|---|---|---|---|
| 1 | The two sheets overlap in early December 2010 | 1,088 invoices appear in both sheets, with 22,523 rows each; the rows are identical when sorted and compared | Removed the shared invoices from sheet 2. Result: 1,044,848 rows. Whole invoices were removed, not `drop_duplicates()`, so legitimate repeated lines are kept |
| 2 | Missing Customer ID | 22.5% of rows; none of the 53,628 invoices mixes rows with and without an ID | Kept for revenue and forecasting, excluded from cohort analysis. No IDs were imputed |
| 3 | Negative quantities | 22,557 rows: 19,164 are on invoices starting with "C" (cancellations); 3,393 are not | "C" invoices labelled as cancellations. The 3,393 others all have price 0 and no customer, with descriptions like damaged, missing, check, found, so they are stock adjustments, not sales |
| 4 | Cancellations cannot be linked to the original sale | 8,292 distinct cancellation invoices; 0 have a matching invoice number once the "C" is removed | No linking. Cancellations are counted in the month they were recorded |
| 5 | Exact duplicate rows | 11,812 rows identical in all 8 columns: 1.1% of rows, 0.29% of revenue (54,228 of 18.9M). Another 11,612 rows repeat Invoice + StockCode but differ in quantity or price (separate lines) | Flagged with `IsExactDuplicate`, not deleted, because a true double entry cannot be told apart from a second unit scanned separately |
| 6 | No natural unique key | Invoice + StockCode repeats 23,424 times; adding Customer ID changes nothing (one customer per invoice verified) | Added a surrogate `RowID` after sorting by date and invoice |
| 7 | Zero-price sales | 2,631 positive-quantity rows with price 0; 97.3% have no customer; descriptions mix staff notes (check, found, adjustment) with product names | Labelled `zero_price`. They add 0 revenue but would inflate unit counts |
| 8 | Non-product StockCodes | Descriptions checked code by code (postage, fees, manual entries, bad debt, vouchers, test rows) | Added `ItemType` (see below) |
| 9 | "A" invoices | 6 invoices with code B, "Adjust bad debt", quantity 1, money stored in Price (total -147,614). They were first labelled as sales | All accounting rows were moved to `RowType = adjustment`, so sales and cancellations contain only customer activity |
| 10 | Mixed data types | StockCode mixes numbers and text; Invoice mixes numbers and text; Customer ID is float because of missing values | StockCode and Invoice as text; CustomerID as nullable integer (`Int64`) |

## Two labels per row

**RowType** (what kind of event)

| Value | Meaning | Rows |
|---|---|---|
| sale | Customer purchase | 1,018,708 |
| cancellation | Customer return or cancellation ("C" invoice) | 18,217 |
| adjustment | Stock write-off or accounting entry, not a customer transaction | 5,299 |
| zero_price | Positive quantity with price 0 | 2,624 |

**ItemType** (what the row is about)

| Value | Meaning | Rows |
|---|---|---|
| product | Merchandise | 1,039,035 |
| shipping | Postage and carriage (POST, DOT, C2) | 3,788 |
| accounting | Fees, discounts, manual and bad-debt entries (D, M, m, ADJUST, ADJUST2, B, S, BANK CHARGES, AMAZONFEE, CRUK) | 1,906 |
| gift_voucher | Gift vouchers (gift_0001_*) | 100 |
| test | Test products (TEST001, TEST002) | 17 |
| unknown | No description, negative quantity, price 0 (C3, GIFT) | 2 |

## Metric definitions

- **Revenue** = Quantity x Price (per row).
- **Gross sales revenue**: `RowType = sale` and `ItemType` in product, shipping, gift_voucher.
- **Net sales revenue**: gross sales plus `cancellation` rows (cancellations are negative), same item types.
- **Units sold**: `RowType = sale`, `ItemType = product`.
- **Adjustments**: shown separately (stock corrections have price 0, so only quantities are known).
- Rows flagged `IsExactDuplicate` are kept; filter them out to test the effect (about 0.3% of revenue).

## Cohort analysis

- **Cohort** = the month of a customer's first purchase, using the same rows as units sold (`sale`, `product`, with a Customer ID).
- 5,852 customers have a cohort; 25 cohorts (Dec 2009 to Dec 2011).
- `MonthIndex` = months between the cohort month and the purchase month (0 to 24), computed as year x 12 + month difference, so it is exact and tool-independent.
- **Retention** = distinct active customers in the month / cohort size (Month 0).

## Output files

`online_retail_ii_clean.csv` (1,044,848 rows, 16 columns)

| Column | Description |
|---|---|
| RowID | Unique row number (sorted by date, then invoice) |
| Invoice | Invoice number as text; "C" prefix = cancellation, "A" prefix = accounting entry |
| InvoiceDateTime | Original timestamp |
| InvoiceDate | Date only (YYYY-MM-DD) |
| InvoiceMonth | First day of the invoice month |
| CustomerID | Customer number; empty for anonymous purchases |
| Country | Customer country |
| StockCode | Product or service code as text |
| Description | Original description (may be empty) |
| ItemType | product, shipping, accounting, gift_voucher, test, unknown |
| RowType | sale, cancellation, adjustment, zero_price |
| IsExactDuplicate | True for the extra copies of fully identical rows |
| Quantity | Units (negative for cancellations and write-offs) |
| Price | Unit price |
| Revenue | Quantity x Price |
| CohortMonth | Customer's first-purchase month (empty if none) |

`online_retail_ii_cohort.csv` (325 rows): CohortMonth, MonthIndex, Customers, CohortSize, Retention.

All dates use year-month-day format, which both BI tools read unambiguously.

## Limitations

- **Left-censoring:** the data starts in December 2009, so the first cohort (951 customers) includes existing customers, not only new ones. Its retention looks high.
- **Cancellations** are recorded in the month they happened, which may differ from the month of the original sale, and cannot be linked to it.
- **Stock adjustments and write-offs** have price 0, so their money value is unknown.
- About 22.5% of rows have no Customer ID and are not part of the cohort analysis.
- Some classifications rest on the code family only: DCGSSBOY, DCGSSGIRL, DCGSLBOY and DCGSLGIRL have mostly missing descriptions and are treated as products.
- Zero-price product rows are labelled but unexplained by the data.
- 17 test rows remain in the file and are excluded through `ItemType`.

## Repository structure

```
notebooks/        cleaning and cohort notebook
data/             cohort CSV and instructions to regenerate the clean CSV
README.md
```

The raw workbook and the full clean CSV are not stored in the repository. Download the raw file from UCI, run the notebook, and the clean CSV is written to your data folder.

## How to reproduce

1. Download the Online Retail II workbook from the UCI repository.
2. Create a virtual environment and install `pandas`, `openpyxl` and `jupyter`.
3. Run the notebook from top to bottom (reading the xlsx takes a few minutes).
4. Load the resulting CSVs into Looker Studio and Tableau Public.

## Tools

Python (pandas, NumPy), Jupyter, Looker Studio, Tableau Public.
