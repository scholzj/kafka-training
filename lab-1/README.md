# Kafka Training - Lab 1: Install and Configuration

* Checkout this repository which will be used during the lab:
  * `git clone https://github.com/scholzj/kafka-training.git`
* Go to the `lab-1` directory
  * `cd lab-1`
* Download the ZIP file with Kafka / AMQ Streams (from VPN):
  * `curl -LO http://download-ipv4.eng.brq.redhat.com/devel/candidates/amq/AMQ-STREAMS-1.0.0-BETA-CR2/amq-streams-1.0.0.BETA-CR2-bin.zip`
* Unzip the file:
  * `unzip amq-streams-1.0.0.BETA-CR2-bin.zip -d kafka && mv kafka/kafka_2.12-2.0.0.redhat-00003/* kafka/ && rm -r kafka/kafka_2.12-2.0.0.redhat-00003/`
* Go to the Kafka directory
  * `cd kafka`

_Note: This lab will use a loooot of terminals ;-)_

## Single node installation

* Review the `config/zookeeper.properties` file with default Zookeeper configuration
* Start Zookeeper in the foreground
  * `bin/zookeeper-server-start.sh config/zookeeper.properties`
* Review the `config/server.properties` file with default Kafka configuration
* Start the Kafka broker in the foreground
  * `bin/kafka-server-start.sh config/server.properties`
* Try to send and receive messages using Kafka console clients to make sure the cluster works
  * `echo "Hello World" | bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test-topic`
  * `bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --from-beginning --topic test-topic --max-messages 1`
* Stop the Kafka broker:
  * `bin/kafka-server-stop.sh`
* Once Kafka is stopped, stop Zookeeper:
  * `bin/zookeeper-server-stop.sh`
* If you want to start the servers in the background, use the `-daemon` option:
  * `bin/zookeeper-server-start.sh -daemon config/zookeeper.properties`
  * `bin/kafka-server-start.sh -daemon config/server.properties`
* Make sure to stop the servers again before moving to the next part

## Multinode cluster

### Zookeeper

* Copy the file `config/zookeeper.properties` to 3 separate files:
  * `cp config/zookeeper.properties config/zookeeper-1.properties && cp config/zookeeper.properties config/zookeeper-2.properties && cp config/zookeeper.properties config/zookeeper-3.properties`
* Edit the configuration files and change following:
  * In `config/zookeeper-1.properties`:
    * Change the `clientPort` field to `2181`
    * Change the `dataDir` field to `dataDir=/tmp/zookeeper-1/`
  * In `config/zookeeper-2.properties`:
    * Change the `clientPort` field to `2182`
    * Change the `dataDir` field to `dataDir=/tmp/zookeeper-2/`
  * In `config/zookeeper-3.properties`:
    * Change the `clientPort` field to `2183`
    * Change the `dataDir` field to `dataDir=/tmp/zookeeper-3/`
  * Into all three configuration files add following block configuring the cluster:

```ini
timeTick=2000
initLimit=5
syncLimit=2

server.1=localhost:2888:3888
server.2=localhost:2889:3889
server.3=localhost:2890:3890
```

* Create the data directories for the three Zookeeper nodes:
  * `mkdir /tmp/zookeeper-1/ && mkdir /tmp/zookeeper-2/ && mkdir /tmp/zookeeper-3/`
* Create the file with the Zookeeper node ID in each of these files:
  * `echo "1" > /tmp/zookeeper-1/myid`
  * `echo "2" > /tmp/zookeeper-2/myid`
  * `echo "3" > /tmp/zookeeper-3/myid`
* Start all three Zookeeper nodes in different terminals:
  * `bin/zookeeper-server-start.sh config/zookeeper-1.properties`
  * `bin/zookeeper-server-start.sh config/zookeeper-2.properties`
  * `bin/zookeeper-server-start.sh config/zookeeper-3.properties`
* To make sure Zookeeepr is running and formed successfully a cluster, you can run:
  * `echo stat | ncat localhost 2181`
  * `echo stat | ncat localhost 2182`
  * `echo stat | ncat localhost 2183`
  * One of these should report `Mode: leader` while the other two should report `Mode: follower`

