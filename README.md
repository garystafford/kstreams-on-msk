# Kafka Streams Example for Amazon MSK with AWS IAM

Simple [`WordCountLambdaExample.java`](https://github.com/confluentinc/kafka-streams-examples/blob/7.0.0-post/src/main/java/io/confluent/examples/streams/WordCountLambdaExample.java)
code example from Confluent, adopted for use
with [IAM access control for Amazon MSK](https://docs.aws.amazon.com/msk/latest/developerguide/iam-access-control.html).
The example code demonstrates the use of
the [Apache Kafka Streams API](https://kafka.apache.org/documentation/streams/) (_aka Kafka Streams or KStreams_).

For this demonstration, the appliation was built with [Gradle 7.4](https://gradle.org/releases/) as a fat
jar (`shadowJar`) using [`com.github.johnrengelman.shadow`](https://github.com/johnrengelman/shadow). The application
was compiled using Java [OpenJDK 8u322](https://mail.openjdk.java.net/pipermail/jdk8u-dev/2022-January/014522.html). The
application was run from within an OpenJDK 8u322 container on an Amazon EKS cluster. The Kafka API commands were run from
within a second container, containing the Kafka APIs, also running on the same Amazon EKS cluster. The Amazon MSK cluster was
running Apache Kafka version 2.8.1.

## Build, Copy, and Run the KStreams Application on EKS/MSK

Commands to create the Kafka topics, build the application with Gradle, copy fat jar to an Amazon EKS pod container, run
the application, and produce input messages.

```shell
# build kstreams application fat jar
gradle clean shadowJar
```

```shell
# get kstream application pod name running on eks cluster
export AWS_ACCOUNT=$(aws sts get-caller-identity --output text --query 'Account')
export EKS_REGION="us-east-1"
export CLUSTER_NAME="eks-demo-cluster"
export NAMESPACE="kafka"
export APPLICATION="kstreams-demo"
export KAFKA_POD=$(
  kubectl get pods -n $NAMESPACE -l app=$APPLICATION | \
    awk 'FNR == 2 {print $1}')
```

```shell
# in separate terminal window, exec into kafka container running on eks cluster to run the kafka api commands
kubectl exec -it $KAFKA_POD -n kafka -c kafka-connect -- bash

# *** CHANGE ME - msk bootstrap servers ***
export BOOTSTRAP_SERVERS="b-2.msk-demo-cluster...kafka.us-east-1.amazonaws.com:9098,b-1.msk-demo-cluster...kafka.us-east-1.amazonaws.com:9098"

# create two topics
bin/kafka-topics.sh \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --command-config config/client-iam.properties \
  --create \
  --topic streams-plaintext-input \
  --partitions 1 \
  --replication-factor 1

bin/kafka-topics.sh \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --command-config config/client-iam.properties \
  --create \
  --topic streams-wordcount-output \
  --partitions 1 \
  --replication-factor 1

# input text (produce messages) (NOTE kstreams app must be running first - see below)
bin/kafka-console-producer.sh \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --producer.config config/client-iam.properties \
  --topic streams-plaintext-input

# display word counts (consume messages) processed by kstreams app (NOTE kstreams app must be running first - see below)
bin/kafka-console-consumer.sh \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --consumer.config config/client-iam.properties \
  --topic streams-wordcount-output \
  --from-beginning \
  --property print.key=true \
  --property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer
```

```shell
# copy kstreams application fat jar to java container running on eks cluster
kubectl cp -n kafka -c kstreams-app build/libs/KStreamsDemo-1.0-SNAPSHOT-all.jar $KAFKA_POD:/kafka_2.13-3.1.0

# in separate terminal window, exec into java container running on eks cluster to run the kstreams application
kubectl exec -it $KAFKA_POD -n kafka -c kstreams-app -- bash

# *** CHANGE ME - msk bootstrap servers ***
export BOOTSTRAP_SERVERS="b-2.msk-demo-cluster...kafka.us-east-1.amazonaws.com:9098,b-1.msk-demo-cluster...kafka.us-east-1.amazonaws.com:9098"

# run kstreams application (will run continuously)
# with debug/verbose output
java -verbose -Xdebug -cp KStreamsDemo-1.0-SNAPSHOT-all.jar io.confluent.examples.streams.WordCountLambdaExampleIAMAuth $BOOTSTRAP_SERVERS

# without verbose output
java -cp KStreamsDemo-1.0-SNAPSHOT-all.jar io.confluent.examples.streams.WordCountLambdaExampleIAMAuth $BOOTSTRAP_SERVERS
```

### Optional Kafka API Commands

```shell
# list all topics
bin/kafka-topics.sh --list \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --command-config config/client-iam.properties

# display input text (consume messages)
bin/kafka-console-consumer.sh \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --consumer.config config/client-iam.properties \
  --topic streams-plaintext-input \
  --from-beginning --max-messages 100 \

# describe topic / get topic size
bin/kafka-log-dirs.sh --describe \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --command-config config/client-iam.properties \
  --topic-list streams-wordcount-output

# delete topics
bin/kafka-topics.sh --delete \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --command-config config/client-iam.properties \
  --topic streams-plaintext-input 

bin/kafka-topics.sh --delete \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --command-config config/client-iam.properties \
  --topic streams-wordcount-output
```

## Local Docker Version of Kafka

Local Dockerized version of Apache Kafka 2.8.1 (same as Amazon MSK version used in demo) and ZooKeeper for debugging
project. Using unauthenticated Kafka configuration.

```shell
# create Apache Kafka and ZooKeeper containers locally
docker-compose up -d

# exec into Kafka container to interact with Kafka
docker exec -it kstreamsdemo_kafka_1 bash

cd ./opt/bitnami/kafka/bin/

# create two topics
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic streams-plaintext-input \
  --partitions 1 \
  --replication-factor 1

kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic streams-wordcount-output \
  --partitions 1 \
  --replication-factor 1

# list all topics
kafka-topics.sh \
  --list \
  --bootstrap-server localhost:9092

# input text (produce messages) (NOTE kstreams app must be running first - see below)
kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic streams-plaintext-input

# display word counts (consume messages) processed by kstreams app (NOTE kstreams app must be running first - see below)
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic streams-wordcount-output \
  --from-beginning \
  --property print.key=true \
  --property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer

# run kstreams application locally using dockerized version of kafka with no auth
java -cp build/libs/KStreamsDemo-1.0-SNAPSHOT-all.jar io.confluent.examples.streams.WordCountLambdaExampleNoAuth localhost:9092

# optional: delete topics
kafka-topics.sh --delete \
  --bootstrap-server localhost:9092 \
  --topic streams-plaintext-input 

kafka-topics.sh --delete \
  --bootstrap-server localhost:9092 \
  --topic streams-wordcount-output
```

---

<i>The contents of this repository represent my viewpoints and not of my past or current employers, including Amazon Web
Services (AWS). All third-party libraries, modules, plugins, and SDKs are the property of their respective owners. The author(s) assumes no responsibility or liability for any errors or omissions in the content of this site. The information contained in this site is provided on an "as is" basis with no guarantees of completeness, accuracy, usefulness or timeliness.</i>
