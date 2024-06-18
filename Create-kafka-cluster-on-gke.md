Deploying a small Kafka cluster on Google Kubernetes Engine (GKE) involves several steps, including setting up Kubernetes resources and configuring Kafka. Below are the essential YAML files and steps to get a basic Kafka cluster up and running on GKE.

### 1. Setup GKE Cluster

First, ensure you have a GKE cluster. You can create one using the following command:

```sh
gcloud container clusters create kafka-cluster --num-nodes=3
```

### 2. Kafka Namespace

Create a namespace for Kafka:

```yaml
# kafka-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kafka
```

Apply it:

```sh
kubectl apply -f kafka-namespace.yaml
```

### 3. Zookeeper Deployment and Service

Kafka depends on Zookeeper. Deploy Zookeeper first:

```yaml
# zookeeper.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: zookeeper
  namespace: kafka
spec:
  serviceName: "zookeeper"
  replicas: 3
  selector:
    matchLabels:
      app: zookeeper
  template:
    metadata:
      labels:
        app: zookeeper
    spec:
      containers:
      - name: zookeeper
        image: confluentinc/cp-zookeeper:latest
        ports:
        - containerPort: 2181
        env:
        - name: ZOOKEEPER_SERVER_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: ZOOKEEPER_CLIENT_PORT
          value: "2181"
        - name: ZOOKEEPER_TICK_TIME
          value: "2000"
        readinessProbe:
          exec:
            command:
            - sh
            - -c
            - "echo ruok | nc 127.0.0.1 2181 | grep imok"
          initialDelaySeconds: 10
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: zookeeper
  namespace: kafka
spec:
  ports:
  - port: 2181
  selector:
    app: zookeeper
```

Apply it:

```sh
kubectl apply -f zookeeper.yaml
```

### 4. Kafka Deployment and Service

Deploy Kafka:

```yaml
# kafka.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
  namespace: kafka
spec:
  serviceName: "kafka"
  replicas: 3
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      containers:
      - name: kafka
        image: confluentinc/cp-kafka:latest
        ports:
        - containerPort: 9092
        env:
        - name: KAFKA_BROKER_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: KAFKA_ZOOKEEPER_CONNECT
          value: zookeeper:2181
        - name: KAFKA_ADVERTISED_LISTENERS
          value: PLAINTEXT://kafka-0.kafka.kafka.svc.cluster.local:9092
        - name: KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR
          value: "3"
---
apiVersion: v1
kind: Service
metadata:
  name: kafka
  namespace: kafka
spec:
  ports:
  - port: 9092
  selector:
    app: kafka
```

Apply it:

```sh
kubectl apply -f kafka.yaml
```

### 5. Persistent Volumes and Persistent Volume Claims

Ensure persistent storage for Kafka and Zookeeper:

```yaml
# persistent-volumes.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: kafka-pv
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/kafka"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kafka-pvc
  namespace: kafka
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: zookeeper-pv
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/zookeeper"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: zookeeper-pvc
  namespace: kafka
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

Apply it:

```sh
kubectl apply -f persistent-volumes.yaml
```

### 6. Update StatefulSets to use PVCs

Update Kafka and Zookeeper StatefulSets to use the PVCs:

```yaml
# Update kafka.yaml and zookeeper.yaml StatefulSets

# Add under spec.template.spec.containers:
volumeMounts:
- mountPath: /var/lib/kafka/data
  name: kafka-storage

# Add under spec:
volumeClaimTemplates:
- metadata:
    name: kafka-storage
  spec:
    accessModes: [ "ReadWriteOnce" ]
    storageClassName: "manual"
    resources:
      requests:
        storage: 10Gi
```

Do similar changes for Zookeeper:

```yaml
# Update zookeeper.yaml StatefulSet

# Add under spec.template.spec.containers:
volumeMounts:
- mountPath: /var/lib/zookeeper/data
  name: zookeeper-storage

# Add under spec:
volumeClaimTemplates:
- metadata:
    name: zookeeper-storage
  spec:
    accessModes: [ "ReadWriteOnce" ]
    storageClassName: "manual"
    resources:
      requests:
        storage: 10Gi
```

Apply the updated StatefulSets:

```sh
kubectl apply -f kafka.yaml
kubectl apply -f zookeeper.yaml
```

### Verification

Verify that the pods are running:

```sh
kubectl get pods -n kafka
```

You should see both Kafka and Zookeeper pods running.

### Accessing Kafka

To access Kafka from outside the cluster, you can expose it using a LoadBalancer service or NodePort.

```yaml
# kafka-external-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka-external
  namespace: kafka
spec:
  type: LoadBalancer
  ports:
  - port: 9094
    targetPort: 9092
  selector:
    app: kafka
```

Apply it:

```sh
kubectl apply -f kafka-external-service.yaml
```

### Conclusion

This setup provides a basic Kafka cluster on GKE. You might need to customize configurations based on your specific requirements and production needs, such as security settings, resource limits, and further scaling.
