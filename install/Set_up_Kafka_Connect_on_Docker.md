
# Set up Kafka Connect on Docker

* There are two docker images available :

 1. cp-kafka-connect
 2. cp-kafka-connect-base

* Functionally, the `cp-kafka-connect` and the `cp-kafka-connect-base` images are identical. 

* The only difference is that the `cp-kafka-connect` image already contains several 
  of Confluent’s connectors, whereas the `cp-kafka-connect-base` image comes with none 
  by default

# Part 1 : Set up Kafka

## Step 1 : Setup a kafka instance

**Launch external Zookeeper instance**

```bash
docker pull zookeeper:3.6.1

docker container run \
--network host \
--name zookeeper-standalone \
-e ZOO_LOG4J_PROP="INFO,ROLLINGFILE" \
-v zk-datalog:/datalog \
-v zk-data:/data \
-v zk-logs:/logs \
-d zookeeper:3.6.1
```

```bash
# Housekeeping
docker container logs zookeeper-standalone

docker container stop zookeeper-standalone

docker container rm zookeeper-standalone
```

**Launch Kafka**

```bash
mkdir -p ${HOME}/volumes/kafka
chmod -R ugo+rwx ${HOME}/volumes/kafka

KAFKA_HOME=${HOME}/volumes/kafka

docker pull wurstmeister/kafka

docker container run -d \
--name kafka-standalone \
--network host \
-e KAFKA_ADVERTISED_HOST_NAME=localhost \
-e KAFKA_ADVERTISED_PORT=9092 \
-e KAFKA_BROKER_ID=1 \
-e KAFKA_ZOOKEEPER_CONNECT=localhost:2181 \
-e ZK=zk \
-v kafka-volume:/kafka \
-v ${HOME}/volumes/kafka/samples:/opt/samples/ \
-t wurstmeister/kafka
```

## Step 2 : Create housekeeping topics

* Connect stores config, status, and offsets of the connectors in Kafka topics

* We will create these topics now using the Kafka broker we created above.

```bash
docker container exec -it kafka-standalone bash

kafka-topics.sh \
--create \
--zookeeper localhost:2181 \
--partitions 1 \
--replication-factor 1 \
--config cleanup.policy=compact \
--topic connect-config-storage

kafka-topics.sh \
--create \
--zookeeper localhost:2181 \
--partitions 1 \
--replication-factor 1 \
--config cleanup.policy=compact \
--topic connect-offset-storage

kafka-topics.sh \
--create \
--zookeeper localhost:2181 \
--partitions 1 \
--replication-factor 1 \
--config cleanup.policy=compact \
--topic connect-status-storage

# Check
kafka-topics.sh \
--list \
--zookeeper localhost:2181
```

```bash
# Housekeeping
docker container logs kafka-standalone

docker container stop kafka-standalone

docker container rm kafka-standalone
```

# Part 2 : Set up Kafka Connect

## Step 1 : Create a folder to hold jars for Kafka Connect

```bash
mkdir -p ${HOME}/volumes/kafka-connect

chmod -R ugo+rwx ${HOME}/volumes/kafka-connect
```

## Step 2 : Start a connect worker with JSON support

```bash
docker pull confluentinc/cp-kafka-connect:5.4.3

docker run -d \
--name=kafka-connect-demo \
--network host \
-e CONNECT_BOOTSTRAP_SERVERS=localhost:9092 \
-e CONNECT_REST_PORT=28082 \
-e CONNECT_REST_HOST_NAME="localhost" \
-e CONNECT_REST_ADVERTISED_HOST_NAME="localhost" \
-e CONNECT_GROUP_ID="connect-group" \
-e CONNECT_CONFIG_STORAGE_TOPIC="connect-config-storage" \
-e CONNECT_OFFSET_STORAGE_TOPIC="connect-offset-storage" \
-e CONNECT_STATUS_STORAGE_TOPIC="connect-status-storage" \
-e CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR=1 \
-e CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR=1 \
-e CONNECT_STATUS_STORAGE_REPLICATION_FACTOR=1 \
-e CONNECT_KEY_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
-e CONNECT_VALUE_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
-e CONNECT_INTERNAL_KEY_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
-e CONNECT_INTERNAL_VALUE_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
-e CONNECT_LOG4J_ROOT_LOGLEVEL=DEBUG \
-e CONNECT_PLUGIN_PATH=/usr/share/java,/etc/kafka-connect/jars \
-v ${HOME}/volumes/kafka-connect/file:/tmp/quickstart \
-v ${HOME}/volumes/kafka-connect/jars:/etc/kafka-connect/jars \
confluentinc/cp-kafka-connect:5.4.3
```

**Test**

Wait a while for Kafka connect to set itself up

```bash
docker container logs kafka-connect-demo

docker container exec -it kafka-connect-demo bash

# Check to see the connectors loaded
curl -sS localhost:28082/connector-plugins 
```

**Housekeeping**

```bash
docker container logs kafka-connect-demo

docker container stop kafka-connect-demo

docker container rm kafka-connect-demo

docker container exec -it kafka-connect-demo bash
```

# Note

* There seems to be a bug wherein, the REST port is not exposed outside container.
  Hence, port 28082 is only accessible inside the container despite using host networking

References
==========
https://docs.confluent.io/3.1.1/cp-docker-images/docs/tutorials/connect-avro-jdbc.html

https://docs.confluent.io/3.1.1/cp-docker-images/docs/quickstart.html
