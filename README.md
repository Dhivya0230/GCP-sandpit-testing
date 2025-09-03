# **GCP-sandpit-testing**
This is just for creating source management using GCP Sandpit testing

## Part 1: Create a data pipeline between GCS to bigquery using bigquery external tables all created via dataform

In this task, we are going to create a datapipeline using Dataform which access the sample file in a GCS bucket using bigquery external table.
```
Pipeline overview: 
GCS CSV File (gs://my-sandpit-data/emp_city.csv)
            │
            ▼
  BigQuery External Table (employees.emp_details)
            │
            ▼
       Dataform transformations
```
- External tables are a read only reference to the GCS CSV.
- Dataform operations: Ensures the table exists and can be tracked in the DAG.

Requirements for this task:
- GCS Bucket containing the CSV (gs://my-sandpit-data/emp_city.csv).
- Must be accessible by the BigQuery service account.
- BigQuery Dataset (employees) already exists.
- Dataform Project initialized and connected to the target BigQuery project.
- Proper IAM permissions:
- roles/bigquery.dataEditor or roles/bigquery.admin for table creation.
- GCS read access for BigQuery.

In Dataform create a workspace and under definitions, create a new .sqlx file for creating an external table in bigquery pointing to a GCS file and add the below code to the .sqlx file and execute it.

```
config { 
    type: "operations",
    tags: ["external"],
    schema: "employees",
    hasOutput: true
}

CREATE OR REPLACE EXTERNAL TABLE `dhivya-sandpit.employees.emp_details` (
    name STRING,
    age INT64,
    city STRING
)
OPTIONS (
    format = 'CSV',
    uris = ['gs://my-sandpit-data/emp_city.csv'],
    field_delimiter = ",",
    ignore_unknown_values = true,
    skip_leading_rows = 1
);
```
Explanation of each section of the code:
```
config { 
    type: "operations",
    tags: ["external"],
    schema: "employees",
    hasOutput: true
}
```
- **type:** "operations" – Used for DDL statements (CREATE TABLE, GRANT, etc.) rather than SELECT-based tables.
- **tags:** Helps organize operations in Dataform DAG or documentation.
- **hasOutput:** true – Signals that this operation produces a table in BigQuery.

- **CREATE OR REPLACE EXTERNAL TABLE** defines a read only table that points to CSV files in GCS.
````
OPTIONS (
    format = 'CSV',
    uris = ['gs://my-sandpit-data/emp_city.csv'],
    field_delimiter = ",",
    ignore_unknown_values = true,
    skip_leading_rows = 1
);
````
- format is to mention what format the file in GCS is stored as example: ".CSV"
- uris is the specify the path to the GCS file.
- field_delimiter - Colim seperator
- skip_leading_rows - Skips the header row in the CSV.
- ingnore_unkown_values - is to ensure extra columns in CSV does not break the table.
  
Go to bigquery UI and under the employees dataset, there will be a table named emp_details in which you can query your results.
Example query code:

```sql
SELECT *
FROM `dhivya-sandpit.employees.emp_details`
ORDER BY name
LIMIT 10
```

## Part 2: Create a data pipeline between pub/sub topics, pub/sub subscriptions to Bigquery (BQ tables to be created via dataform) Confirm messages published to pub/sub appear in Bigquery

Components:

1. Pub/Sub Topic (my-topic)
    - Source of events/messages.
2. Pub/Sub Subscription (my-bq-sub)
    - Direct BigQuery sink subscription that writes messages into a table.
    - Can be append-only or schema-aware.
3. BigQuery Table (my_dataset.data_pubsub_event)
    - Stores all events in a structured format

Step 1:

Create a pub/sub Topic (my-topic).

Step 2: Create a Bigquery table via dataform:

Create a new .sqlx file under definitions and use the below code to create a new table (assuming we know the pubsub message structure)

```
config {
  type: "table",
  schema: "data_pubsub_event"  
}

 -- Initial schema to match Pub/Sub messages
SELECT
  "" AS event_id,
  TIMESTAMP(NULL) AS event_time,
  "" AS message
FROM UNNEST([])
```
**UNNEST([])** produces zero rows, so the table is created with the correct schema but no data.

If we dont use UNNEST, there will be fake null rows created.

Execure this in Dataform.

Step 3: In cloud shell create a bigquery sink subscription using the following commands.

```
gcloud pubsub subscriptions create my-bq-sub \
  --topic=my-topic \
  --bigquery-table=dhivya-sandpit:data_pubsub_event.part2_pubsub_BQ \
  --use-table-schema
```
Ensure the pubsub service account has required IAM permissions like Bigquery data editor access.

Step 4: Now we can validate by passing a test message in pub/sub and check if it appears in Bigquery.
Write the below line in cloud shell to pass a message to the topic.
```
gcloud pubsub topics publish my-topic \
  --message '{"event_id":"123","event_time":"2025-09-03T14:00:00Z","message":"Hello Pub/Sub!"}'
```
To see if the bigquery sink subscription has received the message, use the following code to query the BQ table

```
SELECT *
FROM `dhivya-sandpit.data_pubsub_event.part2_pubsub_BQ`
ORDER BY event_time DESC
```
The message sent by the topic will be displayed in the BQ table.

## Part 3: What happens when Dataform creates a table and loads it via GCS external table while it is also targetted by a Bigquery Pub/sub subscription

In this situation, we are trying to create an external table in bigquery via dataform which is pointing to a GCS file and the same table is connected as a bigquery sink subscription in pub/sub.

Since dataform is creating a metadata reference to GCS file (no data is written in BQ, just displayed), and when pub/sub streaming inserts is trying to write the messages in the same table, bigquery will not allow streaming writes to an external table. you will get an error saying "Streaming buffer is not supported for external tables"

External tables are read only

To avoid this conflict, we can create a raw table to store the Pub/Sub messages and the GCS ingestion can be stored as a Staging table and in dataform we can configure the final table type to "Incremental" and ensure it only appends/merges the table and not re-crete it.

To do this, we must also ensure the schema bet ween Pub/Sub and Dataform is exactly aligned.
