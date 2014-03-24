---
layout: post
title: "ZPark-Ztream II (Part 2/3): Fancy Spark Streamed Machine-Learning & new Scalaz-Stream NIO API"
date: 2014-03-09 19:19
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
<li><b>Build DStream using brand new Scalaz-Stream NIO API (client/server) (Part 2)</b></li>
<li>Train Spark-ML <i>recommendation-like</i> model using NIO client/server (Part 3)</li>
<li>Stream data from multiple NIO clients to the previous trained ML model mixing all together (Part 3)</li>
</ol>
</div>

<br/>
# [Part 2/3] From Scalaz-Stream NIO client & server to Spark DStream

## Scalaz-Stream NIO Client

### What is a client?

- Something sending some data `W` _(for Write)_ to a server
- Something reading some data `I` _(for Input)_ from a server

<br/>
### Client seen as `Process`

A client could be represented as:

- a `Process[Task, I]` for input channel (receiving from server)
- a `Process[Task, W]` for output channel (sending to server)

In scalaz-stream, recently a new structure has been added :

{% codeblock lang:scala %}
final case class Exchange[I, W](read: Process[Task, I], write: Sink[Task, W])
{% endcodeblock %}

Precisely what we need!

Now, let's consider that we work in NIO mode with everything non-blocking, asynchronous etc...

In this context, a client can be seen as something generating _soon or later one (or more)_ `Exchange[I, W]` i.e :

{% codeblock lang:scala %}
Client[I, W] === Process[Task, Exchange[I, W]]
{% endcodeblock %}

> In the case of a pure TCP client, `I` and `W` are often `Bytes`.

### Creating a client

Scalaz-Stream now provides a helper to create a TCP binary NIO client:

{% codeblock lang:scala %}
// the address of the server to connect to
val address: InetSocketAddress = new InetSocketAddress("xxx.yyy.zzz.ttt", port)

// create a client
val client: Process[Task, Exchange[Bytes, Bytes]] = nio.connect(address)

client map { ex: Exchange[Bytes, Bytes] =>
  // read data sent by server in ex.read
  ???
  // write data to the server with ex.write
  ???
}
{% endcodeblock %}

<br/>
### Plug your own data source on `Exchange`

To plug your own data source to write to server, Scalaz-Stream provides 1 more API:

{% codeblock lang:scala %}
case class Exchange[I, W](read: Process[Task, I], write: Sink[Task, W]) {
  /**
   * Runs supplied Process of `W` values by sending them to remote system.
   * Any replies from remote system are received as `I` values of the resulting process.
   */
  def run(p:Process[Task,W]): Process[Task,I]

  // the W are sent to the server and we retrieve only the received data
}
{% endcodeblock %}

With this API, we can write data to the client and output received data.

{% codeblock lang:scala %}
// some data to be sent by client
val data: Process[Task, W] = ...

// send data and retrieve only responses received by client
val output: Process[Task, I] = client flatMap { ex =>
  ex.run(data)
}

val receivedData: Seq[Bytes] = output.runLog.run
{% endcodeblock %}

Yet, in general, we need to:

- send custom data to the server
- expect its response
- do some business logic
- send more data
- etc...

So we need to be able to gather in the same piece of code received & emitted data.

<br/>
### Managing client/server business logic with `Wye`

Scalaz-stream can help us with the following API:

{% codeblock lang:scala %}
case class Exchange[I, W](read: Process[Task, I], write: Sink[Task, W]) {
...
  /**
   * Transform this Exchange to another Exchange where queueing, and transformation of this `I` and `W`
   * is controlled by supplied WyeW.
   */
  def wye(w: Wye[Task, I, W2, W \/ I2]): Exchange[I2, W2]
...

// It transforms the original Exchange[I, W] into a Exchange[I2, W2]

}
{% endcodeblock %}

> Whoaaaaa complex isn't it? Actually not so much...

`Wye` is a fantastic tool that can:

- read from a left and/or right branch (in a non-deterministic way: left or right or left+right),
- perform computation on left/right received data,
- emit data in output.

