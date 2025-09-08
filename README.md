# **GCP-sandpit-testing**
This is just for creating source management using GCP Sandpit testing

## Part 1: Create a data pipeline between GCS to bigquery using bigquery external tables all created via dataform

In this task, we are going to create a datapipeline using Dataform which access the sample file in a GCS bucket using bigquery external table.
```
Pipeline overview: 
GCS CSV File (gs://my-sandpit-data/emp_city.csv)
            │
            ▼
  BigQuery External Table (employees.emp_details_raw)
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
  
![task1 (1)](https://github.com/user-attachments/assets/5de2a544-bbe5-4a2c-81a8-7ad4b76735b9)


In Dataform create a workspace and under definitions, create a new .sqlx file for creating an external table in bigquery pointing to a GCS file and add the below code to the .sqlx file and execute it.

```
config { 
    type: "operations",
    tags: ["external"],
    schema: "employees",
    hasOutput: true
}

CREATE OR REPLACE EXTERNAL TABLE `dhivya-sandpit.employees.emp_details_raw` (
    emp_id INT64 NOT NULL,
    name STRING,
    dept STRING,
    hire_date STRING,
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
FROM `dhivya-sandpit.employees.emp_details_raw`
ORDER BY emp_id
LIMIT 10
```

## Part 2: Create a data pipeline between pub/sub topics, pub/sub subscriptions to Bigquery (BQ tables to be created via dataform) Confirm messages published to pub/sub appear in Bigquery

Components:

1. Pub/Sub Topic (emp_streamingdata)
    - Source of events/messages.
2. BigQuery Table (employees.emp_details)
    - Stores all events in a structured format    
3. Pub/Sub Subscription (emp_streamingdata-sub)
    - Direct BigQuery sink subscription that writes messages into a table.
    - enable "Write Metadata"
    - Use Table Schema
  
![task2](https://github.com/user-attachments/assets/aed58a00-1910-4962-9f3a-6416adfa01e5)


Step 1:

Create a pub/sub Topic (emp_streamingdata). Go to pub/Sub in cloud console and make sure you are in the correct project.

Click Topics in the left menu and click create Topic. Enter a topic ID: "emp_streamingdata" Leave other settings as default and click create.

Step 2: Create a Bigquery table via dataform:

Create a new .sqlx file under definitions and use the below code to create a new table.

```
config {
  type: "operations",
  hasOutput: true,
  tags: ["ddl"]
}

CREATE TABLE IF NOT EXISTS `dhivya-sandpit.employees.emp_details` (
  emp_id INT64 NOT NULL,
  name STRING,
  dept STRING,
  hire_date DATE,
  age INT64,
  city STRING,
  subscription_name STRING,
  message_id STRING,
  publish_time TIMESTAMP,
  attributes STRING
);

```


Execute this in Dataform.

Step 3: Create a Bigquery Sink Subscription

- Click the topic you created "emp_streamingdata" and click create subscription.
- you can give the subscription name **"emp_streamingdata-sub"**
- In Delivery type, click **"Write to Bigquery"**
- Select or enter your project id: "dhivya-sandpit", dataset: "employees", table: "emp_details"
- In Schema configuration: select **"Use table Schema"**
- Select **"Wite metadata"** (If you enable this, then your Bigquery table should have the 4 columns in bigquery table so that the pub/sub can write metadata to it "subscription_name", "message_id", "publish_time" and "attributes".


Step 4: Now we can validate by passing a test message in pub/sub and check if it appears in Bigquery.
Write the below line in cloud shell to pass a message to the topic.

Create a python file 
```
nano send_pubsub_message.py
```
Use this example python code to send message to the topic. The message should match your "emp_details" table schema.

````
from google.cloud import pubsub_v1
import json

project_id = "dhivya-sandpit"
topic_id = "emp_streamingdata"

publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path(project_id, topic_id)

# Message payload (must match table schema)
message = {
    "emp_id": 111,
    "name": "Alice Johnson",
    "dept": "Finance",
    "hire_date": "2025-09-08",  # YYYY-MM-DD string
    "age": 29,
    "city": "Melbourne",
    }

# Convert to JSON string and encode as bytes
data = json.dumps(message).encode("utf-8")

future = publisher.publish(topic_path, data)
print(f"Published message ID: {future.result()}")
````
To save and exit, press, CTRL+O and CTRL+X

Run the Script:
````
python3 send_pubsub_message.py
````
In Cloud shell environment you will see something like "Published message ID: 1234567890123456"


Step5: To see if the bigquery sink subscription has received the message, use the following code to query the BQ table

```
SELECT *
FROM `dhivya-sandpit.employee.emp_details`
ORDER BY emp_id 
```
The message sent by the topic will be displayed in the BQ table.

## Part 3: What happens when Dataform creates a table and loads it via GCS external table while it is also targetted by a Bigquery Pub/sub subscription

Here, we are going to merge the external table to the emp_details table which is also targetted by the Bigquery pub/sub subscription.

![task3](https://github.com/user-attachments/assets/04bb4e05-3ab0-4127-956d-7bed6cba66ca)

Create a new .sqlx file in dataform under definitions to merge the external table "emp_details_raw" to the final table "emp_details".

- Option 1: Use config type "incremental"

````
config {
  type: "incremental",
  schema: "employees",
  name: "emp_details"
}

SELECT
  emp_id,
  name,
  dept,
  PARSE_DATE('%Y-%m-%d', hire_date) AS hire_date,
  age,
  city
FROM ${ref("emp_details_raw")}
${when(incremental(),
   `WHERE emp_id NOT IN (SELECT emp_id FROM ${self()})`,
   ``)}
````


- Option 2: Use config type "operations" and merge the tables.

````

config {
  type: "operations",
  tags: ["merge", "emp_details"]
}

MERGE `dhivya-sandpit.employees.emp_details` T
USING (
  SELECT
    emp_id,
    name,
    dept,
    PARSE_DATE('%Y-%m-%d', hire_date) AS hire_date,  -- cast as needed
    age,
    city
  FROM ${ref("emp_details_raw")}
) S
ON T.emp_id = S.emp_id
WHEN NOT MATCHED THEN
  INSERT (emp_id, name, dept, hire_date, age, city)
  VALUES (S.emp_id, S.name, S.dept, S.hire_date, S.age, S.city);
  ````

In this code, we have merged the data from the external table "emp_details_raw" to the final table "emp_details" only for those whose emp_id is unique. If the emp_id is same to the data that is streamed before by pub/sub, then those employee details are not merged.

We can also include update option to this code to update the values of the other fields in the table if the emp_id matches between the external table and the existing pub/sub subscription table (emp_details). This will ensure values in the emp_details table are up-to date. 

Example scenaio:

When a person's age is changed from 29 to 30, but the emp_id still remains the same, so we want the table to be updated with the new values for the same emp_id rather than neglecting that row.

- Options 3: Use the following code to update the fields where the emp_id matches and merge the rest of the table where the emp_id does not match.

````
config {
  type: "operations",
  tags: ["merge", "emp_details"]
}

MERGE `dhivya-sandpit.employees.emp_details` T
USING (
  SELECT
    emp_id,
    name,
    dept,
    PARSE_DATE('%Y-%m-%d', hire_date) AS hire_date,
    age,
    city
  FROM ${ref("emp_details_raw")}
) S
ON T.emp_id = S.emp_id
WHEN MATCHED THEN
  UPDATE SET
    name = S.name,
    dept = S.dept,
    hire_date = S.hire_date,
    age = S.age,
    city = S.city
   
WHEN NOT MATCHED THEN
  INSERT (emp_id, name, dept, hire_date, age, city)
  VALUES (S.emp_id, S.name, S.dept, S.hire_date, S.age, S.city);
````

In the bigquery UI, you can query the following command to view your merged table.

````
SELECT * FROM `dhivya-sandpit.employees.emp_details` 
ORDER BY emp_id
````

<img width="885" height="385" alt="Merge_query" src="https://github.com/user-attachments/assets/10ff3d0e-b1f0-4763-81a1-c65b7d088eb9" />


Note:

In my results, when I tried a scenaio where the pub/sub messages were sent as duplicates, but bigquery subscription table "emp_details" added the duplicate messages from pub/sub. The reason is, we are not performing any transformations for the pub/sub messages, these messages are added to the table directly.

But to prevent duplicates in the table, we can run a transformation on "emp_details" table periodically to remove the duplicates.

Use the below code to remove duplicates from the table based on ranking by publish time. Only the latest message will be kept.
````
CREATE OR REPLACE TABLE `dhivya-sandpit.employees.emp_details` AS
WITH ranked AS (
  SELECT *,
         ROW_NUMBER() OVER (PARTITION BY emp_id ORDER BY publish_time DESC) AS rn
  FROM `dhivya-sandpit.employees.emp_details`
)
SELECT *
FROM ranked
WHERE rn = 1;
````

<img width="901" height="354" alt="querydeduped" src="https://github.com/user-attachments/assets/90ac4381-6d44-4b0c-aa71-8aad273f5c29" />

Here, you can see that only one Alice Johnson is present. In the previous query result, you could see there were two Alice Johnson details (duplicates). Thus, by running this de-deuplication transformation on a scheduled basis, we can remove duplicates from the streaming messages in the table.

