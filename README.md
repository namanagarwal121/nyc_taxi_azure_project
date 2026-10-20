# NYC Green Taxi Lakehouse on Azure (Data Factory, Databricks)

An end-to-end Azure lakehouse project on NYC TLC green taxi trip data for 2023. Azure Data Factory downloads the monthly Parquet files over HTTP into the bronze layer of ADLS Gen2, and Azure Databricks notebooks standardize and transform the data into a silver layer (Parquet) and publish gold Delta tables to Unity Catalog and ADLS.

## Architecture / Flow

```
NYC TLC trip data (HTTP, Parquet)
   -> Azure Data Factory (ForEach month 01..12, Copy)
   -> Bronze  (ADLS Gen2 / UC volume: trip-data, trip_type, trip_zone)
   -> Silver  (Databricks: schema alignment, cleaning, Parquet)
   -> Gold    (Delta tables nyc.gold.* and abfss://gold@... )
```

## Tech Stack

- Azure Data Factory (HTTP and ADLS Gen2 linked services, Git publish branch `adf_publish`)
- Azure Data Lake Storage Gen2
- Azure Databricks (PySpark, Spark SQL, Delta Lake, Unity Catalog volumes)

## Repository Structure

```
.
├── factory/dfnyctaxiazureproject.json   # Data Factory resource
├── linkedService/
│   ├── ls_http.json                     # HTTP source (TLC CloudFront endpoint, anonymous)
│   └── ls_datalake.json                 # ADLS Gen2 storage
├── dataset/
│   ├── SourceDataSet.json               # green_tripdata_2023-<MM>.parquet (parameter P_month)
│   └── nyc_raw_sink.json                # bronze/trips2023data Parquet sink
├── pipeline/nycwebtoDL.json             # Web -> data lake ingestion pipeline
├── DataBricks/
│   ├── Silver_notebook.ipynb            # Bronze -> silver transformations
│   └── gold layer.ipynb                 # Silver -> gold Delta tables
└── publish_config.json                  # ADF publish settings
```

## Stages

**Ingestion: `nycwebtoDL` (ADF)**
- A sequential ForEach over `range(1,12)` builds a two-digit month (`01`..`12`) and copies `trip-data/green_tripdata_2023-<MM>.parquet` from `https://d37ci6vzurychx.cloudfront.net/` into the `bronze` container, folder `trips2023data`.

**Silver: `Silver_notebook.ipynb` (Databricks)**
- Trip type: reads CSV from `bronze/trip_type/`, renames `description` to `trip_description`, writes Parquet to `silver/trip_type`.
- Trip zone: reads `bronze/trip_zone/taxi_zone_lookup.csv`, splits `zone` on `/` into `zone1` and `zone2`, writes Parquet to `silver/trip_zone`.
- Trip data: reads every monthly Parquet file under `bronze/trip-data/`, adds any missing columns as nulls, casts each file to one explicit schema (VendorID, pickup/dropoff timestamps, location IDs, distance, fare components, payment and trip type), and unions them by name.
- Adds `trip_date`, `trip_year` and `trip_month`, then keeps `VendorID`, `PULocationID`, `DOLocationID`, `trip_distance`, `fare_amount`, `total_amount` and writes Parquet to `silver/trips2023data`.

**Gold: `gold layer.ipynb` (Databricks)**
- Creates schema `nyc.gold`.
- Reads `trip_zone`, `trip_type` and `trips2023data` from silver and writes each as a Delta table (`nyc.gold.trip_zone`, `nyc.gold.trip_type`, `nyc.gold.trips2023data`) and as Delta files in the `gold` container of ADLS Gen2.

## Dataset

- NYC TLC green taxi trip records, 2023 (`green_tripdata_2023-MM.parquet`), from the TLC public CloudFront endpoint.
- `taxi_zone_lookup.csv` (taxi zones) and a trip type CSV, placed in the bronze layer separately (not ingested by the ADF pipeline in this repo).

## How to Run

Prerequisites:
- ADLS Gen2 account with `bronze`, `silver` and `gold` containers; update `ls_datalake` with your own account and credentials.
- Databricks workspace with Unity Catalog catalog `nyc`, schema `default`, and volumes `bronze` and `silver`, plus access to the `gold` container for the `abfss://` writes.

Steps:
1. Connect ADF to this repository and run `nycwebtoDL` to land the 2023 monthly files in bronze.
2. Upload `taxi_zone_lookup.csv` to `bronze/trip_zone/` and the trip type CSV to `bronze/trip_type/`.
3. Run `Silver_notebook.ipynb`, then `gold layer.ipynb`.

Note: the ADF sink folder is `trips2023data`, while the silver notebook reads from `bronze/trip-data/`; align the paths for your environment.
