---
published: false
comments: true
header:
  image: /assets/images/patrick-lindenberg-191841.jpg
tags:
  - Apex
  - Kudu
categories:
  - Misc
permalink: using-kudu-output-operator-in-apex
---

# Using the Kudu outout operator in Apex

Malhar release 3.8.0 brings in support for writing streams into Kudu as the store.This blog post describes the various usage patterns that are supported by Kudu output operator on the Apex platform.

The blog describes the features supported by the Kudu output operator by walking through the following fictitious business use case. The business use case requires processing data from a kafka topic and persisting the data into a kudu store. Each message from the kafka topic is to be persisted into two tables in kudu. 

1. Transactions table
2. Devices table

The transactions table has the following columns 
- transactionid ( primary key )
- timestamp ( primary key )
- is_stepup
- transaction_amnt

The devices table has a single column that defines part of its primary key
- deviceid ( primary key )
- timestamp

The message from kafka topic would be in a JSON format having the following fields:
- transactionid
- timestamp
- is_stepup
- transaction_amnt
- deviceid 


