---
layout: post
title: "ShapelessStream, when Akka-Stream meets Shapeless Coproduct at compile-time"
date: 2015-05-05 23:23
comments: true
external-url:
categories: [free monad, functional programming, category, algebra, scala, quadratic complexity, observability, map fusion, cats]
keywords: free monad,functional programming,category,algebra,scala,quadratic complexity,observability,map fusion,cats
---

You might have seen on my twitter that my current company [MFG Labs](http://www.mfglabs.com) has opensourced the library [Akka-Stream Extensions](https://mfglabs.github.com/akka-stream-extensions). We have developed it with [Alexandre Tamborrino](http://twitter.com/altamborrino) and [Damien Pignaud](http://twitter.com/dmpignaud) for our recent production projects based on [Typesafe Akka-Stream](http://doc.akka.io/docs/akka-stream-and-http-experimental/1.0-RC2/scala.html).

In this article, I won't explain all the reasons that motivated our choice of Akka-Stream and the road towards our library [Akka-Stream Extensions](https://mfglabs.github.com/akka-stream-extensions). Here, I'll focus on one precise aspect of our choice: `types`... And I'll tell you about a specific extension I've created for this project: `ShapelessStream`.


## Akka-Stream loves Types 

As you may know, I'm a `Type` lover. Types are proofs and proofs are types. Proofs are what you want in your code to ensure _in a robust & reliable way_ that it does what it pretends _with the support of the compiler_.

**Typesafety is a very interested feature of Akka-Stream.**

Akka-Stream most basic primitive is `Flow[A, B]` that is a data-flow that accepts elements of `A` and will return elements of `B`. You can't pass a `C` to it and you are sure that this flow won't return a `C` also.

Actually, at MFG Labs, we have inherited some Scala legacy code mostly based on Akka actors which provide a very good way to handle failures but which are not typesafe at all (till Akka Typed for that point) and which are not composable and tend to scatter the business logic in your code. It has appeared that in many cases, Akka-Stream would be a very good way to replace those actors because of:

- better type-safety,
- fluent & composable code
- same Akka failure management
- buffer, parallel computing, backpressure

Yes, it's quit weird to say it but Akka-Stream helped us correct most problems that had been introduced using Akka.

## Multi-type flows

Ok, Akka-Stream promotes `Types` as first citizen in your data flows. That's cool!
But it appears that you often need to handle multiple types in the same input channel:

<img src="/images/mandubian/flow_11.png" />

When you control completely the types in input, you can represent input types by a classic ADT:

{% codeblock lang:scala %}
sealed trait In
case class A1(...) extends In
case class A2(...) extends In
case class A3(...) extends In
{% endcodeblock %}

... And manage it in `Flow[A, B]`:

{% codeblock lang:scala %}
Flow[In].map {
  case A1(...) => // some B
  case A2(...) => // some B
  case A3(...) => // some B
}
{% endcodeblock %}

That is nice but you need to wrap all input types in an ADT and this involves some boring code that can even be different for every custom flow.

Going further, in general, you don't want to do that, you want to dispatch every element to a different flow according to its type:

<img src="/images/mandubian/flow_22.png" />

... and merge all results of all flows in one single channel...
... and every flow has its own behavior in terms of parallelism, bufferization and back-pressure...

In Akka-Stream, if you wanted to write a flow managing the previous schema, you would have to use a [FlexiRoute](http://doc.akka.io/docs/akka-stream-and-http-experimental/1.0-RC2/scala/stream-customize.html) for the dispatch and a [FlexiMerge](http://doc.akka.io/docs/akka-stream-and-http-experimental/1.0-RC2/scala/stream-customize.html) for the merge.

Have a look at the doc and see that it requires quite a bunch of lines of code to write one of those. It's really powerful but quite tedious to implement and not so typesafe after all. Moreover, you certainly would have to write one FlexiRoute and one FlexiMerge per use-case as the number of inputs types and return types depend on your context.



## Miles Sabin to the rescue

In my latest project, this `dispatch/flows/merge` pattern was required in several places and as I'm lazy, I wanted something more elegant & typesafe if possible.

Thinking in terms of pure types, we can see the previous `dispatch/flows/merge` flow graph in pseudo-code as:

{% codeblock lang:scala %}
Flow[
  Input = A1 or A2 or A3,
  Ouput = B1 or B2 or B3
]
{% endcodeblock %}

and we need to provide a list of flows for all pairs of input/output types:

{% codeblock lang:scala %}
Flow[A1, B1] and Flow[A2, B2] and Flow[A3, B3]
{% endcodeblock %}


In [Shapeless](https://github.com/milessabin/shapeless), there are 2 very very very useful structures:

- `Coproduct` which is a generalization of the well known `Either`. You have `A or B` in `Either[A, B]`. With `Coproduct`, you can have more than 2 alternatives `A or B or C or D`. For our previous flow graph, using `Coproduct`, it could be written as:

{% codeblock lang:scala %}
Flow[
  A1 :+: A2 :+: A3 :+: CNil,
  B1 :+: B2 :+: B3 :+: CNil
]
{% endcodeblock %}

- `HList` which allows to build heterogenous `List` of elements keeping & tracking all types at compile time. For our previous list of flows, it would be really good as we want to know all the input/output of all flows. It would give:

{% codeblock lang:scala %}
Flow[A1, B1] :: Flow[A2, B2] :: Flow[A3, B3] :: HNil
{% endcodeblock %}

Finally, from an external point of view, building such `dispatch/flows/merge` looks like a function taking a `Hlist of flows` in input and building a `Flow of Coproducts`:


{% codeblock lang:scala %}
Flow[A1, B1] :: Flow[A2, B2] :: Flow[A3, B3] :: HNil => Flow[A1 :+: A2 :+: A3 :+: CNil, B1 :+: B2 :+: B3 :+: CNil]
{% endcodeblock %}


I'm really lazy and didn't want


## Mutable builders of flow graphs

An important concept in Akka-Stream is the separation of concerns between:

- constructing/describing a data-flow
- materializing with live resources
- running the data-flow by plugging live sources/sinks on it.

> You find the same idea in scalaz-stream but in a purer way as scalaz-stream relies on `Free` concepts that formalize this idea quite directly.

To build complex data flows, Akka-Stream provides a very nice DSL described [here](http://doc.akka.io/docs/akka-stream-and-http-experimental/1.0-RC2/scala/stream-graphs.html). This DSL takes advantage of the idea of mutable structures used to build until fixing definitely into an immutable structure.

An example from the doc:

{% codeblock lang:scala %}
val g = FlowGraph.closed() { implicit builder: FlowGraph.Builder[Unit] =>
  import FlowGraph.Implicits._
  val in = Source(1 to 10)
  val out = Sink.ignore
 
  val bcast = builder.add(Broadcast[Int](2))
  val merge = builder.add(Merge[Int](2))
 
  val f1, f2, f3, f4 = Flow[Int].map(_ + 10)
 
  in ~> f1 ~> bcast ~> f2 ~> merge ~> f3 ~> out
              bcast ~> f4 ~> merge
}
{% endcodeblock %}

`builder` is the mutable structure used to build the flow graph using the DSL.

The value `g` is the immutable structure resulting from the builder.

> This idea of mutable builder is really cool because mutability in the small can help a lot to make efficient and powerful without endangering your code in the large.



To do that, you would also have to write a lot of case match on all your types and if you don't use a `sealed trait ADT`, the compiler won't even help you not to forget one case.

The idea: 
- use a coproduct to gather all incoming data types so you can manipulate a  Source[Coproduct]

- provide a list of flow for each type in the coproduct and dispatch every piece of data to the right flow according to its type.

- let the compiler build everything for us and check that we haven't missed any type in our crazy machine.


Sample

Errors

Limitations
- order not respected
- the slowest branch will slow down all other branches as in a broadcast. One can use Buffers to allow other branches to go on consuming




