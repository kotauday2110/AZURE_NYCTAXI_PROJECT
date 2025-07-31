# NYC Taxi Data Engineering Project

## Overview

This project implements a comprehensive data engineering pipeline for processing NYC Taxi and Limousine Commission (TLC) trip record data. The solution leverages Azure cloud services to build a scalable, automated ETL pipeline that processes taxi trip data through Bronze, Silver, and Gold layers following the medallion architecture pattern.

## Architecture

The project follows a modern data lake architecture with the following components:

### Data Sources
- **NYC TLC Trip Record Data**: Green taxi trip data from the official NYC government website
- **Source URL**: https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page
- **Data Format**: Parquet files with monthly partitions (2023 data)

### Azure Cloud Infrastructure
- **Azure Data Lake Storage Gen2**: Multi-tier storage (Bronze, Silver, Gold containers)
- **Azure Data Factory**: Orchestration and data movement
- **Azure Databricks**: Data processing and transformation
- **Service Principal**: Secure authentication and access control

### Data Architecture Layers

#### Bronze Layer (Raw Data)
- Stores raw data exactly as received from source
- Parquet format preservation
- Contains trip data, trip types, and trip zones
- Path structure: `bronze/trips2023data/`, `bronze/trip_type/`, `bronze/trip_zone/`

#### Silver Layer (Cleaned Data)
- Cleaned and transformed data
- Data quality improvements and standardization
- Schema enforcement and validation
- Path structure: `silver/trips2023data/`, `silver/trip_type/`, `silver/trip_zone/`

#### Gold Layer (Business-Ready Data)
- Analytics-ready datasets
- Delta Lake format for ACID transactions
- Optimized for reporting and analytics
- Database: `gold` with tables: `trip_zone`, `trip_type`, `trip_trip`

## Technical Stack

### Core Technologies
- **Apache Spark**: Distributed data processing
- **PySpark**: Python API for Spark
- **Delta Lake**: Storage layer providing ACID transactions
- **Azure Data Factory**: Data orchestration
- **Azure Databricks**: Analytics platform

### Data Formats
- **Source**: Parquet files
- **Processing**: Parquet (Bronze/Silver)
- **Analytics**: Delta Lake (Gold)

## Project Structure

```
nyc-taxi-project/
├── notebooks/
│   ├── bronze_to_silver.py
│   ├── silver_to_gold.py
│   └── data_validation.py
├── data-factory/
│   ├── pipelines/
│   └── datasets/
├── config/
│   └── authentication.py
└── README.md
```

## Setup and Configuration

### Prerequisites
- Azure subscription with appropriate permissions
- Azure Data Lake Storage Gen2 account
- Azure Data Factory instance
- Azure Databricks workspace
- Service Principal with necessary RBAC roles

### Authentication Configuration

The project uses Service Principal authentication with OAuth2:

```python
# Azure Data Lake Authentication
spark.conf.set("fs.azure.account.auth.type.{storage_account}.dfs.core.windows.net", "OAuth")
spark.conf.set("fs.azure.account.oauth.provider.type.{storage_account}.dfs.core.windows.net", 
               "org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider")
spark.conf.set("fs.azure.account.oauth2.client.id.{storage_account}.dfs.core.windows.net", 
               "{client_id}")
spark.conf.set("fs.azure.account.oauth2.client.secret.{storage_account}.dfs.core.windows.net", 
               "{client_secret}")
spark.conf.set("fs.azure.account.oauth2.client.endpoint.{storage_account}.dfs.core.windows.net", 
               "https://login.microsoftonline.com/{tenant_id}/oauth2/token")
```

### Storage Account Setup

Create containers in Azure Data Lake Storage Gen2:
- `bronze` - Raw data storage
- `silver` - Processed data storage  
- `gold` - Analytics-ready data storage

## Data Pipeline

### Data Factory Pipeline

The pipeline includes conditional logic to handle different file naming patterns:

**Condition 1**: Files 01-09 (single digit months)
- Expression: `@greater(item(),9)`
- Dataset: `trip-data/green_tripdata_2023-0@{dataset().u_month}.parquet`

**Condition 2**: Files 10-12 (double digit months)  
- Dataset: `trip-data/green_tripdata_2023-@{dataset().u_monthgreater9}.parquet`

### Data Processing Workflow

#### 1. Bronze Layer Ingestion
- Raw data ingestion from NYC TLC website
- Preserve original data structure and format
- Store in Bronze container with partitioning

#### 2. Silver Layer Transformation
Key transformations include:

**Trip Type Data**:
- Rename `description` to `trip_description`
- Data validation and cleansing

**Trip Zone Data**:
- Split `Zone` column into `zone1` and `zone2` using `/` delimiter
- Geographic data standardization

