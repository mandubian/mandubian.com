---
layout: post
title: "Scalaz-Stream Plug'n'Play2 Iteratee/WS + Recursive streaming"
date: 2013-08-21 23:23
comments: true
external-url:
categories: [scalaz-stream,scalaz,play framework,stream,async,non-blocking,realtime,future,promise,fibonacci]
keywords: scalaz-stream,scalaz,play framework,stream,async,non-blocking,realtime,future,promise,fibonacci
---

The code for all autosources & sample apps can be found on Github [here](https://github.com/mandubian/playzstream)


>The aim of this article is to show how [scalaz-stream](https://github.com/scalaz/scalaz-stream) could be plugged on existing Play Iteratee/Enumerator and used in your web projects. I also wanted to evaluate in depth the power of [scalaz-stream](https://github.com/scalaz/scalaz-stream) _Processes_ by trying to write a recursive streaming action: I mean a web endpoint streaming data and re-injecting its own streamed data in itself.

<br/>
>If you want to see now how scalaz-stream is used with Play, go to <a href="#plug-n-play">this paragraph</a> directly.
<br/>

## Why Scalaz-Stream when you have Play Iteratees?
<br/>
### Play Iteratees are powerful & cool but...

I'm a fan of everything dealing with data streaming and realtime management in backends. I've worked a lot on [Play Framework](http://www.playframework.org) and naturally I've been using the cornerstone behind Play's reactive nature: _Play Iteratees_.

_Iteratees_ (with its counterparts, _Enumerators_ and _Enumeratees_) are great to **manipulate/transform linear streams of data chunks** in a very **reactive** (non-blocking & asynchronous) and purely **functional** way: 

- **Enumerators** identifies the **data producer** that can generate finite/infinite/procedural data streams. 
- **Iteratee** is simply a **data folder built as a state machine** based on 3 states (Continue, Done, Error) which consumes data from Enumerator to compute a final result. 
- **Enumeratee** is a kind of **transducer** able to adapt an Enumerator producing some type of data to an Iteratee that expects other type of data. Enumeratee can be used as both a pipe transformer and adapter.

Iteratee is really powerful but I must say I've always found them quite picky to use, practically speaking. In Play, they are used in their best use-case and they were created for that exactly. **I've been using Iteratees for more than one year now but I still don't feel fluent with them**. Each time I use them, I must spend some time to know how I could write what I need. It's not because they are purely functional (piping an Enumerator into an Enumeratee into an Iteratee is quite trivial) but there is something that my brain doesn't want to catch.

>If you want more details about my experience with Iteratees, go to <a href="#iteratee-details">this paragraph</a>

That's why **I wanted to work with other functional streaming tools to see if they suffer the same kind of usability toughness** or can bring something more natural to me. There are lots of other competitors on the field such as **pipes**, **conduits** and **machines**. As I don't have physical time to study all of them in depth, I've chosen the one that appealed me the most i.e. Machines.

> I'm not yet a Haskell coder even if I can mumble it so I preferred to evaluate the concept with [scalaz-stream](https://github.com/scalaz/scalaz-stream), a Scala implementation trying to bring machines to _normal_ coders focusing on the aspect of IO streaming.

<br/>
<br/>

## Scratching the concepts of Machine / Process ?

>I'm not going to judge if Machines are better or not than Iteratees, this is not my aim. I'm just experimenting the concept in an objective way.

I won't explain the concept of Machines in depth because it's huge and I don't think I have the theoretical background to do it right now. So, let's focus on very basic ideas at first:

- _Machine_ is a very generic concept that represents **a data processing mechanism with potential multiple inputs, an output and monadic effects** (typically Future input chunks, side-effects while transforming, delayed output...)
- To simplify, let say a machine is **a bit like a mechano that you construct by plugging together other more generic machines** (such as source, transducer, sink, tee, wye) as simply as pipes.
- Building a machine also means **planning all the steps you will go through when managing streamed data but it doesn't do anything until you run it** (no side-effect, no resource consumption). You can re-run a machine as many times as you want.
- A machine is a **state machine** (Emit/Await/Halt) as Iteratee but it manages error in a more explicit way IMHO (fallback/error)

In [scalaz-stream](https://github.com/scalaz/scalaz-stream), you don't manipulate machines which are too abstract for real-life use-cases but you manipulate simpler concepts:

- `Process[M, O]` is a restricted machine outputting a stream of `O`. It can be a source if the monadic effect gets input from I/O or generates procedural data, or a sink if you don't care about the output. _Please note that it doesn't infer the type of potential input at all_.
- `Wye[L, R, O]` is a machine that takes 2 inputs (left `L` / right `R`) and outputs chunks of type `O` (you can read from left or right or wait for both before ouputting)
- `Tee[L, R, O]` is a Wye that can only read alternatively from left or from right but not from both at the same time.
- `Process1[I, O]` can be seen as a transducer which accepts inputs of type `I` and outputs chunks of type `O` (a bit like Enumeratee)
- `Channel[M, I, O]` is an effectul channel that accepts input of type `I` and use it in a monadic effect `M` to produce potential `O`

<br/>
### What I find attractive in Machines?

- Machines is **producer/consumer/transducer in the same place** and Machines can consume/fold as Iteratee, transform as Enumeratee and emit as Enumerator at the same time and it opens lots of possibilities (even if 3 concepts in one could make it more complicated too).
- I feel like **playing with legos as you plug machines on machines** and this is quite funny actually.
- Machines manages **monadic effects in its design** and doesn't infer the type of effect so you can use it with I/O, Future and whatever you can imagine that is monadic...
- Machines provide out-of-the-box **Tee/Wye to compose streams**, interleave, zip them as you want without writing crazy code.
- The early code samples I've seen were quite easy to read (even the implementation is not so complex). Have a look at the `StartHere` sample provided by scalaz-stream:

{% codeblock lang:scala %}
  property("simple file I/O") = secure {

    val converter: Task[Unit] =
      io.linesR("testdata/fahrenheit.txt")
        .filter(s => !s.trim.isEmpty && !s.startsWith("//"))
        .map(line => fahrenheitToCelsius(line.toDouble).toString)
        .intersperse("\n")
        .pipe(process1.utf8Encode)
        .to(io.fileChunkW("testdata/celsius.txt"))
        .run

    converter.run
    true
  }
{% endcodeblock %}

>But don't think everything is so simple, machines is a complex concept with lots of theory behind it which is quite abstract. what I find very interesting is that **it's possible to vulgarize this very abstract concept with simpler concepts such as Process, Source, Sink, Tee, Wye... that you can catch quite easily** as these are concepts you already manipulated when you were playing in your bathtub when you were child (or even now).

<br/>
<br/>
## <a name="plug-n-play">Scalaz-stream Plug'n'Play  Iteratee/Enumerator</a>

After these considerations, I wanted to experiment scalaz-stream with Play streaming capabilities in order to see how it behaves in a context I know. 

Here is what I decided to study:

- **Stream data out of a controller action using a scalaz-stream `Process`**
- **Call an AsyncWebService & consume the response as a stream of `Array[Byte]` using a scalaz-stream `Process`**

Here is existing Play API :

- Action provides `Ok.stream(Enumerator)`
- WS call consuming response as a stream of data `WS.get(r: ResponseHeader => Iteratee)`

<br/>
>As you can see, these API depends on Iteratee/Enumerator. As I didn't want to hack Play too much as a beginning, I decided to try & plug scalaz-stream on Play Iteratee (if possible).

### Building `Enumerator[O]` from `Process[Task, O]`

The idea is to take a scalaz-stream Source[O] (`Process[M,O]`) and wrap it into an `Enumerator[O]` so that it can be used in Play controller actions.

An Enumerator is a data producer which can generate those data using monadic `Future` effects (Play Iteratee is tightly linked to `Future`).

`Process[Task, O]` is a machine outputting a stream of `O` so it's logically the right candidate to be adapted with a `Enumerator[O]`. _Let's remind' `Task` is just a scalaz `Future[Either[Throwable,A]]` with a few helpers and it's used in scalaz-stream_.

So I've implemented (at least tried) an `Enumerator[O]` that accepts a `Process[Task, O]`:

{% codeblock lang:scala %}
  def enumerator[O](p: Process[Task, O])(implicit ctx: ExecutionContext) = 
    new Enumerator[O] {
      ...
      // look the code in github project
      ...
  }
{% endcodeblock %}

>The implementation just synchronizes the states of the `Iteratee[O, A]` consuming the `Enumerator` with the states of `Process[Task, O]` emitting data chunks of `O`. It's quite simple actually.

<br/>
<br/>
### Building `Process1[I, O]` from `Iteratee[I, O]`

The idea is to drive an Iteratee from a scalaz-stream Process so that it can consume an Enumerator and be used in Play WS.

An `Iteratee[I, O]` accepts inputs of type `I` (_and nothing else_) and will fold the input stream into a single result of type `O`.

A `Process1[I, O]` accepts inputs of type `I` and emits chunks of type `O` but not necessarily one single output chunk. So it's a good candidate for our use-case but we need to choose which emitted chunk will be the result of the `Iteratee[I, O]`. here, totally arbitrarily, I've chosen to take the first emit as the result (_but the last would be as good if not better_).

So I implemented the following:

{% codeblock lang:scala %}
  def iterateeFirstEmit[I, O](p: Process.Process1[I, O])(implicit ctx: ExecutionContext): Iteratee[I, O] = {
  ...
  // look the code in github project
  ...
}
{% endcodeblock %}

>The implementation is really raw for experimentation as it goes through the states of the `Process1[I,O]` and generates the corresponding states of `Iteratee[I,O]` until first emitted value. Nothing more nothing less...

<br/>
<br/>
## A few basic action samples

> Everything done in those samples could be done with Iteratee/Enumeratee more or less simply. The subject is not there!

<br/>
### Sample 1 : Generates a stream from a Simple Emitter Process

{% codeblock lang:scala %}
def sample1 = Action {
  val process = Process.emitAll(Seq(1, 2, 3, 4)).map(_.toString)

  Ok.stream(enumerator(process))
}
{% endcodeblock %}

{% codeblock lang:scala %}
> curl "localhost:10000/sample1" --no-buffer
1234
{% endcodeblock %}

<br/>
### Sample 2 : Generates a stream from a continuous emitter

{% codeblock lang:scala %}
/** A process generating an infinite stream of natural numbers */
val numerals = Process.unfold(0){ s => val x = s+1; Some(x, x) }.repeat

// we limit the number of outputs but you don't have it can stream forever...
def sample2 = Action {
  Ok.stream(enumerator(numerals.map(_.toString).intersperse(",").take(40)))
}
{% endcodeblock %}

{% codeblock lang:scala %}
> curl "localhost:10000/sample2" --no-buffer
1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,
{% endcodeblock %}

<br/>
### Sample 3 : Generates a stream whose output frequency is controlled by a tee with numeral generator on left and ticker on right

{% codeblock lang:scala %}
/** ticks constant every delay milliseconds */
def ticker(constant: Int, delay: Long): Process[Task, Int] = Process.await(
  scalaFuture2scalazTask(delayedNumber(constant, delay))
)(Process.emit).repeat

def sample3 = Action {
  Ok.stream(enumerator(
    // creates a Tee outputting only numerals but consuming ticker // to have the delayed effect
    (numerals tee ticker(0, 100))(processes.zipWith((a,b) => a))
      .take(100)
      .map(_.toString)
      .intersperse(",")
  ))
}
{% endcodeblock %}

Please note :

- `scalaFuture2scalazTask` is just a helper to convert a `Future` into `Task`
- `ticker`is quite simple to understand: it awaits `Task[Int] and emits this `Int and repeats it again...
- `processes.zipWith((a,b) => a)` is a tee (2 inputs left/right) that outputs only left data but consumes right also to have the delay effect.
- `.map(_.toString)` simply converts into something writeable by `Ok.stream`
- `.intersperse(",")` which simply add `"," between each element

{% codeblock lang:scala %}
> curl "localhost:10000/sample3" --no-buffer
1... // to simulate the progressive apparition of numbers on screen
1,...
1,2...
...
1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80,81,82,83,84,85,86,87,88,89,90,91,92,93,94,95,96,97,98,99,100
{% endcodeblock %}

<br/>
### Sample 4 : Generates a stream using side-effect to control output frequency

{% codeblock lang:scala %}
/** Async generates this Int after delay*/
def delayedNumber(i: Int, delay: Long): Future[Int] =
  play.api.libs.concurrent.Promise.timeout(i, delay)

/** Creates a process generating an infinite stream natural numbers
  * every `delay milliseconds
  */
def delayedNumerals(delay: Long) = {
  def step(i: Int): Process[Task, Int] = {
    Process.emit(i).then(
      Process.await(scalaFuture2scalazTask(delayedNumber(i+1, delay)))(step)
    )
  }
  Process.await(scalaFuture2scalazTask(delayedNumber(0, delay)))(step)
}

def sample4 = Action {
  Ok.stream(enumerator(delayedNumerals(100).take(100).map(_.toString).intersperse(",")))
}
{% endcodeblock %}

Please note:

- `delayedNumber` uses an Akka scheduler to trigger our value after timeout
- `delayedNumerals` shows a simple recursive `Process[Task, Int] construction which shouldn't be too hard to understand

{% codeblock lang:scala %}
> curl "localhost:10000/sample4" --no-buffer
0... // to simulate the progressive apparition of numbers every 100ms
0,...
0,1...
0,1,...
0,1,2...
...
0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80,81,82,83,84,85,86,87,88,89,90,91,92,93,94,95,96,97,98,99
{% endcodeblock %}

<br/>
### Sample 5 : Generates a stream by consuming completely another stream

{% codeblock lang:scala %}
// a process folding all Array[Byte] into a big String
val reader: Process.Process1[Array[Byte], String] = processes.fold1[Array[Byte]]((a, b) => a ++ b )
  .map{ arr => new String(arr) } |> processes.last

def sample5 = Action {
  // the WS call with response consumer by previous Process1[Array[Byte], String] driving the Iteratee[Array[Byte], String]
  val maybeValues: Future[String] =
    WS.url(routes.Application.sample2().absoluteURL())
      .get(rh => iterateeFirstEmit(reader))
      .flatMap(_.run)

  Ok.stream(enumerator(
    // wraps the received String in a Process
    // re-splits it to remove ","
    // emits all chunks
    Process.wrap(scalaFuture2scalazTask(maybeValues))
      .flatMap{ values => Process.emitAll(values.split(",")) }
  ))
}
{% endcodeblock %}

Please note:

- `reader` is a `Process1[Array[Byte], String] that folds all received `Array[Byte]` into a `String`
- `iterateeFirstEmit(reader)` simulates an `Iteratee[Array[Byte], String]` driven by the `reader` process that will fold all chunks of data received from WS call to `routes.Application.sample2()`
- `.get(rh => iterateeFirstEmit(reader))` will return a `Future[Iteratee[Array[Byte], String]` that is run in `.flatMap(_.run)` to return a `Future[String]`
- `Process.wrap(scalaFuture2scalazTask(maybeValues))` is a trick to wrap the folded `Future[String]` into a `Process[Task, String]`
- `Process.emitAll(values.split(","))` splits the resulting string again and emits all chunks outside (stupid, just for demo)

{% codeblock lang:scala %}
> curl "localhost:10000/sample5" --no-buffer
1234567891011121314151617181920
{% endcodeblock %}

<br/>
> Still there? Let's dive deeper and be sharper!
<br/>

## Building recursive streaming action consuming itself

<br/>
### Hacking WS to consume & re-emit WS in realtime

`WS.executeStream(r: ResponseHeader => Iteratee[Array[Byte], A])` is cool API because you can build an iteratee from the ResponseHeader and then the iteratee will consume received `Array[Byte] chunks in a reactive way and will fold them. The problem is that until the iteratee has finished, you won't have any result.

But I'd like to be able to receive chunks of data in realtime and re-emit them immediately so that I can inject them in realtime data flow processing. WS API doesn't allow this so I decided to hack it a bit. I've written `WSZ` which provides the API:

{% codeblock lang:scala %}
def getRealTime(): Process[Future, Array[Byte]]
// based on
private[libs] def realtimeStream: Process[Future, Array[Byte]]
{% endcodeblock %}

This API outputs a realtime Stream of `Array[Byte]` whose flow is controlled by promises (`Future`) being redeemed in AsyncHttpClient `AsyncHandler`. _I didn't care about ResponseHeaders for this experimentation but it should be taken account in a more serious impl._

I obtain a `Process[Future, Array[Byte]]` streaming received chunks in realtime and I can then take advantage of the power of machines to manipulate the data chunks as I want.

<br/>
### Sample 6 : Generates a stream by forwarding/refolding another stream in realtime

{% codeblock lang:scala %}
/** A Process1 splitting input strings using splitter and re-grouping chunks */
def splitFold(splitter: String): Process.Process1[String, String] = {
  // the recursive splitter / refolder
  def go(rest: String)(str: String): Process.Process1[String, String] = {
    val splitted = str.split(splitter)
    println(s"""$str - ${splitted.mkString(",")} --""")
    (splitted.length match {
      case 0 => 
        // string == splitter
        // emit rest
        // loop
        Process.emit(rest).then( Process.await1[String].flatMap(go("")) )
      case 1 => 
        // splitter not found in string 
        // so waiting for next string
        // loop by adding current str to rest
        // but if we reach end of input, then we emit (rest+str) for last element
        Process.await1[String].flatMap(go(rest + str)).orElse(Process.emit(rest+str))
      case _ =>
        // splitter found
        // emit rest + splitted.head
        // emit all splitted elements but last
        // loops with rest = splitted last element
        Process.emit(rest + splitted.head)
               .then( Process.emitAll(splitted.tail.init) )
               .then( Process.await1[String].flatMap(go(splitted.last)) )
    })
  }
  // await1 simply means "await an input string and emits it"
  Process.await1[String].flatMap(go(""))
}

def sample6 = Action { implicit request =>
  val p = WSZ.url(routes.Application.sample4().absoluteURL()).getRealTime.translate(Task2FutureNT)

  Ok.stream(enumerator(p.map(new String(_)) |> splitFold(",")))
}
{% endcodeblock %}

Please note:

- `def splitFold(splitter: String): Process.Process1[String, String]` is just a demo that coding a Process transducer isn't so crazy... Look at comments in code
- `.translate(Task2FutureNF)` converts the `Process[Future, Array[Byte]]` to `Process[Task, Array[Byte]]` using Scalaz Natural Transformation.
- `p |> splitFold(",")` means "pipe output of process `p` to input of `splitFold`".

{% codeblock lang:scala %}
> curl "localhost:10000/sample6" --no-buffer
0...
01...
012...
...
01234567891011121314151617181920212223242526272829303132333435363738394041424344454647484950515253545556575859606162636465666768697071727374757677787980818283848586878889909192939495969798
{% endcodeblock %}

<br/>
> Let's finish our trip with a bit of puzzle and mystery.
<br/>
### THE FINAL MYSTERY: recursive stream generating Fibonacci series

As soon as my first experimentations of scalaz-stream with Play were operational, I've imagined an interesting case: 

>Is it possible to build an action generating a stream of data fed by itself: a kind of recursive stream.

With Iteratee, it's not really possible since it can't emit data before finishing iteration. It would certainly be possible with an Enumeratee but the API doesn't exist and I find it much more obvious with scalaz-stream API! 

The mystery isn't in the answer to my question: YES it is possible!

The idea is simple:

- Create a simple action
- Create a first process emitting a few initialization data
- Create a second process which consumes the WS calling my own action and re-emits the received chunks in realtime
- Append first process output and second process output
- Stream global output as a result of the action which will back-propagated along time to the action itself...

Naturally, if it consumes its own data, it will recall itself again and again and again until you reach the connections or opened file descriptors limit. As a consequence, you must limit the depth of recursion.

I performed different experiences to show this use-case by zipping the stream with itself, adding elements with themselves etc...
And after a few tries, I implemented the following code quite fortuitously :

{% codeblock lang:scala %}
/** @param curDepth the current recursion depth
  * @param maxDepth the max recursion depth
  */
def sample7(curDepth: Int, maxDepth: Int) = Action { implicit request =>

  // initializes serie with 2 first numerals output with a delay of 100ms
  val init: Process[Task, String] = delayedNumerals(100).take(2).map(_.toString)

  // Creates output Process
  // If didn't reach maxDepth, creates a process consuming my own action
  // If reach maxDepth, just emit 0
  val outputProcess = 
    if(curDepth < maxDepth) {
      // calling my own action and streaming chunks using getRealTime

      val myself = WSZ.url(
        routes.Application.sample7(curDepth+1, maxDepth).absoluteURL()
      ).getRealTime.translate(Task2FutureNT).map(new String(_))
      // splitFold isn't useful, just for demo
      |> splitFold(",")

      // THE IMPORTANT PART BEGIN
      // appends `init` output with `myself` output
      // pipe it through a helper provided scalaz-stream `processes.sum[Long]`
      // which sums elements and emits partial sums
      ((init append myself).map(_.toLong) |> processes.sum[Long])
      // THE IMPORTANT PART END
      // just for output format
      .map(_.toString).intersperse(",")
    }
    else Process.emit(0).map(_.toString)

  Ok.stream(enumerator(outputProcess))
}
{% endcodeblock %}

Launch it:

{% codeblock lang:scala %}
curl "localhost:10000/sample7?curDepth=0&maxDepth=10" --no-buffer
0,1,1,2,3,5,8,13,21,34,55,89,144,233,377,610,987,1597,2584,4181,6765
{% endcodeblock %}

WTF??? This is Fibonacci series?

Just to remind you about it:
{% codeblock lang:scala %}
e(0) = 0
e(1) = 1
e(n) = e(n-1) + e(n-2)
{% endcodeblock %}

>Here is the mystery!!!
>
>How does it work???
>
>I won't tell the answer to this puzzling side-effect and let you think about it and discover why it works XD

But this sample shows exactly what I wanted: **Yes, it's possible to feed an action with its own feed! Victory!**

<br/>
<br/>

## Conclusion

Ok all of that was really funky but is it useful in real projects? I don't really know yet but it provides a great proof of the very reactive character of scalaz-stream and Play too!

I tend to like scalaz-stream and I feel more comfortable, more natural using Process than Iteratee right now... Maybe this is just an impression so I'll keep cautious about my conclusions for now...

All of this code is just experimental so be aware about it. If you like it and see that it could be useful, tell me so that we create a real library from it!

Have Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,
Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,
Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,Fun,!

<br/>
<br/>
## PostScriptum

### <a name="iteratee-details">A few more details about Iteratees</a>

Here are a few things that bother me when I use Play Iteratee (you don't have to agree, this is very subjective):

- Enumeratees are really powerful (maybe the most powerful part of the API) but they can be tricky: for ex, defining a new Enumeratee from scratch isn't easy at first sight due to the signature of the Enumeratee itself, Enumeratee composes differently on left (with Enumerators) and on right (with Iteratees) and it can be strange at beginning...
- Enumerators are not defined (when not using helpers) in terms of the data they produce but with respect to the way an Iteratee will consume the data they will produce. You must somewhat reverse your way of thinking which is not so natural.
- Iteratees are great to produce one result by folding a stream of data but if you want to consume/cut/aggregate/re-emit the chunks, the code you write based on Iteratee/Enumeratee quickly becomes complex, hard to re-read and edge cases (error, end of stream) are hard to treat.
- When you want to manipulate multiple streams together, zip/interleave them, you must write very complex code too.
- End of iteration and Error management with Iteratees isn't really clear IMHO and when you begin to compose Iteratees together, it becomes hard to know what will happen...
- If you want to manipulate a stream with side-effecting, you can do it with Enumeratees but it's not so obvious...

<br/>
<br/>
<br/>
<br/>