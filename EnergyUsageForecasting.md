## Energy Usage Forecasting Use Case

This example is based on a synthetic dataset created from an Advanced Metering Infrastructure (AMI) Head End System simulator. The data set contains two files. The first file contains the energy usage of 10,000 smart meters at 15-minute intervals. The second file contains the electric distribution network topology for the smart meters. In this use case, the power company forecasts the energy usage at each smart meter location.  As a sample dataset, the purpose of this data isn't to produce highly accurate insights; instead this is to demonstrate a workflow pattern so you can understand how to adapt your own data for use with Amazon Forecast.

Instead of following the [normal set of steps provided for the MLOps workflow](https://github.com/aws-samples/amazon-forecast-samples/tree/main/ml_ops), you may use this dataset as an override.

1. Complete the [MLOps dependency stack](https://github.com/aws-samples/amazon-forecast-samples/blob/main/ml_ops/docs/DependencyStack.md) prior to attemping the instructions below.  The MLOps dependency stack creates necessary underlying permissions.  This step only needs to occur once per AWS account.  Remember the name of the S3 bucket you created/chose.
2. Once the MLOPs dependency stack is in place, navigate to [CloudFormation service](https://us-west-2.console.aws.amazon.com/cloudformation) and select your desired deployment region.
3. Now you will launch a second stack to create the resources for your specific use-case.  Click the "Create Stack, with new resources (Standard)".  
4. Provide "energyusageforecasting" as Stack Name and provide the following URL as the Amazon S3 URL.  Or, instead you may [download the file](https://amazon-forecast-samples.s3.us-west-2.amazonaws.com/ml_ops/forecast-mlops-energy-plus-weather.yaml) locally and choose to upload the template file manually.

	 ```
     https://amazon-forecast-samples.s3.us-west-2.amazonaws.com/ml_ops/forecast-mlops-energy-plus-weather.yaml
     ```

5.  Specify stack details: several parameters are collected that define how the entire workload behaves.  Use these overrides (most are provided by the Cloudformation template).  NOTE: for the <b>S3Bucket</b> parameter, use the same name you provided for the in Step 1 above.  Also, please be aware that this template includes additional parameters to turn on the weather index.  You can examine the cloudformation template to understand which geolocation and time zone options were defaulted to turn on the weather index.

| Parameter | Recommended Value |
|--|--|
|Stack name|energyusageforecasting|
|DatasetGroupFrequencyRTS|D|
|DatasetGroupFrequencyTTS|==15min==|
|DatasetGroupName|energyusageforecasting
|DatasetIncludeItem|true|
|DatasetIncludeRTS|false|
|ForecastForecastTypes|["0.50"]|
|PredictorExplainPredictor| true
|PredictorForecastDimensions |["lat_long"]|
|PredictorForecastFrequency |<mark>15min</mark>|
|PredictorForecastHorizon | <mark>16</mark>|
|PredictorForecastOptimizationMetric| AverageWeightedQuantileLoss|
|PredictorForecastTypes | ["0.30", "0.40", "0.50", "0.60", "0.70"]|
|S3Bucket | <mark>{your bucket}</mark> (same as Dependency stack) |
|SNSEndpoint | <mark>{your email}</mark> |
|TimestampFormatRTS |yyyy-MM-dd|
|TimestampFormatTTS |<mark>yyyy-MM-dd HH:mm:ss</mark>|

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
            "AttributeName": "locationid",
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
            "AttributeName": "capacity",
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
            "AttributeName": "gridid",
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
            "AttributeName": "reading_type",
            "AttributeType": "string"
        },
        {
            "AttributeName": "lat_long",
            "AttributeType": "geolocation"
        }
    ]
}
```  

6. In the <b>S3Bucket</b> you created in Step 1 above, create a folder to hold the tts, rts, and item files used by the Forecast service.  This folder name should match the Stack Name and Dataset Name from Step 5 above.  For example, since your Stack Name is "energyusageforecasting", the top-level folder in your S3 bucket should also be named "energyusageforecasting".
7. In the <b>S3Bucket</b> you created in Step 1 above, create a <b>"rawdata"</b> folder to hold your raw data.  
8. Download the 'smart_meter_data.zip' file contained in this folder in the repository, unzip it on your laptop, and then upload the folders <b>energyusage</b> and <b>topology</b> from the <b>smart_meter_data</b> to the <b>rawdata</b> subfolder created in step 7. 
9. On the [Glue console](https://us-east-1.console.aws.amazon.com/glue/home?region=us-east-1#/v2/getting-started) , create a database in the glue catalog and define a crawler to crawl the raw data folder.   For more information on how to do this, see Getting started with the [AWS Glue Data Catalog](https://docs.aws.amazon.com/glue/latest/dg/start-data-catalog.html).   Run the crawler.
10. Once the crawler creates the raw table in the Glue Data Catalog, use Athena to create a query to shape the raw data into a format that matehes the shape of the TTS file.  Since the synthetic data sample you uploaded was already shaped correctly, you can use the following simple query as an example:

	 ```
     SELECT * FROM "AwsDataCatalog"."rawdata"."energyusage";
     ```


11. Copy the SQL statement above and paste it into the TTS query parameter in Parameter Store.  

  - In the AWS Console, search for "Parameter Store" and navigate to that service page.
  - A list of all parameters is provided.  Type in 'query' in the search bar.  
  - In the filtered list, select the <b>DatasetGroup/QueryTTS</b> parameter.   
  - Click the 'EDIT' button and replace the parameter's 'value' with the SQL statement from step 10.
  - Save the parameter.

12. Next, use Athena to shape the raw data into a format that matches the shape of the ITEM file.  Once again, since the sample data is already shaped correctly, use the following simple query as an example:

	 ```
     SELECT * FROM "AwsDataCatalog"."rawdata"."topology";
     ```


13. Copy the SQL statement above and paste it into the ITEM query parameter in Parameter Store.  

  - In the AWS Console, search for "Parameter Store" and navigate to that service page.
  - A list of all parameters is provided.  Type in 'query' in the search bar.  
  - In the filtered list, select the <b>DatasetGroup/QueryITEM</b> parameter.   
  - Click the 'EDIT' button and replace the parameter's 'value' with the SQL statement from step 12.
  - Save the parameter.

14. Run the 'Create-Dataset-Group' state machine which was created in the Step Functions service.  This creates your data set group in the AWS Forecast service. 

  - In the AWS Console, search for "Step Functions" and navigate to that service page. 
  - Once in AWS Step Functions, a list of all state machines is provided.  Type the name of your StackName in the "Search for state machines" control to filter the list, if needed.
  - In the filtered list, one state machine is named <b>Create-Dataset-Group</b>.  Click on the link name to open this state machine.
  - Next, simply click Start Execution towards the upper-right of the screen.  
  - Click Start Execution on the secondary screen without changing anything.  Allow the state machine to run.  When all steps have turned green, it is complete.

15. Run the 'Workflow' state machine.  This will execute the complete end-to-end forecasting workflow, performing ETL to shape your raw data, creating and training a predictor, and generating a forecast.

  - In the AWS Console, search for "Step Functions" and navigate to that service page. 
  - Once in AWS Step Functions, a list of all state machines is provided.  Type the name of your StackName in the "Search for state machines" control to filter the list, if needed.
  - In the filtered list, one state machine is named <b>Workflow</b>.  Click on the link name to open this state machine.
  - Next, simply click Start Execution towards the upper-right of the screen.  
  - Click Start Execution on the secondary screen without changing anything.  Allow the state machine to run.   This may take several hours to finish.  This workflow will 

Note:  As an alternative option (recommended if you wish to learn each sub-step in the workflows) resume the overall instruction set for MLOps [here](https://github.com/aws-samples/amazon-forecast-samples/blob/main/ml_ops/docs/UploadData.md).  Run them in the same order as the 'Workflow' state machine.

16. When the last step of the forecasting pipeline is complete, your forecast data will be deposited in the folder you created for step 6 above.
     

## Conclusion
The steps above help you understand how produce a forecast on a sample dataset.  Please use overrides as a way to learn how to adapt data to your bespoke schema and use case.  If you have any questions, please reach out to your AWS Solutions Architect or account team.
