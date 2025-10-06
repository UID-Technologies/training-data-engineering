# Lab 02 - Advanced AWS RDS Lab: Relational Database Service

This lab exercise provides an in-depth, hands-on guide to deploying and managing a production-grade relational database using Amazon RDS. You will learn to create a secure, scalable, and highly available RDS instance with advanced configurations such as Multi-AZ deployments, read replicas, custom parameter groups, automated backups, and performance monitoring. Additionally, you will integrate your RDS instance with an application (via EC2 or Lambda) to perform database operations.

---

## Objectives

- **RDS Instance Creation:** Launch a production-ready RDS instance with advanced settings.
- **High Availability & Scalability:** Configure Multi-AZ deployment and read replicas.
- **Security & Connectivity:** Set up VPC, subnets, and security groups for secure database access.
- **Performance Tuning:** Utilize custom parameter groups and enhanced monitoring.
- **Backup & Maintenance:** Configure automated backups, snapshots, and maintenance windows.
- **Application Integration:** Connect to the database from an external client (EC2/Lambda) and perform CRUD operations.
- **Cleanup:** Properly decommission resources to prevent ongoing charges.

---

## Prerequisites

- **AWS Account:** With permissions to create and manage RDS, EC2, Lambda, VPCs, and IAM roles.
- **VPC & Subnet Basics:** Familiarity with AWS VPC configuration and security groups.
- **Database Fundamentals:** Basic SQL knowledge and experience with a relational database engine (e.g., MySQL or PostgreSQL).
- **AWS CLI (Optional):** For automating resource creation.
- **Programming Environment:** Python is used for sample code (install required libraries like `pymysql` for MySQL or `psycopg2` for PostgreSQL).

---

## Lab Steps

### 1. Set Up VPC, Subnets, and Security Group

#### a. Create a VPC and Subnets
- If you don't have an existing VPC, create one with public and private subnets.
- Ensure the subnets are in at least two different Availability Zones for high availability.

#### b. Configure a Security Group
- Create a security group that allows inbound connections on your chosen database port:
  - **MySQL:** Port `3306`
  - **PostgreSQL:** Port `5432`
- Allow access only from trusted IP addresses or EC2 instances within your VPC.

*Example using AWS CLI (MySQL port):*
```bash
aws ec2 create-security-group --group-name RDS-SG --description "Security group for RDS instance"
aws ec2 authorize-security-group-ingress --group-name RDS-SG --protocol tcp --port 3306 --cidr <your-ip-address>/32
```

-----

## 2. Launch an Amazon RDS Instance
### a. Choose the Database Engine
 - Select the desired engine (e.g., MySQL or PostgreSQL).
### b. Configure the Instance
- Instance Specifications: Choose an instance class (e.g., db.t3.medium).
- Multi-AZ Deployment: Enable for high availability.
- Storage: Allocate sufficient storage and enable autoscaling if needed.
- Backup & Maintenance: Configure automated backups, set backup and maintenance windows.

### c. Create a Custom Parameter Group (Optional)
- Create a parameter group to tune database settings.
- Modify parameters such as connection limits, query cache settings, or logging options.

*Example using AWS CLI for MySQL:*

```bash
aws rds create-db-parameter-group --db-parameter-group-name my-mysql-params --db-parameter-group-family mysql8.0 --description "Custom MySQL parameter group for performance tuning"
```

### d. Launch the Instance
- In the AWS Console, navigate to Amazon RDS and click Create database.
- Follow the guided steps, applying your VPC, subnet group, security group, and custom parameter group.
- Example CLI command (simplified):

```bash
aws rds create-db-instance \
    --db-instance-identifier advanced-rds-instance \
    --db-instance-class db.t3.medium \
    --engine mysql \
    --allocated-storage 20 \
    --master-username adminuser \
    --master-user-password yourpassword \
    --vpc-security-group-ids <security-group-id> \
    --db-subnet-group-name <your-db-subnet-group> \
    --backup-retention-period 7 \
    --multi-az \
    --publicly-accessible false \
    --db-parameter-group-name my-mysql-params
```

*Note: Adjust the command parameters based on your requirements and chosen engine.*


----

## 3. Create Read Replicas (For Scaling Reads)

### a. Configure a Read Replica
- Once your primary instance is running, create a read replica to offload read traffic.
- In the RDS Console, select your instance and choose Actions > Create read replica.

*Example CLI command:*

