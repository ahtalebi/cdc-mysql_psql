# 🚀 MySQL → Kafka → PostgreSQL CDC Pipeline with Debezium

A simple end-to-end **Change Data Capture (CDC)** pipeline that replicates changes from **MySQL** to **PostgreSQL** in near real time using:

* 🐬 MySQL (Source Database)
* 🦠 Debezium
* 📨 Apache Kafka
* 🔌 Kafka Connect + JDBC Sink Connector
* 🐘 PostgreSQL (Replica/Sink)

Every `INSERT`, `UPDATE`, and `DELETE` executed on MySQL is automatically propagated to PostgreSQL within seconds.

---

# 🏗 Architecture

```text
MySQL (source)
   │  binlog (row-level change log)
   ▼
Debezium Source Connector
   │  reads binlog and emits JSON events
   ▼
Kafka Topic: cdc.shop.orders
   │  stores change events as messages
   ▼
Debezium JDBC Sink Connector
   │  consumes messages and executes SQL
   ▼
PostgreSQL (sink/replica)
```

---

# 📸 Concept

Think of it like this:

* **MySQL** generates change events.
* **Debezium** captures those changes.
* **Kafka** transports the events.
* **JDBC Sink** applies the changes.
* **PostgreSQL** becomes a near real-time replica.

```text
Database Changes → Event Stream → Data Replication
```

---

# 🐳 Infrastructure

Everything runs in Docker via:

```bash
docker-compose up -d
```

The stack consists of:

| Service                  | Purpose                        |
| ------------------------ | ------------------------------ |
| MySQL                    | Source database                |
| PostgreSQL               | Replica/Sink database          |
| Kafka                    | Event streaming platform       |
| Zookeeper                | Kafka coordination             |
| Kafka Connect + Debezium | CDC source and sink connectors |

---

# ⚠️ Port Changes

To avoid conflicts with locally installed databases:

| Service    | Default | Used |
| ---------- | ------- | ---- |
| MySQL      | 3306    | 3307 |
| PostgreSQL | 5432    | 5433 |

---

# 🔌 JDBC Sink Connector Setup

Downloaded:

```text
confluentinc-kafka-connect-jdbc-10.9.5.zip
```

Additional dependencies added:

```text
postgresql-42.7.3.jar
re2j-1.7.jar
```

These files were copied into the connector's `lib/` directory and mounted into the `connect` container via Docker volumes.

---

# 🗄 MySQL Permissions

Debezium requires access to MySQL's binary log.

```sql
GRANT ALL PRIVILEGES ON *.* TO 'debezium'@'%';
FLUSH PRIVILEGES;
```

---

# ⚙️ Connectors

## Source Connector

`connectors/01-mysql-source.json`

Responsible for:

* Reading MySQL binlogs
* Capturing row-level changes
* Publishing events to:

```text
cdc.shop.orders
```

---

## Sink Connector

`connectors/02-postgres-sink.json`

Responsible for:

* Reading from `cdc.shop.orders`
* Writing changes into PostgreSQL
* Performing UPSERT operations
* Handling DELETE events

---

# 🔄 Single Message Transform (SMT)

The sink connector uses:

```text
ExtractNewRecordState
```

Debezium events are normally wrapped inside a complex envelope:

```json
{
  "before": {},
  "after": {},
  "source": {},
  "op": "c"
}
```

The SMT unwraps this into a plain row structure that PostgreSQL can understand.

---

# 🪄 Automatic Table Creation

The sink connector is configured with:

```json
"auto.create": true
```

This means the PostgreSQL table is automatically created when the first event arrives.

---

# ✅ Verification

Inserted a row into MySQL:

```sql
INSERT INTO orders (...) VALUES (...);
```

Within seconds, the same row appeared in PostgreSQL.

CDC replication is working 🎉

---

# 📂 Project Structure

```text
.
├── docker-compose.yml
├── connectors
│   ├── 01-mysql-source.json
│   └── 02-postgres-sink.json
├── extra-plugins
│   ├── confluentinc-kafka-connect-jdbc-10.9.5
│   ├── postgresql-42.7.3.jar
│   └── re2j-1.7.jar
└── scripts
    └── test-crud.sh
```

---

# 📄 File Breakdown

## docker-compose.yml

Defines all containers and networking.

---

## connectors/01-mysql-source.json

Tells Debezium:

> Watch MySQL's binlog and publish every change to Kafka.

---

## connectors/02-postgres-sink.json

Tells Debezium:

> Read Kafka events and apply them to PostgreSQL.

---

## extra-plugins/

Contains the JDBC Sink connector and its dependencies.

Without these files, PostgreSQL replication cannot work.

---

## scripts/test-crud.sh

Convenience script for:

* INSERT
* UPDATE
* DELETE
* Verifying both databases

Optional and not required.

---

# 🤔 Kafka vs Debezium

They are **not** the same thing.

| Component | Purpose                                         |
| --------- | ----------------------------------------------- |
| Debezium  | Reads database logs and generates change events |
| Kafka     | Stores and transports events                    |
| Zookeeper | Coordinates Kafka                               |

### Mental Model

```text
Kafka      = Highway
Debezium   = Truck carrying database changes
Zookeeper  = Traffic controller
```

Debezium runs **inside Kafka Connect** as a plugin, which is why they often appear to be one system.

---

# 📊 Kafka UI (Recommended)

Adding a Kafka UI makes debugging much easier.

Benefits:

* Browse topics
* View raw CDC events
* Monitor consumer lag
* Debug connector issues
* Inspect event payloads

Useful for seeing exactly what happened after an `INSERT`, `UPDATE`, or `DELETE`.

---

# 🎯 Final Result

```text
MySQL
   ↓
Debezium Source Connector
   ↓
Kafka Topic
   ↓
JDBC Sink Connector
   ↓
PostgreSQL
```

A lightweight, near real-time **ETL/CDC pipeline** built entirely with open-source tools.

