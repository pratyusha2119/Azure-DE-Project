# Azure-DE-Project
Car Sales Projcet


The data is present in Azure SQL Database which is ingested into Azure DataLake Storage Gen2 (ADLS) using Azure Data Factory(ADF).
**Linked Service** : They are used to connect to particular database or Containers.
**Datasets** : They are used to connect to particular tables in the database.

The Medallion architechure is used (Bronze-Silver-Gold layers)

For the Source data to be loaded into ADLS into bronze container.

A Linked Service **LS_Github_Car_Sales_Prep** was used to connect to the data in Github repository 
Then Dataset **DS_Git** was created and given base and realtive URL. A parameter called "load_flag" was created. 

Then a pipeline (**Source_Prep_pipeline**) was created to get Data from github and load it to Azure SQL DB.

With the help of Copy Data activity 
Source : Using DS_Git with property load_flag as SalesData.csv and using GET method to get the data from Github and load it into Sink
Sink: Using DS_AzureSqlDB to sink data into Azure SQL DB. Dataset property table name as source_cars_data
LS_AzureSqlDB: Autoresolve integration runtime as data is being moved within cloud. Given database name DB-CAR-SALES
The flow is as follows: Github -> LS_Github_Car_Sales_Prep -> DS_Git -> Autoresolve integration runtime -> LS_AzureSqlDB -> DS_AzureSqlDB -> Source_cars_data

WaterMark table is created to have the value that is less than the min value of date value present in source_cars_data table. 

Then a pipeline (**Incremental_Data_Pipeline**) was created with 2 lookup, 1 copy data and 1 stored procedure activities to get the data and incrementally load newly added data.
1 Lookup - Last_Load : Was used to get the till to the max current date possible 
1 Lookup - Current Load: Was used to get the data for incremental loading
Copy data - Used to load the data into sink
Stored Procedure- used to change the max current data in the watermark table to current max date for any future data loading

Last_load lookup : select * from water_mark is used to get the min value into copy data source statement
Current_Load lookup: select max(date_id) as max_date from source_cars_data is used to get the max current date present in the source table.
CopyData - source : select * from source_cars_data
where date_id > '@{activity('Last_Load').output.value[0].last_load}' and date_id <= '@{activity('Current_Load').output.value[0].max_date}'
is used to get the data from day 1 to the current date.
Copydata - sink : LS_ADLS and DS_ADLS is used to connect to bronze layer in ADLS
Stored Procedure : LS_AzureSqlDB is used to connect to stored procedure updatewatermarktable

create procedure updatewatermarktable @lastload varchar(2000)
as
begin

begin transaction;

update water_mark
set  last_load = @lastload

commit transaction;
end;

The stored Procedure updates the last_load column in "water_mark" table usin the parameter created in stored procedure activity lastload which takes the value - @activity('Current_Load').output.value[0].max_date


Parameter can be created in Dataset which when used in activities to connect to any source or sink will ask the details in the dataset property
with propert name as parameter name and value needs to be given.  Will the value be automatically be taken or given?






