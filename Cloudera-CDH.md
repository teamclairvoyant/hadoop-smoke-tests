# Cloudera Distribution of Apache Hadoop Smoke Tests

These are smoke tests to be used to determine basic functionality of the various parts of a Hadoop cluster.  One might use these when setting up a new cluster or after a cluster upgrade.

TOC

## Non-Secured Cluster
These examples assume a non-secured cluster and use of a non-cluster user (i.e. the user "centos").

### HDFS
Basic HDFS functionality.

```
hdfs dfs -ls /
hdfs dfs -put /etc/hosts /tmp/hosts
hdfs dfs -get /tmp/hosts /tmp/hosts123
cat /tmp/hosts123
```

### MapReduce
Pi Estimator

```
yarn jar /opt/cloudera/parcels/CDH/lib/hadoop-0.20-mapreduce/hadoop-examples.jar pi 10 1000
```

### Hive
Create an external table and query it.

Note:
For Hive on MapReduce, add "set hive.execution.engine=mr;" to the query.
For Hive on Spark, add "set hive.execution.engine=spark;" to the query.


```
# Replace $HIVESERVER2 with the correct hostname that is running the HS2
HIVESERVER2=

# Create hive table
beeline -n `whoami` -u "jdbc:hive2://${HIVESERVER2}:10000/" -e "CREATE TABLE test(id INT, name STRING) ROW FORMAT DELIMITED FIELDS TERMINATED BY ' ' STORED AS TEXTFILE;"

# Create HDFS file
echo "1 justin" >/tmp/hive.$$
echo "2 michael" >>/tmp/hive.$$
# You should probably be the hive user here.
hdfs dfs -put /tmp/hive.$$ /user/hive/warehouse/test/hive

# Query hive table
beeline -n `whoami` -u "jdbc:hive2://${HIVESERVER2}:10000/" -e "SELECT * FROM test WHERE id=1;"
```

### HBase
Create a table and query it.

```
cat <<EOF >/tmp/hbase.$$
create 'test', 'cf'
list 'test'
put 'test', 'row1', 'cf:a', 'value1'
scan 'test'
exit
EOF

hbase shell -n /tmp/hbase.$$
```

### Impala
Query the hive table created earlier.

```
# Replace $IMPALAD with the correct hostname that's running the Impala Daemon
IMPALAD=

impala-shell -i $IMPALAD -q "invalidate metadata;"
impala-shell -i $IMPALAD -q "SELECT * FROM test;"
```

### Spark
Pi Estimator

```
MASTER=yarn /opt/cloudera/parcels/CDH/lib/spark/bin/run-example SparkPi 100
```

Wordcount

```
echo "this is the end. the only end. my friend." > /tmp/sparkin.$$
hdfs dfs -put /tmp/sparkin.$$ /tmp/

cat <<EOF >/tmp/spark.$$
val file = sc.textFile("hdfs:///tmp/sparkin.$$")
val counts = file.flatMap(line => line.split(" ")).map(word => (word, 1)).reduceByKey(_ + _)
counts.saveAsTextFile("hdfs:///tmp/sparkout.$$")
exit
EOF

cat /tmp/spark.$$ | spark-shell --master yarn-client

hdfs dfs -cat /tmp/sparkout.$$/part-\*
```

### Pig
Query data in a file.

```
hdfs dfs -copyFromLocal /etc/passwd /tmp/test.pig.passwd.$$

cat <<EOF >/tmp/pig.$$
A = LOAD '/tmp/test.pig.passwd.$$' USING PigStorage(':');
B = FOREACH A GENERATE \$0 AS id;
STORE B INTO '/tmp/test.pig.out.$$';
EOF

pig /tmp/pig.$$

hdfs dfs -cat /tmp/test.pig.out.$$/part-m-00000
```

### Solr
Create a test collection.  Index it and query it.

```
SOLRSERVER=

solrctl instancedir --generate /tmp/solr.$$
solrctl instancedir --create test_config /tmp/solr.$$
solrctl collection --create test_collection -s 1 -c test_config
cd /opt/cloudera/parcels/CDH/share/doc/solr-doc*/example/exampledocs
java -Durl=http://${SOLRSERVER}:8983/solr/test_collection/update -jar post.jar *.xml
curl "http://${SOLRSERVER}:8983/solr/test_collection_shard1_replica1/select?q=*%3A*&wt=json&indent=true"
```

