# ⭐ Power BI Semantic Model – Star Schema Architecture

The WW IPD semantic model follows a **pure star-schema design**, optimized for DirectQuery to Databricks and aligned with the Gold-layer filtered fact tables.  
All business logic (relationships, hierarchy alignment, and measures) is handled in Power BI—not in Databricks.

The star schema enables:
- High performance on DirectQuery  
- Consistent slicing across Product, Geography, Date, TimeMode, Currency, Scenario  
- Clean separation between transactional facts and conformed dimensions  
- Simplified DAX and lower modeling complexity  
- Full alignment with Finance, Supply, and SFDC hierarchies  

---

# 1. ⭐ Star Schema Overview

## Core Facts (from Gold layer)
The semantic model imports the following Gold-level IPD fact tables:

- `g_bowler_orders`
- `g_bowler_billing_master_ww_ipd`
- `g_bowler_inventory`
- `g_bowler_backorder`
- `g_bowler_finance_actuals_ipd_cy`
- `g_bowler_finance_budget_ipd_cy`
- `g_bowler_finance_projections_ipd_cy`
- `g_bowler_sfdc`
- `g_bowler_complaints`
- `g_bowler_demand`
- `g_bowler_fcst`

These tables are **already filtered to IPD** and **trimmed to Current FY + Prior FY**.

Each fact exposes four natural-key columns:
- `nkeyProduct`
- `nkeyGeography`
- `nkeyDate`
- `nkeyCustomer` *(where applicable)*

These are duplicated inside Power BI and renamed to:
- `skeyProduct`
- `skeyGeography`
- `skeyDate`
- `skeyCustomer`

These skeys form the relationships to shared dimensions.

# 2. 🧩 Shared Dimensions (Conformed)

## Product Dimension
- **Source:** `fin_v_reltioproduct_hier`
- **Purpose:** Global authoritative hierarchy for all product-level reporting  
- **Relationship:**  
Fact.nkeyProduct → DimProduct.skeyProduct

## Geography Dimension
- **Source:** `g_geo_mapping`
- **Purpose:** Align all facts to Finance global geographies (country → region → cluster → WW)  
- **Relationship:**

Fact.nkeyGeography → DimGeography.skeyGeography


## Date Dimension
- **Source:** `g_dimdate_analytics_ipd`
- **Purpose:** Unified fiscal + calendar backbone  
- **Relationship:**  
Fact.nkeyDate → DimDate.skeyDate

markdown
Copy code

## Period Slicer
- **Source:** `g_periodslicer_analytics_ipd`
- **Purpose:** Unified YTD/QTD/Rolling period selection  
- **Relationship:** Many-to-one to DimDate

## Currency & FX Helpers
- `v_budget_exchange_rate_bpc`  
- `v_actual_exchange_rate_rate_bpc`  
Used by measures only (not physical star-joins).

## Parameter & Control Tables
Used as slicers:
- `slc_Metric`
- `slc_Scale`
- `slc_Scenario`
- `slc_Currency`
- `slc_TimeMode`
- `slc_Parameter`

These tables are disconnected dimensions that control measure logic via DAX.
