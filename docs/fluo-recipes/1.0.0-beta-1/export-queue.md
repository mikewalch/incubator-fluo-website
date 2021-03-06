---
layout: recipes-doc
title: Export Queue Recipe
version: 1.0.0-beta-1
---

## Background

Fluo is not suited for servicing low latency queries for two reasons.  The
first reason is that the implementation of transactions are designed for
throughput.  To get throughput, transactions recover lazily from failures and
may wait on another transaction that is writing.  Both of these design decisions
can lead to delays for an individual transaction, but do not negatively impact
throughput.   The second reason is that Fluo observers executing transactions
will likely cause a large number of random accesses.  This could lead to high
response time variability for an individual random access.  This variability
would not impede throughput but would impede the goal of latency.

One way to make data transformed by Fluo available for low latency queries is
to export that data to another system.  For example Fluo could be running
cluster A, continually transforming a large data set, and exporting data to
Accumulo tables on cluster B.  The tables on cluster B would service user
queries.  This recipe could be used to export to systems other than Accumulo,
like Redis, Elasticsearch, MySQL, etc.

Exporting data from Fluo is easy to get wrong which is why this recipe exists.
To understand what can go wrong consider the following example observer
transaction.

```java
public class MyObserver extends AbstractObserver {

    private static final TYPEL = new TypeLayer(new StringEncoder());

    //reperesents a Query system extrnal to Fluo that is updated by Fluo
    QuerySystem querySystem;

    @Override
    public void process(TransactionBase tx, Bytes row, Column col) {

        TypedTransactionBase ttx = TYPEL.wrap(tx);
        int oldCount = ttx.get().row(row).fam("meta").qual("counter1").toInteger(0);
        int numUpdates = ttx.get().row(row).fam("meta").qual("numUpdates").toInteger(0);
        int newCount = oldCount + numUpdates;
        ttx.mutate().row(row).fam("meta").qual("counter1").set(newCount);
        ttx.mutate().row(row).fam("meta").qual("numUpdates").set(0);

        //Build an inverted index in the query system, based on count from the
        //meta:counter1 column in fluo.  Do this by creating rows for the
        //external query system based on the count.        
        String oldCountRow = String.format("%06d", oldCount);
        String newCountRow = String.format("%06d", newCount);
        
        //add a new entry to the inverted index
        querySystem.insertRow(newCountRow, row);
        //remove the old entry from the inverted index
        querySystem.deleteRow(oldCountRow, row);
    }
}
```

The above example would keep the external index up to date beautifully as long
as the following conditions are met.

  * Threads executing transactions always complete successfully.
  * Only a single thread ever responds to a notification.

However these conditions are not guaranteed by Fluo.  Multiple threads may
attempt to process a notification concurrently (only one may succeed).  Also at
any point in time a transaction may fail (for example the computer executing it
may reboot).   Both of these problems will occur and will lead to corruption of
the external index in the example.  The inverted index and Fluo  will become
inconsistent.  The inverted index will end up with multiple entries (that are
never cleaned up) for single entity even though the intent is to only have one.

The root of the problem in the example above is that its exporting uncommitted
data.  There is no guarantee that setting the column `<row>:meta:counter1` to
`newCount` will succeed until the transaction is successfully committed.
However, `newCountRow` is derived from `newCount` and written to the external query
system before the transaction is committed (Note : for observers, the
transaction is committed by the framework after `process(...)` is called).  So
if the transaction fails, the next time it runs it could compute a completely
different value for `newCountRow` (and it would not delete what was written by the
failed transaction).  

## Solution 

The simple solution to the problem of exporting uncommitted data is to only
export committed data.  There are multiple ways to accomplish this.  This
recipe offers a reusable implementation of one method.  This recipe has the
following elements:

 * An export queue that transactions can add key/values to.  Only if the transaction commits successfully will the key/value end up in the queue.  A Fluo application can have multiple export queues, each one must have a unique id.
 * When a key/value is added to the export queue, its given a sequence number.  This sequence number is based on the transactions start timestamp.
 * Each export queue is configured with an observer that processes key/values that were successfully committed to the queue.
 * When key/values in an export queue are processed, they are deleted so the export queue does not keep any long term data.
 * Key/values in an export queue are placed in buckets.  This is done so that all of the updates in a bucket can be processed in a single transaction.  This allows an efficient implementation of this recipe in Fluo.  It can also lead to efficiency in a system being exported to, if the system can benefit from batching updates.  The number of buckets in an export queue is configurable.

