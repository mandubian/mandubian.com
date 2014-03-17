---
layout: post
title: "ZPark-Ztream II (Part 1/3): Fancy Spark Streamed Machine-Learning & new Scalaz-Stream NIO API"
date: 2014-03-08 19:19
comments: true
external-url:
categories: [scala,spark,scalaz-stream,NIO,machine learning,stream,map reduce,ML,recommendation]
keywords: scala,spark,scalaz-stream,NIO,machine learning,stream,map reduce,ML,recommendation
---

<br/>
## Synopsis

The code & sample apps can be found on [Github](https://github.com/mandubian/zpark-ztream)

> [Zpark-Zstream I article](/2014-02-13-zpark) was a PoC trying to use [Scalaz-Stream](https://github.com/scalaz/scalaz-stream) instead of DStream with [Spark-Streaming](https://spark.incubator.apache.org/). I had deliberately decided not to deal with fault-tolerance & stream graph persistence to keep simple but without it, it was quite useless for real application...

<div class="well">
<p>Here is a tryptic of articles trying to do something <i>concrete</i> with <a href="https://github.com/scalaz/scalaz-stream">Scalaz-Stream</a> and <a href="https://spark.incubator.apache.org/">Spark</a>.</p>
<p>So, what do I want? I wantttttttt <i>a shrewburyyyyyy</i> and to do the following:</p>
<ol>
<li><b>Plug Scalaz-Stream Process[Task, T] on Spark DStream[T] (Part 1)</b></li>
<li>Build DStream using brand new Scalaz-Stream NIO API (client/server) (Part 2)</li>
<li>Train Spark-ML <i>recommendation-like</i> model using NIO client/server (Part 3)</li>
<li>Stream data from multiple NIO clients to the previous trained ML model mixing all together (Part 3)</li>
</ol>
</div>

# [Part 1/3] From Scalaz-Stream Process to Spark DStream

<br/>
<br/>
## Reminders on Process[Task, T]

Scalaz-stream `Process[Task, T]` is a stream of `T` elements that can interleave some `Task`s (representing an external something doing somewhat). `Process[Task, T]` is built as a state machine that you need to run to process all `Task` effects and emit a stream of `T`. This can manage both continuous or discrete, finite or infinite streams.

>I restricted to `Task` for the purpose of this article but it can be any `F[_]`.

<br/>
## Reminders on DStream[T]

Spark `DStream[T]` is a stream of `RDD[T]` built by discretizing a continuous stream of `T`. `RDD[T]` is a _resilient distributed dataset_ which is the ground data-structure behind Spark for distributing in-memory batch/map/reduce operations to a cluster of nodes with fault-tolerance & persistence.

In a summary, `DStream` _slices_ a continuous stream of `T` by windows of time and gathers all `T`s in the same window into one `RDD[T]`. So it discretizes the continuous stream into a stream of `RDD[T]`. Once built, those `RDD[T]`s are distributed to Spark cluster. Spark allows to perform transform/union/map/reduce/... operations on `RDD[T]`s. Therefore `DStream[T]` takes advantage if the same operations.

Spark-Streaming also persists all operations & relations between `DStream`s in a graph. Thus, in case of fault in a remote node while performing operations on `DStream`s, the whole transformation can be replayed (_it also means streamed data are also persisted_).

Finally, the resulting `DStream` obtained after map/reduce operations can be _output_ to a file, a console, a DB etc...

> Please note that `DStream[T]` is built with respect to a `StreamingContext` which manages its distribution in Spark cluster and all operations performed on it. Moreover, `DStream` map/reduce operations & output must be scheduled before starting the `StreamingContext`. It could be somewhat compared to a state machine that you build statically and run later.

<br/>
## From Process[Task, T] to RDD[T]

> You may ask why not simply build a `RDD[T]` from a `Process[Task, T]` ?

Yes sure we can do it:

{% codeblock lang:scala %}
// Initialize Spark Context
implicit scc = new SparkContext(...)

// Build a process
val p: Process[Task, T] = ...

// Run the process using `runLog` to aggregate all results
// and build a RDD using spark context parallelization
val rdd = sc.parallelize(p.runLog.run)
{% endcodeblock %}

This works but what if this `Process[Task, T]` emits huge quantity of data or is infinite?
You'll end in a `OutOfMemoryException`...

So yes you can do it but it's not so interesting. `DStream` seems more natural since it can manage stream of data as long as it can discretize it over time.

<br/>
<br/>
## From Process[Task, T] to DStream[T]

<br/>
### Pull from `Process[Task, T]`, Push to `DStream[T]` with `LocalInputDStream`

To build a `DStream[T]` from a `Process[Task, T]`, the idea is to:

- Consume/pull the `T` emitted by `Process[Task, O]`,
- Gather emitted `T` during a window of time & generate a `RDD[T]` with them,
- Inject `RDD[T]` into the `DStream[T]`,
- Go to next window of time...

Spark-Streaming library provides different helpers to create `DStream` from different sources of data like files (local/HDFS), from sockets...

The helper that seemed the most appropriate is the `NetworkInputDStream`:

- It provides a `NetworkReceiver` based on a Akka actor to which we can push streamed data.
- `NetworkReceiver` gathers streamed data over windows of time and builds a `BlockRDD[T]` for each window.
- Each `BlockRDD[T]` is registered to the global Spark `BlockManager` (responsible for data persistence).
- `BlockRDD[T]` is injected into the `DStream[T]`.

So basically, `NetworkInputDStream` builds a stream of `BlockRDD[T]`.
It's important to note that `NetworkReceiver` is also meant to be sent to remote workers so that data can be gathered on several nodes at the same time.

But in my case, the data source `Process[Task, T]` run on the Spark driver node (at least for now) so instead of `NetworkInputDStream`, a `LocalInputDStream` would be better. It would provide a `LocalReceiver` based on an actor to which we can push the data emitted by the process in an async way.

> `LocalInputDStream` doesn't exist in Spark-Streaming library (or I haven't looked well) so I've implemented it as I needed. It does exactly the same as `NetworkInputDStream` without the remoting aspect. The current code is [there](https://github.com/mandubian/zpark-ztream/blob/master/src/main/scala/LocalInputDStream.scala)...

<br/>
### `Process` vs `DStream` ?

There is a common point between `DStream` and `Process`: both are built as state machines that are passive until run.

- In the case of `Process`, it is run by playing all the `Task` effects while gathering emitted values or without taking care of them, in blocking or non-blocking mode etc...

- In the case of `DStream`, it is built and registered in the context of a `SparkStreamingContext`. Then you must also declare some outputs for the `DStream` like a simple print, an `HDFS` file output, etc... Finally you start the `SparkStreamingContext` which manages everything for you until you stop it.

So if we want to adapt a `Process[Task, T]` to a `DStream[T]`, we must perform 4 steps  (_on the Spark driver node_):

- build a `DStream[T]` using `LocalInputDStream[T]` providing a `Receiver` in which we'll be able to push asynchronously `T`.
- build a custom scalaz-stream `Sink[Task, T, Unit]` in charge of consuming all emitted data from `Process[Task, T]` and pushing them using previous `Receiver`.
- pipe the `Process[Task, T]` to this `Sink[Task, T, Unit]` & when `Process[Task, T]` has halted, stop previous `DStream[T]`: the result of this pipe operation is a `Process[Task, Unit]` which is a pure effectful process responsible for pushing `T` into the dstream without emitting anything.
- return previous `DStream[T]` and effectful consumer `Process[Task, Unit]`.

### `dstreamize` implementation

{% codeblock lang:scala %}
def dstreamize[T : ClassTag](
  p: Process[Task, T],
  ssc: StreamingContext,
  storageLevel: StorageLevel = StorageLevel.MEMORY_AND_DISK_SER_2
): (Process[Task, Unit], ZparkInputDStream[T]) = {

  // Build a custom LocalInputDStream
  val dstream = new ZparkInputDStream[T](ssc, storageLevel)

  // Build a Sink pushing into dstream receiver
  val sink = receiver2Sink[T](dstream.receiver.asInstanceOf[ZparkReceiver[T]])

  // Finally pipe the process to the sink and when finished, closes the dstream
  val consumer: Process[Task, Unit] =
    (p to sink)
    // when finished, it closes the dstream
    .append ( eval(Task.delay{ dstream.stop() }) )
    // when error, it closes the dstream
    .handle { case e: Exception =>
      println("Stopping on error "+e.getMessage)
      e.printStackTrace()
      eval(Task.delay{ dstream.stop() })
    }

  // Return the effectful consumer sink and the DStream
  (consumer, dstream)
}
{% endcodeblock %}

> Please remark that this builds a `Process[Task, Unit]` and a `DStream[T]` but nothing has happened yet in terms of data consumption & streaming. Both need to be run now.

### Use it...

{% codeblock lang:scala %}
// First create a streaming context
val ssc = new StreamingContext(clusterUrl, "SparkStreamStuff", Seconds(1))

// Create a data source sample as a process generating a natural every 50ms 
// (take 1000 elements)
val p: Process[Task, Int] = naturalsEvery(50 milliseconds).take(1000)

// Dstreamize the process in the streaming context
val (consumer, dstream) = dstreamize(p, ssc)

// Prepare the dstream operations (count) & output (print)
dstream.count().print()

// Start the streaming context
ssc.start()

// Run the consumer for its effects (consuming p and pushing into dstream)
// Note this is blocking but it could be runAsync too
consumer.run.run

// await termination of stream with a timeout
ssc.awaitTermination(1000)

// stops the streaming context
ssc.stop()
{% endcodeblock %}

>Please note that you have to:
>
>- schedule your dstream operations/output before starting the streaming context.
>- start the streaming context before running the consumer.

### Run it...

{% codeblock lang:scala %}
14/03/11 11:32:09 WARN util.Utils: Your hostname, localhost.paris.zenexity.com resolves to a loopback address: 127.0.0.1; using 10.0.24.228 instead (on interface en0)
14/03/11 11:32:09 WARN util.Utils: Set SPARK_LOCAL_IP if you need to bind to another address
14/03/11 11:32:13 WARN storage.BlockManager: Block input-0-1394533932800 already exists on this machine; not re-adding it
14/03/11 11:32:13 WARN storage.BlockManager: Block input-0-1394533933000 already exists on this machine; not re-adding it
14/03/11 11:32:13 WARN storage.BlockManager: Block input-0-1394533933200 already exists on this machine; not re-adding it
-------------------------------------------
Time: 1394533933000 ms
-------------------------------------------
0

14/03/11 11:32:13 WARN storage.BlockManager: Block input-0-1394533933600 already exists on this machine; not re-adding it
14/03/11 11:32:14 WARN storage.BlockManager: Block input-0-1394533933800 already exists on this machine; not re-adding it
-------------------------------------------
Time: 1394533934000 ms
-------------------------------------------
20

14/03/11 11:32:14 WARN storage.BlockManager: Block input-0-1394533934000 already exists on this machine; not re-adding it
14/03/11 11:32:14 WARN storage.BlockManager: Block input-0-1394533934200 already exists on this machine; not re-adding it
14/03/11 11:32:14 WARN storage.BlockManager: Block input-0-1394533934400 already exists on this machine; not re-adding it
14/03/11 11:32:14 WARN storage.BlockManager: Block input-0-1394533934600 already exists on this machine; not re-adding it
14/03/11 11:32:15 WARN storage.BlockManager: Block input-0-1394533934800 already exists on this machine; not re-adding it
-------------------------------------------
Time: 1394533935000 ms
-------------------------------------------
20

...
{% endcodeblock %}

> Ok cool, we can see a first empty window and then windows of 1 sec counting 20 elements which is great since one element every 50ms gives 20 elements in 1sec.

<br/>
<br/>
## Part 1's conclusion

Now we can pipe a `Process[Task, T]` into a `DStream[T]`.

Please not that as we run the `Process[Task, T]` on the Spark driver node, if this node fails, there is no real way to restore lost data. Yet, `LocalInputDStream` relies on `DStreamGraph` & `BlockRDD`s which persist all DStream relations & all received blocks. Moreover, `DStream` has exactly the same problem with respect to driver node for now.

That was fun but what can we do with that?

**In [part2](./2014/03/09/zpark-ml-nio-2/), I propose to have more fun and stream data to `DStream` using the brand new Scalaz-Stream NIO API to create cool NIO client/server streams...**

<br/>
<br/>
> -------------------------------------------------------------------------------------------------------> [GO TO PART2](/2014/03/09/zpark-ml-nio-2/)

<br/>
<br/>
<br/>
<br/>

