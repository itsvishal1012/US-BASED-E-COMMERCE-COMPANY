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
├─ US_E-COMMERCE_Detailed_Documentation.md  # This detailed file
└─ scripts/
   ├─ data_cleaning.py            # Optional Python scripts for cleaning
   └─ eda_notebook.ipynb          # Jupyter notebook with EDA and plots
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

---

## 6 — Data quality checks & cleaning (step-by-step)

Below is a canonical Python (pandas) workflow you can use to clean the CSV prior to loading into Power BI (or use directly within Power BI via Power Query). Save these steps as `scripts/data_cleaning.py` or run in a notebook.

### 6.1 — Outline of cleaning actions
- Inspect schema and head of data
- Convert date columns to `datetime`
- Remove duplicate rows
- Handle missing values (impute, infer, or drop depending on business logic)
- Standardize state names and map to 2-letter codes
- Create derived fields (Year, Month, OrderMonth, OrderWeek, ShippingDays)
- Detect and treat outliers on `Sales`, `Quantity`, `Profit`
- Save cleaned version as `ecommerce_data_cleaned.csv`

### 6.2 — Example pandas code (copy & run)

```python
# scripts/data_cleaning.py
import pandas as pd
import numpy as np

# 1. Load
df = pd.read_csv('ecommerce_data.csv', low_memory=False)

# 2. Quick inspect
print(df.shape)
print(df.dtypes)
print(df.head())

# 3. Convert date columns
for col in ['OrderDate', 'ShipDate']:
    if col in df.columns:
        df[col] = pd.to_datetime(df[col], errors='coerce')

# 4. Remove exact duplicates
df = df.drop_duplicates()

# 5. Handle missing values (example strategy)
# - If OrderDate missing -> drop (can't analyze)
# - For numeric fields, fill zeros or median if appropriate
if 'OrderDate' in df.columns:
    df = df.dropna(subset=['OrderDate'])

numeric_cols = ['Sales', 'Quantity', 'Profit', 'Discount']
for c in numeric_cols:
    if c in df.columns:
        df[c] = pd.to_numeric(df[c], errors='coerce')
        # Impute numeric NA with 0 for quantity, median for amounts
        if c == 'Quantity':
            df[c] = df[c].fillna(0).astype(int)
        else:
            df[c] = df[c].fillna(df[c].median())

# 6. Standardize state names and add 2-letter codes using us_state_long_lat_codes.csv
states = pd.read_csv('us_state_long_lat_codes.csv')
# expect columns like State, StateCode, Latitude, Longitude
if 'State' in df.columns:
    df = df.merge(states[['State','StateCode']], how='left', left_on='State', right_on='State')

# 7. Derived columns
if 'OrderDate' in df.columns:
    df['Year'] = df['OrderDate'].dt.year
    df['Month'] = df['OrderDate'].dt.month
    df['MonthName'] = df['OrderDate'].dt.strftime('%b')
    df['OrderMonth'] = df['OrderDate'].dt.to_period('M').astype(str)
    df['DayOfWeek'] = df['OrderDate'].dt.day_name()

# Shipping duration (if ShipDate exists)
if 'ShipDate' in df.columns and 'OrderDate' in df.columns:
    df['ShipDays'] = (df['ShipDate'] - df['OrderDate']).dt.days

# 8. Outlier detection (IQR method) — example on Sales
if 'Sales' in df.columns:
    q1 = df['Sales'].quantile(0.25)
    q3 = df['Sales'].quantile(0.75)
    iqr = q3 - q1
    lower = q1 - 1.5 * iqr
    upper = q3 + 1.5 * iqr
    # Flag outliers
    df['Sales_outlier'] = ((df['Sales'] < lower) | (df['Sales'] > upper))

# 9. Save cleaned file
df.to_csv('ecommerce_data_cleaned.csv', index=False)
print('Saved cleaned file: ecommerce_data_cleaned.csv')
```

### 6.3 — Power Query (Power BI) equivalent
If you prefer Power Query (M language), use the UI to:
- Change Type for each column
- Use `Remove Duplicates`
- `Fill` or `Replace Values` for missing states
- Add `Custom Column` for derived fields like `Year = Date.Year([OrderDate])`

---

## 7 — Data modeling & star schema recommendation

