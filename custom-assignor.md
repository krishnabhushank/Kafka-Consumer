Certainly! Adding logging can help monitor the behavior of the custom partition assignor, especially to understand when and why rebalancing is stopped. We will use the SLF4J (Simple Logging Facade for Java) library, which is commonly used for logging in Java applications.

### Complete Implementation with Logging

**CustomPartitionAssignor.java**

```java
import org.apache.kafka.clients.consumer.internals.AbstractPartitionAssignor;
import org.apache.kafka.common.TopicPartition;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.IOException;
import java.net.HttpURLConnection;
import java.net.URL;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;

public class CustomPartitionAssignor extends AbstractPartitionAssignor {

    private static final Logger logger = LoggerFactory.getLogger(CustomPartitionAssignor.class);
    private Map<String, List<TopicPartition>> currentAssignment = new HashMap<>();

    @Override
    public String name() {
        return "custom";
    }

    @Override
    public Map<String, List<TopicPartition>> assign(Map<String, Integer> partitionsPerTopic, Map<String, Subscription> subscriptions) {
        logger.info("Starting partition assignment.");
        Map<String, List<TopicPartition>> assignments = new HashMap<>();

        if (shouldStopRebalance()) {
            logger.info("Stopping rebalance as the API returned true.");
            assignments = getCurrentAssignment(subscriptions.keySet());
        } else {
            assignments = super.assign(partitionsPerTopic, subscriptions);
            updateCurrentAssignment(subscriptions.keySet(), assignments);
            logger.info("New partition assignment completed: {}", assignments);
        }

        return assignments;
    }

    private boolean shouldStopRebalance() {
        boolean result = callApiToCheckIfStopRebalance();
        logger.info("API call result for stopping rebalance: {}", result);
        return result;
    }

    private Map<String, List<TopicPartition>> getCurrentAssignment(Set<String> memberIds) {
        Map<String, List<TopicPartition>> assignments = new HashMap<>();
        for (String memberId : memberIds) {
            List<TopicPartition> assignment = currentAssignment.get(memberId);
            if (assignment == null) {
                logger.warn("No current assignment found for member {}. Returning an empty list.", memberId);
                assignments.put(memberId, List.of());
            } else {
                assignments.put(memberId, assignment);
            }
        }
        logger.info("Returning current assignment: {}", assignments);
        return assignments;
    }

    private void updateCurrentAssignment(Set<String> memberIds, Map<String, List<TopicPartition>> assignments) {
        logger.info("Updating current assignment for members: {}", memberIds);
        for (String memberId : memberIds) {
            currentAssignment.put(memberId, assignments.get(memberId));
        }
    }

    private boolean callApiToCheckIfStopRebalance() {
        try {
            URL url = new URL("http://your-api-endpoint");
            HttpURLConnection con = (HttpURLConnection) url.openConnection();
            con.setRequestMethod("GET");

            int status = con.getResponseCode();
            if (status == 200) {
                // Parse response to check if it should stop rebalancing
                // Assuming the response is a simple boolean
                logger.info("API response indicates to stop rebalancing.");
                return true; // Modify as per your response parsing logic
            } else {
                logger.warn("API call returned non-200 status code: {}", status);
            }
        } catch (IOException e) {
            logger.error("Error while calling API to check if rebalance should be stopped", e);
        }
        return false;
    }
}
```

### Configuration

**CustomKafkaConsumer.java**

```java
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.Duration;
import java.util.List;
import java.util.Properties;

public class CustomKafkaConsumer {

    private static final Logger logger = LoggerFactory.getLogger(CustomKafkaConsumer.class);

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "custom-consumer-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG, CustomPartitionAssignor.class.getName());

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(List.of("your-topic"));

        logger.info("Kafka consumer started and subscribed to topic.");

        while (true) {
            consumer.poll(Duration.ofMillis(100));
            // Process records
            logger.info("Polling records...");
        }
    }
}
```

### Notes

- **Logging Levels:**
  - `info`: For normal operation messages, e.g., when rebalancing is stopped or an assignment is updated.
  - `warn`: For non-critical issues that should be looked at, e.g., when no current assignment is found or when the API returns a non-200 status code.
  - `error`: For critical issues, e.g., when there is an exception during the API call.

- **SLF4J with Logback:** Ensure SLF4J is included in your project dependencies. You can use Logback as the underlying logging framework.

**Maven Dependency:**

```xml
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>1.7.30</version>
</dependency>
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.2.3</version>
</dependency>
```

**Suggestions for next steps:**
**a.** Add unit tests to validate logging behavior and assignment logic.
**b.** Implement error handling and retries for the API call to improve robustness.
