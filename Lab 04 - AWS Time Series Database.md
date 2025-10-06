# Lab 04 - Advanced AWS Time Series Database Lab Using Amazon Timestream

## Objective

- **Deploy and Configure:** Set up an AWS time series database using Amazon Timestream.
- **Automated Ingestion:** Configure automated ingestion of time-based data (e.g., simulated sensor readings).
- **Advanced Querying:** Execute advanced queries for time series analysis.
- **Visualization:** Use AWS tools (or third-party dashboards) to visualize trends.

---

## Prerequisites

- **AWS Account:** An account with permissions to create Timestream databases, Lambda functions, and CloudWatch rules.
- **IAM Roles:** Roles with policies for Amazon Timestream, CloudWatch, and Lambda.
- **Development Environment:** Familiarity with Python (for Lambda) and basic SQL.
- **AWS CLI (optional):** For interacting with AWS resources via the command line.

---

## Lab Steps

### 1. Create an Amazon Timestream Database and Table

#### **Create a Database:**
- Navigate to the **Amazon Timestream** console.
- Click **Create database** and provide a name (e.g., `TimeSeriesDB`).
- Choose between “Standard” or “Serverless” depending on your needs.

#### **Create a Table with Retention Settings:**
- Within your database, create a table (e.g., `SensorData`).
- Set the retention policies:
  - **Memory store retention:** e.g., 24 hours (for recent data).
  - **Magnetic store retention:** e.g., 6 months or more (for historical data).
- Click **Create table**.

*Tip: Retention settings balance performance and cost by keeping recent data in a fast, expensive store and older data in a cost-effective magnetic store.*

---

### 2. Set Up Data Ingestion with AWS Lambda

#### **Create a Lambda Function:**
- In the AWS Lambda console, create a new function using Python 3.x.
- Assign an IAM role with permissions to write data to Timestream.

#### **Lambda Code Example:**

```python
import json
import time
import random
import boto3
from botocore.exceptions import ClientError

# Initialize Timestream write client
timestream = boto3.client('timestream-write')

DATABASE_NAME = 'TimeSeriesDB'
TABLE_NAME = 'SensorData'

def lambda_handler(event, context):
    current_time = str(int(time.time() * 1000))  # time in milliseconds
    
    # Simulate sensor data (e.g., temperature and humidity)
    dimensions = [
        {'Name': 'sensor_id', 'Value': 'sensor_01'},
        {'Name': 'location', 'Value': 'lab'}
    ]
    
    records = [
        {
            'Dimensions': dimensions,
            'MeasureName': 'temperature',
            'MeasureValue': f"{random.uniform(20.0, 30.0):.2f}",
            'MeasureValueType': 'DOUBLE',
            'Time': current_time
        },
        {
            'Dimensions': dimensions,
            'MeasureName': 'humidity',
            'MeasureValue': f"{random.uniform(30.0, 60.0):.2f}",
            'MeasureValueType': 'DOUBLE',
            'Time': current_time
        }
    ]
    
    try:
        result = timestream.write_records(DatabaseName=DATABASE_NAME,
                                            TableName=TABLE_NAME,
                                            Records=records)
        print("WriteRecords status:", result['ResponseMetadata']['HTTPStatusCode'])
    except ClientError as err:
        print("Error:", err)
    
    return {
        'statusCode': 200,
        'body': json.dumps('Data ingested successfully!')
    }
```

### Deploy and Test:
- Save and deploy your Lambda function.
- Use the built-in test functionality to verify that data is written to Timestream.

----

## 3. Automate Data Ingestion
### Set Up a CloudWatch Event Rule (EventBridge):
- In the CloudWatch console, create a new rule to trigger your Lambda function every minute (or your chosen frequency).
- Under Event Source, select Schedule and configure a cron or rate expression (e.g., `rate(1 minute)`).
- Set your Lambda function as the target.

*This automation simulates continuous time series data ingestion.*

-----

## 4. Querying and Analyzing Time Series Data
### Using the Timestream Query Editor:
- Navigate back to Amazon Timestream in the AWS console.

- Open the Query Editor and run queries such as:

- **Retrieve Recent Data:**

```sql
SELECT * FROM "TimeSeriesDB"."SensorData"
WHERE time > ago(1h)
ORDER BY time DESC
```

- **Aggregate Query (e.g., Average Temperature over 5-minute bins):**

```sql
SELECT BIN(time, 5m) AS binned_time, AVG(measure_value::double) AS avg_temp
FROM "TimeSeriesDB"."SensorData"
WHERE measure_name = 'temperature'
  AND time > ago(6h)
GROUP BY BIN(time, 5m)
ORDER BY binned_time DESC
```

### Advanced Query Techniques:
- Explore joins based on dimensions.
- Utilize functions like `latest(), min(), max()`, and windowed aggregations.

----

## 5. Visualize Time Series Data
### Amazon QuickSight or Custom Dashboard:
- Connect QuickSight to Amazon Timestream using a custom connector or by exporting query results.
- Alternatively, build a custom dashboard using a Python web framework or Grafana (with the Timestream plugin) to visualize real-time trends.

*Visualization aids in identifying patterns, trends, and anomalies over time.*

-----

## 6. Advanced Lab Enhancements
- **Error Handling & Retries:**
    - Enhance the Lambda function with exponential backoff and improved error handling for throttling or write failures.
- **Scaling Considerations:**
    - Experiment with batch writing and adjust the ingestion frequency.
    - Leverage Timestream’s partitioning strategies for handling large data volumes.
- **Security Best Practices:**
    - Follow the principle of least privilege for IAM roles.
    - Implement proper VPC configurations if enhanced network security is needed.

-------

## 7. Cleanup
### Remove or Disable Resources:
- Delete the Timestream database (if no longer needed) to avoid ongoing charges.
- Remove the Lambda function and CloudWatch rules created during the lab.
- Review and delete any IAM policies that were exclusively set up for this exercise.

-----

## Conclusion

This lab exercise provides hands-on experience with AWS's time series capabilities using Amazon Timestream. You will have:

- Set up a time series database with custom retention policies.
- Developed an automated data ingestion pipeline.
- Executed advanced queries to analyze time-based data.
- Explored data visualization and scalability considerations.


These skills can be applied to build robust monitoring, IoT, or analytics applications that leverage the strengths of time series data processing in AWS.