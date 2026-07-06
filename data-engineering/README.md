# Data Engineering Repository – Architecture & Productionization

> **Purpose:** Central repository for all Databricks pipelines feeding the **WW IPD Analytics ecosystem**, following a clean **Bronze → Silver → Gold** structure.  
> Gold produces **IPD-only, time-filtered analytic tables**, consumed directly by the **Power BI semantic model (star schema)**.

---

# 1. Data Entering the Lakehouse

## Core Fact Tables (Consumed in Power BI)

- **Sales Orders (Shipped + Open)**  
  Unified ERP orders dataset.  
  Pipeline: `ww-sales-orders/g_bowler_ww_orders_optimized.ipynb`

- **Inventory**  
  Global stock positions and product availability.  
  Pipeline: `ww-inventory/g_bowler_ww_inventory.ipynb`

- **Billing Master**  
  Actual billing lines representing recognized revenue.  
  Pipeline: `ww-billing-master/g_bowler_ww_billing_master.ipynb`

- **Finance**  
  Budgets, performance, and financial plan data.  
  Pipeline: `ww-finance/g_bowler_ww_finance.ipynb`

- **SFDC Pipeline**  
  Opportunities, stages, probabilities, competitive reasons.  
  Pipeline: `ww-sfdc/g_bowler_ww_sfdc.ipynb`

- **Forecast (Two Fact Tables)**  
  - **Demand**  
  - **Forecast**  
  Used for regional + WW forecasting and scenario modeling.

- **Complaints**  
  Complaints extracted from TrackWise.  
  Pipeline: `ww-complaints/g_bowler_ww_complaints.ipynb`

- **R&D Projects**  
  R&D project portfolio from Project Center.  
  Pipeline: Sourced from a semantic model built by Dan O’Donnell (OData feed).  
  Note: Currently not Databricks-optimized; migration planned to push full R&D data into Databricks.

---

## Helper Tables / Dimensions

- **Exchange Rates (FXB / FXA)**  
  Financial exchange rate tables for global currency normalization.  
  Sources:  
  - `s_shared.v_budget_exchange_rate_bpc`  
  - `s_shared.v_actual_exchange_rate_bpc`

- **Product Master (Reltio Hierarchy)**  
  Authoritative global product hierarchy used across all functions.  
  Source: `g_bpc_finance.v_reltioproduct_hier`

- **Geography Mapping**  
  Mapping of natural country keys to BD Finance global geography.  
  Sources:  
  - `g_bpc_finance.v_revenue_plan_country`  
  - `g_bpc_finance.v_country_global_hier`  
  - Mapping file: **WW Geo.xlsx**

- **Time Intelligence**  
  Time dimensions and slicers aligned to BD Fiscal Calendar.  
  Pipelines:  
  - `helper/g_periodslicer_ipd_analytics.ipynb`  
  - `helper/s_ipd.s_dimdate_analytics_ipd.ipynb`

- **Customer Master**  
  Not yet unified across Orders, Billing, and SFDC.  
  Future objective: cross-system customer normalization + surrogate keys.

---

# 2. Bronze → Silver → Gold Structure

## Bronze – Raw Landing

- Direct ingestion from source systems  
- Minimal transformations  
- Schema standardization + metadata enrichment  
- Stored under `<catalog>.bronze_*`  
- Represents traceable raw system-of-record data

## Silver – Cleaned & Standardized

- Deduplication and correction of data issues  
- Harmonized data types  
- Normalized country, product, and status codes  
- Stored under `<catalog>.silver_*`  
- Clean but still close to source structure

## Gold – IPD Filtered & Time-Filtered Fact Tables  
**Gold does *not* join facts to dimensions.**  
All relational modeling is done inside **Power BI**, not Databricks.

Gold layer responsibilities:

- Filtering to **IPD-specific product ranges**  
- Filtering to **Current Fiscal Year + Prior Fiscal Year**  
- Enforcement of IPD business rules  
- Producing stable, analytic-ready fact tables

