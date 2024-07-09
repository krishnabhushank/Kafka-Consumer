To achieve the desired behavior—where partitions are not revoked if the API check returns true, and partitions are not assigned if the API check returns false—you can modify the custom rebalance listener to handle these conditions.

### Updated Custom Rebalance Listener

Here's how you can modify the `CustomRebalanceListener` to include these checks:

```java
import org.apache.kafka.clients.consumer.Consumer;
import org.apache.kafka.clients.consumer.ConsumerRebalanceListener;
import org.apache.kafka.common.TopicPartition;
import org.springframework.web.client.RestTemplate;

import java.util.Collection;

public class CustomRebalanceListener implements ConsumerRebalanceListener {

    private final RestTemplate restTemplate = new RestTemplate();
    private final Consumer<?, ?> consumer;

    public CustomRebalanceListener(Consumer<?, ?> consumer) {
        this.consumer = consumer;
    }

    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        System.out.println("Partitions revoked: " + partitions);
        // Perform any necessary actions when partitions are revoked
        if (checkApi()) {
            System.out.println("API check returned true, preventing revocation.");
            consumer.assign(partitions);
        } else {
            System.out.println("API check returned false, allowing revocation.");
            // Proceed with the revocation
        }
    }

    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        System.out.println("Partitions assigned: " + partitions);
        // Perform any necessary actions when partitions are assigned
        if (!checkApi()) {
            System.out.println("API check returned false, preventing assignment.");
            consumer.pause(partitions);
            consumer.unsubscribe();
        } else {
            System.out.println("API check returned true, allowing assignment.");
        }
    }

    private boolean checkApi() {
        String apiUrl = "http://your-api-endpoint";
        Boolean response = restTemplate.getForObject(apiUrl, Boolean.class);
        return response != null && response;
    }
}
```

### Kafka Configuration Class

Modify the Kafka configuration class to pass the consumer instance to the custom rebalance listener.

```java
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.listener.config.ContainerProperties;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class KafkaConsumerConfig {

    @Value("${kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Value("${kafka.group-id}")
    private String groupId;

    @Bean
    public Map<String, Object> consumerConfigs() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        return props;
    }

    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        return new DefaultKafkaConsumerFactory<>(consumerConfigs());
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.getContainerProperties().setConsumerRebalanceListener(new CustomRebalanceListener(factory.getContainerProperties().getConsumer()));

        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
        return factory;
    }
}
```

### Explanation:

1. **CustomRebalanceListener**: 
   - In the `onPartitionsRevoked` method, if the API check returns true, reassign the partitions to prevent revocation.
   - In the `onPartitionsAssigned` method, if the API check returns false, pause the consumer and unsubscribe from the partitions to prevent assignment.

2. **Kafka Configuration**:
   - Ensure the `CustomRebalanceListener` has access to the consumer instance. This example assumes you can get the consumer from the container properties. If not, you might need to adapt the approach to pass the consumer instance correctly.

By implementing these modifications, you can control the partition assignment and revocation based on the API check results.
