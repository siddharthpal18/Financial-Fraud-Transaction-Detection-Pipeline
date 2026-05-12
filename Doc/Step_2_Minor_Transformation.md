Step_2_Minor_Transformation.md


Overview
A Synapse notebook reads raw CSV data from the Bronze layer, applies data cleaning and type-casting, and writes the output as a Delta table to the Silver layer.  

Transformations Applied


Standardization: Applied initcap, trim, lower, and upper to string columns like transaction_type, fraud_label, and ip_address.  


Type Casting: Converted identifiers (Transaction_ID, Customer_ID) to IntegerType and Transaction_Date to a proper date format.  


Data Quality: Executed dropna(how='any') to remove 1,234 records with NULL values, retaining 96.4% quality data for the ML model.  


Metadata: Added load_date and load_timestamp for audit tracking.