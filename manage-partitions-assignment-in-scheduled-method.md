Yes, you can manage the check API, unassign-pause, and resume logic outside the `onMessage` method. One way to do this is to use a scheduled task that periodically checks the API and manages the consumer's partitions accordingly. This approach decouples the message processing logic from the partition management logic, making your code cleaner and easier to maintain.

### Step-by-Step Implementation

1. **Create a Scheduled Task**: Use a scheduled task to periodically check the API and manage the consumer's partitions.
2. **Manage Partitions in the Scheduled Task**: Use the `KafkaListenerEndpointRegistry` to access the `MessageListenerContainer` and manage partitions.
3. **Message Processing**: Keep the `onMessage` method focused on processing messages.

### Kafka Consumer Service

Implement the `ConsumerAwareMessageListener` to process messages and use a scheduled task to handle the API check and partition management.

```java
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.common.TopicPartition;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.config.KafkaListenerEndpointRegistry;
import org.springframework.kafka.listener.ConsumerAwareMessageListener;
import org.springframework.kafka.listener.MessageListenerContainer;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.util.Collection;

@Service
public class KafkaConsumerService implements ConsumerAwareMessageListener<String, String> {

    @Autowired
    private KafkaListenerEndpointRegistry kafkaListenerEndpointRegistry;

    private final RestTemplate restTemplate = new RestTemplate();
    private volatile boolean shouldConsume = true;

    @Override
    @KafkaListener(id = "kafkaListenerId", topics = "your_topic", containerFactory = "kafkaListenerContainerFactory")
    public void onMessage(ConsumerRecord<String, String> record, org.apache.kafka.clients.consumer.Consumer<?, ?> consumer) {
        if (shouldConsume) {
            // Process the message
            String key = record.key();
            String message = record.value();
            System.out.println("Consumed message: Key = " + key + ", Value = " + message);
        }
    }

    @Scheduled(fixedRate = 10000) // Check the API every 10 seconds
    public void checkApiAndManagePartitions() {
        String apiUrl = "http://your-api-endpoint";
        Boolean response = restTemplate.getForObject(apiUrl, Boolean.class);
        shouldConsume = response != null && response;

        MessageListenerContainer listenerContainer = kafkaListenerEndpointRegistry.getListenerContainer("kafkaListenerId");
        if (listenerContainer != null && listenerContainer.isRunning()) {
            Collection<TopicPartition> partitions = listenerContainer.getAssignedPartitions();
            if (!shouldConsume) {
                // Pause all partitions
                listenerContainer.pause();
                System.out.println("Paused all partitions: " + partitions);
            } else {
                // Resume all paused partitions
                listenerContainer.resume();
                System.out.println("Resumed all partitions: " + partitions);
            }
        }
    }
}
```

### Kafka Configuration Class

Ensure your Kafka configuration class is set up to provide the necessary beans.

```java
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;

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
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
        return factory;
    }
}
```

### Explanation:

1. **ConsumerAwareMessageListener**: Implement the `ConsumerAwareMessageListener` interface to access the `ConsumerRecord` and `Consumer` objects.
2. **Scheduled Task**: Use the `@Scheduled` annotation to define a method that checks the API at a fixed rate (e.g., every 10 seconds).
3. **Check API and Manage Partitions**: The `checkApiAndManagePartitions` method checks the API and uses the `KafkaListenerEndpointRegistry` to get the `MessageListenerContainer`. Depending on the API response, it pauses or resumes the partitions.
4. **Message Processing**: The `onMessage` method processes messages only if `shouldConsume` is `true`.

By using a scheduled task to manage the partition assignments and API checks, you keep the message processing logic in the `onMessage` method clean and focused solely on handling messages. This decouples the partition management logic from the message processing logic, improving code maintainability and readability.
