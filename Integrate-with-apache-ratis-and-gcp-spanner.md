# Integrate Kafka Consumer with Apache Ratis and GCP Spanner 

To integrate Kafka with Apache Ratis for distributed coordination and then send the data to Google Cloud Spanner, we can follow these steps:

1. **Set Up Kafka and Ratis**: Ensure both Kafka and Ratis are running.
2. **Create a Kafka Producer to Send Sample Data**: Generate data for Kafka.
3. **Implement Kafka Consumers with Ratis Coordination**: Ensure fault-tolerant consumption of Kafka messages.
4. **Forward Data from Kafka Consumers to Google Cloud Spanner**: Store the processed data in Spanner.

Here's a detailed setup:

### Step 1: Set Up Kafka and Ratis

1. **Kafka Setup**: Use Docker Compose to set up Kafka.

**docker-compose.yml for Kafka**:

```yaml
version: '3.8'
services:
  zookeeper:
    image: wurstmeister/zookeeper:3.4.6
    ports:
      - "2181:2181"

  kafka:
    image: wurstmeister/kafka:2.13-2.7.0
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    depends_on:
      - zookeeper
```

2. **Ratis Setup**: Use a Dockerfile and configuration files to set up a Ratis cluster.

**Dockerfile for Ratis**:

```Dockerfile
FROM openjdk:11
ADD ratis-server.jar /ratis/ratis-server.jar
ADD conf /ratis/conf
WORKDIR /ratis
CMD ["java", "-jar", "ratis-server.jar"]
```

**ratis-server.conf**:

```properties
# Ratis Server Configuration
ratis.serverId = server1
ratis.storageDir = /ratis/data
```

### Step 2: Create a Kafka Producer to Send Sample Data

**KafkaProducerExample.java**:

```java
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.common.serialization.StringSerializer;

import java.util.Properties;

public class KafkaProducerExample {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

        KafkaProducer<String, String> producer = new KafkaProducer<>(props);

        for (int i = 0; i < 10; i++) {
            ProducerRecord<String, String> record = new ProducerRecord<>("events", "key-" + i, "value-" + i);
            producer.send(record);
        }

        producer.close();
        System.out.println("Messages sent successfully");
    }
}
```

### Step 3: Implement Kafka Consumers with Ratis Coordination

**KafkaConsumerWithRatis.java**:

```java
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.apache.ratis.client.RaftClient;
import org.apache.ratis.conf.RaftProperties;
import org.apache.ratis.grpc.GrpcClientRpc;
import org.apache.ratis.protocol.Message;
import org.apache.ratis.protocol.RaftGroup;
import org.apache.ratis.protocol.RaftGroupId;
import org.apache.ratis.protocol.RaftPeer;
import org.apache.ratis.retry.ExponentialBackoffRetry;
import org.apache.ratis.retry.RetryPolicies;

import java.nio.charset.StandardCharsets;
import java.time.Duration;
import java.util.Collections;
import java.util.Properties;
import java.util.UUID;

public class KafkaConsumerWithRatis {
    private static final String KAFKA_TOPIC = "events";
    private static final String RAFT_GROUP_ID = UUID.randomUUID().toString();
    private static final String RAFT_SERVER_ADDRESS = "127.0.0.1:8080";

    private final KafkaConsumer<String, String> consumer;
    private final RaftClient raftClient;

    public KafkaConsumerWithRatis(String kafkaBootstrapServers, RaftGroup raftGroup) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, kafkaBootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "kafka-consumer-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        this.consumer = new KafkaConsumer<>(props);
        RaftProperties raftProperties = new RaftProperties();
        raftClient = RaftClient.newBuilder()
                .setProperties(raftProperties)
                .setRaftGroup(raftGroup)
                .setClientRpc(new GrpcClientRpc(raftProperties))
                .setRetryPolicy(RetryPolicies.exponentialBackoffRetry(100, 1000, 100, ExponentialBackoffRetry.Uniform.random()))
                .build();
    }

    public void start() {
        consumer.subscribe(Collections.singletonList(KAFKA_TOPIC));

        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
            for (ConsumerRecord<String, String> record : records) {
                System.out.printf("Consumed message: %s%n", record.value());

                // Send data to Ratis
                Message message = Message.valueOf(record.value());
                raftClient.send(message);
            }
        }
    }

    public static void main(String[] args) {
        RaftPeer peer = RaftPeer.newBuilder().setId("peer1").setAddress(RAFT_SERVER_ADDRESS).build();
        RaftGroup raftGroup = RaftGroup.valueOf(RaftGroupId.valueOf(UUID.fromString(RAFT_GROUP_ID)), peer);

        KafkaConsumerWithRatis consumerApp = new KafkaConsumerWithRatis("localhost:9092", raftGroup);
        consumerApp.start();
    }
}
```

