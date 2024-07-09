To unassign all partitions from a Kafka consumer instance after checking an API, you can use the `Consumer` object to handle the partition assignments directly within your listener method. This can be achieved by implementing the `ConsumerAwareListener` interface, which allows you to access the `Consumer` object and control its behavior.

Here’s how you can do it:

### Kafka Consumer Service

Implement the `ConsumerAwareListener` interface and use the `Consumer` object to unassign all partitions if the API check fails.

```java
import org.apache.kafka.clients.consumer.Consumer;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.common.TopicPartition;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.listener.ConsumerAwareMessageListener;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.util.Collection;

@Service
public class KafkaConsumerService implements ConsumerAwareMessageListener<String, String> {

    private final RestTemplate restTemplate = new RestTemplate();

    @Override
    @KafkaListener(topics = "your_topic", containerFactory = "kafkaListenerContainerFactory")
    public void onMessage(ConsumerRecord<String, String> record, Consumer<?, ?> consumer) {
        boolean shouldConsume = checkApi();
        if (!shouldConsume) {
            // Unassign all partitions
            Collection<TopicPartition> partitions = consumer.assignment();
            consumer.unsubscribe();
            consumer.assign(partitions);
            consumer.pause(partitions);
            System.out.println("Unassigned and paused all partitions: " + partitions);
            return;
        }

        // Access the key and the message
        String key = record.key();
        String message = record.value();
        System.out.println("Consumed message: Key = " + key + ", Value = " + message);

        // Resume all paused partitions if any
        if (!consumer.paused().isEmpty()) {
            consumer.resume(consumer.paused());
            System.out.println("Resumed all paused partitions: " + consumer.paused());
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

Make sure your Kafka configuration class is properly set up to provide the necessary beans. Here is a basic example:

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

1. **ConsumerAwareMessageListener**: Implementing `ConsumerAwareMessageListener` allows access to the `Consumer` object within the `onMessage` method.
2. **Check API**: The `checkApi()` method checks the API response.
3. **Unassign Partitions**: If the API response indicates that the consumer should not process messages, `consumer.unsubscribe()` is called to unassign all partitions, followed by `consumer.assign(partitions)` to manually reassign the partitions and `consumer.pause(partitions)` to pause them.
4. **Process Messages**: If the API response indicates that the consumer should process messages, it processes the message and resumes paused partitions if any.

By following these steps, you can unassign and pause all partitions for a Kafka consumer instance if the API check fails, ensuring that the consumer stops processing messages until the API indicates otherwise.
