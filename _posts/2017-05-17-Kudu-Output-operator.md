---
published: true
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

# Using the Kudu output operator in Apex

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

## Design of the output operator

Kudu output operator expects the upstream operator to send the tuple as an instance of ExecutionContext. The ExecutionContext instance contains the payload/POJO class that is to be used to write to the Kudu table. Apart from this, the ExecutionContext also specifies the type of mutation that needs to be performed on the kudu store. This allows a single instance of the kudu output operator to perform all models of a mutation in a single apex application.

The following sections describe the features of the Kudu output operator using various use cases.

## Creating an instance of the Kudu Output operator
The simplest way to create an instance of the Kudu Output operator is by adding a file with the name kuduoutputoperator.properties. This file follows the simple java property file conventions. The following are the main keys that are supported and mandatory. 

The following is a sample template that can be used to specify the output operator properties.
~~~
masterhosts=192.168.1.41:7051
tablename=devicestatus
pojoclassname=github.ananthc.sampleapps.apex.TransactionPayload
~~~

The masterhosts key represents the Kudu cluster master hosts. The value can be comma separated expression to specify multiple masters.
The tablename key represents the table name of the Kudu cluster that is being used to write the data to.
The pojoclassname key is used to specify the instance of the payload class object that will be embedded in the ExecutionContext object that comes from the upstream operator. 

The following snippet of code represents the instantiation of the operator in the Apex Application class. 

~~~java
    try {
      BaseKuduOutputOperator deviceStatusTableKuduOutputOperator = new BaseKuduOutputOperator();
      ....
    } catch (IOException| ClassNotFoundException e) {
      ....
    }
    dag.addOperator("devicestatuskuduoutput",deviceStatusTableKuduOutputOperator);    
~~~

## Using multiple instances of the operator in a single apex application
The kudu output operator also allows for multiple instances of the kudu output operator to be created in the same application. Consider a sample application wherein messages from a single Kafka topic will be used to persist to multiple Kudu tables in the same application. 

You can create multiple instances of the Kudu operator. Since we cannot have multiple configuration files with the same name, the Kudu output operator constructor allows a string parameter to be passed which represents the configuration property file. This file needs to be present in the root classpath of the application. The java code snippet for this style of operator instance creation can be represented as follows:

~~~java
    try {
      TransactionsTableKuduOutputOperator transactionsTableKuduOutputOperator = new TransactionsTableKuduOutputOperator("transactiontable.properties");
      .....
    } catch (IOException| ClassNotFoundException e) {
      ....
    } 
~~~

Note that the Kudu output operator is using a file named "transactiontable.properties" to define the values for master hosts, table name and the pojo payload class. Note that in the code snippet above, we are using a derived class of the Baseoutput operator which also supports the string parameter based constructor. 

## Specifying the type of mutation from the upstream operator
The upstream operator can specify the type of mutation that needs to be used to perform the mutation. The following types of mutations are supported set of mutations:

1. Insert 
2. Upsert
3. Update
4. Delete

The following snippet of code can be used in the upstream operator to pass to the Kudu output operator. 

~~~java
    KuduExecutionContext<TransactionPayload> context = new KuduExecutionContext<>();
    context.setPayload(payload);
    context.setMutationType(KuduMutationType.INSERT);
    outputPortForWrites.emit(context); 
~~~
The above code snippet creates a new instance of the KuduExecution context. The instance "payload" in the above snippet represents a POJO that is to be used by the Kudu output operator to persist to the Kudu table that has been set at constructions time. 

## Automatic schema detection

Note that the Kudu output operator automatically detects the kudu table schema and aligns the POJO payload filed names to the table column names. The default behavior of the operator is to perform a case insensitive alignment of the POJO field names to the Kudu table columns. 

## Overriding POJO mappings to column names 

It is not always possible to align the POJO field names to the Kudu table column names. It is possible to override the POJO field name to the Kudu table column name by extending the BaseKuduOutput Operator. 

~~~java
public class TransactionsTableKuduOutputOperator extends BaseKuduOutputOperator {
.....
}
~~~

In the child class, override the following method to override the mapping of POJO field name to the Kudu table name.

~~~java
  @Override
  protected Map<String, String> getOverridingColumnNameMap()
  {
    Map<String,String> overridingColumns = new HashMap<>();
    overridingColumns.put("transactionAmount","transaction_amnt");
    return overridingColumns;
  }
~~~
In the above code snippet, we are mapping the Kudu table column "transaction_amnt" to the POJO field name called transactionAmount.
Note that we can effectively use this to map any POJO field name to any of the kudu column names. 

## Exactly once semantics

Implementing exactly once semantics in Kudu is a tricky issue as Kudu does not provide a transaction construct. This prevents in implementing patterns for "exactly once" semantics like other operators to commit tuples at the end of the window boundary. 

To achieve exactly once semantics, the developer can override the default implementation of the method isEligibleForPassivationInReconcilingWindow(..). The following code snippet depicts a simple way to 

~~~java

  @Override
  protected boolean isEligibleForPassivationInReconcilingWindow(KuduExecutionContext executionContext,
                                                                long reconcilingWindowId)
  {
    if (tableForReads == null) {
      initConnectionForReads();
    }
    KuduScanner.KuduScannerBuilder scannerBuilder = apexKuduConnectionForReads.getKuduClient()
                                                    .newScannerBuilder(tableForReads);
    KuduExecutionContext<TransactionPayload> executionContextForThisTuple =
      (KuduExecutionContext<TransactionPayload>) executionContext;
    KuduScanner scannerForThisRead = scannerBuilder
                                      .limit(1)                                         .addPredicate(KuduPredicate.newComparisonPredicate(transactionidCol,
                                       KuduPredicate.ComparisonOp.EQUAL,                                       executionContextForThisTuple.getPayload().getTransactionId()))                                      .addPredicate(KuduPredicate.newComparisonPredicate(timestampCol,
                                        KuduPredicate.ComparisonOp.EQUAL,
                                        executionContextForThisTuple.getPayload().getTimestamp()))
                                      .build();
    if (scannerForThisRead.hasMoreRows()) {
      return false;
    } else {
      return true;
    }
  }
~~~

## Setting the timestamps for write resolution 


## Do not write columns for lock free implementations



## Metric generated by the output operator
The Kudu output operator supports the following metrics as tuples are processed: 

- 


