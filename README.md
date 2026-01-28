# Databricks AutoLoader
# Databricks AutoLoader Pipelines (Bronze → Silver → Gold)

This repository demonstrates how to ingest files using Databricks Auto Loader (cloudFiles) into a medallion architecture:
- Bronze: raw ingestion (Delta)
- Silver: cleaned / typed / deduplicated records
- Gold: aggregates and curated datasets for BI / reporting

The examples are implemented as Databricks notebooks:
- `SETUP.ipynb` — volume / schema creation and environment setup
- `Static Autoloader BronzeLayer.ipynb` — Auto Loader ingestion into Bronze and examples of writing Delta
(See notebooks for full code and sample data outputs.)

## Contents / Directory layout (as used in the notebooks)
- /Volumes/workspace/bronze/bronzevolume/...  — Bronze Delta paths and checkpoints
- /Volumes/workspace/silver/silvervolume/...  — Silver Delta paths and checkpoints
- /Volumes/workspace/gold/goldvolume/...      — Gold Delta paths and checkpoints

Adjust these paths to match your workspace or mount points.

## Prerequisites
- Databricks workspace (cluster with Spark 3.x and Delta Lake)
- Access to a cloud object store (AWS S3 / Azure Blob / ADLS) or local workspace volumes used in notebooks
- Cluster configured with sufficient driver/executor memory for streaming and Delta writes
- Databricks Runtime that supports Auto Loader (DBR 7.x+ recommended)

## Setup (from SETUP.ipynb)
The notebooks create volumes/schemas used by pipelines. Example SQL used in SETUP notebook:
```sql
create volume workspace.bronze.bronzevolume;
create volume workspace.silver.silvervolume;
create volume workspace.gold.goldvolume;
```
Make sure to create corresponding storage or use existing mounted volumes before running pipelines.

## Bronze: Autoloader ingestion (example)
The Bronze notebook uses Auto Loader (cloudFiles) to read CSV files and write Delta with a checkpoint. Key pattern used:

```python
df = spark.readStream.format("cloudFiles") \
    .option("cloudFiles.format", "csv") \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .load("<input-path>")  # e.g., a mount or S3 path

# write to Delta (append mode) with checkpoint and path
df.writeStream.format("delta") \
    .outputMode("append") \
    .trigger(once=True) \
    .option("checkpointLocation", "/Volumes/workspace/bronze/bronzevolume/bookings/checkpoint") \
    .option("path", "/Volumes/workspace/bronze/bronzevolume/bookings/Data") \
    .start()
```

Notes:
- The example uses `trigger(once=True)` for a static / batch style incremental run. For continuous streaming, use a processing trigger or no trigger param.
- Provide a unique `checkpointLocation` per stream to maintain progress and avoid reprocessing.
- The notebook used `cloudFiles.format="csv"` and basic CSV options; adapt to JSON/Parquet as required.

### Useful Auto Loader options (recommendations)
- `cloudFiles.schemaLocation` — path to save inferred schema information (required for schema inference/evolution in production).
- `cloudFiles.maxFilesPerTrigger` — throttling ingestion rate for large directories.
- `cloudFiles.inferColumnTypes` (if using auto type inference) or supply explicit schema for stability.
- `cloudFiles.useNotifications=true` (for efficient event-based ingestion on supported storage).

## Silver: Cleaning and transformations
Typical steps to build Silver from Bronze:
1. Read Bronze Delta table or path as batch / streaming.
2. Cast columns to proper types and apply validation rules.
3. Deduplicate using a natural key and `row_number()` over event-time or ingestion-time, or use Delta Merge for idempotency.
4. Write out to Silver Delta with checkpointing (if streaming) or atomic writes (if batch).

Example (conceptual):
```python
bronze_df = spark.read.format("delta").load("/Volumes/workspace/bronze/bronzevolume/bookings/Data")

# transform & clean
from pyspark.sql.functions import col, to_timestamp, row_number
from pyspark.sql.window import Window

silver_df = (bronze_df
    .withColumn("event_ts", to_timestamp(col("date_col"), "yyyy-MM-dd"))
    .filter("amount IS NOT NULL")
    # dedup sample
    .withColumn("rn", row_number().over(Window.partitionBy("booking_id").orderBy(col("event_ts").desc())))
    .filter("rn = 1")
    .drop("rn")
)

silver_df.write.format("delta").mode("overwrite").save("/Volumes/workspace/silver/silvervolume/bookings/Data")
```

Consider using Delta `MERGE` to incrementally apply updates from bronze to silver safely.

## Gold: Aggregations / Curated datasets
Gold datasets are curated and possibly aggregated for downstream consumption (BI, ML). Example tasks:
- Time-series aggregates (daily bookings, revenue)
- Customer-level rolling metrics
- Lookup / dimension tables joined with facts

Write gold outputs to `/Volumes/workspace/gold/goldvolume/...` in Delta format.

## Checkpointing and Idempotency
- Always provide a distinct checkpoint location per streaming query.
- Use Delta's ACID guarantees and `MERGE` for idempotent upserts.
- For batch runs driven by Autoloader with `trigger(once=True)`, checkpoint still prevents re-ingestion on resumed runs.

## Schema evolution and management
- Prefer explicit schema where feasible to avoid surprises from `inferSchema`.
- When using Auto Loader's schema inference, set `cloudFiles.schemaLocation` to persist evolving schema.
- Use Delta `ALTER TABLE ... SET TBLPROPERTIES ('delta.enableChangeDataFeed' = 'true')` if CDC is required (DBR + Delta version permitting).

## Running locally vs Databricks workspace
- These notebooks target Databricks notebooks and workspace volumes; replace paths with cloud storage mounts or S3/ADLS paths when running outside the workspace.
- If running locally with spark, install Delta Lake and configure appropriate cloud access credentials.

## Troubleshooting
- "Files reprocessed" — check checkpoint locations, ensure each stream uses a unique checkpoint path.
- "Schema inference inconsistent" — provide explicit schema or use `cloudFiles.schemaLocation` and manage schema changes deliberately.
- Performance issues — tune `maxFilesPerTrigger`, partitioning, and cluster sizing.

## Where to look in this repo
- `SETUP.ipynb` — volume/schema creation SQL and environment setup
- `Static Autoloader BronzeLayer.ipynb` — complete example of Auto Loader readStream + writeStream to Bronze (Delta), sample data previews and queries

