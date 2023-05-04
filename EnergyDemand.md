## Energy Demand Use Case

This example is based on the public [Smart meters in London](https://www.kaggle.com/datasets/jeanmidev/smart-meters-in-london?select=daily_dataset.csv) dataset on Kaggle that contains the energy consumption readings for a sample of 5,567 London Households that took part in the UK Power Networks led Low Carbon London project between November 2011 and February 2014.  In this scenario, the power company must anticipate how much energy at each location will be needed, so they can have enough to cover the demand.  As a sample dataset, the purpose of this data isn't to produce highly accurate insights; instead this is to demonstrate a workflow pattern so you can understand how to adapt your own data for use with Amazon Forecast.

Instead of following the [normal set of steps provided for the MLOps workflow](https://github.com/aws-samples/amazon-forecast-samples/tree/main/ml_ops), you may use this dataset as an override.

1. Complete the [MLOps dependency stack](https://github.com/aws-samples/amazon-forecast-samples/blob/main/ml_ops/docs/DependencyStack.md) prior to attemping the instructions below.  The MLOps dependency stack creates necessary underlying permissions.  This step only needs to occur once per AWS account.
2.  Once the MLOPs dependency stack is in place, navigate to [CloudFormation service](https://us-west-2.console.aws.amazon.com/cloudformation) and select your desired deployment region.
3.  Click the "Create Stack, with new resources (Standard)".
4.  Provide "energydemo" as Stack Name and provide the following URL as the Amazon S3 URL.  You may [download the file](https://amazon-forecast-samples.s3.us-west-2.amazonaws.com/ml_ops/forecast-mlops-energy-plus-weather.yaml) locally or clone using git.

	 ```
     https://amazon-forecast-samples.s3.us-west-2.amazonaws.com/ml_ops/forecast-mlops-energy-plus-weather.yaml
     ```

5.  "Specify stack details": several parameters are collected that define how the entire workload behaves.  Use these overrides (many are provided by the Cloudformation template).  NOTE: for the S3Bucket name, use the same name you provided for the in Step 1 above.  Also, please be aware that this template includes parameters to turn on the weather index.  You can examine the cloudformation template to understand which geolocation and time zone options were defaulted to turn on the weather index.

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
        "AttributeName": "substation",
        "AttributeType": "string"
      },
      {
        "AttributeName": "loop",
        "AttributeType": "string"
      },
      {
        "AttributeName": "feeder",
        "AttributeType": "string"
      },
      {
        "AttributeName": "power_line",
        "AttributeType": "string"
      },
      {
        "AttributeName": "steiner_node",
        "AttributeType": "string"
      }
    ]
}
```   

<b>SchemaRTS</b>
```
{

    "Attributes": [
      {
        "AttributeName": "item_id",
        "AttributeType": "string"
      },
      {
        "AttributeName": "some_attribute",
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
8. Download the daily meter data from the London dataset: [EnergyDemand.zip](https://amazon-forecast-samples.s3.us-west-2.amazonaws.com/ml_ops/EnergyDemand.zip) Unzip this file on your laptop and then upload the files from the <b>daily_dataset</b> raw data folder to the S3 folder you defined in Step 6. , inside a child <b>rawdata</b> folder. 
9. On the Glue console, create a database in the glue catalog and define a crawler to crawl the raw data folder.   For more information on how to do this, see Getting started with the [AWS Glue Data Catalog](https://docs.aws.amazon.com/glue/latest/dg/start-data-catalog.html).   Run the crawler.
10. Once the crawler creates the raw table in the Glue Data Catalog, use Athena to shape the raw data into a format that matehes the shape of the TTS file.   Use the following query as an example:

	 ```
     SELECT lclid as "item_id", energy_sum as "target_value",date_add('year', 6, DATE(day)) as "timestamp", '51.509865_-0.118092' as "location" FROM "AwsDataCatalog"."raw-data"."rawdata" order by location, item_id
     ```

10. (Option A: easiest) Run the 'Workflow' Step Function, which will execute the complete end-to-end forecasting workflow.   It will...   (Option B: to learn each sub-step in the workflows) Resume the overall instruction set for MLOps [here](https://github.com/aws-samples/amazon-forecast-samples/blob/main/ml_ops/docs/UploadData.md).  
     

## Conclusion

The steps above help you understand how produce a forecast on a sample dataset.  Please use overrides as a way to learn how to adapt data to your bespoke schema and use case.  If you have any questions, please reach out to your AWS Solutions Architect or account team.