To keep the Power BI model performant, create a star schema:

- **FactSales** (fact) — each row is a transaction / line item: OrderID, OrderDate (key), ProductID, CustomerID, Sales, Quantity, Discount, Profit
- **DimDate** (dimension) — Date, Year, Quarter, Month, MonthName, IsHoliday, FiscalYear
- **DimProduct** — ProductID, ProductName, Category, SubCategory, Cost, Supplier
- **DimCustomer** — CustomerID, CustomerName, Segment, State, City, PostalCode
- **DimRegion** — Region mapping (StateCode -> Region)

Only keep relationships: FactSales[OrderDate] → DimDate[Date], FactSales[ProductID] → DimProduct[ProductID], FactSales[CustomerID] → DimCustomer[CustomerID]. Make relationships single-direction (recommended) unless you need bi-directional filtering for special cases.

---

## 8 — Key measures & DAX (copy-ready)

Below are DAX measures to compute main KPIs. Replace table/column names with your actual names if they differ.

```dax
-- Basic base measures
Total Sales = SUM('FactSales'[Sales])
Total Profit = SUM('FactSales'[Profit])
Total Quantity = SUM('FactSales'[Quantity])

-- Profit margin
Profit Margin = DIVIDE([Total Profit], [Total Sales], 0)

-- YTD measures (requires DimDate is marked as date table)
YTD Sales = TOTALYTD([Total Sales], 'DimDate'[Date])
YTD Profit = TOTALYTD([Total Profit], 'DimDate'[Date])

-- Year-over-year growth (Sales)
Sales LY = CALCULATE([Total Sales], SAMEPERIODLASTYEAR('DimDate'[Date]))
Sales YoY % = DIVIDE([Total Sales] - [Sales LY], [Sales LY], 0)

-- Running total
Cumulative Sales =
CALCULATE(
    [Total Sales],
    FILTER(ALL('DimDate'), 'DimDate'[Date] <= MAX('DimDate'[Date]))
)

-- Average order value
Average Order Value = DIVIDE([Total Sales], DISTINCTCOUNT('FactSales'[OrderID]), 0)

-- Top N products (dynamic top N)
Top N Parameter = 10 -- create a What‑If parameter or slicer instead of hardcoding
Top N Sales =
IF(
    HASONEVALUE('DimProduct'[ProductName]),
    SUM('FactSales'[Sales]),
    CALCULATE([Total Sales], TOPN([Top N Parameter], VALUES('DimProduct'[ProductName]), [Total Sales], DESC))
)

-- Top product by sales (example)
Top Product =
TOPN(1, SUMMARIZE('DimProduct', 'DimProduct'[ProductName], "ProdSales", [Total Sales]), [ProdSales], DESC)
```

**Notes:**
- Always create a Calendar/DimDate table and mark it as the model’s date table (`Modeling → Mark as Date Table`).
- Use measures rather than calculated columns where aggregation across filters is required.

---

## 9 — Power BI page-by-page design & visual choices

### Overview / Executive page
- Big number cards: Total Sales, Total Profit, Profit Margin, Qty Sold, YoY %.
- Line chart: Monthly Sales trend (use `Month` on axis and `Total Sales` as measure).
- Bar/Treemap: Sales by Category.
- Map/Choropleth: Sales by State (use StateCode with Filled Map or Mapbox).

### Product analysis page
- Bar chart: Top 10 products by Sales with drill-through to product details.
- Table: Product-level metrics (Sales, Profit, Qty, Margin).
- Slicer: Category, SubCategory, Region, Year

### Regional / Map page
- Filled Map or Shape Map showing sales intensity by state (use the `us_state_long_lat_codes.csv` to match codes or lat/long).
- Small multiples for region-level trends.

### Customer & RFM page
- RFM grid heatmap (R=Recency, F=Frequency, M=Monetary) with segments colored by average profit.
- Top customers list and LTV summary.

### Operational page
- Orders by day of week / hour (if order timestamp available)
- Shipping lead time distribution (ShipDays)

---

## 10 — Mapping and geospatial tips

