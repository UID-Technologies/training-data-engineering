# **Lab 12 - Data Replication with Fivetran: Automating ELT Pipelines**

---

##  Overview

In this lab, you will learn how to use **Fivetran**, a fully managed ELT platform, to **replicate data** from a source system (e.g., MySQL) into a cloud destination (e.g., Amazon Redshift, Snowflake, or BigQuery). You’ll configure a **source connector**, set up a **destination warehouse**, and explore how Fivetran manages **schema evolution, transformations, and scheduling** with minimal setup.

---

##  What is Fivetran?

Fivetran is a **managed ELT (Extract, Load, Transform) service** that helps you **replicate data from sources to destinations automatically**. It supports over 500+ pre-built connectors (databases, SaaS apps, events, files) and handles schema drift, scaling, and monitoring.

* **ELT, not ETL**: Extract & Load first, then let your warehouse (e.g., Snowflake) handle Transformations.
* **Fully managed**: No pipelines or scripts to maintain.
* **Incremental syncs**: Moves only new/changed records, reducing costs.
* **Schema drift handling**: New columns automatically appear in your destination.

---

## Important Features

* **Pre-built connectors**: SaaS apps, DBs, files, events, APIs
* **Destinations**: Snowflake, BigQuery, Redshift, Databricks, Postgres, S3, etc.
* **Automatic schema migration** (schema drift)
* **Incremental syncs** (CDC - Change Data Capture)
* **Data transformations** (via dbt integration)
* **Security & governance**: encryption, role-based access

---

##  Use Case Scenario

**E-commerce company analytics**:
The company runs a MySQL-based order management system. Analysts need real-time insights in **Snowflake**. Instead of writing custom ETL pipelines, you will use **Fivetran** to replicate orders, products, and customer data into Snowflake. Fivetran will keep the data **continuously in sync**, so dashboards in Power BI/Tableau are always fresh.



---

## **Step 1. Create Fivetran Account**

**Why:** You need access to Fivetran’s web console to configure sources and destinations.

1. Go to [https://fivetran.com](https://fivetran.com).
2. Sign up for a free trial (no credit card needed).
3. Log in → you’ll land on the **Dashboard**.

---

## **Step 2. Configure Your Destination (e.g., Snowflake or Redshift)**

**Why:** Fivetran needs to know where to load replicated data.

1. In Fivetran Dashboard → **Destinations → Add Destination**.
2. Select **Snowflake** (or Redshift/BigQuery).
3. Enter:

   * **Host** (your Snowflake account URL)
   * **Warehouse** (e.g., `COMPUTE_WH`)
   * **Database** (e.g., `FIVETRAN_DB`)
   * **User/Password** (create a dedicated `FIVETRAN_USER` in Snowflake/Redshift).
4. Save → Fivetran will test connection.

---

## **Step 3. Configure Your Source (e.g., MySQL Database)**

**Why:** This is the system from which Fivetran will extract data.

1. Fivetran Dashboard → **Connectors → Add Connector**.
2. Choose **MySQL** (or any other supported source, e.g., Salesforce).
3. Enter:

   * **Host** (DB host or IP)
   * **Port** (3306 for MySQL)
   * **Database Name**
   * **Username/Password** (create a read-only user for Fivetran).
4. Allow Fivetran’s IPs in your DB firewall (listed in Fivetran setup wizard).
5. Save → Fivetran will test connection.

---

## **Step 4. Map Source to Destination Schema**

**Why:** Decide where your source tables should land.

1. Fivetran automatically scans source schema.
2. You can rename schemas/tables if needed (e.g., `mysql_orders` → `ecommerce.orders`).
3. Confirm schema mapping → Save.

---

## **Step 5. Set Sync Frequency**

**Why:** Control how often replication occurs.

* Default is **15 minutes** (can be 5 mins → 24 hrs).
* For real-time dashboards, choose 15 minutes.
* For low-volume systems, choose hourly or daily.

---

## **Step 6. Run Initial Sync**

**Why:** Fivetran will load historical + new data into your warehouse.

1. Click **Start Initial Sync**.
2. Fivetran extracts all tables from source → loads into destination.
3. Monitor **Status** in Dashboard (you’ll see rows processed).

---

## **Step 7. Verify Data in Destination**

**Why:** Validate replication worked.

* In Snowflake/Redshift, run:

```sql
SELECT * FROM ecommerce.orders LIMIT 10;
SELECT COUNT(*) FROM ecommerce.customers;
```

* Confirm row counts match your source DB.

---

## **Step 8. Explore Monitoring & Alerts**

**Why:** Airflow-like orchestration isn’t needed; Fivetran monitors for you.

* Dashboard shows:

  * **Last sync time**
  * **Latency** (time lag from source to destination)
  * **Errors** (network, schema, permissions)
* Enable **Email/Slack alerts**.

---

## **Step 9. Handle Schema Changes (Drift)**

**Why:** Sources evolve; Fivetran adapts.

* Add a new column to MySQL table:

```sql
ALTER TABLE orders ADD COLUMN delivery_status VARCHAR(50);
```

* Fivetran automatically detects new column and adds it to Snowflake table.
* No pipeline code changes needed.

---

## **Step 10. Enable Transformations (Optional)**

**Why:** To apply business logic post-load.

* Fivetran integrates with **dbt**.
* You can create a dbt model to calculate `customer_lifetime_value` and Fivetran will run it after each sync.

---

#  Recap & Key Learnings

* Fivetran provides **out-of-the-box data replication** with **zero pipelines to maintain**.
* You set up **Source → Destination → Sync Schedule** in <30 minutes.
* It automatically handles **incremental loads, schema drift, retries, and monitoring**.
* Perfect for analytics, BI, and ML pipelines.

---

##  Use Case Examples

* **Marketing**: Replicate Google Ads + Facebook Ads into BigQuery for unified spend analysis.
* **Finance**: Replicate NetSuite into Snowflake for real-time financial dashboards.
* **E-commerce**: Replicate MySQL orders + Shopify + Zendesk into Redshift for a 360° customer view.

---
