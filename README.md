# Kafka-Consumer

Let's create a Kafka Consumer in Java that reads messages from the Kafka topic and saves them to a database using Spring Boot and Spring Data JPA.

### Step-by-Step Plan

1. **Setup Spring Boot Project**:
   - Include necessary dependencies for Spring Boot, Spring Kafka, and Spring Data JPA.

2. **Create Kafka Consumer Configuration**:
   - Configure the Kafka consumer settings.

3. **Create an Entity for the Financial Data**:
   - Define a JPA entity to represent the financial data.

4. **Create a Repository for the Financial Data**:
   - Define a Spring Data JPA repository to save the financial data.

5. **Create a Kafka Listener**:
   - Create a Kafka listener to consume messages and save them to the database.

### Dependencies

Add the following dependencies to your `pom.xml`:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

### Kafka Consumer Configuration

Create a configuration class for Kafka consumer settings:

```java
// Path: src/main/java/com/example/kafkaconsumer/config/KafkaConsumerConfig.java

package com.example.kafkaconsumer.config;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafka;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.listener.ConcurrentMessageListenerContainer;
import org.springframework.kafka.support.serializer.ErrorHandlingDeserializer;
import org.springframework.kafka.support.serializer.JsonDeserializer;

import java.util.HashMap;
import java.util.Map;

@EnableKafka
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "group_id");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        return new DefaultKafkaConsumerFactory<>(props);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        return factory;
    }
}
```

### Financial Data Entity

Create an entity to represent the financial data:

```java
// Path: src/main/java/com/example/kafkaconsumer/model/FinancialData.java

package com.example.kafkaconsumer.model;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

@Entity
public class FinancialData {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String productType;
    private String data;

    // Constructors, getters, setters

    public FinancialData() {
    }

    public FinancialData(String productType, String data) {
        this.productType = productType;
        this.data = data;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getProductType() {
        return productType;
    }

    public void setProductType(String productType) {
        this.productType = productType;
    }

    public String getData() {
        return data;
    }

    public void setData(String data) {
        this.data = data;
    }
}
```

### Financial Data Repository

Create a repository for the financial data:

```java
// Path: src/main/java/com/example/kafkaconsumer/repository/FinancialDataRepository.java

package com.example.kafkaconsumer.repository;

import com.example.kafkaconsumer.model.FinancialData;
import org.springframework.data.jpa.repository.JpaRepository;

public interface FinancialDataRepository extends JpaRepository<FinancialData, Long> {
}
```

### Kafka Listener

Create a Kafka listener to consume messages and save them to the database:

```java
// Path: src/main/java/com/example/kafkaconsumer/service/FinancialDataListener.java

package com.example.kafkaconsumer.service;

import com.example.kafkaconsumer.model.FinancialData;
import com.example.kafkaconsumer.repository.FinancialDataRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;

@Service
public class FinancialDataListener {

    private final FinancialDataRepository repository;

    @Autowired
    public FinancialDataListener(FinancialDataRepository repository) {
        this.repository = repository;
    }

    @KafkaListener(topics = "financial_data", groupId = "group_id")
    public void listen(String message, String key) {
        FinancialData data = new FinancialData(key, message);
        repository.save(data);
    }
}
```

### Main Application

Set up the main Spring Boot application class:

```java
// Path: src/main/java/com/example/kafkaconsumer/KafkaConsumerApplication.java

package com.example.kafkaconsumer;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class KafkaConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(KafkaConsumerApplication.class, args);
    }
}
```

### Application Properties

Configure the Kafka broker and database settings in `src/main/resources/application.properties`:

```properties
spring.kafka.bootstrap-servers=localhost:9092
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=password
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
```

### Running the Application

1. **Start Kafka Broker**: Make sure your Kafka broker is running on `localhost:9092`.
2. **Run the Spring Boot Application**: Use your IDE or `mvn spring-boot:run`.

### Next Steps

**a.** Verify that the consumer is saving messages to the database by querying the H2 database.

**b.** Add error handling and logging for robustness.

Would you like assistance with any of these next steps or anything else?
