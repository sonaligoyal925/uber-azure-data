# Code_Files

This folder contains the core data pipeline assets used to ingest, clean, enrich, and model Uber ride events through a bronze → silver staging pattern.

The pipelines are built using **PySpark + the Spark Pipelines API** (`pyspark.pipelines`), leveraging streaming tables and CDC-style flows to continuously process events from Azure Event Hubs into analytic tables/ Databricks.

---

## Overall data flow 

1. **Source data is stored in Git** (JSON/CSV files or other exports).
2. An **Azure Data Factory (ADF) pipeline** ingests that source data into Azure storage (e.g., ADLS Gen2).
3. **`ingest.py`** (or the notebook `bronze_adls.ipynb`) is used to register/read the bronze tables into Spark streaming tables (e.g., `bulk_rides` / `rides_raw`).
4. **`silver.py`** reads both the bulk/historic load (`bulk_rides`) and/or the streaming load (`rides_raw`) and appends both into a staging stream table called `stg_rides`.
5. **`silver_obt.sql`** builds the silver-layer streaming view (`silver_obt`) by joining `stg_rides` against the mapping/dimension tables (`map_*`) to enrich IDs with human-readable names and attributes.
6. **`model.py`** defines CDC-backed dimensional tables (`dim_*`) and a fact table (`fact`) based on the `silver_obt` stream.

---

## What's in this folder

- **`ingest.py`**
  - Reads from Azure Event Hubs (Kafka-compatible) using Spark Structured Streaming.
  - Converts the `value` payload into a JSON string column (`rides`).
  - Produces a streaming table named `rides_raw` that downstream flows can consume.

- **`silver.py`**
  - Defines the expected ride schema (`rides_schema`) used to parse the JSON payload.
  - Creates a streaming staging table `stg_rides`.
  - Implements two append flows:
    - `rides_bulk()` reads a bulk snapshot table `bulk_rides` (for initial/load-backfill).
    - `rides_stream()` reads from `rides_raw`, parses JSON via `from_json(.., rides_schema)`, and selects exploded ride fields.

- **`silver_obt.sql`**
  - Builds (or refreshes) a streaming silver table called `silver_obt`.
  - Joins the staging `stg_rides` stream against several mapping tables (`map_vehicle_makes`, `map_vehicle_types`, `map_ride_statuses`, `map_payment_methods`, `map_cities`, `map_cancellation_reasons`) to enrich numeric IDs with their descriptive attributes.
  - Uses a 3-minute watermark on `booking_timestamp` to allow for late-arriving events.

- **`model.py`**
  - Defines dimensional views and CDC pipelines using `dp.create_auto_cdc_flow`.
  - Builds `dim_passenger`, `dim_driver`, `dim_vehicle`, `dim_payment`, `dim_booking`, `dim_location`, and a `fact` table.
  - Each dimension/fact is derived from `uber.bronze.silver_obt` (the silver streaming table) and uses natural keys plus CDC sequencing.

- **`bronze_adls.ipynb`**
  - Notebook to explore the raw (bronze) input data and optionally write it into ADLS / another bronze storage location.

- **`silver_obt.ipynb`**
  - Notebook to run and validate the silver-layer query results; useful for QA and ad-hoc analysis.

---
