### **Order Pipeline Overview**

The **Sales Orders pipeline** uses the existing table `hub_prd.s_shared.sales_orders_all_sources`, which contains both shipped and open orders. The pipeline focuses on curating a subset of essential columns and filtering the dataset to include only IPD-relevant materials, ensuring alignment with business scope and integration with other domains such as Billing, Inventory, and Finance.

---

#### **Step 1: Column Selection**
We retain only the relevant fields needed for analytics and joins:

- **Order Identifiers & Customer Details:** `OrderNumber`, `OrderType`, `ShipToNumber`, `SoldToNumber`, `ShipToName`, `SoldToName`, `Country`
- **Material Information:** `MaterialNumber`, `MaterialNumberHarmonized` (used for joining with material master)
- **Order Status:** `OrderStatusLineLast`, `OrderStatusLineNext`, `OrderStatusHeader`, `OrderStatusLineHarmonized`, `OrderStatusHeaderHarmonized`
- **Sales Channel:** `SalesDistributionChannel`, `SalesDistChannelHarmonized`
- **Quantities:** `OrderQuantityOriginal`, `OrderQuantityBase`, `DeliveredQuantityOriginal`, `DeliveredQuantityBase`, `OpenQuantityOriginal`, `OpenQuantityBase`, `CancelledQuantityOriginal`, `CancelledQuantityBase`, `BackOrderQuantityOriginal`, `BackOrderQuantityBase`
- **Pricing & Currency:** `TotalPriceLocal`, `TotalPriceUsd`, `UnitPriceLocal`, `UnitPriceUsd`, `CurrencyType`, `ExchangeRate`, `ExchangeDate`, `LocalCurrency`
- **Dates:** `CreateDate`, `PromisedDeliveryDate`, `RequestedDeliveryDate`, `ActualShipDate`, `LastChangeDate`, `ExtractDate`
- **Contractual Info:** `AgreementNumber`, `ContractStartDate`, `ContractEndDate`

---

#### **Step 2: IPD Filtering**
After selecting the required columns, the dataset is **filtered to include only IPD materials**. This is achieved by joining `MaterialNumberHarmonized` from `hub_prd.s_shared.sales_orders_all_sources` with the authoritative material reference table `hub_prd.s_isc.material_master_reltio`. The filter is applied based on `ReltioLevel1Desc` to retain only materials belonging to the in-scope IPD categories:

- *Alaris System*
- *Hazardous Drug Solutions*
- *IV Solutions*
- *Gravity and Syringe Delivery*
- *IV Access*
- *CME*
- *Alaris Plus*
- *BD Nexus*
- *IPD OEM*

The resulting curated and filtered dataset is published as the view:

`hub_sbx.g_ipd.g_bowler_sales_order_all_ipd`

- **Goal:** Ensure that only IPD-relevant materials are included for downstream processing and reporting.
- **Constraint:** Filtering occurs dynamically at query time; no additional Reltio data is stored in the pipeline output.

---

#### **Step 3: Size optimization for reporting**

## f	20,579,641
    Improvement made: - Filtering on IPD only - 25M
                      - Aggregation - down to 19M
                      - Filtering on CY + PY to account for lagging orders - Dropped to 461,754 lines
                      - Made an analysis on what should be included see below screenshot - relation of created order date to delivery - we took 3 years previous to be safe
                      - Gets us down to 
                      
    # Row(Letter='f', Count=spark.table("hub_sbx.g_ipd.g_bowler_sales_order_all_ipd").count())

---

Relationship of create date of order to actual ship date for data cut off:

<img width="440" height="560" alt="image" src="https://github.com/user-attachments/assets/de443c79-345c-4275-ab39-2ee09e6e1608" />

---

#### **Step 4: Data Issues with Gold Layer**

In the **Gold Layer** of the IPD data model, a recurring data integrity issue was identified in the `sales_order_all_sources` dataset.  
All **LOTS**-related orders lacked harmonized values for both **Sales Channel** and **Sales Order Status**, despite the presence of the original (“natural”) system fields.

These discrepancies originated from the **source harmonization gap** between LOT and other systems. The natural fields existed in the raw layer but were never mapped to their harmonized equivalents during the Silver → Gold transition.