### Step 4: Forward Data from Kafka Consumers to Google Cloud Spanner

1. **Set Up Google Cloud Spanner**: Create a Spanner instance and database.
2. **Integrate Spanner with the Consumer**: Use the Google Cloud Spanner client library to store data.

**SpannerClient.java**:

```java
import com.google.cloud.spanner.DatabaseClient;
import com.google.cloud.spanner.Spanner;
import com.google.cloud.spanner.SpannerOptions;

public class SpannerClient {
    private final DatabaseClient dbClient;

    public SpannerClient(String projectId, String instanceId, String databaseId) {
        SpannerOptions options = SpannerOptions.newBuilder().setProjectId(projectId).build();
        Spanner spanner = options.getService();
        dbClient = spanner.getDatabaseClient(DatabaseId.of(projectId, instanceId, databaseId));
    }

    public void insertData(String key, String value) {
        dbClient.write(Collections.singletonList(Mutation.newInsertOrUpdateBuilder("MyTable")
                .set("Key").to(key)
                .set("Value").to(value)
                .build()));
    }
}
```

### `KafkaConsumerWithRatisAndSpanner.java`

We'll ensure the class can handle sending data to both Ratis and Google Cloud Spanner. Here's the complete implementation:

```java
import com.google.cloud.spanner.DatabaseId;
import com.google.cloud.spanner.Mutation;
import com.google.cloud.spanner.Spanner;
import com.google.cloud.spanner.SpannerOptions;
import com.google.cloud.spanner.DatabaseClient;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.apache.ratis.client.RaftClient;
import org.apache.ratis.conf.RaftProperties;
import org.apache.ratis.grpc.GrpcClientRpc;
import org.apache.ratis.protocol.Message;
import org.apache.ratis.protocol.RaftGroup;
import org.apache.ratis.protocol.RaftGroupId;
import org.apache.ratis.protocol.RaftPeer;
import org.apache.ratis.retry.ExponentialBackoffRetry;
import org.apache.ratis.retry.RetryPolicies;

import java.nio.charset.StandardCharsets;
import java.time.Duration;
import java.util.Collections;
import java.util.Properties;
import java.util.UUID;

public class KafkaConsumerWithRatisAndSpanner {
    private static final String KAFKA_TOPIC = "events";
    private static final String RAFT_GROUP_ID = UUID.randomUUID().toString();
    private static final String RAFT_SERVER_ADDRESS = "127.0.0.1:8080";

    private final KafkaConsumer<String, String> consumer;
    private final RaftClient raftClient;
    private final SpannerClient spannerClient;

    public KafkaConsumerWithRatisAndSpanner(String kafkaBootstrapServers, RaftGroup raftGroup, SpannerClient spannerClient) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, kafkaBootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "kafka-consumer-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        this.consumer = new KafkaConsumer<>(props);
        RaftProperties raftProperties = new RaftProperties();
        this.raftClient = RaftClient.newBuilder()
                .setProperties(raftProperties)
                .setRaftGroup(raftGroup)
                .setClientRpc(new GrpcClientRpc(raftProperties))
                .setRetryPolicy(RetryPolicies.exponentialBackoffRetry(100, 1000, 100, ExponentialBackoffRetry.Uniform.random()))
                .build();
        this.spannerClient = spannerClient;
    }

    public void start() {
        consumer.subscribe(Collections.singletonList(KAFKA_TOPIC));

        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
            for (ConsumerRecord<String, String> record : records) {
                System.out.printf("Consumed message: %s%n", record.value());

                try {
                    // Send data to Ratis
                    Message message = Message.valueOf(record.value());
                    raftClient.send(message);

                    // Insert data into Spanner
                    spannerClient.insertData(record.key(), record.value());
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public static void main(String[] args) {
        RaftPeer peer = RaftPeer.newBuilder().setId("peer1").setAddress(RAFT_SERVER_ADDRESS).build();
        RaftGroup raftGroup = RaftGroup.valueOf(RaftGroupId.valueOf(UUID.fromString(RAFT_GROUP_ID)), peer);

        // Replace with your GCP project, instance, and database IDs
        String projectId = "your-gcp-project-id";
        String instanceId = "your-spanner-instance-id";
        String databaseId = "your-database-id";

        SpannerClient spannerClient = new SpannerClient(projectId, instanceId, databaseId);

        KafkaConsumerWithRatisAndSpanner consumerApp = new KafkaConsumerWithRatisAndSpanner("localhost:9092", raftGroup, spannerClient);
        consumerApp.start();
    }
}
```

