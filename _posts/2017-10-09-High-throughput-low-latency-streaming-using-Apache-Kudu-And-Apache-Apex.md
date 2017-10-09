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
- With the arrival of SQL-on-Hadoop in a big way with the introduction new age SQL engines like Impala, ETL pipelines resulted in choosing columnar oriented formats albeit with a penalty of accumulating data for a while to gain advantages of the columnar format storage on disk.