### Kafka

* Copy the file `config/server.properties` to 3 separate files:
  * `cp config/server.properties config/server-0.properties && cp config/server.properties config/server-1.properties && cp config/server.properties config/server-2.properties`
* Edit the configuration files and change following:
  * In `config/server-0.properties`:
    * Change the `broker.id` field to `0`
    * Set the `listeners` field to `PLAINTEXT://:9092`
    * Set the `log.dirs` field to `/tmp/kafka-logs-0`
    * Change the `zookeeper.connect` field to `localhost:2181,localhost:2182,localhost:2183`
    * Set the following fields accordingly:

```ini
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
```

  * In `config/server-1.properties`:
    * Change the `broker.id` field to `1`
    * Set the `listeners` field to `PLAINTEXT://:9093`
    * Set the `log.dirs` field to `/tmp/kafka-logs-1`
    * Change the `zookeeper.connect` field to `localhost:2181,localhost:2182,localhost:2183`
    * Set the following fields accordingly:

```ini
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
```

  * In `config/server-2.properties`:
    * Change the `broker.id` field to `2`
    * Set the `listeners` field to `PLAINTEXT://:9094`
    * Set the `log.dirs` field to `/tmp/kafka-logs-2`
    * Change the `zookeeper.connect` field to `localhost:2181,localhost:2182,localhost:2183`
    * Set the following fields accordingly:

```ini
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
```

* Create the data directories for the three Kafka brokers:
  * `mkdir /tmp/kafka-logs-0/ && mkdir /tmp/kafka-logs-1/ && mkdir /tmp/kafka-logs-2/`
* Start all three Kafka brokers in different terminals:
  * `bin/kafka-server-start.sh config/server-0.properties`
  * `bin/kafka-server-start.sh config/server-1.properties`
  * `bin/kafka-server-start.sh config/server-2.properties`
* Check that all brokers onnected to the Zookeeper cluster using
  * `echo dump | ncat localhost 2181`

* Try to send and receive messages using Kafka console clients to make sure the cluster works
  * `echo "Hello World" | bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test-topic`
  * `bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --from-beginning --topic test-topic --max-messages 1`

### Adding Kafka node

* Create new topic with 4 partitions and 3 replicas:
  * `bin/kafka-topics.sh --zookeeper localhost:2181 --create --topic my-topic --partitions 4 --replication-factor 3`
* Copy the file `config/server-2.properties` to `config/server-3.properties`:
  * `cp config/server-2.properties config/server-3.properties`
* Edit the configuration files and change following:
  * In `config/server-3.properties`:
    * Change the `broker.id` field to `3`
    * Set the `listeners` field to `PLAINTEXT://:9095`
    * Set the `log.dirs` field to `/tmp/kafka-logs-3`
* Create the data directory 
  * `mkdir /tmp/kafka-logs-3/`
* Start the new broker:
  * `bin/kafka-server-start.sh config/server-3.properties`
* Check that the new broker joined the cluster:
  * `echo dump | ncat localhost 2181`

* To reassign some partitions to the new node:
  * Create a JSON file named `topics-to-move.json` with the names of the topics you want to move:

```json
{
  "version": 1,
  "topics": [
    {
      "topic": "my-topic"
    }
  ]
}
```

* Generate the new recommended assignment which will distribute the topic between brokers 0, 1, 2, and 3
  * `bin/kafka-reassign-partitions.sh --zookeeper localhost:2181 --topics-to-move-json-file topics-to-move.json --broker-list 3 --generate`
  * Save the proposed output to JSON file name `reassignment.json`
  * If needed, you can make changes to the proposed reassignment
* Execute the reassignment:
  * `bin/kafka-reassign-partitions.sh --zookeeper localhost:2181 --reassignment-json-file reassignment.json --throttle 5000000 --execute`
* Check for reassignment completion
    * `bin/kafka-reassign-partitions.sh --zookeeper localhost:2181 --reassignment-json-file reassignment.json --verify`
    * If it completed, check the reassignment results using `bin/kafka-topics.sh --zookeeper localhost:2181 --describe --topic my-topic`