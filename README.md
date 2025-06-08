# Uber Trips Analysis

# Objective
The goal of this project is to perform an end-to-end analysis of Uber Trips Data using various tools and techniques such as GCP Storage, Python, Compute Instance, Mage Data Pipeline Tool, BigQuery, and Looker Studio.

# Table of Contents


# Tools used
* Google Cloud - Data storage in GCP platform
* Google Compute -  Creates a VM instance and then connecting to MAGE using SSH with the use of external IP/port address
* Jupyter Notebook - Creating the Python codes for performing ETL process.
* MAGE - Deployed on Google Compute Engine for creating the ETL data pipeline. Python codes from Jupyter Notebook are deployed on MAGE for running the data pipeline.
* Big Query - SQL querying and database management
* Looker - Generating the visualizations

# Data Architecture
<img src="https://github.com/bayyangjie/Uber-Trips-Analysis/blob/main/images/data%20architecture.png" width="100%">

**Data storage - Google Cloud Platform** <br>

The dataset is uploaded to Google Cloud Platform by creating a bucket and then storing the dataset in it. A public URL is generated which is used as part of the data exporting process in the data pipeline later on. <br>

<img src="https://github.com/bayyangjie/Uber-Trips-Analysis/blob/main/images/dataset_bucket.png" width="100%">

**ETL - Google Compute Engine (GCE), MAGE** <br>

After uploading the dataset to Google Cloud, a VM instance is set up in GCE for installing and running MAGE on it to build and execute the data pipeline for the ETL process. <br>

<img src="https://github.com/bayyangjie/Uber-Trips-Analysis/blob/main/images/VM_instance.png" width="100%">

**Database Management - Google Bigquery** <br>

After exporting the transformed dataset from MAGE's exporter block, the data is loaded into Bigquery where further SQL-based data transformation. It is done through a series of JOIN statements to extract required columns from the DIMENSION tables and FACT tables to create a separate table for the visualisation phase. <br>

<img src="https://github.com/bayyangjie/Uber-Trips-Analysis/blob/main/images/bigquery.png" width="100%">

**Data Visualization - Google Looker Studio** <br>





# Data Modelling 

## Schema Design 
The purpose of modelling the data is to convert the dataset from a flat format into a star schema comprising the fact and dimension tables.
- Dimension Tables: Descriptive variables and create ID columns that serve as the primary key in the dimension table
- Fact Table: Combine the ID columns from the Dimension tables and are used as Foreign keys in the Fact table. Other quantitative columns from original dataset are included as well. The Fact table only consists of numerical values for analysis.

<br>

