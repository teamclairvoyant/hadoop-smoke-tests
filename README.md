# Hadoop Smoke Tests

This repository containes documents that describe how to run compact, simple,
cut and paste code that will test the basic functionality of the various
components of a Hadoop cluster. One might use these when setting up a new
cluster or after a cluster upgrade to prove that the services work.

## Documents

- [Smoke Tests for a Cloudera CDH Apache Hadoop Cluster](Cloudera-CDH.md)
- [Smoke Tests for a Cloudera CDK Apache Kafka Cluster](Cloudera-CDK.md)

## Guidelines:

* Tests are performed from the Edge/Gateway node.
* No additional software should be needed to run the test.
* No downloads from the internet.
* No changes to the environment.  Tests should clean up after themselves.
* Tests should be compact and simple.
* Results (Pass/Fail) should be obvious/self explainatory.

## Example for HDFS:

* Can I get a directory listing from HDFS?
* Can I write a file to HDFS?
* Can I read that file back from HDFS?

## Contributing

Please see [CONTRIBUTING.md](CONTRIBUTING.md) for information on how to contribute.

Copyright (C) 2018 [Clairvoyant, LLC.](http://clairvoyantsoft.com/)

Licensed under the Creative Commons Attribution-ShareAlike 4.0 International Public License.
