---
published: true
publish: true
comments: true
tags:
  - Apex
  - Kudu
categories:
  - Misc
permalink: High throughput low latency streaming using Apache Apex and Apache Kudu
---
## Introduction


The last few years has seen HDFS as a great enabler that would help organizations store extremely large amounts of data on commodity hardware. However over the last couple of years the technology landscape changed rapidly and new age engines like Apache Spark, Apache Apex and Apache Flink have started enabling more powerful use cases on a distributed data store paradigm. This has quickly brought out the short-comings of an immutable data store. The primary short comings are
- Immutability resulted in complex lambda architectures when HDFS is used as a store by a query engine
- When data files had to be generated in time bound windows data pipeline frameworks resulted in creating files which are very small in size. Over a period of time this resulted in very small sized files in very large numbers eating up the namenode namespaces to a very great extent.
- With the arrival of SQL-on-Hadoop in a big way and the introduction new age SQL engines like Impala, ETL pipelines resulted in choosing columnar oriented formats albeit with a penalty of accumulating data for a while to gain advantages of the columnar format storage on disk. This reduced the impact of "information now" approach for a hadoop eco system based solution.

Apache Kudu is a next generation storage engine that comes with the following strong points 
- A distributed store
- No single point of failure by adopting the RAFT consensus algorithm under the hood
- Mutable 
- Auto compaction of data sets
- Columnar storage model wrapped over a simple CRUD style API
- Bulk scan patterns possible 
- Predicate push downs when scanning

This post explores the capabilties of [Apache Kudu](https://kudu.apache.org/) in conjunction with the Apex streaming engine. [Apache Apex](https://apex.apache.org/) is a low latency streaming engine which can run on top of YARN and provides many Enterprise grade features out of the box. The post describes the features using a hypothetical use case. The transactions that are processed by a streaming engine need to be written to a data store and subsequently avaiable for a read pattern. The caveat is that the write path needs to be completed in sub-second time windows and read paths should be available within sub-second time frames once the data is written.

## Write paths


## Read paths

