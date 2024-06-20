To deploy the `provectuslabs/kafka-ui` on Kubernetes to connect to a Kafka cluster and monitor topics, you need to create a Kubernetes Deployment and a Service. Below are the necessary YAML configurations to deploy `kafka-ui`.

### Deployment YAML

The Deployment will specify the `kafka-ui` container and its configuration.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-ui
  namespace: kafka
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kafka-ui
  template:
    metadata:
      labels:
        app: kafka-ui
    spec:
      containers:
      - name: kafka-ui
        image: provectuslabs/kafka-ui:latest
        ports:
        - containerPort: 8080
        env:
        - name: KAFKA_CLUSTERS_0_NAME
          value: "local"
        - name: KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS
          value: "<your-kafka-bootstrap-servers>"
        - name: KAFKA_CLUSTERS_0_ZOOKEEPER
          value: "<your-zookeeper-connect-string>"
        - name: KAFKA_CLUSTERS_0_READONLY
          value: "false"
```

Replace `<your-kafka-bootstrap-servers>` and `<your-zookeeper-connect-string>` with the appropriate values for your Kafka cluster.

### Service YAML

The Service will expose the `kafka-ui` Deployment so that it can be accessed.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka-ui
  namespace: kafka
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: kafka-ui
```

### Complete Example

Here is the complete YAML for both the Deployment and the Service:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-ui
  namespace: kafka
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kafka-ui
  template:
    metadata:
      labels:
        app: kafka-ui
    spec:
      containers:
      - name: kafka-ui
        image: provectuslabs/kafka-ui:latest
        ports:
        - containerPort: 8080
        env:
        - name: KAFKA_CLUSTERS_0_NAME
          value: "local"
        - name: KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS
          value: "<your-kafka-bootstrap-servers>"
        - name: KAFKA_CLUSTERS_0_ZOOKEEPER
          value: "<your-zookeeper-connect-string>"
        - name: KAFKA_CLUSTERS_0_READONLY
          value: "false"
---
apiVersion: v1
kind: Service
metadata:
  name: kafka-ui
  namespace: kafka
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: kafka-ui
```

### Applying the YAML

Save the above YAML content into a file, e.g., `kafka-ui-deployment.yaml`, and apply it to your Kubernetes cluster:

```sh
kubectl apply -f kafka-ui-deployment.yaml
```

### Accessing Kafka UI

After deploying, the `kafka-ui` service will be exposed via a LoadBalancer. You can get the external IP of the service by running:

```sh
kubectl get service kafka-ui -n kafka
```

Access the Kafka UI by navigating to the external IP address in your web browser.

### Note

Ensure that your Kafka cluster is accessible from the namespace where you deploy the `kafka-ui`. You may need to adjust network policies or security groups depending on your cluster setup.
