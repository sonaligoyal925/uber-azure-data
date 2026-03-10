# Uber Data Engineering Pipeline

This repository contains a prototype end-to-end real time data engineering pipeline for Uber ride events, designed to demonstrate how source data can be ingested from Github into Azure, transformed in Databricks using Spark streaming patterns, and materialized into insightful silver-layer and dimensional models.

---

## Project structure

- **`api.py` / `connection.py` / `data.py` / `main.py`**
  - Core orchestration + test-data generation for the streaming pipeline.
  - `data.py` generates synthetic Uber ride events (passenger/driver/vehicle/payment/city data + pricing logic).
  - `connection.py` sends generated ride events into Azure Event Hubs (using `.env` variables for connection strings).
  - `api.py` provides a simple FastAPI web app that lets you trigger ride generation and send events via an HTTP endpoint.

- **`files_array.json`**
  - Example configuration or manifest used by ingestion/transform scripts.

- **`Code_Files/`**
  - The core data pipeline assets (ingest/transformation scripts, SQL definitions, notebooks, and modeling).

- **`Data/`**
  - Reference lookup/mapping files (e.g. `map_*` JSON files used to enrich IDs into labels).

- **`templates/`**
  - HTML templates used by the web UI in `api.py`.

---

##  Pipeline intent / data flow (end-to-end)

1. **Source data lives in Git**
   - Raw exports (JSON/CSV) are stored in source control.

2. **Azure Data Factory pipeline ingests the data**
   - ADF copies the source files from Git into Azure storage (e.g., ADLS Gen2) or directly into Databricks bronze tables.

3. **Databricks / Spark reads the bronze data**
   - `bronze_adls.ipynb` (or `ingest.py`) is used to register bronze tables (e.g., `bulk_rides`, `rides_raw`).

4. **Silver layer is built using Spark streaming + SQL**
   - `silver.py` combines bulk load + streaming load into `stg_rides`.
   - `silver_obt.sql` enriches `stg_rides` by joining with the mapping tables to produce `silver_obt`.

5. **Dimensional modeling / fact table creation**
   - `model.py` defines CDC-powered dimension tables (`dim_*`) and a star-schema fact table (`fact`) using the `silver_obt` stream.

---

## Core components 

### `data.py`
- Generates synthetic Uber ride confirmation events (including passenger, driver, vehicle, location, pricing, and status fields).
- Includes in-code mapping tables (vehicle makes/types, payment methods, ride statuses, cities, cancellation reasons) used downstream to enrich analytics.

### `connection.py`
- Reads Event Hub configuration from `.env` (`CONNECTION_STRING`, `EVENT_HUBNAME`).
- Sends JSON ride events into Azure Event Hubs using the official Event Hubs Python client.
- Provides a standalone script mode for testing event production.

### `api.py`
- A FastAPI web application that exposes two endpoints:
  - `/` renders a booking homepage (`templates/home.html`).
  - `/book` generates a new ride event and sends it to Event Hubs, then renders a confirmation page (`templates/confirmation.html`).
- Useful for generating test traffic into the streaming ingestion pipeline.

### `Code_Files/ingest.py`
- Uses Spark Structured Streaming to read from Azure Event Hubs (Kafka-compatible) into a streaming table called `rides_raw`.
- Converts the Kafka `value` payload into a JSON string, enabling downstream parsing.

### `Code_Files/silver.py`
- Defines the schema for ride records (`rides_schema`).
- Creates a staging streaming table `stg_rides`.
- Implements two append flows:
  - `rides_bulk()` reads from a bulk/historic table (`bulk_rides`).
  - `rides_stream()` reads from `rides_raw`, parses the JSON payload, and emits structured columns.

### `Code_Files/silver_obt.sql`
- Creates a streaming table `silver_obt` by joining `stg_rides` with mapping tables (`map_*`).
- Fully enriches IDs into human-readable attributes (vehicle make/type, payment method, ride status, city info, cancellation reason).
- Applies a 3-minute watermark on `booking_timestamp` to handle late arrivals.

### `Code_Files/model.py`
- Defines views and CDC flows to create dimension tables (`dim_passenger`, `dim_driver`, etc.).
- Builds a `fact` table for analytical reporting.

### `Code_Files/bronze_adls.ipynb`
- Notebook for loading and validating bronze data (the initial ingest stage).

### `Code_Files/silver_obt.ipynb`
- Notebook for inspecting and validating the silver-layer output.

---

##  How to run 

1. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```

2. Ensure your environment has access to:
   - Azure storage / Databricks metastore
   - Event Hubs (if streaming from it)
   - Any secrets used by the pipeline (connection strings, keys)

3. Execute the bronze ingestion steps (from ADF or notebooks) to populate the bronze tables (`bulk_rides`, `rides_raw`).

4. Run the pipeline components:
   - `python Code_Files/ingest.py` (registers `rides_raw` stream)
   - `python Code_Files/silver.py` (builds `stg_rides` from bulk + stream)
   - Run the SQL in `Code_Files/silver_obt.sql` (creates/refreshes `silver_obt`)
   - `python Code_Files/model.py` (creates dim/fact models)

5. Optionally, explore results:
   - Use `Code_Files/bronze_adls.ipynb` and `Code_Files/silver_obt.ipynb` for interactive validation.

---
