# Lab 03 - Advanced AWS DynamoDB Lab: Managed NoSQL Database for Scalable Applications

This lab exercise guides you through an advanced setup of AWS DynamoDB, demonstrating key features such as data modeling, global secondary indexes, DynamoDB Streams, Lambda triggers, and API Gateway integration. You will learn how to create and manage a highly scalable NoSQL database and build a serverless application around it.

---

## Objectives

- **Create and Configure a DynamoDB Table:** Design a table with a partition key and sort key, set up auto-scaling, and enable DynamoDB Streams.
- **Implement Global Secondary Indexes (GSI):** Enhance query flexibility with additional indexes.
- **Integrate with AWS Lambda:** Process stream events and perform server-side logic.
- **Build an API with API Gateway:** Expose DynamoDB operations via a RESTful interface.
- **Demonstrate Advanced Use Cases:** Execute transactions, error handling, and performance tuning.

---

## Prerequisites

- **AWS Account:** With sufficient permissions to create DynamoDB tables, Lambda functions, API Gateway endpoints, and IAM roles.
- **AWS CLI / SDK Setup:** Familiarity with AWS CLI and a programming language (Python in this lab) to interact with DynamoDB and Lambda.
- **Basic Understanding:** NoSQL concepts, AWS IAM, and serverless application architecture.

---

## Lab Steps

### 1. Create a DynamoDB Table

#### **Design the Data Model:**
For this lab, we will create an **Orders** table with:
- **Partition Key:** `order_id` (String)
- **Sort Key:** `created_at` (Number) – Unix timestamp for order creation

Additionally, we will set up a **Global Secondary Index (GSI)** for querying by customer:
- **GSI Partition Key:** `customer_id` (String)
- **GSI Sort Key:** `order_status` (String)

#### **Steps to Create the Table:**

1. **Using the AWS Console:**
   - Navigate to the **DynamoDB** console.
   - Click **Create table**.
   - Enter **Table name:** `Orders`.
   - Define the primary key attributes:
     - Partition key: `order_id` (String)
     - Sort key: `created_at` (Number)
   - Under **Additional settings**, enable auto-scaling for read and write capacity.
   - Enable **DynamoDB Streams** with the view type set to `NEW_AND_OLD_IMAGES`.
   - Click **Create table**.

2. **Add a Global Secondary Index (GSI):**
   - In the table details, navigate to the **Indexes** tab.
   - Click **Create index**.
   - Enter **Index name:** `CustomerIndex`.
   - Set the partition key: `customer_id` (String) and sort key: `order_status` (String).
   - Configure provisioned throughput or auto-scaling.
   - Create the index.

---

### 2. Insert and Query Data

#### **Sample Data Insertion with Python (Using Boto3):**

Create a Python script (or use AWS CLI) to insert sample order data:

```python
import boto3
import time
import uuid

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Orders')

def insert_order(customer_id, order_status, details):
    order = {
        'order_id': str(uuid.uuid4()),
        'created_at': int(time.time()),
        'customer_id': customer_id,
        'order_status': order_status,
        'details': details
    }
    table.put_item(Item=order)
    print(f"Inserted order: {order['order_id']}")

# Insert sample orders
insert_order('cust123', 'processing', {'item': 'Laptop', 'quantity': 1})
insert_order('cust456', 'completed', {'item': 'Smartphone', 'quantity': 2})
insert_order('cust123', 'completed', {'item': 'Headphones', 'quantity': 3})
```

### Query Using the GSI:
Query orders for a specific customer using the GSI:

```python
response = table.query(
    IndexName='CustomerIndex',
    KeyConditionExpression=boto3.dynamodb.conditions.Key('customer_id').eq('cust123')
)
print("Orders for customer cust123:", response['Items'])
```

-----

## 3. Process DynamoDB Streams with AWS Lambda
### Configure a Lambda Function to Process Stream Events:
1. **Create a New Lambda Function:**
    - In the AWS Lambda console, create a new function (Python 3.x).
    - Assign an IAM role with permissions for DynamoDB Streams and CloudWatch Logs.
2. **Lambda Code Example:**

