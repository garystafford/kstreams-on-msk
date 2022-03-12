# Kafka Streams Example for Amazon MSK with IAM Auth

Adoption of the simple `WordCountLambdaExample.java` code example from Confluent for use with IAM access control for Amazon MSK. Example code demonstrates the Apache Kafka Streams API (aka Kafka Streams or KStreams).

Build Gradle 7.4. Compiled and ran on OpenJDK 8u322. Build as fat jar (`shadowJar`) using `com.github.johnrengelman.shadow`. Application ran from within OpenJDK 8u322 docker base container, running on an Amazon EKS cluster.

## Reference

- <https://github.com/confluentinc/kafka-streams-examples/blob/7.0.0-post/src/main/java/io/confluent/examples/streams/WordCountLambdaExample.java>
- <https://github.com/JohnReedLOL/kafka-streams/blob/master/src/main/java/io/confluent/examples/streams/SecureKafkaStreamsExample.java>

## Commands to Build, Copy, and Run

Commands to build with Gradle, copy to an Amazon EKS pod container and run.

```shell
# build kstreams application fat jar
gradle clean shadowJar

# get kstream application pod name running on eks cluster 
export KAFKA_POD=$(
  kubectl get pods -n kafka -l app=kstreams-demo | \
    awk 'FNR == 2 {print $1}')

# copy kstreams application fat jar to java container running on eks cluster
kubectl cp -n kafka -c kstreams-app build/libs/KStreamsDemo-1.0-SNAPSHOT-all.jar $KAFKA_POD:/kafka_2.13-3.1.0

# in separate terminal window, exec into java container running on eks cluster to run the kstreams application
kubectl exec -it $KAFKA_POD -n kafka -c kstreams-app -- bash

# in separate terminal window, exec into kafka container running on eks cluster to run producer and consumer commands
kubectl exec -it $KAFKA_POD -n kafka -c kafka-connect -- bash

# *** CHANGE ME - msk bootstrap servers ***
export BOOTSTRAP_SERVERS="b-2.msk-demo-cluster...kafka.us-east-1.amazonaws.com:9098,b-1.msk-demo-cluster...kafka.us-east-1.amazonaws.com:9098"

# run kstreams application (will run continuously)
java -verbose -Xdebug -cp KStreamsDemo-1.0-SNAPSHOT-all.jar io.confluent.examples.streams.WordCountLambdaExample $BOOTSTRAP_SERVERS
java -cp KStreamsDemo-1.0-SNAPSHOT-all.jar io.confluent.examples.streams.WordCountLambdaExample $BOOTSTRAP_SERVERS

# create two topics
bin/kafka-topics.sh \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --command-config config/client-iam.properties \
  --create \
  --topic streams-plaintext-input \
  --partitions 3 \
  --replication-factor 1

bin/kafka-topics.sh \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --command-config config/client-iam.properties \
  --create \
  --topic streams-wordcount-output \
  --partitions 3 \
  --replication-factor 1

# produce messages containing phrases with words
bin/kafka-console-producer.sh \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --producer.config config/client-iam.properties \
  --topic streams-plaintext-input

# display (consume) messages containing phrases with words
bin/kafka-console-consumer.sh \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --consumer.config config/client-iam.properties \
  --topic streams-plaintext-input \
  --from-beginning --max-messages 10 \

# display (consume) word counts, which were processed by kstreams application
bin/kafka-console-consumer.sh \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --consumer.config config/client-iam.properties \
  --topic streams-wordcount-output \
  --from-beginning \
  --property print.key=true \
  --property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer

# list all topics
bin/kafka-topics.sh --list \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --command-config config/client-iam.properties

# get topic size
bin/kafka-log-dirs.sh --describe \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --command-config config/client-iam.properties \
  --topic-list streams-wordcount-output

# describe topic
bin/kafka-log-dirs.sh --describe \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --command-config config/client-iam.properties \
  --topic-list streams-wordcount-output

# delete topic
bin/kafka-topics.sh --delete \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --command-config config/client-iam.properties \
  --topic streams-wordcount-output 
```

## Local Docker Version of Kafka

For debugging project locally using Dockerized version of Apache Kafka and ZooKeeper.

```shell
docker-compose up -d

# exec into Kafka container to interact with Kafka
docker exec -it container_id bash

./opt/bitnami/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic streams-plaintext-input \
  --partitions 1 \
  --replication-factor 1

./opt/bitnami/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic streams-wordcount-output \
  --partitions 1 \
  --replication-factor 1

./opt/bitnami/kafka/bin/kafka-topics.sh \
  --list \
  --bootstrap-server localhost:9092

./opt/bitnami/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic streams-plaintext-input

./opt/bitnami/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic streams-wordcount-output \
  --from-beginning \
  --property print.key=true \
  --property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer

# run locally with Kafka 2.8.1/ZooKeeper running in local containers
java -cp build/libs/KStreamsDemo-1.0-SNAPSHOT-all.jar io.confluent.examples.streams.WordCountLambdaExample localhost:9092
```

---

<i>The contents of this repository represent my viewpoints and not of my past or current employers, including Amazon Web
Services (AWS). All third-party libraries, modules, plugins, and SDKs are the property of their respective owners.</i>
