### Dockerizing a Spring Kafka Consumer with Maven Jib

**Maven Jib** is a plugin that builds optimized Docker and OCI images for Java applications without needing a Dockerfile. It integrates with Maven and can be used to package Spring Boot applications into Docker images efficiently.

### Steps to Dockerize a Spring Kafka Consumer with Maven Jib

1. **Create a Maven Project**
2. **Add Dependencies**
3. **Create Kafka Consumer Configuration**
4. **Create Kafka Listener**
5. **Configure Application Properties**
6. **Add Maven Jib Plugin**

### Detailed Steps

#### 1. Create a Maven Project

Generate a new Maven project using Spring Initializr or manually by creating the `pom.xml`.

#### 2. Add Dependencies

In your `pom.xml`, add the necessary dependencies for Spring Kafka and Maven Jib.

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>spring-kafka-consumer</artifactId>
    <version>1.0-SNAPSHOT</version>
    <properties>
        <java.version>17</java.version>
        <spring.kafka.version>3.0.0</spring.kafka.version>
        <jib-maven-plugin.version>3.3.1</jib-maven-plugin.version>
    </properties>
    <dependencies>
        <!-- Spring Boot Starter -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <!-- Spring Kafka -->
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>
        <!-- Kafka Clients -->
        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka-clients</artifactId>
        </dependency>
        <!-- Spring Boot Test -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <!-- Spring Kafka Test -->
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>com.google.cloud.tools</groupId>
                <artifactId>jib-maven-plugin</artifactId>
                <version>${jib-maven-plugin.version}</version>
                <configuration>
                    <to>
                        <image>docker.io/yourusername/spring-kafka-consumer</image>
                    </to>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

#### 3. Create Kafka Consumer Configuration

Create a configuration class to set up the Kafka consumer.

```java
package com.example.kafkaconsumer.config;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.listener.MessageListener;
import org.springframework.kafka.support.serializer.ErrorHandlingDeserializer;
import org.springframework.kafka.support.serializer.JsonDeserializer;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "group_id");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
        props.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, JsonDeserializer.class.getName());
        props.put(JsonDeserializer.TRUSTED_PACKAGES, "*");
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

#### 4. Create Kafka Listener

Create a Kafka listener to consume messages.

```java
package com.example.kafkaconsumer.listener;

import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;

@Service
public class KafkaConsumerListener {

    @KafkaListener(topics = "your_topic", groupId = "group_id")
    public void listen(String message) {
        System.out.println("Received Message: " + message);
    }
}
```

#### 5. Application Properties

Configure Kafka properties in `application.properties`.

```properties
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.consumer.group-id=group_id
spring.kafka.consumer.auto-offset-reset=earliest
```

#### 6. Build and Push Docker Image with Jib

To build and push the Docker image, run the following Maven command:

```bash
mvn compile jib:build
```

This will create and push the Docker image to your specified Docker repository.

### Full Maven Project Structure

```
spring-kafka-consumer/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   ├── com/
│   │   │   │   ├── example/
│   │   │   │   │   ├── kafkaconsumer/
│   │   │   │   │   │   ├── config/
│   │   │   │   │   │   │   └── KafkaConsumerConfig.java
│   │   │   │   │   │   ├── listener/
│   │   │   │   │   │   │   └── KafkaConsumerListener.java
│   │   │   │   │   │   └── SpringKafkaConsumerApplication.java
│   │   └── resources/
│   │       └── application.properties
├── pom.xml
```

This structure sets up a basic Spring Kafka consumer using Maven Jib for Dockerization.

---

### Next Steps

**a.** Add unit tests to validate the Kafka consumer functionality.

**b.** Implement error handling for the Kafka listener to ensure robustness.