Gold fact tables:

- `FactSalesOrders_IPD`  
- `FactBilling_IPD`  
- `FactInventory_IPD`  
- `FactFinance_IPD`  
- `FactSFDC_IPD`  
- `FactDemand_IPD`  
- `FactForecast_IPD`  
- `FactComplaints_IPD` *(when needed)*  
- `FactRND_IPD` *(when migrated)*

---

# 3. Power BI Semantic Model

## Dataset & Infrastructure

- Workspace: **WW_IPD_Center_of_Excellence**  
- Backed by **F32 Premium Capacity**  
- Connection: **DirectQuery** to Databricks SQL Warehouse  
- Source tables: **Gold schema only**

## Modeling Strategy

Facts expose **nkey** columns for:

- Product  
- Customer  
- Date  
- Geography

Power BI transforms them into star-schema join keys:

1. Duplicate each `nkey` column  
2. Rename duplicate to:  
   - `skeyProduct`  
   - `skeyCustomer`  
   - `skeyDate`  
   - `skeyGeography`  
3. Link these to the corresponding **dimension skeys**

This creates a clean, scalable **star schema**.

## Dimensions Used

- **DimProduct** (from Reltio global hierarchy)  
- **DimGeography**  
- **DimDate**  
- **Period Slicer**  
- **DimCustomer** (future integration)

## Measures

All business metrics—Revenue, Units, Backorders, SFDC Win Rates, OTIF, etc.—are built as **DAX measures in Power BI**.

---

# 4. Productionization Workflow

## Environment Separation

- **DEV Environment**  
  - Active notebook development  
  - Experimental pipelines  
  - Uses your personal dev catalog and permissions

- **PRD Environment**  
  - Fully automated pipelines  
  - Code deployed via **pull requests** into `main`  
  - Only the `main` branch executes production workflows

## Databricks Workflows (Orchestration)

Each domain operates a full **Bronze → Silver → Gold** pipeline:

- Orders  
- Billing  
- Inventory  
- Finance  
- SFDC  
- Forecast (Demand + Forecast)  
- Complaints  
- R&D *(future)*  
- Helper Dimensions (FX, Geo, DimDate)

**Execution order:**

1. FX / Geo / DimDate (dimensions)  
2. Domain Silver transformations  
3. Gold (IPD-filtered finalized fact tables)

---

# 5. Power BI Refresh Strategy

The WW IPD semantic model refreshes **twice per day**:

| Region | Time | Purpose |
|--------|------|----------|
| **EMEA Morning** | ~08:00 CET | Guarantees fresh data for Europe |
| **US Morning** | ~08:00 EST | Guarantees fresh data for Americas |

Because the semantic model uses **DirectQuery**, refreshes primarily:

- Reload metadata  
- Refresh table structures  
- Update relationship mappings  
- Sync partition definitions  

Actual data is always read live from Gold Delta tables in Databricks.

---

# 6. End-to-End Data Flow Overview

```mermaid
flowchart TB

    %% Source Systems
    A[Source Systems<br/>(ERP, SFDC, Finance, TrackWise, Project Center)] --> B[Bronze Layer<br/>(Raw Landing)];

    %% Bronze -> Silver
    B --> C[Silver Layer<br/>(Cleaned & Standardized)];

    %% Silver -> Gold
    C --> D[Gold Layer<br/>(IPD Filter + Time Windowed Facts<br/>No Joins, No Modeling)];

    %% Gold -> Databricks SQL
    D --> E[Databricks SQL Warehouse<br/>(Gold Catalog)];

    %% Databricks SQL -> Power BI
    E --> F[Power BI Semantic Model<br/>(DirectQuery)];

    %% PBI Modeling Details
    F --> G[nkey → skey Mapping<br/>Star Schema Joins<br/>Shared Dimensions];

    %% Final Dashboards
    G --> H[Dashboards<br/>WW IPD (F32 Capacity)<br/>Refreshed Twice Daily];
