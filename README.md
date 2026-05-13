# CDC Incremental Load Pipeline using Azure Data Factory (ADF)

## Project Overview
This project implements **CDC (Change Data Capture) / Incremental Data Load** using **Azure Data Factory (ADF)**.

Goal:
- Read transaction data from **SQL Server**
- Load **only new records** into a CSV file
- Avoid duplicate loading
- Track last successful load using a **Watermark Table**

---

# Architecture

```text
SQL Server (SourceTransactions)
        |
        v
Lookup Activity (Read LastLoadDate from WatermarkTable)
        |
        v
Copy Activity (Load only new records)
        |
        v
CSV File (Target Storage)
        |
        v
Script Activity (Update WatermarkTable)
```

---

# Step 1: Create Source Table

```sql
CREATE TABLE SourceTransactions (
    TxnID INT,
    TxnDT DATE,
    TxnAmount DECIMAL(10,2),
    ProductCD VARCHAR(10),
    ProductQty VARCHAR(50)
);
```

## Sample Data

```sql
INSERT INTO SourceTransactions VALUES
(123,'2025-07-21',200,'BV','Cocacola'),
(124,'2025-07-21',1100,'CM','Lotion'),
(125,'2025-07-22',500,'ET','Biscuits'),
(126,'2025-07-22',200,'CH','Detergent');
```

---

# Step 2: Create Watermark Table

Watermark table stores the **last successful load date**.

```sql
CREATE TABLE WatermarkTable (
    PipelineName VARCHAR(100),
    LastLoadDate DATE
);
```

## Insert Initial Value

```sql
INSERT INTO WatermarkTable
VALUES ('CDC_Pipeline','2025-07-21');
```

---

# Step 3: Azure Data Factory Pipeline

## Activity 1: Lookup Activity

Purpose:
- Read `LastLoadDate` from `WatermarkTable`

### Query

```sql
SELECT LastLoadDate
FROM WatermarkTable
WHERE PipelineName='CDC_Pipeline';
```

---

## Activity 2: Copy Data Activity

Purpose:
- Load only new records from source table

### Dynamic Query

```sql
SELECT *
FROM SourceTransactions
WHERE TxnDT > '@{activity('Lookup_Watermark').output.firstRow.LastLoadDate}'
```

---

## Sink Configuration

Target:
- CSV File (DelimitedText)

Dynamic file name:

```text
TxnFile_@{formatDateTime(utcNow(),'ddMMMyyyy')}.csv
```

Example output:

```text
TxnFile_22Jul2025.csv
```

---

## Activity 3: Script Activity

Purpose:
- Update watermark after successful load

### SQL Script

```sql
UPDATE WatermarkTable
SET LastLoadDate = GETDATE()
WHERE PipelineName='CDC_Pipeline';
```

---

# Final Pipeline Flow

```text
Lookup_Watermark
       ↓
Copy_New_Records
       ↓
Script_Update_Watermark
```

---

# Example Incremental Load

## Day 1

Source:

| TxnID | TxnDT | Amount |
|-------|-------|--------|
| 123 | 21-Jul-2025 | 200 |
| 124 | 21-Jul-2025 | 1100 |

Output:

```text
TxnFile_21Jul2025.csv
```

---

## Day 2

New Source Records:

| TxnID | TxnDT | Amount |
|-------|-------|--------|
| 125 | 22-Jul-2025 | 500 |
| 126 | 22-Jul-2025 | 200 |

Output:

```text
TxnFile_22Jul2025.csv
```

Only new records are loaded.

---

# Technologies Used

- Azure Data Factory (ADF)
- SQL Server
- Azure Blob Storage / ADLS
- SQL

---

# Benefits

- Incremental data loading
- No duplicate records
- Better performance
- Easy maintenance
- Production-ready CDC pattern

---

# Interview Explanation

Implemented a CDC (Change Data Capture) incremental load pipeline using Azure Data Factory.

Used a **Watermark Table** to store the last successful load date.  
ADF **Lookup Activity** reads the watermark, **Copy Data Activity** loads only records where `TxnDT > LastLoadDate`, and **Script Activity** updates the watermark after successful execution.

This ensures only new records are processed and prevents duplicate data loading.
