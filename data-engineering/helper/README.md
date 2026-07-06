# Helper Tables Required for Reporting

The helper tables provide the **shared dimensions and reference structures** needed for a consistent, global, and analytics-ready Power BI semantic model.  
They ensure that all fact tables—Orders, Billing, Inventory, Finance, SFDC, Forecast, Complaints, R&D—use **aligned keys, aligned hierarchies, aligned timeframes, and aligned currencies**.

**Helper tables include:**

1. **Exchange Rates (FXB / FXA)**
2. **Geography Mapping**
3. **Date Intelligence (DimDate + Period Slicer)**
4. **Product Hierarchy (Reltio + Material Master Integration)**
5. **Customer Master (Future)** – Needed for unified customer naming and distributor consolidation

---

## 1. Exchange Rates per Month

**Sources:**
- `s_shared.v_budget_exchange_rate_bpc`
- `s_shared.v_actual_exchange_rate_bpc`

These FX tables are essential for converting local/entity currencies into **USD** for reporting alignment with Finance.

---

### **Currency Conversion Logic**

#### **Average vs. Closing**

- **AVG (Average Rate) – Use for Revenue**
  - Represents monthly average FX.
  - Used globally for revenue, billing, and sales reporting.

- **CLO (Closing Rate) – Use for Balance Sheet**
  - End-of-period rate.
  - Used for inventory valuation and other balance sheet metrics.

Finance standard:
- **Revenue → Average (AVG)**
- **Balance Sheet → Closing (CLO)**

---

### **Conversion Flow**

Ideal world:
1. Transactional Currency  
2. → Entity Currency  
3. → USD using average or closing rates  

Reality in current BD datasets:
- Transactional currency is **often missing**
- Only entity currency is available  
- Some systems convert USD → LC upstream, creating FX variance illusions

**Example FX distortion:**
- Budget: $100  
- Actual: $100  
- Converted to EUR using different rates → EUR 85 vs EUR 86 → FX variance introduced  

---

### **Practical Rule for Bowler / Billing**
- If **local currency** is available → convert using **monthly AVG**
- If only **entity currency** exists → treat it the same way for consistency

**Final rule:**  
➡️ **Use average monthly FX rates unless Finance explicitly requires otherwise.**

---

## 2. Geography Mapping

Analytic reporting requires consistent **BD Finance geography mapping**.

**Sources Used:**
- `g_bpc_finance.v_revenue_plan_country`
- `g_bpc_finance.v_country_global_hier`
- Manual mapping file: **WW Geo.xlsx**

### **Inventory Site Master**

For inventory reporting, site-to-country mapping is needed.

**Source:**
- `hub_prd.s_isc.site_master_all_sources`

**Key:**  
- `siteId`

**Columns required:**
- `CountryName`  
- `SiteName1`  
- `SiteName2`  
- `City`

Purpose:
- Distinguish **local vs. global** inventory  
- Build geography alignment for all inventory views  

---

## 3. 📅 Date Intelligence  
Centralized time logic ensures Bowler reports use one unified definition of month, quarter, year, and rolling periods.

### What We Created

#### **1. Unified Date Dimension – `g_dimdate_analytics_ipd`**
- Day-level date table
- Aligned to BD fiscal calendar (Oct–Sep)
- Fiscal + calendar attributes:
  - FY, FQ, FM
  - CY, CQ, CM
  - Rolling periods
  - Offsets for time intelligence

#### **2. Central Period Slicer – `g_periodslicer_analytics_ipd`**
- Attaches high-level period labels:
  - Calendar YTD
  - Fiscal YTD
  - Fiscal QTD
  - Prior YTD
  - Rolling 12M
- One slicer controls all dashboards
- No need to redo YTD/QTD logic in each model

---

### Natural Keys for Dates in Power BI

Power BI uses **natural date values** to join fact tables to the unified date dimension—not surrogate numeric keys.  
To support this, each fact table duplicates its original date fields into standardized **natural key columns** following this naming convention:

- The `nkey_*` fields link directly to `skeyDate` in the unified date dimension
- Ensures all facts share one calendar logic

---

### Integration Across Facts

All Bowler fact tables link to the unified date dimension:

- Orders  
- Billing  
- Inventory  
- Finance  
- Backorders  
- Forecast  
- Complaints  
- R&D (planned)

---

### 📂 Code Repositories
- **`nb_bowler_dimdate_unified`** → Creates `g_dimdate_analytics_ipd`  
- **`nb_bowler_periodslicer`** → Creates `g_periodslicer_analytics_ipd`

---

### ✅ Benefits
- One single source of truth for **time** and **period slicing**
- All Bowler datasets use consistent calendar logic
- Removes duplicated DAX/SQL logic for YTD/QTD/Rolling periods
- Fully scalable across regions and fiscal-year changes

## 4. 📦 Product Hierarchy

### Why Product Helpers Are Needed
- Supply systems use **natural material numbers**  
- Finance uses **CFN (Customer-Facing Number)**  
- Reltio Product Hierarchy uses **ReltioInternalID**

To unify all systems:
- We integrate **Material LOTS master** tables to bridge natural product IDs with CFN codes  
- This enables mapping to **ReltioInternalID**  
- Result: Supply, Finance, SFDC, and Billing products all align to the same hierarchy

### Key Challenges Solved
- ECC products without CFN  
- Finance-only CFN tables  
- Missing links between material → CFN → Reltio  
- Product lineage inconsistencies across systems  

### Source Used
- `g_bpc_finance.v_reltioproduct_hier`
- Material LOTS Master tables (supply systems)

**Outcome:**  
➡️ All facts can join the same **DimProduct** in Power BI through consistent `nkeyProduct` → `skeyProduct` logic.

---

## 5. 🧑‍🤝‍🧑 Customer Master (Future)

A unified customer master is not yet established but will be required for:

- Consistent customer naming  
- Distributor consolidation  
- Linking SFDC → Orders → Billing  
- Global customer reporting  
- Removing mismatched customer ID structures between systems  

This helper will eventually become:
- **DimCustomer** with global surrogate keys  
- Mapped to nkey fields in each fact


