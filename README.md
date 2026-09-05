# FHIR Medallion Pipeline

A data ingestion and analytics pipeline that fetches healthcare data from the public HAPI FHIR API and processes it through a Medallion Architecture (Raw → Bronze → Silver → Gold) using Microsoft Fabric and Spark.

## Overview

This project ingests four core FHIR resources — Patient, Encounter, Observation, and Condition — from the public FHIR API, with pagination support,and builds a layered lakehouse architecture with metadata tracking, deduplication, SCD Type 2 versioning, and reporting-ready Gold tables.

## Architecture

The pipeline follows a 4-layer Medallion Architecture:

1. **FHIR API** (`hapi.fhir.org`) — source of all data

2. **Raw Layer (Files)**
   - Raw JSON responses stored as-is, organized by resource and date
   - Path: `Files/raw/{resource_name}/{date}/page_N.json`

3. **Bronze Layer (Delta Tables)**
   - Parsed JSON with metadata columns (`extraction_timestamp`, `api_url_or_params`)
   - Tables: `bronze_patient`, `bronze_encounter`, `bronze_observation`, `bronze_condition`

4. **Silver Layer (Delta Tables)**
   - Cleaned, deduplicated data with key fields extracted into columns
   - Tables: `silver_patient`, `silver_encounter`, `silver_observation`, `silver_condition`

5. **Gold Layer & SCD Type 2 Layer** (built from Silver)
   - **Gold:** Reporting-ready view — `gold_patient_summary`
   - **SCD Type 2:** Historical version tracking — `scd_patient`, `scd_encounter`, `scd_observation`, `scd_condition`

## Data Flow / Orchestration

Data is ingested in the following order, as required:

Patient → Encounter → Observation → Condition

This order is maintained inside the notebook using a `RESOURCE_ORDER` list, and the entire notebook is triggered via a Fabric Data Pipeline (`fhir_ingestion_pipeline`) using a Notebook activity.

## Table Relationships

| Table | Key Field | Relationship |
|---|---|---|
| `silver_patient` | `resource_id` | Primary entity |
| `silver_encounter` | `patient_reference` | Links to `Patient/{resource_id}` |
| `silver_observation` | `patient_reference` | Links to `Patient/{resource_id}` |
| `silver_condition` | `patient_reference` | Links to `Patient/{resource_id}` |
| `gold_patient_summary` | `patient_reference` | Aggregates counts from Encounter, Observation, and Condition per patient |

## Metadata & Versioning

Every Bronze record includes:
- `extraction_timestamp` — when the data was fetched
- `api_url_or_params` — the exact API call used to fetch it

SCD Type 2 is implemented on top of the Silver layer:
- Each record's content is hashed (`record_hash`) to detect changes
- Unchanged records are left as-is
- Changed records are versioned: the old row is marked `is_current = False` with an `end_date`, and a new row is inserted with `is_current = True`
- This maintains a full history of how each record has changed over time

## Design Decisions

## Design Decisions

- **Incremental date-based fetching:** The pipeline uses a reusable `run_full_pipeline()` function that automatically determines the current day's time window and fetches only new/updated records using the FHIR `_lastUpdated` filter — no hardcoded dates.
- **Validating incremental behavior:** Since the public HAPI FHIR test server does not reliably isolate historical data by date (existing records often appear as "recently updated"), a small set of test records were created via the FHIR API across distinct time-windows to validate that date-based incremental fetching and SCD Type 2 change-detection work correctly end-to-end.
- **Single reusable pipeline entry point:** `run_full_pipeline()` orchestrates the entire flow (Bronze → Silver → SCD Type 2 → Gold) in the required resource order (Patient → Encounter → Observation → Condition), and is the single function triggered by the Fabric Data Pipeline — enabling this to run automatically on any future date without manual changes.

## How to Run

1. Open `notebooks/fhir_medallion_pipeline.ipynb` in Microsoft Fabric, attached to a Lakehouse.
2. Run all cells in order — this will:
   - Fetch data from the FHIR API (Raw + Bronze layers)
   - Clean and deduplicate into the Silver layer
   - Apply SCD Type 2 versioning
   - Build the Gold reporting table
3. Alternatively, trigger the entire process via the `fhir_ingestion_pipeline` Data Pipeline, which runs the notebook end-to-end.

## Tech Stack

- Microsoft Fabric (Lakehouse, Notebooks, Data Pipelines)
- PySpark
- Delta Lake
- Public HAPI FHIR R4 API

## Project Structure

```
fhir-medallion-pipeline/
├── README.md
├── notebooks/
│   └── fhir_medallion_pipeline.ipynb
└── pipeline/
    ├── README.md
    └── fhir_ingestion_pipeline.zip  (Fabric Data Pipeline export)
```
