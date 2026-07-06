# SFDC Latest Snapshot View – g_bowler_ww_sfdc_latestsnap

This view provides the latest IPD-filtered SFDC opportunity snapshot for the current fiscal year. It is used as the Gold-level source for the Power BI semantic model (g_bowler_sfdc).

---

# Origin of This Logic

This pipeline was originally created by **Jennifer Olphin**.  
We are currently using the logic exactly as provided and **it has not yet been fully engineered or migrated** into the standardized Bowler Bronze → Silver → Gold framework.

The objective of keeping this version is:
- Preserve reporting continuity  
- Avoid breaking existing SFDC reporting  
- Maintain alignment with historical logic  
- Prepare for a future rebuild inside the engineered pipeline structure  

This view should be considered a **transitional compatibility layer**, not the final engineered implementation.

---

# What This View Does

## 1. Calculates the Current Fiscal Year Window

The view determines the start of the BD fiscal year (October 1) and limits the snapshot data to the current fiscal year only.

Logic used:
- Fiscal year begins October 1  
- SnapshotDate must be between fiscal year start and fiscal year start + 12 months

This ensures that only the current FY is included.

---

## 2. Reads Monthly SFDC Snapshots

The source table is: hub_prd.s_global_marketing.opportunity_monthly_snapshot


This table contains one row per opportunity per month and includes:
- Opportunity metadata  
- Product hierarchy metadata  
- Geographic ownership  
- Stage, probability, and close date  
- Monetized values in USD  

---

## 3. Filters to IPD Using Reltio Product Hierarchy

The join to Reltio filters the dataset to the IPD portfolio: hub_prd.g_bpc_finance.v_reltioproduct_hier

Filtering is based on Level1Desc values such as:
- Alaris System  
- Hazardous Drug Solutions  
- IV Solutions  
- Gravity and Syringe Delivery  
- IV Access  
- CME  
- Alaris Plus  
- IPD OEM  
- BD Nexus  

Only rows matching these product families are kept.

---

## 4. Aggregates Opportunity Amounts

Three monetized metrics are aggregated:

- AnnualAmountUSD  
- FiscalYearAmountUSD  
- NextYearCarryOverAmountUSD  

They are summed per:
- SnapshotDate  
- Opportunity  
- OpportunityLineItem  
- Product attributes  

This preserves the complete opportunity and product grain while ensuring correct monetization.

---

## 5. Preserves Detailed Opportunity and Product Attributes

The view keeps full granularity, including:

**Product hierarchy**
- Product  
- ProductSubset  
- ProductLine  
- ProductCategory  
- ProductSet  
- ProductFamily  
- ProductClassification  
- Platform  

**Opportunity fields**
- Id, LineItemId  
- Name, AccountName, Owner  
- Type, Stage, Probability  
- CreatedDate, CloseDate  
- ForecastCategory  

**Geographic fields**
- Region  
- SubRegion  
- Country  
- BusinessUnit  
- Segment  

This detailed structure supports all SFDC-related Bowler analytics.

---

# Final Output

The view created is: sts_prd.g_ipd.g_bowler_ww_sfdc_latestsnap

