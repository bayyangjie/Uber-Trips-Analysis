# Uber Trips Analysis

# Table of Contents
* [Objective](#objective)  
* [Tools Used](#tools-used)  
* [Data Architecture](#data-architecture)  
    * [Data Storage - GCP](#data-storage---gcp)  
    * [ETL - GCE, MAGE](#etl---gce-mage)  
    * [Analytics - Google Bigquery](#analytics---google-bigquery)  
    * [Data Visualization - Google Looker Studio](#data-visualization---google-looker-studio)  
* [Data Cleaning](#data-cleaning)  
* [Data Modelling](#data-modelling)  
    * [Schema Design](#schema-design)  
    * [Creating DIMENSION tables](#creating-dimension-tables)  
    * [Creating FACT table](#creating-fact-table)  
* [Conclusion](#conclusion)
    
# Objective
The goal of this project is to perform an end-to-end analysis of Uber Trips Data using various tools and techniques such as GCP Storage, Python, Compute Instance, Mage Data Pipeline Tool, BigQuery, and Looker Studio.

# Tools used
* Google Cloud Platform(GCP) - Data storage in GCP platform
* Google Compute Engine(GCE) -  Creates a VM instance and then connecting to MAGE using SSH with the use of external IP/port address
* Jupyter Notebook - Creating the Python codes for performing ETL process.
* MAGE - Deployed on Google Compute Engine for creating the ETL data pipeline. Python codes from Jupyter Notebook are deployed to run the data pipeline.
* Google Bigquery - SQL querying and analytics
* Google Looker Studio - Generating visualizations

# Data Architecture
<img src="https://github.com/bayyangjie/Uber-Trips-Analysis/blob/main/images/data%20architecture.png" width="100%">

## Data storage - GCP 
The dataset is uploaded to Google Cloud Platform by creating a bucket and then storing the dataset in it. A public URL is generated which is used as part of the data exporting process in the data pipeline later on. <br>

<img src="https://github.com/bayyangjie/Uber-Trips-Analysis/blob/main/images/dataset_bucket.png" width="100%">

## ETL - GCE, MAGE 
### Google Compute Engine
After uploading the dataset to Google Cloud, a VM instance is set up in GCE for installing and running MAGE on it to build and execute the data pipeline for the ETL process. <br>

<img src="https://github.com/bayyangjie/Uber-Trips-Analysis/blob/main/images/VM_instance.png" width="100%">

### MAGE
#### DATA LOADER block
The csv data is extracted using the url generated in GCP and then converted into a dataframe. A HTTP GET request is used to fetch the dataset content using the cloud URL directory in GCP. The CSV file is then converted into a file-like object (io.StringIO) to be pandas-readable which is then parsed into a DataFrame using pd.read_csv().

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

#### TRANSFORMER block
The data from the LOAD stage is then parsed into the TRANSFORMATION block. This is achieved by simply by copying the python code that was written in Jupyter Notebook into MAGE. The transformation block here is named as "uber_transformation".

#### EXPORTER block
The YAML file contains credentials for establishing a connection to Bigquery and the full path is located through the use of "config_path" and "config_profile" which indicates the location of the YAML file in the MAGE project as well as the specific section of the YAML file where the credentials are located. For each table name and dataframe pair in the dictionary 'data' , a table_id is built which indicates the export destinations in Bigquery. Together, the Bigquery credentials are loaded using ConfigFileLoader() and each iterated table name-dataframe pairs are exported to the respective table id destinations in Bigquery.
```mage
def export_data_to_big_query(data: dict[str, DataFrame], **kwargs) -> None:
    """
    Export each DataFrame in the dictionary to a separate BigQuery table.
    """
    config_path = path.join(get_repo_path(), 'io_config.yaml')
    config_profile = 'default'

    for table_name, df in data.items():
        table_id = f'uber-trips-analysis.uber_data_engineering_yt.{table_name}'

        BigQuery.with_config(ConfigFileLoader(config_path, config_profile)).export(
            df,
            table_id,
            if_exists='replace',
        )
```

## Analytics - Google Bigquery
After exporting the transformed dataset from MAGE's exporter block, the data is loaded into Bigquery under the dataset "uber_data_engineering_yt" where further SQL-based data transformation is performed to extract required columns from DIMENSION tables and the FACT table. <br>

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

## Data Visualization - Google Looker Studio
In the Looker Dashboard, it provides an overview of different metrics about the Uber Dataset including filters to enhance user interaction. The visuals encompass a use of summary scorecards, bar plots and geographical map.

### Overview of financial metrics
The scorecards provide an overview of financial metrics such as average revenue generated by all trips, average fare amount charged to passengers and average tip amount by passengers. <br>

<img src="https://github.com/bayyangjie/Uber-Trips-Analysis/blob/main/images/scorecard.png" width="100%">


### Location with most pickups
The geographical map points to the location where most pickups happened in the country. A calculated field "pickup_location" was created to concatenate the different latitude and longitude coordinates that point to the pickup location points. The map shows that most pickup locations were around New York City. <br>

```sql
CONCAT(pickup_latitude, ", ",pickup_longitude)
```

<img src="https://github.com/bayyangjie/Uber-Trips-Analysis/blob/main/images/bubble_map.png" width="100%">

### Revenue Amount by Rate Code/Payment types
The barplots shows a comparison between the different rate code types in terms of total revenue. The rate codes Nassau/Westchester and Newark generated the most amount of revenue. In terms of revenue generated by the payment types, the bulk of the revenue generated came from Credit Card payment method.

<img src="https://github.com/bayyangjie/Uber-Trips-Analysis/blob/main/images/barplot.png" width="100%">

# Data Cleaning
Upon inspecting the data types of the variables in the dataset, it was discovered that the pickup and dropoff date variables 'tpep_pickup_datetime' and 'tpep_dropoff_datetime' had incorrect datatype 'object' and was converted into 'datetime64' datatype.

# Data Modelling 

## Schema Design 
The purpoose of the data modelling process in this analysis is to convert the dataset from a flat format into a star schema comprising of FACT and DIMENSION tables. <br>

![data model](https://github.com/bayyangjie/Uber-Trips-Analysis/blob/main/images/data%20model.png)

## Creating DIMENSION tables
The DIMENSION tables are created with Python in Jupyter Notebook. Each table is created by extracting the descriptive column keys from the dataset together with the creation of an ID column to represent the Primary Key for the specific DIMENSION table. 

* passenger_count_dim <br>

DIMENSION table for passenger count in each trip.
```python
# Creating the dimension table "passenger_count_dim" by extracting the 'passenger_count' column from the dataset
# double[] is used because we want to return a DIMENSION table which is a DataFrame so the 2D structure is required
passenger_count_dim = df[['passenger_count']].drop_duplicates().reset_index(drop=True)  

# Creating an indexing column
# single[] is used because each column in the DataFrane is a Series and in this case we are assigning a new column to the DIMENSION table/DataFrame
passenger_count_dim['passenger_count_id'] = passenger_count_dim.index 
```

* trip_distance_dim <br>

DIMENSION table for the trip distance which shows the distance travelled of each trip.
```python
# Creating the dimension table "trip_distance_dim" by extracting the 'trip_distance' column from the dataset and resetting the indexing to start from 0 in sequence
trip_distance_dim = df[['trip_distance']].drop_duplicates().reset_index(drop=True)

# Use the new sequential index and assigns a name to the index column
trip_distance_dim['trip_distance_id'] = trip_distance_dim.index 
```

* pickup_location_dim <br>

DIMENSION table showing the latitude and longitude coordinates of each pickup location.
```python
# create new dataframe by extracting the pickup longitude and latitude columns from the original dataset and resetting index to start from 0 sequentially 
pickup_location_dim = df[['pickup_longitude','pickup_latitude']].drop_duplicates().reset_index(drop=True)

# use the new sequential index and assign a new coulmn header to the column index
pickup_location_dim['pickup_location_id'] = pickup_location_dim.index
```

* dropoff_location_dim <br>

DIMENSION table showing the latitude and longitude coordinates of each dropoff location.
```python
# create new dataframe by extracting the dropoff longitude and latitude columns from the original dataset and resetting index to start from 0 sequentially 
dropoff_location_dim = df[['dropoff_longitude','dropoff_latitude']].drop_duplicates().reset_index(drop=True)

# use the new sequential index and assign a new coulmn header to the column index
dropoff_location_dim['dropoff_location_id'] = dropoff_location_dim.index
```
There are some DIMENSION tables that require a mapping step to map the actual label names to the corresponding distinct numerical values in the original column, as shown below. <br>

* datetime_dim <br>

The hour, day, month, year, weekday values were extracted from the pickup/dropoff date-time column to provide a more detailed analysis. <br>
```python
# Creating the dimension table "datetime_dim"
datetime_dim = df[['tpep_pickup_datetime','tpep_dropoff_datetime']].drop_duplicates().reset_index(drop=True)

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

* rate_code_dim <br>

There are 6 distinct values in the column "RatecodeID" with each distinct value corresponding to a rate code type name.
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
rate_code_dim = df[['RatecodeID']].drop_duplicates().reset_index(drop=True)

# uses the new sequential index and assigns a name to the index column
rate_code_dim['rate_code_id'] = rate_code_dim.index  

# creating a new column to map names to the values in the 'RatecodeID' column
rate_code_dim['rate_code_name'] = rate_code_dim['RatecodeID'].map(rate_code_type)
```

* payment_type_dim <br>

There are 6 distinct numerical values for the column 'payment_type' each corresponding to a payment type name.
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
payment_type_dim = df[['payment_type']].drop_duplicates().reset_index(drop=True)

# use the newly resetted sequential index and assign a new column name
payment_type_dim['payment_type_id'] = payment_type_dim.index

# map the original numerical values in column 'payment_type' to actual label names under a newly created column
payment_type_dim['payment_type_name'] = payment_type_dim['payment_type'].map(payment_type_name)
```

## Creating FACT table
The FACT table was created by merging the original dataframe with the DIMENSION tables based on the Primary-Foreign key relationship. The ID columns serve as the Primary Key in the DIMENSION tables and the Foreign Key in the FACT table.
```python
# creating the fact table by merging the original dataframe with the individual dimension tables using the common key
fact_table = df.merge(passenger_count_dim, on='passenger_count')\
             .merge(trip_distance_dim, on='trip_distance')\
             .merge(pickup_location_dim, on=['pickup_longitude','pickup_latitude'])\
             .merge(dropoff_location_dim, on=['dropoff_longitude','dropoff_latitude'])\
             .merge(datetime_dim, on=['tpep_pickup_datetime','tpep_dropoff_datetime'])\
             .merge(rate_code_dim, on='RatecodeID')\
             .merge(payment_type_dim, on='payment_type')\
             [['VendorID', 'datetime_id', 'passenger_count_id',
               'trip_distance_id', 'pickup_location_id', 'dropoff_location_id', 'rate_code_id',
               'payment_type_id', 'fare_amount', 'extra', 'mta_tax', 'tip_amount', 'tolls_amount',
               'improvement_surcharge', 'total_amount', 'store_and_fwd_flag']]
```

# Conclusion
This project showcases how Google Cloud Platform offers a full stack data engineering solution from data cloud storage, hosting platform for running ETL orchestration tools (MAGE) to analysis and visualisation. This end to end data engineering solution also exemplifies how businesses can leverage modern tools like MAGE which lays out the basic framework for creating data pipelines,
to streamline day-to-day workloads.
