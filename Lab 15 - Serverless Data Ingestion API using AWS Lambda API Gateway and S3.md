#  **Lab 15: Serverless Data Ingestion API using AWS Lambda, API Gateway, and S3**

---

##  **Objective**

By the end of this lab, you will learn how to:

- Build an API endpoint using **API Gateway**
- Trigger a **Lambda function** from that endpoint
- Process JSON payloads and store them as files in **S3**
- Validate and transform data in Lambda before ingestion
- Test the full serverless pipeline (end-to-end)

---

##  **Prerequisites**

* AWS Account (with Lambda, S3, API Gateway, IAM access)
* Basic knowledge of JSON and REST APIs
* Postman or `curl` for API testing

---

##  **Architecture Overview**

```
Client → API Gateway → Lambda → S3 (Data Lake)
```

**Workflow:**

1. API Gateway exposes an HTTP POST endpoint `/upload`
2. Client sends JSON payload (e.g., sales record, IoT data)
3. Lambda validates and enriches data
4. Lambda stores the file in S3 with timestamped filenames

---

##  **Step 1: Create an S3 Bucket**

1. Go to **AWS S3 Console → Create bucket**
2. Bucket name: `serverless-data-lake-<yourname>`
3. Region: `us-east-1` (or your preferred)
4. Uncheck *Block all public access* (optional for this lab)
5. Click **Create bucket**

 You now have a data lake bucket ready for ingestion.

---

##  **Step 2: Create a Lambda Function**

1. Navigate to **AWS Lambda → Create Function**
2. Select **Author from scratch**
3. Function name: `DataIngestionLambda`
4. Runtime: **Python 3.9**
5. Permissions: *Create new role with basic Lambda permissions*
6. Click **Create Function**

---

###  Lambda Code (`lambda_function.py`)

```python
import json
import boto3
import datetime
import os

s3 = boto3.client('s3')
BUCKET_NAME = os.environ.get("BUCKET_NAME", "serverless-data-lake-varun")

def lambda_handler(event, context):
    # Parse incoming request body
    body = json.loads(event["body"])
    
    # Validate required fields
    required_fields = ["customer_id", "amount", "timestamp"]
    missing = [f for f in required_fields if f not in body]
    if missing:
        return {
            "statusCode": 400,
            "body": json.dumps({"error": f"Missing fields: {missing}"})
        }

    # Add ingestion metadata
    body["ingested_at"] = datetime.datetime.utcnow().isoformat()

    # Convert to JSON string
    record_json = json.dumps(body)

    # Generate file name
    file_name = f"ingested/{body['customer_id']}_{int(datetime.datetime.utcnow().timestamp())}.json"

    # Upload to S3
    s3.put_object(
        Bucket=BUCKET_NAME,
        Key=file_name,
        Body=record_json,
        ContentType="application/json"
    )

    return {
        "statusCode": 200,
        "body": json.dumps({"message": "Data uploaded successfully", "file": file_name})
    }
```

---

##  **Step 3: Add Environment Variable**

* Scroll to **Configuration → Environment Variables**
* Add:

  * Key: `BUCKET_NAME`
  * Value: `<your_s3_bucket_name>`

Click **Save**

---

##  **Step 4: Add IAM Permissions**

1. Go to **Configuration → Permissions → Role name**
2. In IAM Console, edit permissions
3. Attach the policy **AmazonS3FullAccess**
   *(for production, use a tighter policy — only `s3:PutObject` to your bucket)*

---

##  **Step 5: Create API Gateway**

1. Open **API Gateway → Create API**
2. Choose **REST API → Build**
3. **API name:** `DataIngestionAPI`
4. Click **Create API**

---

##  **Step 6: Create Resource and Method**

1. Under the new API → click **Actions → Create Resource**

   * Resource name: `upload`
   * Resource path: `/upload`
   * Check  “Enable API Gateway CORS”
2. Select `/upload` → **Actions → Create Method → POST**
3. Integration Type → **Lambda Function**
4. Region: same as Lambda
5. Lambda Function: `DataIngestionLambda`
6. Click **Save** → Approve the popup for permission

---

##  **Step 7: Deploy the API**

1. Click **Actions → Deploy API**
2. **Stage name:** `dev`
3. Click **Deploy**

You’ll get an **Invoke URL**, e.g.:

```
https://abcd1234.execute-api.us-east-1.amazonaws.com/dev/upload
```

---

##  **Step 8: Test the API**

### Using Postman:

* Method: `POST`
* URL:

  ```
  https://abcd1234.execute-api.us-east-1.amazonaws.com/dev/upload
  ```
* Body → **raw JSON** → `application/json`:

```json
{
  "customer_id": "CUST1001",
  "amount": 1500.75,
  "timestamp": "2025-10-08T10:15:00Z"
}
```

### Expected Response:

```json
{
  "message": "Data uploaded successfully",
  "file": "ingested/CUST1001_1733672134.json"
}
```

---

##  **Step 9: Verify S3 Output**

1. Go to your **S3 bucket**
2. Navigate to `/ingested/` folder
3. You should see your file uploaded
4. Open it to confirm content:

```json
{
  "customer_id": "CUST1001",
  "amount": 1500.75,
  "timestamp": "2025-10-08T10:15:00Z",
  "ingested_at": "2025-10-08T10:17:14.512Z"
}
```

---

##  **Step 10: Cleanup Resources**

After testing:

* Delete the **API Gateway API**
* Delete the **Lambda function**
* Delete the **S3 bucket** (if not needed)

---

##  **Outcome**

You’ve built a **Serverless Ingestion API** that:

* Accepts incoming JSON via REST API
* Validates and enriches it in Lambda
* Stores structured data in S3 for downstream analytics (e.g., Glue, Athena, or Lake Formation)

---

##  **Next Steps for Data Engineers**

| Next Goal                 | AWS Service                      | Description                            |
| ------------------------- | -------------------------------- | -------------------------------------- |
| Automate ETL on S3 upload | **AWS Glue / Lambda Trigger**    | Trigger Glue job when new file arrives |
| Query ingested data       | **Athena**                       | Create external table pointing to S3   |
| Schedule ingestion        | **EventBridge / Step Functions** | Automate periodic ingestion            |
| Add monitoring            | **CloudWatch + SNS**             | Track failed invocations and alert     |

---

