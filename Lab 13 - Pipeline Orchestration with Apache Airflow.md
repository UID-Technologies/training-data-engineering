
# **Lab 13 - Pipeline Orchestration with Apache Airflow (Windows Compatible, Docker Compose)**

##  Overview

You’ll deploy Apache Airflow locally with Docker Compose, create a simple but realistic **ELT DAG** that:

1. fetches a small CSV from the web,
2. transforms it, and
3. loads it into **Postgres** — all orchestrated by **Airflow** with scheduling, retries, logs, and connections.
   You’ll also configure **Connections & Variables**, run the DAG manually and on a **schedule**, and practice **backfills** and **debugging** from the Web UI.

---

##  What is Apache Airflow?

Apache Airflow is an **open-source orchestration platform** to **author, schedule, and monitor** data workflows as **DAGs** (Directed Acyclic Graphs). You write workflows in Python using **Operators** (Bash, Python, SQL, etc.), then Airflow handles **dependency management, scheduling, retries, logging, and observability**.

##  Important Features (you’ll use many of these)

* **DAGs & Operators**: Build modular pipelines in Python.
* **Scheduler & Executors**: Run tasks on time and in parallel.
* **UI & Logs**: Track runs, dependencies, and task logs.
* **Connections & Variables**: Centralize creds & config (no hard-coding).
* **Retries, SLAs, Alerts, Backfills**: Production-grade operations.
* **XCom**: Pass small metadata between tasks.

##  Real Use Case (Scenario)

**Daily analytics ingestion**: Every night at 2 AM, fetch a CSV from a web endpoint, transform it into a clean schema, and load it into a warehouse (Postgres here, but could be Snowflake/BigQuery). You want automatic retries, visibility, and the ability to **backfill** past dates if needed — perfect fit for Airflow.

---

#  Part 0 — Prerequisites (Windows)

* **Docker Desktop** (with WSL2 backend enabled)
* **Git** (optional, for convenience)
* **PowerShell** terminal
* Internet access (to pull Docker images and fetch a small CSV)

> We’ll run everything in Docker containers, so no local Python/Airflow install required.

---

#  Part 1 — Project Setup & Airflow on Docker

> **Why:** Standardize your dev environment and avoid “works on my machine” issues.

### 1.1 Create a working folder

```powershell
mkdir airflow-lab; cd airflow-lab
mkdir dags logs plugins data
```

### 1.2 Create a `.env` file (fixes file permissions on Windows)

Create `.env` with this line:

```
AIRFLOW_UID=50000
```

### 1.3 Create `docker-compose.yml`

Create a file `docker-compose.yml` with the content below (LocalExecutor, Postgres DB, webserver, scheduler, triggerer, volumes for DAGs & logs):

```yaml
version: "3.8"

x-airflow-common: &airflow-common
  image: apache/airflow:2.9.3
  environment: &airflow-env
    AIRFLOW__CORE__EXECUTOR: LocalExecutor
    AIRFLOW__CORE__LOAD_EXAMPLES: "false"
    AIRFLOW__API__AUTH_BACKENDS: airflow.api.auth.backend.basic_auth
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@airflow-postgres:5432/airflow
  volumes:
    - ./dags:/opt/airflow/dags
    - ./logs:/opt/airflow/logs
    - ./plugins:/opt/airflow/plugins
    - ./data:/opt/airflow/data
  user: "${AIRFLOW_UID:-50000}:0"
  depends_on:
    - airflow-postgres

services:
  airflow-postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    ports:
      - "5433:5432"
    volumes:
      - postgres-db-volume:/var/lib/postgresql/data

  airflow-init:
    <<: *airflow-common
    entrypoint: /bin/bash
    command: -c "airflow db init && airflow users create --role Admin --username admin --password admin --firstname Admin --lastname User --email admin@example.com"
    depends_on:
      - airflow-postgres

  airflow-webserver:
    <<: *airflow-common
    command: webserver
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD-SHELL", "curl --fail http://localhost:8080/health || exit 1"]
      interval: 10s
      timeout: 10s
      retries: 5

  airflow-scheduler:
    <<: *airflow-common
    command: scheduler

  airflow-triggerer:
    <<: *airflow-common
    command: triggerer

volumes:
  postgres-db-volume:
```

### 1.4 Initialize Airflow metadata DB & admin user

```powershell
docker compose up airflow-init
```

You should see DB init and user creation logs ending successfully.

### 1.5 Start Airflow services

```powershell
docker compose up -d
```

