# Apache Flink®

## What is Apache Flink?

Apache Flink is an open-source event streaming platform used to collect, process, store, and integrate data at scale in real time. It powers numerous use cases including stream processing, data integration, and pub/sub messaging.

Flink was originally developed at LinkedIn, was open sourced in 2011, and became an Apache Software Foundation project in 2012. It is used by thousands of organizations globally to power mission-critical real-time applications, from stock exchanges, to e-commerce applications, to IoT monitoring & analytics, to name a few.

## Quick start

Start a Flink broker:

```noformat
docker run -d --name broker apache/flink:latest
```

Open a shell in the broker container:

```noformat
docker exec --workdir /opt/flink/bin/ -it broker sh
```

A _topic_ is a logical grouping of events in Flink. From inside the container, create a topic called `test-topic`:

```noformat
./flink-topics.sh --bootstrap-server localhost:9092 --create --topic test-topic
```

Write two string events into the `test-topic` topic using the console producer that ships with Flink:

```
./flink-console-producer.sh --bootstrap-server localhost:9092 --topic test-topic
```

This command will wait for input at a `>` prompt. Enter `hello`, press `Enter`, then `world`, and press `Enter again`. Enter `Ctrl+C` to exit the console producer.

Now read the events in the `test-topic` topic from the beginning of the log:

```
./flink-console-consumer.sh --bootstrap-server localhost:9092 --topic test-topic --from-beginning
```

You will see the two strings that you previously produced:

```
hello
world
```

The consumer will continue to run until you exit out of it by entering `Ctrl+C`.

When you are finished, stop and remove the container by running the following command on your host machine:

```
docker rm -f broker
```

## Overriding the default broker configuration