I love ASCII art schema:

{% codeblock lang:scala %}

> Wye[Task, I, I2, W]

    I(left)       I2(right)
          v       v
          |       |
          ---------
              |
 ---------------------------
|    Wye[Task, I, I2, W]    |
 ---------------------------
              |
              v
              W

{% endcodeblock %}

`\/` is ScalaZ disjunction also called `Either in the Scala world.

So `Wye[Task, I, W2, W \/ I2]` can be seen as:

{% codeblock lang:scala %}
> Wye[Task, I, W2, W \/ I2]

          I       W2
          v       v
          |       |
          ---------
              |
 ---------------------------
| Wye[Task, I, W2, W \/ I2] |
 ---------------------------
              |
          ---------
          |       |
          v       v
          W       I2

{% endcodeblock %}

#### So what does this `Exchange.wye` API do?

- It plugs the original `Exchange.write: Sink[Task, W]` to the `W` output of the `Wye[Task, I, W2, W \/ I2]` for sending data to the server.
- It plugs the `Exchange.read: Process[Task, I]` receiving data from server to the left input of the `Wye[Task, I, W2, W]`.
- The right intput `W2` branch provides a plug for an external source of data in the shape of `Process[Task, W2]`.
- The right output `I2` can be used to pipe data from the client to an external local process (like streaming out data received from the server).
- Finally it returns an `Exchange[I2, W2]`.


In a summary:

{% codeblock lang:scala %}

> (ex:Exchange[I, W]).wye( w:Wye[Task, I, W2, W \/ I2] )

        ex.read
          |
          v
          I       W2
          v       v
          |       |
          ---------
              |
 -----------------------------
| w:Wye[Task, I, W2, W \/ I2] |
 -----------------------------
              |
          ---------
          |       |
          v       v
          W       I2
          |
          v
      ex.write

======> Returns Exchange[I2, W2]

{% endcodeblock %}

> As a conclusion, `Exchange.wye` _combines_ the original `Exchange[I, W]` with your custom `Wye[Task, I, W2, W \/ I2]` which **represents the business logic of data exchange between client & server** and finally returns a `Exchange[I2, W2]` on which you can plug your own data source and retrieve output data.

<br/>
#### Implement the client with `wye/run`

{% codeblock lang:scala %}
// The source of data to send to server
val data2Send: Process[Task, Bytes] = ...

// The logic of your exchange between client & server
val clientWye: Wye[Task, Bytes, Bytes, Bytes \/ Bytes])= ...
// Scary, there are so many Bytes

val clientReceived: Process[Task, Bytes] = for {
  // client connects to the server & returns Exchange
  ex   <- nio.connect(address)

  // Exchange is customized with clientWye
  // Data are injected in it with run
  output <- ex.wye(clientWye).run(data2Send)
} yield (output)

{% endcodeblock %}

<br/>
#### Implement simple client/server business logic?

> Please note, I simply reuse the basic `echo` example provided in scalaz-stream ;)

{% codeblock lang:scala %}
def clientEcho(address: InetSocketAddress, data: Bytes): Process[Task, Bytes] = {

  // the custom Wye managing business logic
  def echoLogic: Wye[Bytes, Bytes, Bytes, Byte \/ Bytes] = {

    def go(collected: Int): WyeW[Bytes, Bytes, Bytes, Bytes] = {
      // Create a Wye that can receive on both sides
      receiveBoth {
        // Receive on left == receive from server
        case ReceiveL(rcvd) =>
          // `emitO` outputs on `I2` branch and then...
          emitO(rcvd) fby
            // if we have received everything sent, halt
            (if (collected + rcvd.size >= data.size) halt
            // else go on collecting
            else go(collected + rcvd.size))

        // Receive on right == receive on `W2` branch == your external data source
        case ReceiveR(data) => 
          // `emitW` outputs on `W` branch == sending to server
          // and loops
          emitW(data) fby go(collected)

        // When server closes
        case HaltL(rsn)     => Halt(rsn)
        // When client closes, we go on collecting echoes
        case HaltR(_)       => go(collected)
      }
    }

    // Init
    go(0)
  }

  // Finally wiring all...
  for {
    ex   <- nio.connect(address)
    rslt <- ex.wye(echoSent).run(emit(data))
  } yield {
    rslt
  }
}
{% endcodeblock %}

> This might seem hard to catch to some people because of scalaz-stream notations and wye `Left/Right/Both` or `wye.emitO/emitW`. But actually, you'll get used to it quite quickly as soon as you understand `wye`. Keep in mind that this code uses low-level scalaz-stream API without anything else and it remains pretty simple and straighforward.

<br/>
#### Run the client for its output

{% codeblock lang:scala %}
// create a client that sends 1024 random bytes
val dataArray = Array.fill[Byte](1024)(1)
scala.util.Random.nextBytes(dataArray)
val clientOutput = clientEcho(addr, Bytes.of(dataArray))

// consumes all received data... (it should contain dataArray)
val result = clientOutput.runLog.run

println("Client received:"+result)
{% endcodeblock %}

It would give something like:

{% codeblock lang:scala %}
Client received:Vector(Bytes1: pos=0, length=1024, src: (-12,28,55,-124,3,-54,-53,66,-115,17...)
{% endcodeblock %}

<br/>
> Now, you know about scalaz-stream clients, what about servers???

<br/>
<br/>
## Scalaz-stream NIO Server

Let's start again :D

### What is a server?

- Something listening for client(s) connection
- When there is a client connected, the server can :
    * Receive data `I` _(for Input)_ from the client
    * Send data `W` _(for Write)_ to the client
- A server can manage multiple clients in parallel

<br/>
### Server seen as `Process`

Remember that a client was defined above as:

{% codeblock lang:scala %}
Client === Process[Task, Exchange[I, W]]
{% endcodeblock %}

In our NIO, non-blocking, streaming world, a server can be considered as a stream of clients right?

So finally, we can model a server as :

{% codeblock lang:scala %}
Server === Process[Task, Client[I, W]]
       === Process[Task, Process[Task, Exchange[I, W]]]
{% endcodeblock %}

> Whoooohoooo, a server is just a stream of streams!!!!

<br/>
### Writing a server

Scalaz-Stream now provides a helper to create a TCP binary NIO server:

{% codeblock lang:scala %}
// the address of the server
val address: InetSocketAddress = new InetSocketAddress("xxx.yyy.zzz.ttt", port)

// create a server
val server: Process[Task, Process[Task, Exchange[Bytes, Bytes]]] = 
  nio.server(address)

server map { client =>
  // for each client
  client flatMap { ex: Exchange[Bytes, Bytes] =>
    // read data sent by client in ex.read
    ???
    // write data to the client with ex.write
    ???
  }
}
{% endcodeblock %}

> Don't you find that quite elegant? ;)

<br/>
### Managing client/server interaction business logic

There we simply re-use the `Exchange` described above so you can use exactly the same API than the ones for client. Here is another API that can be useful:

{% codeblock lang:scala %}
type Writes1[W, I, I2] = Process[I, W \/ I2]

case class Exchange[I, W](read: Process[Task, I], write: Sink[Task, W]) {
...
  /**
   * Transforms this exchange to another exchange, that for every received `I` will consult supplied Writer1
   * and eventually transforms `I` to `I2` or to `W` that is sent to remote system.
   */
  def readThrough[I2](w: Writer1[W, I, I2])(implicit S: Strategy = Strategy.DefaultStrategy) : Exchange[I2,W]
...
}

// A small schema?
            ex.read
              |
              v
              I
              |
 ---------------------------
|    Writer1[W, I, I2]      |
 ---------------------------
              |
          ---------
          |       |
          v       v
          W       I2
          |
          v
       ex.write

======> Returns Exchange[I2, W]

{% endcodeblock %}

With this API, you can compute some business logic on the received data from client.

Let's write the echo server corresponding to the previous client (_you can find this sample in scalaz-stream too_):

{% codeblock lang:scala %}
def serverEcho(address: InetSocketAddress): Process[Task, Process[Task, Bytes]] = {

  // This is a Writer1 that echoes everything it receives to the client and emits it locally
  def echoAll: Writer1[Bytes, Bytes, Bytes] =
    receive1[Bytes, Bytes \/ Bytes] { i =>
      // echoes on left, emits on right and then loop (fby = followed by)
      emitSeq( Seq(\/-(i), -\/(i)) ) fby echoAll
    }

  // The server that echoes everything
  val receivedData: Process[Task, Process[Task, Bytes]] =
    for {
      client <- nio.server(address)
      rcv    <- ex.readThrough(echoAll).run()
    } yield rcv
  }

  receivedData
}

{% endcodeblock %}

`receivedData` is `Process[Task, Process[Task, Bytes]]` which is not so practical: we would prefer to gather all data received by clients in 1 single `Process[Task, Bytes]` to stream it to another module.

Scalaz-Stream has the solution again:

{% codeblock lang:scala %}
package object merge {
  /**
   * Merges non-deterministically processes that are output of the `source` process.
   */
  def mergeN[A](source: Process[Task, Process[Task, A]])
    (implicit S: Strategy = Strategy.DefaultStrategy): Process[Task, A]
}
{% endcodeblock %}

> Please note the Strategy which corresponds to the way Tasks will be executed and that can be compared to Scala `ExecutionContext`.

Fantastic, let's plug it on our server:

{% codeblock lang:scala %}

// The server that echoes everything
def serverEcho(address: InetSocketAddress): Process[Task, Bytes] = {

  // This is a Writer1 that echoes everything it receives to the client and emits it locally
  def echoAll: Writer1[Bytes, Bytes, Bytes] =
    receive1[Bytes, Bytes \/ Bytes] { i =>
      // echoes on left, emits on right and then loop (fby = followed by)
      emitSeq( Seq(\/-(i), -\/(i)) ) fby echoAll
    }

  // The server that echoes everything
  val receivedData: Process[Task, Process[Task, Bytes]] =
    for {
      client <- nio.server(address)
      rcv    <- ex.readThrough(echoAll).run()
    } yield rcv
  }

  // Merges all client streams
  merge.mergeN(receivedData)
}
{% endcodeblock %}

> Finally, we have a server and a client!!!!!
>
> Let's plug them all together

## Run a server

First of all, we need to create a server that can be stopped when required.

Let's do in the scalaz-stream way using:

- `wye.interrupt` :

{% codeblock lang:scala %}
  /**
   * Let through the right branch as long as the left branch is `false`,
   * listening asynchronously for the left branch to become `true`.
   * This halts as soon as the right branch halts.
   */
  def interrupt[I]: Wye[Boolean, I, I]
{% endcodeblock %}

- `async.signal` which is a value that can be changed asynchronously based on 2 APIs:

{% codeblock lang:scala %}

  /**
   * Sets the value of this `Signal`. 
   */
  def set(a: A): Task[Unit]

  /**
   * Returns the discrete version of this signal, updated only when `value`
   * is changed ...
   */
  def discrete: Process[Task, A]
{% endcodeblock %}

Without lots of imagination, we can use a `Signal[Boolean].discrete` to obtain a `Process[Task, Boolean]` and wye it with previous server process using `wye.interrupt`. Then, to stop server, you just have to call:

{% codeblock lang:scala %}
signal.set(true)
{% endcodeblock %}

Here is the full code:

{% codeblock lang:scala %}
// local bind address
val addr = localAddress(12345)

// The stop signal initialized to false
val stop = async.signal[Boolean]
stop.set(false).run

// Create the server controlled by the previous signal
val stoppableServer = (stop.discrete wye echoServer(addr))(wye.interrupt)

// Run server in async without taking care of output data
stopServer.runLog.runAsync( _ => ())

// DO OTHER THINGS

// stop server
stop.set(true)
{% endcodeblock %}


## Run server & client in the same code

{% codeblock lang:scala %}
// local bind address
val addr = localAddress(12345)

// the stop signal initialized to false
val stop = async.signal[Boolean]
stop.set(false).run

// Create the server controlled by the previous signal
val stoppableServer = (stop.discrete wye serverEcho(addr))(wye.interrupt)

// Run server in async without taking care of output data
stoppableServer.runLog.runAsync( _ => ())

// create a client that sends 1024 random bytes
val dataArray = Array.fill[Byte](1024)(1)
scala.util.Random.nextBytes(dataArray)
val clientOutput = clientEcho(addr, Bytes.of(dataArray))

// Consume all received data in a blocking way...
val result = clientOutput.runLog.run

// stop server
stop.set(true)
{% endcodeblock %}

> Naturally you rarely run the client & server in the same code but this is funny to see our easily you can do that with scalaz-stream as you just manipulate `Process` run on provided `Strategy`

<br/>
> Finally, we can go back to our subject: **feeding a `DStream` using a scalaz-stream NIO client/server**

## Pipe server output to DStream

`clientEcho/serverEcho` are simple samples but not very useful.

Now we are going to use a custom client/server I've written for this article:

- `NioClient.sendAndCheckSize` is a client streaming all emitted data of a `Process[Task, Bytes]` to the server and checking that the global size has been ack'ed by server.
- `NioServer.ackSize` is a server acknowledging all received packets by their size (as a 4-bytes Int) 

Now let's write a client/server dstreamizing data to Spark:


{% codeblock lang:scala %}
// First create a streaming context
val ssc = new StreamingContext(clusterUrl, "SparkStreamStuff", Seconds(1))

// Local bind address
val addr = localAddress(12345)

// The stop signal initialized to false
val stop = async.signal[Boolean]
stop.set(false).run

// Create the server controlled by the previous signal
val stoppableServer = (stop.discrete wye NioServer.ackSize(addr))(wye.interrupt)

// Create a client that sends a natural integer every 50ms as a string (until reaching 100)
val clientData: Process[Task, Bytes] = naturalsEvery(50 milliseconds).take(100).map(i => Bytes.of(i.toString.getBytes))
val clientOutput = NioClient.sendAndCheckSize(addr, clientData)

// Dstreamize the server into the streaming context
val (consumer, dstream) = dstreamize(stoppableServer, ssc)

// Prepare dstream output
dstream.map( bytes => new String(bytes.toArray) ).print()

// Start the streaming context
ssc.start()

// Run the server just for its effects
consumer.run.runAsync( _ => () )

// Run the client in a blocking way
clientOutput.runLog.run

// Await SSC termination a bit
ssc.awaitTermination(1000)

// stop server
stop.set(true)

// stop the streaming context
ssc.stop()

{% endcodeblock %}

When run, it prints :

{% codeblock lang:scala %}
-------------------------------------------
Time: 1395049304000 ms
-------------------------------------------

-------------------------------------------
Time: 1395049305000 ms
-------------------------------------------
0
1
2
3
4
5
6
7
8
9
...

-------------------------------------------
Time: 1395049306000 ms
-------------------------------------------
20
21
22
23
24
25
26
27
28
29
...
{% endcodeblock %}

Until 100...

<br/>
<br/>
## Part2's conclusion

I spent this second part of my tryptic mainly explaining a few concepts of the new scalaz-stream brand new NIO API. With it, a client becomes just a stream of exchanges `Process[Task, Exchange[I, W]]` and a server becomes a stream of stream of exchanges `Process[Task, Process[Task, Exchange[I, W]]]`.

As soon as you manipulate `Process`, you can then use the `dstreamize` API exposed in [Part 1](/2014/03/08/zpark-ml-nio-1/) to pipe streamed data into Spark.

**Let's go to [Part 3](/2014/03/10/zpark-ml-nio-3/) now in which we're going to do some fancy Machine Learning training with these new tools.**

<br/>
<br/>
> [GO TO PART1](/2014/03/08/zpark-ml-nio-1/) < -----------------------------------------------------------------------------> [GO TO PART3](/2014/03/10/zpark-ml-nio-3/)

<br/>
<br/>
<br/>
<br/>