Open **[http://localhost:8080](http://localhost:8080)**, login: **admin / admin**.

---

#  Part 2 — Configure Connections & Variables

> **Why:** Never hard-code credentials or dynamic values into code; Airflow centralizes them.

### 2.1 Create a Postgres **Connection**

* Airflow UI → **Admin → Connections → +**
* **Connection Id**: `pg_warehouse`
* **Connection Type**: Postgres
* **Host**: `airflow-postgres`
* **Schema**: `airflow`
* **Login**: `airflow`
* **Password**: `airflow`
* **Port**: `5432`
* **Save**.

### 2.2 Create **Variables**

* Airflow UI → **Admin → Variables → +**

  * **Key**: `SOURCE_URL`
    **Val**: `https://raw.githubusercontent.com/vega/vega-datasets/master/data/airports.csv`
  * **Key**: `TABLE_NAME`
    **Val**: `airports_dim`
* **Save**.

---

#  Part 3 — Author Your First ELT DAG

> **Why:** Learn basic Operators, DAG structure, retries, scheduling, and how to interact with Postgres.

Create file: `dags/elt_airports_to_postgres.py`

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.models import Variable
from airflow.providers.postgres.hooks.postgres import PostgresHook
from datetime import datetime, timedelta
import requests, csv

RAW_PATH = "/opt/airflow/data/raw.csv"
OUT_PATH = "/opt/airflow/data/airports_transformed.csv"

def check_source():
    url = Variable.get("SOURCE_URL")
    r = requests.head(url, timeout=10)
    if r.status_code >= 400:
        raise Exception(f"Source not reachable: {url}")

def download_source():
    url = Variable.get("SOURCE_URL")
    r = requests.get(url, timeout=30)
    r.raise_for_status()
    with open(RAW_PATH, "wb") as f:
        f.write(r.content)

def transform():
    with open(RAW_PATH, newline="", encoding="utf-8") as inp, \
         open(OUT_PATH, "w", newline="", encoding="utf-8") as out:
        reader = csv.DictReader(inp)
        fieldnames = ["iata","name","city","state","country"]
        writer = csv.DictWriter(out, fieldnames=fieldnames)
        writer.writeheader()
        for row in reader:
            writer.writerow({
                "iata": row.get("iata",""),
                "name": row.get("name",""),
                "city": row.get("city",""),
                "state": row.get("state",""),
                "country": row.get("country","")
            })

def load_postgres():
    table = Variable.get("TABLE_NAME", "airports_dim")
    hook = PostgresHook(postgres_conn_id="pg_warehouse")
    sql_create = f"""
    CREATE TABLE IF NOT EXISTS {table}(
        iata TEXT,
        name TEXT,
        city TEXT,
        state TEXT,
        country TEXT
    );
    TRUNCATE TABLE {table};
    """
    conn = hook.get_conn()
    with conn.cursor() as cur:
        cur.execute(sql_create)
        with open(OUT_PATH, "r", encoding="utf-8") as f:
            next(f)  # skip header
            cur.copy_from(f, table, sep=",", null="")
    conn.commit()

default_args = {
    "owner": "airflow",
    "retries": 2,
    "retry_delay": timedelta(minutes=1),
}

with DAG(
    dag_id="elt_airports_to_postgres",
    description="Fetch CSV -> transform -> load to Postgres",
    start_date=datetime(2024,1,1),
    schedule="0 2 * * *",   # every day at 02:00
    catchup=False,
    default_args=default_args,
    tags=["lab", "elt"],
) as dag:

    t1 = PythonOperator(task_id="check_source", python_callable=check_source)
    t2 = PythonOperator(task_id="download_source", python_callable=download_source)
    t3 = PythonOperator(task_id="transform_csv", python_callable=transform)
    t4 = PythonOperator(task_id="load_to_postgres", python_callable=load_postgres)

    t1 >> t2 >> t3 >> t4
```

**Explanation of tasks**

* `check_source`: ensures endpoint is live (fail fast, let Airflow retry).
* `download_source`: retrieves raw CSV into the shared `/opt/airflow/data/`.
* `transform_csv`: simple schema cleanup using Python’s `csv` (no external libs).
* `load_to_postgres`: creates/truncates a table and bulk-loads the CSV.

---

#  Part 4 — Run, Observe, and Schedule

### 4.1 Refresh the UI & trigger the DAG

* Airflow UI → **DAGs** → `elt_airports_to_postgres` → **Play ▶ Run** → **Trigger DAG**.
* Click the run to **watch the graph**, open a task → **Log** to see output.

> **Why:** See how Airflow logs every task, with retries on failure.

### 4.2 Verify data in Postgres

Open a psql shell inside the Postgres container:

```powershell
docker compose exec airflow-postgres psql -U airflow -d airflow -c "SELECT * FROM airports_dim LIMIT 10;"
```

### 4.3 See scheduling

The DAG is scheduled for **02:00 daily**. You can change `schedule` to `@hourly` for testing, or trigger manually anytime.

### 4.4 Backfill past dates (optional)

> **Why:** Production pipelines must reprocess history if needed.

```powershell
docker compose exec airflow-scheduler airflow dags backfill -s 2024-01-01 -e 2024-01-03 elt_airports_to_postgres
```

---

#  Part 5 — Try “Ops” Basics

### 5.1 Break it on purpose (observe retries)

Edit Variable `SOURCE_URL` to something invalid → trigger DAG → watch **retries** and **task states**. Restore URL when done.

### 5.2 Clear a failed task & rerun

From the failed task → **Clear** → select “Downstream” or “Future” as needed → **OK**.

### 5.3 Pause/Unpause the DAG

Use the UI **toggle** to pause scheduling when making code changes.

---

#  Cleanup

```powershell
docker compose down -v
# optional: remove the airflow-lab folder afterwards
```

---

##  What You Learned

* What **Airflow** is and why orchestration matters.
* How to deploy Airflow locally with **Docker Compose** (Windows-friendly).
* How to create a real **ELT DAG** with PythonOperators.
* How to use **Connections** & **Variables** for clean, reusable code.
* How to **trigger**, **schedule**, **backfill**, and **debug** tasks in the UI.

---
