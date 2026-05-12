Step_5_Machine_Learning_Prediction.md

Overview
The Gold layer data is used to build predictive models within Azure Synapse Analytics to classify future transactions.  

Workflow


Algorithms: Implementation of Logistic Regression and Decision Tree models.  

Features: Key indicators used include transaction_amount, failed_transaction_count, distance_from_home, and previous_fraud_count.

Logic Verification: The ml_notebook.ipynb includes specific test cases, such as evaluating a transaction with an amount of 455 and a high distance from home, to verify prediction accuracy.