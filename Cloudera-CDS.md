# Cloudera CDS Powered by Apache Spark Smoke Tests

These are smoke tests to be used to determine basic functionality of the various parts of a Cloudera Spark2-on-YARN cluster.  One might use these when setting up a new cluster or after a cluster upgrade.

<!-- TOC depthFrom:2 depthTo:3 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Non-Secured Cluster](#non-secured-cluster)
	- [Spark2](#spark2)
	- [Clean It Up](#clean-it-up)
- [Secured Cluster](#secured-cluster)
	- [Preparation](#preparation)
	- [Spark2](#spark2)
	- [Clean It Up](#clean-it-up)

<!-- /TOC -->

## Non-Secured Cluster
These examples assume a non-secured cluster and use of a non-cluster user (i.e. the user "centos").

### Spark2
Pi Estimator

```bash
MASTER=yarn /opt/cloudera/parcels/SPARK2/lib/spark2/bin/run-example SparkPi 100
```
Wordcount

```bash
echo "this is the end. the only end. my friend." > /tmp/sparkin2.$$
hdfs dfs -put /tmp/sparkin2.$$ /tmp/

cat <<EOF >/tmp/spark2.$$
val file = sc.textFile("hdfs:///tmp/sparkin2.$$")
val counts = file.flatMap(line => line.split(" ")).map(word => (word, 1)).reduceByKey(_ + _)
counts.saveAsTextFile("hdfs:///tmp/sparkout2.$$")
exit
EOF

cat /tmp/spark2.$$ | spark2-shell --master yarn-client

hdfs dfs -cat /tmp/sparkout2.$$/part-\*
```

### Clean It Up
Get rid of all the test bits.

```bash
hdfs dfs -rm -R /tmp/sparkout2.$$ /tmp/sparkin2.$$
rm -f /tmp/spark2.$$
```

## Secured Cluster
These examples assume a secured (Kerberized) cluster with TLS and use of a non-cluster principal (i.e. the user/principal "centos").  If the cluster is not using TLS, then do not define the variables that enable it for the individual tests (ie, do not define ITOPTS in the Impala test).

### Preparation
All below commands require Kerberos tickets.

```bash
kinit
```

### Spark2
Pi Estimator

```bash
MASTER=yarn /opt/cloudera/parcels/SPARK2/lib/spark2/bin/run-example SparkPi 100
```
Wordcount

```bash
echo "this is the end. the only end. my friend." > /tmp/sparkin2.$$
hdfs dfs -put /tmp/sparkin2.$$ /tmp/

cat <<EOF >/tmp/spark2.$$
val file = sc.textFile("hdfs:///tmp/sparkin2.$$")
val counts = file.flatMap(line => line.split(" ")).map(word => (word, 1)).reduceByKey(_ + _)
counts.saveAsTextFile("hdfs:///tmp/sparkout2.$$")
exit
EOF

cat /tmp/spark2.$$ | spark2-shell --master yarn-client

hdfs dfs -cat /tmp/sparkout2.$$/part-\*
```

### Clean It Up
Get rid of all the test bits.

```bash
hdfs dfs -rm -R /tmp/sparkout2.$$ /tmp/sparkin2.$$
rm -f /tmp/spark2.$$


kdestroy
```
