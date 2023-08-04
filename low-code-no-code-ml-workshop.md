## Retail Forecasting Use Case

This example illustrates how to forecast retail product demand using Amazon Forecast.  The sample time series data contains sales transactions for various products sold at various locations.   This example illustrates how to forecast demand for any product, such as food, household items or virtually any other product. 

If you have not already done so, download the sample dataset by clicking [this link](https://static.us-east-1.prod.workshops.aws/public/216ef52a-92ed-4c56-84b6-a9bbd3ea7ef0/static/datasets/consumer_electronics.csv)  

Steps:

1. Complete the [MLOps dependency stack](https://github.com/aws-samples/amazon-forecast-samples/blob/main/ml_ops/docs/DependencyStack.md) prior to attemping the instructions below.  The MLOps dependency stack creates necessary underlying permissions.  This step only needs to occur once per AWS account.
2.  Once the MLOPs dependency stack is in place, navigate to [CloudFormation service](https://us-west-2.console.aws.amazon.com/cloudformation) and select your desired deployment region.
3.  Click the "Create Stack, with new resources (Standard)".
4.  Provide "retaildemo" as Stack Name and provide the following URL as the Amazon S3 URL.  You may [download the file](https://amazon-forecast-samples.s3.us-west-2.amazonaws.com/ml_ops/forecast-mlops-solution-guidance.yaml) locally or clone using git.

	 ```
     https://amazon-forecast-samples.s3.us-west-2.amazonaws.com/ml_ops/forecast-mlops-solution-guidance.yaml
     ```

5.  "Step 2: Specify stack details", several parameters are collected that define how the entire workload behaves.  Use these overrides.

| Parameter | Recommended Value |
|--|--|
|Stack name|retaildemo|
|DatasetGroupFrequencyRTS|W|
|DatasetGroupFrequencyTTS|W|
|DatasetGroupName|retaildemo|
|DatasetIncludeItem|false|
|DatasetIncludeRTS|true|
|ForecastForecastTypes|["0.50"]|
|PredictorExplainPredictor| TRUE
|PredictorForecastDimensions |["dept"]|
|PredictorForecastFrequency |W|
|PredictorForecastHorizon | 12|
|PredictorForecastOptimizationMetric| AverageWeightedQuantileLoss|
|PredictorForecastTypes | ["0.30", "0.40", "0.50", "0.60", "0.70"]|
|S3Bucket | {your bucket} |
|SNSEndpoint | {your email} |
|SchemaITEM| null |
|TimestampFormatRTS |yyyy-MM-dd|
|TimestampFormatTTS |yyyy-MM-dd|

These next set of values are multi-line and can be copied to your clipboard with the copy icon and pasted into the CloudFormation parameter.

<b>PredictorAttributeConfigs</b>
```
[
   {
      "AttributeName":"target_value",
      "Transformations":{
         "aggregation":"sum",
         "backfill":"nan",
         "frontfill":"none",
         "middlefill":"nan"
      }
   }
]
```   


<b>SchemaRTS</b>
```
{
   "Attributes":[
      {
         "AttributeName":"item_id",
         "AttributeType":"string"
      },
      {
         "AttributeName":"timestamp",
         "AttributeType":"timestamp"
      },
      {
         "AttributeName":"dept",
         "AttributeType":"string"
      },
      {
         "AttributeName":"temperature",
         "AttributeType":"float"
      },
      {
         "AttributeName":"fuel",
         "AttributeType":"float"
      },
      {
         "AttributeName":"markdown1",
         "AttributeType":"float"
      },
      {
         "AttributeName":"markdown2",
         "AttributeType":"float"
      },
      {
         "AttributeName":"markdown3",
         "AttributeType":"float"
      },
      {
         "AttributeName":"markdown4",
         "AttributeType":"float"
      },
      {
         "AttributeName":"markdown5",
         "AttributeType":"float"
      },
      {
         "AttributeName":"cpi",
         "AttributeType":"float"
      },
      {
         "AttributeName":"unemployment",
         "AttributeType":"float"
      },
      {
         "AttributeName":"holiday",
         "AttributeType":"float"
      }
   ]
}
```   

<b>SchemaTTS</b>
```
{
   "Attributes":[
      {
         "AttributeName":"item_id",
         "AttributeType":"string"
      },
      {
         "AttributeName":"timestamp",
         "AttributeType":"timestamp"
      },
      {
         "AttributeName":"dept",
         "AttributeType":"string"
      },
      {
         "AttributeName":"target_value",
         "AttributeType":"float"
      }
   ]
}
```   

6. Once you have prepared the data to conform to the shape to the RTS and TTS above, place the files in your S3 bucket, inside a child <b>retaildemo</b> folder.  Please note the S3 bucket and child stack folder that will contain the tts and rts folder should match the Stack Name and S3 bucket name from above.  Stated differently, if your Stack Name is abc123, the top-level folder in your S3 bucket should also be named abc123.

7. Resume the overall instruction set for MLOps [here](https://github.com/aws-samples/amazon-forecast-samples/blob/main/ml_ops/docs/UploadData.md).


## Conclusion

The steps above help you understand how to override the default MLOps toy dataset with an alternate Kaggle originated set.  Please use these overrides as a way to learn how to adapt data to your bespoke schema and use case.  If you have any questions, please reach out to your AWS Solutions Architect or account team.
