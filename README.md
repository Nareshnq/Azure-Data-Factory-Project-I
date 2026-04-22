# Azure Data Factory — Project 1: Foundation

This is my first hands-on Azure Data Engineering project. The goal was simple — stop just reading theory and actually build something. So I set up a real Azure environment, created pipelines from scratch, broke things, fixed them, and documented everything along the way.

The project covers the full ADF pipeline lifecycle: connecting to data sources, moving files, filtering with conditions, looping through folders, transforming data with Data Flows, and finally scheduling pipelines to run automatically. Everything you see here was built and tested live in Azure.

> 📁 **Note:** The full project PDF (`Azure_Data_Factory_Project_1.pdf`) and all data files (`Fact_Sales_1.csv`, `Fact_Sales_2.csv`, `Bank_txns1.xlsx`, `Bank_txns2.xlsx`, etc.) will be added to this repository shortly in their respective folders.

---

## Table of Contents

1. [What I Built — Quick Summary](#what-i-built--quick-summary)
2. [How the Architecture Flows](#how-the-architecture-flows)
3. [Project Setup](#project-setup)
4. [Pipeline 1 — Copy Activity: ADLS to ADLS](#pipeline-1--copy-activity-adls-to-adls)
5. [Pipeline 2 — Copy Activity: GitHub to ADLS](#pipeline-2--copy-activity-github-to-adls)
6. [Pipeline 3 — Get Metadata Activity](#pipeline-3--get-metadata-activity)
7. [Pipeline 4 — Metadata + ForEach + If Condition + Copy](#pipeline-4--metadata--foreach--if-condition--copy)
8. [Pipeline 4 Clone — Adding Data Flow + Scheduled Trigger](#pipeline-4-clone--adding-data-flow--scheduled-trigger)
9. [ADF Core Concepts (Quick Reference)](#adf-core-concepts-quick-reference)
10. [Things I Learned / Got Confused By](#things-i-learned--got-confused-by)
11. [Files in This Repo](#files-in-this-repo)

---

## What I Built — Quick Summary

| Pipeline | What it does |
|---|---|
| **Pipeline 1** | Copies a CSV file from one ADLS folder to another using Copy Activity |
| **Pipeline 2** | Pulls a CSV file directly from a GitHub repo via HTTP connector and lands it in ADLS |
| **Pipeline 3** | Uses Get Metadata to inspect what files are sitting in a folder |
| **Pipeline 4** | Loops through all files in a folder, checks if the name starts with "Fact", and copies only those |
| **Pipeline 4 Clone** | Same as above but adds a Data Flow transformation step and a scheduled trigger |

---

## How the Architecture Flows

```
External Sources (GitHub, on-prem DB, etc.)
            │
            ▼
   Azure Data Factory ── orchestrates everything, stores nothing
            │
            ▼
   ADLS Gen2 ── source container (raw files land here)
            │
            ▼
   ADF Copy Activity ── moves files as-is, no transformation
            │
            ▼
   ADLS Gen2 ── destination / reporting container
            │
            ▼
   ADF Data Flow (Spark) ── transformations happen here
            │
            ▼
   ADLS Gen2 ── dataflowsOutput folder (clean, transformed data)
```

One thing that clicked early on — **ADF doesn't store any data itself.** It's purely an orchestration tool. It reaches into sources, moves or transforms data, and writes it somewhere else. All the actual data lives in ADLS.

---

## Project Setup

Before building any pipeline, there are a few one-time setup steps.

### Resources Created

- **Resource Group:** `ADF_RG` — everything lives under this
- **ADLS Storage Account:** `AdfCluster01` — used LRS redundancy to keep costs low
- **Azure Data Factory:** `azure0adf0storage`

### Containers and Folders in ADLS

```
AdfCluster01 (Storage Account)
├── source/
│   └── csv_files/          ← raw files uploaded or pulled from external sources
├── destination/
│   └── csv_files/          ← where Copy Activity lands the data
└── reportingOdept/
    ├── csv_files/           ← filtered Fact files land here (Pipeline 4)
    └── dataflowsOutput/     ← transformed output from Data Flow
```

### Linked Service Setup

The first thing you do in ADF before building any pipeline is create a **Linked Service** — this is basically ADF's way of knowing how to connect to a data source.

- **Name:** `LinkServiceDL`
- **Type:** Azure Data Lake Storage Gen2
- **Authentication:** Account key, pointing to `AdfCluster01`
- Always hit **Test Connection** before saving — saves a lot of debugging later

Once the linked service is in place, ADF is connected to ADLS and you're ready to start building.

---

## Pipeline 1 — Copy Activity: ADLS to ADLS

**Pipeline name:** `Pipeline1` | **Activity name:** `Copy CSV`

This was the first pipeline — the simplest use case. Take a file sitting in the source container and copy it over to the destination container.

```
source/csv_files/Fact_Sales_1.csv  →  ADF  →  destination/csv_files/
```

### How it was built

1. Opened **Author** tab in ADF Studio → created a new pipeline
2. Dragged **Copy Data** activity onto the canvas
3. **Source tab** — selected ADLS Gen2, Delimited Text format, linked it to `LinkServiceDL`, pointed to `Fact_Sales_1.csv`
4. **Sink tab** — same linked service, pointed to the destination container's `csv_files` folder
5. Hit **Debug** to test run

It worked on the first try. The file showed up in the destination container. Simple, but it's the foundation everything else builds on.

### What Copy Activity can actually connect to

This is what surprised me — Copy Activity isn't just for ADLS to ADLS. It can pull from basically anything:

```
Databases    → SQL Server, Azure SQL, MySQL, PostgreSQL, Oracle, Snowflake, SAP
Cloud        → Amazon S3, Google Cloud Storage, Salesforce, Dynamics 365
Files        → SFTP, FTP, On-premises file shares (via Self-Hosted IR), HDFS
APIs         → REST APIs, HTTP endpoints, GitHub raw URLs
Formats      → CSV, JSON, XML, Parquet, ORC, Excel, Binary
```

---

## Pipeline 2 — Copy Activity: GitHub to ADLS

**Pipeline name:** `pipeline2with_Git` | **Activity name:** `Copy Git`

This one was more interesting. Instead of copying from within Azure, the source is a raw file sitting in a **GitHub repository** — pulled using an HTTP connector.

```
GitHub raw URL  →  HTTP Linked Service  →  ADF  →  destination/csv_files/
```

### How it was built

1. Created a new pipeline and dragged a **Copy Data** activity
2. **Source tab** → `+New` → searched for `HTTP` as the data store → selected `DelimitedText`
3. Created a new Linked Service called `Linked_Service_Git`:
   - **Base URL:** `https://raw.githubusercontent.com`
   - **Authentication:** Anonymous
   - Tested connection ✅
4. **Relative URL:** `Nareshnq/Azure-Data-Factory-Project-I/refs/heads/main/Raw_files/Fact_Sales_2.csv`
5. Sink was the same destination folder as Pipeline 1
6. Hit **Debug** → success

The file landed in the destination container. One thing I noticed — because I didn't explicitly name the destination folder, ADF created a subfolder that mirrored the full GitHub path structure. Lesson learned: always set the destination folder name explicitly in the Sink tab.

---

## Pipeline 3 — Get Metadata Activity

**Pipeline name:** `P3_Get_Meta_Data`

Before you can loop through files or conditionally copy them, you need to know what's actually in a folder. That's exactly what the **Get Metadata** activity does — it inspects a folder and tells you what's inside.

It doesn't touch the data. It just reads properties:
- What files are in this folder? (`childItems`)
- Does this file exist? (`exists`)
- When was it last modified? (`lastModified`)
- How big is it? (`size`)

A good way to think about it: *"Tell me about this file"* not *"Give me the data in this file."*

### What the output looked like

I pointed it at the `destination/csv_files` folder (which had 3 files in it at the time) and got this back:

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

That list becomes the input for the next pipeline.

### The pattern that shows up everywhere

```
Get Metadata → ForEach (loop through list) → Copy / Transform each item
```

You don't always need ForEach — sometimes you just want to check if a single file exists before copying it. But the combo of Metadata + ForEach + If Condition is the most common dynamic file processing pattern in ADF.

---

## Pipeline 4 — Metadata + ForEach + If Condition + Copy

**Pipeline name:** `pipeline4_Get_Meta_data_Adv`

This is where things got properly interesting. The goal: loop through everything in a folder, check if the file name starts with "Fact", and only copy those files. Everything else gets skipped.

```
Get Metadata (list all files)
        ↓
ForEach (visit each file one by one)
        ↓
If Condition — does name start with "Fact"?
   ├── True  → Copy file to reportingOdept ✅
   └── False → Skip, do nothing
```

### A question I had — why do we need ForEach if we already have If Condition?

They do completely different things and can't replace each other:

- **ForEach** iterates — it visits each file in the list one by one
- **If Condition** evaluates — it checks true/false on a single item

You need ForEach to loop, and If Condition to filter. Without ForEach, If Condition has nothing to evaluate. Without If Condition, ForEach would try to copy everything.

### The actual pipeline steps

| Step | Activity | What it does |
|---|---|---|
| 1 | **Get Metadata** | Points to source folder → fetches `childItems` (list of all files) |
| 2 | **ForEach** | `Items = @activity('GetMetadata1').output.childItems` — loops through the list |
| 3 | **If Condition** (inside ForEach) | `@startsWith(item().name, 'Fact')` — only True branch runs |
| 4 | **Copy Data** (True branch) | Copies the matched file from `param_ds_source` to `reporting_sink` |

The False branch was left empty — non-matching files are simply skipped.

### Result

Only the `Fact_Sales_1.csv` and `Fact_Sales_2.csv` files were copied into `reportingOdept`. The bank transaction files were skipped exactly as expected.

I also ran a variation called `pipeline4_1` to move the `Bank_txns1.xlsx` and `Bank_txns2.xlsx` files to a `bank0dept` container using the same pattern — just with a different name prefix condition.

---

## Pipeline 4 Clone — Adding Data Flow + Scheduled Trigger

**Pipeline name:** `pipeline4_Get_Meta_data_Adv_copy1`

Up to this point, every pipeline was just moving raw files without touching them. This pipeline adds a **Data Flow** activity at the end — which is where actual transformation happens.

I cloned Pipeline 4 so the file copying logic was already in place, then added a Data Flow step after all the files had landed in the reporting folder.

### Copy Activity vs Data Flow — what's the difference?

| | Copy Activity | Data Flow |
|---|---|---|
| Purpose | Move raw files as-is | Transform and clean data |
| Transformations | None | Full ETL logic |
| Performance | Very fast | Runs on Spark — first run takes ~1-2 min to warm up |
| Use case | Bronze/raw layer | Silver/gold layer |

### What the Data Flow (dataflow1) actually did

The bank transaction data that had landed in the reporting folder was processed through a series of transformation steps:

```
source1 → select1 → filter1 → Conditional Split (VISA / Master / amex)
                                      ↓
                              aggregate1 → alterRow1 → sink
                                      ↓
                              derivedColumn (new columns)
```

Step by step:
- **select1** — renamed columns to clean names (`transaction_id`, `transactional_date`)
- **filter1** — filtered rows based on `customer_id` expressions
- **Conditional Split** — split the dataset into three streams by payment type: VISA, Master, amex
- **aggregate1** — grouped by `customer_id`, calculated totals (13 output rows from 343 source rows)
- **derivedColumn** — created a combined column using `transactional_date`, `product_id`, and `customer_id`
- **alterRow1** — set row-level insert/update policies
- **sink** — wrote the transformed output to `reportingOdept/dataflowsOutput/`

Data Flow reads the entire folder as one combined dataset — similar to how Spark reads a directory:
```python
# What's happening under the hood
df = spark.read.csv("reporting_sink/")  # all files in the folder, read at once
```

### Scheduled Trigger

After validating the pipeline, I added a **scheduled trigger** to run it automatically. Both pipeline runs showed up in the Monitor tab as Succeeded with the trigger listed as `trigger1_selected_files`. The `dataflowsOutput` folder was created with the transformed data inside.

---

## ADF Core Concepts (Quick Reference)

### The five building blocks of ADF

| Component | What it is | Think of it as |
|---|---|---|
| **Linked Service** | Connection string to a data source | A saved login to reach a system |
| **Dataset** | Pointer to the data structure and location | "The Sales table in that database" |
| **Activity** | A single action (Copy, DataFlow, Metadata, etc.) | One step in a recipe |
| **Pipeline** | A group of activities with sequencing logic | The full recipe |
| **Trigger** | What kicks the pipeline off | The alarm clock |

### ADF Studio tabs — what each one does

| Tab | Purpose |
|---|---|
| **Manage** | Set up linked services, integration runtimes, credentials |
| **Author** | Build and edit pipelines |
| **Monitor** | See run history, debug failures, check execution times |

---

## Things I Learned / Got Confused By

**Metadata activity isn't just for ForEach loops.** I initially thought they always go together. They don't — Get Metadata can also be used on its own to check if a file exists before doing anything with it.

**Always name your destination folder explicitly.** In Pipeline 2, I skipped this and ADF created a deeply nested subfolder path mirroring the GitHub URL structure. Not wrong, just messy.

**Data Flow has a cold start.** The first time a Data Flow runs it needs to spin up a Spark cluster — expect a 1-2 minute wait. This is normal.

**Azure Table Storage is not the same as ADLS containers.** Tables are for pipeline metadata (watermarks, run status, row counts). Actual data always goes into containers.

**ADF doesn't transform, it orchestrates.** Heavy transformations belong in Data Flow (Spark) or Databricks. ADF's job is to coordinate and move — not to do the heavy lifting itself.

---

## Files in This Repo

> All files below will be added to this repository shortly. The PDF has full theory notes, pipeline walkthroughs, and screenshots from every step of this project.

| File | Description |
|---|---|
| `Azure_Data_Factory_Project_1.pdf` | *(coming soon)* Full project notes — theory, pipeline steps, screenshots |
| `Raw_files/Fact_Sales_1.csv` | *(coming soon)* Sales dataset — uploaded directly to ADLS source container |
| `Raw_files/Fact_Sales_2.csv` | *(coming soon)* Sales dataset — pulled from GitHub via HTTP connector |
| `Raw_files/Bank_transaction_small_dataset.xlsx` | *(coming soon)* Bank transactions — used in metadata pipeline tests |
| `Raw_files/Bank_txns1.xlsx` | *(coming soon)* Bank transactions batch 1 — moved via pipeline4_1 |
| `Raw_files/Bank_txns2.xlsx` | *(coming soon)* Bank transactions batch 2 — moved via pipeline4_1 |
| `Raw_files/bank_transactions_data_2_augmented_clean_2.csv` | *(coming soon)* Augmented clean bank transaction dataset used in Data Flow |
| `ADF_Copy_Activity_Notes.docx` | *(coming soon)* Reference notes on Copy Activity source/destination options |

---

*Built by Nareshnq as part of an Azure Data Engineering learning path.*  
*Project 1 of an ongoing series — more projects to follow.*
