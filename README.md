# Azure Logistics Data Pipeline

An end-to-end data engineering project developed as part of an academic Data Engineering diploma.

The project simulates a logistics company's data workflow, integrating, transforming, and preparing shipment data for operational analysis using Microsoft Azure services.

## Project Overview

The pipeline processes logistics data from CSV and JSON files and organizes it into a layered data architecture. It applies data quality checks and generates analytical tables that can support business reporting and decision-making.

## Architecture

The project follows a medallion-style architecture with four layers:

- **Landing:** Stores the source files ingested into the data lake.
- **Bronze:** Preserves the ingested data for processing.
- **Silver:** Cleans, validates, and transforms the data.
- **Gold:** Stores curated datasets ready for analytical use.

## Technologies Used

- **Azure Data Factory:** Data ingestion and pipeline orchestration.
- **Azure Data Lake Storage Gen2:** Cloud storage for the data layers.
- **Azure Databricks:** Data processing and transformation.
- **PySpark:** Data cleaning, validation, and transformation.
- **Delta Lake:** Storage format for the analytical tables.

## Pipeline Workflow

1. Ingest CSV and JSON source files using Azure Data Factory.
2. Store the source data in Azure Data Lake Storage Gen2.
3. Process and validate the data using PySpark notebooks in Azure Databricks.
4. Apply data quality rules and exclude invalid records.
5. Generate curated analytical tables in Delta format.
6. Configure a scheduled trigger to automate pipeline execution.

## Data Quality

The pipeline includes validation rules to identify invalid shipment records. For example, records with non-positive shipment weights are excluded from the validated dataset.

In the dataset analyzed during the project, 89 records were evaluated and 87 valid records remained after the identified invalid records were excluded.

## Analytical Outputs

The Gold layer contains three analytical tables:

- `envios_por_sucursal` — shipment information organized by branch.
- `kpi_envios` — operational indicators related to shipments and deliveries.
- `rutas` — route-related information, including distance and estimated operational costs.

## Project Status

This project was completed as an academic exercise. The Azure resources used during development are no longer active, so the pipeline is not currently running in a live environment.

## Key Learnings

- Designing an end-to-end data pipeline.
- Organizing data using a layered architecture.
- Orchestrating ingestion and processing workflows.
- Applying data quality rules with PySpark.
- Preparing structured datasets for analytical reporting.

## Future Improvements

- Add automated tests for data quality rules.
- Implement monitoring and failure notifications.
- Add a dashboard connected to the Gold analytical tables.
- Improve documentation with architecture diagrams and execution evidence.

---

**Project type:** Academic Data Engineering Project  
**Focus:** Data ingestion, transformation, data quality, and analytics preparation
