

#  **Lab 17 - Project — Spotify Data Engineering End-to-End Project on AWS**

---

##  **Lab Overview**

In this lab, you will build a complete **data engineering pipeline on AWS** using **S3, Glue, Athena, and QuickSight**.
You’ll simulate real-world ETL (Extract–Transform–Load) operations by processing Spotify data, transforming it with AWS Glue, and visualizing insights in QuickSight.

---

##  **Learning Objectives**

By the end of this lab, you will be able to:

1. Create and configure AWS IAM users and roles for secure access.
2. Create S3 buckets for data staging and data warehouse layers.
3. Build an ETL pipeline in AWS Glue using the **visual ETL editor**.
4. Execute, monitor, and validate ETL job output.
5. Use AWS Glue Crawler to automatically infer schema and create data catalog tables.
6. Query transformed data using **AWS Athena**.
7. Connect AWS QuickSight to Athena for business visualization.

---

##  **Architecture**

```
S3 (Staging Layer) 
   ↓
AWS Glue (ETL)
   ↓
S3 (Data Warehouse)
   ↓
AWS Glue Crawler → AWS Data Catalog
   ↓
AWS Athena (SQL Queries)
   ↓
AWS QuickSight (Visualization)
```

---

##  **Services Used**

* **IAM** → User and role management
* **S3** → Data storage for staging and data warehouse layers
* **AWS Glue** → ETL pipeline creation and data transformation
* **Athena** → SQL query engine on S3 data
* **QuickSight** → Data visualization tool

---

##  **Dataset Description**

The dataset used is **Spotify 2023**, consisting of 5 CSV files:
https://www.kaggle.com/datasets/tonygordonjr/spotify-dataset-2023

1. `albums.csv`
2. `artists.csv`
3. `spotify_data.csv`
4. `spotify_features.csv`
5. `spotify_tracks.csv`

After preprocessing, we will use:

* `albums.csv`
* `artists.csv`
* `tracks.csv`

Each contains:

| File            | Description                                |
| --------------- | ------------------------------------------ |
| **albums.csv**  | Album ID, album name, artist, release date |
| **artists.csv** | Artist name, followers, genre              |
| **tracks.csv**  | Track ID, popularity, playlist ID          |

---

##  **Step-by-Step Lab Instructions**

---

###  **Step 1: Create an IAM User**

1. Log in to **AWS Console** as **Root User**.
2. Navigate to **IAM (Identity and Access Management)**.
3. Click **Users → Create user**.
4. Enter user name (e.g., `spotify-admin`).
5. Enable **AWS Management Console access**.
6. Set a **custom password** (require password reset on first login).
7. Click **Next** → Attach policies directly.
8. Attach the following permissions:

   * `AmazonS3FullAccess`
   * `AWSGlueServiceRole`
   * `AmazonAthenaFullAccess`
   * `AmazonQuickSightAthenaAccess`
9. Click **Create user**.
10. Copy the **sign-in URL** and log in as the new IAM user.

---

###  **Step 2: Create S3 Buckets**

1. In AWS Console, go to **S3**.
2. Click **Create bucket** and give a unique name (e.g., `spotify-data-engineering-lab`).
3. Create two folders inside the bucket:

   * `/staging`
   * `/datawarehouse`
4. Upload the **three CSV files** (`albums.csv`, `artists.csv`, `tracks.csv`) into the `/staging` folder.

 *You now have a staging area ready for your ETL pipeline.*

---

###  **Step 3: Create AWS Glue ETL Pipeline**

1. Open the **AWS Glue** console.
2. Navigate to **ETL Jobs → Visual ETL Editor**.
3. Click **Create job** → Choose “Visual with Source and Target.”
4. Add three **S3 sources**:

   * `staging/albums.csv`
   * `staging/artists.csv`
   * `staging/tracks.csv`
