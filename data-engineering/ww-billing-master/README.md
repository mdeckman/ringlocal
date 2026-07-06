# Billing Pipeline – IPD Worldwide Consolidation

## 1. Purpose
The Billing pipeline creates and maintains **regional billing master tables**, filters them to **IPD scope** using Reltio/unified material master, and consolidates them into a **worldwide billing table**.  
This ensures consistent analytics across regions while adhering to medallion architecture, naming conventions, and column consistency.  

👉 Only the **columns relevant to the bowler** are retained in the worldwide billing master (we do not take all columns from each region’s Silver billing table).

---

## 2. Regional Billing Master (Silver Layer)

For each region, billing data is validated and reduced to required attributes, then filtered to **IPD**.  

- **United States (US)**  
  - Source: `hub_prd.s_commops_na.billing_master_us`  
  - Enrichment: `hub_prd.s_commops_na.unified_material_master_na`  
  - Special logic:  
    - `MaterialId` extracted from `MaterialKey` (text after `"|"`, e.g., `ECC|2420-0007 → 2420-0007`).

- **Canada (CA)**  
  - Status: **To Reproduce – no native Silver table exists (only a Gold view)**.  
  - Action: Reproduced as **Silver in PRD IPD** to align with other regions.  
  - Sources consolidated into the reproduced Silver:  
    - `s_commops_na.direct_sales_everest`  
    - `s_shared.billing_hdr_itm_everest`  
    - `s_commops_na.traced_sales_everest`  
    - `s_commops_na.traced_sales_agreement_conditional_pricing_everest`  
    - `s_commops_na.traced_sales_measures_everest`  
    - `s_commops_na.direct_sales_jde`  
    - `s_commops_na.direct_sales_tahiti`  
    - `s_commops_na.traced_sales_tahiti_rebated`  
    - `s_commops_na.traced_sales_tahiti_non_rebated`  
    - `s_commops_na.sales_i5_ca`  
  - Reference: [CA Lineage](https://bd.alationcloud.com/app/table/low)  
  - Enrichment: US unified material master view (assumption).  
  - Special logic:  
    - `MaterialId` derived from `MaterialKey` (same split logic as US).  
  - Action: **Ingested at the end of the process** after US and GA.  

- **Greater Asia (GA)**  
  - Status: **To Reproduce – no native Silver table exists (only a Gold view)**.  
  - Action: Reproduced as **Silver in PRD IPD** from raw sources.  
  - Sources consolidated:  
    - `s_commops_ga.direct_sales_everest`  
    - `s_shared.calendar_periods_ga`  
    - `s_commops_ga.direct_sales_jde`  
    - `s_commops_ga.traced_sales_everest`  
  - Enrichment: `hub_prd.s_commops_ga.unified_material_master_ga`  
  - Final structure aligned to the same schema as other regions.  

- **LATAM**  
  - Source: `hub_prd.s_region_latam.billing_master_latam`  
  - Enrichment: `hub_prd.s_region_latam.unified_material_master_latam`  
  - Billing key = `(SourceSystemId, MaterialId)`  
  - `MaterialKey` ensured to be non-null by join with material master.

- **India**  
  - Status: **TBD** – confirm scope.

---

## 3. IPD Filtering
- **US/CA/GA**: Filtered via Reltio attribute `ReltioPHLevel1`.  
- **LATAM**: Filtered via `ErpPhLevel1` from LATAM unified material master.  
- **Constraint**: No permanent storage of Reltio attributes; filtering applied at query time only.

---

## 4. Standardization Rules
All regional outputs are aligned to a **common schema** before union:  

| Column                | Type           | Notes |
|------------------------|----------------|-------|
| MaterialId             | STRING         | Always text (from `MaterialKey` split or source `MaterialId`) |
| MaterialKey            | STRING         | Preserved as-is |
| ErpCustomerKey         | STRING         | Nullable in LATAM |
| DistributorKey         | STRING         | Nullable in LATAM |
| ShipToAccountNumber    | STRING         | |
| SoldToAccountNumber    | STRING         | |
| InvoiceDate            | DATE           | Cast applied |
| IndirectSellDate       | DATE           | Nullable where missing |
| DateAdded              | DATE           | Cast applied |
| BillingTypeDescription | STRING         | |
| BillingQuantity        | DECIMAL(38,10) | |
| InvoiceQuantity        | DECIMAL(38,10) | Nullable where missing |
| ActualNetRevenue       | DECIMAL(38,10) | |
| DistributionChannel    | STRING         | |
| TransactionType        | STRING         | |
| ActualGrossProfit      | DECIMAL(38,10) | Nullable where missing |
| Country                | STRING         | |
| Currency               | STRING         | Unified from `LocalCurrency` |
| DirectSalesViewFlag    | BOOLEAN        | Nullable in LATAM |
| CustomerViewFlag       | BOOLEAN        | Nullable in LATAM |
| SalesFlag              | BOOLEAN        | Nullable in LATAM |
| Source                 | STRING         | Region + table lineage |

---

## 5. Worldwide Consolidation 

A **union of all standardized IPD billing tables** produces the consolidated dataset:  

- **Input**: US, CA, GA, LATAM (future: India, EMEA).  
- **Process**:  
  1. Regional filtering + enrichment in Silver.  
  2. Schema alignment with consistent types.  
  3. **UNION ALL** into worldwide table.  
- **Output**:  
  - Table: `hub_sbx.s_ipd.s_billing_master_ww_ipd`  
  - Used for Power BI and global reporting.  
  - Follows **PascalCase column naming**.
 
---

## 6. Size optimization for reporting

line counts in tables filtered for IPD Silver:
# a	3,779,781
# b	25,116,211
# c	749,022
# d	1,509,628
  Improvement made: - Aggregated all lines - sum on quantities, revenue and gross profit
                    - Decision to get to lower lines and filtered on CY only - dropped to appx. 3M lines for WW (w/o EMEA)                      

    # Row(Letter='a', Count=spark.table("hub_prd.s_region_latam.billing_master_latam").count()),      
    # Row(Letter='b', Count=spark.table("hub_prd.s_commops_na.billing_master_us").count()),        
    # Row(Letter='c', Count=spark.table("hub_sbx.s_ipd.s_billing_master_ca_ipd").count()),               
    # Row(Letter='d', Count=spark.table("hub_sbx.s_ipd.s_billing_master_ga_ipd").count()),             

---

## 7. Notes

For US billing:

Customer View Flag = CCO report - Give same ActualNetRevenue
Give same Units for Tracings
But the Invoice and Billing quantity are different 

Direct Sales View Flag = Finance app - Give very close to the Global Finance App Revenue




