---
published: false
---

# Using the Kudu output operator in Apex

Malhar release 3.8.0 brings in support for writing streams into Kudu as the store.This blog post describes the various usage patterns that are supported by Kudu output operator on the Apex platform.

The following fictitious use case is used to describe the features of the Kudu output operator. The business use case requires processing data from a kafka topic and persisting the data into a kudu store. Each message from the kafka topic is to be persisted into two tables in kudu. 

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

The following image gives a pictorial representation of the application we would like to build:
![alt text](https://github.com/ananthc/sampleapps/blob/master/apache-apex/SimplekuduoutputApp/src/test/resources/Design-of-the-app.png  "Kudu output operator application design")

## Design of the output operator

Kudu output operator expects the upstream operator to send the tuple as an instance of ExecutionContext. The ExecutionContext instance contains the payload/POJO class that is to be used to write to the Kudu table. Apart from this, the ExecutionContext also specifies the type of mutation that needs to be performed on the kudu store for that payload/POJO instance. This allows a single instance of the kudu output operator to perform all models of a mutation in a single apex application.

The following sections describe the features of the Kudu output operator using various use cases.

## Creating an instance of the Kudu Output operator
The simplest way to create an instance of the Kudu Output operator is by adding a file with the name kuduoutputoperator.properties in the application classpath. This file follows the simple java property file conventions. The following are the main keys that are supported and mandatory. 

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
The upstream operator can specify the "mutation type" that needs to be used to perform the mutation. The following types of mutations are supported set of mutations:

1. Insert 
2. Upsert
3. Update
4. Delete

The following snippet of code can be used in the **upstream operator** to pass to the Kudu output operator. 

~~~java
    KuduExecutionContext<TransactionPayload> context = new KuduExecutionContext<>();
    context.setPayload(payload);
    context.setMutationType(KuduMutationType.INSERT);
    outputPortForWrites.emit(context); 
~~~
The above code snippet creates a new instance of the KuduExecution context. The instance "payload" in the above snippet represents a POJO that is to be used by the Kudu output operator to persist to the Kudu table. 

## Automatic schema detection

Note that the Kudu output operator automatically detects the kudu table schema and aligns the POJO payload field names to the table column names. The default behavior of the operator is to perform a case insensitive alignment of the POJO field names to the Kudu table columns. 

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

To achieve exactly once semantics, the developer can override the default implementation of the method isEligibleForPassivationInReconcilingWindow(..). The following code snippet depicts a simple approach to do so.

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
                                      .limit(1)
                                      .addPredicate(KuduPredicate.newComparisonPredicate(transactionidCol,
                                       KuduPredicate.ComparisonOp.EQUAL,                                       executionContextForThisTuple.getPayload().getTransactionId()))
                                       .addPredicate(KuduPredicate.newComparisonPredicate(timestampCol,
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

In this example, the class TransactionsTableKuduOutputOperator extends the BaseKuduOutputOperator. The class is overriding the above method to achieve exactly once semantics. The base implementation ensures that the above method is called when the application is resuming from a crash. The method returns false if the mutation is not allowed to execute in this "reconciling" phase. Since the business logic decides the "exactly once" semantics, overriding this method helps to achieve the required outcome even though Kudu does not support transactions.

Note that the above method is called at the maximum for a single window i.e. the window in which a crash might have occured and a subsequent restart of the application starts the operator in a  reconciling mode. The processing resumes to normal mode after this reconciling window from the subsequent window. We are thus using the logic in this method to check for duplicate writes to the table. Ideally this could have been solved using an "upsert" mutation type. But "upsert" mutation might not always work when there is a use case of another operator doing an "update" on the same row.

## Setting the timestamps for write resolution 
Consider the use case wherein we are maintaning the last time a device was seen in the devices table.The table gives the last time stamp the device was seen. However there is a caveat that not all tuples arriving in the Kafka topic are time ordered. Hence there is a probability of erraneous timestamp showing in the devices table due out of sequence events. 

Kudu output operator allows the client side timestamps to be propagated to the Kudu server where the mutation is executed. This allows for out of sequence data tuples to be ordered on the server side. The following snippet of code in the upstream operator shows how this can be done.

~~~java
    KuduExecutionContext<TransactionPayload> context = new KuduExecutionContext<>();
    context.setPayload(payload);
    context.setMutationType(KuduMutationType.UPSERT);
    context.setPropagatedTimestamp(payload.getTimestamp());
~~~
The last line as implemented in the upstream operator allows the timestamp to be propagated to the Kudu server. In this case, we are setting the timestamp as given in the incoming tuple from the Kafka topic.

## Do not write columns for lock free implementations

There are situations wherein we would like to skip writing a few columns for a given row. For example, in the transactions table, we have the use case wherein the is_stepup column is coming as a separate tuple. However this tuple is coming in as a separate transaction from the upstream system. This upstream system which is performing the logic for step up is writing the original transaction id and whether a step up has been initiated. It however does not have the transaction amount column which was part of the original transaction and already pumped in to the kafka topic. While performing the write, we would like to keep the remaining rows constant and only write the "is_stepup" column. The following snippet shows how we can ignore a certain collection of columns when performing such writes. 

~~~java
    Set<String> doNotWriteColumnsForTransactionsTable = new HashSet<>();
    doNotWriteColumnsForTransactionsTable.add("transaction_amnt");
    
    ...
    ...
    
    KuduExecutionContext<TransactionPayload> context = new KuduExecutionContext<>();
    context.setPayload(payload);
    context.setMutationType(KuduMutationType.UPDATE);
    context.setDoNotWriteColumns(doNotWriteColumnsForTransactionsTable);
~~~

In the above example we are ensuring that the transaction amount column is not over written when the step up transaction is flowing through and is getting processed. 

## Metric generated by the output operator
No operator is good if it does not provide an insight into its operational metrics.The Kudu output operator supports the following metrics as tuples are processed: 

There are two class of metrics that are exposed by the Kudu output operator.
- One class of metrics which summarize the metrics for the current window 
- One class of metrics which persists the metrics across application launches ( provided the application name does not change)

For both of these classes of metrics, the following metrics are captured.

1. Number of inserts 
2. Number of upserts
3. Number of updates
4. Number of deletes
5. Number of bytes written
6. Number of write operations
7. Number of write RPCs
8. Number of RPC errors 
9. Number of Operational errors

Note that the Kudu output operator uses async threads while performing the writes to the Kudu operator. Hence there is a subtle difference between number of write operations and number of write RPCs.

## Conclusion
The Kudu output operator integrated with Apex thus allows for very low latency writes to a distributed kudu store at very high throughputs. If there are SQL engines like Impala using the Kudu store, Apex enables for a sub-second writes to Kudu as an SQL enabled store. This pattern improves upon the drawbacks of parquet styled batch write file patterns for query engines like Impala. The sub-second writes also provides for a new pattern wherein we could use Kudu as a deduping store.

The following metrics were observed on a cluster setup and configured on a **single host**. The single host was running **all** of the stack outlined below

- CDH 5.10
- Apex 3.6.0
- Confluent Kafka 3.2.1
- Kudu 1.3

The cluster was built on a single host using the approach outlined here : https://ananthc.github.io/setting-up-clusters-on-a-single-home-office-machine 

- Approx 1300 transactions per second distributed over two kudu tables from a single apex application
- Average latency of 5 ms 

The single host running the **entire** stack and the test application shows the following metrics of the application
![alt text](https://github.com/ananthc/sampleapps/blob/master/apache-apex/SimplekuduoutputApp/src/test/resources/Running-application.png  "Running metrics Kudu output operator application")

The Kudu output operator shows the following latencies 
![alt text](https://github.com/ananthc/sampleapps/blob/master/apache-apex/SimplekuduoutputApp/src/test/resources/Kudu-output-operator-metrics.png "Kudu output operator metrics")
