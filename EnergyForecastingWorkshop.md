﻿## MODULE ONE:  Setting up the Workshop Environment

This example is based on the public [SmartMeter Energy Consumption Data in London Households](https://data.london.gov.uk/dataset/smartmeter-energy-use-data-in-london-households) dataset that contains the smart meter readings from a sample of 5,567 London Households that took part in the UK Power Networks led Low Carbon London project between November 2011 and February 2014.  In this scenario, the power company must anticipate how much energy at each location will be needed, so they can have enough to cover the demand.  As a sample dataset, the purpose of this data isn't to produce highly accurate insights; instead this is to demonstrate a workflow pattern so you can understand how to adapt your own data for use with Amazon Forecast.

For background on building automated pipelines for Amazon Forecast, refer to the [normal set of steps provided for the MLOps workflow](https://github.com/aws-samples/amazon-forecast-samples/tree/main/ml_ops) samples here.

First you need to prepare the environment to hold and process the sample data.    Follow the steps below:

**Step 1: Get data for the Workshop**

  a) Create an S3 bucket to hold your data.  (Note: remember the name of this S3 bucket. You will need it for subsequest steps).  

  b) Upload the sample data for the workshop to your bucket.  (Note: you should have previously downloaded the sample data set onto your local machine.)  Find the folder named 'workshop-data' on your local machine and upload it into the S3 bucket you created in step 1a above.   This may take about 5 minutes.
  
  c) When the upload finishes, navigate to the S3 console, then find and explore this folder.  You should find these three subfolders:
  - **raw-meter-daily** - which contains the actual raw data collected from London smart meters, 
  - **synthetic-grid-master-data** - containing synthetic metadata about the grid to demonstrate more advanced use cases on the sample dashboards
  - **synthetic-meter-master-data** - additional synthetic metadata about the meters to demonstrate additional use cases.

**Step 2: Setting up Athena**

  Athena is Amazon's serverless query engine, which allows you to read and transform raw data files using standard SQL commands.   We will use Athena to perform several different data functions during the workshop.   But first, before you use Athena for the first time, you will need to you will take some steps to configure it: 

  a) Create a folder to store query results.  Go to the S3 console.  Select your workshop bucket and create a subfolder.   Remember the name you chose.  You will need it for the next step.

  b) Navigate to the Athena console.  At the top of the Athena screen you may see a dialog box that says:  "Before you run your first query, you need to set up a query result location in Amazon S3".  If so, click the "Edit Settings" button on the dialog box.  (NOTE: if you do not see the dialog box, select the "Settings" tab)

  ![Athena console - setting up a location for query results](https://github.com/rgleave/proj2/blob/master/workshop-athena-query-screen.png)
  
  You should now see the "Manage Settings" screen.  Click the "Browse S3" button.  Find the new folder you created in Step 3a and choose it.  Then, press the "Save" button to finish this setup.

  ![Manage Athena settings](https://github.com/rgleave/proj2/blob/master/workshop-manage-settings-bucket.png)

  Now we are ready to run SQL statements.

**Step 3: Establishing Glue Datbases and Tables**

  Before you can interact with raw text data, you must apply a structure (schema) over the data set to allow Athena to execute structured queries.  This process is called cataloging and generates databases and tables in the AWS Glue Catalog.  There are several ways to create databases and tables, however in this workshop we will use Athena to do so.  Navigate to the Athena console, and select the Editor tab.  Select  and run the following queries:

  a) This query creates a database to catalog our sample data.

```
create database samples_db;

```
  Once the query completes successfully, look at the **Data Pane** on the left side of the screen.  You will see a **Database** window. Select **samples_db** from the dropdown list of databases.

  b) This query applies a schema to the **raw-meter-daily** file which was uploaded in step 1c, then stores it as a table in the samples database which was created in step 3a.  NOTE: you must replace "[YOUR-BUCKET-NAME]" with the name of your S3 workshop bucket. 