### Kudu
#### Impala
Create a Kudu table and query it.

```
# Replace $IMPALAD with the correct hostname that's running the Impala Daemon
IMPALAD=

impala-shell -i $IMPALAD -q 'CREATE TABLE kudu_test(id BIGINT, name STRING, PRIMARY KEY(id)) PARTITION BY HASH PARTITIONS 3 STORED AS KUDU;'

impala-shell -i $IMPALAD -q 'INSERT INTO TABLE kudu_test VALUES (1, "wasim"), (2, "ninad"), (3, "mohsin");'

impala-shell -i $IMPALAD -q 'SELECT * FROM kudu_test WHERE id=1;'
```

### Clean It Up
Get rid of all the test bits.

```
hdfs dfs -rm /tmp/hosts
rm -f /tmp/hosts123

beeline -n `whoami` -u "jdbc:hive2://${HIVESERVER2}:10000/" -e "DROP TABLE test;"
rm -f /tmp/hive.$$

cat <<EOF >/tmp/hbase-rm.$$
disable 'test'
drop 'test'
exit
EOF
hbase shell -n /tmp/hbase-rm.$$
rm -f /tmp/hbase.$$ /tmp/hbase-rm.$$

hdfs dfs -rm -R /tmp/sparkout.$$ /tmp/sparkin.$$
rm -f /tmp/spark.$$

hdfs dfs -rm -R /tmp/test.pig.passwd.$$ /tmp/test.pig.out.$$
rm -f /tmp/pig.$$

solrctl collection --delete test_collection
solrctl instancedir --delete test_config
sudo su - solr -s /bin/bash -c "hdfs dfs -rm -R -skipTrash /solr/test_collection"
rm -rf /tmp/test_config.$$

impala-shell -i $IMPALAD -q 'DROP TABLE kudu_test;'
```

## Secured Cluster
These examples assume a secured (Kerberized) cluster with TLS and use of a non-cluster principal (i.e. the user/principal "centos").  If the cluster is not using TLS, then do not define the variables that enable it for the individual tests (ie, do not define ITOPTS in the Impala test).

### Preparation
All below commands require Kerberos tickets.

```
kinit
```

### HDFS
Basic HDFS functionality.

```
hdfs dfs -ls /
hdfs dfs -put /etc/hosts /tmp/hosts
hdfs dfs -get /tmp/hosts /tmp/hosts123
cat /tmp/hosts123
```

### MapReduce
Pi Estimator

```
yarn jar /opt/cloudera/parcels/CDH/lib/hadoop-0.20-mapreduce/hadoop-examples.jar pi 10 1000
```

### Hive
Create an external table and query it.

Note:
For Hive on MapReduce, add "set hive.execution.engine=mr;" to the query.
For Hive on Spark, add "set hive.execution.engine=spark;" to the query.


```
# Replace $HIVESERVER2 with the correct hostname that is running the HS2
HIVESERVER2=
REALM=`awk '/^ *default_realm/{print $3}' /etc/krb5.conf`
BKOPTS=";principal=hive/_HOST@${REALM}"
BTOPTS=";ssl=true;sslTrustStore=/usr/java/default/jre/lib/security/jssecacerts;trustStorePassword=changeit"

# Create hive table
beeline -u "jdbc:hive2://${HIVESERVER2}:10000/${BKOPTS}${BTOPTS}" -e "CREATE TABLE test(id INT, name STRING) ROW FORMAT DELIMITED FIELDS TERMINATED BY ' ' STORED AS TEXTFILE;"

# Create HDFS file
echo "1 justin" >/tmp/hive.$$
echo "2 michael" >>/tmp/hive.$$
hdfs dfs -put /tmp/hive.$$ /user/hive/warehouse/test/hive

# Query hive table
beeline -u "jdbc:hive2://${HIVESERVER2}:10000/${BKOPTS}${BTOPTS}" -e "SELECT * FROM test WHERE id=1;"
```

### HBase
Create a table and query it.

```
cat <<EOF >/tmp/hbase.$$
create 'test', 'cf'
list 'test'
put 'test', 'row1', 'cf:a', 'value1'
scan 'test'
exit
EOF

hbase shell -n /tmp/hbase.$$
```

### Impala
Query the hive table created earlier.

