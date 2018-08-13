# Cloudera CDK Powered By Apache Kafka Smoke Tests

These are smoke tests to be used to determine basic functionality of the various parts of a Cloudera Kafka cluster.  One might use these when setting up a new cluster or after a cluster upgrade.

<!-- TOC depthFrom:2 depthTo:3 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Non-Secured Cluster](#non-secured-cluster)
	- [Kafka](#kafka)
	- [Clean It Up](#clean-it-up)
- [Secured Cluster](#secured-cluster)
	- [Preparation](#preparation)
	- [Kafka](#kafka)
	- [Clean It Up](#clean-it-up)

<!-- /TOC -->

## Non-Secured Cluster
These examples assume a non-secured cluster and use of a non-cluster user (i.e. the user "centos").

### ZooKeeper
Basic ZooKeeper functionality.

```bash
# Replace $ZOOKEEPER 'localhost' with the correct hostname.
# Multiple ZooKeepers can be specified with commas: 'host1:2181,host2:2181,host3:2181'
ZOOKEEPER=localhost:2181

cat <<EOF >/tmp/zk.$$
create /zk_test my_data
ls /
get /zk_test
set /zk_test junk
get /zk_test
quit
EOF

cat /tmp/zk.$$ | zookeeper-client -server $ZOOKEEPER
```

### Kafka
Create a test topic.  Write to/read from it.

```bash
# Replace $KAFKA 'localhost' with the correct hostname.
KAFKA=localhost:9092
# Replace the ZOOKEEPER_TOPIC '/kafka' with the correct ZooKeeper root (if you configured one).
ZOOKEEPER_TOPIC=/kafka

kafka-topics --zookeeper ${ZOOKEEPER}${ZOOKEEPER_TOPIC} --create --topic test --partitions 1 --replication-factor 1
kafka-topics --zookeeper ${ZOOKEEPER}${ZOOKEEPER_TOPIC} --list

# Run the consumer and producer in separate windows.
# Type in text to the producer and watch it appear in the consumer.
# ^C to quit.
kafka-console-consumer --zookeeper ${ZOOKEEPER}${ZOOKEEPER_TOPIC} --new-consumer --topic test
kafka-console-producer --broker-list $KAFKA --topic test
```

### Clean It Up
Get rid of all the test bits.

```bash
cat <<EOF >/tmp/zk-rm.$$
delete /zk_test
quit
EOF
cat /tmp/zk-rm.$$ | zookeeper-client -server $ZOOKEEPER
rm -f /tmp/zk.$$ /tmp/zk-rm.$$

kafka-topics --zookeeper $ZOOKEEPER --delete --topic test
```

## Secured Cluster
These examples assume a secured (Kerberized) cluster with TLS and use of a non-cluster principal (i.e. the user/principal "centos").  If the cluster is not using TLS, then do not define the variables that enable it for the individual tests (ie, do not define ITOPTS in the Impala test).

### Preparation
All below commands require Kerberos tickets.

```bash
kinit
```

### ZooKeeper
Basic ZooKeeper functionality.

```bash
# Replace $ZOOKEEPER 'localhost' with the correct hostname.
# Multiple ZooKeepers can be specified with commas: 'host1:2181,host2:2181,host3:2181'
ZOOKEEPER=localhost:2181

cat <<EOF >/tmp/zk.$$
create /zk_test my_data
ls /
get /zk_test
set /zk_test junk
get /zk_test
quit
EOF

cat /tmp/zk.$$ | zookeeper-client -server $ZOOKEEPER
```

### Kafka
Create a test topic.  Write to/read from it.

```bash
# Replace $KAFKA 'localhost' with the correct hostname.
KAFKA=localhost:9093
# Replace the ZOOKEEPER_TOPIC '/kafka' with the correct ZooKeeper root (if you configured one).
ZOOKEEPER_TOPIC=/kafka

kafka-topics --zookeeper ${ZOOKEEPER}${ZOOKEEPER_TOPIC} --create --topic test --partitions 1 --replication-factor 1
kafka-topics --zookeeper ${ZOOKEEPER}${ZOOKEEPER_TOPIC} --list

# Run the consumer and producer in separate windows.
# Type in text to the producer and watch it appear in the consumer.
# ^C to quit.
kafka-console-consumer --zookeeper ${ZOOKEEPER}${ZOOKEEPER_TOPIC} --bootstrap-server $KAFKA --new-consumer --topic test
kafka-console-producer --broker-list $KAFKA --topic test
```

### Clean It Up
Get rid of all the test bits.

```bash
cat <<EOF >/tmp/zk-rm.$$
delete /zk_test
quit
EOF
cat /tmp/zk-rm.$$ | zookeeper-client -server $ZOOKEEPER
rm -f /tmp/zk.$$ /tmp/zk-rm.$$

kafka-topics --zookeeper $ZOOKEEPER --delete --topic test


kdestroy
```
