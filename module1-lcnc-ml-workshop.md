## Low-code No-code ML using Amazon Forecast

This example illustrates how to forecast retail product demand using Amazon Forecast.  The sample time series data contains sales transactions for various products sold at various locations.   This example illustrates how to forecast demand for any product, such as food, household items or virtually any other product. 

**BEFORE YOU BEGIN:** If you have not already done so, download the sample dataset by clicking [this link](https://static.us-east-1.prod.workshops.aws/public/216ef52a-92ed-4c56-84b6-a9bbd3ea7ef0/static/datasets/consumer_electronics.csv).


## Establish permissions for AWS Forecast 

1. In the AWS console, select the desired region for workload deployment.  The Amazon Forecast service is available in these [AWS Regions](https://docs.aws.amazon.com/general/latest/gr/forecast.html).  The region selector can be found, as a dropdown, right-of-center on the black menu bar in the AWS Console.  Choose the option that best meets your needs, but do take care to choose a region where AWS Forecast is available.
2.  From the AWS Console, navigate to the CloudFormation service.  You can do this by tying CloudFormation in the "search for services" control in the black menu bar.  Next, click the orange "Create Stack" button.
3. At stack creation, "[Step 1: Specify template](../images/create-solution-guidance-stack-1.jpg)", simply paste the URL string into the control as follows:

	 ```
     https://amazon-forecast-samples.s3.us-west-2.amazonaws.com/ml_ops/forecast-mlops-dependency.yaml
     ```

	If needed, you may [download the file](https://amazon-forecast-samples.s3.us-west-2.amazonaws.com/ml_ops/forecast-mlops-dependency.yaml) locally or clone using git.

4. Click Next to continue.
5. At stack creation, "[Step 2: Specify stack details](../images/create-dependency-stack-2.jpg)" complete page details as follows:
	
	 - [ ] Stack name should be forecast-mlops-dependency
	 - [ ] If you already have a S3 bucket for this purpose, choose "true" for ExistingS3Bucket.  If you need a new bucket, select false (default).
	 - [ ] Provide a valid and unique S3 bucket name.  To ensure a globally unique S3 bucket name, some customers append a portion of their AWS account number or random alpha numeric digits.
	 - [ ] Click the next button to continue.
6. At stack creation, "Step 3: Configure stack options", go to the bottom of the page and click next.  No options are required to be changed here.
7. At stack creation, "[Step 4: Review](../images/create-dependency-stack-4.jpg)", check the box at the bottom to accept there will be resources created as part of the stack creation, and click next.
8. Review the resources created, which may include your S3 bucket, but will include one supporting Lambda function and two Forecast-related IAM roles.   You can click on any of the Physical ID links to review their definition.<br><br>![CloudFormation Resource Review](../images/create-dependency-stack-resources.jpg)<br><br>



## Deploy the Low-code No-code Retail Forecasting Pipeline

1. Navigate to [CloudFormation service](https://us-west-2.console.aws.amazon.com/cloudformation) once again.
3.  Click the "Create Stack, with new resources (Standard)".
4.  Provide "forecasting-pipeline" as the Stack Name and provide the following URL as the Amazon S3 URL.  You may [download the file](https://amazon-forecast-samples.s3.us-west-2.amazonaws.com/ml_ops/forecast-mlops-solution-guidance.yaml) locally or clone using git.

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
