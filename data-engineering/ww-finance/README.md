


---

#### **Step 3: Size optimization for reporting**

    # Row(Letter='g', Count=spark.table("hub_prd.g_bpc_finance.v_revenue_actual_mds_mms").count()),       
    # Row(Letter='h', Count=spark.table("hub_prd.g_bpc_finance.v_revenue_budget_mds_mms").count()),      
    # Row(Letter='i', Count=spark.table("hub_prd.g_bpc_finance.v_revenue_projection_mds_mms").count())   

# g	5,315,987
    Improvement made: - Filtering on IPD only
                      - Filtering on CY - Dropped to 325,892 lines
# h	1,522,598
    Improvement made: - Filtering on IPD only
                      - Filtering on CY - Dropped to 153,271 lines
# i	4,850,809
    Improvement made: - Filtering on IPD only
                      - Filtering on CY - Dropped to 625,769 lines


---

#### **Validation: Gold view in power bi to finance app**

<img width="2094" height="656" alt="image" src="https://github.com/user-attachments/assets/48cc5ed7-3522-4662-83b4-59286a534d37" />

<img width="1729" height="962" alt="image" src="https://github.com/user-attachments/assets/4814dfd8-b856-4345-a04c-2affb66351fa" />