```
CREATE EXTERNAL TABLE samples_db.raw_meter_table (
  lclid string, 
  day string,
  energy_median float,
  energy_mean float,
  energy_max float,
  energy_count bigint,
  energy_std float,
  energy_sum float,
  energy_min float
  )
ROW FORMAT DELIMITED 
  FIELDS TERMINATED BY ',' 
STORED AS INPUTFORMAT 
  'org.apache.hadoop.mapred.TextInputFormat' 
OUTPUTFORMAT 
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  's3://[YOUR-BUCKET-NAME]/workshop-data/synthetic-meter-master-data/'
TBLPROPERTIES (
  'classification'='csv', 
  'columnsOrdered'='true', 
  'compressionType'='none', 
  'delimiter'=',', 
  'skip.header.line.count'='1', 
  'typeOfData'='file');

```

  c) This query applies a schema to the **synthetic-grid-master-data** file which was uploaded in step 1c, then also stores it as a table named **grid_master_table** in the samples database.   NOTE: remember to replace "[YOUR-BUCKET-NAME]" with the name of your S3 workshop bucket. 

```
CREATE EXTERNAL TABLE samples_db.grid_master_table (
  block_id string, 
  servicetransformerid string,
  distributiontransformerid string,
  capacity string,
  substationid string,
  substation_name string,
  address string,
  lat_long string,
  lat float,
  long float,
  gridid string,
  grid_name string
  )
ROW FORMAT DELIMITED 
  FIELDS TERMINATED BY ',' 
STORED AS INPUTFORMAT 
  'org.apache.hadoop.mapred.TextInputFormat' 
OUTPUTFORMAT 
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  's3://[YOUR-BUCKET-NAME]/workshop-data/synthetic-meter-master-data/'
TBLPROPERTIES (
  'classification'='csv', 
  'columnsOrdered'='true', 
  'compressionType'='none', 
  'delimiter'=',', 
  'skip.header.line.count'='1', 
  'typeOfData'='file');

```

  c) This query applies a schema to the **synthetic-meter-master-data** which was uploaded in step 1c, then also stores it as a table named **meter_master_table** in the samples database.  NOTE: remember to replace "[YOUR-BUCKET-NAME]" with the name of your S3 workshop bucket. 


```
CREATE EXTERNAL TABLE samples_db.meter_master_table (
  meter_id string, 
  meter_type string)
ROW FORMAT DELIMITED 
  FIELDS TERMINATED BY ',' 
STORED AS INPUTFORMAT 
  'org.apache.hadoop.mapred.TextInputFormat' 
OUTPUTFORMAT 
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  's3://[YOUR-BUCKET-NAME]/workshop-data/synthetic-meter-master-data/'
TBLPROPERTIES (
  'classification'='csv', 
  'columnsOrdered'='true', 
  'compressionType'='none', 
  'delimiter'=',', 
  'skip.header.line.count'='1', 
  'typeOfData'='file');

```
  
  Now let's check to see the results of our work.  Navigate to the Glue console and select the 'Database' option on the left navigation bar.  You should see two new databases.  Select the 'sample_database' and you should see 3 new tables pointing to the sample data files you uploaded in step 1c above (Note: you may have to refresh your screen). These Glue structures were created automatically by the queries that you just executed.  (For more information on AWS Glue see Getting started with the [AWS Glue Data Catalog](https://docs.aws.amazon.com/glue/latest/dg/start-data-catalog.html). )

**Step 4: Set up Security and Configurations (parameters)**

 Even though this is a fully-serverless solution, you will need to establish some basic infrastructure and permissions.  A cloudformation template is provided to do that for you.  Download and examine the Dependency Stack cloudformation template.   

  a) Download and launch the following cloudformation template to build the resources.   Note: You do not need to change any of the pre-set parameters.

  ```
  https://amazon-forecast-samples.s3.us-west-2.amazonaws.com/ml_ops/workshop-dependency-stack.yaml
  ```

  b) Navigate to [CloudFormation console](https://us-west-2.console.aws.amazon.com/cloudformation) and select your desired deployment region.   Select "Create Stack, with new resources (Standard)".  For the template source, choose 'Upload a template file'.   Select the template you just downloaded. 

  Note: for more background on the purpose of this infrastructure, refer to the [MLOps dependency stack](https://github.com/aws-samples/amazon-forecast-samples/blob/main/ml_ops/docs/DependencyStack.md).

  Review the default Cloudformation parameters provided by the Cloudformation template.  

  ![Launch Cloudformation Stack](https://github.com/rgleave/proj2/blob/master/dependency-stack.png)
  
  c) For the stack name, enter any name (e.g.'dependency-stack').  
  
  d) Enter the name of the S3 bucket you created at the beginning of this workshop.

  e) Click through the default values of the remaining screens until you can submit the cloudformation stack build.

