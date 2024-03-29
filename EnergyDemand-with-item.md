﻿## MODULE ONE:  Setting up the Workshop Environment

This example is based on the public [SmartMeter Energy Consumption Data in London Households](https://data.london.gov.uk/dataset/smartmeter-energy-use-data-in-london-households) dataset that contains the smart meter readings from a sample of 5,567 London Households that took part in the UK Power Networks led Low Carbon London project between November 2011 and February 2014.  In this scenario, the power company must anticipate how much energy at each location will be needed, so they can have enough to cover the demand.  As a sample dataset, the purpose of this data isn't to produce highly accurate insights; instead this is to demonstrate a workflow pattern so you can understand how to adapt your own data for use with Amazon Forecast.

For background on building automated pipelines for Amazon Forecast, refer to the [normal set of steps provided for the MLOps workflow](https://github.com/aws-samples/amazon-forecast-samples/tree/main/ml_ops) samples here.

First you need to prepare the environment to hold and process the sample data.    Follow the steps below:

**Step 1: Getting data for the Workshop**

  a) Create an S3 bucket to hold your data (e.g. 'energy-forecasts').  (Note: remember the name of this S3 bucket. You will need it for subsequest steps).  

  b) Upload the sample data for the workshop to your bucket.  (Note: you should have previously downloaded the sample data set onto your local machine.)  Find the folder named 'workshop-data' on your local machine and upload it into the S3 'energy-forecasts' bucket.   
  
  When the upload finishes, explore this folder.  You should find the three subfolders listed below: 
    **- raw-meter-daily**   This contains the actual raw data collected from London smart meters.
    **- synthetic-grid-master-data**   Synthetic metadata about the grid to demonstrate more advanced use cases on the sample dashboards.
    **- synthetic-meter-master-data**   More synthetic metadata about the meters to demonstrate additional use cases.

**Step 2: Building Foundational Infrastructure for the Workshop**

  Note: Even though this is a fully-serverless solution, you will need to set up some basic security roles and system parameters.  The Workshop Dependency Stack is created by launching a cloudformation stack which builds basic infrastructure and permissions needed for the workshop, as well as some Glue databases and tables.  You do not need to change any of the pre-set parameters.   (Note: for more background on the purpose of this infrastructure, refer to the [MLOps dependency stack](https://github.com/aws-samples/amazon-forecast-samples/blob/main/ml_ops/docs/DependencyStack.md).

  a)  Navigate to [CloudFormation service](https://us-west-2.console.aws.amazon.com/cloudformation) and select your desired deployment region.  Launch the following cloudformation template to build the resources.

	  ```
     https://amazon-forecast-samples.s3.us-west-2.amazonaws.com/workshop-dependency-stack.yaml

    ```

  b) Once the cloudformation script completes successfully, navigate to the Glue console and select the 'Database' option on the left navigation bar.  You should see a new database called 'sample_database'.  Select it.  Within that database, you should see 3 new tables pointing to the sample data files you uploaded in step 1c above.  (Note: For more information on how to do this, see Getting started with the [AWS Glue Data Catalog](https://docs.aws.amazon.com/glue/latest/dg/start-data-catalog.html). )

**Step 3: Preparing Raw Meter Data for Processing** 
This is done by executing a SQL statement using Athena, Amazon's serverless query engine.   

  a) First, navigate to the Athena console and make sure you have selected 'sample_database' from Database dropdown (left side of screen).

  a) Copy the SQL statement below and run it in the query window.  It creates a new copy of the raw meter data in a new Glue table which is opmtimized for weather forecasting and also for data visualization.  

  	 ```
create table london_meter_table 
    WITH (
          external_location = 's3://energy-forecasts/workshop_data/london_meter_data')
as SELECT regexp_extract("$path", '[ \w-]+?(?=\.)') as "block_id", lclid as "item_id", energy_sum as "target_value", date_add('year', 6, DATE(day)) as "timestamp" FROM "AwsDataCatalog"."sample_database"."raw_meter_table"
     ```

  Once the query has completed, navigate to the S3 console and note that a new sub-folder called 'london_meter_data' has been created within the 'workshop_data' folder.  Within that folder you will find a new data set.   Using the S3-Select operation (see instructions here) note the differences between this new data set and raw meter data that you uploaded in step 1c.   



Recap:  What we learned in this module.

- How to automatically generate Glue databases and tables using Cloudformation.
- How to use Athena to generate a refined data set from raw data and simultaneously register that as a new Glue tables.






## MODULE TWO: - BUILDING A FORECAST PIPELINE


The previous module established a refined data environment with database and table entries in the Glue Data Catalog as preparation for building forecasts against that data.   In this module, we will instantiate the infrastructure needed to load this data into Amazon forecast.   

**Step 1: Building the Infrastructure**

  a) Navigate to the Cloudformation console.

  b) Select "Create Stack, with new resources (Standard)".  For the template source, choose Upload a template file.

  c) Upload and run the following cloudformation file (https://amazon-forecast-samples.s3.us-west-2.amazonaws.com/ml_ops/forecast-mlops-energy-plus-weather.yaml).

	 ```
     https://amazon-forecast-samples.s3.us-west-2.amazonaws.com/ml_ops/forecast-mlops-energy-plus-weather.yaml
     ```



  d) Specify stack details: several parameters are collected that define how the entire workload behaves.  Use these overrides (most are provided by the Cloudformation template).  NOTE: for the <b>S3Bucket</b> parameter, use the same name you provided for the in Step 1 above.  Also, please be aware that this template includes additional parameters to turn on the weather index.  You can examine the cloudformation template to understand which geolocation and time zone options were defaulted to turn on the weather index.

