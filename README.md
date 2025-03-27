# ms_fabric_de_project

📌 In this project, I built a complete end-to-end data warehouse solution using Microsoft Fabric, following the Medallion Architecture. The pipeline starts by ingesting raw CSV files stored in GitHub. These datasets go through structured layers — Bronze for raw data, Silver for cleaned and transformed data, and Gold for dimensional modeling in a star schema using Fabric’s Data Warehouse. Finally, I connected the warehouse to Power BI to build a dashboard for data visualization. This project showcases my hands-on experience with Microsoft Fabric ecosystem.

Table of Content

· Data Source Overview
∘ 📁 CRM System Files
∘ 🧾 ERP System Files
· Step 1: Setting Up the Fabric Workspace
· Step 2: Bronze Layer — Data Ingestion
∘ Creating the Bronze Lakehouse
∘ Building the Data Pipeline to Load Data from GitHub
∘ Raw Data Storage in Bronze Layer
· Step 3: Silver Layer — Data Cleaning & Transformation
∘ Creating the Silver Lakehouse
∘ Data Cleaning & Transformation using Notebooks (PySpark & SQL)
∘ Storing Processed Data in Silver Layer
· Step 4: Gold Layer — Data Warehouse Modeling
∘ Creating the Fabric Data Warehouse
∘ Building the Star Schema (Dimensional Modeling)
· Step 5: Data Visualization with Power BI
∘ Connecting Power BI to the Data Warehouse
∘ Building Dashboards and Visuals
· Step 6: Workspace Lineage and Deploying Workspace
∘ Check Workspace Lineage
∘ Deploying Workspace

Data Source Overview
For this project, I worked with six CSV files sourced from simulated CRM and ERP systems. These files were uploaded to my GitHub repository and served as the raw input for the data pipeline.

Source Files :
https://github.com/DinuAR/FabricDataWarehouseSourceFiles.git

📁 CRM System Files
cust_info.csv – Contains basic customer information
prd_info.csv – Includes product-level details
sales_details.csv – Holds transaction-level sales data
🧾 ERP System Files
CUST_AZ12.csv – Adds customer attributes like birthdate and gender
LOC_A101.csv – Maps customers to their countries
PX_CAT_G1V2.csv – Provides product category and subcategory information
These datasets represent a realistic business scenario with scattered data across different systems. The goal was to integrate, clean, and model this data into a unified structure for analytics using the Medallion Architecture in Microsoft Fabric.

Step 1: Setting Up the Fabric Workspace
To kick off the project, I created a new Microsoft Fabric workspace dedicated to this project. This workspace served as a centralized environment to organize and manage all the components I would use throughout the development lifecycle — such as Lakehouses, Data Pipelines, Notebooks, the Data Warehouse, and the Power BI.


Workspace
Step 2: Bronze Layer — Data Ingestion
Creating the Bronze Lakehouse
Following the Medallion Architecture pattern, the next step was to create a Bronze Lakehouse in my workspace. This Lakehouse acts as the raw data storage layer, where I ingest and store the unprocessed source files exactly as they are.


Bronze Lakehouse
Building the Data Pipeline to Load Data from GitHub
To ingest the raw source data into the Bronze Lakehouse, I created a Fabric Data Pipeline. This pipeline was configured to extract the CSV files directly from my GitHub repository and load them into the Bronze layer.

This is the home page of Fabric Data Pipeline Item


DataPipeline Home
Configured copy data activity to extract CRM data files from Github


Copy Data Activity Configurations
Created a new connection to connect with Github


Setup New connection to Github
Used a ForEach activity to dynamically extract all data files in Github CRM folder


ForEach Activity to extract all files in a folder
Set up the Copy Data activity inside the For Each activity.


CRM Data Extraction Activity
Configured the same setup to extract ERP data from GitHub and connected the components to complete the source data extraction pipeline.


Complete Data Pipeline
Running Data Pipeline.


Pipeline Status
Raw Data Storage in Bronze Layer
Once the data pipeline successfully runs, the source files are uploaded to the bronze data lake.


Bronze Data Lake with Source Files
Step 3: Silver Layer — Data Cleaning & Transformation
Creating the Silver Lakehouse
After setting up the Bronze layer, I created a second Fabric Lakehouse to serve as the Silver Layer in the Medallion Architecture. This Silver Lakehouse is used to store cleaned, transformed, and enriched data derived from the raw source files in the Bronze layer.