There are three requirements for using this recipe :

 * Must configure export queues before initializing a Fluo application.
 * Transactions adding to an export queue must get an instance of the queue using its unique QID.
 * Must implement a class that extends [Exporter][1] in order to process exports.

## Schema

Each export queue stores its data in the Fluo table in a contiguous row range.
This row range is defined by using the export queue id as a row prefix for all
data in the export queue.  So the row range defined by the export queue id
should not be used by anything else.

All data stored in an export queue is [transient](../transient/). When an export
queue is configured, it will recommend split points using the [table
optimization process](../table-optimization/).

## Example Use

This example will show how to build an inverted index in an external
query system using an export queue.  The class below is simple POJO to hold the
count update, this will be used as the value for the export queue.

```java
class CountUpdate {
  public int oldCount;
  public int newCount;
  
  public CountUpdate(int oc, int nc) {
    this.oldCount = oc;
    this.newCount = nc;
  }
}
```

The following code shows how to configure an export queue.  This code will
modify the FluoConfiguration object with options needed for the export queue.
This FluoConfiguration object should be used to initialize the fluo
application.

```java
   FluoConfiguration fluoConfig = ...;

   //queue id "ici" means inverted count index, would probably use static final in real application
   String exportQueueID = "ici";  
   Class<CountExporter> exporterType = CountExporter.class;
   Class<Bytes> keyType = Bytes.class;
   Class<CountUpdate> valueType = CountUpdate.class;
   int numBuckets = 1009;

   ExportQueue.Options eqOptions =
        new ExportQueue.Options(exportQueueId, exporterType, keyType, valueType, numBuckets);
   ExportQueue.configure(fluoConfig, eqOptions);

   //initialize Fluo using fluoConfig
```

Below is updated version of the observer from above thats now using the export
queue.

```java
public class MyObserver extends AbstractObserver {

    private static final TYPEL = new TypeLayer(new StringEncoder());

    private ExportQueue<Bytes, CountUpdate> exportQueue;

    @Override
    public void init(Context context) throws Exception {
      exportQueue = ExportQueue.getInstance("ici", context.getAppConfiguration());
    }

    @Override
    public void process(TransactionBase tx, Bytes row, Column col) {

        TypedTransactionBase ttx = TYPEL.wrap(tx);
        int oldCount = ttx.get().row(row).fam("meta").qual("counter1").toInteger(0);
        int numUpdates = ttx.get().row(row).fam("meta").qual("numUpdates").toInteger(0);
        int newCount = oldCount + numUpdates;
        ttx.mutate().row(row).fam("meta").qual("counter1").set(newCount);
        ttx.mutate().row(row).fam("meta").qual("numUpdates").set(0);

        //Because the update to the export queue is part of the transaction,
        //either the update to meta:counter1 is made and an entry is added to
        //the export queue or neither happens.
        exportQueue.add(tx, row, new CountUpdate(oldCount, newCount));
    }
}
```

The observer setup for the export queue will call the `processExports()` method
on the class below to process entries in the export queue.  It possible the
call to `processExports()` can fail part way through and/or be called multiple
times.  In the case of failures the exporter will eventually be called again
with the exact same data.  The possibility of the same export entry being
processed on multiple computer at different times can cause exports to arrive
out of order.  The system receiving exports has to be resilient to data
arriving out of order and multiple times.  The purpose of the sequence number
is to help systems receiving data from Fluo process out of order and redundant
data.

