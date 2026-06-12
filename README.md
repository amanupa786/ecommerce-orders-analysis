# 🛒 E-Commerce Orders — Data Cleaning & Analysis Pipeline
### End-to-End Data Pipeline using Python, Pandas, Matplotlib & OpenPyXL

![Python](https://img.shields.io/badge/Python-3.x-blue?logo=python)
![Pandas](https://img.shields.io/badge/Pandas-Data%20Cleaning-green?logo=pandas)
![NumPy](https://img.shields.io/badge/NumPy-Numerical-orange?logo=numpy)
![Matplotlib](https://img.shields.io/badge/Matplotlib-Charts-red?logo=python)
![OpenPyXL](https://img.shields.io/badge/OpenPyXL-Excel%20Export-brightgreen)
![Status](https://img.shields.io/badge/Status-Complete-success)

---

## 📌 Project Overview

This project builds a **production-style, 7-phase data pipeline** on a simulated messy
e-commerce orders dataset (15 rows × 14 columns). It covers everything from raw data profiling
and multi-problem cleaning through business KPI analysis, chart generation, formatted
Excel reporting, and automated email delivery.

The project intentionally introduces **10 distinct real-world data problems** and solves
each one with a clear, documented fix — making it an ideal reference pipeline for any
data analyst working with business data.

---

## 🎯 Business Problem

Raw e-commerce order data from multiple regions arrives with:
- Mixed date formats (`DD/MM/YYYY` vs `YYYY-MM-DD`)
- Currency symbols blocking numeric parsing (`$1200.00` as text)
- Discount percentages on two different scales (`0.10` vs `10.0` for 10%)
- Inconsistent country codes (`USA`, `US`, `United States` all meaning the same thing)
- Negative quantities from data entry errors
- Invalid email addresses
- A delivery date recorded **before** the order date (logical violation)
- Duplicate order IDs
- Null values across 4 columns
- Inconsistent text casing across all text columns

This pipeline detects and fixes every single one of these issues systematically.

---

## 🗂️ Dataset

Simulated e-commerce orders dataset — `np.random.seed(7)` — 15 rows × 14 columns.

| Column | Type | Known Issues |
|---|---|---|
| order_id | int | 1 duplicate (order 1002) |
| customer_id | str | Mixed casing (`c002` vs `C002`) |
| customer_name | str | Whitespace, ALL CAPS, mixed case, 1 null |
| email | str | 2 invalid emails (no `@` symbol) |
| product_category | str | Inconsistent casing (`ELECTRONICS`, `electronics`, `Electronics`) |
| product_name | str | Leading/trailing whitespace |
| quantity | float | 1 null, 1 negative value (`-3`) |
| unit_price | str | `$` prefix blocks numeric parsing, 1 null |
| discount_pct | float | Mixed scales: some 0–1, some 0–100 |
| order_date | str | Mixed formats: `YYYY-MM-DD` and `DD/MM/YYYY` |
| delivery_date | str | 1 row delivered before order date |
| status | str | Inconsistent casing (`Delivered`, `delivered`, `DELIVERED`) |
| payment_method | str | 2 nulls |
| country | str | 5 variations (`USA`, `US`, `United States`, `UK`, `GB`) |

---

## 🔧 7-Phase Pipeline

### Phase 1 — Data Inspection
Profile every column before touching a single value.
- `df.shape` — row and column count
- `df.dtypes` — data types (spot `object` where numeric expected)
- `df.select_dtypes(include="object")` — find all text columns
- `df.isnull().sum()` — null count and percentage per column
- `df.describe()` — numeric distribution summary
- `df.value_counts(dropna=False)` — categorical distributions
- `df.duplicated(subset=["order_id"])` — duplicate detection

### Phase 2 — Schema Validation
Catch missing or renamed columns before the pipeline runs.
- Custom `validate_schema()` function
- Raises `ValueError` with readable message if any column is missing
- Fails fast — prevents downstream errors from silent missing columns

### Phase 3 — Cleaning (6 sub-tasks)

**Nulls:**
- `dropna(subset=["customer_name"])` — drop unrecoverable rows
- `fillna("Unknown")` — payment_method placeholder
- `fillna(median)` — quantity filled with median

**Type Conversion:**
- `.str.replace("$", "", regex=False)` → `.str.strip()` → `pd.to_numeric(errors="coerce")` — strip currency symbol and parse unit_price
- Discount scale fix: `df.loc[mask, "discount_pct"] / 100` — normalise 0–100 values to 0–1
- `pd.to_datetime(dayfirst=True, errors="coerce")` — parse mixed date formats

**Text Standardisation:**
- `.str.strip().str.title()` — 5 text columns standardised
- `.str.upper()` — customer_id codes normalised
- `dict.map().fillna()` — country code mapping (`USA`/`US` → `United States`, `UK`/`GB` → `United Kingdom`, `FR` → `France`)

**New Problems Detected:**
- `str.contains("@", na=False)` — invalid email flag
- `.abs()` — negative quantity correction
- Date logic check: `delivery_date < order_date` → set to `NaT`

**Deduplication:**
- `drop_duplicates(subset=["order_id"], keep="first")`

### Phase 4 — Feature Engineering
Create business-ready derived columns.
- `gross_revenue = quantity × unit_price`
- `net_revenue = gross_revenue × (1 − discount_pct)`
- `discount_amount = gross_revenue − net_revenue`
- `delivery_days` from timedelta `.dt.days`
- `is_late = delivery_days > 7` (SLA flag)
- `order_year`, `order_month`, `order_month_name` from `.dt` accessor

### Phase 5 — KPI Analysis
Four business summaries using `groupby().agg()`.
- Country summary — net revenue, discount given, order count, avg delivery days, late rate %, revenue share %
- Category summary — net revenue, gross revenue, avg discount, units sold
- Top 5 products by net revenue
- Monthly revenue trend
- `rank(method="dense")` — revenue ranking per order
- `nlargest()` / `nsmallest()` — top and bottom 3 orders
- Best order per country using `groupby().first()`

### Phase 6 — Charts & Excel Export
Three Matplotlib charts embedded into a formatted Excel report.
- Bar chart — Net revenue by country
- Horizontal bar chart — Top 5 products by net revenue
- Line chart with fill — Monthly revenue trend
- `BytesIO` buffer — charts saved in-memory before embedding
- `pd.ExcelWriter` with `openpyxl` — 4-sheet workbook
- `Font`, `PatternFill`, `Alignment`, `Border` — header styling
- `column_dimensions` — auto-fit column widths
- `freeze_panes = "A2"` — frozen headers
- `number_format = "#,##0.00"` — currency formatting
- Alternating row shading
- Timestamped output filename

### Phase 7 — Automated Email Delivery
HTML email report with KPI summary table sent via `yagmail` + `.env` credentials.
- `build_ecommerce_email()` — generates styled HTML email with KPI cards and country table
- `dotenv` — secure credential loading (no hardcoded passwords)
- `yagmail.SMTP` — Gmail SMTP delivery with Excel attachment
- `send_email = False` flag — safe preview mode for development

---

## 📊 Key Functions Used

| Function | Purpose |
|---|---|
| `df.select_dtypes()` | Filter columns by data type |
| `validate_schema()` | Custom schema validation with error raising |
| `pd.to_datetime(dayfirst=True)` | Parse mixed date formats |
| `pd.to_numeric(errors="coerce")` | Safe numeric conversion |
| `.str.replace(regex=False)` | Literal string character removal |
| `df.loc[mask, col] / 100` | Conditional value correction |
| `.map(dict).fillna()` | Dictionary-based value replacement |
| `df.abs()` | Fix negative quantities |
| `df.groupby().agg()` | Multi-metric business summaries |
| `df.rank(method="dense")` | Revenue ranking |
| `df.nlargest() / nsmallest()` | Top and bottom N rows |
| `BytesIO()` | In-memory chart buffer |
| `openpyxl.drawing.image.Image` | Embed charts into Excel |
| `pd.ExcelWriter` | Multi-sheet Excel export |
| `load_dotenv()` | Secure environment variable loading |
| `yagmail.SMTP` | Gmail email with attachment |
| `.dt.days` | Extract integer days from timedelta |
| `assert` | Inline data quality assertion |

---

## 📁 Project Structure

```
ecommerce-orders-analysis/
│
├── E-commerce_orders.ipynb     ← Main Jupyter notebook (7 phases)
├── requirements.txt             ← Python dependencies
├── .env.example                 ← Email credential template (no real passwords)
├── .gitignore                   ← Git ignore (hides .env and output files)
├── README.md                    ← Project documentation
├── data/
│   └── ecommerce_orders.csv     ← Auto-generated messy dataset
└── output/
    └── ecommerce_report_YYYY_MM_DD_HHMM.xlsx  ← Generated report
```

---

## ▶️ How to Run

```bash
# 1. Clone the repository
git clone https://github.com/amanupa786/ecommerce-orders-analysis.git
cd ecommerce-orders-analysis

# 2. Install dependencies
pip install -r requirements.txt

# 3. (Optional) Set up email delivery
cp .env.example .env
# Edit .env with your Gmail credentials

# 4. Open and run the notebook
jupyter notebook E-commerce_orders.ipynb
# Kernel → Restart & Run All
```

---

## 📦 Requirements

```
pandas
numpy
matplotlib
openpyxl
python-dotenv
yagmail
jupyter
```

---

## 📈 Results

| Problem | Before | After |
|---|---|---|
| Null values | 4 columns affected | 0 nulls remaining |
| Duplicate orders | 1 duplicate order_id 1002 | Removed, 14 unique orders |
| Text consistency | 5 columns with casing issues | Fully standardised |
| Country codes | 5 variants for 3 countries | 3 clean country names |
| Discount scale | Mixed 0–1 and 0–100 | All normalised to 0–1 |
| Date formats | 2 formats in same column | Single datetime dtype |
| Currency symbol | `$` blocking numeric parse | Clean float64 column |
| Negative quantity | 1 negative value (`-3`) | Fixed with `.abs()` |
| Invalid emails | 2 undetected bad emails | Flagged with `email_valid` column |
| Impossible dates | 1 delivery before order | Corrected to NaT |
| Output | Raw DataFrame | 4-sheet Excel report + 3 embedded charts |

---

## 💡 Skills Demonstrated

- Multi-format date parsing with `dayfirst`
- Currency symbol stripping and numeric conversion
- Discount scale normalisation with conditional fix
- Dictionary-based country code standardisation
- Custom schema validation function
- Invalid email detection with regex-free `.str.contains()`
- Logical date violation detection and correction
- Feature engineering (revenue, discounts, delivery days, SLA flag)
- Four-layer business KPI analysis with groupby
- In-memory chart generation with `BytesIO`
- Multi-sheet Excel export with embedded charts and professional formatting
- Automated HTML email report with `yagmail` and `.env` credential management

---

## 👤 Author

**Aman** — Data Analyst
Python | SQL | Power BI | DAX | Excel | Data Visualization

[![GitHub](https://img.shields.io/badge/GitHub-amanupa786-black?logo=github)](https://github.com/amanupa786)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-amanupa786-blue?logo=linkedin)](https://www.linkedin.com/in/amanupa786)

---

> 💬 *"Every dirty dataset is a problem waiting to be understood — not just cleaned."*
