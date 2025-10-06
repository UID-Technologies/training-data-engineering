# **Lab 05 - Simple ELT Pipeline with AWS Glue & dbt (Windows Compatible)**

---

##  Lab Description

In this lab, you’ll practice how to:

1. **Store raw JSON data in S3**
2. **Discover schema with a Glue Crawler**
3. **Transform raw → Parquet using a Glue Job**
4. **Run SQL transformations with dbt on Athena**
5. **Reuse the workflow** for new datasets

---

##  What is AWS Glue?

AWS Glue is a **serverless data integration service** that can:

* Automatically **detect schema** with **Crawlers**
* Store metadata in the **Glue Data Catalog**
* Run **Spark jobs** for transformations
* Orchestrate steps with **Workflows**

---

##  Why dbt?

dbt lets you **write SQL models** to transform your data into clean, analytics-ready tables. It supports:

* **Incremental models** (only process new data)
* **Reusable macros** (like functions)
* **Testing & documentation**

---

##  Practical Use Case

**Log Analytics for a Website:**

* Logs land in **S3 (raw)**
* Glue **crawls** and discovers schema
* Glue Job **converts JSON → Parquet**
* dbt **aggregates logs** into reports (top URLs, counts by status)
* Analysts use **Athena** to query the curated results

---

#  Step-by-Step Lab (Windows Friendly)

---

### **Step 0. Setup**

* Install [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
* Install Python (3.9+) + pip
* Ensure IAM user has **S3, Glue, Athena** permissions

Configure CLI:

```powershell
aws configure
```

---

### **Step 1. Create Buckets**

```powershell
$env:LAB = "simple-glue-dbt-lab"
$env:REGION = "us-east-1"
$env:S3_DATA = "$($env:LAB)-data"
$env:S3_STAGE = "$($env:LAB)-stage"

aws s3 mb "s3://$env:S3_DATA" --region $env:REGION
aws s3 mb "s3://$env:S3_STAGE" --region $env:REGION
```

Create folders:

```powershell
aws s3api put-object --bucket $env:S3_DATA --key "raw/"
aws s3api put-object --bucket $env:S3_DATA --key "bronze/"
aws s3api put-object --bucket $env:S3_DATA --key "gold/"
```

---

### **Step 2. Load Sample Data**

Use AWS Glue public dataset:

```powershell
aws s3 cp "s3://awsglue-datasets/examples/us-legislators/all/" `
          "s3://$env:S3_DATA/raw/us-legislators/" --recursive
```

---

### **Step 3. Create Glue Database + Crawler**

```powershell
aws glue create-database --database-input Name=$env:LAB
```

* In Glue Console → **Crawlers → Create**
* Point to `s3://$env:S3_DATA/raw/us-legislators/`
* IAM role: `AWSGlueServiceRole-<name>`
* Target DB: `$env:LAB`
* Run crawler → check tables in Data Catalog

---

### **Step 4. Create Glue Job (JSON → Parquet)**

Create script `json_to_parquet.py`:

```python
import sys
from awsglue.utils import getResolvedOptions
from pyspark.sql import SparkSession

args = getResolvedOptions(sys.argv, ['JOB_NAME','src','dst'])
src = args['src']; dst = args['dst']

spark = SparkSession.builder.appName("json-to-parquet").getOrCreate()
df = spark.read.json(src)
df.write.mode("overwrite").parquet(dst)
spark.stop()
```

Upload:

```powershell
aws s3 cp .\json_to_parquet.py "s3://$env:S3_DATA/script/"
```

In Glue Console:

* Create Job → Spark
* Script location: `s3://.../script/json_to_parquet.py`
* Parameters:

  * `--src` = `s3://$env:S3_DATA/raw/us-legislators/`
  * `--dst` = `s3://$env:S3_DATA/bronze/us_legislators/`
* Run job → check Parquet output in `bronze/`

---

### **Step 5. Query Bronze Data in Athena**

* Open Athena → set query result location = `s3://$env:S3_STAGE/`
* Run:

```sql
SELECT * FROM "<LAB>".us_legislators limit 10;
```

---

### **Step 6. Install dbt + Athena Adapter**

```powershell
python -m venv .venv
. .\.venv\Scripts\Activate.ps1
pip install --upgrade pip
pip install dbt-core dbt-athena
```

---

### **Step 7. Setup dbt Profile**

Edit `C:\Users\<YOU>\.dbt\profiles.yml`:

```yaml
simple:
  target: dev
  outputs:
    dev:
      type: athena
      region_name: us-east-1
      database: simple-glue-dbt-lab   # same as crawler DB
      schema: dbt_dev
      s3_staging_dir: s3://simple-glue-dbt-lab-stage/
      s3_data_dir:    s3://simple-glue-dbt-lab-data/gold/
```

---

### **Step 8. Create a Simple dbt Model**

In `models/stg_legislators.sql`:

```sql
{{ config(materialized='table') }}
select name, gender, year(birthdate) as birth_year
from "{{ source('bronze','us_legislators') }}"
where birthdate is not null
```

Run:

```powershell
dbt debug
dbt run
dbt test
```

---

### **Step 9. Verify Results**

* Check Athena: table `dbt_dev.stg_legislators` created
* Query:

```sql
SELECT gender, count(*) FROM dbt_dev.stg_legislators GROUP BY gender;
```

---

### **Step 10. Cleanup**

```powershell
aws s3 rb "s3://$env:S3_DATA"  --force
aws s3 rb "s3://$env:S3_STAGE" --force
```

---

#  Learning Outcomes

* Use **Glue Crawler** for schema discovery
* Run a **Glue Job** to standardize format (JSON → Parquet)
* Query Parquet with **Athena**
* Build a simple **dbt model** for transformations
* Create a **reusable pipeline pattern** (raw → bronze → dbt → gold)

---
