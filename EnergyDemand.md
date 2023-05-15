## Energy Demand Use Case

This example is based on the public [Smart meters in London](https://www.kaggle.com/datasets/jeanmidev/smart-meters-in-london?select=daily_dataset.csv) dataset on Kaggle that contains the energy consumption readings for a sample of 5,567 London Households that took part in the UK Power Networks led Low Carbon London project between November 2011 and February 2014.  In this scenario, the power company must anticipate how much energy at each location will be needed, so they can have enough to cover the demand.  As a sample dataset, the purpose of this data isn't to produce highly accurate insights; instead this is to demonstrate a workflow pattern so you can understand how to adapt your own data for use with Amazon Forecast.

Instead of following the [normal set of steps provided for the MLOps workflow](https://github.com/aws-samples/amazon-forecast-samples/tree/main/ml_ops), you may use this dataset as an override.

1. Complete the [MLOps dependency stack](https://github.com/aws-samples/amazon-forecast-samples/blob/main/ml_ops/docs/DependencyStack.md) prior to attemping the instructions below.  The MLOps dependency stack creates necessary underlying permissions.  This step only needs to occur once per AWS account.  Remember the name of the S3 bucket you created/chose.
2.  Once the MLOPs dependency stack is in place, navigate to [CloudFormation service](https://us-west-2.console.aws.amazon.com/cloudformation) and select your desired deployment region.
3.  Click the "Create Stack, with new resources (Standard)".  For the template source, choose Upload a template file.
4.  Provide "energydemo" as Stack Name and provide the following URL as the Amazon S3 URL.  You may [download the file](https://amazon-forecast-samples.s3.us-west-2.amazonaws.com/ml_ops/forecast-mlops-energy-plus-weather.yaml) locally or clone using git.

	 ```
     https://amazon-forecast-samples.s3.us-west-2.amazonaws.com/ml_ops/forecast-mlops-energy-plus-weather.yaml
     ```

5.  Specify stack details: several parameters are collected that define how the entire workload behaves.  Use these overrides (most are provided by the Cloudformation template).  NOTE: for the <b>S3Bucket</b> parameter, use the same name you provided for the in Step 1 above.  Also, please be aware that this template includes additional parameters to turn on the weather index.  You can examine the cloudformation template to understand which geolocation and time zone options were defaulted to turn on the weather index.

| Parameter | Recommended Value |
|--|--|
|Stack name|energydemo|
|DatasetGroupFrequencyRTS|D|
|DatasetGroupFrequencyTTS|D|
|DatasetGroupName|energydemo|
|DatasetIncludeItem|false|
|DatasetIncludeRTS|false|
|ForecastForecastTypes|["0.50"]|
|PredictorExplainPredictor| TRUE
|PredictorForecastDimensions |["location"]|
|PredictorForecastFrequency |D|
|PredictorForecastHorizon | 14|
|PredictorForecastOptimizationMetric| AverageWeightedQuantileLoss|
|PredictorForecastTypes | ["0.30", "0.40", "0.50", "0.60", "0.70"]|
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
      }
    ]
}
```  

6. Create another folder to hold the tts, rts, and item files used by the Forecast service.  This folder name should match the Stack Name and S3 bucket name from Step 5 above.  For example, if your Stack Name is abc123, the top-level folder in your S3 bucket should also be named abc123.
7. Create a <b>rawdata</b> folder to hold your raw data in the same S3 bucket you created and identified in the cloudformation stacks in Steps 1 and 5 above.  
8. Download the daily meter data from the London dataset: [Smart meters in London](https://www.kaggle.com/datasets/jeanmidev/smart-meters-in-london?select=daily_dataset.csv) dataset on Kaggle.  Unzip this file on your laptop and then upload the files from the <b>daily_dataset</b> raw data folder to the S3 folder you defined in Step 6. , inside a child <b>rawdata</b> folder. 
9. On the [Glue console](https://us-east-1.console.aws.amazon.com/glue/home?region=us-east-1#/v2/getting-started) , create a database in the glue catalog and define a crawler to crawl the raw data folder.   For more information on how to do this, see Getting started with the [AWS Glue Data Catalog](https://docs.aws.amazon.com/glue/latest/dg/start-data-catalog.html).   Run the crawler.
10. Once the crawler creates the raw table in the Glue Data Catalog, use Athena to shape the raw data into a format that matehes the shape of the TTS file.   Use the following query as an example:

	 ```
     SELECT lclid as "item_id", energy_sum as "target_value",date_add('year', 6, DATE(day)) as "timestamp", '51.509865_-0.118092' as "location" FROM "AwsDataCatalog"."raw-data"."rawdata" order by location, item_id
     ```
    Note: this query does several things:  1) it reshapes the raw London meter data to conform to the TTS item schema, 2) it establishes a location field (of type geolocation, with a lat/long format) and sets it to a valid value for London, 3) It bumps the dates forward 6 years, in order to fit a date range where AWS Forecast has weather history.

11. Copy the SQL statement above and paste it into the TTS query parameter in Parameter Store.  

  - In the AWS Console, search for "Parameter Store" and navigate to that service page.
  - A list of all parameters is provided.  Type in 'query' in the search bar.  
  - In the filtered list, select the <b>DatasetGroup/QueryTTS</b> parameter.   
  - Click the 'EDIT' button and replace the parameter's 'value' with the 
  - Allow the state machine to run.   This may take several hours to finish. 

12. Run the 'Create-Dataset-Group' state machine which was created in the Step Functions service.  This creates your data set group in the AWS Forecast service. 

  - In the AWS Console, search for "Step Functions" and navigate to that service page. 
  - Once in AWS Step Functions, a list of all state machines is provided.  Type the name of your StackName in the "Search for state machines" control to filter the list, if needed.
  - In the filtered list, one state machine is named <b>Create-Dataset-Group</b>.  Click on the link name to open this state machine.
  - Next, simply click Start Execution towards the upper-right of the screen.  
  - Click Start Execution on the secondary screen without changing anything.  Allow the state machine to run.  When all steps have turned green, it is complete.

13. Run the 'Workflow' state machine.  This will execute the complete end-to-end forecasting workflow, performing ETL to shape your raw data, creating and training a predictor, and generating a forecast.

  - In the AWS Console, search for "Step Functions" and navigate to that service page. 
  - Once in AWS Step Functions, a list of all state machines is provided.  Type the name of your StackName in the "Search for state machines" control to filter the list, if needed.
  - In the filtered list, one state machine is named <b>Workflow</b>.  Click on the link name to open this state machine.
  - Next, simply click Start Execution towards the upper-right of the screen.  
  - Click Start Execution on the secondary screen without changing anything.  Allow the state machine to run.   This may take several hours to finish.  This workflow will 

Note:  As an alternative option (recommended if you wish to learn each sub-step in the workflows) resume the overall instruction set for MLOps [here](https://github.com/aws-samples/amazon-forecast-samples/blob/main/ml_ops/docs/UploadData.md).  Run them in the same order as the 'Workflow' state machine.

13. When the last step of the forecasting pipeline is complete, your forecast data will be deposited in the folder you created for step 6 above.
     

## Conclusion
The steps above help you understand how produce a forecast on a sample dataset.  Please use overrides as a way to learn how to adapt data to your bespoke schema and use case.  If you have any questions, please reach out to your AWS Solutions Architect or account team.
