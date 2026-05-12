# Financial-Fraud-Transaction-Detection-Pipeline

### End to End Azure Data Engineering Project

#Repository	                                             Tech Stack
Fraud Transaction Analysis Pipeline	Azure Data Factory | Azure Synapse Analytics | Pyspark
ADLS Gen2 | Azure SQL | Power BI | Machine Learning (Logistic Regression and DecisionTree)

 

## Project Overview

This project executes a full end to end Azure Data Engineering Pipeline for a Financial Transaction Fraud Dataset. Raw data is ingested from an HTTP source into the Bronze layer, cleaned and standardize before storing it to the silver layer with the help of Azure Synapse Analytics and Spark SQL and stored in Delta table format. Later, it has been aggregated into dimensions and fact tables i.e. in Delta format, which is further taken by Power BI for visualization purpose and similarly, for the prediction purpose driven by Machine Learning algorithmns namely Logistics Regression.

The pipeline follows the Medallion Architecture (Bronze -> Silver -> Gold) and is orchestrated entirely by Azure Data Factory.

 
## Architecture Diagram:
 
<img width="940" height="676" alt="image" src="https://github.com/user-attachments/assets/ca551ef2-313a-4189-94f2-d0a6b7253c1f" />


Layer	Description
Bronze	Raw CSV data copied from Kaggle HTTP source via ADF Copy Activity into ADLS Gen2
Silver	Cleaned & typed Delta table — Minor Transformation Synapse notebook
Gold	12 aggregated Delta tables partitioned for analytics — Major Transformation notebook
Azure SQL	Same 12 tables loaded via JDBC for direct Power BI Import / Synapse Notebook for Machine Learning
Power BI	8 chart visuals 

## Setup Steps
1.	Create ADLS Gen2 storage account adlstoragesiddharth with containers: bronze, silver, gold
2.	Create Azure SQL Database AzureSQLDB on server sqldbc37.database.windows.net
3.	Create Synapse workspace and setup System assigned managed identity. And inside the Synapse Workspace  provide ROLE BASED ACCESS CONTROL rule to the user.
4.	Store SQL password in DBSecrets under key azsqldb
5.	Create Azure Data Factory and configure linked services: HTTP source, ADLS Gen2 sink, Synapse
6.	Write  MinorTransformation.ipynb, Major_Transformation.ipynb and ml_notebooks as per your requirements to Synapse.
7.	Deploy and trigger the Financial_Fraud_Detection_Pipeline in ADF
8.	Connect Power BI Desktop to Azure SQL and build visuals from the 12 tables
9.	Use data from gold layer to perform machine learning tasks such as choosing, preparing features, standardizing and feeding to the algorithms.

 


 
## Step 1 — ADF Data Ingestion (HTTP -> Bronze)

Overview
Azure Data Factory copies the raw Financial Transaction Fraud Detection Dataset CSV from a Kaggle HTTP endpoint into the Bronze container of ADLS Gen2. This is the first activity in the fraud_transaction_detection_Pipeline and acts as the entry point for all downstream processing.

### Configuration
Setting	FinancialDataCopy
Activity type	Copy Activity
Source type	Binary (HTTP)
Request method	GET
Compression	ZipDeflate
Sink type	Binary (ADLS Gen2)
Copy behaviour	FlattenHierarchy
File extension	.csv
Quote all text	true
Staging enabled	false
Timeout	12 hours
Retry	0
Sink container	bronze
Sink folder	Financial_fraud_dataset
Input dataset	Financial_fraud_dataset.csv
Output dataset	Financial_fraud_dataset.csv

## ADF Pipeline JSON 
 <img width="940" height="467" alt="image" src="https://github.com/user-attachments/assets/19e4af0f-345e-4400-807a-2d4b28871f39" />

Challenges & Resolutions
Challenge	Resolution
ZIP compression on HTTP source	Added ZipDeflateReadSettings to formatSettings so ADF auto-decompresses on download
Special characters in CSV	Enabled quoteAllText on the sink and set quote/escape chars in the downstream Spark reader
Pipeline dependency ordering	Set financecsvdata as the predecessor so Synapse notebooks only trigger after copy succeeds


Step 2 — Minor Transformation (Bronze -> Silver)

Overview
The Minor Transformation Synapse notebook reads raw CSV from Bronze, applies cleaning and type-casting, filters invalid records, and writes the result as Delta to the Silver layer. This is the second activity in the pipeline and runs only after the Copy Activity succeeds.

Configuration
Setting	Value
Notebook path	 /MinorTransformation
Linked service	AzureSynapseAnalyticsLS
Activity type	SynapseNotebook
Depends on	financecsvdata — Succeeded
Timeout	12 hours
Source path	abfss://bronze@adlstoragesiddharth.dfs.core.windows.net/Financial Transaction Fraud Dataset/FraudShield_Banking_Data.csv

Sink path	abfss://silver@adlstoragesiddharth.dfs.core.windows.net/fraud_detection/clean_transactions

Output format	Delta (overwrite mode)

