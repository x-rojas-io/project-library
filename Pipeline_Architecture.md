# Adventech Data Pipeline Architecture

## 1. Overview
The Adventech data pipeline follows a modern **Medallion Architecture** (Bronze, Silver, Gold), orchestrating on-premise database ingestion to a fully serverless data warehouse inside Google Cloud Platform (GCP). It leverages a hybrid approach: local orchestration via Dockerized Apache Airflow handles local network connections seamlessly, while natively relying on GCS Eventarc, Dataproc Serverless, and BigQuery in the cloud.

## 2. Component Details & Logic

### Bronze Layer (Ingestion)
- **Tool**: Apache Airflow (Dockerized) & Python Pandas
- **Flow**: Airflow triggers `01_bronze_ingestion.py` on a set schedule. The script connects to the on-premise SQL Server, extracts tables (`SalesOrderHeader`, `SalesOrderDetail`), converts them to highly compressed Parquet formats, and streams them to Google Cloud Storage (GCS).
- **Core Logic Snippet**:
```python
# Extract from local SQL Server
query = f"SELECT * FROM {schema}.{table}"
df = pd.read_sql(query, engine)

# Upload strictly to GCS Bronze Bucket
bucket = storage_client.bucket(BUCKET_NAME)
blob = bucket.blob(f"bronze/{schema}_{table}.parquet")
blob.upload_from_filename(f"/tmp/{schema}.{table}.parquet")
```

### Event-Driven Orchestration
- **Tool**: Eventarc & Cloud Functions (Gen 2)
- **Flow**: The moment a file lands in the `bronze/` folder, GCS emits an Eventarc trigger to a Python Cloud Function. The function analyzes the object route and dynamically submits a Dataproc Serverless Batch leveraging the appropriate PySpark mapping.
- **Core Logic Snippet**:
```python
script_mapping = {
    "Sales_SalesOrderHeader": f"gs://{BUCKET_NAME}/scripts/05_silver_sales_pyspark.py",
    "Sales_SalesOrderDetail": f"gs://{BUCKET_NAME}/scripts/05_silver_salesdetail_pyspark.py"
}
pyspark_script = script_mapping.get(table_basename)

batch = dataproc_v1.Batch()
batch.pyspark_batch = dataproc_v1.PySparkBatch(main_python_file_uri=pyspark_script)
client.create_batch(request=...)
```

### Silver Layer (Cleansing & Transformation)
- **Tool**: Dataproc Serverless (PySpark)
- **Flow**: A serverless auto-scaling Spark cluster boots instantly to read the Parquet file from GCS. It applies strict data typing, handles null coalescing, and drops empty identifiers. The cleaned data frame is written directly into BigQuery using the pre-installed BigQuery Spark Connector.
- **Core Logic Snippet**:
```python
# Transform and clean
transformed_df = df \
    .filter(col("SalesOrderID").isNotNull()) \
    .withColumn("SalesOrderID", col("SalesOrderID").cast(IntegerType())) \
    .withColumn("ModifiedDate", col("ModifiedDate").cast(TimestampType()))

# Write natively to BigQuery
transformed_df.write \
    .format("bigquery") \
    .option("table", "adventech-project.silver_adventureworks.SalesOrderHeader") \
    .mode("overwrite") \
    .save()
```

### Gold Layer (Business Aggregation)
- **Tool**: BigQuery & Python
- **Flow**: Reads the cleansed Silver tables and aggregates them into business-ready Star Schema tables (Fact and Dimension). Analysts query these tables directly.
- **Core Logic Snippet**:
```python
overwrite_sql = f"""
    CREATE OR REPLACE TABLE `{GOLD_DATASET}.FactSalesOrder` AS
    SELECT * FROM `{SILVER_DATASET}.SalesOrderHeader`
"""
client.query(overwrite_sql).result()
```

---

## 3. Operations & Best Practices

### Monitoring & Observability
1. **Airflow UI**: Dedicated pipeline monitoring for the on-premise extraction process. Any SQL connection failures or GCS upload network drops are immediately visualized here.
2. **GCP Log Explorer**: All Cloud Function invocation logs and Dataproc Spark execution standard output (stdout/stderr) are centralized in Cloud Logging perfectly mapping to individual batch IDs.
3. **Cloud Monitoring Alerts**: You can define alerting policies targeting `cloud_function` metrics (e.g., execution failures > 0) to automatically notify engineers via Email, PagerDuty, or Slack.

