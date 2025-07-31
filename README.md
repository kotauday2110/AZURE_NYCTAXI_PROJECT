# NYC Taxi Data Engineering Project

## Overview

This project implements an automated ETL pipeline for processing NYC Taxi and Limousine Commission (TLC) trip record data using Azure cloud services. The solution follows a medallion architecture (Bronze-Silver-Gold) for scalable data processing and analytics.

**Data Source**: https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page

## Architecture

### Infrastructure
- **Azure Data Lake Storage Gen2**: Multi-tier storage (Bronze, Silver, Gold containers)
- **Azure Data Factory**: Pipeline orchestration with conditional logic
- **Azure Databricks**: PySpark data processing
- **Delta Lake**: ACID transactions and versioning
- **Service Principal**: OAuth2 authentication

### Data Flow
1. **Bronze Layer**: Raw parquet files from NYC TLC
2. **Silver Layer**: Cleaned and transformed data
3. **Gold Layer**: Analytics-ready Delta tables

## Setup

### Prerequisites
- Azure subscription with Data Lake Storage Gen2, Data Factory, and Databricks
- Service Principal with appropriate RBAC permissions

### Storage Containers
```
nyctaxiprojectudaystora/
├── bronze/          # Raw data
├── silver/          # Processed data  
└── gold/            # Analytics tables
```

### Authentication Configuration
```python
spark.conf.set("fs.azure.account.auth.type.{storage}.dfs.core.windows.net", "OAuth")
spark.conf.set("fs.azure.account.oauth.provider.type.{storage}.dfs.core.windows.net", 
               "org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider")
spark.conf.set("fs.azure.account.oauth2.client.id.{storage}.dfs.core.windows.net", "{client_id}")
spark.conf.set("fs.azure.account.oauth2.client.secret.{storage}.dfs.core.windows.net", "{secret}")
spark.conf.set("fs.azure.account.oauth2.client.endpoint.{storage}.dfs.core.windows.net", 
               "https://login.microsoftonline.com/{tenant_id}/oauth2/token")
```

## Data Pipeline

### Data Factory Configuration
**Conditional Logic for File Patterns**:
- Files 01-09: `trip-data/green_tripdata_2023-0@{dataset().u_month}.parquet`
- Files 10-12: `trip-data/green_tripdata_2023-@{dataset().u_monthgreater9}.parquet`
- Condition: `@greater(item(),9)`

### Data Processing

#### Bronze to Silver Transformations
```python
# Trip Type: Rename description column
df_trip_type = df_trip_type.withColumnRenamed('description','trip_description')

# Trip Zone: Split zone column
df_trip_zone = df_trip_zone.withColumn('zone1',split(col('Zone'),'/')[0])\
                           .withColumn('zone2',split(col('Zone'),'/')[1])

# Trip Data: Add date columns and select key fields
df_trip = df_trip.withColumn('trip_date',to_date('lpep_pickup_datetime'))\
                 .withColumn('trip_year',year('lpep_pickup_datetime'))\
                 .withColumn('trip_month',month('lpep_pickup_datetime'))\
                 .select('VendorID','PULocationID','DOLocationID','fare_amount','total_amount')
```

#### Silver to Gold (Delta Lake)
```python
# Create Delta tables in gold database
df_zone.write.format('delta')\
        .mode('append')\
        .option('path',f'{gold}/trip_zone')\
        .saveAsTable('gold.trip_zone')
```

### Schema Definition
```python
trip_schema = '''
    VendorID BIGINT,
    lpep_pickup_datetime TIMESTAMP,
    lpep_dropoff_datetime TIMESTAMP,
    PULocationID BIGINT,
    DOLocationID BIGINT,
    fare_amount DOUBLE,
    total_amount DOUBLE,
    trip_type BIGINT
    // ... additional fields
'''
```

## Key Features

- **Data Quality**: Schema validation and data type enforcement
- **Performance**: Partitioned storage and optimized file formats
- **Reliability**: Delta Lake ACID transactions and time travel
- **Scalability**: Auto-scaling Databricks clusters
- **Security**: Service Principal authentication and RBAC

## Delta Lake Operations

```sql
-- Query data
SELECT * FROM gold.trip_zone WHERE Borough = 'Manhattan'

-- Update records
UPDATE gold.trip_zone SET Borough = 'EWR' WHERE LocationID = 1

-- Time travel
SELECT * FROM gold.trip_zone VERSION AS OF 0
DESCRIBE HISTORY gold.trip_zone

-- Restore previous version
RESTORE gold.trip_zone TO VERSION AS OF 0
```

## Usage

1. **Run Data Factory Pipeline**: Ingest raw data to Bronze layer
2. **Execute Databricks Notebooks**: Transform Bronze → Silver → Gold
3. **Query Analytics Tables**: Use SQL or Spark for analysis

### Sample Analytics Query
```sql
SELECT Borough, COUNT(*) as trip_count,
       AVG(fare_amount) as avg_fare
FROM gold.trip_zone tz
JOIN gold.trip_trip tt ON tz.LocationID = tt.PULocationID
GROUP BY Borough
```

## Monitoring & Troubleshooting

- **Data Factory**: Monitor pipeline execution and failures
- **Databricks**: Check job logs and cluster metrics
- **Delta Lake**: Use `DESCRIBE HISTORY` for version tracking
- **Common Issues**: Authentication errors, schema mismatches, performance bottlenecks

## Project Structure

```
├── notebooks/
│   ├── bronze_to_silver.py
│   └── silver_to_gold.py
├── data-factory/
│   └── pipelines/
└── README.md
```

## Future Enhancements

- Real-time streaming processing
- Advanced analytics and ML models
- Automated data quality monitoring
- Cost optimization strategies