```java
  public class CountExporter extends Exporter<Bytes, CountUpdate> {
    //represents the external query system we want to update from Fluo
    QuerySystem querySystem;

    @Override
    protected void processExports(Iterator<SequencedExport<Bytes, CountUpdate>> exportIterator) {
      BatchUpdater batchUpdater = querySystem.getBatchUpdater();
      while(exportIterator.hasNext()){
        SequencedExport<Bytes, CountUpdate> exportEntry =  exportItertor.next();
        Bytes row =  exportEntry.getKey();
        UpdateCount uc = exportEntry.getValue();
        long seqNum = exportEntry.getSequence();

        String oldCountRow = String.format("%06d", uc.oldCount);
        String newCountRow = String.format("%06d", uc.newCount);

        //add a new entry to the inverted index
        batchUpdater.insertRow(newCountRow, row, seqNum);
        //remove the old entry from the inverted index
        batchUpdater.deleteRow(oldCountRow, row, seqNum);
      }

      //flush all of the updates to the external query system
      batchUpdater.close();
    }
  }
```

## Concurrency

Additions to the export queue will never collide.  If two transactions add the
same key at around the same time and successfully commit, then two entries with
different sequence numbers will always be added to the queue.  The sequence
number is based on the start timestamp of the transactions.

If the key used to add items to the export queue is deterministically derived
from something the transaction is writing to, then that will cause a collision.
For example consider the following interleaving of two transactions adding to
the same export queue in a manner that will collide. Note, TH1 is shorthand for
thread 1, ek() is a function the creates the export key, and ev() is a function
that creates the export value.

 1. TH1 : key1 = ek(`row1`,`fam1:qual1`)
 1. TH1 : val1 = ev(tx1.get(`row1`,`fam1:qual1`), tx1.get(`rowA`,`fam1:qual2`))
 1. TH1 : exportQueueA.add(tx1, key1, val1)
 1. TH2 : key2 = ek(`row1`,`fam1:qual1`)
 1. TH2 : val2 = ev(tx2.get(`row1`,`fam1:qual1`), tx2.get(`rowB`,`fam1:qual2`))
 1. TH2 : exportQueueA.add(tx2, key2, val2)
 1. TH1 : tx1.set(`row1`,`fam1:qual1`, val1)
 1. TH2 : tx2.set(`row1`,`fam1:qual1`, val2)

In the example above only one transaction will succeed because both are setting
`row1 fam1:qual1`.  Since adding to the export queue is part of the
transaction, only the transaction that succeeds will add something to the
queue.  If the funtion ek() in the example is deterministic, then both
transactions would have been trying to add the same key to the export queue.

With the above method, we know that transactions adding entries to the queue for
the same key must have executed [serially][2]. Knowing that transactions which
added the same key did not overlap in time makes reasoning about those export
entries very simple.

The example below is a slight modification of the example above.  In this
example both transactions will successfully add entries to the queue using the
same key.  Both transactions succeed because they are writing to different
cells (`rowB fam1:qual2` and `rowA fam1:qual2`).  This approach makes it more
difficult to reason about export entries with the same key, because the
transactions adding those entries could have overlapped in time.  This is an
example of write skew mentioned in the Percolater paper.
 
 1. TH1 : key1 = ek(`row1`,`fam1:qual1`)
 1. TH1 : val1 = ev(tx1.get(`row1`,`fam1:qual1`), tx1.get(`rowA`,`fam1:qual2`))
 1. TH1 : exportQueueA.add(tx1, key1, val1)
 1. TH2 : key2 = ek(`row1`,`fam1:qual1`)
 1. TH2 : val2 = ev(tx2.get(`row1`,`fam1:qual1`), tx2.get(`rowB`,`fam1:qual2`))
 1. TH2 : exportQueueA.add(tx2, key2, val2)
 1. TH1 : tx1.set(`rowA`,`fam1:qual2`, val1)
 1. TH2 : tx2.set(`rowB`,`fam1:qual2`, val2)

[1]: /apidocs/fluo-recipes/1.0.0-beta-1/io/fluo/recipes/export/Exporter.html
[2]: https://en.wikipedia.org/wiki/Serializability