##### **Observed Issues**
- `SalesChannelHarmonized` → null for LOT orders  
- `SalesOrderStatusHarmonized` → null for LOT orders  
- `SalesOrderStatus` (natural) → populated with LOT-specific codes
- `SalesDsitributionChannel` (natural) → populated with LOT-specific codes

##### **LOT Order Status Reference**

Below are the reference mappings for the native LOT order status codes:

<img width="1086" height="408" alt="LOT Status Reference 1" src="https://github.com/user-attachments/assets/241eedb9-1df9-4a51-bea9-54ee765e0019" />
<img width="1033" height="418" alt="LOT Status Reference 2" src="https://github.com/user-attachments/assets/efb211c7-28ce-4a30-9a5c-357189780ac4" />
<img width="1001" height="407" alt="LOT Status Reference 3" src="https://github.com/user-attachments/assets/3ae7ee6e-610f-4479-b019-28b5ce16b89b" />
<img width="1034" height="419" alt="image" src="https://github.com/user-attachments/assets/e4cc7662-1f42-4e70-9391-845e6edd5d8d" />
<img width="986" height="408" alt="image" src="https://github.com/user-attachments/assets/d153764f-f1db-40e3-ab8c-b1f9127f7fa3" />
<img width="1269" height="543" alt="image" src="https://github.com/user-attachments/assets/81f52630-ad79-4d93-bcb6-cc4008426bbf" />


##### **LOT Sales Channel Reference**
Mappings for LOT sales channels (to harmonized business taxonomy) are defined and applied in the same correction logic.

One option is to map organization selling to internal or external.
Internal demand is usually pointing to another selling entity / market in LOTS than direct customers.
 
Example:
delivery to external customer in France : market 1000
delivery to internal customer in France: market 1080 or market 1090

<img width="627" height="49" alt="image" src="https://github.com/user-attachments/assets/e7aab888-8c67-4650-89be-23160aca6d93" />
<img width="789" height="50" alt="image" src="https://github.com/user-attachments/assets/e7067a5c-e961-4ec5-bace-21dd0f057d82" />
<img width="1233" height="1008" alt="image" src="https://github.com/user-attachments/assets/a59ea5c2-b807-42d6-b182-e30d8fa57505" />

 otherwise its the mapping of the sales dist channel codes:
   HERE PUT GHULAM LOGIC
 


##### **Fix Implementation**
A **correction layer** was added in the **Silver → Gold transformation** of IPD:

- If `SourceSystem = 'LOT'`, use natural status and channel mappings from LOT lookup tables.
- Apply harmonized translation to populate:
  - `SalesOrderStatusHarmonized`
  - `SalesChannelHarmonized`
- These harmonized fields now appear correctly in reporting and Power BI consumption views.

> ✅ The fix ensures consistent cross-system comparability and alignment of LOT data with the unified BD harmonized data model.

---

#### **Quality Check**
The validation of this pipeline is based on replicating the metrics and numbers displayed in the **Global order dashboard** used by Supply. The pipeline output should match the dashboard totals for IPD scope, ensuring consistency with existing reporting standards.
  [https://app.powerbi.com/groups/me/apps/aa08e86b-5cde-420c-9cb7-3aec53ef4f87/reports/0a589932-e6ac-4c71-8d02-68633766d324/ReportSection5f400a6acb300504527e?ctid=94c3e67c-9e2d-4800-a6b7-635d97882165&experience=power-bi](https://app.powerbi.com/groups/me/apps/aa08e86b-5cde-420c-9cb7-3aec53ef4f87/reports/0a589932-e6ac-4c71-8d02-68633766d324/5ca5b234848c5f4537eb?ctid=94c3e67c-9e2d-4800-a6b7-635d97882165&experience=power-bi)

---

This approach ensures a lean, harmonized dataset that aligns with business scope and supports efficient integration with other core pipelines.


<img width="1991" height="787" alt="image" src="https://github.com/user-attachments/assets/a87a03c9-8192-407c-84bd-0ae46fb4a111" />

<img width="1295" height="348" alt="image" src="https://github.com/user-attachments/assets/5a703696-435d-474e-8749-75436011b31d" />


---

#### **Reporting note**

Actual ship date = recognized for exworks
Orderlinestatus = closed = recognized for non exworks (MENAT  etc)
