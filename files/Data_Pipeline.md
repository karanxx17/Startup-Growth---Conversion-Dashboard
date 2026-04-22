# Data Pipeline Documentation
## Startup Growth & Conversion Dashboard

---

## Pipeline Overview

```
[Kaggle CSV] → [Power Query] → [Cleaning] → [Power BI Model] → [DAX Measures] → [Dashboard]
    Raw             Ingest       Transform        Load             Calculate         Visualize
```

---

## Stage 1: Raw Data

- **Source:** Kaggle — Ecommerce Behavior Data from Multi-Category Store
- **File:** 2020-Feb.csv
- **Raw size:** 200,000 rows × 9 columns
- **Storage:** /raw-data/2020-Feb.csv (never modified)

### Raw Schema

| Column | Type | Description |
|---|---|---|
| event_time | Text (raw) | Timestamp of user event |
| event_type | Text | view / cart / purchase |
| product_id | Integer | Product identifier |
| category_id | Integer | Category identifier |
| category_code | Text | Dot-separated category path |
| brand | Text | Product brand name |
| price | Float | Product price in USD |
| user_id | Integer | Unique user identifier |
| user_session | Text | Session UUID |

---

## Stage 2: Ingestion

**Tool:** Power BI Desktop → Get Data → Text/CSV

```
Home ribbon → Get Data → Text/CSV → Browse to 2020-Feb.csv → Open
→ Preview window opens
→ Click "Transform Data" (NOT Load)
→ Power Query Editor opens
```

**Why not direct Excel open:** The raw file has 7M+ rows across all months. Direct open causes Excel to freeze. Power Query only loads a preview sample during editing.

---

## Stage 3: Transformation (Power Query)

All transformations applied in Power Query Editor. Each step recorded in Applied Steps panel.

### Step 1: Fix Null Brand Values
```
Column selected: brand
Action: Transform → Replace Values
Value to find: (empty)
Replace with: unknown
Result: 35,182 nulls filled
Applied Step name: "Replaced Value"
```

### Step 2: Fix Null Category Code Values
```
Column selected: category_code
Action: Transform → Replace Values
Value to find: (empty)
Replace with: unknown
Result: 20,594 nulls filled
Applied Step name: "Replaced Value1"
```

### Step 3: Remove Zero-Price Rows
```
Column selected: price
Action: Dropdown arrow → Number Filters → Greater Than → 0
Result: 1,294 rows removed
Applied Step name: "Filtered Rows"
```

### Step 4: Fix event_time Data Type
```
Column selected: event_time
Action: Click ABC icon → Change to Date/Time
Result: Enables time intelligence in Power BI
Applied Step name: "Changed Type1"
```

### Applied Steps Final State
```
Source
Promoted Headers
Changed Type
Replaced Value       ← brand nulls
Replaced Value1      ← category_code nulls
Filtered Rows        ← price > 0
Changed Type1        ← event_time to DateTime
```

### Close & Apply
```
Home → Close & Apply (direct click)
Power BI loads 173,198 clean rows into data model
```

---

## Stage 4: Data Model

**Table name in Power BI:** feb2020_clean_powerbi

**Final clean schema:**

| Column | Type | Notes |
|---|---|---|
| event_time | Date/Time | Fixed from Text |
| event_type | Text | view/cart/purchase |
| product_id | Whole Number | |
| category_id | Whole Number | |
| category_code | Text | nulls → "unknown" |
| brand | Text | nulls → "unknown" |
| price | Decimal | Zero rows removed |
| user_id | Whole Number | |
| user_session | Text | |

**Row count:** 173,198 (cleaned from 200,000 raw)
**Rows removed:** 26,802 (1,294 zero-price + 25,508 duplicates)

---

## Stage 5: DAX Measures

All measures created under Modeling → New Measure.

### Group 1: Core KPI Counts

