Yes, you can check for partition assignments and revocations in Spring Kafka by implementing a custom `ConsumerRebalanceListener`. This allows you to perform specific actions when partitions are assigned or revoked, such as checking an API or managing offsets.

### Implementing a Custom ConsumerRebalanceListener

1. **Create a Custom Rebalance Listener**: Implement the `ConsumerRebalanceListener` interface.
2. **Register the Listener**: Register the custom rebalance listener in the Kafka listener container.

### Step-by-Step Implementation

#### Custom Rebalance Listener

Create a class that implements the `ConsumerRebalanceListener` interface.

```java
import org.apache.kafka.clients.consumer.Consumer;
import org.apache.kafka.clients.consumer.ConsumerRebalanceListener;
import org.apache.kafka.common.TopicPartition;
import org.springframework.web.client.RestTemplate;

import java.util.Collection;

public class CustomRebalanceListener implements ConsumerRebalanceListener {

    private final RestTemplate restTemplate = new RestTemplate();

    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        System.out.println("Partitions revoked: " + partitions);
        // Perform any necessary actions when partitions are revoked
    }

    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        System.out.println("Partitions assigned: " + partitions);
        // Perform any necessary actions when partitions are assigned
        if (!checkApi()) {
            // Unassign partitions if API check fails
            Consumer<?, ?> consumer = ...; // Obtain the consumer instance
            consumer.unsubscribe();
            consumer.assign(partitions);
            consumer.pause(partitions);
            System.out.println("Unassigned and paused all partitions: " + partitions);
        }
    }

    private boolean checkApi() {
        String apiUrl = "http://your-api-endpoint";
        Boolean response = restTemplate.getForObject(apiUrl, Boolean.class);
        return response != null && response;
    }
}
```

#### Kafka Configuration Class

Register the custom rebalance listener in your Kafka configuration.

```java
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.listener.config.ContainerProperties;
import org.springframework.kafka.listener.ConsumerAwareListenerErrorHandler;

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

        // Set the custom rebalance listener
        factory.getContainerProperties().setConsumerRebalanceListener(new CustomRebalanceListener());

        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
        return factory;
    }
}
```

#### Explanation:

1. **CustomRebalanceListener**: Implements the `ConsumerRebalanceListener` interface and overrides `onPartitionsRevoked` and `onPartitionsAssigned` methods.
   - `onPartitionsRevoked`: Called when partitions are revoked from the consumer.
   - `onPartitionsAssigned`: Called when partitions are assigned to the consumer.
   - `checkApi`: Checks an external API to determine whether the partitions should remain assigned.
2. **Kafka Configuration**: In the Kafka configuration class, register the custom rebalance listener using `setConsumerRebalanceListener`.

### Using the Listener in the Service

In your Kafka consumer service, you don't need to change the `onMessage` method. The custom rebalance listener will handle partition assignment and revocation logic.

This approach ensures that you can perform specific actions whenever partitions are assigned or revoked, such as checking an API or managing consumer state.