**Trip Data**:
- Schema enforcement with predefined structure
- Date extraction: `trip_date`, `trip_year`, `trip_month`
- Column selection for optimized storage
- Data type validation and conversion

#### 3. Gold Layer Analytics
- Convert to Delta Lake format
- Create managed tables in `gold` database
- Enable time travel and versioning
- Support for CRUD operations

### Schema Definition

```python
trip_schema = '''
    VendorID BIGINT,
    lpep_pickup_datetime TIMESTAMP,
    lpep_dropoff_datetime TIMESTAMP,
    store_and_fwd_flag STRING,
    RatecodeID BIGINT,
    PULocationID BIGINT,
    DOLocationID BIGINT,
    passenger_count BIGINT,
    trip_distance DOUBLE,
    fare_amount DOUBLE,
    extra DOUBLE,
    mta_tax DOUBLE,
    tip_amount DOUBLE,
    tolls_amount DOUBLE,
    ehail_fee DOUBLE,
    improvement_surcharge DOUBLE,
    total_amount DOUBLE,
    payment_type BIGINT,
    trip_type BIGINT,
    congestion_surcharge DOUBLE
'''
```

## Key Features

### Data Quality & Governance
- Schema validation and enforcement
- Data type consistency checks
- Duplicate detection and handling
- Data lineage tracking

### Performance Optimization
- Partitioning strategies for large datasets
- Optimized file formats (Parquet, Delta)
- Caching for frequently accessed data
- Parallel processing with Spark

### Reliability & Monitoring
- Delta Lake ACID transactions
- Time travel capabilities for data recovery
- Comprehensive error handling
- Pipeline monitoring and alerting

### Scalability
- Auto-scaling Databricks clusters
- Distributed processing architecture
- Horizontal scaling capabilities
- Cost-optimized resource management

## Delta Lake Operations

The Gold layer supports advanced Delta Lake operations:

```sql
-- Time travel queries
SELECT * FROM gold.trip_zone VERSION AS OF 0

-- Data updates
UPDATE gold.trip_zone SET Borough = 'EWR' WHERE LocationID = 1

-- Data deletion with history
DELETE FROM gold.trip_zone WHERE LocationID = 1

-- History tracking
DESCRIBE HISTORY gold.trip_zone

-- Version restoration
RESTORE gold.trip_zone TO VERSION AS OF 0
```

## Usage

### Running the Pipeline

1. **Data Ingestion**: Execute Data Factory pipeline to ingest raw data
2. **Bronze Processing**: Run bronze layer notebooks for raw data storage
3. **Silver Transformation**: Execute silver layer transformations
4. **Gold Analytics**: Convert to Delta format and create analytics tables

### Querying Data

```sql
-- Analyze trip patterns by borough
SELECT Borough, COUNT(*) as trip_count 
FROM gold.trip_zone tz
JOIN gold.trip_trip tt ON tz.LocationID = tt.PULocationID
GROUP BY Borough

-- Monthly trip analysis
SELECT trip_year, trip_month, 
       AVG(fare_amount) as avg_fare,
       SUM(total_amount) as total_revenue
FROM gold.trip_trip
GROUP BY trip_year, trip_month
ORDER BY trip_year, trip_month
```

### Monitoring and Maintenance

- Monitor Data Factory pipeline execution
- Check Databricks job logs for processing status
- Validate data quality metrics
- Review Delta Lake table versions and optimization

## Security

- Service Principal authentication for secure access
- Role-based access control (RBAC)
- Network security groups and firewalls
- Data encryption at rest and in transit
- Audit logging and compliance tracking

## Performance Considerations

- **Partitioning**: Data partitioned by year/month for optimal query performance
- **File Sizing**: Optimized file sizes (1-2 MB) for efficient processing
- **Caching**: Strategic caching of frequently accessed datasets
- **Cluster Management**: Auto-scaling clusters based on workload demands

## Troubleshooting

### Common Issues
- **Authentication Failures**: Verify Service Principal credentials and permissions
- **Schema Mismatches**: Check data types and column names in source files
- **Performance Issues**: Review cluster configuration and data partitioning
- **Data Quality**: Implement data validation rules and monitoring

### Debugging Tips
- Enable detailed logging in Databricks notebooks
- Use Data Factory monitoring for pipeline execution tracking
- Implement data quality checks at each layer
- Regular testing with sample datasets

## Contributing

1. Follow the established naming conventions
2. Implement proper error handling
3. Add comprehensive logging
4. Update documentation for any changes
5. Test thoroughly before deployment

## Future Enhancements

- Real-time streaming data processing
- Advanced analytics and machine learning models
- Data visualization dashboards
- Automated data quality monitoring
- Cost optimization and resource management
- Integration with additional data sources

## Contact

For questions or support regarding this project, please contact the data engineering team or create an issue in the project repository.
