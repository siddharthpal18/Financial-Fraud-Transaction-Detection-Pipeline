Step_1_ADF_Ingestion.md

Overview


Azure Data Factory (ADF) orchestrates the ingestion of the raw financial dataset from a Kaggle HTTP endpoint into the Bronze 
container of ADLS Gen2. This serves as the entry point for the entire fraud detection pipeline.  


ConfigurationActivity:
 	Copy Activity (Binary).  
	Source: HTTP GET request with ZipDeflate compression.  
	Sink: ADLS Gen2 (Binary) with FlattenHierarchy behavior and .csv extension.  
	Storage Path: bronze/Financial_fraud_dataset/Financial_fraud_dataset.csv.  


Challenges & Resolutions

	Decompression: Configured ZipDeflateReadSettings to allow ADF to automatically decompress the source ZIP file upon download.  	Special Characters: Enabled quoteAllText on the sink to handle special characters within the CSV for downstream processing.