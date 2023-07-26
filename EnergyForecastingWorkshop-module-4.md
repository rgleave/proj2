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









  **Recap:  What we Learned in this Module**

- How to ...
- How to ...
- How to ...
- How to ...
- How to ...
- How to ...

You have now completed all modules of this workshop.