Transformations Applied
Column	Operation
transaction_type, merchant_category, transaction_location, customer_home_location, card_type	initcap + trim
fraud_label	upper + trim
ip_address	lower + trim
transaction_amount, distance_from_home, account_balance, daily_transaction_count, weekly_transaction_count, avg_transaction_amount, max_transaction_last_24h, failed_transaction_count, previous_fraud_count	Numeric validation
transaction_date, transaction_time	to_date/to_timestamp
load_date, load_timestamp	Auto-generated

 
Challenges & Resolutions
Challenge	Resolution
NULL values scattered across 24 columns	Applied dropna(how='any') to remove entire rows with any NULL. Removed 1,234 records keeping 96.4% quality data for ML model
Column names with inconsistent formatting and spaces	Applied multi-step column rename: remove " (in Million)" suffix, replace spaces with underscores, replace hyphens with underscores, convert to lowercase
Column names have leading/trailing whitespace	Used cols.strip() to trim whitespace from column names only
String column data types instead of proper integer IDs
	Cast Transaction_ID, Customer_ID, Merchant_ID, Device_ID to IntegerType for proper numeric operations
	
Transaction_Date stored as string not date type	Used to_date(col("Transaction_Date"), "yyyy-MM-dd") to convert to proper date format
Need to track when data was processed for audit trail	Added load_date and load_timestamp metadata columns using current_date() and current_timestamp()
Duplicate transactions causing skewed aggregations	Checked for duplicates using COUNT - COUNT(DISTINCT) but no removal applied; duplicates remain in data

 
Step 3 — Major Transformation (Silver -> Gold + Azure SQL)
Overview
 The Major Transformation stage involves aggregating the cleaned data from the Silver layer into dimensions and fact tables. These tables are stored in Delta format in the Gold layer for visualization and machine learning purposes. 
Key Details:
Architecture: Medallion Architecture (Silver to Gold transition). 
Output Tables: A total of 12 aggregated Delta tables are created and partitioned for analytics. 
Dimensions
 

 
Facts
 
Storage & Sink:
Gold Layer: Tables are stored as Delta files in ADLS Gen2 for analytics. 
Azure SQL Database: The same 12 tables are loaded via JDBC into AzureSQLDB on the server :sqldbc37.database.windows.net. 
Configuration
Setting	Value
Notebook path	/Major_Transformation
Linked service	AzureSynapseAnalyticsLS
Activity type	SynapseNotebook
Depends on	Min_Transformation
Silver source	abfss://silver@adlstoragesiddharth.dfs.core.windows.net/fraud_detection/clean_transactions/

Gold base path	abfss://gold@adlstoragesiddharth.dfs.core.windows.net/Financial_Transaction_Fraud_Dataset/dimensions/dim_customer

SQL server	sqldbc37.database.windows.net
SQL database	AzureSQLDB
SQL user	siddharthdb
SQL driver	com.microsoft.sqlserver.jdbc.SQLServerDriver
Write mode	Append or ignore(both destinations)

Downstream Consumption:
Power BI: Connects to the 12 tables in Azure SQL to build 8 chart visuals. 
Machine Learning: The Gold layer data is used for feature preparation and feeding into algorithms like Logistic Regression and Decision Tree. 
Technical Execution:
•	Tools: Azure Synapse Analytics and PySpark. 
•	Notebook: The Major_Transformation.ipynb notebook is deployed to Synapse to handle these aggregations. 
•	Security: Credentials for the Azure SQL load are managed using Azure Key Vault, specifically storing the SQL password under the key azsqldb in DBSecrets

Step 4 — Power BI Visualization
 

This step details the development of the Fraud Transactions Analysis dashboard, designed to monitor and identify anomalous financial activities. 
Dashboard Overview: The dashboard provides a high-level summary of transaction health, focusing on the identification of fraudulent versus normal activity. 
Key Performance Indicators (KPIs):
Transaction Volume: Monitors 98.22K Total Transactions and 392.14K Daily Transactions.
Fraud Metrics: Highlights 4.699K Total Frauds, representing approximately 4.78% of total transactions.
Operational Health: Tracks 97.48K Failed Transactions and 98.207K International Transactions.
Visual Elements and Insights

No. of Merchants by Fraud Labels (Bar Chart): A horizontal bar chart comparing "Normal" vs. "Fraud" labels, used to identify the scale of merchant-related fraud.

Fraud Transactions (Pie Chart): A distribution chart showing the ratio of Normal (95.22%) to Fraud (4.78%) labels.

Transaction Detail Table: A granular view displaying sum_of_transaction_amount, merchant_id, fraud_label, and unusual_time_transaction status to assist in investigative drilling.

Data Formatting & Polish: Numerical values are standardized in the Format pane to ensure 



 
Step 5 — Machine Learning Prediction
This step leverages high-quality data from the Gold layer to build and evaluate predictive models for identifying fraudulent transactions.

Algorithm Selection: The project implements supervised learning using Logistic Regression and Decision Tree algorithms to classify transactions as "Fraud" or "Normal". 

Workflow (ml_notebook.ipynb):
Feature Engineering: Critical features such as transaction_amount, failed_transaction_count, distance_from_home, daily_transaction_count, and previous_fraud_count are selected and standardized for the model. 

Data Preparation: The notebook includes a manual data entry check for specific test cases to verify model logic.

Data Quality Impact: The models benefit from a 96.4% data quality rate established in the Silver layer, where 1,234 records with NULL values were removed to prevent model bias. 

Execution Environment: All machine learning tasks—including feature preparation, scaling, and algorithm training—are performed within Azure Synapse Analytics using Spark-based libraries. 

Business Logic Integration: The predictive model aims to automate the detection of the 4.78% of transactions identified as fraudulent in the historical analysis, focusing on patterns like unusual transaction times or high failed transaction counts.

 


Contributors

Siddharth Pal


