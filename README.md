# Uber-Trips-Analysis

# Dimensional Modelling

* Converting the dataset from a flat format into a fact & dimension tables format. The tool Lucid Chart is used to perform this transformation.

* Created the data model by creating dimension tables of descriptive variables and the fact table of quantitative variables. The individual dimension tables are linked to the fact table through linking up the primary key in the dimension tables to the foreign key in the fact table. The foreign keys in the fact table are created in this analysis with the same name as the primary keys in the related dimension tables.
  * "passenger_count" is added to the fact table because it is a quantitative and measurable figure which would nautrally keep changing (passengers in a cab). But in this analysis, 'passenger_count' and 'trip_distance' are put in dimension tables for learning how to create dimension tables in the data model.

![data model](https://github.com/bayyangjie/Uber-Trips-Analysis/blob/main/images/data%20model.png)
