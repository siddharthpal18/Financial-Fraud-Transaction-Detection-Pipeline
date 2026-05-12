Step_3_Major_Transformation_Gold.md

Overview


Cleaned Delta tables from the Silver layer are aggregated into 12 distinct dimensions and fact tables.  

Dual-Write Strategy

Gold Layer: Tables are saved in partitioned Delta format within ADLS Gen2 for optimized analytics.  

Serving Layer: The same 12 tables are loaded into an Azure SQL Database via JDBC for direct Power BI consumption.  