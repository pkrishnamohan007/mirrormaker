
## **Mirror Maker:** 

Mirror Maker is the cross-cluster data mirroring mechanism provided by Apache Kafka, facilitating the replication of data between two Kafka clusters. This mirroring process can be deployed as a connector within the Kafka Connect framework, enhancing scalability, monitoring, and availability of the mirroring application. Replicating data between two clusters serves various purposes, including improving data availability, migrating to a new cluster, aggregating data from edge clusters into a central cluster, and copying data between regions. In our case, we utilize Mirror Maker to replicate data from a topic in a different cluster and account to our own MSK topic.


## **Prerequisites**

Before deploying Kafka MirrorMaker on Amazon EKS (Elastic Kubernetes Service), it's essential to ensure that you have all the necessary configurations and prerequisites in place. Here are the steps you should follow:
##### 1. Set Up Amazon EKS Cluster
- Create an Amazon EKS cluster if you haven't already done so.
- Make sure the cluster has the appropriate permissions to access other AWS resources like Amazon MSK (Managed Streaming for Kafka) clusters.
##### 2. Configure Network Peering for Amazon MSK
- If your Amazon MSK clusters are in different VPCs or AWS accounts, establish VPC peering connections between them.
- Update the route tables and modify security groups to allow traffic between the peered VPCs.


## **Use cases**

Kafka mirroring, often implemented using Kafka's MirrorMaker tool, is a powerful feature for replicating data between Kafka clusters. Here are some common use cases:

### 1. **Disaster Recovery (DR)**
Mirroring allows organizations to create a replica of their Kafka cluster in a different geographical location or data center. This ensures that in case of a failure in the primary cluster, data can still be accessed from the secondary cluster, minimizing downtime and data loss.

### 2. **Multi-Datacenter Deployments**
For organizations that operate in multiple data centers, Kafka mirroring enables seamless data replication across these locations. This ensures data consistency and availability, allowing applications to read and write data from their nearest data center to reduce latency.

### 3. **Data Migration**
When migrating from an old Kafka cluster to a new one, mirroring can help in gradually transferring the data. It ensures that the new cluster is synchronized with the old one before fully transitioning the applications to the new setup, minimizing disruption.

### 4. **Load Balancing**
Kafka mirroring can distribute the load across multiple clusters. By replicating topics across different clusters, consumer applications can be balanced to read from different clusters, thus avoiding overloading a single cluster.

### 5. **Hybrid Cloud Deployments**
In scenarios where part of the infrastructure is on-premises and part is in the cloud, Kafka mirroring can ensure data synchronization between on-premises Kafka clusters and cloud-based Kafka clusters, enabling seamless hybrid cloud operations.

### 6. **Analytics and Reporting**
Organizations may want to keep their production Kafka cluster isolated from heavy analytics and reporting queries to ensure performance. By mirroring data to a dedicated analytics Kafka cluster, they can perform complex queries and data processing without impacting the performance of the production environment.

### 7. **Global Data Distribution**
For global companies, Kafka mirroring can ensure that data produced in one region is available in another region. This is useful for applications that require global data access and synchronization, such as real-time analytics, monitoring, or global user data replication.

### 8. **Latency Optimization**
By mirroring Kafka data to clusters that are closer to the end users, organizations can optimize read latencies. This is particularly useful for applications with a global user base that needs real-time data access.

### 9. **Regulatory Compliance**
Certain regulations may require data to be stored in specific geographical locations. Kafka mirroring can help organizations comply with such regulations by replicating data to clusters located in compliant regions.

### 10. **Backup and Archiving**
Kafka mirroring can be used to create backups of Kafka data in a separate cluster. This backup can be used for long-term storage and archiving purposes, providing an additional layer of data protection.

In this blog post, we will guide you through setting up Kafka MirrorMaker as a Kubernetes deployment from scratch. While there are several options for containerizing MirrorMaker, our goal is to demonstrate a straightforward setup. We'll also cover how to ensure fault tolerance for MirrorMaker. This example will involve a simple setup with a single consumer and producer, although you can scale it to include multiple consumers if needed.


## Folder Structure for Kafka MirrorMaker Deployment

To organize the files and configurations for setting up Kafka MirrorMaker on Kubernetes, you should structure your folder as follows:

```shell
kafka-mirrormaker-k8s/
│
├── config/
│   ├── consumer.properties
│   ├── producer.properties
│   └── tools-log4j.properties
│
├── deployment/
│   └── deployment.yaml
│   
│
├── Dockerfile
├── entrypoint.sh
├── README.md
└── .gitignore

```


## Steps for the Setup:

1. **Identify Configurations:**
   We will begin by identifying and configuring the necessary parameters for MirrorMaker to function correctly. This includes specifying the source and destination Kafka clusters, the topics to be mirrored, and any other relevant settings.