```dax
Total Users =
DISTINCTCOUNT('feb2020_clean_powerbi'[user_id])
-- Result: 37,287

Total Views =
CALCULATE(
    COUNT('feb2020_clean_powerbi'[event_type]),
    'feb2020_clean_powerbi'[event_type] = "view"
)
-- Result: 159,762

Total Carts =
CALCULATE(
    COUNT('feb2020_clean_powerbi'[event_type]),
    'feb2020_clean_powerbi'[event_type] = "cart"
)
-- Result: 11,018

Total Purchases =
CALCULATE(
    COUNT('feb2020_clean_powerbi'[event_type]),
    'feb2020_clean_powerbi'[event_type] = "purchase"
)
-- Result: 2,418

Total Revenue =
CALCULATE(
    SUM('feb2020_clean_powerbi'[price]),
    'feb2020_clean_powerbi'[event_type] = "purchase"
)
-- Result: $697,551

Avg Order Value =
DIVIDE([Total Revenue], [Total Purchases])
-- Result: $288.48
```

### Group 2: Conversion Rates

```dax
View to Cart % =
DIVIDE(
    CALCULATE(DISTINCTCOUNT('feb2020_clean_powerbi'[user_id]),
              'feb2020_clean_powerbi'[event_type] = "cart"),
    CALCULATE(DISTINCTCOUNT('feb2020_clean_powerbi'[user_id]),
              'feb2020_clean_powerbi'[event_type] = "view")
) * 100
-- Result: 14.85%

Cart to Purchase % =
DIVIDE(
    CALCULATE(DISTINCTCOUNT('feb2020_clean_powerbi'[user_id]),
              'feb2020_clean_powerbi'[event_type] = "purchase"),
    CALCULATE(DISTINCTCOUNT('feb2020_clean_powerbi'[user_id]),
              'feb2020_clean_powerbi'[event_type] = "cart")
) * 100
-- Result: 34.55%

Overall Conversion % =
DIVIDE(
    CALCULATE(DISTINCTCOUNT('feb2020_clean_powerbi'[user_id]),
              'feb2020_clean_powerbi'[event_type] = "purchase"),
    CALCULATE(DISTINCTCOUNT('feb2020_clean_powerbi'[user_id]),
              'feb2020_clean_powerbi'[event_type] = "view")
) * 100
-- Result: 5.13% ← NORTH STAR
```

### Group 3: Retention

```dax
Repeat Rate % =
VAR PurchaseTable =
    CALCULATETABLE(
        VALUES('feb2020_clean_powerbi'[user_id]),
        'feb2020_clean_powerbi'[event_type] = "purchase"
    )
VAR TotalBuyers = COUNTROWS(PurchaseTable)
VAR UserPurchaseCounts =
    ADDCOLUMNS(
        PurchaseTable,
        "PCount",
        CALCULATE(
            COUNT('feb2020_clean_powerbi'[event_type]),
            'feb2020_clean_powerbi'[event_type] = "purchase"
        )
    )
VAR RepeatBuyers = COUNTROWS(FILTER(UserPurchaseCounts, [PCount] > 1))
RETURN DIVIDE(RepeatBuyers, TotalBuyers) * 100
-- Result: 17.85%
-- Logic: 339 repeat buyers / 1,894 total buyers
```

---

## Stage 6: Visualization Layer

**8 visuals on Page 1 — "Startup Dashboard"**

| Visual | Type | Fields Used |
|---|---|---|
| Title | Text Box | — |
| North Star | Card | Overall Conversion % |
| KPI Row (×5) | Cards | Total Users, Purchases, Revenue, V→C%, C→P% |
| Conversion Funnel | Clustered Bar Chart | event_type (Y), user_id Count Distinct (X) |
| What's Wrong? | Text Boxes | Static PM insights |
| Top 10 Brands | Clustered Bar Chart | brand (Y), Total Purchases (X), Top N filter |
| Revenue by Category | Clustered Bar Chart | category_code (Y), Total Revenue (X), Top N filter |

---

## Data Quality Summary

| Check | Result |
|---|---|
| Null brand values | Fixed (35,182 → "unknown") |
| Null category_code values | Fixed (20,594 → "unknown") |
| Zero-price rows | Removed (1,294 rows) |
| Duplicate rows | Removed (25,508 rows) |
| event_time data type | Fixed (Text → Date/Time) |
| Final row count | 173,198 |
| Data completeness | 100% (no remaining nulls in key fields) |
