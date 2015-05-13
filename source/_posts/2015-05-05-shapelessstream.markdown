---
layout: post
title: "ShapelessStream, when Akka-Stream meets Shapeless Coproduct at compile-time"
date: 2015-05-05 23:23
comments: true
external-url:
categories: [free monad, functional programming, category, algebra, scala, quadratic complexity, observability, map fusion, cats]
keywords: free monad,functional programming,category,algebra,scala,quadratic complexity,observability,map fusion,cats
---

You might have seen on my twitter that my current company [MFG Labs](http://www.mfglabs.com) has opensourced the library [Akka-Stream Extensions](http://mfglabs.github.io/akka-stream-extensions/). We have developed it with [Alexandre Tamborrino](http://twitter.com/altamborrino) and [Damien Pignaud](http://twitter.com/dmnpignaud) for our recent production projects based on [Typesafe Akka-Stream](http://doc.akka.io/docs/akka-stream-and-http-experimental/1.0-RC2/scala.html).

> In this article, I won't explain all the reasons that motivated our choice of Akka-Stream at MFG Labs and the road towards our library [Akka-Stream Extensions](https://mfglabs.github.com/akka-stream-extensions). Here, I'll focus on one precise aspect of our choice: `types`... And I'll tell you about a specific extension I've created for this project: `ShapelessStream`.


The code is [there on github](https://github.com/MfgLabs/akka-stream-extensions/tree/master/extensions/shapeless/src)

<br/>
<br/>
## Akka-Stream loves Types 

As you may know, I'm a `Type` lover. Types are proofs and proofs are types. Proofs are what you want in your code to ensure _in a robust & reliable way_ that it does what it pretends _with the support of the compiler_.

**First of all, let's remind that typesafety is a very interesting feature of Akka-Stream.**

Akka-Stream most basic primitive `Flow[A, B]` represents a `data-flow` that accepts elements of type `A` and will return elements of type `B`. You can't pass a `C` to it and you are sure that this flow won't return any `C` for example.

At MFG Labs, we have inherited some Scala legacy code mostly based on Akka actors which provide a very good way to handle failures but which are not typesafe at all (till Akka Typed) and not composable. Developers using Akka tend to scatter the business logic in the code and it can become hard to maintain. It has appeared that in many cases where Akka was used to transform data in a flow, call external services, Akka-Stream would be a very good way to replace those actors:

- better type-safety,
- fluent & composable code with great builder DSL
- same Akka failure management
- buffer, parallel computing, backpressure out-of-the-box

> Yes, it's quite weird to say it but Akka-Stream helped us correct most problems that had been introduced using Akka (rightly or wrongly).

<br/>
<br/>
## Multi-type flows

Ok, Akka-Stream promotes `Types` as first citizen in your data flows. That's cool!
But it appears that you often need to handle multiple types in the same input channel:

<img src="/images/mandubian/flow_11.png" />

When you control completely the types in input, you can represent input types by a classic ADT:

{% codeblock lang:scala %}
sealed trait A
case class A1(...) extends In
case class A2(...) extends In
case class A3(...) extends In
{% endcodeblock %}

... And manage it in `Flow[A, B]`:

{% codeblock lang:scala %}
Flow[A].map {
  case A1(...) => // some B
  case A2(...) => // some B
  case A3(...) => // some B
}
{% endcodeblock %}

Nice but you need to wrap all input types in an ADT and this involves some boring code that can even be different for every custom flow.

Going further, in general, you don't want to do that, you want to dispatch every element to a different flow according to its type:

<img src="/images/mandubian/flow_22.png" />

... and merge all results of all flows in one single channel...

... and every flow has its own behavior in terms of parallelism, bufferization, data generation and back-pressure...

In Akka-Stream, if you wanted to build a flow corresponding to the previous schema, you would have to use:

- a [FlexiRoute](http://doc.akka.io/docs/akka-stream-and-http-experimental/1.0-RC2/scala/stream-customize.html) for the input dispatcher
- a [FlexiMerge](http://doc.akka.io/docs/akka-stream-and-http-experimental/1.0-RC2/scala/stream-customize.html) for the output merger.

Have a look at the doc and see that it requires quite a bunch of lines to write one of those. It's really powerful but quite tedious to implement and not so typesafe after all. Moreover, you certainly would have to write one `FlexiRoute` and one `FlexiMerge` per use-case as the number of inputs types and return types depend on your context.


<br/>
<br/>
## Miles Sabin to the rescue

In my latest project, this `dispatcher/flows/merger` pattern was required in multiple places and as I'm lazy, I wanted something more elegant & typesafe if possible to build this kind of flow graphs.

Thinking in terms of pure types and from an external point of view, we can see the previous `dispatcher/flows/merger` flow graph in pseudo-code as:

{% codeblock lang:scala %}
Flow[
  Input = A1 or A2 or A3, // in input it accepts A1 or A2 or A3
  Ouput = B1 or B2 or B3  // in output it generates B1 or B2 or B3
]
{% endcodeblock %}

And to build the full flow graph, we need to provide a list of flows for all pairs of input/output types corresponding to our graph branches:

{% codeblock lang:scala %}
Flow[A1, B1] and Flow[A2, B2] and Flow[A3, B3]
{% endcodeblock %}


In [Shapeless](https://github.com/milessabin/shapeless), there are 2 very very very useful structures:

- `Coproduct` is a generalization of the well known `Either`. You have `A or B` in `Either[A, B]`. With `Coproduct`, you can have more than 2 alternatives `A or B or C or D`. So, for our previous external view of flow graph, using `Coproduct`, it could be written as:

{% codeblock lang:scala %}
Flow[
  A1 :+: A2 :+: A3 :+: CNil, 
  B1 :+: B2 :+: B3 :+: CNil
]
{% endcodeblock %}

- `HList` allows to build heterogenous `List` of elements keeping & tracking all types at compile time. For our previous list of flows, it would fit quite well as we want to match all input/output types of all flows. It would give:

{% codeblock lang:scala %}
Flow[A1, B1] :: Flow[A2, B2] :: Flow[A3, B3] :: HNil
{% endcodeblock %}

So, from an external point of view, the process of building our `dispatcher/flows/merger` flow graph looks like a `Function taking a `Hlist of flows` in input and returning the built `Flow of Coproducts`:

{% codeblock lang:scala %}
Flow[A1, B1] :: Flow[A2, B2] :: Flow[A3, B3] :: HNil =>
  Flow[A1 :+: A2 :+: A3 :+: CNil, B1 :+: B2 :+: B3 :+: CNil]
{% endcodeblock %}

Let's write it in terms of Shapeless Scala code:

{% codeblock lang:scala %}
/**
 * Builds at compile-time a fully typed-controlled flow that transforms a HList of Flows to a Flow of the Coproduct of inputs to Coproduct of outputs.
 *
 * @param a Hlist of flows Flow[A1, B1] :: FLow[A2, B2] :: ... :: Flow[An, Bn] :: HNil
 * @return the flow of the Coproduct of inputs and the Coproduct of outputs Flow[A1 :+: A2 :+: ... :+: An :+: CNil, B1 :+: B2 :+: ... +: Bn :+: CNil, Unit]
 */
def coproductFlow[HL <: HList, CIn <: Coproduct, COut <: Coproduct](
  flows: HL
): Flow[CIn, COut, Unit]
{% endcodeblock %}

Fantastic !!!

>Now the question is how can we build **at compile-time** this `Flow[CIn, COut, Unit]` from an `HList` of `Flows` and be sure that the compiler checks all links are correctly typed and all types are managed by the provided flows?


<br/>
<br/>
## Akka-Stream Graph Mutable builders

An important concept in Akka-Stream is the separation of concerns between:

- constructing/describing a data-flow
- materializing with live resources (like actor system)
- running the data-flow by plugging live sources/sinks on it (like web, file, hdfs, queues etc...).

> For the curious, you find the same idea in scalaz-stream but in a FP-purer way as scalaz-stream directly relies on `Free` concepts that formalize this idea quite directly.

Akka-Stream has taken a more custom way to respond to these requirements. To build complex data flows, it provides a very nice DSL described [here](http://doc.akka.io/docs/akka-stream-and-http-experimental/1.0-RC2/scala/stream-graphs.html). This DSL is based on the idea of a mutable structure used while building your graph until you decide to fix it definitely into an immutable structure.

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

`builder` is the mutable structure used to build the flow graph using the DSL inside the `{...}` block.

The value `g` is the immutable structure resulting from the builder that will later be materialized and run using live resources.

Please remark that once built, `g` value can reused and materialized/run several times, it is just the description of your flow graph.

> This idea of mutable builders is really interesting in general: mutability in the small can help a lot to make your building block efficient and easy to write/read without endangering immutability in the large.



<br/>
<br/>
## Hacking mutable builders with Shapeless

> My intuition was to _hack_ these mutable Akka-Stream builders using Shapeless type-dependent mechanics to build a Flow of Coproducts from an HList of Flows...

Let's show the real signature of `coproductFlow`:

{% codeblock lang:scala %}
def coproductFlow[HL <: HList, CIn <: Coproduct, COut <: Coproduct, CInOutlets <: HList, COutInlets <: HList](
  flows: HL
)(
  implicit
    flowTypes: FlowTypes.Aux[HL, CIn, COut],
    obuild: OutletBuilder.Aux[CIn, CInOutlets],
    ibuild: InletBuilder.Aux[COut, COut, COutInlets],
    otrav: ToTraversable.Aux[CInOutlets, List, Outlet[_]],
    itrav: ToTraversable.Aux[COutInlets, List, Inlet[COut]],
    selOutletValue: SelectOutletValue.Aux[CIn, CInOutlets],
    flowBuilder: FlowBuilderC.Aux[CIn, COut, CInOutlets, HL, COutInlets]
): Flow[CIn, COut, Unit]
{% endcodeblock %}

Frightening!!!!!!!

No, don't be, it's just the transcription in types of the requirements to build the full flow.

{% codeblock lang:scala %}
def coproductFlow[HL <: HList, CIn <: Coproduct, COut <: Coproduct, CInOutlets <: HList, COutInlets <: HList](
  flows: HL
)(
  implicit
    // 1 - Introspects HList of Flows to extract
    //      * all input types in CIn Coproduct
    //      * all output types in COut Coproduct
    flowTypes: FlowTypes.Aux[HL, CIn, COut],

    // 2 - Builds all Akka-Stream outlets branching the FlexiRoute
    //     outlets to the right Flow inlet in the HList
    obuild: OutletBuilder.Aux[CIn, CInOutlets],

    // 3 - Builds all Akka-Stream inlets branching the outlets of each Flow
    //     in the HList to the right inlet of FlexiMerge
    ibuild: InletBuilder.Aux[COut, COut, COutInlets],

    // 4 - Technical structures to be able to traverse
    //     and select all those previously built thingies
    otrav: ToTraversable.Aux[CInOutlets, List, Outlet[_]],
    itrav: ToTraversable.Aux[COutInlets, List, Inlet[COut]],
    selOutletValue: SelectOutletValue.Aux[CIn, CInOutlets],

    // 5 - The AkkaStream mutable Builder that:
    //      * builds the input FlexiRoute with the right output types
    //      * plugs all FlexiRoute outlets to the right flow inlet
    //      * builds the output FlexiMerge with the right input types
    //      * plugs all flow outlets to the right inlet of FlexiMerge
    flowBuilder: FlowBuilderC.Aux[CIn, COut, CInOutlets, HL, COutInlets]
): Flow[CIn, COut, Unit]
{% endcodeblock %}

> The Scala code might seem a bit ugly to a few of you. That's not false but keep in mind what we have done: mixing shapeless-style recursive implicit typeclass inference with the versatility of Akka-Stream mutable builders... And we were able to build our complex flow graph, to check all types and to plug all together **at compile-time**...

<br/>
<br/>
## Sample

{% codeblock lang:scala %}
// 1 - Create a type alias for your coproduct
type C = Int :+: String :+: Boolean :+: CNil

// The sink to consume all output data
val sink = Sink.fold[Seq[C], C](Seq())(_ :+ _)

// 2 - a sample source wrapping incoming data in the Coproduct
val f = FlowGraph.closed(sink) { implicit builder => sink =>
  import FlowGraph.Implicits._
  val s = Source(() => Seq(
    Coproduct[C](1),
    Coproduct[C]("foo"),
    Coproduct[C](2),
    Coproduct[C](false),
    Coproduct[C]("bar"),
    Coproduct[C](3),
    Coproduct[C](true)
  ).toIterator)

// 3 - our typed flows
  val flowInt = Flow[Int].map{i => println("i:"+i); i}
  val flowString = Flow[String].map{s => println("s:"+s); s}
  val flowBool = Flow[Boolean].map{s => println("s:"+s); s}
  
// >>>>>> THE IMPORTANT THING
// 4 - build the coproductFlow in a 1-liner
  val fr = builder.add(ShapelessStream.coproductFlow(flowInt :: flowString :: flowBool :: HNil))
// <<<<<< THE IMPORTANT THING

// 5 - plug everything together using akkastream DSL
  s ~> fr.inlet
       fr.outlet ~> sink
}

// 6 - run it
f.run().futureValue.toSet should equal (Set(
  Coproduct[C](1),
  Coproduct[C]("foo"),
  Coproduct[C](2),
  Coproduct[C](false),
  Coproduct[C]("bar"),
  Coproduct[C](3),
  Coproduct[C](true)
))
{% endcodeblock %}

> FYI, Shapeless Coproduct provides a lot of useful operations on Coproducts such as unifying all types or merging Coproducts together.


<br/>
<br/>
## Some compile errors now ?

Imagine you forget to manage one type of the Coproduct in the HList of flows:

{% codeblock lang:scala %}
...

// 4 - build the coproductFlow in a 1-liner
  val fr = builder.add(ShapelessStream.coproductFlow(flowInt :: flowString :: HNil))

..
{% endcodeblock %}

If you compile, it will produce this:

{% codeblock lang:scala %}
ShapelessExtensionsSpec.scala:97: overloaded method value ~> with alternatives:
[error]   (to: akka.stream.SinkShape[C])(implicit b: akka.stream.scaladsl.FlowGraph.Builder[_])Unit <and>
[error]   (to: akka.stream.Graph[akka.stream.SinkShape[C], _])(implicit b: akka.stream.scaladsl.FlowGraph.Builder[_])Unit <and>
[error]   [Out](flow: akka.stream.FlowShape[C,Out])(implicit b: akka.stream.scaladsl.FlowGraph.Builder[_])akka.stream.scaladsl.FlowGraph.Implicits.PortOps[Out,Unit] <and>
[error]   [Out](junction: akka.stream.UniformFanOutShape[C,Out])(implicit b: akka.stream.scaladsl.FlowGraph.Builder[_])akka.stream.scaladsl.FlowGraph.Implicits.PortOps[Out,Unit] <and>
[error]   [Out](junction: akka.stream.UniformFanInShape[C,Out])(implicit b: akka.stream.scaladsl.FlowGraph.Builder[_])akka.stream.scaladsl.FlowGraph.Implicits.PortOps[Out,Unit] <and>
[error]   [Out](via: akka.stream.Graph[akka.stream.FlowShape[C,Out],Any])(implicit b: akka.stream.scaladsl.FlowGraph.Builder[_])akka.stream.scaladsl.FlowGraph.Implicits.PortOps[Out,Unit] <and>
[error]   (to: akka.stream.Inlet[C])(implicit b: akka.stream.scaladsl.FlowGraph.Builder[_])Unit
[error]  cannot be applied to (akka.stream.Inlet[shapeless.:+:[Int,shapeless.:+:[String,com.mfglabs.stream.extensions.shapeless.FlowTypes.last.CIn]]])
[error]       s ~> fr.inlet
[error]         ^
[error] /Users/pvo/workspaces/mfg/akka-stream-extensions/extensions/shapeless/src/test/scala/ShapelessExtensionsSpec.scala:98: overloaded method value ~> with alternatives:
[error]   (to: akka.stream.SinkShape[shapeless.:+:[Int,shapeless.:+:[String,com.mfglabs.stream.extensions.shapeless.FlowTypes.last.COut]]])(implicit b: akka.stream.scaladsl.FlowGraph.Builder[_])Unit <and>
[error]   (to: akka.stream.Graph[akka.stream.SinkShape[shapeless.:+:[Int,shapeless.:+:[String,com.mfglabs.stream.extensions.shapeless.FlowTypes.last.COut]]], _])(implicit b: akka.stream.scaladsl.FlowGraph.Builder[_])Unit <and>
[error]   [Out](flow: akka.stream.FlowShape[shapeless.:+:[Int,shapeless.:+:[String,com.mfglabs.stream.extensions.shapeless.FlowTypes.last.COut]],Out])(implicit b: akka.stream.scaladsl.FlowGraph.Builder[_])akka.stream.scaladsl.FlowGraph.Implicits.PortOps[Out,Unit] <and>
[error]   [Out](junction: akka.stream.UniformFanOutShape[shapeless.:+:[Int,shapeless.:+:[String,com.mfglabs.stream.extensions.shapeless.FlowTypes.last.COut]],Out])(implicit b: akka.stream.scaladsl.FlowGraph.Builder[_])akka.stream.scaladsl.FlowGraph.Implicits.PortOps[Out,Unit] <and>
[error]   [Out](junction: akka.stream.UniformFanInShape[shapeless.:+:[Int,shapeless.:+:[String,com.mfglabs.stream.extensions.shapeless.FlowTypes.last.COut]],Out])(implicit b: akka.stream.scaladsl.FlowGraph.Builder[_])akka.stream.scaladsl.FlowGraph.Implicits.PortOps[Out,Unit] <and>
[error]   [Out](via: akka.stream.Graph[akka.stream.FlowShape[shapeless.:+:[Int,shapeless.:+:[String,com.mfglabs.stream.extensions.shapeless.FlowTypes.last.COut]],Out],Any])(implicit b: akka.stream.scaladsl.FlowGraph.Builder[_])akka.stream.scaladsl.FlowGraph.Implicits.PortOps[Out,Unit] <and>
[error]   (to: akka.stream.Inlet[shapeless.:+:[Int,shapeless.:+:[String,com.mfglabs.stream.extensions.shapeless.FlowTypes.last.COut]]])(implicit b: akka.stream.scaladsl.FlowGraph.Builder[_])Unit
[error]  cannot be applied to (sink.Shape)
[error]            fr.outlet ~> sink
[error]                      ^
{% endcodeblock %}

**OUCHHHH**, this is a mix of the worst error of Akka-Stream and the kind of errors you get with Shapeless :)

> Don't panic, breathe deep and just tell yourself that in this case, it just means that your types do not fit well

In general, the first line and the last lines are the important ones.

#### For input:

{% codeblock lang:scala %}
ShapelessExtensionsSpec.scala:97: overloaded method value ~> with alternatives:
[error]   (to: akka.stream.SinkShape[C])(implicit b: akka.stream.scaladsl.FlowGraph.Builder[_])Unit <and>
[error]  cannot be applied to (akka.stream.Inlet[shapeless.:+:[Int,shapeless.:+:[String,com.mfglabs.stream.extensions.shapeless.FlowTypes.last.CIn]]])
[error]       s ~> fr.inlet
[error]         ^
{% endcodeblock %}

It just means you try to plug a `C == Int :+: String :+: Bool :+: CNil` to a `Int :+: String :+: CNil` and the compiler is angry against you!!

#### For output:

{% codeblock lang:scala %}
ShapelessExtensionsSpec.scala:98: overloaded method value ~> with alternatives:
[error]   (to: akka.stream.SinkShape[shapeless.:+:[Int,shapeless.:+:[String,com.mfglabs.stream.extensions.shapeless.FlowTypes.last.COut]]])(implicit b: akka.stream.scaladsl.FlowGraph.Builder[_])Unit <and>
[error]   (to: akka.stream.Inlet[shapeless.:+:[Int,shapeless.:+:[String,com.mfglabs.stream.extensions.shapeless.FlowTypes.last.COut]]])(implicit b: akka.stream.scaladsl.FlowGraph.Builder[_])Unit
[error]  cannot be applied to (sink.Shape)
[error]            fr.outlet ~> sink
{% endcodeblock %}

It just means you try to plug a `Int :+: String :+: CNil` to a `C == Int :+: String :+: Bool :+: CNil` and the compiler is 2X-angry against you!!!

<br/>
<br/>
## Conclusion

Mixing the power of Shapeless compile-time type dependent structures and Akka-Stream mutable builders, we are able to build at compile-time a complex `dispatcher/flows/merger` flow graph that checks all types and all flows correspond to each other and plugs all together in a 1-liner...

This code is the first iteration on this principle but it appeared to be so efficient and I trusted the mechanism so much (nothing happens at runtime but just at compile-time) that I've put that in production two weeks ago. __It runs like a charm.__

Finally, there are a few specificities/limitations to know:

- Wrapping input data into the `Coproduct` is still the boring part with some pattern matching potentially. But this is like Json/Xml validation, you need to validate only the data you expect. Yet I expect to reduce the work soon by providing a Scala macro that will generate this part for you as it's just mechanical...

- Wrapping everything in `Coproduct` could have some impact on performance if what you expect is pure performance but in my use-cases IO are so much more impacting that this is not a problem...

- `coproductFlow` is built with a custom FlexiRoute with `DemandFromAll` condition & FlexiMerge using `ReadAny` condition. This implies :

    * the order is NOT guaranteed due to the nature of used FlexiRoute & FlexiMerge and potentially to the flows you provide in your HList (each branch flow has its own parallelism/buffer/backpressure behavior and is not necessarily a 1-to-1 flow).

    * the slowest branch will slow down all other branches (as with a broadcast). _To manage these issues, you can add buffers in your branch flows to allow other branches to go on pulling input data_


The future?

- A macro generating the Coproduct wrapping flow

- Some other flows based on Shapeless

Have more backpressured and typed fun...