2. **Containerize MirrorMaker:**
   Next, we'll create a Docker container for MirrorMaker. This involves writing a Dockerfile that installs Kafka and configures MirrorMaker, ensuring it's ready to be deployed within a Kubernetes environment.

3. **Deploy on Kubernetes:**
   Finally, we'll deploy the containerized MirrorMaker on a Kubernetes cluster. This step involves creating Kubernetes deployment and service files, setting up the necessary resources, and applying these configurations to your Kubernetes cluster.

By the end of this tutorial, you'll have a basic but functional MirrorMaker deployment on Kubernetes, with fault tolerance measures in place. This setup will help you get started with Kafka mirroring in a containerized environment and can be expanded to suit more complex requirements.

Let's dive into each step.
### 1. Identify Configurations

First, we need to identify and set up the necessary configurations for MirrorMaker. Our setup will include:

- **Mirroring Records from Source to Sink:** Specify the source and destination Kafka clusters using consumer and producer properties, provided through the `--consumer.config` and `--producer.config` command-line arguments when starting the MirrorMaker process.

#### Server Configuration

The `server.properties` file contains configuration settings for the Kafka broker. In the context of setting up MirrorMaker, this file ensures that the Kafka broker is correctly configured to interact with both the source and destination Kafka clusters.

**server.properties file:**
```shell
# Kafka Server Configurations
broker.id=0
listeners=PLAINTEXT://${KAFKA_HOSTNAME}:9092
advertised.listeners=PLAINTEXT://${KAFKA_HOSTNAME}:9092
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/var/lib/kafka/logs
num.partitions=1
num.recovery.threads.per.data.dir=1
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
zookeeper.connect=${ZOOKEEPER_HOSTNAME}:2181
zookeeper.connection.timeout.ms=18000
```


#### Producer Configuration

Configure the producer properties to define how MirrorMaker interacts with the destination Kafka cluster.
**producer.properties file:**
```shell
bootstrap.servers=$SINK_CLUSTER_BOOTSTRAP_SERVERS
#optional
 #max.in.flight.requests.per.connection=1 
 #acks=all
compression.type=gzip
offset.commit.interval.ms=10000
```


#### Consumer Configuration

Configure the consumer properties to define how MirrorMaker consumes data from the source Kafka cluster. Additionally, set a `group.id` to uniquely identify the consumer component.
**consumer.properties file:**
```shell
bootstrap.servers=$SOURCE_CLUSTER_BOOTSTRAP_SERVERS
group.id=$SOURCE_CLUSTER_CONSUMER_GROUP
exclude.internal.topics=true
enable.auto.commit=true
auto.offset.reset=latest
consumer.auto.offset.reset=latest
```


#### Specifying Topics to Mirror

Use the `--whitelist` command-line argument to specify which topics should be mirrored. This argument accepts a pattern to match the topics of interest.
```shell
bin/kafka-run-class.sh kafka.tools.MirrorMaker \
--consumer.config config/consumer.properties \
--producer.config config/producer.properties \
--whitelist $WHITELISTED_TOPICS
```


#### Entrypoint Command

The entrypoint command will start the MirrorMaker process with the specified configurations. We have used environment variables to stand in for the actual configuration values. These variable names will be later replaced with their actual values from the environment using the entrypoint script.

**entrypoint.sh file:**
```shell
#!/bin/bash

cd /opt/Kafka/config

# set consumer properties
envsubst < "consumer.properties" > "temp"
cat temp > consumer.properties

# set producer properties
envsubst < "producer.properties" > "temp"
cat temp > producer.properties

# configure logging properties
envsubst < "tools-log4j.properties" > "temp"
cat temp > tools-log4j.properties

rm temp
cd ../

# start mirror maker
bin/kafka-run-class.sh kafka.tools.MirrorMaker \
--consumer.config config/consumer.properties \
--producer.config config/producer.properties \
--whitelist $WHITELISTED_TOPICS
```


#### Configuring Logging

Customize the default logging configuration by modifying the `config/tools-log4j.properties` file inside the Kafka installation directory. This configuration applies to all Kafka command-line tools.

**tools-log4j.properties file:**
```shell
log4j.rootLogger=$LOG_LEVEL, stdout
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=[%d] %p %m (%c)%n
log4j.appender.stdout.Target=System.out
```



### 2. Containerize MirrorMaker

Next, we'll create a Docker container for MirrorMaker. This involves writing a Dockerfile that installs Kafka and configures MirrorMaker.

#### Create Dockerfile:

