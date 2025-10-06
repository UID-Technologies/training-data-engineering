
#  **Lab 09 - Log Analysis with AWS EMR & Spark**


---

##  Overview

In this lab, you’ll process a dataset of **web server access logs** (Apache/Nginx style) using **Apache Spark** on EMR.
You will parse raw log data from S3, extract useful insights (e.g., top IP addresses, most requested URLs), and save results back to S3.

This simulates a **real-world log analytics pipeline** where companies analyze clickstream or server logs to detect usage trends, security anomalies, or performance bottlenecks.

---

##  Real-World Use Case

**Scenario:**
An e-commerce website stores **web access logs** in S3 daily.
Business teams want to:

* See the **top 10 IPs** hitting the site.
* Find the **most requested pages**.
* Track **HTTP response codes** (e.g., 200, 404, 500).

Instead of analyzing on a single machine, they use **EMR Spark** for **distributed log parsing**.

---

##  Step-by-Step Lab (Windows PowerShell Compatible)

---

### **0. Prereqs**

Same as previous lab:

* AWS CLI installed
* EMR roles created
* Key pair available

---

### **1. Setup Environment**

```powershell
$env:AWS_REGION = "us-east-1"
$env:LAB_NAME   = "emr-logs-lab-2807"
$env:KEY_NAME   = "aws-key-emr"
```

Create buckets:

```powershell
aws s3 mb "s3://$env:LAB_NAME-data" --region $env:AWS_REGION
aws s3 mb "s3://$env:LAB_NAME-logs" --region $env:AWS_REGION
```

---

### **2. Prepare Sample Log Data**

We’ll use a **public sample Apache log dataset** provided by AWS:

```
s3://elasticmapreduce/samples/logs/
```

Copy a few files into your bucket (optional for sandboxing):

```powershell
aws s3 cp s3://elasticmapreduce/samples/logs/ "s3://$env:LAB_NAME-data/input/" --recursive --exclude "*" --include "*.log"
```

---

### **3. Create Spark Log Analysis Script**

Save as **`log_analysis.py`**:

```python
import sys
from pyspark.sql import SparkSession
import re

# Regex for Apache log format
log_pattern = r'(\S+) - - \[(.*?)\] "(.*?)" (\d{3}) (\d+)'

def parse_line(line):
    match = re.match(log_pattern, line)
    if match:
        return (match.group(1), match.group(3), match.group(4))  # IP, URL, Status
    return ("Invalid", "Invalid", "0")

if __name__ == "__main__":
    if len(sys.argv) != 3:
        print("Usage: log_analysis.py <input_s3> <output_s3>")
        sys.exit(-1)

    input_path = sys.argv[1]
    output_path = sys.argv[2]

    spark = SparkSession.builder.appName("LogAnalysis").getOrCreate()
    sc = spark.sparkContext

    logs = sc.textFile(input_path)
    parsed = logs.map(parse_line).filter(lambda x: x[0] != "Invalid")

    # Top IP addresses
    top_ips = parsed.map(lambda x: (x[0], 1)).reduceByKey(lambda a,b: a+b).takeOrdered(10, key=lambda x: -x[1])

    # Most requested URLs
    top_urls = parsed.map(lambda x: (x[1], 1)).reduceByKey(lambda a,b: a+b).takeOrdered(10, key=lambda x: -x[1])

    # Count by response code
    status_counts = parsed.map(lambda x: (x[2], 1)).reduceByKey(lambda a,b: a+b).collect()

    # Save output
    with open("/tmp/log_output.txt", "w") as f:
        f.write("Top IPs:\n" + str(top_ips) + "\n\n")
        f.write("Top URLs:\n" + str(top_urls) + "\n\n")
        f.write("Status Codes:\n" + str(status_counts) + "\n")

    # Upload to S3
    import boto3
    s3 = boto3.client("s3")
    s3.upload_file("/tmp/log_output.txt", sys.argv[2].replace("s3://","").split("/")[0], "/".join(sys.argv[2].split("/")[1:]) + "log_output.txt")

    spark.stop()
```

Upload to S3:

```powershell
aws s3 cp .\log_analysis.py "s3://$env:LAB_NAME-data/script/"
```

---

### **4. Launch EMR Cluster**

(Similar to previous lab)

```powershell
$CLUSTER_ID = aws emr create-cluster `
  --name "emr-logs-cluster" `
  --region $env:AWS_REGION `
  --release-label emr-6.15.0 `
  --applications Name=Hadoop Name=Spark `
  --ec2-attributes KeyName=$env:KEY_NAME `
  --instance-type m5.xlarge `
  --instance-count 3 `
  --use-default-roles `
  --enable-debugging `
  --log-uri "s3://$env:LAB_NAME-logs/" `
  --query 'ClusterId' --output text
echo "Cluster ID: $CLUSTER_ID"
```

---

### **5. Submit Spark Step**

```powershell
aws emr add-steps --region $env:AWS_REGION --cluster-id $CLUSTER_ID --steps `
Type=CUSTOM_JAR,Name="LogAnalysisStep",ActionOnFailure=CONTINUE,Jar=command-runner.jar,Args=["spark-submit","s3://$env:LAB_NAME-data/script/log_analysis.py","s3://elasticmapreduce/samples/logs/","s3://$env:LAB_NAME-data/output/logs/"]
```

---

### **6. Verify Output**

```powershell
aws s3 ls "s3://$env:LAB_NAME-data/output/logs/"
aws s3 cp "s3://$env:LAB_NAME-data/output/logs/log_output.txt" .
type .\log_output.txt
```

---

### **7. Monitor**

* Open EMR Console → Application UIs
* Check **Spark History Server** for DAGs/stages.
* Check **YARN ResourceManager** for execution status.

---

### **8. Scale & Cleanup**

Same as previous lab:

* Scale up cluster if needed (`modify-instance-groups`)
* Terminate cluster and delete buckets when done

---

#  Summary

* Learned to process **semi-structured log data** with Spark.
* Practiced extracting **business insights** (top IPs, URLs, status codes).
* Understood how EMR can power **real-world analytics pipelines** beyond wordcount.

---
