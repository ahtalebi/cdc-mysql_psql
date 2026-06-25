Stack: spun up MySQL (source), PostgreSQL (sink), Kafka, Zookeeper, and Debezium Kafka Connect via docker-compose up -d

Port conflicts: changed MySQL to 3307:3306 and PostgreSQL to 5433:5432 to avoid clashes with local installs

JDBC Sink plugin: downloaded confluentinc-kafka-connect-jdbc-10.9.5.zip, unzipped it, copied re2j-1.7.jar and postgresql-42.7.3.jar into its lib/ folder, then mounted the whole folder into the connect container via docker-compose.yml volumes

MySQL privileges: granted full privileges to the debezium user via GRANT ALL PRIVILEGES ON *.* TO 'debezium'@'%'

MySQL source connector: registered 01-mysql-source.json — tells Debezium to read MySQL binlog and publish changes to Kafka topic cdc.shop.orders

PostgreSQL sink connector: registered 02-postgres-sink.json — reads from cdc.shop.orders topic and applies changes to PostgreSQL using JDBC upsert

SMT (Single Message Transform): ExtractNewRecordState in the sink config unwraps Debezium's complex event envelope into a plain row format PostgreSQL understands

Auto table creation: auto.create: true in the sink config meant PostgreSQL table was created automatically on first event
Verified CDC: inserted a row in MySQL, it appeared in PostgreSQL within seconds

Key flow: MySQL binlog → Debezium → Kafka topic → JDBC Sink → PostgreSQL


connectors/ — both files are needed:

01-mysql-source.json — tells Debezium what to read from MySQL
02-postgres-sink.json — tells Debezium where to write in PostgreSQL

scripts/test-crud.sh — not needed, just a helper script you can ignore now that you're running commands manually
docker-compose.yml — needed, defines all containers


The Files
docker-compose.yml

Defines all 5 containers and how they talk to each other. The backbone of the whole experiment.
connectors/01-mysql-source.json

Tells Debezium: "watch MySQL's binlog, capture every INSERT/UPDATE/DELETE on shop.orders and publish to Kafka topic cdc.shop.orders"
connectors/02-postgres-sink.json

Tells Debezium: "read from Kafka topic cdc.shop.orders and apply every change into PostgreSQL". Uses JDBC upsert so INSERT/UPDATE become upserts and DELETE removes the row.
extra-plugins/confluentinc-kafka-connect-jdbc-10.9.5/

The actual JDBC sink connector plugin. Without this, Debezium has no idea how to write to PostgreSQL.
extra-plugins/postgresql-42.7.3.jar (inside lib/)

The PostgreSQL JDBC driver. The JDBC connector uses this to actually open a connection to PostgreSQL.
extra-plugins/re2j-1.7.jar (inside lib/)

A regex library that the JDBC connector depends on internally. Missing this caused our 500 errors earlier.
scripts/test-crud.sh

A convenience script that runs INSERT/UPDATE/DELETE and prints both databases. Not essential.

What We Just Built
MySQL (source)
  │  binlog (row-level change log)
  ▼
Debezium Source Connector
  │  reads binlog, converts to JSON events
  ▼
Kafka Topic: cdc.shop.orders
  │  stores events as messages
  ▼
Debezium Sink Connector (JDBC)
  │  reads messages, runs SQL on PostgreSQL
  ▼
PostgreSQL (sink/replica)
Every time you INSERT/UPDATE/DELETE in MySQL, it shows up in PostgreSQL within seconds.

Kafka vs Debezium — Are They the Same?
No, they are completely separate:
DebeziumKafkaWhat it isCDC tool, reads database logsDistributed message busRole hereCaptures MySQL changes, writes to PostgreSQLStores and transports the change eventsWithout the otherDebezium needs somewhere to put eventsKafka is just a pipe, has no DB knowledgeContainerconnectkafka + zookeeper
Think of it this way:

Kafka is the highway
Debezium is the truck that picks up cargo (DB changes) and delivers it
Zookeeper is the traffic controller for Kafka

Debezium runs inside Kafka Connect (the connect container) as a plugin. That's why they feel like one thing but they're not.

Kafka UI
Right now you have no UI — you're flying blind. Kafka UI would let you:

See all topics (cdc.shop.orders, etc.)
Browse messages — see the raw JSON Debezium events
Monitor lag — is the sink keeping up with the source?
Debug — see exactly what event was published when you did that INSERT