| Parameter | Recommended Value |
|--|--|
|Stack name|energydemo|
|DatasetGroupFrequencyRTS|D|
|DatasetGroupFrequencyTTS|D|
|DatasetGroupName|energydemo|
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

These next set of values are multi-line and can be copied to your clipboard with the copy icon and pasted into the CloudFormation parameter.

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

**Step 4: Shape the Data for Amazon Forecast**

  a) Create another folder to hold the tts, rts, and item files used by the Forecast service.  This folder name should match the Stack Name and S3 bucket name from Step 5 above.  For example, if your Stack Name is abc123, the top-level folder in your S3 bucket should also be named abc123.

  b) Next we will use Athena to shape the raw data into a format that matches the shape of the TTS file.  Copy the SQL statement below and run it in the Athena query console to test that it is working properly.  

    Note: this query does several things:  1) reshapes the raw London meter data to conform to the TTS item schema, 2) creates a block_id field in the file (important for next step), 3) bumps the dates forward 6 years, in order to fit a date range where AWS Forecast has weather history.

	 ```
    SELECT a.item_id, a.target_value, a.timestamp, b.lat_long FROM "AwsDataCatalog"."sample_database"."london_meter_table" as a  left join "AwsDataCatalog"."sample_database"."grid_master_table" as b on a.block_id=b.block_id;
     ```

  c) Copy the SQL statement above and paste it into the TTS query parameter in Parameter Store.  

  - In the AWS Console, search for "Parameter Store" and navigate to that service page.
  - A list of all parameters is provided.  Type in 'query' in the search bar.  
  - In the filtered list, select the <b>DatasetGroup/QueryTTS</b> parameter.   
  - Click the 'EDIT' button and replace the parameter's 'value' with the 
  - Allow the state machine to run.   This may take several hours to finish. 

13.  Repeat the process above to copy the SQL statement below and paste it into the QueryITEM query parameter in Parameter Store.  

	 ```
      SELECT a.item_id, b.servicetransformerid, b.distributiontransformerid, b.substationid, b.substation_name, b.lat_long, b.grid_id, b.grid_name FROM "AwsDataCatalog"."raw-data"."london_meter_data_with_block" as a  left join "AwsDataCatalog"."raw-data"."london_meter_info_cleaned" as b on a.block_id=b.block_id;
     ```




## MODULE THREE -- RUNNING YOUR FIRST PIPELINE


1. Run the 'Create-Dataset-Group' state machine which was created in the Step Functions service.  This creates your data set group in the AWS Forecast service. 

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