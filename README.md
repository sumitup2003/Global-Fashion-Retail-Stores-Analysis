# 🛍️ Global Fashion Retail Sales — Power BI Report

An end-to-end data analytics project covering data cleaning, data modelling, DAX development, and interactive dashboard design using a real-world-scale retail dataset.

---

## 📌 Project Overview

This project analyses **6.4 million transactions** from a global fashion retail chain operating across **7 countries and 35 stores** (2023–2025). The goal was to build a professional, insight-driven Power BI report from raw CSV files — covering every step from import and cleaning through to a polished multi-page dashboard.

| Metric | Value |
|--------|-------|
| Total Transactions | 6,416,827 rows |
| Total Revenue | ~€732.8 Million |
| Date Range | Jan 2023 – Mar 2025 |
| Countries | 7 (US, UK, France, Germany, Spain, Portugal, China) |
| Stores | 35 |
| Customers | 1,643,306 |
| Products | 17,940 across 3 categories |

---

## 📁 Dataset

**Source:** [Global Fashion Retail Stores Dataset — Kaggle](https://www.kaggle.com/datasets/ricgomes/global-fashion-retail-stores-dataset)

The dataset consists of 6 related CSV files:

| File | Rows | Description |
|------|------|-------------|
| `transactions.csv` | 6,416,827 | Core fact table — every sale and return line item |
| `customers.csv` | 1,643,306 | Customer demographics |
| `products.csv` | 17,940 | Product catalogue with multilingual descriptions |
| `stores.csv` | 35 | Store locations with coordinates |
| `employees.csv` | 404 | Employee-store assignments |
| `discounts.csv` | 181 | Promotional campaigns by date range and category |

---

## 🧹 Data Cleaning

All cleaning was performed in **Power Query Editor** inside Power BI. Each decision was made with explicit reasoning — no rows or columns were removed without a documented justification.

### Key issues found and resolved

| File | Issue | Decision & Reason |
|------|-------|-------------------|
| `transactions.csv` | 798 exact duplicate rows (identical across all 19 columns) | **Removed** — true accidental double-entries confirmed by matching timestamps |
| `transactions.csv` | 339,627 negative Line Total values | **Kept** — correctly represent Return transactions, not errors |
| `transactions.csv` | 413K rows missing Size / 4.35M missing Color | **Kept as blank** — 100% of these are Accessories (no size/colour attribute applies) |
| `customers.csv` | Wrong file encoding (Windows-1252 instead of UTF-8) | **Fixed** — changed `Encoding=1252` to `Encoding=65001` in M source step |
| `customers.csv` | 584K rows with missing Job Title | **Filled "Not Provided"** — optional field at sign-up, no pattern suggesting data error |
| `customers.csv` | Telephone stored as mixed numeric/text types | **Set to Text type** — phone numbers should never be treated as numbers |
| `products.csv` | 12,445 rows missing Color (69%) | **Filled "Not Specified"** — inconsistent data capture across same sub-categories |
| `products.csv` | 2,070 rows missing Sizes | **Kept as blank** — 100% are Accessories; blank is the correct value |
| `discounts.csv` | 10 rows missing Category/Sub Category | **Filled "All Categories"** — these are storewide promotions (Black Friday, Holiday Season) |
| `discounts.csv` | Column named "Discont" (typo) | **Renamed to "Discount"** |
| All files | Non-English country names (Deutschland, España, 中国) | **Left as-is** — consistent within the dataset; technically correct, not broken |

### A key lesson from this project

Missing data ≠ broken data. Every blank value was investigated for a *reason* before any action was taken:
- **Accessories have no sizes** → correct blank, not missing
- **Storewide discounts have no category** → filled with "All Categories", not deleted
- **Returns have negative Line Total** → real business data, not an error

---

## 🗃️ Data Model

The model follows a **Star Schema** — the industry standard for Power BI / analytical models:

```
                    [Date]
                      |  1
                      |
[customers] 1───── [transactions] ─────1 [products]
                      |  *
                 *    |    *
            [stores]  |  [employees]
                      |
                  [discounts]
                  (no direct key —
                   connected via DAX)
```

**Relationships:**

| From (Many) | To (One) | Column |
|-------------|----------|--------|
| transactions | customers | Customer ID |
| transactions | products | Product ID |
| transactions | stores | Store ID |
| transactions | employees | Employee ID |
| transactions | Date | Date Only (pure date, stripped of time component) |
| employees | stores | Store ID |

> **Note:** `Transactions[Date]` was a Date/Time column, causing relationship failures against the pure-date `Date` table. A calculated column `Date Only = DATEVALUE(Transactions[Date])` was created and used as the relationship key instead.

> **Note:** `Discounts` has no shared key with transactions — it connects via a date-range + category match handled in DAX.

---

## 📐 DAX Measures & Columns

All measures are stored in a dedicated **"All Measures"** table (no measures buried inside fact/dimension tables).

### Calculated Columns (fixed per-row values)

```dax
-- Strip time from Date/Time for relationship key
Date Only = DATEVALUE(Transactions[Date])

-- Classify transactions
Is Return = IF(Transactions[Transaction Type] = "Return", "Yes", "No")

-- Profit per line item (uses RELATED to pull Production Cost across relationship)
Profit = Transactions[Line Total] - (RELATED(Products[Production Cost]) * Transactions[Quantity])

-- Customer age grouping
Age Group = 
VAR Age = DATEDIFF(customers[Date Of Birth], TODAY(), YEAR)
RETURN SWITCH(TRUE(),
    Age < 25, "18-24",  Age < 35, "25-34",
    Age < 45, "35-44",  Age < 55, "45-54",
    Age >= 55, "55+",   "Unknown"
)
```

### Core Measures

```dax
Total Sales       = SUM(Transactions[Line Total])
Total Quantity    = SUM(Transactions[Quantity])
Total Profit      = SUM(Transactions[Profit])
Total Customers   = DISTINCTCOUNT(Transactions[Customer ID])
Total Returns     = CALCULATE(COUNTROWS(Transactions), Transactions[Transaction Type] = "Return")
```

### Ratio & Rate Measures

```dax
-- Always use DIVIDE (never /) to handle zero-division gracefully
Profit Margin %     = DIVIDE([Total Profit], [Total Sales], 0)
Return Rate %       = DIVIDE([Total Returns], COUNTROWS(Transactions), 0)
Average Order Value = DIVIDE([Total Sales], DISTINCTCOUNT(Transactions[Invoice ID]), 0)
% of Total Sales    = DIVIDE([Total Sales], CALCULATE([Total Sales], ALL(Transactions)), 0)
```

### Time Intelligence Measures

```dax
-- All time-intelligence functions reference the Date TABLE, not Transactions[Date]
Sales LY    = CALCULATE([Total Sales], SAMEPERIODLASTYEAR('Date'[Date]))
Sales PM    = CALCULATE([Total Sales], DATEADD('Date'[Date], -1, MONTH))
Sales MTD   = TOTALMTD([Total Sales], 'Date'[Date])
Sales QTD   = TOTALQTD([Total Sales], 'Date'[Date])
Sales YTD   = TOTALYTD([Total Sales], 'Date'[Date])
YoY Growth% = DIVIDE([Total Sales] - [Sales LY], [Sales LY], 0)
MoM Growth% = DIVIDE([Total Sales] - [Sales PM], [Sales PM], 0)
```

### Customer Behaviour

```dax
Repeat Customers = 
CALCULATE(
    DISTINCTCOUNT(Transactions[Customer ID]),
    FILTER(
        VALUES(Transactions[Customer ID]),
        CALCULATE(DISTINCTCOUNT(Transactions[Invoice ID])) > 1
    )
)
```

---

## 📊 Report Pages

### Page 1 — Dashboard (Overview)
The "30-second glance" for executives and stakeholders.

**Visuals:**
- 4 KPI cards — Total Sales, Total Customers, Total Quantity, Total Returns
- Line chart — Monthly Sales & Profit trend (Jan 2023 → Mar 2025)
- Column chart — Year-over-Year Sales comparison (2023 / 2024 / 2025)
- Pie chart — Sales split by Category (3 categories)
- Bar chart — Top Products by Total Sales & Total Profit
- Map — Store-level sales by precise geographic coordinates (Latitude/Longitude)
- Year slicer

### Page 2 — Report (Deep Dive)
Granular analysis for business analysts and regional managers.

**Visuals:**
- 4 KPI cards — Total Profit, Profit Margin %, YoY Growth %, Average Order Value
- Pivot table — Monthly breakdown (Year Month × Total Sales, Sales PM, MoM Growth %)
- Detailed table — Country-level breakdown (Total Sales, Sales LY, Total Profit, Profit Margin, Return Rate, Repeat Customers, % of Total Sales)
- Country slicer + Year slicer

---

## 💡 Key Insights

1. **Strong YoY growth** — Revenue grew from ~€305M (2023) to ~€383M (2024), a 25%+ increase.
2. **Return rate is significant** — 339,627 return transactions out of 6.4M total (~5.3%) — worth monitoring by category and store.
3. **2025 is partial data** — Only Jan–Mar 2025 (~€45M) is in the dataset; full-year projection would exceed 2024.
4. **Geographic concentration** — 7 countries but performance varies significantly by store; the map reveals which city-level locations drive the most volume.
5. **Repeat customer rate** — Tracking via the Repeat Customers measure reveals loyalty patterns across demographics.

---

## 🛠️ Tools & Skills Used

| Area | Tool / Skill |
|------|-------------|
| Data Cleaning | Power Query (M language) |
| Data Modelling | Star Schema, relationships, cardinality |
| Analytics | DAX (measures, calculated columns, time intelligence) |
| Visualisation | Power BI Desktop |
| Version Control | Git / GitHub |

---

## 🚀 How to Run

1. Download the dataset from [Kaggle](https://www.kaggle.com/datasets/ricgomes/global-fashion-retail-stores-dataset)
2. Clone this repository
3. Open `retai_sales_report.pbix` in Power BI Desktop
4. Go to **Transform Data** → update file paths in each Source step to match your local download folder
5. Click **Close & Apply**

---

## 👤 Author

**Sumit**
- GitHub: [https://github.com/sumitup2003?tab=repositories]
- LinkedIn: [https://www.linkedin.com/in/sumit-upadhyay-0b48632b8/]

---

*This project was built as a complete end-to-end learning exercise — from raw CSV files through to a production-style Power BI report.*
