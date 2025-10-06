# Lab 01 - Advanced AWS S3 Lab: Scalable Object Storage with Amazon S3

This lab exercise guides you through an advanced setup of Amazon S3, demonstrating how to create and secure a bucket, enable versioning and encryption, configure lifecycle policies, host a static website, set up cross-region replication, and integrate S3 with AWS Lambda via event notifications.

---

## Objectives

- **Bucket Configuration:** Create a secure S3 bucket with proper access policies.
- **Versioning & Encryption:** Enable versioning to maintain object history and enforce encryption for data protection.
- **Lifecycle Management:** Implement lifecycle rules for automated transition and expiration of objects.
- **Static Website Hosting:** Configure S3 to host a static website.
- **Cross-Region Replication:** Set up replication to a bucket in another region for disaster recovery.
- **Event-Driven Architecture:** Trigger AWS Lambda functions on S3 events.
- **Testing & Cleanup:** Validate the configuration and clean up resources to avoid ongoing costs.

---

## Prerequisites

- **AWS Account:** Ensure your account has the necessary permissions to create and manage S3 buckets, IAM roles, Lambda functions, and related resources.
- **AWS CLI:** (Optional) Installed and configured for command-line interactions.
- **Basic Knowledge:** Familiarity with S3, IAM, Lambda, and AWS Console operations.

---

## Lab Steps

### 1. Create an S3 Bucket

#### a. Using the AWS Console

1. Navigate to the **Amazon S3** console.
2. Click **Create bucket**.
3. Enter a unique bucket name (e.g., `advanced-s3-bucket-<your-unique-id>`).
4. Select your desired region.
5. Configure options:
   - Block public access (unless your use case requires public access).
   - (Optionally) Enable bucket versioning.
6. Click **Create bucket**.

#### b. Using AWS CLI

```bash
aws s3api create-bucket --bucket advanced-s3-bucket-<your-unique-id> --region us-east-1 --create-bucket-configuration LocationConstraint=us-east-1
```

-----

## 2. Enable Versioning
Versioning keeps multiple variants of an object, providing a history of changes.

### Via the AWS Console
1. Open your bucket.
2. Navigate to Properties.
3. Under Bucket Versioning, click Edit.
4. Enable versioning and save the changes.

### Using AWS CLI
```bash
aws s3api put-bucket-versioning --bucket advanced-s3-bucket-<your-unique-id> --versioning-configuration Status=Enabled
```

-----

## 3. Configure Bucket Encryption & Logging
### a. Enable Default Encryption
1. In the bucket’s Properties, scroll to Default encryption.
2. Enable encryption (choose AWS-KMS or S3-managed keys).
3. Save the settings.


### b. Using AWS CLI for Encryption
```bash
aws s3api put-bucket-encryption --bucket advanced-s3-bucket-<your-unique-id> --server-side-encryption-configuration '{
  "Rules": [{
    "ApplyServerSideEncryptionByDefault": {
      "SSEAlgorithm": "AES256"
    }
  }]
}'
```

### c. Enable Server Access Logging
1. In Properties, locate Server access logging.
2. Specify a target bucket (ensure the target bucket is properly configured) and a log file prefix.
3. Save the changes.

------

## 4. Set Up Lifecycle Policies
Lifecycle rules help manage objects by transitioning them to cheaper storage classes and expiring them when no longer needed.

### a. Create a Lifecycle Policy JSON
Create a file named lifecycle.json with the following content:

```json
{
  "Rules": [
    {
      "ID": "TransitionAndExpire",
      "Status": "Enabled",
      "Filter": { "Prefix": "" },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 60,
          "StorageClass": "GLACIER"
        }
      ],
      "Expiration": {
        "Days": 365
      },
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 90
      }
    }
  ]
}
```


### b. Apply the Lifecycle Policy
```bash
aws s3api put-bucket-lifecycle-configuration --bucket advanced-s3-bucket-<your-unique-id> --lifecycle-configuration file://lifecycle.json
```

------

## 5. Configure Static Website Hosting
S3 can host static websites with minimal configuration.

### a. Via the AWS Console
1. Open your bucket, navigate to Properties.
2. Scroll to Static website hosting.
3. Enable static website hosting.
4. Specify the Index document (e.g., index.html) and Error document (e.g., error.html).


