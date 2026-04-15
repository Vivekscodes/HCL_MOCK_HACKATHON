# 🔷 Project O.R.A. — Oracle Retail Analytics Bridge

> **An end-to-end data engineering pipeline built on Informatica Intelligent Cloud Services (IICS) that automates the extraction of retail data from flat-file sources and performs a Full Refresh (Truncate & Load) into an Oracle Data Warehouse.**

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Pipeline Components](#pipeline-components)
  - [m\_Store\_Load](#1-m_store_load)
  - [m\_Product\_Load](#2-m_product_load)
  - [m\_Sales\_Fact\_Load](#3-m_sales_fact_load)
- [Taskflow Orchestration](#taskflow-orchestration)
- [Configuration & Performance](#configuration--performance)
- [Success Criteria & Validation](#success-criteria--validation)
- [Tech Stack](#tech-stack)

---

## Overview

Project O.R.A. is designed with a **modular pipeline architecture**, consisting of three primary mappings controlled by a single orchestrating taskflow. The pipeline ensures high data accuracy, standardized reporting, and zero-duplication through a full truncate-and-reload strategy on every run.

---

## Architecture

```
src_stores.csv   ──►  m_Store_Load   ──►  DIM_STORES   ─┐
                                                          │
src_products.csv ──►  m_Product_Load ──►  DIM_PRODUCTS  ─┼──► FACT_SALES
                                                          │
src_sales_txn.csv──►  m_Sales_Fact_Load ─────────────────┘
```

**Execution is managed by:** `tf_ORA_Bridge` (linear taskflow with error-gate and completion notification)

---

## Pipeline Components

### 1. `m_Store_Load`

| Property | Detail |
|---|---|
| **Source** | `src_stores.csv` |
| **Transformation** | Expression — derives `LOAD_DATE` via `SYSDATE` |
| **Target** | Oracle `DIM_STORES` |
| **Refresh Strategy** | Truncate & Load (Pre-SQL) |

**Column Mapping**

| Source Column | Expression Logic | Target Column |
|---|---|---|
| `STORE_ID` | `LTRIM(RTRIM(STORE_ID))` | `STORE_SK` (PK / natural key) |
| `CITY` | `INITCAP(LTRIM(RTRIM(CITY)))` | `CITY_NAME` |
| `REGION` | `UPPER(LTRIM(RTRIM(REGION)))` | `REGION_NAME` |
| *(derived)* | `SYSDATE()` | `LOAD_DATE` |

---

### 2. `m_Product_Load`

| Property | Detail |
|---|---|
| **Source** | `src_products.csv` |
| **Transformation** | Filter — excludes records where `BASE_PRICE <= 500` |
| **Target** | Oracle `DIM_PRODUCTS` |
| **Refresh Strategy** | Truncate & Load (Pre-SQL) |

**Filter Condition**

```sql
BASE_PRICE > 500
```

---

### 3. `m_Sales_Fact_Load`

The core mapping engine. Performs dimension lookups, string standardization, date conversion, and revenue aggregation.

| Property | Detail |
|---|---|
| **Source** | `src_sales_txn.csv` |
| **Lookups** | Connected Lookups on `DIM_STORES` and `DIM_PRODUCTS` |
| **Aggregator** | `SUM(UNITS * PRICE)` grouped by `TXN_ID` |
| **Target** | Oracle `FACT_SALES` |
| **Refresh Strategy** | Truncate & Load (Pre-SQL) |

**Lookup Details**

| Lookup Table | Input Port | Output Ports |
|---|---|---|
| `DIM_STORES` | `STORE_ID` | `CITY`, `REGION` |
| `DIM_PRODUCTS` | `PROD_ID` | `PROD_NAME`, `CATEGORY` |

**Expression Logic**

| Column | Logic |
|---|---|
| `CITY` | `UPPER(CITY)` |
| `CATEGORY` | `UPPER(CATEGORY)` |
| `S_DATE` | `TO_DATE(S_DATE, 'YYYY-MM-DD')` |
| `TOTAL_REVENUE` | `SUM(UNITS * PRICE)` *(via Aggregator)* |

---

## Taskflow Orchestration

**Taskflow Name:** `tf_ORA_Bridge`  
**Type:** Linear / Sequential

```
[Start]
   │
   ▼
[Step 1] Load Store Dimensions  ──── FAIL ──► [Log Error] ──► [Abort]
   │
   ▼
[Step 2] Load Product Dimensions ─── FAIL ──► [Log Error] ──► [Abort]
   │
   ▼
[Step 3] Load Sales Fact Table
   │
   ▼
[Notify] Email / Event on Success
   │
   ▼
[End]
```

> ⚠️ If either dimension load fails, the taskflow halts immediately. The Fact load will **not** execute with missing dimension data.

---

## Configuration & Performance

| Parameter | Value |
|---|---|
| **Truncate Strategy** | Pre-SQL on Target transformation |
| **Commit Interval** | `5000` records |
| **Error Handling** | `ERROR('transformation error')` as default |
| **Date Format** | `YYYY-MM-DD` → Oracle `DATE` type |
| **String Standard** | `UPPER()` for `CITY` and `CATEGORY` |

---

## Success Criteria & Validation

After a successful pipeline run, verify the following:

- [ ] `FACT_SALES` contains exactly **100 records**
- [ ] `DIM_PRODUCTS` contains **zero records** with `BASE_PRICE <= 500`
- [ ] All `CITY` and `CATEGORY` values are in **ALL CAPS**
- [ ] Re-running the pipeline produces **no duplicate records** (truncate verified)

---

## Tech Stack

| Layer | Technology |
|---|---|
| **ETL Tool** | Informatica Intelligent Cloud Services (IICS) |
| **Database** | Oracle 19c / 21c |
| **Source Format** | Flat Files (`.csv`) |
| **Scripting** | SQL, Informatica Expression Language |

---

<div align="center">
  <sub>Built with IICS · Powered by Oracle · Project O.R.A.</sub>
</div>
