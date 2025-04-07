# Your First Kafka Tutorial

Let's build a simple application that uses a `KafkaConsumer` to read records from Kafka.

## Create a Consumer

There are required properties needed to create a Kafka Consumer. At a minimum, the `Consumer` needs to know:
* How to find the Kafka broker(s).
* How to deserialize the key and value of events.
* A group ID value for consumer group coordination.

```java annotate
  final String bootstrapServers = "localhost:9092";
  final String consumerGroupId = "my-group-id";
  final Properties consumerAppProps = new Properties() {{
        put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        put(ConsumerConfig.GROUP_ID_CONFIG, consumerGroupId);
      }};
```

For more about Kafka Consumer Configurations, please [refer to the documentation](https://docs.confluent.io/platform/current/clients/consumer.html#ak-consumer-configuration).

## Consume Events

To consume a stream of events, the `Consumer` needs to subscribe to a topic (or multiple topics), and poll said topic(s)
to retrieve `ConsumerRecords`. In our example, we use an implementation of a `ConsumerRecordsHandler` to process the
`ConsumerRecords` from a poll interval.

```java annotate
  public void runConsume(final List<String> topicNames, final ConsumerRecordsHandler<String, String> recordsHandler) {
  try {
    consumer.subscribe(topicNames);
    while (keepConsuming) {
      final ConsumerRecords<String, String> consumerRecords = consumer.poll(Duration.ofSeconds(1));
      recordsHandler.process(consumerRecords);
    }
  } finally {
    consumer.close();
  }
}
```
