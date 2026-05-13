# CDC Incremental Load using Azure Data Factory (ADF)

## Project Overview

This project implements **Change Data Capture (CDC)** / **Incremental Data Load** using **Azure Data Factory (ADF)**.

The objective is to load **only new records** from a source SQL table into daily CSV files, avoiding duplicate data loads.

This is achieved using a **Watermark Table** that stores the **last successful load date**.

---

# Architecture

```text
Source SQL Table
      |
      v
Lookup Watermark Table (LastLoadDate)
      |
      v
Copy Activity (Load only new records)
      |
      v
Target CSV File (Daily Output)
      |
      v
Update Watermark Table
```

---

# Source Table

## Create Source Table

```sql
CREATE TABLE SourceTransactions (
    TxnID INT,
    TxnDT DATE,
    TxnAmount DECIMAL(10,2),
    ProductCD VARCHAR(10),
    ProductQty VARCHAR(50)
);
```

## Insert Sample Data

```sql
INSERT INTO SourceTransactions VALUES
(123,'2025-07-21',200,'BV','Cocacola'),
(124,'2025-07-21',1100,'CM','Lotion'),
(125,'2025-07-22',500,'ET','Biscuits'),
(126,'2025-07-22',200,'CH','Detergent');
```

---

# Watermark Table

The watermark table stores the **last successful load date**.

## Create Watermark Table

```sql
CREATE TABLE WatermarkTable (
    PipelineName VARCHAR(100),
    LastLoadDate DATE
);
```

## Insert Initial Watermark

```sql
INSERT INTO WatermarkTable
VALUES ('CDC_Pipeline','2025-07-21');
```

---

# Azure Data Factory Pipeline Steps

## Step 1: Create Linked Services

Create linked services for:

- Azure SQL Database (Source)
- Azure Blob Storage / ADLS (Target)

---

## Step 2: Create Datasets

Create datasets for:

- Source SQL Table (`SourceTransactions`)
- Watermark Table (`WatermarkTable`)
- Sink CSV File

---

## Step 3: Lookup Activity

Fetch the last loaded date from the watermark table.

### Query

```sql
SELECT LastLoadDate
FROM WatermarkTable
WHERE PipelineName='CDC_Pipeline';
```

---

## Step 4: Copy Activity (Incremental Load)

Use dynamic SQL to load only new records.

### Query

```sql
SELECT *
FROM SourceTransactions
WHERE TxnDT > '@{activity('LookupWatermark').output.firstRow.LastLoadDate}'
```

---

## Step 5: Dynamic File Naming

Generate daily output files dynamically.

```text
TxnFile_@{formatDateTime(utcNow(),'ddMMMyyyy')}.csv
```

### Example Output

```text
TxnFile_22Jul2025.csv
```

---

## Step 6: Update Watermark

After successful load, update the watermark table.

```sql
UPDATE WatermarkTable
SET LastLoadDate = GETDATE()
WHERE PipelineName='CDC_Pipeline';
```

---

# Example Flow

### Day 1 Data

| TxnID | TxnDT | Amount |
|-------|-------|--------|
| 123 | 21-Jul-2025 | 200 |
| 124 | 21-Jul-2025 | 1100 |

Output:

```text
TxnFile_21Jul2025.csv
```

---

### Day 2 Data

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

# Benefits

- Incremental data loading
- Avoids duplicate records
- Improves pipeline performance
- Easy to maintain
- Production-ready CDC pattern

---

# Technologies Used

- Azure Data Factory (ADF)
- Azure SQL Database
- Azure Blob Storage / ADLS
- SQL

---

# Interview Explanation

Implemented a CDC (Change Data Capture) incremental load pipeline using Azure Data Factory.

Used a watermark table to track the last successful load date. ADF Lookup activity retrieves the watermark, and Copy Activity loads only records where `TxnDT > LastLoadDate`.

Daily output files are generated dynamically, and the watermark table is updated after successful execution to support future incremental loads.

CDC-ADF-Project/
│
├── README.md
├── sql/
│   ├── source_table.sql
│   ├── watermark_table.sql
│   └── sample_data.sql
└── adf_pipeline/
    └── pipeline_screenshots/# CDC1
