# ☁️ Azure Data Factory — Project 1: Foundation

> **End-to-end data engineering project** covering ADF pipelines, ADLS storage, Copy Activity, Metadata Activity, Data Flows, and scheduled triggers.

---

## 📑 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Big Data & Core Concepts](#big-data--core-concepts)
3. [Data Storage Layers Explained](#data-storage-layers-explained)
4. [ADLS Storage Account Components](#adls-storage-account-components)
5. [Azure Data Factory — Core Components](#azure-data-factory--core-components)
6. [Project Setup](#project-setup)
7. [Pipeline 1 — Copy Activity (ADLS to ADLS)](#pipeline-1--copy-activity-adls-to-adls)
8. [Pipeline 2 — Copy Activity (GitHub to ADLS)](#pipeline-2--copy-activity-github-to-adls)
9. [Pipeline 3 — Get Metadata Activity](#pipeline-3--get-metadata-activity)
10. [Pipeline 4 — Metadata + ForEach + If Condition + Copy](#pipeline-4--metadata--foreach--if-condition--copy)
11. [Pipeline 4 Clone — Data Flow Activity with Trigger](#pipeline-4-clone--data-flow-activity-with-trigger)
12. [Key Interview Answers](#key-interview-answers)
13. [Files in This Repository](#files-in-this-repository)

---

## Architecture Overview

```
On-Prem / External Sources
        │
        ▼
  Azure Data Factory (ADF)  ← Orchestration Layer
        │
        ▼
  ADLS Gen2 (Source)        ← Raw / Bronze Layer
        │
        ▼
  ADF Data Flow (Spark)     ← Transformation Layer
        │
        ▼
  ADLS Gen2 (Destination /  ← Processed / Silver / Gold Layer
   Reporting)
        │
        ▼
  BI Tools / Synapse / Power BI
```

**Key principle:** ADF does NOT store data. It **orchestrates** the movement and transformation of data between systems.

---

## Big Data & Core Concepts

### Database Engines

| Engine | Company | Management Tool |
|---|---|---|
| Microsoft SQL Server | Microsoft | SSMS / Azure Data Studio |
| MySQL | Oracle | MySQL Workbench |
| PostgreSQL | PostgreSQL GDG | pgAdmin |
| Oracle DB | Oracle | Oracle SQL Developer |
| Multiple | Open Source | DBeaver / DataGrip |

### Processing Types

| Type | Description |
|---|---|
| **ETL** | Extract → Transform → Load (transform before loading) |
| **ELT** | Extract → Load → Transform (transform inside target system) |
| **Batch** | Process large datasets at intervals |
| **Stream / Real-time** | Process data as it arrives |
| **Apache Spark** | Fast, in-memory distributed processing engine |

---

## Data Storage Layers Explained

> **Common confusion:** "If Azure SQL and Cosmos DB store data, why do we need a Data Lake or Data Warehouse?"

They serve completely **different purposes**:

| Storage Type | Use Case | When to Use | When NOT to Use |
|---|---|---|---|
| **Azure SQL** | Operational DB (OLTP) | Live apps, real-time transactions | Historical analysis, huge volumes |
| **Cosmos DB** | Operational NoSQL | Mobile apps, global users, high-speed inserts | Complex JOINs, analytical queries |
| **Data Lake (ADLS)** | Raw data storage | Landing raw data, cheap long-term storage | Real-time queries, fast analytics |
| **Data Warehouse (Synapse/Snowflake)** | Analytical queries (OLAP) | BI dashboards, historical analysis | Real-time writes, raw unprocessed data |
| **Data Lakehouse (Delta/Iceberg)** | Hybrid raw + processed | Both OLTP & OLAP, ML pipelines | Simple transactional DB |

### Why Can't Azure SQL Do Everything?

| Question | Azure SQL | Data Warehouse |
|---|---|---|
| "What's the current balance?" | ✅ Instant (OLTP optimized) | ❌ Too slow (full scan) |
| "Show me 10 years of trends" | ❌ Terrible (row-oriented) | ✅ Fast (columnar, indexed) |
| "Process 50 TB of data" | ❌ Expensive | ✅ Cheap (cost-effective) |
| "10,000 concurrent users" | ✅ Yes (OLTP designed) | ❌ No (batch queries) |

**Interview answer:**
> *"Azure SQL and Cosmos DB are operational databases used to support applications. For analytics, we move data into a data lake for raw storage and then into a data warehouse or lakehouse where it's transformed, historical, and optimized for reporting, BI, and advanced analytics without impacting production systems."*

---

## ADLS Storage Account Components

### Containers (Blob / Data Lake)
- Store large-scale analytical data: CSV, Parquet, Delta files
- Foundation of ADLS Gen2 with folder hierarchies and access controls
- Used for **raw, processed, and curated datasets**
- Tools like ADF, Databricks, and Synapse read/write here

### File Shares
- Cloud-based shared file system (like a network drive)
- Used for lift-and-shift workloads and application data
- ❌ Not optimized for large-scale analytics

### Queues
- Store small messages for **asynchronous communication** between systems
- Trigger workflows, notify services when data arrives
- Used for **orchestration**, not data storage

### Tables (Azure Table Storage)
- NoSQL key-value store — **not part of the data lake itself**
- Used by ETL pipelines to store **metadata** such as:
  - Pipeline execution status (Success / Failed)
  - Load timestamps (last successful run)
  - Row counts (source vs target)
  - Watermark values for incremental loads

> ✅ **Interview answer:** *"Azure Table Storage is commonly used to store ETL metadata such as load status, watermarks, and row counts, while the actual data is stored in ADLS containers."*

---

## Azure Data Factory — Core Components

Think of ADF like a **plumbing system**:

| Component | Description | Analogy |
|---|---|---|
| **Linked Services** | Connection strings to data sources | Water pipe connections |
| **Datasets** | Structure and location of data | Pipe specifications |
| **Activities** | Actions performed on data (Copy, DataFlow, etc.) | Water valves |
| **Pipelines** | Workflow grouping multiple activities | The full plumbing layout |
| **Triggers** | When to run the pipeline (scheduled, event-based) | The on/off switch |

### ADF Studio Menu Workflow

```
Manage  → Set up linked services & IR
Author  → Build pipelines & transformations
Debug   → Test logic
Publish → Deploy changes
Monitor → Track execution & troubleshoot
```

---

## Project Setup

### Step 1 — Create Resource Group
- **Resource Group:** `ADF_RG`

### Step 2 — Create Two Resources
- **ADLS Storage Account:** `AdfCluster01`
  - Redundancy: `LRS` (cheapest, keeps cost low)
- **Azure Data Factory:** `azure0adf0storage`

### Step 3 — Create Containers & Folders in ADLS

| Container | Folder | Purpose |
|---|---|---|
| `source` | `csv_files` | Raw files land here (pull from DB or upload) |
| `destination` | `csv_files` | Copy activity writes here |
| `reportingOdept` | — | Reporting output (filtered/transformed files) |

### Step 4 — Create Linked Service
- **Name:** `LinkServiceDL`
- Type: Azure Data Lake Storage Gen2
- Connects ADF to ADLS
- ✅ Always test connection before saving

---

## Pipeline 1 — Copy Activity (ADLS to ADLS)

**Flow:** `ADLS source/csv_files` → ADF → `ADLS destination/csv_files`

**Activity name:** `Copy CSV`

### Steps
1. Go to **Author** tab → Create new pipeline named `Pipeline1`
2. Drag **Copy Data** activity onto canvas
3. **Source tab:** Select ADLS Gen2 → Delimited Text → `LinkServiceDL` → select file → Import schema: None
4. **Sink tab:** Same linked service → add `csv_files` directory in destination container
5. Click **Debug** to run

✅ **Result:** File successfully moved from source to destination container.

### Copy Activity — Source Flexibility

```
External Sources → ADLS (most common ingestion pattern)
  ├── Databases: SQL Server, Azure SQL, MySQL, PostgreSQL, Oracle, Snowflake
  ├── Cloud: Amazon S3, Google Cloud Storage, Salesforce, Dynamics 365
  ├── Files: SFTP, FTP, On-premises (via Self-Hosted IR), HDFS
  ├── Apps: SAP ECC, SAP BW, Jira, REST APIs
  └── Formats: CSV, JSON, XML, Parquet, ORC, Excel, Binary

ADLS → ADLS (same or different storage accounts)

ADLS → External Destinations (e.g., write to Synapse, Snowflake)
```

---

## Pipeline 2 — Copy Activity (GitHub to ADLS)

**Flow:** `GitHub Raw URL` → HTTP Connector → ADF → `ADLS destination`

**Pipeline name:** `pipeline2with_Git` | **Activity name:** `Copy Git`

### Steps
1. Create new pipeline → drag **Copy Data** activity
2. **Source tab:** Click `+New` → search `HTTP` as data store → `DelimitedText`
3. Create new Linked Service named `Linked_Service_Git`:
   - **Base URL:** `https://raw.githubusercontent.com`
   - **Authentication:** Anonymous → Test Connection ✅
4. **Relative URL:** `Nareshnq/Azure-Data-Factory-Project-I/refs/heads/main/Raw_files/Fact_Sales_2.csv`
5. **Sink tab:** Same destination as Pipeline 1
6. Click **Debug**

✅ **Result:** `Fact_Sales_2.csv` successfully pulled from GitHub and landed in destination.

> 💡 **Note:** If no destination folder name is specified, ADF creates a new subfolder mirroring the GitHub path. Always specify the destination folder name explicitly.

---

## Pipeline 3 — Get Metadata Activity

**Pipeline name:** `P3_Get_Meta_Data`

### What is Get Metadata?
Used to retrieve **information about** a file or folder — not the data inside it.

Properties it can retrieve:
- File name, size, last modified time
- Whether the file exists (`exists`)
- List of child items in a folder (`childItems`)

> Think of it as: *"Tell me about this file"* rather than *"Give me the data in this file"*

### Example Output (childItems)
```json
{
  "childItems": [
    { "name": "Bank_transaction_small_dataset.xlsx", "type": "File" },
    { "name": "Fact_Sales_1.csv", "type": "File" },
    { "name": "Fact_Sales_2.csv", "type": "File" },
    { "name": "Nareshnq", "type": "Folder" }
  ]
}
```

### Common Pattern
```
Metadata → (optional) ForEach → Action (Copy / Transform)
```

### Real-World Use Case — Check if file exists before copying
```
1. Get Metadata  → points to sales_data.csv, asks for "exists" property
2. If Condition  → @activity('GetMeta').output.exists == true
3. Copy Activity → (True branch) copy file to destination
4. Fail Activity → (False branch) log that file was missing
```

---

## Pipeline 4 — Metadata + ForEach + If Condition + Copy

**Pipeline name:** `pipeline4_Get_Meta_data_Adv`

**Goal:** Copy only files whose names start with `Fact` — skip all others.

### Why Both ForEach AND If Condition?

| Activity | Role |
|---|---|
| **ForEach** | Iterates over the list — visits each file one by one |
| **If Condition** | Evaluates true/false on a **single item** |

```
Get Metadata returns 4 files
        ↓
ForEach → sales_jan.csv   → If Condition → False → SKIP
ForEach → Fact_feb.csv    → If Condition → True  → COPY ✅
ForEach → Fact_mar.csv    → If Condition → True  → COPY ✅
ForEach → orders_apr.csv  → If Condition → False → SKIP
```

### Pipeline Steps

| Step | Activity | Configuration |
|---|---|---|
| 1 | **Get Metadata** | Points to source folder, fetches `childItems` |
| 2 | **ForEach** | Items = `@activity('GetMetadata1').output.childItems` |
| 3 | **If Condition** (inside ForEach) | `@startsWith(item().name, 'Fact')` → True branch runs Copy |
| 4 | **Copy Data** (True branch) | Copies matched file from `param_ds_source` to `reporting_sink` |

✅ **Result:** Only `Fact_*.csv` files copied to `reportingOdept` container. Bank transaction files skipped.

---

## Pipeline 4 Clone — Data Flow Activity with Trigger

**Pipeline name:** `pipeline4_Get_Meta_data_Adv_copy1`

### Copy Activity vs Data Flow Activity

| | Copy Activity | Data Flow Activity |
|---|---|---|
| **Purpose** | Move raw data as-is | Transform / clean data |
| **Transformations** | None | Full ETL logic |
| **Performance** | Very fast | Runs on Spark cluster |
| **Use case** | Land raw files | Business logic layer |

### How Data Flow Works
- Runs on a **Spark cluster** behind the scenes
- First run has ~1–2 min warmup (cluster spin-up)
- When source points to a **folder**, it reads all files as one combined dataset

```python
# Under the hood — equivalent Spark operation:
df = spark.read.csv("reporting_sink/")  # reads all files in folder
```

### Transformations Applied in dataflow1

```
source1 → select1 → filter1 → VISA branch   → aggregate1 → alterRow1 → sink
                             → Master branch
                             → amex branch   → derivedColumn
```

- **select1:** Renamed columns (`transaction_id`, `transactional_date`)
- **filter1:** Filtered rows using expressions on `customer_id`
- **Conditional Split:** Distributed data by payment type (VISA / Master / amex)
- **aggregate1:** Aggregated by `customer_id` — grouped totals
- **derivedColumn:** Updated `transactional_date` + combined `product_id` and `customer_id`
- **alterRow1:** Set row policies for insert/update
- **sink:** Exported to `dataflowsOutput` folder in `reportingOdept`

### Scheduled Trigger
- Trigger type: **Scheduled**
- Pipeline ran automatically at configured time
- ✅ Successfully created `dataflowsOutput` folder with transformed data

---

## Key Interview Answers

**Q: What is ADF?**
> *"Azure Data Factory is a cloud ETL/ELT orchestration service used to ingest, move, and schedule data between systems. It connects to many sources (databases, files, APIs) and coordinates pipelines, retries, and monitoring. ADF is great at data movement and workflow control, not heavy transformations."*

**Q: What is the difference between Copy Activity and Data Flow?**
> *"Copy Activity moves raw data as-is with no transformations — it's fast and ideal for landing files. Data Flow runs on Spark and supports full ETL logic like joins, aggregations, and derived columns — it's used for the transformation layer."*

**Q: Why use Get Metadata before Copy Activity?**
> *"Get Metadata retrieves file properties like existence, size, and child items before acting on data. It enables dynamic, conditional pipelines — for example, only copying files that exist or match a naming pattern."*

**Q: Why do we need both ForEach and If Condition?**
> *"ForEach iterates over a list visiting each item one by one, while If Condition evaluates a true/false check on a single item. You need ForEach to loop and If Condition to filter — they do completely different jobs."*

---

## Files in This Repository

| File | Description |
|---|---|
| `Fact_Sales_1.csv` | Sales data — uploaded directly from laptop to ADLS source |
| `Fact_Sales_2.csv` | Sales data — pulled from GitHub via HTTP connector |
| `Bank_transaction_small_dataset.xlsx` | Bank transactions — used in metadata/pipeline tests |
| `Bank_txns1.xlsx` | Bank transactions batch 1 — moved via pipeline4_1 |
| `Bank_txns2.xlsx` | Bank transactions batch 2 — moved via pipeline4_1 |
| `ADF_Copy_Activity_Notes.docx` | Copy Activity source/destination reference notes |
| `Azure_Data_Factory_Project_1.pdf` | Full project notes and theory |

---

*Project by Nareshnq | Azure Data Engineering — Foundation Project*
