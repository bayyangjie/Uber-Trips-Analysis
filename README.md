# Analysis of Uber Trips

# Objective


# Table of Contents


# Tools used
* Google Cloud - Used to store the data
* Google Compute - MAGE is used to build, deploy, and run data pipelines for performing ETL (Extract, Transform, Load) or ELT processes. Google Compute Engine (GCE) is just the infrastructure platform where Mage runs.
* Big Query - Used for running SQL queries
* Looker - Generating the visualizations

# Data Architecture


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