### Error Handling & Reliability
1. **Idempotency**: All operations are built to be safely re-runnable. Rerunning Airflow simply overwrites the Bronze Parquet file. Re-triggering the Dataproc batch inherently overwrites the Silver BigQuery table without duplicating data.
2. **Dead-Letter Queues (DLQ)**: If the Cloud Function encounters recurring failure processing a new file, Eventarc integrates nicely with Pub/Sub DLQs to preserve the failed payloads for manual inspection without halting the entire system.
3. **Execution Retries**: While Airflow features native Python task retries, Cloud Functions must only have external retries enabled if the downstream execution natively handles identical deduplication.

### Data Governance & Security
1. **IAM Least Privilege**: Operations utilize a dedicated `user-advantech` Service Account configured with only the explicit roles needed (`Dataproc Worker`, `Dataproc Editor`, `BigQuery Admin`, `Storage Admin`).
2. **Column-Level Security (BigQuery)**: Highly sensitive Personally Identifiable Information (PII), such as `CreditCardID` or `AccountNumber`, can be restricted dynamically via BigQuery Policy Tags to only authorized HR/Finance groups.
3. **Lineage Tracking**: Activating BigQuery Data Lineage in Dataplex enables end-to-end impact analysis, visually mapping schema relationships from Silver directly to dependent downstream Gold BI dashboards.

### Advanced Data Management Strategy
To support robust historical reporting and quick error recovery, the pipeline integrates standard data warehousing patterns with native BigQuery capabilities:

1. **BigQuery Time Travel**
   - BigQuery provides built-in **Time Travel**, allowing continuous access to historical data stored in any table (Silver or Gold) going back up to 7 days natively without needing explicit backups.
   - **Why it matters**: If a bug in transformation logic accidentally overwrites your datasets, you can instantly restore the table to its exact state from hours ago using a simple SQL query (e.g., `SELECT * FROM table FOR SYSTEM_TIME AS OF TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)`).

2. **Slowly Changing Dimensions (SCD)**
   - To track changes to business entities over time (e.g., a Customer moving to a new Territory), the Gold Layer integrates SCD patterns.
   - **SCD Type 1 (Overwrite)**: Applied when only the most recent state matters. For instance, our Silver layer pipeline uses an `overwrite` mode to constantly mirror the exact state of the Bronze files.
   - **SCD Type 2 (Historical Tracking)**: Used for analytical Dimensions where history must be preserved (e.g., a `DimCustomer` table). Instead of updating existing records, SCD Type 2 inserts new rows with `ValidFrom`, `ValidTo`, and `IsActive` tracking columns. This allows sales reports to correctly attribute historical orders to the customer's *past* address rather than their *current* address.

---

## 4. Potential Follow-Up Considerations
Review the following areas to continue scaling the architecture:

1. **Airflow DAG Integration**: Should we add the `06_gold_sales_fact.py` script as the final explicit task directly inside your Airflow DAG to enforce sequential dependencies?
2. **Data Quality Tests**: Would you like to introduce row-level validation (e.g., alerting when Order Quantity is negative or primary keys are duplicated) inside the PySpark Silver layer before the data hits BigQuery?
3. **Infrastructure as Code**: Are you interested in converting these GCP resources (Buckets, Datasets, Cloud Functions, and Eventarc IAM Bindings) into Terraform files (`.tf`) for easy, reproducible deployments?
4. **CI/CD Pipelines**: Should we configure a GitHub Actions workflow to automatically update the Dataproc GCS scripts and deploy the Cloud Function whenever you push a code change to `main`?

---

## 5. Interview Preparation (Q&A)

If asked to defend or explain this architecture in an interview, here are the key questions you should anticipate and how to answer them:

### Architecture & Design Choices

