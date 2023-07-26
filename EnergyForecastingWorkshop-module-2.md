## MODULE TWO: BUILDING A FORECAST PIPELINE


In the previous module, we created enhanced data sets from the actual raw meter data, in preparation for building forecasts.   In this module, we will create the pipeline infrastructure needed to load this data into Amazon forecast.   

**Step 1: Setup a Forecasting Project**

The first step in starting a new forecasting project is to create a folder to hold the input files which are loaded into Amazon Forecast as well as the forecast data that the service generates.

  a) Go to the S3 console and select the bucket you created in Module One. 
  b) Create a project folder (e.g. 'daily-forecast') to hold the tts, rts, and item files created by the Forecast service.  Remember the name of this project folder for the following steps.   

**Step 2: Create Infrastructure for Forecasting**

  a) Download the following cloudformation template to build the resources.

  ```
  https://amazon-forecast-samples.s3.us-west-2.amazonaws.com/ml_ops/workshop-forecast-pipeline.yaml
  ```

  b) Navigate to the Cloudformation console.  Select "Create Stack, with new resources".  For the template source, choose 'Upload a template file'.   Select the template that you just downloaded. 
  
  c) Review the default Cloudformation parameters provided by the Cloudformation template.  Each parameter controls some aspect of the Amazon Forecasting engine. Pay particular attention to the schemas definitions.  We will refer to them again in a future step.   
  
  When you are ready, launch the template.   
  
  Important Notes: 
  - for the stack name, use the same name that you chose for the folder in Step 1 of this module. 
  - for the <b>S3Bucket</b> parameter, enter the name of the bucket you created in Step 1 of Module One. 
  - for the Dataset Group name, also enter the name of the folder you created in Step 1 of this module. 
  - it should take about 5 minutes to create the forecasting pipeline and other necessary resources.

| Parameter | Recommended Value |
|--|--|
|Stack name|[YOUR PROJECT FOLDER NAME]|
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

a) Copy the SQL statement below and run it in the Athena console to test that it is working properly.  

  Note: this query does several things:
  - reshapes the raw London meter data to conform to the TTS item schema
  - creates a block_id field in the file (important for next step)
  - bumps the dates forward 6 years, in order to fit a date range where AWS Forecast has weather history.

  ```
  SELECT a.item_id, a.target_value, a.timestamp, b.lat_long, a.block_id FROM "AwsDataCatalog"."samples_db"."london_meter_table" as a  left join "AwsDataCatalog"."samples_db"."grid_master_table" as b on a.block_id=b.block_id;
  ```

  Compare the query results to the structure of the TTS file in Step 1.

b) Copy the SQL statement above and paste it into the TTS query parameter in Parameter Store (see steps below) 

- In the AWS Console, search for "Parameter Store" and navigate to that service page.
- A list of all parameters is provided.  Type in 'query' in the search bar.  
- In the filtered list, select the <b>DatasetGroup/QueryTTS</b> parameter (e.g. /forecast/dailyforecast/DatasetGroup/QueryTTS) 
- Click the 'EDIT' button and replace the parameter's 'value' with the SQL statement in step 2a above.

c) Repeat the steps you followed above to shape an item file for the Forecast service.  Item files are optional, however we will shape one to illustrate the process.   Copy the SQL statement below and test it using Athena.

  ```
  SELECT a.item_id, b.servicetransformerid, b.distributiontransformerid, b.substationid, b.substation_name, b.lat_long, b.gridid, b.grid_name FROM "AwsDataCatalog"."samples_db"."enhanced_raw_meter_table" as a  left join "AwsDataCatalog"."samples_db"."grid_master_table" as b on a.block_id=b.block_id;
  ```

  If the query runs successfully, follow the steps in 2b above to save the query in the parameter named: /forecast/dailyforecast/DatasetGroup/QueryITEM.

  **Recap:  What we Learned in this Module**

- How to set up a new forecasting project.
- Requirements for integrating data into Amazon Forecast.
- How to build a low-code forecasting pipeline for your project using Cloudformation.
- How to write Athena queries to shape raw data into the standard file formats required by Amazon Forecast for all input data.
- How to register those shaping queries into the automated pipeline.
- How to alter and save run-time parameters for the forecasting pipeline.

You are now ready to move on to the next module of this workshop.