- **Dockerfile file:**
```Dockerfile
FROM ubuntu
ARG KAFKA_VERSION=3.6.1
ARG SCALA_VERSION=2.12
ARG BINARY_NAME="kafka_${SCALA_VERSION}-${KAFKA_VERSION}"
ARG BINARY_TAR_NAME="${BINARY_NAME}.tgz"
ARG BINARY_ASC_NAME="${BINARY_TAR_NAME}.asc"
ARG BINARY_URI="https://downloads.apache.org/kafka/${KAFKA_VERSION}/${BINARY_TAR_NAME}"
ARG BINARY_ASC_URI="https://downloads.apache.org/kafka/${KAFKA_VERSION}/${BINARY_ASC_NAME}"
RUN apt-get -qq update
RUN apt-get install -qq wget -y > /dev/null
RUN apt-get install -qq gpg -y > /dev/null
RUN apt-get install -qq gettext -y > /dev/null
RUN apt-get install -qq openjdk-11-jre -y > /dev/null
RUN wget -nv $BINARY_URI
RUN wget -nv $BINARY_ASC_URI
RUN wget -nv https://downloads.apache.org/kafka/KEYS
RUN gpg --import KEYS
RUN gpg --verify $BINARY_ASC_NAME $BINARY_TAR_NAME
RUN mkdir -p /opt/Kafka
RUN tar xzf $BINARY_TAR_NAME -C /optRUN mv /opt/$BINARY_NAME/* /opt/Kafka
RUN rm $BINARY_TAR_NAME $BINARY_ASC_NAME KEYS
WORKDIR "/opt/Kafka/"
COPY --chown=600 config config
COPY --chown=755 entrypoint.sh ./
RUN chmod +x entrypoint.sh
ENTRYPOINT ./entrypoint.sh
```


####  Build Docker Image

Build a custom Image with below given Docker file related to the mirror maker build the image according to your environment example (amd64, arm64).
```shell
docker buildx build –platform linux/amd64 -t . #For amd64 environment

docker buildx build –platform linux/arm64 -t . #For arm64 environment
```
After docker image build, push the docker image to ECR/ACR repository.

### 3. Deploy on Kubernetes

Finally, we'll deploy the containerized MirrorMaker on a Kubernetes cluster.

#### Kubernetes Deployment

Create deployment.yml to deploy mirror maker in destination EKS cluster 

**Deployment.yml file:**
```YAML
apiVersion: v1
kind: Namespace
metadata:
 name: $NAMESPACE
 labels:
  app: kafka_mirroring
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: $NAMESPACE
  name: ${NAMESPACE}
  labels:
    app: kafka_mirroring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: kafka_mirroring
  template:
    metadata:
      labels:
        app: kafka_mirroring
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: ${NODE_AFFINITY_KEY}
                operator: ${NODE_AFFINITY_OPERATOR}
                values: ${NODE_AFFINITY_VALUES}
      containers:
      - name: fiber
        image: ${ACCOUNTID}.dkr.ecr.${REGION}.amazonaws.com/${REPOSITORY_URI}:latest
        imagePullPolicy: Always
        resources:
          limits:
            memory: "500Mi"
            cpu: "1000m"
        env:
        - name: SOURCE_CLUSTER_BOOTSTRAP_SERVERS
          value: ${SOURCE_CLUSTER_BOOTSTRAP_SERVERS}
        - name: SOURCE_CLUSTER_CONSUMER_GROUP
          value: ${CONSUMER_GROUP_NAME}
        - name: NUM_STREAMS
          value: "5"
        - name: WHITELISTED_TOPICS
          value: "sample-topic"
        - name: SINK_CLUSTER_BOOTSTRAP_SERVERS
          value: ${SINK_CLUSTER_BOOTSTRAP_SERVERS}
        - name: LOG_LEVEL
          value: INFO

```

After deployment is done, please check the pod logs there you can see whether the connection is established are not and check consumer group is created in the source cluster or not compare the source topic and destination topic message for conformation.

## Conclusion

Setting up Kafka MirrorMaker on Kubernetes provides a robust solution for data replication between Kafka clusters with fault tolerance. By following the steps outlined in this article, you've learned how to:

- Identify and configure MirrorMaker properties for mirroring data from a source to a sink Kafka cluster.
- Containerize MirrorMaker using Docker, ensuring portability and ease of deployment.
- Deploy MirrorMaker on Kubernetes using Kubernetes deployment and ConfigMap resources.
- Customize logging and other configurations to suit your specific requirements.
- Implement fault tolerance measures to ensure uninterrupted data replication.

With this setup, you can reliably replicate data between Kafka clusters, facilitating various use cases such as disaster recovery, data migration, and load balancing. Additionally, the flexibility of Kubernetes allows you to scale the MirrorMaker deployment according to your needs and easily manage resources in a cloud-native environment.