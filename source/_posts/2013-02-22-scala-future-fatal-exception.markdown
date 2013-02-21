---
layout: post
title: "Being aware Scala 2.10.0 Futures conceal Fatal exceptions"
date: 2013-02-22 14:14
comments: true
external-url: 
categories: [scala,scala 2.10,future,concurrent,future,execution context,thread,forkjoin]
keywords: scala,scala 2.10,future,concurrent,future,execution context,thread,forkjoin
---

A short article to talk about **an interesting issue concerning Scala 2.10.0 Future that might interest you**. 

<div class="well">
<h3>Summary</h3><br/>
<p>When a <code>Fatal</code> exception is thrown in your <code>Future</code> callback, it's not caught by the <code>Future</code> and is thrown to the provided <code>ExecutionContext</code>.</p>
<p><i>But the current default Scala global <code>ExecutionContext</code> doesn't register an <code>UncaughtExceptionHandler</code> for these fatal exceptions and your <code>Future</code> just hangs forever without notifying anything to anybody.</i></p>
</div>

> This issue is well [known](https://issues.scala-lang.org/browse/SI-7029) and a solution to the problem has already been [merged](https://github.com/scala/scala/pull/2044) into branch 2.10.x. But this issue is present in Scala 2.10.0 so it's interesting to keep this issue in mind IMHO. Let's explain clearly about it.


## Exceptions can be contained by Future

Let's write some stupid code with Futures.

{% codeblock lang:scala %}
scala> import scala.concurrent._
scala> import scala.concurrent.duration._

// Take default Scala global ExecutionContext which is a ForkJoin Thread Pool
scala> val ec = scala.concurrent.ExecutionContext.global
ec: scala.concurrent.ExecutionContextExecutor = scala.concurrent.impl.ExecutionContextImpl@15f445b7

// Create an immediately redeemed Future with a simple RuntimeException
scala> val f = future( throw new RuntimeException("foo") )(ec)
f: scala.concurrent.Future[Nothing] = scala.concurrent.impl.Promise$DefaultPromise@27380357

// Access brutally the value to show that the Future contains my RuntimeException
scala> f.value
res22: Option[scala.util.Try[Nothing]] = Some(Failure(java.lang.RuntimeException: foo))

// Use blocking await to get Future result
scala> Await.result(f, 2 seconds)
warning: there were 1 feature warnings; re-run with -feature for details
java.lang.RuntimeException: foo
  at $anonfun$1.apply(<console>:14)
  at $anonfun$1.apply(<console>:14)
  ...
{% endcodeblock %}

> You can see that a `Future` can contain an `Exception` (or more generally `Throwable`).

<br/>
## Fatal Exceptions can't be contained by Future

If you look in [Scala 2.10.0 Future.scala](https://github.com/scala/scala/blob/v2.10.0/src/library/scala/concurrent/Future.scala#L53), in the scaladoc, you can find:

{% codeblock %}
* The following throwable objects are not contained in the future:
* - `Error` - errors are not contained within futures
* - `InterruptedException` - not contained within futures
* - all `scala.util.control.ControlThrowable` except `NonLocalReturnControl` - not contained within futures
{% endcodeblock %}

and in the code, in several places, in `map` or `flatMap` for example, you can read:

{% codeblock lang:scala %}
try {
...
} catch {
  case NonFatal(t) => p failure t
}
{% endcodeblock %}

> This means that every `Throwable` that is Fatal can't be contained in the `Future.Failure`.

<br/>
## What's a _Fatal_ Throwable?

To define what's fatal, let's see what's declared as non-fatal in [NonFatal ScalaDoc](http://www.scala-lang.org/archives/downloads/distrib/files/nightly/docs/library/index.html#scala.util.control.NonFatal$).

{% codeblock %}
* Extractor of non-fatal Throwables.  
* Will not match fatal errors like VirtualMachineError  
* (for example, OutOfMemoryError, a subclass of VirtualMachineError),  
* ThreadDeath, LinkageError, InterruptedException, ControlThrowable, or NotImplementedError. 
*
* Note that [[scala.util.control.ControlThrowable]], an internal Throwable, is not matched by
* `NonFatal` (and would therefore be thrown).
{% endcodeblock %}

> Let's consider Fatal exceptions are just critical errors that can't be recovered in general.

## So what's the problem?

It seems right not to catch fatal errors in the `Future, isn't it?

But, look at following code:

{% codeblock lang:scala %}
// Let's throw a simple Fatal exception
scala> val f = future( throw new NotImplementedError() )(ec)
f: scala.concurrent.Future[Nothing] = scala.concurrent.impl.Promise$DefaultPromise@59747b17

scala> f.value
res0: Option[scala.util.Try[Nothing]] = None

{% endcodeblock %}

Ok, the `Future` doesn't contain the Fatal Exception as expected.

**But where is my Fatal Exception if it's not caught??? No crash, notification or whatever?**

There should be an `UncaughtExceptionHandler at least notifying it!

<br/>
## The problem is in the default Scala `ExecutionContext`. 

As explained in this [issue](https://issues.scala-lang.org/browse/SI-7029), the exception is lost due to the implementation of the default global `ExecutionContext` provided in Scala. 

This is a simple ForkJoin pool of threads but it has no `UncaughtExceptionHandler`. Have a look at code in [Scala 2.10.0 ExecutionContextImpl.scala](https://github.com/scala/scala/blob/v2.10.0/src/library/scala/concurrent/impl/ExecutionContextImpl.scala#L72)

{% codeblock lang:scala %}
try {
      new ForkJoinPool(
        desiredParallelism,
        threadFactory,
        null, //FIXME we should have an UncaughtExceptionHandler, see what Akka does
        true) // Async all the way baby
    } catch {
      case NonFatal(t) =>
        ...
    }
{% endcodeblock %}

> Here it's quite clear: there is no registered `UncaughtExceptionHandler.

> What's the consequence?

## Your Future hangs forever

{% codeblock lang:scala %}
scala> Await.result(f, 30 seconds)
warning: there were 1 feature warnings; re-run with -feature for details
java.util.concurrent.TimeoutException: Futures timed out after [30 seconds]
  at scala.concurrent.impl.Promise$DefaultPromise.ready(Promise.scala:96)
{% endcodeblock %}

As you can see, you can wait as long as you want, the Future is never redeemed properly, it just hangs forever and you don't even know that a Fatal Exception has been thrown.

As explained in the issue, please note, if you use a custom `ExecutionContext` based on `SingleThreadExecutor`, this issue doesn't appear!

{% codeblock lang:scala %}
scala> val es = java.util.concurrent.Executors.newSingleThreadExecutor
es: java.util.concurrent.ExecutorService = java.util.concurrent.Executors$FinalizableDelegatedExecutorService@1e336f59

scala> val ec = ExecutionContext.fromExecutorService(es)
ec: scala.concurrent.ExecutionContextExecutorService = scala.concurrent.impl.ExecutionContextImpl$$anon$1@34f43dac

scala>  val f = Future[Unit](throw new NotImplementedError())(ec)
Exception in thread "pool-1-thread-1" f: scala.concurrent.Future[Unit] = scala.concurrent.impl.Promise$DefaultPromise@7d01f935
scala.NotImplementedError: an implementation is missing
  at $line41.$read$$iw$$iw$$iw$$iw$$iw$$iw$$anonfun$1.apply(<console>:15)
  at $line41.$read$$iw$$iw$$iw$$iw$$iw$$iw$$anonfun$1.apply(<console>:15)
{% endcodeblock %}


## Conclusion

**In Scala 2.10.0, if you have a Fatal Exception in a Future callback, your Future just trashes the Fatal Exception and hangs forever without notifying anything.**

Hopefully, due to this [already merged PR](https://github.com/scala/scala/pull/2044), in a future delivery of Scala 2.10.x, this problem should be corrected. 

To finish, in the same old good [issue](https://issues.scala-lang.org/browse/SI-7029), Viktor Klang also raised the question of what should be considered as fatal or not: 

> there's a bigger topic at hand here, the one whether NotImplementedError, InterruptedException and ControlThrowable are to be considered fatal or not.

Meanwhile, be aware and take care ;)

Have `Promise[NonFatal]`!