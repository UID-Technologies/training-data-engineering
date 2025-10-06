#  Lab 11 — **Apache Kafka Producer/Consumer on Windows (Docker)**

## Overview

Stand up a single-broker Kafka locally (Docker Desktop), create a topic, produce & consume messages with CLI tools and a small Python app.

## Scenario

Your team prototypes a streaming pipeline locally before deploying to a managed Kafka (e.g., **Amazon MSK**). You need a quick way to run Kafka and verify producer/consumer logic.

## Why Kafka (important features)

* **Topics/partitions** for horizontal scalability and ordering within a partition.
* **Consumer groups** for parallel processing and offset management.
* **Broad ecosystem** (Connect, Schema Registry, ksqlDB) and standard protocol/clients. ([Apache Kafka][5])

> Prefer managed Kafka on AWS? **MSK Serverless** removes capacity management and scales on demand. ([AWS Documentation][6])

## Common use cases

Event-driven microservices, log aggregation, CDC pipelines, stream processing (Flink/Spark), and audit logs. ([Apache Kafka][7])

---

## Step-by-step (Windows-friendly)

> Prereqs: **Docker Desktop for Windows**. Open PowerShell in an empty folder.

### 0) Create `docker-compose.yml` (Why: one-command Kafka up)

```yaml
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    ports: ["2181:2181"]

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on: [zookeeper]
    ports: ["9092:9092"]
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
```

> This uses Confluent’s Docker images (common for local labs). ([Confluent Documentation][8])

Start it:

```powershell
docker compose up -d
docker ps
```

### 1) Create a topic (Why: storage unit for events)

```powershell
docker exec -it $(docker ps -qf "name=kafka") \
  kafka-topics --create --topic orders --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
```

Kafka’s quickstart uses similar commands to create topics & test flows. ([Apache Kafka][5])

### 2) Produce test messages with the **console producer** (Why: quick sanity check)

```powershell
docker exec -it $(docker ps -qf "name=kafka") \
  kafka-console-producer --topic orders --bootstrap-server localhost:9092
# type a few lines, then Ctrl+C to exit
```

CLI producer/consumer is the fastest way to prove your setup. ([Confluent][9])

### 3) Consume with the **console consumer** (Why: verify receive path & consumer groups)

```powershell
docker exec -it $(docker ps -qf "name=kafka") \
  kafka-console-consumer --topic orders --bootstrap-server localhost:9092 --from-beginning --group lab-consumers
```

Consumer groups coordinate parallel reads and track offsets. ([Apache Kafka][5])

### 4) Python producer with `confluent-kafka` (Why: realistic app code)

Install lib and create `producer_kafka.py`:

```powershell
pip install confluent-kafka
```

```python
from confluent_kafka import Producer
import json, time, uuid, random

p = Producer({"bootstrap.servers":"localhost:9092"})
users = [f"user-{i}" for i in range(1,6)]
items = ["book","pen","bag","mouse","keyboard"]

def delivery(err, msg):
    print(("ERR: "+str(err)) if err else f"DELIVERED: {msg.topic()} [{msg.partition()}] @ {msg.offset()}")

while True:
    evt = {"order_id":str(uuid.uuid4()), "user_id":random.choice(users),
           "item":random.choice(items)}
    p.produce("orders", json.dumps(evt).encode("utf-8"),
              key=evt["user_id"], callback=delivery)
    p.poll(0)
    time.sleep(0.5)
```

### 5) Python consumer (Why: read with a group)

Create `consumer_kafka.py`:

```python
from confluent_kafka import Consumer
import json

c = Consumer({
    "bootstrap.servers":"localhost:9092",
    "group.id":"lab-consumers",
    "auto.offset.reset":"earliest"
})
c.subscribe(["orders"])
try:
    while True:
        msg = c.poll(1.0)
        if msg is None: continue
        if msg.error():
            print("ERR:", msg.error()); continue
        print("Got:", json.loads(msg.value().decode("utf-8")))
finally:
    c.close()
```

Run both Python apps and watch events flow.

### 6) Inspect partitions & groups (Why: understand scaling)

```powershell
docker exec -it $(docker ps -qf "name=kafka") \
  kafka-consumer-groups --bootstrap-server localhost:9092 --group lab-consumers --describe

docker exec -it $(docker ps -qf "name=kafka") \
  kafka-topics --describe --topic orders --bootstrap-server localhost:9092
```

### 7) Clean up

```powershell
docker compose down -v
```

---

## (Optional) Where this goes in AWS

* Replace local Kafka with **Amazon MSK (or MSK Serverless)**, keep your producers/consumers the same. MSK manages brokers/partitions and scales capacity for you. ([AWS Documentation][10])

---

## Compare at a glance

| Capability     | Kinesis Data Streams                       | Apache Kafka (local / MSK)          |
| -------------- | ------------------------------------------ | ----------------------------------- |
| Scaling unit   | **Shard** (managed)                        | **Partition/Broker**                |
| Ordering       | Per partition key within shard             | Per key within partition            |
| Consumer model | Polling API; optional **enhanced fan-out** | Pull by **consumer groups**         |
| Ops model      | Fully managed stream                       | Local (Docker) or managed (**MSK**) |

(Concept references: Kinesis shards/partition key, enhanced fan-out; Kafka quickstart & consumer groups.) ([AWS Documentation][1])

---
