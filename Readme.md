# US E-COMMERCE — Detailed Documentation

> **Purpose:** This document is a comprehensive, step-by-step guide describing the data, analysis, preprocessing, dashboard design, measures (DAX), analytical techniques, deployment, and recommended next steps for the US E‑Commerce Power BI project.

---

## 1 — Executive summary

This repository contains a Power BI dashboard and source data for a US‑based e‑commerce business. Key Year‑to‑Date (YTD) numbers presented in the dashboard are:

- **YTD Sales:** **$11.53M** (down 0.83% YoY)
- **YTD Profit:** **$1.34M** (up 4.50% YoY)
- **YTD Quantity Sold:** **107.2K units** (down 7.29% YoY)
- **Profit Margin:** **11.58%** (up 5.37% YoY)

Category-level highlights:
- **Office Supplies:** $6.92M (≈ 60% of sales)
- **Furniture:** $2.52M
- **Technology:** $2.10M

Regional distribution (approx):
- **West:** $3.72M (32.22%)
- **East:** $3.28M (28.42%)
- **Central:** $2.67M (23.19%)
- **South:** $1.87M (16.17%)

Top / bottom product lists and other dashboard elements are included in the PDF and Power BI template.

---

## 2 — Purpose & audience

This document is intended for:
- Data analysts or BI developers who will maintain or extend the Power BI report.
- Business stakeholders who want to understand metrics, definitions and how they are computed.
- Data engineers who will operationalize refreshes, gateways and data quality checks.

It explains the data model, cleaning and transformation steps, analysis logic (pandas / SQL / DAX examples), visualization choices, and recommended advanced analyses.

---

## 3 — Repository structure (recommended)

```
US_E-COMMERCE/
├─ US_E-COMMERCE.pbit             # Power BI template file
├─ US_E-COMMERCE.pdf              # PDF export of the dashboard
├─ ecommerce_data.csv              # Raw e-commerce dataset (source)
├─ us_state_long_lat_codes.csv     # State lat/long or 2-letter codes for mapping
├─ README.md                       # Short project README (already present)

```

---

## 4 — Data sources & ingestion

### Files provided
- `ecommerce_data.csv` — primary transactional dataset (orders / line items).
- `us_state_long_lat_codes.csv` — mapping between state names/codes and latitude/longitude (used for maps).
- `US_E-COMMERCE.pbit` — Power BI template with visuals and data model.
- `US_E-COMMERCE.pdf` — exported PDF of the report pages.

### Ingestion recommendations
- Prefer loading the CSV into Power BI using **Get Data → Text/CSV** and inspect the automatic type detection.
- For production, push the CSV into a database (Postgres / SQL Server / Azure SQL) and connect Power BI to the DB for scheduled refresh.
- Always create a **calendar (date)** table (see DAX section) and mark it as a date table in Power BI.

---

## 5 — Suggested data dictionary (common ecommerce fields)

> NOTE: column names differ between datasets. Confirm with `ecommerce_data.csv` and adapt these names as required.

| Field name | Type | Description |
|---|---:|---|
| OrderID | string/ID | Unique identifier for the order (order-level) |
| OrderDate | date | Date when the order was placed |
| ShipDate | date | Date when the order was shipped |
| CustomerID | string/ID | Unique id for the customer |
| CustomerName | string | Customer full name |
| Segment | string | Business segment (Consumer / Corporate / Home Office etc.) |
| ProductID | string/ID | SKU or product identifier |
| ProductName | string | Display name of the product |
| Category | string | High-level category (Office Supplies, Furniture, Technology) |
| SubCategory | string | More granular product grouping |
| Sales | numeric | Sales amount (currency) |
| Quantity | integer | Number of units ordered |
| Discount | decimal | Discount applied (0–1 or percent) |
| Profit | numeric | Profit amount (Sales − Cost) |
| State | string | US state name |
| StateCode | string | 2-letter state code (preferred for maps) |
| City | string | City name |
| PostalCode | string/int | Postal or ZIP code |
| Region | string | Region (West / East / Central / South) |

If your dataset contains additional columns (e.g., `OrderPriority`, `ShipMode`, `SalesChannel`), add them to the dictionary.


## Power Query (Power BI) equivalent
If you prefer Power Query (M language), use the UI to:
- Change Type for each column
- Use `Remove Duplicates`
- `Fill` or `Replace Values` for missing states
- Add `Custom Column` for derived fields like `Year = Date.Year([OrderDate])`

### Regional / Map page
- Filled Map or Shape Map showing sales intensity by state (use the `us_state_long_lat_codes.csv` to match codes or lat/long).
- Small multiples for region-level trends.


---

*End of document.*

