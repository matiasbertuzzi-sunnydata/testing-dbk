# Open-Meteo Weather Data Engineering Project

## Overview

This project implements a Databricks Medallion architecture for ingesting and transforming weather forecast data from the Open-Meteo API.

The project is developed in a Databricks sandbox workspace using Unity Catalog and is structured into three data layers:

```text
Open-Meteo API
      │
      ▼
   BRONZE
      │
      ▼
   SILVER
      │
      ▼
    GOLD
```

The objective is to demonstrate a production-oriented Databricks data engineering workflow, including API ingestion, Delta Lake transformations, data quality validation, incremental processing, deduplication, and Git-based promotion between environments.

---

## Architecture

### Bronze

The Bronze layer stores the raw response received from the Open-Meteo API.

Responsibilities:

* Retrieve weather forecast data from Open-Meteo.
* Preserve the original API response.
* Add ingestion metadata.
* Store the raw payload in Delta format.
* Maintain source lineage.

Example metadata includes:

* `_bronze_ingestion_timestamp`
* `_source_url`
* `load_date`

Bronze acts as the raw landing layer and provides a reproducible source for downstream processing.

---

### Silver

The Silver layer transforms the raw Bronze data into a structured hourly weather dataset.

Responsibilities:

* Parse the raw API response.
* Normalize nested weather data.
* Produce one row per location and forecast hour.
* Apply schema and data-quality validations.
* Remove duplicate forecast records.
* Incrementally merge new or updated records into the Silver Delta table.

The resulting dataset is:

```text
silver_weather_hourly
```

The intended grain is:

```text
location_name + forecast_time
```

Silver therefore provides a clean, structured representation of the weather forecast suitable for analytical transformations.

---

### Gold

The Gold layer provides business-oriented weather aggregations derived from Silver.

The primary Gold dataset is:

```text
gold_weather_daily
```

Its intended grain is:

```text
location_name + weather_date
```

The daily aggregation includes metrics such as:

* Minimum temperature
* Maximum temperature
* Average temperature
* Average relative humidity
* Total precipitation
* Number of precipitation hours
* Dominant weather code
* Number of forecast hours

Gold is intended to provide a simplified dataset for downstream analytics, dashboards, reporting, and business consumption.

---

## Unity Catalog

The project uses Unity Catalog for data governance.

The sandbox implementation uses:

```text
Catalog: mbertuzzi
Schema:  weatherapi
```

Tables:

```text
mbertuzzi.weatherapi.bronze_weather_raw
mbertuzzi.weatherapi.silver_weather_hourly
mbertuzzi.weatherapi.gold_weather_daily
```

The same sandbox workspace and Unity Catalog are intentionally used throughout the current implementation.

A multi-workspace environment would normally be used for stronger environment isolation, but this project currently operates within the company's provided sandbox workspace.

---

## Delta Lake

All Medallion layers use Delta Lake tables.

The project uses Delta capabilities including:

* ACID transactions
* MERGE-based incremental processing
* Schema enforcement
* Table properties
* OPTIMIZE
* VACUUM
* Liquid Clustering

The Silver layer uses incremental `MERGE` processing to update existing hourly records when newer source data is received.

The Gold layer follows the same incremental philosophy by recalculating affected location/date combinations and merging them into the daily Gold table.

---

## Data Processing Strategy

The pipeline follows an incremental processing pattern.

### Bronze

Raw API responses are ingested and persisted.

### Silver

New Bronze records are parsed and merged into the hourly Silver dataset.

Existing records can be updated when a newer ingestion contains a more recent version of the forecast.

### Gold

Affected dates are identified from changes in Silver.

Only the affected location/date combinations are recalculated rather than rebuilding the entire Gold table on every execution.

This makes the Gold layer idempotent at its business grain:

```text
location_name + weather_date
```

---

## Data Quality

Data quality checks are incorporated into the transformation process.

Examples include:

* Required fields validation
* Schema validation
* Duplicate detection
* Grain validation
* Null checks
* Regression tests for transformation functions

The objective is to fail the pipeline when fundamental assumptions about the data are violated rather than silently producing incorrect downstream data.

---

## Environment Parameterization

The notebooks use Databricks widgets for configuration.

Typical parameters include:

```text
catalog
schema
environment
table names
```

This allows the same processing code to be reused across environments without hardcoding environment-specific configuration into the transformation logic.

The current project uses:

```text
dev
```

for development-oriented behavior.

---

## Git Workflow

The project is maintained using Git and follows a feature-branch promotion model.

The intended workflow is:

```text
feature
   │
   │ Pull Request
   ▼
  dev
   │
   │ Pull Request
   ▼
 prod/main
```

Feature branches should be short-lived and focused on a specific change.

Example:

```text
feature/openmeteo-gold-layer
```

After development and validation, the feature branch is merged into `dev`.

Once the changes have been validated in the development environment, `dev` is promoted to `main`/production through another Pull Request.

---

## Current Development Workflow

The current project is intentionally being developed manually before introducing Databricks Declarative Automation Bundles and CI/CD.

The progression is:

```text
Manual notebook development
        │
        ▼
Git version control
        │
        ▼
Manual branch promotion
        │
        ▼
Databricks Declarative Automation Bundles
        │
        ▼
CI/CD automation
```

This allows the data-processing logic and Git workflow to be established before introducing deployment automation.

---

## Future Improvements

Potential future improvements include:

* Databricks Declarative Automation Bundles
* Automated CI/CD
* Automated unit and integration testing
* Separate Databricks workspaces per environment
* Automated job deployment
* Environment-specific configuration
* Automated data-quality monitoring
* Additional Gold analytical datasets
* Production-grade observability and alerting

---

## Project Structure

The current implementation is organized around the Medallion layers:

```text
Bronze
└── Open-Meteo raw ingestion

Silver
└── Hourly weather normalization and incremental merge

Gold
└── Daily weather aggregation
```

The project intentionally keeps the data-processing logic independent from the future deployment mechanism so that the same notebooks can later be incorporated into a Databricks Declarative Automation Bundle.

---

## Technologies

* Databricks
* Apache Spark
* PySpark
* Delta Lake
* Unity Catalog
* Open-Meteo API
* Git
* Databricks Declarative Automation Bundles (future deployment layer)

