# Apache Kafka®

## What is Apache Kafka?

Apache Kafka is an open-source event streaming platform used to collect, process, store, and integrate data at scale in real time. It has numerous use cases including stream processing, data integration, and pub/sub messaging.

Kafka was originally developed at LinkedIn, was open sourced in 2011, and became an Apache Software Foundation project in 2012. It is used by thousands of organizations globally to power mission-critical real-time applications, from stock exchanges, to e-commerce applications, to IoT monitoring & analytics, to name a few.

## Quick start

Start a Kafka broker:

```noformat
docker run -d --name broker apache/kafka:latest
```

Open a shell in the broker container:

```noformat
docker exec --workdir /opt/kafka/bin/ -it broker sh
```

A _topic_ is a logical grouping of events in Kafka. From inside the container, create a topic called `test-topic`:

```noformat
./kafka-topics.sh --bootstrap-server localhost:9092 --create --topic test-topic
```

Write two string events into the `test-topic` topic using the console producer that ships with Kafka:

```
./kafka-console-producer.sh --bootstrap-server localhost:9092 --topic test-topic
```

This command will wait for input at a `>` prompt. Enter `hello`, press `Enter`, then `world`, and press `Enter again`. Enter `Ctrl+C` to exit the console producer.

Now read the events in the `test-topic` topic from the beginning of the log:

```
./kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test-topic --from-beginning
```

You will see the two strings that you previously produced:

```
hello
world
```

The consumer will continue to run until you exit out of it by entering `Ctrl+C`.

## Overriding the default broker configuration

Apache Kafka supports a broad set of broker configurations that you may override via environment variables. The environment variables must begin with `KAFKA_`, and any dots in broker configurations should be specified as underscores in the corresponding environment variable. For example, to set the default number of partitions in topics, [`num.partitions`](https://kafka.apache.org/documentation/#brokerconfigs_num.partitions), set the environment variable `KAFKA_NUM_PARTITIONS`. See [here](https://github.com/apache/kafka/blob/trunk/docker/examples/README.md) for more information on overriding broker configuration in Docker.

It's important to note that, if you are overriding _any_ configuration, then _none_ of the default configurations will be used. For example, to run Kafka in KRaft [combined mode](https://kafka.apache.org/documentation/#kraft_role) (meaning that the broker handling client requests and the controller handling cluster coordination both run in the same container) and set the default number of topic partitions to 3 instead of the default 1, we would specify `KAFKA_NUM_PARTITIONS` in addition to other required configurations:

```
docker run -d  \
  --name broker \
  -e KAFKA_NODE_ID=1 \
  -e KAFKA_PROCESS_ROLES=broker,controller \
  -e KAFKA_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093 \
  -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092 \
  -e KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER \
  -e KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT \
  -e KAFKA_CONTROLLER_QUORUM_VOTERS=1@localhost:9093 \
  -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
  -e KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR=1 \
  -e KAFKA_TRANSACTION_STATE_LOG_MIN_ISR=1 \
  -e KAFKA_NUM_PARTITIONS=3 \
  apache/kafka:latest
```



docker compose way:

save `docker-compose.yml`
```yaml
services:
  broker:
    image: apache/kafka:latest
    container_name: kafka
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://localhost:9092,CONTROLLER://localhost:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@localhost:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_NUM_PARTITIONS: 3
```
run `docker compose up -d`

External clients

Two steps needed:

1. expose port: docker run -d -p 9092:9092 --name broker apache/kafka:latest
2. download and unzip the latest Kafka release: https://kafka.apache.org/documentation/#quickstart_download. then navigate into the distribution's `bin` directory and then run the same steps as before (localhost:9092 as bootstrap servers, but here localhost is your host machine and not the container)

KRaft isolated mode

- no need for separate advertised listeners in the broker
```yaml
services:
  controller-1:
    image: apache/kafka:latest
    hostname: controller-1
    container_name: controller-1
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: controller
      KAFKA_LISTENERS: CONTROLLER://:9093
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@controller-1:9093,2@controller-2:9093,3@controller-3:9093
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0

  controller-2:
    image: apache/kafka:latest
    hostname: controller-2
    container_name: controller-2
    environment:
      KAFKA_NODE_ID: 2
      KAFKA_PROCESS_ROLES: controller
      KAFKA_LISTENERS: CONTROLLER://:9093
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@controller-1:9093,2@controller-2:9093,3@controller-3:9093
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0

  controller-3:
    image: apache/kafka:latest
    hostname: controller-3
    container_name: controller-3
    environment:
      KAFKA_NODE_ID: 3
      KAFKA_PROCESS_ROLES: controller
      KAFKA_LISTENERS: CONTROLLER://:9093
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@controller-1:9093,2@controller-2:9093,3@controller-3:9093
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0

  broker-1:
    image: apache/kafka:latest
    hostname: broker-1
    container_name: broker-1
    ports:
      - 29092:9092
    environment:
      KAFKA_NODE_ID: 4
      KAFKA_PROCESS_ROLES: broker
      KAFKA_LISTENERS: 'PLAINTEXT://:19092,PLAINTEXT_HOST://:9092'
      KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://broker-1:19092,PLAINTEXT_HOST://localhost:29092'
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@controller-1:9093,2@controller-2:9093,3@controller-3:9093
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
    depends_on:
      - controller-1
      - controller-2
      - controller-3

  broker-2:
    image: apache/kafka:latest
    hostname: broker-2
    container_name: broker-2
    ports:
      - 39092:9092
    environment:
      KAFKA_NODE_ID: 5
      KAFKA_PROCESS_ROLES: broker
      KAFKA_LISTENERS: 'PLAINTEXT://:19092,PLAINTEXT_HOST://:9092'
      KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://broker-2:19092,PLAINTEXT_HOST://localhost:39092'
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@controller-1:9093,2@controller-2:9093,3@controller-3:9093
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
    depends_on:
      - controller-1
      - controller-2
      - controller-3

  broker-3:
    image: apache/kafka:latest
    hostname: broker-3
    container_name: broker-3
    ports:
      - 49092:9092
    environment:
      KAFKA_NODE_ID: 6
      KAFKA_PROCESS_ROLES: broker
      KAFKA_LISTENERS: 'PLAINTEXT://:19092,PLAINTEXT_HOST://:9092'
      KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://broker-3:19092,PLAINTEXT_HOST://localhost:49092'
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@controller-1:9093,2@controller-2:9093,3@controller-3:9093
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
    depends_on:
      - controller-1
      - controller-2
      - controller-3
```

Then similar in-broker commands work as before.

./kafka-topics.sh --bootstrap-server broker-1:19092,broker-2:19092,broker-3:19092 --create --topic test-topic
./kafka-console-producer.sh --bootstrap-server broker-1:19092,broker-2:19092,broker-3:19092 --topic test-topic
./kafka-console-consumer.sh --bootstrap-server broker-1:19092,broker-2:19092,broker-3:19092 --topic test-topic --from-beginning


From outside docker:

./kafka-topics.sh --bootstrap-server localhost:29092,localhost:39092,localhost:49092 --create --topic test-topic2
./kafka-console-producer.sh --bootstrap-server localhost:29092,localhost:39092,localhost:49092 --topic test-topic2
./kafka-console-consumer.sh --bootstrap-server localhost:29092,localhost:39092,localhost:49092 --topic test-topic2 --from-beginning

