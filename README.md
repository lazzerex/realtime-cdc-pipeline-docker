An End-to-End Data Engineering project that simulates an E-commerce platform, captures real-time data changes (CDC), processes them using Apache Spark, and stores them in ClickHouse for analytics. The entire workflow is containerized using Docker and orchestrated by Airflow.

## Architecture

The pipeline follows a modern **Kappa Architecture** pattern:
<img width="1054" height="403" alt="image" src="https://github.com/user-attachments/assets/3e4faa18-326d-40ec-809e-8df55ab5d541" />

Data Source (Mock E-commerce): A Python script generating random Customers, Products, Orders, and Order Items.
OLTP Database: PostgreSQL 16 (configured with wal_level=logical).
Ingestion (CDC): Debezium (running on Kafka Connect) captures row-level changes.
Message Broker: Confluent Kafka & Zookeeper.
Stream Processing: Apache Spark 3.5 (PySpark) reads from Kafka, flattens JSON payloads, handles CDC logic (insert/update/delete), and writes to ClickHouse.
OLAP Sink: ClickHouse (MergeTree engine) for high-performance analytics.
Orchestration: Apache Airflow 2.10 managing batch jobs and reporting.

## Tech Stack
Language: Python (PySpark, Faker, Airflow DAGs)
Containerization: Docker, Docker Compose
Databases: PostgreSQL 16, ClickHouse 24+
Streaming: Apache Kafka, Debezium 2.6
Processing: Apache Spark 3.5 (Structured Streaming)
Orchestration: Apache Airflow 2.10

## Quick Start

1. Clone the repository

```bash
git clone git@github.com:HowardZeng123/realtime-cdc-pipeline-docker.git
cd realtime-cdc-pipeline-docker
```

2. Grant permissions (Linux/WSL only)

```bash
chmod +x script/*.sh
mkdir -p logs
chmod -R 777 logs
```

3. Recommended startup order

Follow this ordered sequence to avoid race conditions and missing topics. You can start services individually or run `docker compose up -d` to start all at once.

```bash
# 1) Zookeeper
docker compose up -d zookeeper

# 2) Kafka broker
docker compose up -d broker

# 3) Schema Registry and Debezium Connect
docker compose up -d schema-registry connect

# 4) PostgreSQL and ClickHouse
docker compose up -d postgres clickhouse

# 5) Spark services (master, worker, streaming)
docker compose up -d spark-master spark-worker spark-streaming

# 6) Data generator and Airflow
docker compose up -d data-generator airflow-webserver airflow-scheduler

# Or start everything at once
docker compose up -d
```

4. Register the Debezium connector (after Connect is healthy)

```bash
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" \
	http://localhost:8083/connectors/ -d @register-postgres.json
```

5. For full, step-by-step instructions and troubleshooting see [guide.md](guide.md).

### Data safety (IMPORTANT)

- Do NOT delete `./postgres_data`, `./kafka_data`, or `./clickhouse-data` unless you intend to wipe persisted state. Removing these directories will erase databases, Kafka metadata (including cluster IDs and topics), and ClickHouse metadata which can cause services to fail to start.
- If you must remove or reset volumes: stop the stack first, back up the directories, recreate them with proper permissions, then bring services up again (see `guide.md` for commands).

```bash
docker compose down
mkdir -p ./clickhouse-data ./postgres_data ./kafka_data
chmod 777 ./clickhouse-data ./postgres_data ./kafka_data
docker compose up -d
```