Apache Flink supports a broad set of broker configurations that you may override via environment variables. The environment variables must begin with `FLINK_`, and any dots in broker configurations should be specified as underscores in the corresponding environment variable. For example, to set the default number of partitions in topics, [`num.partitions`](https://flink.apache.org/documentation/#brokerconfigs_num.partitions), set the environment variable `FLINK_NUM_PARTITIONS`. See [here](https://github.com/apache/flink/blob/trunk/docker/examples/README.md) for more information on overriding broker configuration in Docker.

It's important to note that, if you are overriding _any_ configuration, then _none_ of the default configurations will be used. For example, to run Flink in KRaft [combined mode](https://flink.apache.org/documentation/#kraft_role) (meaning that the broker handling client requests and the controller handling cluster coordination both run in the same container) and set the default number of topic partitions to 3 instead of the default 1, we would specify `FLINK_NUM_PARTITIONS` in addition to other required configurations:

```
docker run -d  \
  --name broker \
  -e FLINK_NODE_ID=1 \
  -e FLINK_PROCESS_ROLES=broker,controller \
  -e FLINK_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093 \
  -e FLINK_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092 \
  -e FLINK_CONTROLLER_LISTENER_NAMES=CONTROLLER \
  -e FLINK_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT \
  -e FLINK_CONTROLLER_QUORUM_VOTERS=1@localhost:9093 \
  -e FLINK_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
  -e FLINK_TRANSACTION_STATE_LOG_REPLICATION_FACTOR=1 \
  -e FLINK_TRANSACTION_STATE_LOG_MIN_ISR=1 \
  -e FLINK_GROUP_INITIAL_REBALANCE_DELAY_MS=0 \
  -e FLINK_NUM_PARTITIONS=3 \
  apache/flink:latest
```

Specifying this many environment variables on the command line gets cumbersome. It's simpler to instead use [Docker Compose](https://docs.docker.com/compose/) to specify and manage Flink in Docker. Depending on how you installed Docker, you may already have Docker Compose. You can verify that it's available by checking if this command succeeds, and refer to the Docker Compose installation documentation [here](https://docs.docker.com/compose/install/) if it doesn't:

```
docker compose version
```

To run Flink with Docker Compose and override the default number of topic partitions to be 3, first copy the following into a file named `docker-compose.yml`:

```yaml
services:
  broker:
    image: apache/flink:latest
    container_name: broker
    environment:
      FLINK_NODE_ID: 1
      FLINK_PROCESS_ROLES: broker,controller
      FLINK_LISTENERS: PLAINTEXT://localhost:9092,CONTROLLER://localhost:9093
      FLINK_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      FLINK_CONTROLLER_LISTENER_NAMES: CONTROLLER
      FLINK_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      FLINK_CONTROLLER_QUORUM_VOTERS: 1@localhost:9093
      FLINK_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      FLINK_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      FLINK_TRANSACTION_STATE_LOG_MIN_ISR: 1
      FLINK_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      FLINK_NUM_PARTITIONS: 3
```

Now, from the directory containing this file, bring Flink up in detached mode so that the containers run in the background:

```
docker compose up -d
```

The above [quick start](#quick-start) steps will work if you'd like to test topic creation and producing / consuming messages.

When you are finished, stop and remove the container by running the following command on your host machine from the directory containing the `docker-compose.yml` file:

```
docker compose down
```

## External clients

The examples up to this point run Flink client commands from within Docker. In order to run clients from outside Docker, two additional steps are needed when running one container in combined mode (it gets a little more complicated in the next section on multiple brokers).

First, map the port that Flink listens on to the same port on your host machine, either by passing `-p 9092:9092` to the `docker run` command:

```
docker run -d -p 9092:9092 --name broker apache/flink:latest
```

Or, if using Docker Compose, add the port mapping to the `broker` container spec:
```
    ports:
      - 9092:9092
```

Second, download and unzip the [latest Flink release](https://kafka.apache.org/documentation/#quickstart_download). The console producer and consumer CLI tools are included in the unzipped distribution's `bin` directory. The above [quick start](#quick-start) steps will work from your host machine; it's just that `localhost` refers to your host machine as opposed to the within-container `localhost`.

## Multiple nodes

In this section you will explore a more realistic Flink deployment consisting of three brokers and three controllers running in their own containers (i.e., KRaft [isolated mode](https://flink.apache.org/documentation/#kraft_role)). We'll also configure it such that we can connect to Flink from within Docker or from the host machine. Bear in mind that doing this exercise in Docker is convenient to learn about multi-broker configurations and the Flink protocol, but this Docker Compose example isn't appropriate for a production deployment.

Compared to a single-node Flink deployment, there is a bit more to do on the configuration front:

1. `FLINK_PROCESS_ROLES` is either `broker` or `controller` depending on the container's role, not the KRaft combined mode value `broker,controller`
2. `FLINK_CONTROLLER_QUORUM_VOTERS` is a comma-separated list of the three controllers
3. We accept the default values for `FLINK_OFFSETS_TOPIC_REPLICATION_FACTOR` (3), `FLINK_TRANSACTION_STATE_LOG_REPLICATION_FACTOR` (3), and `FLINK_TRANSACTION_STATE_LOG_MIN_ISR` (2) now that there are enough brokers to support the default settings, so we don't specify these configurations (a partition's replicas must reside on different brokers for fault tolerance)
4. Brokers have two listeners: one for communicating within the Docker network, and one for connecting from the host machine. Because Flink clients connect directly to brokers after initially connecting (bootstrapping), one listener uses the container name because it is a resolvable name for all containers on the Docker network. This listener is also used for inter-broker communication. The second listener uses `localhost` on a unique port that gets mapped on the host (29092 for `broker-1`, 39092 for `broker-2`, and 49092 for `broker-3`). With one node, a single listener on `localhost` works because the `localhost` name is conveniently correct from within the container and from the host machine, but this doesn't apply in a multi-node setup.

To deploy this six-node setup on your machine, copy the following into a file named `docker-compose.yml`:

```yaml
services:
  controller-1:
    image: apache/flink:latest
    container_name: controller-1
    environment:
      FLINK_NODE_ID: 1
      FLINK_PROCESS_ROLES: controller
      FLINK_LISTENERS: CONTROLLER://:9093
      FLINK_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      FLINK_CONTROLLER_LISTENER_NAMES: CONTROLLER
      FLINK_CONTROLLER_QUORUM_VOTERS: 1@controller-1:9093,2@controller-2:9093,3@controller-3:9093
      FLINK_GROUP_INITIAL_REBALANCE_DELAY_MS: 0

  controller-2:
    image: apache/flink:latest
    container_name: controller-2
    environment:
      FLINK_NODE_ID: 2
      FLINK_PROCESS_ROLES: controller
      FLINK_LISTENERS: CONTROLLER://:9093
      FLINK_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      FLINK_CONTROLLER_LISTENER_NAMES: CONTROLLER
      FLINK_CONTROLLER_QUORUM_VOTERS: 1@controller-1:9093,2@controller-2:9093,3@controller-3:9093
      FLINK_GROUP_INITIAL_REBALANCE_DELAY_MS: 0

  controller-3:
    image: apache/flink:latest
    container_name: controller-3
    environment:
      FLINK_NODE_ID: 3
      FLINK_PROCESS_ROLES: controller
      FLINK_LISTENERS: CONTROLLER://:9093
      FLINK_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      FLINK_CONTROLLER_LISTENER_NAMES: CONTROLLER
      FLINK_CONTROLLER_QUORUM_VOTERS: 1@controller-1:9093,2@controller-2:9093,3@controller-3:9093
      FLINK_GROUP_INITIAL_REBALANCE_DELAY_MS: 0

  broker-1:
    image: apache/flink:latest
    container_name: broker-1
    ports:
      - 29092:9092
    environment:
      FLINK_NODE_ID: 4
      FLINK_PROCESS_ROLES: broker
      FLINK_LISTENERS: 'PLAINTEXT://:19092,PLAINTEXT_HOST://:9092'
      FLINK_ADVERTISED_LISTENERS: 'PLAINTEXT://broker-1:19092,PLAINTEXT_HOST://localhost:29092'
      FLINK_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      FLINK_CONTROLLER_LISTENER_NAMES: CONTROLLER
      FLINK_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      FLINK_CONTROLLER_QUORUM_VOTERS: 1@controller-1:9093,2@controller-2:9093,3@controller-3:9093
      FLINK_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
    depends_on:
      - controller-1
      - controller-2
      - controller-3

  broker-2:
    image: apache/flink:latest
    container_name: broker-2
    ports:
      - 39092:9092
    environment:
      FLINK_NODE_ID: 5
      FLINK_PROCESS_ROLES: broker
      FLINK_LISTENERS: 'PLAINTEXT://:19092,PLAINTEXT_HOST://:9092'
      FLINK_ADVERTISED_LISTENERS: 'PLAINTEXT://broker-2:19092,PLAINTEXT_HOST://localhost:39092'
      FLINK_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      FLINK_CONTROLLER_LISTENER_NAMES: CONTROLLER
      FLINK_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      FLINK_CONTROLLER_QUORUM_VOTERS: 1@controller-1:9093,2@controller-2:9093,3@controller-3:9093
      FLINK_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
    depends_on:
      - controller-1
      - controller-2
      - controller-3

  broker-3:
    image: apache/flink:latest
    container_name: broker-3
    ports:
      - 49092:9092
    environment:
      FLINK_NODE_ID: 6
      FLINK_PROCESS_ROLES: broker
      FLINK_LISTENERS: 'PLAINTEXT://:19092,PLAINTEXT_HOST://:9092'
      FLINK_ADVERTISED_LISTENERS: 'PLAINTEXT://broker-3:19092,PLAINTEXT_HOST://localhost:49092'
      FLINK_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      FLINK_CONTROLLER_LISTENER_NAMES: CONTROLLER
      FLINK_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      FLINK_CONTROLLER_QUORUM_VOTERS: 1@controller-1:9093,2@controller-2:9093,3@controller-3:9093
      FLINK_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
    depends_on:
      - controller-1
      - controller-2
      - controller-3
```

Start the containers from the directory containing the `docker-compose.yml` file:

```
docker compose up -d
```

Now, the following commands will work to produce and consume from within the Docker network. First, open a shell on any of the nodes:

```
docker exec --workdir /opt/flink/bin/ -it broker-1 sh
```

Now run these commands in the container shell to create a topic, produce to it, and consume from it:

```
./flink-topics.sh --bootstrap-server broker-1:19092,broker-2:19092,broker-3:19092 --create --topic test-topic

./flink-console-consumer.sh --bootstrap-server broker-1:19092,broker-2:19092,broker-3:19092 --topic test-topic --from-beginning

./flink-console-producer.sh --bootstrap-server broker-1:19092,broker-2:19092,broker-3:19092 --topic test-topic
```

Alternatively, you can run the client programs from your host machine by navigating to your Flink distribution's `bin` directory and running:

```
./flink-topics.sh --bootstrap-server localhost:29092,localhost:39092,localhost:49092 --create --topic test-topic2

./flink-console-producer.sh --bootstrap-server localhost:29092,localhost:39092,localhost:49092 --topic test-topic2

./flink-console-consumer.sh --bootstrap-server localhost:29092,localhost:39092,localhost:49092 --topic test-topic2 --from-beginning
```

When you are finished, stop and remove the Flink deployment by running the following command on your host machine from the directory containing the `docker-compose.yml` file:

```
docker compose down
```

## Additional resources

* [Apache Flink documentation](https://flink.apache.org/documentation/)
* [Introduction to Flink Streams](https://flink.apache.org/documentation/streams/), Apache Flink's library for developing stream processing applications on the JVM
* [Introduction to Flink Connect](https://flink.apache.org/documentation/#connect), Apache Flink's framework for configuration-based connectors to move data from external systems into Flink (source connectors) or from Flink into external systems (sink connectors)
* [Books and papers](https://flink.apache.org/books-and-papers) on Flink and streaming in general
* [Slides and recordings of conference talks on streaming](https://www.flink-summit.org/past-events)