**Step 5: Prepare Raw Meter Data for Processing** 

Often raw meter data must be must be cleaned and validated before it can be used for forecasting.   An easy way to do that is by using Athena.  We will create an improved version of the raw data file and register it as a table in our Glue database.    This query will perform two data transformations:  add a neighborhood 'block_id', then adjust the dates for weather forecast simulation (Note: since this is relatively old data we want to bring it forward several years to within the supported date range of Amazon's weather index).
  
  a) Return to the Athena console.  Open the query window (you may need to select the "Editor" tab to find it).  Make sure you have selected **sample_database** from Database dropdown again.  Notice that you can now expand each of the existing tables on the left Data panel in order to see the table structures of each table.
  
  b) Copy the SQL statement below and run it in the Athena query window.  This statement creates a new Glue table containing a copy of the raw meter data which is structured to support weather forecasting and joining with other data for visualization.  

  NOTE: remember to replace "[YOUR-BUCKET-HERE]" with the name of your S3 workshop bucket. 

```
create table enhanced_raw_meter_table 
    WITH (
          external_location = 's3://[YOUR-BUCKET-HERE]/workshop-data/london-meter-data')
as SELECT regexp_extract("$path", '[ \w-]+?(?=\.)') as "block_id", lclid as "item_id", energy_sum as "target_value", date_add('year', 6, DATE(day)) as "timestamp" FROM "AwsDataCatalog"."samples_db"."raw_meter_table";

```

  Once the query has completed, navigate to the S3 console and note that a new sub-folder called 'london_meter_data' has been created within the 'workshop_data' folder.  Within that folder you will find a new data set.   Using the S3-Select operation (see instructions here) note the differences between this new data set and raw meter data that you uploaded in Step 1.   

  Also navigate to the Glue console and verify that the query created a Glue table called **london_meter_table** which points to the modified raw data file in S3.


**Recap:  What we Learned in this Module**

- How to automatically generate Glue databases and tables using Cloudformation.
- How to set up Athena
- How to use Athena to generate a refined data set from raw data and simultaneously register that as a new Glue table.

You are now ready to move on to the next module of this workshop.


## MODULE TWO: BUILDING A FORECAST PIPELINE


The previous module established refined data sets in preparation for building forecasts against that data.   In this module, we will create the pipeline infrastructure needed to load this data into Amazon forecast.   

**Step 1: Create a Forecast Folder**

For each new forecast that you choose to generate, you will first need to create a new folder to hold the output files generated by Amazon Forecast.   You are normally free to choose any name for a forecast folder.   For this workshop however, please use the folder names designated below. 

  a) Go to the S3 console and select the bucket you created in Step 1 of Module One. 
  b) Create a folder named 'daily-forecast' to hold the tts, rts, and item files created by the Forecast service.   Remember the name of this folder.   You will need it for the next step.

**Step 2: Create Infrastructure for Forecasting**

  a) Download the following cloudformation template to build the resources.

  ```
  https://amazon-forecast-samples.s3.us-west-2.amazonaws.com/ml_ops/workshop-forecast-pipeline.yaml
  ```

  b) Navigate to the Cloudformation console.  Select "Create Stack, with new resources (Standard)".  For the template source, choose 'Upload a template file'.   Select the template you just downloaded. 
  
  c) Review the default Cloudformation parameters provided by the Cloudformation template.  Each parameter controls some aspect of the Amazon Forecasting engine.   Pay particular attention to the schemas definitions.  We will refer to them again in a future step.   
  
  When you are ready, launch the template.   It should take about 5 minutes to create the forecasting pipeline and other necessary resources.
  
  Important Notes: 
  - for the stack name, use the same name that you chose for the folder in Step 1 of this module. 
  - for the <b>S3Bucket</b> parameter, enter the name of the bucket you created in Step 1 of Module One. 
  - for the Dataset Group name, also enter the name of the folder you created in Step 1 of this module. 
  

