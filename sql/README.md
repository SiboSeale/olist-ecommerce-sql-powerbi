# Olist E-commerce: SQL Pipeline (SQL Server)

This is the SQL half of a SQL-to-Power BI ETL project. It takes the raw Olist CSVs, cleans them, and builds a star schema that the [Power BI dashboard](../README.md) reads from.

> **The business question:** Olist grew quickly, but very few customers came back. Is delivery the reason?

## What this part does

| ETL stage | What happens | Where |
|---|---|---|
| **Extract** | The 9 Kaggle CSVs are loaded as-is (all text) into a staging area | `stg` schema |
| **Transform** | Data is profiled, cleaned, typed and reshaped into a star schema | `dw` schema |
| **Load** | Power BI connects to the `dw` tables only (Import mode) | see Power BI README |

Why two schemas? `stg` is the untouched copy of the source, so I can always go back and check. `dw` is the clean, analysis-ready version. Power BI never sees `stg`.

## Scripts, in the order to run them

| # | Script | What it does |
|---|---|---|
| 01 | Setup | Creates the database and the `stg` and `dw` schemas |
| 02 | Load staging | Row counts after importing the CSVs, to confirm nothing was lost |
| 03 | Profiling | Looks for duplicates, nulls, odd statuses and the real grain of each table |
| 04 | Data-quality checks | Order status and date checks, orphan keys, excluded rows |
| 05 | Dimensions | Builds `dim_date`, `dim_customer`, `dim_seller`, `dim_product`, `dim_payment_type` |
| 06 | `fact_orders` | One row per order: dates, delivery status, review score, payment type |
| 07 | `fact_order_items` | One row per item in an order: price, freight, seller, product |
| 08 | Validation | Checks the facts against staging (row counts and GMV totals) |
| 09 | Analysis queries | SQL answers used to cross-check every Power BI measure |

> Match the file names in this table to the actual files in this folder before publishing.

## The star schema

Two fact tables, because the KPIs live at two levels. Delivery and reviews are per **order**. Sales value is per **item**. Keeping them apart stops orders with several items being counted several times.

| Table | One row per |
|---|---|
| `fact_orders` | Order |
| `fact_order_items` | Item in an order |
| `dim_date` | Calendar day (with an Unknown row, key -1) |
| `dim_customer` | Customer ID (the real person is `customer_unique_id`) |
| `dim_seller` | Seller |
| `dim_product` | Product, with English category name |
| `dim_payment_type` | Payment method (with an Unknown row, key -1) |

All dimension keys are surrogate keys (`IDENTITY`), so the model doesn't depend on the source IDs.

## Key decisions

- **Analysis window: Jan 2017 to Aug 2018.** The months outside it have too few orders and would distort growth.
- **Customers are counted by `customer_unique_id`.** The `customer_id` changes with every order, so counting it would hide repeat buyers.
- **One review per order.** Some orders have several, so `ROW_NUMBER()` keeps the latest.
- **One payment type per order.** Same technique, keeping the primary payment type.
- **`is_late` is left blank (NULL) for undelivered orders,** so they never count as "on time".
- **Missing product categories become `Unknown`** and are kept, so sales aren't understated.
- **Only delivered orders count towards GMV,** the north-star metric.

## How to run it

1. Open SQL Server Management Studio and connect to your server.
2. Run `01` to create the database and schemas.
3. Import the 9 CSVs into `stg` (right-click the database, Tasks, Import Flat File). Keep every column as text, and make sure the header row isn't loaded as data.
4. Run `02` to `09` in order, checking each script's output before moving on.
5. Open Power BI and connect to the `dw` tables.

The raw Kaggle CSVs are not in this repo. Download them from the [dataset page](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce).

## How I checked the work

Each build step ends with a check against staging, and the totals tie out to the dashboard:

- Row counts in each fact match the grain I designed for.
- Delivered GMV in SQL matches Delivered GMV in Power BI.
- Every Power BI measure was reconciled to an equivalent query in `09` before it went on a page.

## Limitations

- GMV is a stand-in for revenue. Olist's real income is commission, which isn't in the data.
- About 20 months of data, so customers who returned later aren't captured.
- Staging columns are text, so type conversion happens in the `dw` build. A bad value would surface there, not at load.

## Documentation

A step-by-step SOP with every query, the checks and a troubleshooting section is in [`/docs`](../docs).

## Author

Sibongile Seale
