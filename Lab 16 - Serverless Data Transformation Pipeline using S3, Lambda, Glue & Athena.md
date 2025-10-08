

#  **Lab 16 (continue) — Serverless Data Transformation Pipeline using S3, Lambda, Glue & Athena**

---

##  **Objective**

By the end of this lab, you will learn to:

- Automatically trigger a **Lambda function** when a new file lands in S3
- Transform and move data from **raw zone** → **processed zone**
- Catalog transformed data using **AWS Glue Data Catalog**
- Query the data in **Athena** using SQL

---

##  **Prerequisites**

Before starting, ensure you have completed **Part 1**:

* You have a working ingestion API that stores JSON in S3 (e.g., `/ingested/` folder)
* AWS services: **S3**, **Lambda**, **Glue**, **Athena**, and **IAM** access

---

##  **Architecture**

```
[Client/API] → API Gateway → Lambda → S3 (ingested/raw)
                     ↓ (S3 Event Trigger)
                Lambda (transform)
                     ↓
            S3 (processed zone) → Glue Table → Athena Queries
```

---

##  **Step 1: Organize Your S3 Bucket**

We’ll create logical zones in your data lake.

1. Go to your existing S3 bucket (`serverless-data-lake-varun`)
2. Create the following folders (prefixes):

   ```
   ingested/
   processed/
   ```
3. Confirm that Part 1’s Lambda is writing files into `ingested/`.

---

##  **Step 2: Create Lambda for Data Transformation**

1. Go to **AWS Lambda → Create function**
2. Function name: `DataTransformLambda`
3. Runtime: **Python 3.9**
4. Permissions: *Create new role with basic Lambda permissions*
5. Click **Create Function**

---

###  Lambda Transformation Code (`lambda_function.py`)

This Lambda will:

* Trigger automatically when a new file is uploaded to S3
* Read the JSON content
* Transform (add a calculated field or clean data)
* Write the transformed file into `/processed/`

```python
import json
import boto3
import os

s3 = boto3.client('s3')
BUCKET_NAME = os.environ.get("BUCKET_NAME", "serverless-data-lake-varun")

def lambda_handler(event, context):
    # Get file info from event
    record = event['Records'][0]
    s3_object = record['s3']['object']['key']
    print(f"Processing file: {s3_object}")

    # Read file content
    file_obj = s3.get_object(Bucket=BUCKET_NAME, Key=s3_object)
    raw_data = json.loads(file_obj['Body'].read().decode('utf-8'))

    # Apply simple transformation
    transformed = raw_data
    transformed['amount_with_tax'] = round(transformed['amount'] * 1.18, 2)  # add 18% tax
    transformed['processed_by'] = 'LambdaTransform'

    # Save to processed zone
    output_key = s3_object.replace("ingested/", "processed/")
    s3.put_object(
        Bucket=BUCKET_NAME,
        Key=output_key,
        Body=json.dumps(transformed),
        ContentType="application/json"
    )

    print(f"Transformed file saved to: {output_key}")
    return {
        "statusCode": 200,
        "body": json.dumps({"message": f"Processed file saved to {output_key}"})
    }
```

---

##  **Step 3: Configure Lambda Trigger (S3 Event)**

1. Open **S3 Console → Your Bucket → Properties**
2. Scroll to **Event Notifications → Create Event Notification**
3. Name: `TriggerDataTransform`
4. Event Type: `All object create events`
5. Prefix: `ingested/`
6. Destination: **Lambda Function**
7. Select: `DataTransformLambda`
8. Click **Save**

 Now whenever a new file is added to `ingested/`, Lambda will auto-trigger.

---

##  **Step 4: Add Permissions for Lambda to Access S3**

1. In **Lambda → Configuration → Permissions**
2. Click the role → Add policy:

   * Policy name: `S3ReadWriteAccess`
   * Permissions: `s3:GetObject`, `s3:PutObject`, `s3:ListBucket`
   * Resource: your bucket ARN
3. Attach and save.

---

##  **Step 5: Test the Trigger**

1. Re-use Part 1’s API to upload new data via Postman:

   ```json
   {
     "customer_id": "CUST2001",
     "amount": 2000.00,
     "timestamp": "2025-10-08T10:30:00Z"
   }
   ```
2. Wait a few seconds, then open your bucket:

   * `ingested/` → contains raw file
   * `processed/` → contains transformed file with `amount_with_tax` field

---

###  Example of Processed File:

```json
{
  "customer_id": "CUST2001",
  "amount": 2000.0,
  "timestamp": "2025-10-08T10:30:00Z",
  "ingested_at": "2025-10-08T10:32:00.123Z",
  "amount_with_tax": 2360.0,
  "processed_by": "LambdaTransform"
}
```

---

##  **Step 6: Create Glue Crawler (Data Catalog)**

1. Go to **AWS Glue → Crawlers → Create Crawler**
2. Name: `ProcessedDataCrawler`
3. Source type: **Data Stores**
4. Choose S3 → Point to your bucket’s `/processed/` path
5. IAM Role: create or select existing Glue role
6. Output: choose an existing **Database** or create new, e.g., `serverless_datalake_db`
7. Click **Finish → Run Crawler**

 After it completes, it creates a **table schema** from your JSON files.

---

##  **Step 7: Query Data in Athena**

1. Go to **Athena Console**
2. Select the same database (`serverless_datalake_db`)
3. You’ll see your table (e.g., `processed_data`)
4. Run query:

```sql
SELECT customer_id, amount, amount_with_tax, processed_by
FROM processed_data
ORDER BY amount_with_tax DESC;
```

###  Example Result

| customer_id | amount  | amount_with_tax | processed_by    |
| ----------- | ------- | --------------- | --------------- |
| CUST2001    | 2000.0  | 2360.0          | LambdaTransform |
| CUST1001    | 1500.75 | 1770.89         | LambdaTransform |

---

##  **Step 8: (Optional) Visualize Data with QuickSight**

1. Go to **AWS QuickSight → Manage Data → New Dataset**
2. Choose **Athena**
3. Select your `serverless_datalake_db` and table
4. Build dashboards (e.g., total revenue, tax added, etc.)

---

##  **Step 9: Cleanup Resources**

To avoid charges:

* Delete Glue crawler and database
* Delete Lambda functions (`DataIngestionLambda`, `DataTransformLambda`)
* Delete API Gateway
* Delete S3 bucket

---

##  **Final Outcome**

You’ve built a **fully automated, serverless data pipeline**:

1. **Data ingestion** via REST API → **Lambda + API Gateway**
2. **Data transformation** via S3 trigger → **Lambda**
3. **Cataloging** via **Glue Crawler**
4. **Querying** via **Athena**
5. (Optional) **Visualization** via **QuickSight**

---

##  **Concepts Reinforced**

| Concept          | AWS Service          | Description                    |
| ---------------- | -------------------- | ------------------------------ |
| Serverless API   | API Gateway + Lambda | No server provisioning         |
| Data Lake Zones  | S3                   | Raw vs. Processed organization |
| Event-driven ETL | Lambda Trigger       | Real-time data flow            |
| Data Catalog     | Glue                 | Automated schema discovery     |
| Query Engine     | Athena               | Serverless SQL on S3           |

---