```python
import json

def lambda_handler(event, context):
    for record in event['Records']:
        eventName = record['eventName']
        new_image = record.get('dynamodb', {}).get('NewImage', {})
        old_image = record.get('dynamodb', {}).get('OldImage', {})
        print(f"Event: {eventName}")
        if new_image:
            print("New record:", json.dumps(new_image))
        if old_image:
            print("Old record:", json.dumps(old_image))
    return {
        'statusCode': 200,
        'body': json.dumps('Stream processed successfully!')
    }
```

3. **Attach the Stream as a Trigger:**
    - In the Lambda configuration, add a trigger and select the DynamoDB Streams from the Orders table.
    - Save the changes.

------

## 4. Build a REST API with API Gateway
### Expose DynamoDB Operations via API Gateway:
1. **Create an API:**
    - In the API Gateway console, create a new REST API (e.g., OrdersAPI).
    - Define resources such as /orders.
2. **Integrate with Lambda:**
    - Create a new Lambda function or use an existing one that performs CRUD operations on the Orders table.
    - Configure API Gateway methods (GET, POST, PUT, DELETE) to trigger the Lambda function.
3. **Sample Lambda Code for API Operations:**

```python
import json
import boto3
from boto3.dynamodb.conditions import Key

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Orders')

def lambda_handler(event, context):
    http_method = event['httpMethod']
    if http_method == 'GET':
        # Query orders (for demo: fetch all orders for a customer)
        customer_id = event['queryStringParameters'].get('customer_id')
        response = table.query(
            IndexName='CustomerIndex',
            KeyConditionExpression=Key('customer_id').eq(customer_id)
        )
        return {
            'statusCode': 200,
            'body': json.dumps(response['Items'])
        }
    elif http_method == 'POST':
        # Create a new order
        order = json.loads(event['body'])
        table.put_item(Item=order)
        return {
            'statusCode': 201,
            'body': json.dumps({'message': 'Order created successfully'})
        }
    else:
        return {
            'statusCode': 400,
            'body': json.dumps({'message': 'Unsupported HTTP method'})
        }
```

4. **Deploy the API:**
    - Create a deployment stage (e.g., dev).
    - Test the API endpoints using tools like Postman or cURL.

----

## 5. Advanced Operations and Best Practices
### Transactions:
- Use DynamoDB transactions to perform multiple operations atomically.

```python
client = boto3.client('dynamodb')
response = client.transact_write_items(
    TransactItems=[
        {
            'Put': {
                'TableName': 'Orders',
                'Item': {
                    'order_id': {'S': 'order1'},
                    'created_at': {'N': '1625097600'},
                    'customer_id': {'S': 'cust789'},
                    'order_status': {'S': 'processing'}
                }
            }
        },
        {
            'Update': {
                'TableName': 'Orders',
                'Key': {
                    'order_id': {'S': 'order2'},
                    'created_at': {'N': '1625097601'}
                },
                'UpdateExpression': 'SET order_status = :status',
                'ExpressionAttributeValues': {
                    ':status': {'S': 'completed'}
                }
            }
        }
    ]
)
```

### Error Handling & Retries:
- Implement error handling in your Lambda functions to manage throttling and retries.
- Use AWS SDK’s built-in retry mechanisms or custom logic for 
critical operations.

### Monitoring and Logging:
- Use CloudWatch Logs to monitor Lambda functions and DynamoDB metrics.
- Set up alarms for read/write capacity and throttling events.

-----

## 6. Cleanup
- **Delete API Gateway**: Remove the deployed API if no longer needed.

- **Remove Lambda Functions**: Delete or disable Lambda functions created for this lab.

- **Delete DynamoDB Table**: Remove the Orders table (and associated indexes) to prevent ongoing charges.
- **Review IAM Policies**: Clean up any IAM roles or policies created solely for this exercise.

-----

## Conclusion
By completing this advanced lab exercise, you have demonstrated how to:

- Design and create a DynamoDB table with advanced features.
- Leverage Global Secondary Indexes for enhanced query capabilities.
- Process real-time data changes with DynamoDB Streams and AWS Lambda.
- Build a serverless REST API using API Gateway and Lambda.
- Implement transactions, error handling, and best practices for scalable, NoSQL applications.

This lab provides a solid foundation for building robust, high-performance applications using AWS DynamoDB as your managed NoSQL database solution.