5. Configure each as **CSV** input format.
6. Add a **Join Transform** to merge:

   * `albums` ↔ `artists` (join on `artist_id`)
   * Result ↔ `tracks` (join on `track_id`)
7. Add a **Drop Fields** transformation to remove duplicate `id`, `artist_id`, and redundant columns.
8. Add a **Target** → Select S3, folder `/datawarehouse`.

   * Format: **Parquet**
   * Compression: **Snappy**
9. Click **Script** tab to review the generated PySpark code.

 *AWS Glue automatically generates the PySpark ETL script for you.*

---

###  **Step 4: Create Glue IAM Role and Run the Job**

1. Switch to **Root account** (to create IAM Role).
2. Navigate to **IAM → Roles → Create role**.
3. Choose **AWS Service → Glue**.
4. Attach policy **AmazonS3FullAccess**.
5. Name it `GlueS3AccessRole` and create the role.
6. Return to your **Glue job** and assign this role.
7. Configure:

   * Glue version: **4.0**
   * Language: **Python 3**
   * Worker type: **G.1X**
   * Number of workers: **5**
8. Click **Run job**.
9. Monitor job status in **AWS Glue → Runs**.
10. Once successful, verify output files in:

    ```
    s3://spotify-data-engineering-lab/datawarehouse/
    ```

 Output should be Parquet files evenly distributed (~16 files of 3–4 MB each).

---

###  **Step 5: Create AWS Glue Crawler**

1. Go to **AWS Glue → Crawlers**.
2. Click **Add crawler**.
3. Name it: `spotify_data_crawler`.
4. Data source: `s3://spotify-data-engineering-lab/datawarehouse/`.
5. IAM Role: `GlueS3AccessRole`.
6. Create a new database: `spotify_db`.
7. Run the crawler.

 The crawler automatically detects schema and creates table metadata under **AWS Glue Data Catalog**.

---

###  **Step 6: Query Data with AWS Athena**

1. Open **AWS Athena** service.
2. Configure query output location:

   * Create S3 bucket `athena-output-spotify-project`
   * Set this path in Athena settings.
3. Choose data source: **AWS Data Catalog**.
4. Select database: **spotify_db**.
5. Run query:

   ```sql
   SELECT * FROM datawarehouse LIMIT 10;
   ```
6. Try a few analytical queries:

   ```sql
   SELECT artist, COUNT(track_id) AS total_tracks 
   FROM datawarehouse 
   GROUP BY artist 
   ORDER BY total_tracks DESC;
   ```
7. Verify that output files are saved in your `athena-output` bucket.

---

###  **Step 7: Visualize Data in AWS QuickSight**

1. Go to **QuickSight** and sign up (Enterprise Edition – 30-day free trial).
2. Region: **ap-south-1 (Mumbai)**.
3. After setup, navigate to **Datasets → New Dataset**.
4. Choose **Athena** as the data source.
5. Connect to your **spotify_db** and select the `datawarehouse` table.
6. Click **Create & Visualize**.
7. Build visualizations such as:

   * Bar chart: `Artist vs Total Tracks`
   * Pie chart: `Genre vs Followers`
   * Line chart: `Year vs Album Releases`
8. Click **Publish Dashboard** to share with business users.

---

##  **Lab Validation Checklist**

| Task                                   | Verification                         |
| -------------------------------------- | ------------------------------------ |
| IAM user created                       | ✅ User can log in with proper access |
| S3 staging & warehouse folders created | ✅ Files uploaded                     |
| Glue ETL job created & executed        | ✅ Parquet files generated            |
| Crawler executed                       | ✅ Database & table created           |
| Athena query executed                  | ✅ Query results visible              |
| QuickSight dashboard                   | ✅ Visualizations published           |

---

##  **Extension Exercise**

* Try replacing manual S3 uploads with **AWS Kinesis or DynamoDB Streams** for real-time ingestion.
* Implement **partitioning** in Glue for faster Athena queries.
* Enable **QuickSight scheduled refresh** for live dashboards.

---