| Parameter | Recommended Value |
|--|--|
|Stack name|daily-forecast|
|DatasetGroupFrequencyRTS|D|
|DatasetGroupFrequencyTTS|D|
|DatasetGroupName|daily-forecast|
|DatasetIncludeItem|true|
|DatasetIncludeRTS|false|
|ForecastForecastTypes|["0.50", "0.60", "0.70", "0.80", "0.90"]|
|PredictorExplainPredictor| TRUE
|PredictorForecastDimensions |["block_id","location"]|
|PredictorForecastFrequency |D|
|PredictorForecastHorizon | 14|
|PredictorForecastOptimizationMetric| AverageWeightedQuantileLoss|
|PredictorForecastTypes | ["0.50", "0.60", "0.70", "0.80", "0.90"]|
|S3Bucket | {your bucket} (same as Dependency stack) |
|SNSEndpoint | {your email} |
|TimestampFormatRTS |yyyy-MM-dd|
|TimestampFormatTTS |yyyy-MM-dd|

This next set of values are multi-line and can be copied to your clipboard with the copy icon and pasted into the CloudFormation parameter.

<b>PredictorAttributeConfigs</b>
```
[
    {
      "AttributeName": "target_value",
      "Transformations": {
        "aggregation": "sum",
        "backfill": "nan",
        "frontfill": "none",
        "middlefill": "nan"
      }
    }
]
```   

<b>SchemaITEM</b>
```
{
    "Attributes": [
      {
        "AttributeName": "item_id",
        "AttributeType": "string"
      },
      {
        "AttributeName": "block_id",
        "AttributeType": "string"
      },
      {
        "AttributeName": "servicetransformerid",
        "AttributeType": "string"
      },
      {
        "AttributeName": "distributiontransformerid",
        "AttributeType": "string"
      },
      {
        "AttributeName": "substationid",
        "AttributeType": "string"
      },
      {
        "AttributeName": "substation_name",
        "AttributeType": "string"
      },
      {
        "AttributeName": "lat_long",
        "AttributeType": "string"
      },
      {
        "AttributeName": "grid_id",
        "AttributeType": "string"
      },
      {
        "AttributeName": "grid_name",
        "AttributeType": "string"
      }
    ]
}
```   

<b>SchemaRTS</b>  (Note: shown for illustration only; not required for this demo.)

```
{

    "Attributes": [
      {
        "AttributeName": "item_id",
        "AttributeType": "string"
      },
      {
        "AttributeName": "timestamp",
        "AttributeType": "timestamp"
      },
      {
        "AttributeName": "location",
        "AttributeType": "geolocation"
      }
    ]
}
```   

<b>SchemaTTS</b>
```
{
    "Attributes": [
      {
        "AttributeName": "item_id",
        "AttributeType": "string"
      },
      {
        "AttributeName": "target_value",
        "AttributeType": "float"
      },
      {
        "AttributeName": "timestamp",
        "AttributeType": "timestamp"
      },
      {
        "AttributeName": "location",
        "AttributeType": "geolocation"
      },
      {
        "AttributeName": "block_id",
        "AttributeType": "string"
      }
    ]
}
```  



**Step 2: Shaping Raw Data for Amazon Forecast**

 As part of Module One, you cleaned and enhanced the raw meter data.   Before generating a forecast, our pipeline needs to shape that raw data even more, tranforming it into the standardized formats required by Amazon Forecast (the schemas we defined in the previous **Step 1**).  The pipeline use Amazon Athena to perform the data shaping by executing specific SQL queries.   Since the pipeline runs unattended, we will store these SQL statements in system parameters so that the forecast pipeline can executed them automatically at the appropriate step in the process.  

Amazon Forecast has 3 pre-defined schemas for importing data into the service.  

- TTS file:
- Item file:
- RTS file:


We will use Athena to shape the raw data into a format that matches the shape of the TTS file as well as the ITEM file.  

Do the following:

a) Copy the SQL statement below and run it in the Athena query console to test that it is working properly.  

  Note: this query does several things:
  - reshapes the raw London meter data to conform to the TTS item schema
  - creates a block_id field in the file (important for next step)
  - bumps the dates forward 6 years, in order to fit a date range where AWS Forecast has weather history.

  ```
  SELECT a.item_id, a.target_value, a.timestamp, b.lat_long, a.block_id FROM "AwsDataCatalog"."sample_database"."london_meter_table" as a  left join "AwsDataCatalog"."sample_database"."grid_master_table" as b on a.block_id=b.block_id;
  ```

  Compare the query results to the structure of the TTS file in Step 1.

