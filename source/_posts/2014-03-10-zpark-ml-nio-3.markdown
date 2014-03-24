---
layout: post
title: "ZPark-Ztream II (Part 3/3): Fancy Spark Streamed Machine-Learning & new Scalaz-Stream NIO API"
date: 2014-03-10 19:19
comments: true
external-url:
categories: [scala,spark,scalaz-stream,NIO,machine learning,stream,map reduce,ML,recommendation]
keywords: scala,spark,scalaz-stream,NIO,machine learning,stream,map reduce,ML,recommendation
---

## Synopsis

The code & sample apps can be found on [Github](https://github.com/mandubian/zpark-ztream)

> [Zpark-Zstream I article](/2014-02-13-zpark) was a PoC trying to use [Scalaz-Stream](https://github.com/scalaz/scalaz-stream) instead of DStream with [Spark-Streaming](https://spark.incubator.apache.org/). I had deliberately decided not to deal with fault-tolerance & stream graph persistence to keep simple but without it, it was quite useless for real application...

<div class="well">
<p>Here is a tryptic of articles trying to do something <i>concrete</i> with <a href="https://github.com/scalaz/scalaz-stream">Scalaz-Stream</a> and <a href="https://spark.incubator.apache.org/">Spark</a>.</p>
<p>So, what do I want? I wantttttttt <i>a shrewburyyyyyy</i> and to do the following:</p>
<ol>
<li>Plug Scalaz-Stream Process[Task, T] on Spark DStream[T] (Part 1)</li>
<li>Build DStream using brand new Scalaz-Stream NIO API (client/server) (Part 2)</li>
<li><b>Train Spark-ML <i>recommendation-like</i> model using NIO client/server (Part 3)</b></li>
<li><b>Stream data from multiple NIO clients to the previous trained ML model mixing all together (Part 3)</b></li>
</ol>
</div>

# [Part 3/3] Fancy Spark Machine Learning with NIO client/server & DStream...

<br/>

> Let's remind that I'm not an expert in ML but more a student. So if I tell or do stupid ML things, be indulgent ;)


Here is what I propose:

- Train a collaborative filtering rating model for a recommendation system (as explained in Spark doc [there](https://spark.incubator.apache.org/docs/0.9.0/mllib-guide.html#collaborative-filtering-1)) using a first NIO server and a client as presented in part 2.

- When model is trained, create a second server that will accept client connections to receive data.

- Stream/merge all received data into one single stream, dstreamize it and perform streamed predictions using previous model.

## Train collaborative filtering model

### Training client

As explained in Spark doc about collaborative filtering, we first need some data to train the model. I want to send those data using a NIO client.

Here is a function doing this:

{% codeblock lang:scala %}
//////////////////////////////////////////////////
// TRAINING CLIENT
def trainingClient(addr: InetSocketAddress): Process[Task, Bytes] = {

  // naturally you could provide much more data
  val basicTrainingData = Seq(
    // user ID, item ID, Rating
    "1,1,5.0",
    "1,2,1.0",
    "1,3,5.0",
    "1,4,1.0",
    "2,4,1.0",
    "2,2,5.0"
  )
  val trainingProcess = 
    Process.emitAll(basicTrainingData map (s => Bytes.of((s+"\n").getBytes)))

  // sendAndCheckSize is a special client sending all data emitted 
  // by the process and verifying the server received all data 
  // by acknowledging all data size
  val client = NioClient.sendAndCheckSize(addr, trainingProcess)

  client
}
{% endcodeblock %}

<br/>
### Training server

Now we need the training NIO server waiting for the training client to connect and piping the received data to the model.

Here is a useful function to help creating a server as described in previous article part:

{% codeblock lang:scala %}
def server(addr: InetSocketAddress): (Process[Task, Bytes], Signal[Boolean]) = {

  val stop = async.signal[Boolean]
  stop.set(false).run

  // this is a server that is controlled by a stop signal
  // and that acknowledges all received data by their size
  val server =
    ( stop.discrete wye NioServer.ackSize(addr) )(wye.interrupt)

  // returns a stream of received data & a signal to stop server
  (server, stop)
}
{% endcodeblock %}

We can create the training server with it:

{% codeblock lang:scala %}
val trainingAddr = NioUtils.localAddress(11100)
//////////////////////////////////////////////////
// TRAINING SERVER
val (trainingServer, trainingStop) = server(trainingAddr)

{% endcodeblock %}

`trainingServer` is a `Process[Task, Bytes]` streaming the training data received from training client. We are going to train the rating model with them. 

<br/>
### Training model

To train a model, we can use the following API:

{% codeblock lang:scala %}
// the rating with user ID, product ID & rating
case class Rating(val user: Int, val product: Int, val rating: Double)

// A RDD of ratings
val ratings: RDD[Rating] = ...

// train the model with it
val model: MatrixFactorizationModel = ALS.train(ratings, 1, 20, 0.01)
{% endcodeblock %}

### Building `RDD[Rating]` from server stream

Imagine that we have a continuous flow of training data that can be very long.

We want to train the model with just a slice of this flow. To do this, we can:

- `dstreamize` the server output stream
- run the dstream for some time
- retrieve the `RDD`s received during this time
- union all of those `RDD`s
- push them to the model

Here is the whole code with previous client:

{% codeblock lang:scala %}
val trainingAddr = NioUtils.localAddress(11100)

//////////////////////////////////////////////////
// TRAINING SERVER
val (trainingServer, trainingStop) = server(trainingAddr)

//////////////////////////////////////////////////
// TRAINING CLIENT
val tclient = trainingClient(trainingAddr)

//////////////////////////////////////////////////
// DStreamize server received data
val (trainingServerSink, trainingDstream) = dstreamize(
  trainingServer
      // converts bytes to String (doesn't care about encoding, it shall be UTF8)
      .map  ( bytes => new String(bytes.toArray) )
      // rechunk received strings based on a separator \n 
      // to keep only triplets: "USER_ID,PROD_ID,RATING"
      .pipe (NioUtils.rechunk { s:String => (s.split("\n").toVector, s.last == '\n') } )
  , ssc
)

//////////////////////////////////////////////////
// Prepare dstream output 
// (here we print to know what has been received)
trainingDstream.print()

//////////////////////////////////////////////////
// RUN

// Note the time before
val before = new Time(System.currentTimeMillis)

// Start SSC
ssc.start()

// Launch server
trainingServerSink.run.runAsync( _ => () )

// Sleeps a bit to let server listen
Thread.sleep(300)

// Launches client and awaits until it ends
tclient.run.run

// Stop server
trainingStop.set(true).run

// Note the time after
val after = new Time(System.currentTimeMillis)

// retrieves all dstreamized RDD during this period
val rdds = trainingDstream.slice(
  before.floor(Milliseconds(1000)), after.floor(Milliseconds(1000))
)

// unions them (this operation can be expensive)
val union: RDD[String] = new UnionRDD(ssc.sparkContext, rdds)

// converts "USER_ID,PROD_ID,RATING" triplets into Ratings
val ratings = union map { e =>
  e.split(',') match {
    case Array(user, item, rate) => Rating(user.toInt, item.toInt, rate.toDouble)
  }
}

// finally train the model with it
val model = ALS.train(ratings, 1, 20, 0.01)

// Predict
println("Prediction(1,3)=" + model.predict(1, 3))

//////////////////////////////////////////////////
// Stop SSC
ssc.stop()

{% endcodeblock %}

### Run it

{% codeblock lang:scala %}
-------------------------------------------
Time: 1395079621000 ms
-------------------------------------------
1,1,5.0
1,2,1.0
1,3,5.0
1,4,1.0
2,4,1.0
2,2,5.0

-------------------------------------------
Time: 1395079622000 ms
-------------------------------------------

-------------------------------------------
Time: 1395079623000 ms
-------------------------------------------

-------------------------------------------
Time: 1395079624000 ms
-------------------------------------------

-------------------------------------------
Time: 1395079625000 ms
-------------------------------------------

Prediction(1,3)=4.94897842056338
{% endcodeblock %}

>Fantastic, we have trained our model in a very fancy way, haven't we?
>
>Personally, I find it interesting that we can take advantage of both APIs...

<br/>
<br/>
## Predict Ratings

Now that we have a trained model, we can create a new server to receive data from clients for rating prediction.

<br/>
### Prediction client

Firstly, let's generate some random data to send for prediction.

{% codeblock lang:scala %}
//////////////////////////////////////////////////
// PREDICTION CLIENT
def predictionClient(addr: InetSocketAddress): Process[Task, Bytes] = {

  // PREDICTION DATA
  def rndData = 
    // userID
    (Math.abs(scala.util.Random.nextInt) % 4 + 1).toString +
    // productID
    (Math.abs(scala.util.Random.nextInt) % 4 + 1).toString +
    "\n"

  val rndDataProcess = Process.eval(Task.delay{ rndData }).repeat

  // a 1000 elements process emitting every 10ms
  val predictDataProcess =
    (Process.awakeEvery(10 milliseconds) zipWith rndDataProcess){ (_, s) => Bytes.of(s.getBytes) }
      .take(1000)

  val client = NioClient.sendAndCheckSize(addr, predictDataProcess)

  client
}
{% endcodeblock %}

### Prediction server

{% codeblock lang:scala %}
val predictAddr = NioUtils.localAddress(11101)
//////////////////////////////////////////////////
// PREDICTION SERVER
val (predictServer, predictStop) = server(predictAddr)
{% endcodeblock %}


### Prediction Stream

`predictServer` is the stream of data to predict. Let's stream it to the model by `dstreamizing` it and transforming all built `RDD`s by passing them through model

{% codeblock lang:scala %}
//////////////////////////////////////////////////
// DStreamize server
val (predictServerSink, predictDstream) = dstreamize(
  predictServer
      // converts bytes to String (doesn't care about encoding, it shall be UTF8)
      .map  ( bytes => new String(bytes.toArray) )
      // rechunk received strings based on a separator \n
      .pipe (NioUtils.rechunk { s:String => (s.split("\n").toVector, s.last == '\n') } )
  , ssc
)

//////////////////////////////////////////////////
// pipe dstreamed RDD to prediction model
// and print result
predictDstream map { _.split(',') match {
  // converts to integers required by the model (USER_ID, PRODUCT_ID)
  case Array(user, item) => (user.toInt, item.toInt)
}} transform { rdd =>
  // prediction happens here
  model.predict(rdd)
} print()
{% endcodeblock %}

### Running all in same `StreamingContext`

I've discovered a problem here because the recommendation model is built in a `StreamingContext` and uses `RDD`s built in it. So you must use the same `StreamingContext` for prediction. So I must build my training dstreamized client/server & prediction dstreamized client/server in the same context and thus I must schedule both things before starting this context.

Yet the prediction model is built from training data received after starting the context so it's not known before... So it's very painful and I decided to be nasty and consider the model as a variable that will be set later. For this, I used a horrible `SyncVar` to set the prediction model when it's ready... _Sorry about that but I need to study more about this issue to see if I can find better solutions because I'm not satisfied about it at all..._

So here is the whole training/predicting painful code:

{% codeblock lang:scala %}
//////////////////////////////////////////////////
// TRAINING

val trainingAddr = NioUtils.localAddress(11100)

// TRAINING SERVER
val (trainingServer, trainingStop) = server(trainingAddr)

// TRAINING CLIENT
val tclient = trainingClient(trainingAddr)

// DStreamize server
val (trainingServerSink, trainingDstream) = dstreamize(
  trainingServer
      // converts bytes to String (doesn't care about encoding, it shall be UTF8)
      .map  ( bytes => new String(bytes.toArray) )
      // rechunk received strings based on a separator \n
      .pipe (NioUtils.rechunk { s:String => (s.split("\n").toVector, s.last == '\n') } )
  , ssc
)

// THE HORRIBLE SYNCVAR CLUDGE (could have used atomic but not better IMHO)
var model = new SyncVar[org.apache.spark.mllib.recommendation.MatrixFactorizationModel]
// THE HORRIBLE SYNCVAR CLUDGE (could have used atomic but not better IMHO)


//////////////////////////////////////////////////
// PREDICTING
val predictAddr = NioUtils.localAddress(11101)

// PREDICTION SERVER
val (predictServer, predictStop) = server(predictAddr)

// PREDICTION CLIENT
val pClient = predictionClient(predictAddr)

// DStreamize server
val (predictServerSink, predictDstream) = dstreamize(
  predictServer
      // converts bytes to String (doesn't care about encoding, it shall be UTF8)
      .map  ( bytes => new String(bytes.toArray) )
      // rechunk received strings based on a separator \n
      .pipe ( NioUtils.rechunk { s:String => (s.split("\n").toVector, s.last == '\n') } )
  , ssc
)

// Piping received data to the model
predictDstream.map {
  _.split(',') match {
    case Array(user, item) => (user.toInt, item.toInt)
  }
}.transform { rdd =>
  // USE THE HORRIBLE SYNCVAR
  model.get.predict(rdd)
}.print()

//////////////////////////////////////////////////
// RUN ALL
val before = new Time(System.currentTimeMillis)

// Start SSC
ssc.start()

// Launch training server
trainingServerSink.run.runAsync( _ => () )

// Sleeps a bit to let server listen
Thread.sleep(300)

// Launch training client
tclient.run.run

// Await SSC termination a bit
ssc.awaitTermination(1000)
// Stop training server
trainingStop.set(true).run
val after = new Time(System.currentTimeMillis)

val rdds = trainingDstream.slice(before.floor(Milliseconds(1000)), after.floor(Milliseconds(1000)))
val union: RDD[String] = new UnionRDD(ssc.sparkContext, rdds)

val ratings = union map {
  _.split(',') match {
    case Array(user, item, rate) => Rating(user.toInt, item.toInt, rate.toDouble)
  }
}

// SET THE HORRIBLE SYNCVAR
model.set(ALS.train(ratings, 1, 20, 0.01))

println("**** Model Trained -> Prediction(1,3)=" + model.get.predict(1, 3))

// Launch prediction server
predictServerSink.run.runAsync( _ => () )

// Sleeps a bit to let server listen
Thread.sleep(300)

// Launch prediction client
pClient.run.run

// Await SSC termination a bit
ssc.awaitTermination(1000)
// Stop server
predictStop.set(true).run

{% endcodeblock %}

## Run it...

{% codeblock lang:scala %}
-------------------------------------------
Time: 1395144379000 ms
-------------------------------------------
1,1,5.0
1,2,1.0
1,3,5.0
1,4,1.0
2,4,1.0
2,2,5.0

**** Model Trained -> Prediction(1,3)=4.919459410565401

...

-------------------------------------------
Time: 1395144384000 ms
-------------------------------------------
----------------

-------------------------------------------
Time: 1395144385000 ms
-------------------------------------------
Rating(1,1,4.919459410565401)
Rating(1,1,4.919459410565401)
Rating(1,1,4.919459410565401)
Rating(1,1,4.919459410565401)
Rating(1,2,1.631952450379809)
Rating(1,3,4.919459410565401)
Rating(1,3,4.919459410565401)

-------------------------------------------
Time: 1395144386000 ms
-------------------------------------------
Rating(1,1,4.919459410565401)
Rating(1,1,4.919459410565401)
Rating(1,3,4.919459410565401)
Rating(1,3,4.919459410565401)
Rating(1,3,4.919459410565401)
Rating(1,4,0.40813133837755494)
Rating(1,4,0.40813133837755494)

...
{% endcodeblock %}

<br/>
<br/>
## Final conclusion

3 long articles to end in training a poor recommendation system with 2 clients/servers... A bit bloated isn't it? :)

Anyway, I hope I printed in your brain a few ideas, concepts about spark & scalaz-stream and if I've reached this target, it's already enough!

Yet, I'm not satisfied about a few things:

- Training a model and using it in the same `StreamingContext` is still clumsy and I must say that calling `model.predict` from a `map` function in a `DStream` might not be so good in a cluster environment. I haven't been digging this code enough to have a clear mind on it.
- I tried using multiple clients for prediction (like 100 in parallel) and it works quite well but I have encountered problems ending both my clients/servers and the streaming context and I often end into having zombies SBT process that I can't kill until reboot (some threads remain RUNNING while other AWAITS and sockets aren't released... resources issues...). Closing cleanly all of these tools creating threads & more after intensive work isn't yet good.

**But, I'm satisfied globally:**

- **Piping a scalaz-stream `Process` into a spark `DStream` works quite well and might be interesting after all.**
- **The new scalaz-stream NIO API considering clients & servers as pure streams of data gave me so many ideas that my free-time has suddenly been frightened and went away.**

<br/>
<br/>
> [GO TO PART2](/2014/03/09/zpark-ml-nio-2/) <----------------------------------------------------------------------------------------------------

Have a look at the code on [Github](https://github.com/mandubian/zpark-ztream).

Have distributed & resilient yet continuous fun!

<br/>
<br/>
<br/>
<br/>

