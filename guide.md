**Project Guide**

An end-to-end demo that simulates an e-commerce OLTP system, captures CDC with Debezium, streams via Kafka, processes with Apache Spark (Structured Streaming), and stores results in ClickHouse. The stack is containerized and orchestrated with Docker Compose and Airflow.

**Quick Start (Arch Linux)**

- **Prereqs:** Install Docker and Compose, enable Docker daemon, add your user to the `docker` group.
- **Repository files:** See `README.md`, `docker-compose.yml`, `register-postgres.json`, `init-db.sql`, `init-clickhouse.sql`, `spark-jobs/`, `data-generator/`, `script/`.

**Commands**

```bash
# Install Docker & Compose on Arch
sudo pacman -Syu docker docker-compose
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker

# Prepare repository
cd /home/lazzerex/realtime-cdc-pipeline-docker
chmod +x script/*.sh
mkdir -p logs dags
sudo chown -R $USER:$USER logs dags
chmod -R 755 logs dags

# Start the entire stack
docker compose up --build -d
# or: docker-compose up --build -d

# After Connect is healthy, register the Debezium connector
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" \
  http://localhost:8083/connectors/ -d @register-postgres.json

# Create ClickHouse database (if needed)
curl -u admin:admin 'http://localhost:8123/' --data-binary 'CREATE DATABASE IF NOT EXISTS cdc_data;'

# Tail important logs
docker logs -f data-generator
docker logs -f spark-streaming
docker logs -f airflow-webserver
```

**Recommended Startup Order**

- **Why:** the stack has dependencies (Zookeeper → Kafka broker → Schema Registry/Connect → processing/sinks). Starting core services in order avoids race conditions and missing topics. Could also avoid RAM related problems. You can try to increase Swap Memory.
- **Recommended sequence (examples):**

```bash
# 1) Start Zookeeper
docker compose up -d zookeeper

# 2) Start Kafka broker
docker compose up -d broker

# 3) Start Schema Registry and Connect (Debezium)
docker compose up -d schema-registry connect

# 4) Start PostgreSQL (source) and ClickHouse (sink)
docker compose up -d postgres clickhouse

# 5) Start Spark (master, worker, streaming job)
docker compose up -d spark-master spark-worker spark-streaming

# 6) Start data generator and Airflow services
docker compose up -d data-generator airflow-webserver airflow-scheduler

# Or, if you prefer to start everything at once, run:
docker compose up -d
```

Note: `docker compose up -d` will attempt to start all services, but explicit ordering above reduces transient failures (missing topics, DB init races).

**Data safety / Important warning**

- Do NOT remove `./postgres_data`, `./kafka_data`, or `./clickhouse-data` unless you intentionally want to wipe persisted state. Deleting these directories removes database and Kafka topic metadata and can lead to cluster ID mismatches or lost schema/data.
- If you must wipe volumes, stop services first, and then recreate any host directories with permissive permissions so containers can initialize them, for example:

```bash
docker compose down
mkdir -p ./clickhouse-data ./postgres_data ./kafka_data
chmod 777 ./clickhouse-data ./postgres_data ./kafka_data
docker compose up -d
```

- Recommended: back up any host directories before removing them. If ClickHouse reports filesystem/metadata errors after a cleanup, recreate the directory and ensure the container can write to `/var/lib/clickhouse`.

**What to verify**

- **Connectors & topics:** Visit http://localhost:8083 (Debezium Connect) and confirm connector listed. Check Kafka topics via Schema Registry or Control Center.
- **Topic presence:** `spark-entrypoint.sh` waits for the topic `cdc.public.orders` — ensure it's created after registering the connector.
- **Airflow UI:** http://localhost:8080
- **ClickHouse HTTP:** http://localhost:8123
- **Spark UI:** http://localhost:9090

**Key Files**

- `docker-compose.yml`: Service definitions for Zookeeper, Kafka, Schema Registry, Debezium, Postgres, Spark, Airflow, ClickHouse.
- `init-db.sql`: Creates `customers`, `products`, `orders`, `order_items` tables.
- `register-postgres.json`: Debezium connector configuration (register to `http://localhost:8083/connectors/`).
- `data-generator/main.py`: Inserts static data (customers/products) and simulates ongoing orders and updates.
- `script/spark-entrypoint.sh`: Waits for Kafka topic then runs `spark-submit` to start `cdc_processor.py`.
- `spark-jobs/cdc_processor.py`: Reads Debezium JSON from Kafka, applies CDC logic, writes to ClickHouse.

**Troubleshooting**

- If `spark-streaming` waits for `cdc.public.orders` topic: ensure the connector POST returned success and the connector is healthy. Check Connect logs:

```bash
docker logs connect
```

- If ClickHouse writes fail: ensure database `cdc_data` exists and ClickHouse credentials match (default `admin:admin`). Inspect `clickhouse` container logs.

- If Airflow fails on startup: the `entrypoint.sh` installs OS deps and python packages; check `airflow-webserver` logs for pip or db upgrade errors.

- If Kafka services don't start or topics unavailable: check `broker`, `zookeeper`, and `schema-registry` container logs and wait for their healthchecks.

**Cleaning up Docker (stop & remove)**

Use these commands to stop the demo and remove containers, networks, volumes, and images created by the compose stack. Run these on your host where Docker runs.

```bash
# Stop and remove containers, networks defined in compose
docker compose down
# Remove anonymous volumes created by compose
docker compose down --volumes

# If you used the legacy binary
# docker-compose down
# docker-compose down --volumes

# Remove named volumes or directories if you want to remove persisted data
rm -rf ./postgres_data ./kafka_data ./clickhouse-data ./logs

# Optional: remove images built locally by this repo (use with care)
docker image prune -a --filter "label=io.compose.project=realtime-cdc-pipeline-docker" --force || true

# Global cleanup (use with caution) - frees space but may remove other images/containers
docker system prune -a --volumes --force
```

**Quick Validation Queries**

```bash
# Check connector list
curl -s http://localhost:8083/connectors | jq .

# Check topics listed by kafkacat (requires kafkacat locally)
kafkacat -b localhost:9092 -L -J | jq '.topics[] | select(.topic | test("cdc\\."))'

# Query ClickHouse counts
curl -s -u admin:admin "http://localhost:8123/?query=SELECT%20count()%20FROM%20cdc_data.orders"
```
