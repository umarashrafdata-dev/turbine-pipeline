# turbine-pipeline

A PoC data pipeline for Colibri Digital built on Azure Databricks,
ADLS Gen2 and Delta/Unity Catalog, using a medallion (bronze/silver/gold)
architecture with incremental loading.

## How to run
1. Prerequisites: Databricks ws, ADLS Gen2 with managed-identity external locations configured and CSVs uploaded /turbine/landing/
2. Run `setup/Create_SQL` to create catalog, schemas and relevant tables
3. Run `setup/Corruption_Notebook` to generate missing and corrupt data
4. Run notebooks in order: `Bronze`, `Silver` and `Gold`
5. To re-demo: run:

DELETE FROM turbines.control.watermarks;
DELETE FROM turbines.control.pipeline_log;
DROP TABLE IF EXISTS turbines.silver.readings;
DROP TABLE IF EXISTS turbines.silver.quarantine;

## Design

**Bronze** — reads all landing CSVs with an explicit schema and writes
them as-is to `bronze.raw` with lineage columns (file_name and ingest_ts).
Full overwrite each run because I wanted to mirror the source file.

**Silver** — the incremental layer. Each run it: reads the watermark
(None on first run, meaning full load), filters bronze to rows newer than previous watermark value (max timestamp from the data),
dedupes on timestamp and turbine_id keeping the most recent, quarantines readings outside of plausible reading assuming max is 4.5 (0 and 4.5) to
`silver.quarantine` with a reason, imputes null power_output using per turbine mean after quarantine,
and MERGEs into `silver.readings` on turbine_id and timestamp so re-runs are idempotent.

**Gold** — per-turbine daily min/max/avg plus an anomaly flag: a
turbine-day is flagged when its average is outside mean +- 2*std for that date.

**Control** — `watermarks`, `pipeline_log`, `error_log` tables that hold max timestamps, record records in and out and any errors.

## Key decisions
- Cleaning uses physical bounds (0–4.5 MW) rather than mean ±2σ because the mean and std is affected by extreme outliers
- Imputation uses per-turbine mean computed after quarantine because turbines may behave differently
- Dedupe keeps latest by ingest_ts to mirror source system as best as possible
- The 2σ anomaly rule flags 5% of turbine-days by construction; I observed
  18 flags of which 1 was the injected fault, and in production I would classify outliers using each turbines history as opposed to looking as a whole.

## Assumptions
- Max power output of a turbine is 4.5Mw
- Some process exists to get CSVs into the landing zone
- Profiling showed the provided files are clean and complete, so a corruption script injects defects to demonstrate cleaning and anomaly detection.

## Productionising 
-  incremental loading at bronze, per source watermarks
-  run the layers by creating a Databricks Workflow/ orchestration notebook and alert on failures
-  no secrets exist here, any future credentials via Key Vault-backed secret scope
-  transformation helpers into Python modules 
