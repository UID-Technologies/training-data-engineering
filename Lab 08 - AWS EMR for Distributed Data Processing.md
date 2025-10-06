
# Lab 08 — AWS EMR for Distributed Data Processing

---
##  Overview

In this lab, you will learn how to use **Amazon EMR (Elastic MapReduce)** to run distributed data processing jobs with **Apache Spark**.
You will create an **EMR cluster on EC2**, upload a **PySpark wordcount script**, and run it on a **public dataset** stored in S3.
You’ll also monitor job execution, explore logs, and practice scaling the cluster.

---

##  What is AWS EMR?

Amazon EMR is a **cloud-based big data platform** that makes it easy to process large volumes of data using **open-source frameworks** like **Apache Spark, Hadoop, Hive, HBase, and Presto**.

Instead of manually setting up clusters, EMR provisions and manages them for you — scaling as needed.

---

##  Important Features of EMR

* **Scalability**: Add/remove nodes dynamically (pay per second).
* **Flexibility**: Supports multiple engines — Spark, Hadoop, Hive, Presto.
* **Integration**: Works seamlessly with **S3, DynamoDB, Redshift, and RDS**.
* **Cost-Optimized**: Spot instances + autoscaling reduce costs.
* **Monitoring**: UIs (YARN, Spark History) and CloudWatch integration.
* **Security**: IAM roles, Kerberos, encryption options.

---

##  Practical Use Case

**Example: E-commerce company analyzing logs**

* Raw clickstream logs stored in S3.
* EMR cluster with Spark runs **ETL jobs** → cleans & aggregates logs.
* Results saved back to S3 → queried by **Athena** or visualized in **QuickSight**.
* Business gains insights on **user behavior, product trends, and site performance**.

---

##  Step-by-Step Lab (Windows PowerShell Compatible)

---

### **0. Prerequisites**

* AWS CLI installed ([Install Guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html))
* AWS IAM user with **EC2, EMR, S3** permissions
* EC2 **Key Pair** created in AWS Console

Verify setup:

```powershell
aws --version
aws configure
```

---

### **1. Set Environment Variables**

```powershell
$env:AWS_REGION = "us-east-1"
$env:LAB_NAME   = "emr-lab-2807"   # must be unique globally for S3
$env:KEY_NAME   = "aws-key-emr"    # your EC2 key pair name
```

Check:

```powershell
echo $env:AWS_REGION
```

---

### **2. Create S3 Buckets**

```powershell
aws s3 mb "s3://$env:LAB_NAME-data" --region $env:AWS_REGION
aws s3 mb "s3://$env:LAB_NAME-logs" --region $env:AWS_REGION

aws s3api put-object --bucket "$env:LAB_NAME-data" --key "script/"
aws s3api put-object --bucket "$env:LAB_NAME-data" --key "output/"
```

---

### **3. Create PySpark Script**

Save this as **`wordcount_s3.py`**:

```python
import sys
from pyspark.sql import SparkSession

if __name__ == "__main__":
    if len(sys.argv) != 3:
        print("Usage: wordcount_s3.py <input_s3_path> <output_s3_path>")
        sys.exit(-1)

    input_path = sys.argv[1]
    output_path = sys.argv[2]

    spark = SparkSession.builder.appName("WordCountS3").getOrCreate()
    sc = spark.sparkContext

    lines = sc.textFile(input_path)
    counts = (lines.flatMap(lambda x: x.split())
                   .map(lambda word: (word, 1))
                   .reduceByKey(lambda a, b: a + b))

    counts.saveAsTextFile(output_path)
    spark.stop()
```

Upload to S3:

```powershell
aws s3 cp .\wordcount_s3.py "s3://$env:LAB_NAME-data/script/"
```

---

### **4. Create EMR Roles**

```powershell
aws emr create-default-roles --region $env:AWS_REGION
```

This creates:

* `EMR_DefaultRole` (service role)
* `EMR_EC2_DefaultRole` (EC2 instances role)

---

### **5. Launch EMR Cluster**

```powershell
$CLUSTER_ID = aws emr create-cluster `
  --name "emr-lab-cluster" `
  --region $env:AWS_REGION `
  --release-label emr-6.15.0 `
  --applications Name=Hadoop Name=Spark Name=Hive `
  --ec2-attributes KeyName=$env:KEY_NAME `
  --instance-type m5.xlarge `
  --instance-count 3 `
  --use-default-roles `
  --enable-debugging `
  --log-uri "s3://$env:LAB_NAME-logs/" `
  --query 'ClusterId' --output text
echo "Cluster ID: $CLUSTER_ID"
```

Wait until status is **WAITING**:

```powershell
aws emr describe-cluster --cluster-id $CLUSTER_ID `
  --query "Cluster.Status.State"
```

---

### **6. Submit Spark Job as EMR Step**

```powershell
aws emr add-steps --region $env:AWS_REGION --cluster-id $CLUSTER_ID --steps `
Type=CUSTOM_JAR,Name="WordcountSparkStep",ActionOnFailure=CONTINUE,Jar=command-runner.jar,Args=["spark-submit","s3://$env:LAB_NAME-data/script/wordcount_s3.py","s3://elasticmapreduce/samples/wordcount/input/","s3://$env:LAB_NAME-data/output/wordcount/"]
```

Check job progress:

```powershell
aws emr list-steps --cluster-id $CLUSTER_ID `
  --query "Steps[].Status.State"
```

---

### **7. (Optional) SSH to Master Node**

```powershell
$MASTER_DNS = aws emr list-instances --cluster-id $CLUSTER_ID `
  --instance-group-types MASTER --query "Instances[0].PublicDnsName" --output text

ssh -i C:\path\to\$env:KEY_NAME.pem hadoop@$MASTER_DNS
```

Inside master:

```bash
spark-submit s3://$LAB_NAME-data/script/wordcount_s3.py \
  s3://elasticmapreduce/samples/wordcount/input/ \
  s3://$LAB_NAME-data/output/wordcount-ssh/
```

---

### **8. Monitor the Job**

* Go to **AWS Console → EMR → Cluster → Application user interfaces**
* Open:

  * **YARN ResourceManager UI** (job scheduling, resource allocation)
  * **Spark History Server** (job stages, DAGs, execution times)

---

### **9. Verify Results**

```powershell
aws s3 ls "s3://$env:LAB_NAME-data/output/wordcount/"
aws s3 cp "s3://$env:LAB_NAME-data/output/wordcount/part-0000*" - | Select-Object -First 10
```

---

### **10. Scale the Cluster**

List instance groups:

```powershell
aws emr list-instance-groups --cluster-id $CLUSTER_ID
```

Increase Core nodes to 5:

```powershell
aws emr modify-instance-groups --cluster-id $CLUSTER_ID `
  --instance-groups InstanceGroupId=ig-XXXXXXXXXXXXX,InstanceCount=5
```

---

### **11. Cleanup**

Terminate cluster:

```powershell
aws emr terminate-clusters --cluster-id $CLUSTER_ID
```

Delete S3 buckets:

```powershell
aws s3 rb "s3://$env:LAB_NAME-data" --force
aws s3 rb "s3://$env:LAB_NAME-logs" --force
```

---

#  Summary

You successfully:

* Created **S3 buckets** for data & logs.
* Uploaded a **PySpark wordcount job**.
* Launched and scaled an **EMR cluster**.
* Ran Spark jobs via **EMR Steps** and **SSH**.
* Monitored execution via **YARN/Spark History**.
* Verified output in **S3** and safely **terminated** the cluster.

---

