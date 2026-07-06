### **Inventory Pipeline Overview**

The **Inventory pipeline** uses the existing table `hub_prd.s_isc.inventory_all_sources`, which contains worldwide inventory data, including status and location. The pipeline filters this dataset to include only IPD-relevant materials, ensuring alignment with business scope and integration with other domains such as Billing, Orders, and Finance.

---

#### **Step 1: IPD Filtering**
The dataset is filtered to include only IPD materials by joining `MaterialNumberHarmonized` from `hub_prd.s_isc.inventory_all_sources` with the authoritative material reference table `hub_prd.s_isc.material_master_reltio`. The filter is applied based on `ReltioLevel1Desc` to retain only materials belonging to the in-scope IPD categories:

- *Alaris System*
- *Hazardous Drug Solutions*
- *IV Solutions*
- *Gravity and Syringe Delivery*
- *IV Access*
- *CME*
- *Alaris Plus*
- *BD Nexus*
- *IPD OEM*

#### **Step X: Adding s_keys to dataproduct**

The resulting curated and filtered dataset is published as the view:

`hub_sbx.g_ipd.g_bowler_inventory_all_ipd`

- **Source Table:** `hub_prd.s_isc.inventory_all_sources`
- **Reference Table:** `hub_prd.s_isc.material_master_reltio`
- **Output View:** `hub_sbx.g_ipd.g_bowler_inventory_all_ipd`
- **SQL File:** `pipelines/inventory/sql/create_inventory_ipd_view.sql`

---

#### **Step 3: Size optimization for reporting**

# e	525,293
  Cannot be improved as the inventory is a snapshot - there is not a dedicated date - its always live

      # Row(Letter='e', Count=spark.table("hub_sbx.g_ipd.g_bowler_inventory_all_ipd").count()),

---

#### **Quality Check**

The validation of this pipeline is based on replicating the metrics and numbers displayed in the **Global order dashboard** used by Supply. The pipeline output should match the dashboard totals for IPD scope, ensuring consistency with existing reporting standards.

[https://app.powerbi.com/groups/me/apps/aa08e86b-5cde-420c-9cb7-3aec53ef4f87/reports/0a589932-e6ac-4c71-8d02-68633766d324/ReportSection5f400a6acb300504527e?ctid=94c3e67c-9e2d-4800-a6b7-635d97882165&experience=power-bi](https://app.powerbi.com/groups/me/apps/aa08e86b-5cde-420c-9cb7-3aec53ef4f87/reports/0a589932-e6ac-4c71-8d02-68633766d324/5ca5b234848c5f4537eb?ctid=94c3e67c-9e2d-4800-a6b7-635d97882165&experience=power-bi](https://app.powerbi.com/groups/me/apps/aa08e86b-5cde-420c-9cb7-3aec53ef4f87/reports/0a589932-e6ac-4c71-8d02-68633766d324/ReportSection5f400a6acb300504527e?ctid=94c3e67c-9e2d-4800-a6b7-635d97882165&experience=power-bi)

---

#### **Notes for Reporting**
- **Transit Inventory:** Identified using `PrimaryLocationFlag = 'S'`.
- **Regular Inventory:** Classified by `InventoryStatusHarmonized`.

---

This approach ensures a lean, harmonized dataset that aligns with business scope and supports efficient integration with other core pipelines.

---

!Inventory Pipeline Diagram
<img width="2450" height="664" alt="image" src="https://github.com/user-attachments/assets/64c02ccb-ae9c-4194-a07b-c2369c7fc8bd" />

