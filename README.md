# AzureSQL-to-DeltaTables-ETL-Pipeline

This project implements an incremental ETL pipeline from **Azure SQL Database** to **Azure Data Lake Storage** using **Azure Data Factory**, following the **Medallion Architecture** design pattern.

## 🔷 Architecture Overview

The pipeline is designed based on the Medallion Architecture:

- **Bronze Layer**: Raw data ingested incrementally from Azure SQL.  
- **Silver Layer** *(Next Phase)*: Transformed and cleansed data using Azure Databricks.  
- **Gold Layer** *(Future Work)*: Aggregated, business-ready data for analytics.

## ⚙️ Current Workflow: Azure SQL → ADLS Bronze

### 1. Watermark Table Creation

A watermark table is used to track the last successful load:

```sql
CREATE TABLE watermark (
  prev_load VARCHAR(200)
);

INSERT INTO watermark VALUES ('DT00000');  -- Set to a value less than the minimum Date_ID
```

### 2. Stored Procedure to Update Watermark

```sql
CREATE PROCEDURE updatewatermark
  @watermark VARCHAR(200)
AS
BEGIN
  BEGIN TRANSACTION;

  UPDATE watermark 
  SET prev_load = @watermark;

  COMMIT TRANSACTION;
END;
```

### 3. Azure Data Factory Pipeline

#### Lookup Activity – Previous Load

Query:
```sql
SELECT * FROM watermark;
```
Fetches the previous watermark value.

#### Lookup Activity – Current Load

Query:
```sql
SELECT MAX(Date_ID) AS max_date FROM cars_sales_data;
```
Fetches the most recent date from the source.

#### Copy Activity – Fetch Incremental Records

Query:
```sql
SELECT * FROM cars_sales_data
WHERE Date_ID > '@{activity('prev_load').output.value[0].prev_load}'
  AND Date_ID <= '@{activity('current_load').output.value[0].max_date}';
```
Pulls only new records between the previous and current watermark.

#### Stored Procedure Activity – Update Watermark

After successful copy, this activity runs:
```text
@watermark = @{activity('current_load').output.value[0].max_date}
```
Which updates the watermark for the next incremental load.

## ✅ Outcome

- Incrementally loads data from Azure SQL to **Bronze Layer** in ADLS.  
- Dynamically updates watermark after every run.  
- Ensures idempotent and optimized data movement using ADF.

## 🛠 Tools & Technologies

- Azure Data Factory  
- Azure SQL Database  
- Azure Data Lake Storage Gen2  
- Delta Lake *(planned for Silver Layer)*  
- Azure Databricks *(for upcoming transformations)*

## 🚧 Next Steps

- Build transformation logic in Azure Databricks.  
- Store cleaned data into the **Silver Layer** using Delta Tables.  
- Expand to **Gold Layer** for analytics and reporting.

---

*This README will be updated as Silver and Gold layers are implemented.*