```
# Replace $IMPALAD with the correct hostname that's running the Impala Daemon
IMPALAD=
IKOPTS="-k"
ITOPTS="--ssl --ca_cert=/opt/cloudera/security/x509/ca-chain.cert.pem"

impala-shell -i $IMPALAD $IKOPTS $ITOPTS -q "invalidate metadata;"
impala-shell -i $IMPALAD $IKOPTS $ITOPTS -q "SELECT * FROM test;"
```

### Spark
Pi Estimator

```
MASTER=yarn /opt/cloudera/parcels/CDH/lib/spark/bin/run-example SparkPi 100
```
Wordcount

```
echo "this is the end. the only end. my friend." > /tmp/sparkin.$$
hdfs dfs -put /tmp/sparkin.$$ /tmp/

cat <<EOF >/tmp/spark.$$
val file = sc.textFile("hdfs:///tmp/sparkin.$$")
val counts = file.flatMap(line => line.split(" ")).map(word => (word, 1)).reduceByKey(_ + _)
counts.saveAsTextFile("hdfs:///tmp/sparkout.$$")
exit
EOF

cat /tmp/spark.$$ | spark-shell --master yarn-client

hdfs dfs -cat /tmp/sparkout.$$/part-\*
```

### Pig
Query data in a file.

```
hdfs dfs -copyFromLocal /etc/passwd /tmp/test.pig.passwd.$$

cat <<EOF >/tmp/pig.$$
A = LOAD '/tmp/test.pig.passwd.$$' USING PigStorage(':');
B = FOREACH A GENERATE \$0 AS id;
STORE B INTO '/tmp/test.pig.out.$$';
EOF

pig /tmp/pig.$$

hdfs dfs -cat /tmp/test.pig.out.$$/part-m-00000
```

### Solr
Create a test collection.  Index it and query it.

```
SOLRSERVER=
SKOPTS="--negotiate -u :"
STPROTO=https
STPORT=8985

solrctl instancedir --generate /tmp/solr.$$
solrctl instancedir --create test_config /tmp/solr.$$
solrctl collection --create test_collection -s 1 -c test_config
cd /opt/cloudera/parcels/CDH/share/doc/solr-doc*/example/exampledocs
# Next line does not work.  Need to get java to use SPNEGO.
java -Durl=${STPROTO:-http}://${SOLRSERVER}:${STPORT:-8983}/solr/test_collection/update -jar post.jar *.xml
curl $SKOPTS "${STPROTO:-http}://${SOLRSERVER}:${STPORT:-8983}/solr/test_collection_shard1_replica1/select?q=*%3A*&wt=json&indent=true"
```

### Clean It Up
Get rid of all the test bits.

```
hdfs dfs -rm /tmp/hosts
rm -f /tmp/hosts123

beeline -u "jdbc:hive2://${HIVESERVER2}:10000/${BKOPTS}${BTOPTS}" -e "DROP TABLE test;"
rm -f /tmp/hive.$$

cat <<EOF >/tmp/hbase-rm.$$
disable 'test'
drop 'test'
exit
EOF
hbase shell -n /tmp/hbase-rm.$$
rm -f /tmp/hbase.$$ /tmp/hbase-rm.$$

hdfs dfs -rm -R /tmp/sparkout.$$ /tmp/sparkin.$$
rm -f /tmp/spark.$$

hdfs dfs -rm -R /tmp/test.pig.passwd.$$ /tmp/test.pig.out.$$
rm -f /tmp/pig.$$

solrctl collection --delete test_collection
solrctl instancedir --delete test_config
#kinit solr
#hdfs dfs -rm -R -skipTrash /solr/test_collection
rm -rf /tmp/test_config.$$


kdestroy
```

## Secured Cluster with Sentry
These examples assume a secured (Kerberized) cluster with TLS and Sentry and use of a non-cluster principal (i.e. the user/principal "centos").

**DRAFT**

### Preparation
Sentry needs to be configured with roles.

HBase needs to be configured with roles.

Hive needs to have admin groups (AD or LDAP) added to the Hive Metastore Access Control and Proxy User Groups Overrride (in addition to hue and hive).

All below commands require Kerberos tickets.

```
kinit
```

### HDFS
Basic HDFS functionality.

```
hdfs dfs -ls /
hdfs dfs -put /etc/hosts /tmp/hosts
hdfs dfs -get /tmp/hosts /tmp/hosts123
cat /tmp/hosts123
```

### MapReduce
Pi Estimator

