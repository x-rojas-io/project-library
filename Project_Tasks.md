# Task List

## Completed Work (Project A & B Fundamentals)
- [x] Configure Apache Airflow utilizing Docker Compose.
- [x] Connect Airflow to the on-premise SQL Server for Bronze ingestion.
- [x] Write PySpark scripts inside Dataproc Serverless to clean data (Silver).
- [x] Set up an Event-Driven architecture (Eventarc -> Cloud Functions -> Dataproc).
- [x] Write Python scripts to aggregate Silver data into BigQuery (Gold).
- [x] Fix BigQuery Spark Connector classpath collisions natively.
- [x] Document the entire Architecture, Pipeline, and Interview Preparation.

## What is Left To Build (The Next Steps)

### 1. Advanced Data Governance (Project A Completion)
- [ ] **SCD Type 2 in BigQuery**: Update the `06_gold_sales_fact.py` Gold layer script. Instead of simply overwriting the table, implement a SQL `MERGE` statement that tracks historical dimensional changes using `ValidFrom`, `ValidTo`, and `IsActive` tracking columns to prove 100% regulatory traceability.

### 2. Data Trust & Orchestration (Project B Completion)
- [ ] **Data Quality Monitoring**: Add an automated Data Quality validation step. This could be a Python/PySpark script that runs immediately before or after the Silver Dataproc job to guarantee constraints (e.g., "Ensure no duplicate `SalesOrderID`") before data hits the Gold layer.

### 3. Analytics Enablement (Project C Completion)
- [ ] **BigQuery BI Engine**: Configure and research BigQuery BI Engine in the GCP console to cache the Gold `FactSalesOrder` datasets in-memory. This acts as the "Druid Alternative" for high-concurrency, sub-second latency reporting.

### 4. The API Bridge (Project D)
- [ ] **REST API Macroservice**: Build a simple Python application using `FastAPI` or `Flask`.
- [ ] **Query Integration**: Configure the API to authenticate with your GCP Service Account, query the `FactSalesOrder` Gold table, and return the aggregated analytics as a JSON response to simulate integration with Backend/AI application teams.
