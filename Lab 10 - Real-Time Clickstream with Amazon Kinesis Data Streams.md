# **Lab 10 — Real-Time Clickstream with Amazon Kinesis Data Streams**

## Description

Build a streaming pipeline with **Kinesis Data Streams**. You’ll create a stream, write (produce) JSON events with Python, read (consume) those events, scale shards, and clean up.

## Scenario

A web app emits clickstream events (user id, page URL, timestamp). Product analytics needs a low-latency feed to power dashboards and alerts.

## Why Kinesis (important features)

* **Shards & partition keys** to parallelize writes/reads; records for a partition key are routed to the same shard. ([AWS Documentation][1])
* **Elastic scaling** by increasing shard count.
* **Consumers** via SDK polling or **enhanced fan-out** (dedicated per-consumer throughput). ([Amazon Web Services, Inc.][2])
* **Managed, serverless ingestion** with simple SDKs (boto3). ([Boto3][3])

## Common use cases

Clickstream analytics, IoT telemetry, log ingestion, near-real-time transformations and ML feature feeds.

---

## Step-by-step (Windows-friendly)

> Prereqs: **AWS CLI**, **Python 3.9+**, **pip**, and an IAM identity with Kinesis permissions.
> Open **Windows PowerShell** for the commands below.

### 0) Set your lab variables (Why: reuse them consistently)

```powershell
$env:AWS_REGION = "us-east-1"
$env:STREAM     = "clickstream-lab"
```

### 1) Create a Kinesis stream (Why: the logical pipe for events)

```powershell
aws kinesis create-stream --stream-name $env:STREAM --shard-count 1 --region $env:AWS_REGION
aws kinesis describe-stream-summary --stream-name $env:STREAM --region $env:AWS_REGION
```

Kinesis divides a stream into **shards**; partition keys decide which shard receives a record. ([AWS Documentation][1])

### 2) Create a simple **Python producer** (Why: send JSON events)

Create `producer_kinesis.py`:

```python
import json, time, uuid, random, boto3, datetime
import os
region  = os.getenv("AWS_REGION","us-east-1")
stream  = os.getenv("STREAM","clickstream-lab")
kinesis = boto3.client("kinesis", region_name=region)

users = [f"user-{i}" for i in range(1,6)]
pages = ["/", "/home", "/products", "/cart", "/checkout"]

while True:
    event = {
        "event_id": str(uuid.uuid4()),
        "user_id": random.choice(users),
        "url":     random.choice(pages),
        "ts":      datetime.datetime.utcnow().isoformat()+"Z"
    }
    data = json.dumps(event).encode("utf-8")
    # Partition key keeps a user's events ordered in the same shard
    kinesis.put_record(StreamName=stream, Data=data, PartitionKey=event["user_id"])
    print("Sent:", event)
    time.sleep(0.5)
```

Run it:

```powershell
$env:AWS_REGION=$env:AWS_REGION; $env:STREAM=$env:STREAM
python .\producer_kinesis.py
```

### 3) Create a **Python consumer** (Why: read from a shard iterator)

Create `consumer_kinesis.py`:

```python
import boto3, json, base64, os, time
region = os.getenv("AWS_REGION","us-east-1")
stream = os.getenv("STREAM","clickstream-lab")
k = boto3.client("kinesis", region_name=region)

# Get shard id (single shard in this lab)
shards = k.describe_stream(StreamName=stream)["StreamDescription"]["Shards"]
shard_id = shards[0]["ShardId"]

# Get iterator from TRIM_HORIZON to read from earliest records
it = k.get_shard_iterator(StreamName=stream, ShardId=shard_id,
                          ShardIteratorType="TRIM_HORIZON")["ShardIterator"]

while True:
    resp = k.get_records(ShardIterator=it, Limit=100)
    it = resp["NextShardIterator"]
    for rec in resp["Records"]:
        print("Got:", json.loads(rec["Data"].decode("utf-8")))
    time.sleep(1)
```

This uses `get_shard_iterator` to start reading and polls with `get_records`. ([Boto3][4])

Run it (in a second terminal):

```powershell
$env:AWS_REGION=$env:AWS_REGION; $env:STREAM=$env:STREAM
python .\consumer_kinesis.py
```

### 4) Scale the stream (Why: increase parallelism/throughput)

```powershell
aws kinesis update-shard-count `
  --stream-name $env:STREAM --scaling-type UNIFORM_SCALING --target-shard-count 2 `
  --region $env:AWS_REGION
```

Shards scale write/read capacity and parallelism; partition keys distribute load across shards. ([AWS Documentation][1])

> Optional: Explore **enhanced fan-out** consumers for dedicated read throughput per consumer. ([Amazon Web Services, Inc.][2])

### 5) Clean up (Why: avoid charges)

```powershell
aws kinesis delete-stream --stream-name $env:STREAM --enforce-consumer-deletion --region $env:AWS_REGION
```

---