### b. Using AWS CLI
```bash
aws s3 website s3://advanced-s3-bucket-<your-unique-id> --index-document index.html --error-document error.html
```

*Create and upload a sample `index.html` file:*

```html
<html>
  <head>
    <title>Advanced S3 Website</title>
  </head>
  <body>
    <h1>Welcome to My Advanced S3 Static Website</h1>
    <p>This site is hosted on Amazon S3.</p>
  </body>
</html>
```

Upload the file:

```bash
aws s3 cp index.html s3://advanced-s3-bucket-<your-unique-id>/
```

----

## 6. Set Up Cross-Region Replication (CRR)
CRR replicates objects to a destination bucket in another region, enhancing data durability and availability.

### a. Create a Destination Bucket
```bash
aws s3api create-bucket --bucket advanced-s3-replica-<your-unique-id> --region us-west-2 --create-bucket-configuration LocationConstraint=us-west-2
```

### b. Enable Versioning on the Destination Bucket
```bash
aws s3api put-bucket-versioning --bucket advanced-s3-replica-<your-unique-id> --versioning-configuration Status=Enabled
```


### c. Configure Replication
Create a file named `replication.json` with the following content:

```json
{
  "Role": "arn:aws:iam::<your-account-id>:role/<replication-role>",
  "Rules": [
    {
      "Status": "Enabled",
      "Prefix": "",
      "Destination": {
        "Bucket": "arn:aws:s3:::advanced-s3-replica-<your-unique-id>"
      }
    }
  ]
}
```
*Note: Ensure you have an IAM role for replication with the necessary policies.*

Apply the replication configuration:

```bash
aws s3api put-bucket-replication --bucket advanced-s3-bucket-<your-unique-id> --replication-configuration file://replication.json
```

-----


##7. Integrate S3 with AWS Lambda Using Event Notifications
Trigger Lambda functions automatically when new objects are added.

### a. Create a Lambda Function

Create a new Lambda function (Python 3.x) with the following sample code:

```python
import json

def lambda_handler(event, context):
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = record['s3']['object']['key']
        print(f"New object added: {key} in bucket: {bucket}")
    return {
        'statusCode': 200,
        'body': json.dumps('S3 event processed successfully!')
    }
```

Deploy the function and note its ARN.

### b. Configure S3 Event Notifications
1. In your S3 bucket's Properties, scroll to Event notifications.
2. Create a new event notification:
    - Event Name: NewObjectNotification
    - Event Types: Select "All object create events."
    - Destination: Choose Lambda Function and select your Lambda.
3. Save the configuration.


### c. Test the Integration
Upload a test file to your bucket:

```bash
aws s3 cp testfile.txt s3://advanced-s3-bucket-<your-unique-id>/
```
Then, check CloudWatch Logs for the Lambda function to confirm that the event was triggered.


-----

## 8. Testing and Validation
- **Static Website:** Open your browser and navigate to http://advanced-s3-bucket-<your-unique-id>.s3-website.<region>.amazonaws.com to view your hosted site.

- **Replication:** Upload objects to the source bucket and verify that they appear in the destination bucket.

- **Lambda Integration:** Confirm via CloudWatch Logs that the Lambda function logs the S3 event details.

----


## 9. Cleanup
To avoid incurring further costs, clean up all resources created during the lab.

### 1. Delete Buckets:

- Remove all objects from both the source and replica buckets.
- Delete the buckets:
```bash
aws s3 rb s3://advanced-s3-bucket-<your-unique-id> --force
aws s3 rb s3://advanced-s3-replica-<your-unique-id> --force
```
### 2. Delete the Lambda Function: Remove the Lambda function used for event notifications.

### 3. Clean Up IAM Roles: Delete any IAM roles created exclusively for this lab.


------

## Conclusion
By completing this advanced lab exercise, you have learned to:

- Create and configure an Amazon S3 bucket with advanced security and management features.
- Enable versioning, encryption, and access logging.
- Apply lifecycle policies to manage data transitions and expiration.
- Host a static website directly from S3.
- Set up cross-region replication for data redundancy.
- Integrate S3 with AWS Lambda for event-driven processing.
- These skills are crucial for building robust, scalable, and secure object storage solutions on AWS.