## MODULE ONE:  Setting up the Workshop Environment

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

  b) Navigate to the Athena console.  At the top of the Athena screen you may see a dialog box that says:  "Before you run your first query, you need to set up a query result location in Amazon S3".  If so, click the "Edit Settings" button on the dialog box.  (NOTE: if you do not see the dialog box, simply select the "Settings" tab)

  ![Athena console - setting up a location for query results](https://github.com/rgleave/proj2/blob/master/workshop-athena-query-screen.png)
  
  You should now see the "Manage Settings" screen.  Click the "Browse S3" button.  Find the new folder you created in Step 3a and choose it.  Then, press the "Save" button to finish this setup.

  ![Manage Athena settings](https://github.com/rgleave/proj2/blob/master/workshop-manage-settings-bucket.png)

  Now we are ready to run SQL statements.

**Step 3: Establishing Glue Datbases and Tables**

  Before you can interact with raw text data, you must apply a structure (schema) over the data set to allow Athena to execute structured queries.  This process is called cataloging and generates databases and tables in the AWS Glue Catalog.  There are several ways to create databases and tables, however in this workshop we will use Athena to do so.  Navigate to the Athena console, and select the Editor tab.  Select  and run the following queries:

  a) The first query creates a database to serve as a catalog to hold our sample data tables.

```
create database samples_db;

```
  Once the query completes successfully, look at the **Data Pane** on the left side of the screen.  You will see a **Database** window. Select **samples_db** from the dropdown list of databases.

  b) The second query applies a schema to the **raw-meter-daily** file which was uploaded in step 1c, then stores it as a table in the samples database which was created in step 3a.  NOTE: you must replace "[YOUR-BUCKET-NAME]" with the name of your S3 workshop bucket. 

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
  's3://[YOUR-BUCKET-NAME]/workshop-data/raw-meter-daily/'
TBLPROPERTIES (
  'classification'='csv', 
  'columnsOrdered'='true', 
  'compressionType'='none', 
  'delimiter'=',', 
  'skip.header.line.count'='1', 
  'typeOfData'='file');

```

  Once the query completes you can examine the contents of the table by finding the new table listed on the left side of the screen, then pressing the menu button (3 dots).   Select the **Preview Table** option to see a small sample of the data.   This tables contains the raw meter data which was collected from London households during the research study mentioned previously.

  c) The third query applies a schema to the **synthetic-grid-master-data** file which was uploaded in step 1c, then also stores it as a table named **grid_master_table** in the samples database.   NOTE: remember to replace "[YOUR-BUCKET-NAME]" with the name of your S3 workshop bucket. 

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
  's3://[YOUR-BUCKET-NAME]/workshop-data/synthetic-grid-master-data/'
TBLPROPERTIES (
  'classification'='csv', 
  'columnsOrdered'='true', 
  'compressionType'='none', 
  'delimiter'=',', 
  'skip.header.line.count'='1', 
  'typeOfData'='file');

```

  After running the query, preview the contents of this table as well.   This is synthetic metadata, designed to simulate information which might relate to elements of an electric grid.   It is included purely to simulate what live grid data might look like.


  c) The final query applies a schema to the **synthetic-meter-master-data** which was uploaded in step 1c, then also stores it as a table named **meter_master_table** in the samples database.  NOTE: remember to replace "[YOUR-BUCKET-NAME]" with the name of your S3 workshop bucket. 


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

**Step 4: Set up Security and Forecasting Functions **

 Even though this is a low-code, serverless solution you will need to establish some basic infrastructure like permissions (IAM Roles) and reusable forecasting functions (Lambda).  A cloudformation template is provided to do that for you.  Download and examine the Dependency Stack cloudformation template below.   

  a) Download and launch the following cloudformation template to build the resources.   Note: You do not need to change any of the pre-set parameters.

  ```
  https://amazon-forecast-samples.s3.us-west-2.amazonaws.com/ml_ops/workshop-dependency-stack.yaml
  ```

  b) To launch the template, first navigate to [CloudFormation console](https://us-west-2.console.aws.amazon.com/cloudformation) and select your desired deployment region.   Next, select "Create Stack, with new resources (Standard)".  For the template source, choose 'Upload a template file'.   Select the template (yaml file) that you just downloaded. 

  Note: for more infomation about the purpose of this infrastructure, refer to the [MLOps dependency stack](https://github.com/aws-samples/amazon-forecast-samples/blob/main/ml_ops/docs/DependencyStack.md).

  Review the default Cloudformation parameters provided by the Cloudformation template.  

  ![Launch Cloudformation Stack](https://github.com/rgleave/proj2/blob/master/dependency-stack.png)
  
  c) For the stack name, enter any name (e.g.'dependency-stack').  
  
  d) Enter the name of the S3 bucket you created at the beginning of this workshop.

  e) Click through the default values of the remaining screens until you can submit the cloudformation stack build.

  Once the cloudformation stack build finishes, search for the Lambda service at the top of the console page.   On the Lambda console you should see several Lambda helper functions which were created by Cloudformation.  

  ![Lambda helper functions](https://github.com/rgleave/proj2/blob/master/workshop-lambda-helper-functions.png)


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

- How to set up Athena
- How to automatically generate Glue databases and tables using Athena.
- How to build infrastructure (IAM roles and Lamda Functions) using Cloud Formation.
- How to use Athena to generate a refined data set from raw data, then register it as a new Glue table.

You are now ready to move on to the next module of this workshop.