```
yarn jar /opt/cloudera/parcels/CDH/lib/hadoop-0.20-mapreduce/hadoop-examples.jar pi 10 1000
```

### Hive
Create an external table and query it.

Note:
For Hive on MapReduce, add "set hive.execution.engine=mr;" to the query.
For Hive on Spark, add "set hive.execution.engine=spark;" to the query.

```
# Replace $HIVESERVER2 with the correct hostname that is running the HS2
HIVESERVER2=
REALM=`awk '/^ *default_realm/{print $3}' /etc/krb5.conf`
BKOPTS=";principal=hive/_HOST@${REALM}"
BTOPTS=";ssl=true;sslTrustStore=/usr/java/default/jre/lib/security/jssecacerts;trustStorePassword=changeit"

# Create hive table
beeline -u "jdbc:hive2://${HIVESERVER2}:10000/${BKOPTS}${BTOPTS}" -e "CREATE TABLE test(id INT, name STRING) ROW FORMAT DELIMITED FIELDS TERMINATED BY ' ' STORED AS TEXTFILE;"

# Insert data
beeline -u "jdbc:hive2://${HIVESERVER2}:10000/${BKOPTS}${BTOPTS}" -e "INSERT INTO TABLE test VALUES (1, "justin"), (2, "michael");"

# Query hive table
beeline -u "jdbc:hive2://${HIVESERVER2}:10000/${BKOPTS}${BTOPTS}" -e "SELECT * FROM test WHERE id=1;"
```

### HBase
Create a table and query it.

```
cat <<EOF >/tmp/hbase.$$
create 'test', 'cf'
list 'test'
put 'test', 'row1', 'cf:a', 'value1'
scan 'test'
exit
EOF

hbase shell -n /tmp/hbase.$$
```

### Impala
Query the hive table created earlier.

```
# Replace $IMPALAD with the correct hostname that's running the Impala Daemon
IMPALAD=
IKOPTS="-k"
ITOPTS="--ssl --ca_cert=/opt/cloudera/security/x509/ca-chain.cert.pem"

impala-shell -i $IMPALAD $IKOPTS $ITOPTS -q "invalidate metadata;"
impala-shell -i $IMPALAD $IKOPTS $ITOPTS -q "SELECT * FROM test;"
```

### Spark
Pi Estimator

```
MASTER=yarn /opt/cloudera/parcels/CDH/lib/spark/bin/run-example SparkPi 100
```

Wordcount

```
echo "this is the end. the only end. my friend." > /tmp/sparkin.$$
hdfs dfs -put /tmp/sparkin.$$ /tmp/

cat <<EOF >/tmp/spark.$$
val file = sc.textFile("hdfs:///tmp/sparkin.$$")
val counts = file.flatMap(line => line.split(" ")).map(word => (word, 1)).reduceByKey(_ + _)
counts.saveAsTextFile("hdfs:///tmp/sparkout.$$")
exit
EOF

cat /tmp/spark.$$ | spark-shell --master yarn-client

hdfs dfs -cat /tmp/sparkout.$$/part-\*
```

### Pig
Query data in a file.

```
hdfs dfs -copyFromLocal /etc/passwd /tmp/test.pig.passwd.$$

cat <<EOF >/tmp/pig.$$
A = LOAD '/tmp/test.pig.passwd.$$' USING PigStorage(':');
B = FOREACH A GENERATE \$0 AS id;
STORE B INTO '/tmp/test.pig.out.$$';
EOF

pig /tmp/pig.$$

hdfs dfs -cat /tmp/test.pig.out.$$/part-m-00000
```

### Clean It Up
Get rid of all the test bits.

```
hdfs dfs -rm /tmp/hosts
rm -f /tmp/hosts123

beeline -u "jdbc:hive2://${HIVESERVER2}:10000/${BKOPTS}${BTOPTS}" -e "DROP TABLE test;"
rm -f /tmp/hive.$$

cat <<EOF >/tmp/hbase-rm.$$
disable 'test'
drop 'test'
exit
EOF
hbase shell -n /tmp/hbase-rm.$$
rm -f /tmp/hbase.$$ /tmp/hbase-rm.$$

hdfs dfs -rm -R /tmp/sparkout.$$ /tmp/sparkin.$$
rm -f /tmp/spark.$$

hdfs dfs -rm -R /tmp/test.pig.passwd.$$ /tmp/test.pig.out.$$
rm -f /tmp/pig.$$


kdestroy
```