### Explanation

1. **Kafka Consumer Setup**:
    - Initializes a Kafka consumer with configurations.
    - Subscribes to the Kafka topic `events`.

2. **Ratis Client Setup**:
    - Configures the Ratis client with Raft properties, retry policies, and Raft group information.
    - Uses Ratis to manage leader election and coordination.

3. **Google Cloud Spanner Client**:
    - The `SpannerClient` class handles interactions with Google Cloud Spanner.
    - Inserts data into the Spanner database.

4. **Main Logic**:
    - The `start` method continuously polls Kafka for new messages.
    - For each message, it sends the data to the Ratis cluster.
    - Then, it inserts the data into the Google Cloud Spanner database.

### `SpannerClient.java`

This class manages the connection and operations with Google Cloud Spanner:

```java
import com.google.cloud.spanner.DatabaseId;
import com.google.cloud.spanner.Mutation;
import com.google.cloud.spanner.Spanner;
import com.google.cloud.spanner.SpannerOptions;
import com.google.cloud.spanner.DatabaseClient;
import java.util.Collections;

public class SpannerClient {
    private final DatabaseClient dbClient;

    public SpannerClient(String projectId, String instanceId, String databaseId) {
        SpannerOptions options = SpannerOptions.newBuilder().setProjectId(projectId).build();
        Spanner spanner = options.getService();
        dbClient = spanner.getDatabaseClient(DatabaseId.of(projectId, instanceId, databaseId));
    }

    public void insertData(String key, String value) {
        dbClient.write(Collections.singletonList(Mutation.newInsertOrUpdateBuilder("MyTable")
                .set("Key").to(key)
                .set("Value").to(value)
                .build()));
    }
}
```

### Running the Setup

1. **Build and Run the Kafka Producer**:
    - Compile and run the `KafkaProducerExample` to send messages to Kafka.

2. **Build and Run the Kafka Consumer with Ratis and Spanner**:
    - Compile and run the `KafkaConsumerWithRatisAndSpanner` to consume messages from Kafka, send them to Ratis, and store them in Spanner.

3. **Start Kafka and Ratis Clusters**:
    - Ensure Kafka and Ratis clusters are up and running.

4. **Create the Required Spanner Table**:
    - Ensure that the `MyTable` in your Spanner database has the required schema (`Key` and `Value` columns).

This setup demonstrates the integration of Kafka, Ratis, and Google Cloud Spanner to ensure fault-tolerant message processing and distributed state management.