b) Copy the SQL statement above and paste it into the TTS query parameter in Parameter Store (see steps below) 

- In the AWS Console, search for "Parameter Store" and navigate to that service page.
- A list of all parameters is provided.  Type in 'query' in the search bar.  
- In the filtered list, select the <b>DatasetGroup/QueryTTS</b> parameter (e.g. /forecast/dailyforecast/DatasetGroup/QueryTTS) 
- Click the 'EDIT' button and replace the parameter's 'value' with the SQL statement in step 2a above.

c) Repeat the steps you followed above to shape an item file for the Forecast service.  Item files are optional, however we will shape one to illustrate the process.   Copy the SQL statement below and test it using Athena.

  ```
  SELECT a.item_id, b.servicetransformerid, b.distributiontransformerid, b.substationid, b.substation_name, b.lat_long, b.gridid, b.grid_name FROM "AwsDataCatalog"."sample_database"."london_meter_table" as a  left join "AwsDataCatalog"."sample_database"."grid_master_table" as b on a.block_id=b.block_id;
  ```

  If the query runs successfully, follow the steps in 2b above to save the query in the parameter named: /forecast/dailyforecast/DatasetGroup/QueryITEM.

  Once you are done, move on to the next module of this workshop.


## MODULE THREE: RUNNING YOUR FIRST PIPELINE


1. Run the 'Create-Dataset-Group' state machine which was created in the Step Functions service.  This creates your data set group in the AWS Forecast service.   Do the following:

  - In the AWS Console, search for "Step Functions" and navigate to that service page. 
  - Once in AWS Step Functions, a list of all state machines is provided.  Type the name of your StackName in the "Search for state machines" control to filter the list, if needed.
  - In the filtered list, one state machine is named <b>Create-Dataset-Group</b>.  Click on the link name to open this state machine.
  - Next, simply click Start Execution towards the upper-right of the screen.  
  - Click Start Execution on the secondary screen without changing anything.  Allow the state machine to run.  When all steps have turned green, it is complete.

2. Run the 'Workflow' state machine.  This will execute the complete end-to-end forecasting workflow, performing ETL to shape your raw data, creating and training a predictor, and generating a forecast. 

  Note:  As an alternative option (recommended if you wish to learn each sub-step in the workflows) resume the overall instruction set for MLOps [here](https://github.com/aws-samples/amazon-forecast-samples/blob/main/ml_ops/docs/UploadData.md).  Run them in the same order as the 'Workflow' state machine.

  - In the AWS Console, search for "Step Functions" and navigate to that service page. 
  - Once in AWS Step Functions, a list of all state machines is provided.  Type the name of your StackName in the "Search for state machines" control to filter the list, if needed.
  - In the filtered list, one state machine is named <b>Workflow</b>.  Click on the link name to open this state machine.
  - Next, simply click Start Execution towards the upper-right of the screen.  
  - Click Start Execution on the secondary screen without changing anything.  Allow the state machine to run.   This may take several hours to finish.  This workflow will 

3. When the last step of the forecasting pipeline is complete, your forecast data will be deposited in the folder you created for step 6 above.
     

## Conclusion
The steps above help you understand how produce a forecast on a sample dataset.  Please use overrides as a way to learn how to adapt data to your bespoke schema and use case.  If you have any questions, please reach out to your AWS Solutions Architect or account team.




## MODULE FOUR - CREATING VISUALIZATIONS AND DASHBOARDS FROM THE DATA

**Step 1: Generate export files for Amazon Quicksight**

a) Navigate to the Athena console and execute the following SQL statement to generate a raw data export table.  This statement will join the raw meter data with grid and meter metadata tables to create a single, fully-attributed export file.  

```
create table london_meter_table 
    WITH (
          external_location = 's3://energy-forecasts/workshop_data/london_meter_data')
as SELECT regexp_extract("$path", '[ \w-]+?(?=\.)') as "block_id", lclid as "item_id", energy_sum as "target_value", date_add('year', 6, DATE(day)) as "timestamp" FROM "AwsDataCatalog"."sample_database"."raw_meter_table"

```

b) Next, execute the following Athena SQL statement to generate a fully-attributed Forecast table by joining the forecast data with grid and meter metadata tables.

