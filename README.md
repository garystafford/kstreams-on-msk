# Instructions

## Reference

- <https://github.com/confluentinc/kafka-streams-examples/blob/7.0.1-post/src/main/java/io/confluent/examples/streams/io.confluent.examples.streams.WordCountLambdaExample.java>
- <https://github.com/JohnReedLOL/kafka-streams/blob/master/src/main/java/io/confluent/examples/streams/SecureKafkaStreamsExample.java>

```shell
gradle clean shadowJar

export AWS_ACCOUNT=$(aws sts get-caller-identity --output text --query 'Account')
export EKS_REGION="us-east-1"
export CLUSTER_NAME="istio-observe-demo"
export NAMESPACE="kafka"

kubectl cp -n kafka build/libs/KStreamsDemo-1.0-SNAPSHOT.jar $KAFKA_CONTAINER:/kafka_2.13-3.1.0
kubectl cp -n kafka build/libs/KStreamsDemo-1.0-SNAPSHOT-all.jar $KAFKA_CONTAINER:/kafka_2.13-3.1.0

export KAFKA_CONTAINER=$(
  kubectl get pods -n kafka -l app=kafka-connect-msk-v3 | \
    awk 'FNR == 2 {print $1}')

kubectl exec -it $KAFKA_CONTAINER -n kafka -c kafka-connect-msk-v3 -- bash

java -cp KStreamsDemo-1.0-SNAPSHOT-all.jar io.confluent.examples.streams.WordCountLambdaExample

java -verbose -Xdebug -cp KStreamsDemo-1.0-SNAPSHOT-all.jar io.confluent.examples.streams.WordCountLambdaExample $BOOTSTRAP_SERVERS

# java -cp target/kafka-streams-examples-7.0.1-standalone.jar io.confluent.examples.streams.WordCountLambdaExample

# *** CHANGE ME - Bootstrap servers ***
export BOOTSTRAP_SERVERS="b-1.demo-msk-cluster-iam.99s971.c2.kafka.us-east-1.amazonaws.com:9098,b-2.demo-msk-cluster-iam.99s971.c2.kafka.us-east-1.amazonaws.com:9098,b-3.demo-msk-cluster-iam.99s971.c2.kafka.us-east-1.amazonaws.com:9098"
export BOOTSTRAP_SERVERS="b-4.demo-msk-cluster-iam.99s971.c2.kafka.us-east-1.amazonaws.com:9094,b-3.demo-msk-cluster-iam.99s971.c2.kafka.us-east-1.amazonaws.com:9094,b-2.demo-msk-cluster-iam.99s971.c2.kafka.us-east-1.amazonaws.com:9094"
export BOOTSTRAP_SERVERS="b-2.demo-msk-cluster-iam.99s971.c2.kafka.us-east-1.amazonaws.com:9092,b-3.demo-msk-cluster-iam.99s971.c2.kafka.us-east-1.amazonaws.com:9092,b-4.demo-msk-cluster-iam.99s971.c2.kafka.us-east-1.amazonaws.com:9092"

bin/kafka-topics.sh \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --create \
  --topic streams-plaintext-input \
  --partitions 3 \
  --replication-factor 3

bin/kafka-topics.sh \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --create \
  --topic streams-wordcount-output \
  --partitions 3 \
  --replication-factor 3

bin/kafka-console-producer.sh \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --topic streams-plaintext-input

#  --producer.config config/client-iam.properties \

bin/kafka-console-consumer.sh \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --topic streams-plaintext-input \
  --from-beginning --max-messages 10 \

bin/kafka-console-consumer.sh \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --topic streams-wordcount-output \
  --from-beginning \
  --property print.key=true \
  --property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer

# list topics
bin/kafka-topics.sh --list \
  --bootstrap-server $BOOTSTRAP_SERVERS
#  --command-config config/client-iam.properties

# get topic size
bin/kafka-log-dirs.sh --describe \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --topic-list streams-plaintext-input

bin/kafka-log-dirs.sh --describe \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --topic-list streams-wordcount-output

# delete topic
bin/kafka-topics.sh --delete \
  --topic streams-plaintext-input \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --command-config config/client-iam.properties

```