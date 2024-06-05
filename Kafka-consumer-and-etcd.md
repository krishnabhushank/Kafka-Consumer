# Kafka and usig etcd for leader election and state management

To demonstrate a working example of Kafka consumers with etcd for leader election and state management, we'll create a complete setup. This will include Docker Compose files for Kafka and etcd clusters, as well as a sample Kafka producer and consumer application that uses etcd for coordination.

### Setup Overview

1. **Docker Compose Setup for Kafka and etcd Clusters**
2. **Kafka Producer to Send Sample Data**
3. **Kafka Consumers with etcd for Leader Election and State Management**
4. **Running the Setup and Observing the Behavior**

### Step 1: Docker Compose Setup for Kafka and etcd Clusters

Create a `docker-compose.yml` file to set up Kafka, Zookeeper, and etcd clusters.

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

  etcd-1:
    image: quay.io/coreos/etcd:v3.5.0
    container_name: etcd-1
    command: etcd --name etcd-1 --initial-advertise-peer-urls http://etcd-1:2380 --listen-peer-urls http://0.0.0.0:2380 --advertise-client-urls http://etcd-1:2379 --listen-client-urls http://0.0.0.0:2379 --initial-cluster etcd-1=http://etcd-1:2380,etcd-2=http://etcd-2:2380,etcd-3=http://etcd-3:2380
    ports:
      - "2379:2379"
      - "2380:2380"

  etcd-2:
    image: quay.io/coreos/etcd:v3.5.0
    container_name: etcd-2
    command: etcd --name etcd-2 --initial-advertise-peer-urls http://etcd-2:2380 --listen-peer-urls http://0.0.0.0:2380 --advertise-client-urls http://etcd-2:2379 --listen-client-urls http://0.0.0.0:2379 --initial-cluster etcd-1=http://etcd-1:2380,etcd-2=http://etcd-2:2380,etcd-3=http://etcd-3:2380
    ports:
      - "22379:2379"
      - "22380:2380"

  etcd-3:
    image: quay.io/coreos/etcd:v3.5.0
    container_name: etcd-3
    command: etcd --name etcd-3 --initial-advertise-peer-urls http://etcd-3:2380 --listen-peer-urls http://0.0.0.0:2380 --advertise-client-urls http://etcd-3:2379 --listen-client-urls http://0.0.0.0:2379 --initial-cluster etcd-1=http://etcd-1:2380,etcd-2=http://etcd-2:2380,etcd-3=http://etcd-3:2380
    ports:
      - "32379:2379"
      - "32380:2380"
```

### Step 2: Kafka Producer to Send Sample Data

Create a simple Kafka producer to send messages to the `events` topic.

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

### Step 3: Kafka Consumers with etcd for Leader Election and State Management

Create a Kafka consumer application that uses etcd for leader election.

**KafkaConsumerWithEtcd.java**:

```java
import io.etcd.jetcd.ByteSequence;
import io.etcd.jetcd.Client;
import io.etcd.jetcd.kv.PutResponse;
import io.etcd.jetcd.lock.LockResponse;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.nio.charset.StandardCharsets;
import java.time.Duration;
import java.util.Collections;
import java.util.Properties;
import java.util.concurrent.CompletableFuture;

public class KafkaConsumerWithEtcd {

    private static final String KAFKA_TOPIC = "events";
    private static final String ETCD_LOCK_KEY = "kafka-consumer-lock";

    private final KafkaConsumer<String, String> consumer;
    private final Client etcdClient;

    public KafkaConsumerWithEtcd(String kafkaBootstrapServers, String etcdEndpoint) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, kafkaBootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "kafka-consumer-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        this.consumer = new KafkaConsumer<>(props);
        this.etcdClient = Client.builder().endpoints(etcdEndpoint).build();
    }

    public void start() throws Exception {
        consumer.subscribe(Collections.singletonList(KAFKA_TOPIC));
        
        while (true) {
            try {
                // Attempt to acquire lock
                String lockKey = lock(ETCD_LOCK_KEY);
                
                // Consume messages
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
                for (ConsumerRecord<String, String> record : records) {
                    System.out.printf("Consumed message: %s%n", record.value());
                    
                    // Simulate state change and update in etcd
                    updateState(record.value());
                }
                
                // Release lock
                unlock(lockKey);
                
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    private String lock(String lockKey) throws Exception {
        ByteSequence lockKeySeq = ByteSequence.from(lockKey, StandardCharsets.UTF_8);
        CompletableFuture<LockResponse> lockFuture = etcdClient.getLockClient().lock(lockKeySeq, 0);
        LockResponse lockResponse = lockFuture.get();
        String lockKeyStr = lockResponse.getKey().toString(StandardCharsets.UTF_8);
        System.out.println("Acquired lock: " + lockKeyStr);
        return lockKeyStr;
    }

    private void unlock(String lockKey) throws Exception {
        ByteSequence lockKeySeq = ByteSequence.from(lockKey, StandardCharsets.UTF_8);
        etcdClient.getLockClient().unlock(lockKeySeq).get();
        System.out.println("Released lock: " + lockKey);
    }

    private void updateState(String newState) throws Exception {
        ByteSequence key = ByteSequence.from("state", StandardCharsets.UTF_8);
        ByteSequence value = ByteSequence.from(newState, StandardCharsets.UTF_8);
        CompletableFuture<PutResponse> future = etcdClient.getKVClient().put(key, value);
        PutResponse response = future.get();
        System.out.printf("Updated state to: %s%n", newState);
    }

    public static void main(String[] args) throws Exception {
        String kafkaBootstrapServers = "localhost:9092"; // Kafka bootstrap servers
        String etcdEndpoint = "http://localhost:2379"; // etcd endpoint
        
        KafkaConsumerWithEtcd consumerApp = new KafkaConsumerWithEtcd(kafkaBootstrapServers, etcdEndpoint);
        consumerApp.start();
    }
}
```

### Step 4: Running the Setup and Observing the Behavior

1. **Build the Kafka producer and consumer applications:**

```sh
mvn clean package
```

2. **Start the Kafka and etcd clusters:**

```sh
docker-compose up -d
```

3. **Create the `events` topic in Kafka:**

```sh
docker-compose exec kafka kafka-topics.sh --create --topic events --bootstrap-server kafka:9092 --replication-factor 1 --partitions 1
```

4. **Run the Kafka producer to send sample data:**

```sh
java -cp target/your-jar-file.jar KafkaProducerExample
```

5. **

Run multiple instances of the Kafka consumer:**

```sh
java -cp target/your-jar-file.jar KafkaConsumerWithEtcd
```

You can run multiple instances of the consumer to see how they coordinate via etcd to ensure only one instance processes the messages at a time.

### Observing the Behavior

- **Leader Election:** Only one consumer should acquire the lock at any given time.
- **Message Processing:** The consumer that has the lock processes the messages from the `events` topic.
- **State Management:** The processed state is updated in etcd by the leader consumer.

This setup demonstrates how Kafka consumers can use etcd for leader election and state management, ensuring fault tolerance and avoiding duplicate message processing.
