Senior Data Engineer Assessment – ETL Pipeline (Pandas + Jupyter)
---------------------------------------------------------------------
Overview:
This project implements a production-grade ETL pipeline using **Pandas** and **Jupyter Notebook**, as a solution to the Senior Data Engineer technical assessment. It performs data ingestion, cleaning, enrichment (via external API), and loads the final cleaned data into an SQL database.
---------------------------------------------------------------------

Tools Used:
- Python 3.x
- Jupyter Notebook
- Pandas
- SQL Server (can be replaced with Azure SQL DB)
- `requests` for API integration
- `logging` for error tracking (optional)

---------------------------------------------------------------------


| File                                  | Description |

| `ETL_pipeline.ipynb` | Main notebook implementing the ETL process |
| `sales_data.csv` | Raw sales data |
| `product_reference.csv` | Reference product data for lookup |
| `schema.sql` | SQL script for the target schema |
| `README.md` | This documentation file |
| `sales_clean.db` | Output SQLite database (optional) |
---------------------------------------------------------------------
 ETL Process Steps

1. **Load raw data** from `sales_data.csv` and `product_reference.csv`.
2. **Clean** data by:
   - Removing nulls and duplicates
   - Validating numeric fields (e.g., `SaleAmount`)
3. **Enrich** data by:
   - Merging product reference info
   - Converting all currencies to USD using external exchange rate API
   - Logging each conversion (timestamp, rate, original & converted amount)
4. **Handle errors** by:
   - Logging failed records with details
   - Archiving rejected records
   - Aborting pipeline if >5% of records fail
5. **Load** the cleaned data into an SQL database (`SQL SERVER` used for demo)
6. **Export** logs and rejected data into separate tables
-----------------------------------------------------------------------
How to Run

1. **Clone the repo** or download the project files.
2. Make sure you have Python installed (preferably 3.7+).
3. Install required libraries:

# pip install pandas requests jupyter

# Launch the Jupyter Notebook:

Open etl_pipeline.ipynb, run all cells step by step.

Optional: Inspect sales_clean.db using any SQLite viewer or migrate schema to Azure SQL.

# External API
The notebook fetches live exchange rates from:

https://api.exchangerate-api.com/v4/latest/USD

///////////////////// Assumptions and Notes
Currency values are assumed to be in Currency column in sales_data.csv.

ProductID is the common key for joining product reference data.

The solution uses SQL server locally or SQLite viewer, but can be adapted for Azure SQL DB using pyodbc or sqlalchemy.


-----------------------------------------------------------------------
////1- Sample code using Pandas in Jupyter Notebook: 

import pandas as pd
import numpy as np
import requests
import sqlite3  # or use pyodbc / sqlalchemy / Azure SQL
from datetime import datetime
import logging


////2- Upload Data:

# تحميل البيانات من ملفات CSV
sales_df = pd.read_csv('sales_data.csv')
product_ref_df = pd.read_csv('product_reference.csv')

////3- Data cleaning:

# Remove empty values
sales_df.dropna(inplace=True)

# Remove duplicates
sales_df.drop_duplicates(inplace=True)

# Validate values (numeric columns)
sales_df = sales_df[sales_df['SaleAmount'] > 0]

////4- lookup data linking:
df = sales_df.merge(product_ref_df, how='left', on='ProductID')

////5- Convert currencies using API:

def fetch_exchange_rates(base='EUR'):
try:
url = f'https://api.exchangerate-api.com/v4/latest/{base}'
response = requests.get(url)
response.raise_for_status()
return response.json()['rates']
except Exception as e:
print(f"API error: {e}. Revert to default rates.")
return {'USD': 1.0, 'EUR': 1.1, 'GBP': 1.3} # Default values

exchange_rates = fetch_exchange_rates()


////6- Convert SaleAmount to USD and record transactions:

conversion_logs = []

def convert_to_usd(row): 
currency = row['Currency'] 
amount = row['SaleAmount'] 
rate = exchange_rates.get(currency, None) 

if rate: 
converted = amount / rate 
conversion_logs.append({ 
'Timestamp': datetime.now(), 
'OriginalCurrency': currency, 
'Rate': rate, 
'OriginalAmount': amount, 
'ConvertedUSD': converted, 
'RecordID': row['SaleID'] 
}) 
return round(converted, 2) 
else: 
raise ValueError(f"Missing exchange rate for {currency}")

# Conversion application
try: 
df['SaleAmountUSD'] = df.apply(convert_to_usd, axis=1)
except Exception as e: 
print(f"Error in conversion: {e}")

////7- Log errors and save rejected records:

error_logs = []
rejected_records = []

def safe_apply(row):
    try:
        return convert_to_usd(row)
    except Exception as e:
        error_logs.append({
            'Timestamp': datetime.now(),
            'Error': str(e),
            'RecordID': row['SaleID']
        })
        rejected_records.append(row)
        return np.nan

df['SaleAmountUSD'] = df.apply(safe_apply, axis=1)
df_clean = df.dropna(subset=['SaleAmountUSD'])

////8- Execution fails if errors exceed 5%.:

error_rate = len(rejected_records) / len(df)
if error_rate > 0.05:
    raise Exception("Error rate exceeded 5%. Job failed.")

////9- Load data into SQL:

conn = sqlite3.connect('sales_clean.db')
df_clean.to_sql('cleaned_sales_data', conn, if_exists='replace', index=False)

# Save logs of errors and rejections
pd.DataFrame(error_logs).to_sql('error_log', conn, if_exists='replace', index=False)
pd.DataFrame(rejected_records).to_sql('rejected_records', conn, if_exists='replace', index=False)

------------------------------------------------------------------------------------------------------------------------------