- If `StateCode` is available, use Power BI’s built‑in Shape Map or Filled Map. Shape Map requires mapping file (TopoJSON) optionally. For simple choropleth, the built‑in `Map` or `Filled Map` with `StateCode` will work.
- If you have lat/long in `us_state_long_lat_codes.csv`, join it to the fact or dimension table in Power BI and use the `Map` or `ArcGIS Maps for Power BI` for better control.
- Make sure state names / codes in the dataset match the mapping file exactly (use `Trim()` and `Upper()` in Power Query if needed).

---

## 11 — Advanced analyses (ideas + brief how-to)

### Time series forecasting (monthly sales)
- Export aggregated monthly sales to Python and fit Prophet or SARIMA. Evaluate using rolling origin CV.
- In Power BI, use the built-in Forecast feature on a line chart for quick linear forecasting.

### RFM and Customer segmentation
- Compute Recency (days since last order), Frequency (# orders in past year), Monetary (sum of sales) and cluster using KMeans.
- Visualize segments and compute average LTV, churn probability, and retention actions.

### Market-basket (product affinity)
- Use association rules (Apriori) on transactions to find product sets frequently bought together.

### Anomaly detection
- Use z‑score or isolation forest on daily sales to flag anomalies (promotion spikes, data issues).

### Profit optimization & pricing experiments
- Build uplift models to estimate causal effect of discounts or promotions on sales and profit.

---

## 12 — Deployment & scheduling

- Publish the `.pbix` / `.pbit` to Power BI Service.
- Configure a data gateway if `ecommerce_data.csv` is located on-premise or behind a firewall.
- Use scheduled refresh (daily or hourly depending on SLA) and monitor refresh history.
- Set up workspace roles and App distribution for stakeholders.

---

## 13 — Data governance & security

- Use Row-Level Security (RLS) to restrict users to their region or business unit using role definitions in Power BI.
- Keep the production data source credentials secure (use service principals where possible).
- Document column definitions and transformation logic for auditability.

---

## 14 — Reproducibility & environment

- Power BI Desktop version used to create this report: *record the version here*.
- Python environment (if used for EDA): list of packages (pandas, numpy, scikit-learn, prophet, matplotlib, seaborn) and versions.
- Save any ETL / cleaning scripts under `/scripts` and use Git for version control.

---

## 15 — Limitations & assumptions

- Assumes `Sales` and `Profit` values are in a single currency and comparable (no multi-currency conversion logic included).
- Assumes `ecommerce_data.csv` contains line-item level records; if it is order-level, adjust derived metrics (e.g., Quantity aggregation).
- Missing or incorrect `State` or `StateCode` values will affect mapping and regional aggregations.

---

## 16 — Next steps & recommended experiments

1. **Automate ingestion** — move CSV into a database or cloud storage and connect Power BI via direct query or scheduled refresh.
2. **Add a calendar table** with fiscal year flags and holiday indicators.
3. **Build customer lifetime value (CLTV)** models and embed outputs in the dashboard.
4. **Set up alerts** in Power BI for KPI thresholds (sharp drops in sales, margin erosion).
5. **A/B testing framework** — use promotion windows and track lift.

---

## 17 — Appendix: Useful SQL & pandas snippets

### SQL — total sales by state
```sql
SELECT state, SUM(sales) AS total_sales, SUM(profit) AS total_profit
FROM ecommerce
GROUP BY state
ORDER BY total_sales DESC;
```

### pandas — top 10 products
```python
top_products = df.groupby('ProductName').agg({'Sales':'sum','Profit':'sum','Quantity':'sum'})
top_products = top_products.sort_values('Sales', ascending=False).head(10)
print(top_products)
```

### DAX — calendar table (example)
```dax
Calendar =
ADDCOLUMNS(
    CALENDAR(MIN('FactSales'[OrderDate]), MAX('FactSales'[OrderDate])),
    "Year", YEAR([Date]),
    "Month", FORMAT([Date], "MMMM"),
    "MonthNumber", MONTH([Date]),
    "Quarter", "Q" & FORMAT([Date], "Q")
)
```

---

## 18 — Contact / author

If you want me to:
- Extract the exact column names and include a concrete data dictionary from `ecommerce_data.csv`,
- Run the data cleaning and attach `ecommerce_data_cleaned.csv`,
- Generate the Jupyter notebook with EDA and plots,
- Export this document to a downloadable Markdown/PDF file,

tell me which of the above you'd like me to do next and I will prepare it.

---

*End of document.*