**Q: Can you explain the specific responsibility of each layer in this pipeline, and why it's beneficial to physically separate the raw Bronze data from the cleansed Silver data?**
> **A:** The Medallion Architecture organizes data into varying levels of validation.
> - **Bronze (Raw)**: Acts as an immutable, historical landing zone. It holds data exactly as it was extracted from the source. We store it in GCS as Parquet for cheap, compressed storage.
> - **Silver (Cleansed)**: This is the filtered, typed, and normalized version of Bronze. It resolves nulls, casts data types (like timestamps), and provides a reliable foundation for analytics. It resides natively in BigQuery.
> - **Gold (Aggregated)**: Built from Silver, this layer contains business-specific aggregations and Star Schemas optimized for BI tools natively.
> 
> *Benefits*: Separating raw from cleansed data ensures we never lose the original state. If our transformation logic in Silver is flawed, we can always roll back and reprocess the unchanged Bronze data without needing to hit the production SQL Server again.

**Q: Why run Apache Airflow locally (using Docker) instead of natively natively in GCP (like Cloud Composer), and why not process everything locally?**
> **A:** Running Airflow locally via Docker allows for secure, straightforward connectivity to the on-premise SQL Server without needing to configure complex Cloud VPN tunnels bridging GCP and the on-prem data center. However, while extraction is local, running heavy transformations (PySpark) locally would drain local compute and fail on large datasets. Pushing partitioned Parquet files to GCS and using Dataproc Serverless allows us to leverage virtually infinite, scalable compute in the cloud without actively managing cluster infrastructure.

**Q: What are the advantages of writing the Silver data natively into BigQuery rather than querying Parquet files directly from GCS via External Tables?**
> **A:** While querying external Parquet tables is possible, loading data natively into BigQuery provides significant performance boosts. BigQuery's native storage (Capacitor format) is deeply optimized for its compute engine, allowing for advanced caching, clustering, and partitioning. Furthermore, external tables do not support Data Manipulation Language (DML) operations like `UPDATE`, `DELETE`, or `MERGE`, which are heavily required for Slowly Changing Dimension (SCD) operations in the Gold layer.

### Data Processing & Operations

**Q: Why use Pandas for the Bronze Data Extraction, but switch to PySpark (Dataproc Serverless) for the Silver transformations?**
> **A:** Pandas operates strictly in-memory on a single node. It is highly efficient for lightweight extractions and writing small-to-medium data batches locally. However, Pandas is constrained by RAM and will crash with Out-Of-Memory (OOM) errors on massive datasets. PySpark is a distributed data processing framework. Utilizing Dataproc Serverless PySpark for Silver transformations ensures the pipeline natively horizontally scales to process gigabytes or terabytes of data across multiple worker nodes under the hood.

**Q: How does your pipeline guarantee "idempotency"?**
> **A:** Idempotency means an operation can be applied multiple times without changing the result beyond the initial successful application. This is critical for data pipelines so retries don't cause duplicate data.
> - **Bronze**: Airflow writes the extracted data to the exact same GCS file path, safely overwriting the previous file.
> - **Silver**: The Dataproc PySpark script writes to BigQuery using `.mode("overwrite")` (for full snapshot tables) or executes structurally safe `MERGE` statements.
> If a job fails midway and auto-retries, no duplicate ghost rows are created.

**Q: How would you implement SCD Type 2 tracking in BigQuery for the Gold layer?**
> **A:** To track historical dimensional changes (e.g., a customer moving to a new address over time), an SCD Type 2 implementation requires a `MERGE` statement in BigQuery and three tracking columns: `ValidFrom`, `ValidTo`, and `IsActive`. 
> When a customer record arrives from Silver with a changed attribute:
> 1. We `UPDATE` their existing active record in the Gold table, setting `IsActive = False` and `ValidTo = CURRENT_TIMESTAMP()`.
> 2. We simultaneously execute an `INSERT` for their new incoming record, setting `IsActive = True`, `ValidFrom = CURRENT_TIMESTAMP()`, and `ValidTo = '9999-12-31'`.

**Q: Imagine an analyst accidentally runs a DROP TABLE or bad UPDATE command on your FactSalesOrder Gold table. How do you recover the data?**
> **A:** We would immediately leverage BigQuery Time Travel. BigQuery automatically retains historical data states for up to 7 days natively. We can restore the table back exactly to how it looked an hour ago using a simple SQL query:
> ```sql
> CREATE OR REPLACE TABLE adventech-project.gold_adventureworks.FactSalesOrder AS
> SELECT * FROM adventech-project.gold_adventureworks.FactSalesOrder
> FOR SYSTEM_TIME AS OF TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)
> ```