Silver Lakehouse
Data Cleaning & Transformation using Notebooks (PySpark & SQL)
To process the raw data stored in the Bronze Lakehouse, I created a Fabric Notebook. Using a combination of PySpark and SQL, I performed various data cleaning and transformation tasks — such as handling missing values, standardizing column formats, joining related datasets, and preparing the data for dimensional modeling.


Fabric Notebook

Fabric Notebook
Storing Processed Data in Silver Layer
Once the data cleaning and transformation were complete, I saved the processed datasets into the Silver Lakehouse. These cleaned files now represented a more structured and analytics-ready version of the original data.


Silver Lakehouse
Step 4: Gold Layer — Data Warehouse Modeling
Creating the Fabric Data Warehouse
Next, I created a Data Warehouse item in Microsoft Fabric to serve as the Gold Layer of the pipeline. This layer is designed to support advanced analytics and reporting by storing structured, query-optimized data. Using the cleaned and transformed datasets from the Silver Lakehouse, I began building the dimensional model.


Data Warehouse in Fabric
Inside the Fabric Data Warehouse, I created a new schema named gold to logically organize the dimension tables and fact table.

CREATE SCHEMA gold
GO
Building the Star Schema (Dimensional Modeling)
To implement the star schema model, I created the dimension and fact tables as SQL views within the gold schema of the Data Warehouse.

Dimension Tables: dim_customers, dim_products

create view gold.dim_customers as
select
row_number() over (order by cst_id) as customer_key,
cu.cst_id as customer_id,
cu.cst_key as customer_number,
cu.cst_firstname as first_name,
cu.cst_lastname as last_name,
lc.cntry as country,
cu.cst_marital_status as marital_status,
case when cu.cst_gndr != 'n/a' then cu.cst_gndr
     else coalesce(bd.gen, 'n/a')
end as gender,
bd.bdate as birth_date,
cu.cst_create_date as create_date
from Silver.dbo.crm_cust_info cu
left join Silver.dbo.erp_cust_az12 bd
on cu.cst_key = bd.cid
left join Silver.dbo.erp_loc_a101 lc
on cu.cst_key = lc.cid
create view gold.dim_products as
select
row_number() over (order by prd_start_dt, prd_key) as product_key,
pr.prd_id as product_id,
pr.prd_key as product_number,
pr.prd_nm as product_name,
pr.cat_id as category_id,
ct.cat as category,
ct.subcat as subcategory,
ct.maintenance,
pr.prd_cost as cost,
pr.prd_line as product_line,
pr.prd_start_dt as start_date
from Silver.dbo.crm_prd_info pr
left join Silver.dbo.erp_df_px_cat_g1v2 ct
on pr.cat_id = ct.id
where pr.prd_end_dt is null
Fact Table: fact_sales

CREATE VIEW gold.fact_sales as 
select 
sl.sls_ord_num as order_number,
pr.product_key,
cs.customer_key,
sl.sls_order_dt as order_date,
sl.sls_ship_dt as shipping_date,
sl.sls_due_dt as due_date,
CAST(sl.sls_sales AS float) as sales_amount,
CAST(sl.sls_quantity AS float) as quantity,
CAST(sl.sls_price AS float) as price
from Silver.dbo.crm_sales_details sl
left join gold.dim_products pr
on sl.sls_prd_key = pr.product_number
left join gold.dim_customers cs
on sl.sls_cust_id = cs.customer_id

Gold Layer

Star Schema Model
Step 5: Data Visualization with Power BI
Connecting Power BI to the Data Warehouse
To bring the data to life, I connected the Fabric Data Warehouse to Power BI. This allowed me to seamlessly access the gold schema views (dim_customer, dim_product, and fact_sales) and use them as the foundation for building an interactive report.


Data Warehouse tables in Power BI
Building Dashboards and Visuals
Here is the Power BI report I built using the data from the Fabric Data Warehouse.


PowerBI Report
Step 6: Workspace Lineage and Deploying Workspace
Check Workspace Lineage

Checking Workspace Lineage
Here is the lineage view of all the items in my Microsoft Fabric workspace.


My Workspace Lineage
Deploying Workspace
Once the development phase was complete, I deployed all my workspace items — Lakehouses, Pipelines, Notebooks, Data Warehouse, and Power BI report — into a new Microsoft Fabric workspace. This represented a clean, production-like environment.


Newly created production environment
New Deployment Pipeline


Deployment Pipeline Home
✅ Pipeline Successfully Deployed! 🚀


Deployed Pipeline