![data model](https://github.com/bayyangjie/Uber-Trips-Analysis/blob/main/images/data%20model.png)

## Creating Dimension Tables
The dimension table are created with Python in Jupyter Notebook. Each table is created by extracting the descriptive column keys from the dataset. An ID column is created in each of the dimension tables to represent the indexing sequence from 0.

Some dimension tables involved additional transformation steps to map the actual label names to each distinct numerical value in the original dataset column: <br>

* datetime_dim
The hour, day, month, year, weekday values were extracted from the pickup/dropoff date-time columns. This was done to enable a more detailed analysis. <br>
```python
# Creating the dimension table "datetime_dim"
datetime_dim = df[['tpep_pickup_datetime','tpep_dropoff_datetime']].reset_index(drop=True)

# Creating pickup date/time columns using the "datetime_dim" table
datetime_dim['pickup_hour'] =  datetime_dim['tpep_pickup_datetime'].dt.hour  # creating a pickup hour column
datetime_dim['pickup_day'] =  datetime_dim['tpep_pickup_datetime'].dt.day  # creating a pickup day column
datetime_dim['pickup_month'] =  datetime_dim['tpep_pickup_datetime'].dt.month  # creating a pickup month column
datetime_dim['pickup_year'] =  datetime_dim['tpep_pickup_datetime'].dt.year  # creating a pickup year column
datetime_dim['pickup_weekday'] =  datetime_dim['tpep_pickup_datetime'].dt.weekday  # creating a pickup weekday column

# Creating dropoff date/time columns using the "datetime_dim" table
datetime_dim['dropoff_hour'] =  datetime_dim['tpep_dropoff_datetime'].dt.hour  # creating a dropoff hour column
datetime_dim['dropoff_day'] =  datetime_dim['tpep_dropoff_datetime'].dt.day  # creating a dropoff day column
datetime_dim['dropoff_month'] =  datetime_dim['tpep_dropoff_datetime'].dt.month  # creating a dropoff month column
datetime_dim['dropoff_year'] =  datetime_dim['tpep_dropoff_datetime'].dt.year  # creating a dropoff year column
datetime_dim['dropoff_weekday'] =  datetime_dim['tpep_dropoff_datetime'].dt.weekday  # creating a dropoff weekday column

# Creating a column to represent the row index values in the "datetime_dim" table
datetime_dim['datetime_id'] = datetime_dim.index
```

* rate_code_dim
There are 6 distinct values in the column "RatecodeID". Each distinct value corresponds to a rate code type name. An additional mapping step was done to map the numerical values to it's corresponding code type name.
```python
rate_code_type = {
    1:"Standard rate",
    2:"JFK",
    3:"Newark",
    4:"Nassau or Westchester",
    5:"Negotiated fare",
    6:"Group ride"
}
 
# create new dataframe from extracting 'RatecodeID' column from the original dataset and resetting the indexing to start from 0 sequentially
rate_code_dim = df[['RatecodeID']].reset_index(drop=True)  

# uses the new sequential index and assigns a name to the index column
rate_code_dim['rate_code_id'] = rate_code_dim.index  

# creating a new column to map names to the values in the 'RatecodeID' column
rate_code_dim['rate_code_name'] = rate_code_dim['RatecodeID'].map(rate_code_type)
```

* payment_type_dim
* Similarly for the payment types dimension table, there are 6 distinct numerical values under the column 'payment_type'. Mapping was done to map the numerical values to the payment type labels (Credit card, Cash, Dispute etc).
```python
payment_type_name = {
    1:"Credit card",
    2:"Cash",
    3:"No charge",
    4:"Dispute",
    5:"Unknown",
    6:"Voided trip"
}

# creating new dataframe/table by extracting the column 'payment_type' from the original dataset and resetting the indexing to start from 0 sequentially
payment_type_dim = df[['payment_type']].reset_index(drop=True)

# use the newly resetted sequential index and assign a new column name
payment_type_dim['payment_type_id'] = payment_type_dim.index

# map the original numerical values in column 'payment_type' to actual label names under a newly created column
payment_type_dim['payment_type_name'] = payment_type_dim['payment_type'].map(payment_type_name)
```

## Creating Fact Table
The Fact table was created by merging the original dataframe with the Dimension tables based on the Primary-Foreign key relationship. The ID columns which represented the row indexing in both the dimension tables and fact table were used as the common key. In the original dataframe, the row indexing is reset to a defualt numerical sequence starting from 0 as well and 'trip_id' is assigned as the name of the row index column.
```python
# creating the fact table by merging the original dataframe with the individual dimension tables using the common key
fact_table = df.merge(passenger_count_dim, left_on='trip_id', right_on='passenger_count_id')\
             .merge(trip_distance_dim, left_on='trip_id', right_on='trip_distance_id')\
             .merge(pickup_location_dim, left_on='trip_id', right_on='pickup_location_id')\
             .merge(dropoff_location_dim, left_on='trip_id', right_on='dropoff_location_id')\
             .merge(datetime_dim, left_on='trip_id', right_on='datetime_id')\
             .merge(rate_code_dim, left_on='trip_id', right_on='rate_code_id')\
             .merge(payment_type_dim, left_on='trip_id', right_on='payment_type_id')
```

# Data Cleaning
* When inspecting the data types of the variables in the dataset, it was discovered that the date variables 'tpep_pickup_datetime' and 'tpep_dropoff_datetime' had incorrect 'object' datatype. and was converted into 'datetime64' data type.


# Google Cloud Storage
The csv dataset is loaded and stored on the Google Cloud Platform.

# MAGE - Data Pipeline
MAGE is used as the tool for creating, managing and orchestrating the data pipeline in this project.

## MAGE - DATA LOADER block
The csv data is extracted using the url generated in GCP and then converted into a dataframe. A HTTP GET request is used to fetch the dataset content using the cloud URL directory in GCP. 

The CSV file is then converted into a file-like object (io.StringIO) to be pandas-readable which is then parsed into a DataFrame using pd.read_csv().

```mage
@data_loader
def load_data_from_api(*args, **kwargs):
    """
    Template for loading data from API
    """
    url = 'https://storage.googleapis.com/uber_trip_analysis_project/uber%20dataset.csv'
    response = requests.get(url)

    return pd.read_csv(io.StringIO(response.text), sep=',')
```

## MAGE - TRANSFORMER block
The data from the LOAD stage is then parsed into the TRANSFORMATION block. This is achieved by simply by copying the python code that was written in Jupyter Notebook into MAGE. The transformation block here is named as "uber_transformation".

## MAGE - EXPORTER block
* The YAML file (io.config.yaml) contains the GOOGLE BIGQUERY credentials to be used for connecting to BigQuery. A full directory of where the YAML file is located is formed by merging the paths of the root working directory in MAGE and the location of the YAML file itself. <br>
```mage
def export_data_to_big_query(data: dict[str, DataFrame], **kwargs) -> None:
    """
    Export each DataFrame in the dictionary to a separate BigQuery table.
    Only export tables whose names end with '_dim'.
    """
    config_path = path.join(get_repo_path(), 'io_config.yaml')
    config_profile = 'default'
```

* 'for' loop

The YAML file credentials to BigQuery is stored inside the object "config_path" which designates the directory where the credentials are stored in MAGE. <br>

config_path: path to the YAML file (e.g., /your/mage/project/io_config.yaml) <br>

config_profile: which profile in the YAML to use (e.g., default)
```MAGE
def export_data_to_big_query(data: dict[str, DataFrame], **kwargs) -> None:
    Export each DataFrame in the dictionary to a separate BigQuery table.
    """
    config_path = path.join(get_repo_path(), 'io_config.yaml')
    config_profile = 'default'
```

The table_id refers to the ID of each set of dataset name-table pair in Bigquery.

BigQuery.with_config() loads the BigQuery credentials and settings from the io_config.yaml file (read by ConfigFileLoader). 

Each looped set of table_name (name of each DataFrame returned from TRANSFORMER block) & df (the actual pandas dataframe corresponding to each name) is being dynamically exported and created in BigQuery. 

Any reruns of the segments (i.e LOADER/TRANSFORMER/EXPORTER) in MAGE will also overwrite existing data to prevent data duplication.

```MAGE
for table_name, df in data.items():
        table_id = f'uber-trips-analysis.uber_data_engineering_yt.{table_name}'

        BigQuery.with_config(ConfigFileLoader(config_path, config_profile)).export(
            df,
            table_id,
            if_exists='replace',
```

# Google BigQuery - Database Querying
* Created an empty dataset named "uber_data_engineering_yt" to store the exported dataframe tables (from the TRANSFORMER block) in Google BigQuery.

<img src="https://github.com/bayyangjie/Uber-Trips-Analysis/blob/main/images/bigquery_table_uber_engineering_yt.png" width=30%>

# Analytical Layer - Database SQL Querying
After exporting the FACT table and the DIMENSION tables to BigQuery, they are joined together to create a table that contains required/selected descriptive columns from the DIMENSION tables as well as required/selected numerical columns from the FACT table.

The combined table is stored under the name 'tbl_analytics'. 

```sql
CREATE OR REPLACE TABLE `uber-trips-analysis.uber_data_engineering_yt.tbl_analytics` AS(
SELECT 
f.VendorID,
d.tpep_pickup_datetime,
d.tpep_dropoff_datetime,
p.passenger_count,
t.trip_distance,
r.rate_code_name,   -- only "rate_code_name" key is pulled because we are mapping the numerical values of RatecodeID
pick.pickup_longitude,
pick.pickup_latitude,
drop.dropoff_longitude,
drop.dropoff_latitude,
pay.payment_type_name,  -- we pull payment_type_name because we are mapping the numerical values to the actual label names
f.fare_amount,
f.extra,
f.mta_tax,
f.tip_amount,
f.tolls_amount,
f.improvement_surcharge,
f.total_amount,
f.store_and_fwd_flag

FROM 
`uber-trips-analysis.uber_data_engineering_yt.fact_table` f 
JOIN `uber-trips-analysis.uber_data_engineering_yt.datetime_dim` d ON f.datetime_id = d.datetime_id
JOIN `uber-trips-analysis.uber_data_engineering_yt.passenger_count_dim` p ON f.passenger_count_id = p.passenger_count_id
JOIN `uber-trips-analysis.uber_data_engineering_yt.trip_distance_dim` t ON f.trip_distance_id = t.trip_distance_id
JOIN `uber-trips-analysis.uber_data_engineering_yt.rate_code_dim` r ON f.rate_code_id = r.rate_code_id
JOIN `uber-trips-analysis.uber_data_engineering_yt.pickup_location_dim` pick ON f.pickup_location_id = pick.pickup_location_id
JOIN `uber-trips-analysis.uber_data_engineering_yt.dropoff_location_dim` drop ON f.dropoff_location_id = drop.dropoff_location_id
JOIN `uber-trips-analysis.uber_data_engineering_yt.payment_type_dim` pay ON f.payment_type_id = pay.payment_type_id);
```

# Looker Studio - Building Visualizations
In the Looker Dashboard, it provides an overview of different metrics about the Uber Dataset including filters to enhance user interaction. The visuals encompass a use of summary scorecards, bar plots and geographical map.

The scorecards provide an overview of financial figures such as average revenue generated by all trips, average fare amount charged to passengers and average tip amount by passengers.

The geographical map points to the location where most pickups happened in the country.

The barplot shows a comparison between the different rate code types in terms of total revenue and also total tip amount by each Vendor ID. 