```bash
aws rds create-db-instance-read-replica \
    --db-instance-identifier advanced-rds-replica \
    --source-db-instance-identifier advanced-rds-instance \
    --db-instance-class db.t3.medium
```

-----



## 4. Connect and Configure Your Database
### a. Connect from an EC2 Instance or Local Client
- Launch an EC2 instance in the same VPC (if needed) or use a local SQL client.
- Use the endpoint provided in the RDS Console to connect.
- Example MySQL connection command:

```bash

mysql -h advanced-rds-instance.abcdefg12345.us-east-1.rds.amazonaws.com -P 3306 -u adminuser -p
```

### b. Create a Sample Database and Table
- Run SQL commands to create a database and a sample table.

```sql
CREATE DATABASE sampledb;
USE sampledb;

CREATE TABLE users (
    user_id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```
### c. Insert Sample Data
```sql
INSERT INTO users (username, email) VALUES ('alice', 'alice@example.com'), ('bob', 'bob@example.com');
```
------

## 5. Enable Enhanced Monitoring and Performance Insights
### a. Enable Enhanced Monitoring
- In the RDS Console, enable Enhanced Monitoring to view real-time metrics (CPU, memory, IOPS, etc.).
- Choose a monitoring interval (e.g., 60 seconds).

### b. Performance Insights (Optional)
- Activate Performance Insights to analyze query performance and identify bottlenecks.
- Adjust retention settings based on your needs.

------


## 6. Integrate with an Application (Using AWS Lambda)
### a. Create a Lambda Function to Interact with RDS
- Write a Lambda function that connects to your RDS instance to perform database operations.
- Package your function with the necessary database connector library (e.g., pymysql for MySQL).

*Example Lambda Function (Python with pymysql):*

```python
import pymysql
import os

# Database settings from environment variables
rds_host  = os.environ['RDS_HOST']
username  = os.environ['DB_USERNAME']
password  = os.environ['DB_PASSWORD']
db_name   = os.environ['DB_NAME']

def lambda_handler(event, context):
    try:
        connection = pymysql.connect(host=rds_host, user=username, password=password, db=db_name, connect_timeout=5)
        with connection.cursor() as cursor:
            # Sample query: fetch all users
            cursor.execute("SELECT * FROM users")
            result = cursor.fetchall()
        connection.close()
        return {
            'statusCode': 200,
            'body': result
        }
    except Exception as e:
        return {
            'statusCode': 500,
            'body': str(e)
        }
```
### b. Configure Lambda Environment Variables
- Set environment variables (`RDS_HOST`, `DB_USERNAME`, `DB_PASSWORD`, `DB_NAME`) in your Lambda configuration.
- Ensure your Lambda function has network access to your RDS instance (i.e., attach it to the same VPC/subnets).

------

## 7. Monitoring, Backups, and Maintenance
### a. Automated Backups and Snapshots
- Verify that automated backups are enabled (configured earlier).
- Optionally, create manual snapshots for point-in-time recovery:
```bash
aws rds create-db-snapshot --db-snapshot-identifier advanced-rds-snapshot --db-instance-identifier advanced-rds-instance
```
### b. Maintenance Windows and Updates
- Configure a maintenance window to apply minor updates and parameter changes.
- Monitor logs and CloudWatch metrics to evaluate performance and health.

------

## 8. Cleanup
To avoid incurring further charges, perform the following cleanup steps once the lab is complete:

## 1. Delete Read Replicas:

- Remove the read replica from the RDS Console or via CLI.
```bash
aws rds delete-db-instance --db-instance-identifier advanced-rds-replica --skip-final-snapshot
```

## 2. Delete the Primary RDS Instance:
- Ensure you take a final snapshot if needed.

```bash
aws rds delete-db-instance --db-instance-identifier advanced-rds-instance --skip-final-snapshot
```

## 3. Clean Up Other Resources:
- Terminate any associated EC2 instances or Lambda functions.
- Delete the security group and custom parameter groups if no longer required.
- Remove any manual snapshots to free storage.

--- 
## Conclusion
By completing this advanced lab exercise, you have gained practical experience in:

- Deploying a highly available and secure Amazon RDS instance.
- Configuring Multi-AZ, read replicas, and custom parameter groups for performance tuning.
- Integrating your database with an application using AWS Lambda.
- Implementing monitoring, automated backups, and maintenance strategies.
These skills are essential for building and managing production-grade relational databases on AWS.