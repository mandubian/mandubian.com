---
layout: post
title: "Understanding Play2 Iteratees for Normal Humans"
date: 2012-08-27 11:17
comments: true
external-url: 
categories: [iteratee,enumerator,enumeratee,play2,scala,web,asynchronous,non-blocking,reactive]
keywords: play2, play framework, scala, web, realtime web, iteratee, enumerator, enumeratee, asynchronous, non-blocking, reactive, data flow
---

> Thanks to [@loic_d](http://www.twitter.com/loic_d), we have a French translation of this article [there](https://gist.github.com/3727850#file_iteratees_humains.md)

#### You may have remarked that [Play2](http://www.playframework.org) provides an intriguing feature called _`Iteratee`_ (and its counterparts _`Enumerator`_ and _`Enumeratee`_).  
#### The main aim of this article is **_(to try)_ to make the `Iteratee` concept understandable for most of us** with reasonably simple arguments and without functional/math theory.  

<br/>
> This article is not meant to explain everything about `Iteratee` / `Enumerator` / `Enumeratee` but just the ideas behind it.  
> I'll try to write another article to show practical samples of coding with Iteratee/Enumerator/Enumeratee.

# <a name="intro">Introduction</a>

> **In [Play2 doc](http://www.playframework.org/documentation/2.0.2/Iteratees), `Iteratees` are presented as a great tool to handle data streams reactively in a non-blocking, generic & composable way for modern web programming in distributed environments.**  
> 
> _Seems great, isn't it?  
> But what is an `Iteratee` exactly?  
> What is the difference between the `Iteratee` and the classic `Iterator` you certainly know?  
> Why use it? In which cases?  
> A bit obscure and complex, isn't it?_

<div class = "well">
<h3>If you are lazy and want to know just a few things</h3>
<ul>
  <li><code>Iteratee</code> is an <b>abstraction of iteration over chunks of data in a non-blocking and asynchronous way</b></li>
  <li><code>Iteratee</code> is able to <b>consume chunks of data of a given type from a producer</b> generating data chunks of the same type called <code>Enumerator</code>
  <li><code>Iteratee</code> can <b>compute a progressive result from data chunks over steps of iteration</b> (an incremented total for ex)
  <li><code>Iteratee</code> is a <b>thread-safe and immutable object that can be re-used over several `Enumerators`</b></li>
</ul>
</div>

### 1st advice: DO NOT SEARCH ABOUT ITERATEES ON GOOGLE

When you search on Google for `Iteratee`, you find very obscure explanations based on pure functional approach or even mathematical theories. Even the documentation on Play Framework ([there](http://www.playframework.org/documentation/2.0.2/Iteratees)) explains `Iteratee` with a fairly _low-level_ approach which might be hard for beginners...

As a beginner in Play2, it might seem a bit tough to handle `Iteratee` concept presented in a really abstract way of manipulating data chunks.  
It might seem so complicated that you will occult it and won't use it.  
It would be a shame because Iteratees are so powerful and provide a really interesting and new way to manipulate your data flows in a web app.

So, let's try to explain things in a simple way. I don't pretend to be a theoretical expert on those functional concepts and I may even say wrong things but I want to write an article that reflects what `Iteratee` means for me. Hope this could be useful to somebody...

> This article uses **Scala** for code samples but the code should be understandable by anyone having a few notions of coding and I promise not to use any weird operator (_but in last paragraph_) **><> ><> ><> ><>**

> The code samples are based on incoming **Play2.1 master** code which greatly simplifies and rationalizes the code of `Iteratee`. So don't be surprised if API doesn't look like Play2.0.x sometimes



# <a name="reminders">Reminders about iteration</a>

Before diving into deep `Iteratee` sea, I want to clarify what I call **iteration** and to try to go progressively from the concept of `Iterator` to `Iteratee`.  

#### You may know the `Iterator` concept you can find in Java. An `Iterator` allows to run over a collection of elements and then do something at each step of iteration. Let's begin with a very simple iteration in _Java classic_ way that sums all the integers of a `List[Int]`

<br/>
### <a name="reminders-1">The first very naive implementation with _Java-like_ `Iterator`</a>

{% codeblock lang:scala %}
val l = List(1, 234, 455, 987)

var total = 0 // will contain the final total
var it = l.iterator
while( it.hasNext ) {
  total += it.next
}

total
=> resXXX : Int = 1677
{% endcodeblock %}

Without any surprise, _iterating over a collection_ means :

- Get an iterator from the collection,  
- Get an element from the iterator (if there are any),  
- Do something : _here add the element value to the total_,  
- If there are other elements, go to the next element,  
- Do it again,  
- Etc... till there are no more element to consume in the iterator

<div class="well">
While iterating over a collection, we manipulate:
<ul>
  <li><strong>A state of iteration</strong> (is iteration finished ? This is naturally linked to the fact that there are more elements or not in the iterator?)</li>
  <li><strong>A context</strong> updated from one step to the next (the total)</li>
  <li><strong>An action</strong> updating the context</li>
</ul>
</div>


### <a name="reminders-2">Rewrite that using Scala _for-comprehension_</a>

{% codeblock lang:scala %}
for( item <- l ) { total += item }
{% endcodeblock %}

It's a bit better because you don't have to use the iterator.

### <a name="reminders-3">Rewrite it in a more functional way</a>

{% codeblock lang:scala %}
l.foreach{ item => total += item }
{% endcodeblock %}

Here we introduce `List.foreach` function accepting an anonymous function `(Int => Unit)` as a parameter and iterating over the list: for each element in the list, it calls the function which can update the context (_here the total_). 

The anonymous function contains the action executed at each loop while iterating over the collection.


### <a name="reminders-3">Rewrite in a more generic way</a>

The anonymous function could be stored in a variable so that it can be reused in different places.

{% codeblock lang:scala %}
val l = List(1, 234, 455, 987)
val l2 = List(134, 664, 987, 456)
var total = 0
def step(item: Int) = total += item
l foreach step

total = 0
l2 foreach step
{% endcodeblock %}

> You should then say to me: _"This is ugly design, your function has side-effects and uses a variable which is not a nice design at all and you even have to reset total to 0 at second call!"_

That's completely true: 

> **Side-effect functions are quite dangerous** because they change the state of something that is external to the function. This state is not exclusive to the function and can be changed by other entities, potentially in other threads. Function with side-effects are not recommended to have clean and robust designs and functional languages such as Scala tend to reduce side-effects functions to the strict necessary (IO operations for ex).

> **Mutable variables are also risky** because if your code is run over several threads, if 2 threads try to change the value of the variable, who wins? In this case, you need synchronization which means blocking threads while writing the variable which means breaking one of the reason of being of Play2 (non-blocking web apps)...

### <a name="reminders-4">Rewrite the code in an immutable way without side-effects</a>

{% codeblock lang:scala %}
def foreach(l: List[Int]) = {
  def step(l: List[Int], total: Int): Int = {
    l match {
      case List() => total
      case List(elt) => total + elt
      case head :: tail => step(tail, total + head)
    }
  }

  step(l, 0)
}

foreach(l)
{% endcodeblock %}

####A bit more code, isn't it ?
####But please notice at least:

- `var total` disappeared.
- `step` function is the action executed at each step of iteration but it does something more than before: `step` also manages the state of the iteration. It executes as following:
  * If the list is empty, return current `total`  
  * If the list has 1 element, return `total + elt`  
  * If the list has more than 1 element, calls `step` with the tail elements and the new total `total + head`

So at each step of iteration, depending on the result of previous iteration, `step` can choose between 2 states:

- Continue iteration because it has more elements  
- Stop iteration because it reached end of list or no element at all  

####Notice also that :

- **`step` is a _tail-recursive_ function** (doesn't unfold the full call stack at the end of recursion and returns immediately) preventing from stack overflow and behaving almost like the previous code with `Iterator`  
- **`step` transmits the remaining elements of the list & the new total** to the next step  
- **`step` returns the total without any side-effects** at all  

>So, yes, this code consumes a bit more memory because it re-copies some parts of the list at each step (only the references to the elements) but it has no side-effect and uses only immutable data structures. 
>This makes it very robust and distributable without any problem.

Notice you can write the code in a very shorter way using the wonderful functions provided by Scala collections:

{% codeblock lang:scala %}
l.foldLeft(0){ (total, elt) => total + elt }
{% endcodeblock %}


<div class="well">
<h3><a name="reminders-milestone">Milestone</a></h3>
<br/>
In this article, I consider iteration based on immutable structures propagated over steps.
From this point of view, iterating involves:
<ul>
<li>receiving information from the previous step: context & state</li>
<li>getting current/remaining element(s)</li>
<li>computing a new state & context from remaining elements</li>
<li>propagating the new state & context to next step</li>
</ul>

</div>

---------------------------------------

# <a name="step-by-step">Step by Step to _Iterator_ & _Iteratees_</a>

Now that we are clear about iteration, let's go back to our `Iteratee`!!!

#### Imagine you want to generalize the previous iteration mechanism and be able to write something like:

{% codeblock lang:scala %}
def sumElements(...) = ...
def prodElements(...) = ...
def printElements(...) = ...
l.iterate(sumElements)
l.iterate(prodElements)
l.iterate(printElements)
{% endcodeblock %}

> Yes I know, with Scala collection APIs, you can do many things :)

#### Imagine you want to compose a first iteration with another one:

{% codeblock lang:scala %}
def groupElements(...) = ...
def printElements(...) = ...
l.iterate(groupElements).iterate(printElements)
{% endcodeblock %}

#### Imagine you want to apply this iteration on something else than a collection: 

- a stream of data produced progressively by a file, a network connection, a database connection,  
- a data flow generated by an algorithm,  
- a data flow from an asynchronous data producer such as a scheduler or an actor.

**Iteratees are exactly meant for this...**

Just to tease, here is how you would write the previous sum iteration with an `Iteratee`.

{% codeblock lang:scala %}
val enumerator = Enumerator(1, 234, 455, 987)
enumerator.run(Iteratee.fold(0){ (total, elt) => total + elt }
{% endcodeblock %}

Ok, it looks like the previous code and doesn't seem to do much more...  
Not false but trust me, it can do much more.  
At least, it doesn't seem so complicated?

But, as you can see, `Iteratee` is used with `Enumerator` and both concepts are tightly related.  

**Now let's dive into those concepts on a step by step approach.**

<br/>

## <a name="step-by-step-enumerator">><> About Enumerator ><></a>

### `Enumerator` is a more generic concept than collections or arrays

Till now, we have used collections in our iterations. But as explained before, we could iterate over something more generic, simply being able to produce simple chunks of data available immediately or asynchronously in the future.  

`Enumerator` is designed for this purpose.

A few examples of simple Enumerators:

{% codeblock lang:scala %}
// an enumerator of Strings
val stringEnumerator: Enumerator[String] = Enumerate("alpha", "beta", "gamma")
// an enumerator of Integers
val integerEnumerator: Enumerator[Int] = Enumerate(123, 456, 789)
// an enumerator of Doubles
val doubleEnumerator: Enumerator[Double] = Enumerate(123.345, 456.543, 789.123)

// an Enumerator from a file
val fileEnumerator: Enumerator[Array[Byte]] = Enumerator.fromFile("myfile.txt")

// an Enumerator generated by a callback
// it generates a string containing current time every 500 milliseconds
// notice (and forget for the time being) the Promise.timeout which allows non-blocking mechanism
val dateGenerator: Enumerator[String] = Enumerator.generateM(
  play.api.libs.concurrent.Promise.timeout(
    Some("current time %s".format((new java.util.Date()))), 
    500
  )
)
{% endcodeblock %}

### `Enumerator` is a *PRODUCER* of statically typed chunks of data.
`Enumerator[E]` produces chunks of data of type `E` and can be of the 3 following kinds:

- `Input[E]` is a chunk of data of type E : _for ex, Input[Pizza] is a chunk of Pizza_.
- `Input.Empty` means the enumerator is empty : _for ex, an Enumerator streaming an empty file_.
- `Input.EOF` means the enumerator has reached its end : _for ex, Enumerator streaming a file and reaching the end of file_.

_You can draw a parallel between the kinds of chunks and the states presented above (has more/no/no more elements)._

Actually, `Enumerator[E]` contains `Input[E]` so you can put an `Input[E]` in it:

{% codeblock lang:scala %}
// create an enumerator containing one chunk of pizza
val pizza = Pizza("napolitana")
val enumerator: Enumerator[Pizza] = Enumerator.enumInput(Input.el(pizza))

// create an enumerator containing no pizza
val enumerator: Enumerator[Pizza] = Enumerator.enumInput(Input.Empty)
{% endcodeblock %}



### `Enumerator` is a non-blocking producer

The idea behind Play2 is, as you may know, to be fully non-blocking and asynchronous. Thus, `Enumerator`/`Iteratee` reflects this philosophy.
The `Enumerator` produces chunks in a completely asynchronous and non-blocking way. This means the concept of `Enumerator` is not by default related to an active process or a background task generating chunks of data.

Remember the code snippet above with `dateGenerator` which reflects exactly the asynchronous and non-blocking nature of `Enumerator`/`Iteratee`?
{% codeblock lang:scala %}
// an Enumerator generated by a callback
// it generates a string containing current time every 500 milliseconds
// notice the Promise.timeout which provide a non-blocking mechanism
val dateGenerator: Enumerator[String] = Enumerator.generateM(
  play.api.libs.concurrent.Promise.timeout(
    Some("current time %s".format((new java.util.Date()))), 
    500
  )
)
{% endcodeblock %}

> **What's a Promise?**  
> 
> It would require a whole article but let's say the name corresponds exactly to what it does.  
> A `Promise[String]` means : "_It will provide a String in the future (or an error)_", that's all. Meanwhile, it doesn't block current thread and just releases it.   

### `Enumerator` requires a consumer to produce

Due to its non-blocking nature, if nobody consumes those chunks, the `Enumerator` doesn't block anything and doesn't consume any hidden runtime resources.  
So, **`Enumerator` _MAY_ produce chunks of data only if there is someone to consume them**.

<br/>

<div class="well">
So what consumes the chunks of data produced by <code>Enumerator</code>?<br/>
You have deduced it yourself: the <code>Iteratee</code>
</div> 

<br/>
<br/>
## <a name="step-by-step-iteratee">><> About Iteratee ><></a>

### `Iteratee` is a generic _"stuff"_ that can iterate over an `Enumerator`

Let's be windy for one sentence: 
>`Iteratee` is the generic translation of the concept of iteration in pure functional programming.  
> While `Iterator` is built from the collection over which it will iterate, `Iteratee` is a generic entity that waits for an `Enumerator` to be iterated over.  

Do you see the difference between `Iterator` and `Iteratee`? No? Not a problem… Just remember that:

- an `Iteratee` is a generic entity that can _iterate_ over the chunks of data produced by an `Enumerator` (or something else)  
- an `Iteratee` is created independently of the `Enumerator` over which it will iterate and the `Enumerator` is provided to it  
- an `Iteratee` is immutable, stateless and fully reusable for different enumerators  

That's why we say: 
> An `Iteratee` is applied on an `Enumerator` or run over an `Enumerator`.  

Do you remember the example above computing the total of all elements of an Enumerator[Int] ?  
Here is the same code showing that an `Iteratee` can be created once and reused several times on different Enumerators.

{% codeblock lang:scala %}
val iterator = Iteratee.fold(0){ (total, elt) => total + elt }

val e1 = Enumerator(1, 234, 455, 987)
val e2 = Enumerator(345, 123, 476, 187687)

// we apply the iterator on the enumerator
e1(iterator)		// or e1.apply(iterator)
e2(iterator)

// we run the iterator over the enumerator to get a result
val result1 = e1.run(iterator)	// or e1 run iterator
val result2 = e2.run(iterator)
{% endcodeblock %}

> *`Enumerator.apply` and `Enumerator.run` are slightly different functions and we will explain that later.*

### `Iteratee` is an active consumer of chunks of data

By default, the `Iteratee` awaits a first chunk of data and immediately after, it launches the iteration mechanism. 
The `Iteratee` goes on consuming data until it considers it has finished its computation.  
Once initiated, the `Iteratee` is fully responsible for the full iteration process and decides when it stops.

{% codeblock lang:scala %}
// creates the iteratee
val iterator = Iteratee.fold(0){ (total, elt) => total + elt }

// creates an enumerator
val e = Enumerator(1, 234, 455, 987)

// this injects the enumerator into the iteratee 
// = pushes the first chunk of data into the iteratee
enumerator(iterator)
// the iteratee then consumes as many chunks as it requires
// don't bother about the result of this, we will explain later
{% endcodeblock %}

As explained above, the `Enumerator` is a producer of chunks of data and it expects a consumer to consume those chunks of data.  
To be consumed/iterated, the `Enumerator` has to be injected/plugged into an `Iteratee` or more precisely the first chunk of data has to be 
injected/pushed into the `Iteratee`.  
Naturally the `Iteratee` is dependent on speed of production of `Enumerator`: if it's slow, the `Iteratee` is also slow.

> Notice the relation Iteratee/Enumerator can be considered with respect to inversion of control and dependency injection pattern.


### `Iteratee` is a "_1-chunk-loop_" function

The `Iteratee` consumes chunks one by one until it considers it has ended iteration.  
Actually, the real scope of an `Iteratee` is limited to the treatment of one chunk.
That's why it can be defined as a function being able to consume one chunk of data.


### `Iteratee` accepts static typed chunks and computes a static typed result

Whereas an `Iterator` iterates over chunks of data coming from the collection that created it, an `Iteratee` is a bit more ambitious : it can 
compute something meanwhile it consumes chunks of data.

That's why the signature of Iteratee is :
{% codeblock lang:scala %}
trait Iteratee[E, +A] 
// E is the type of data contained in chunks. So it can only be applied on a Enumerator[E]
// A is the result of the iteration
{% endcodeblock %}

Let's go back to our first sample : compute the total of all integers produced by an `Enumerator[Int]`:
{% codeblock lang:scala %}
// creates the iteratee
val iterator = Iteratee.fold(0){ (total, elt) => total + elt }
val e = Enumerator(1, 234, 455, 987)

// runs the iteratee over the enumerator and retrieve the result
val total: Promise[Int] = enumerator run iterator
{% endcodeblock %}

> Notice the usage of `run`: You can see that the result is not the total itself but a `Promise[Int]` of the total because we are in an asynchronous world.  
> To retrieve the real total, you could use scala concurrent blocking `Await._` functions. But this is NOT good because it's a blocking API. As Play2 is fully async/non-blocking, the best practice is to 
> propagate the promise using `Promise.map/flatMap`.

But a result is not mandatory. For ex, let's just println all consumed chunks:
 
{% codeblock lang:scala %}
// creates the iteratee
val e = Enumerator(1, 234, 455, 987)
e(Iteratee.foreach( println _ ))
// or
e.apply(Iteratee.foreach( println _ ))
// yes here the usage of _ is so trivial that you shall use it
{% endcodeblock %}

The result is not necessarily a primitive type, it can just be the concatenation of all chunks into a List for ex:

{% codeblock lang:scala %}
val enumerator = Enumerator(1, 234, 455, 987)

val list: Promise[List[Int]] = enumerator run Iteratee.getChunks[Int]
{% endcodeblock %}


###  `Iteratee` can propagate the immutable context & state over iterations

To be able to compute final total, the `Iteratee` needs to propagate the partial totals along iteration steps.  
This means the `Iteratee` is able to receive a context (_the previous total for ex_) from the previous step, then compute the new context with current chunk of data 
(_new total = previous total + current element_) and can finally propagate this context to the next step (if there need to be a next step). 


###  `Iteratee` is simply a state machine

Ok this is cool but how does the `Iteratee` know it has to stop iterating?  
What happens if there were an error/ EOF or it has reached the end of `Enumerator`?  
Therefore, in addition to the context, the `Iteratee` should also receive previous state, decides what to do and potentially computes the new state to be sent to next step.

Now, remember the classic iteration states described above. For `Iteratee`, there are almost the same 2 possible states of iteration:

- State `Cont` : the iteration can continue with next chunk and potentially compute new context  
- State `Done` : it signals it has reached the end of its process and can return the resulting context value  

and a 3rd one which seems quite logical:

- State `Error` : it signals there was an Error during current step and stops iterating

> **From this point of view, we can consider the `Iteratee` is just a state machine in charge of looping over state `Cont` until it detects conditions to switch to terminal states `Done` or `Error`.**

### `Iteratee` states `Done/Error/Cont` are also `Iteratee`

Remember, the `Iteratee` is defined as a 1-chunk-loop function and it's main purpose is to change from one state to another one.
Let's consider those states are also `Iteratee`.

We have 3 "_State_" Iteratees:

`Done[E, A](a: A, remaining: Input[E])`

- `a:A` the context received from previous step  
- `remaining: Input[E]` representing the next chunk  

<br/>
`Error[E](msg: String, input: Input[E])`

Very simple to understand also: an error message and the input on which it failed.
<br/>

`Cont[E, A](k: Input[E] => Iteratee[E, A])`

This is the most complicated State as it's built from a function taking an `Input[E]` and returning another `Iteratee[E,A]`.
Without going too deep in the theory, you can easily understand that `Input[E] => Iteratee[E, A]` is simply a good way to consume one input and return a new state/iteratee 
which can consume another input and return another state/iteratee etc… till reaching state Done or Error.  
This construction ensures feeding the iteration mechanism (in a typical functional way). 

Ok lots of information, isn't it?
You certainly wonder why I explain of all of that?
This is just because if you understand that, you will understand how to create an custom `Iteratee`.

Let's write an `Iteratee` computing the total of the 2 first elements in an `Enumerator[Int]` to show an example.

{% codeblock lang:scala %}
// Defines the Iteratee[Int, Int]
def total2Chunks: Iteratee[Int, Int] = {
  // `step` function is the consuming function receiving previous context (idx, total) and current chunk
  // context : (idx, total) idx is the index to count loops
  def step(idx: Int, total: Int)(i: Input[Int]): Iteratee[Int, Int] = i match {
    // chunk is EOF or Empty => simply stops iteration by triggering state Done with current total
    case Input.EOF | Input.Empty => Done(total, Input.EOF)
    // found one chunk 
    case Input.El(e) =>
      // if first or 2nd chunk, call `step` again by incrementing idx and computing new total
      if(idx < 2) Cont[Int, Int](i => step(idx+1, total + e)(i))
      // if reached 2nd chunk, stop iterating
      else Done(total, Input.EOF)
  }
  
  // initiates iteration by initialize context and first state (Cont) and launching iteration
  (Cont[Int, Int](i => step(0, 0)(i)))  
}

// Using it
val promiseTotal = Enumerator(10, 20, 5) run total2Chunks
promiseTotal.map(println _) 
=> prints 30
{% endcodeblock %}

> **With this example, you can understand that writing an Iteratee is not much different than choosing what to do at each step depending on the type of Chunk you received
 and returning the new `State/Iteratee`.**

<br/>
<br/>

## <a name="candies">A few candy for those who did not drown yet</a>

### `Enumerator` is just a helper to deal with `Iteratee`

As you could see, in `Iteratee` API, there is nowhere any mention about `Enumerator`.  
This is just because `Enumerator` is just a helper to interact with `Iteratee`: it can plug itself to `Iteratee` and injects the first chunk of data into it.  
But you don't need `Enumerator` to use `Iteratee` even if this is really easier and well integrated everywhere in Play2.

<br/>

### Difference between `Enumerator.apply(Iteratee)` and `Enumerator.run(Iteratee)`

Let's go back to this point evoked earlier.
Have a look at the signature of main APIs in `Enumerator`:

{% codeblock lang:scala %}
trait Enumerator[E] {

  def apply[A](i: Iteratee[E, A]): Promise[Iteratee[E, A]]
  ...
  def run[A](i: Iteratee[E, A]): Promise[A] = |>>>(i)
}
{% endcodeblock %}

#### `apply` returns last Iteratee/State

The `apply` function injects the `Enumerator` into the `Iteratee` which consumes the chunks, does its job and returns a Promise of `Iteratee`. 
From previous explanation, you may deduce by yourself that the returned `Iteratee` might simply be the last state after it has finished consuming the chunks it required from `Enumerator`.

#### `run` returns a Promise[Result]

`run` has 3 steps:

1. Call previous `apply` function
2. Inject `Input.EOF` into `Iteratee` to be sure it has ended  
3. Get the last context from `Iteratee` as a promise.

Here is an example:

{% codeblock lang:scala %}
// creates the iteratee
val iterator = Iteratee.fold(0){ (total, elt) => total + elt }
val e = Enumerator(1, 234, 455, 987)

// just lets the iterator consume all chunks but doesn't require result right now
val totalIteratee: Promise[Iteratee[Int, Int]] = enumerator apply iterator

// runs the iteratee over the enumerator and retrieves the result as a promise
val total: Promise[Int] = enumerator run iterator

{% endcodeblock %}


<div class="well">
<h3>To Remember</h3>
<h4>When you need the result of <code>Iteratee</code>, you shall use <code>run</code></h4>
<h4>When you need to apply an <code>Iteratee</code> over an <code>Enumerator</code> without retrieving the result, you shall use <code>apply</code></h4>
</div>

<br/>

### `Iteratee` is a Promise[Iteratee] (_IMPORTANT TO KNOW_)

One more thing to know about an Iteratee is that **Iteratee is a Promise[Iteratee]** by definition.

{% codeblock lang:scala %}
// converts a Promise[Iteratee] to Iteratee
val p: Promise[Iteratee[E, A]] = ...    
val it: Iteratee[E, A] = Iteratee.flatten(p)

// converts an Iteratee to a Promise[Iteratee]
// pure promise
val p1: Promise[Iteratee[E, A]] = Promise.pure(it)
// using unflatten
val p2: Promise[Iteratee[E, A]] = it.unflatten.map( _.it ) 
// unflatten returns a technical structure called Step wrapping the Iteratee in _.it
{% endcodeblock %}

<div class="well">
<h3><code>Iteratee</code> <=> <code>Promise[Iteratee]</code></h3>
<h4>This means that you can build your code around Iteratee in a very lazy way : with Iteratee, you can switch to Promise and back as you want.</h4>
</div>
    

<br/>

---------------------------------------

## <a name="enumeratee">Final words about _Enumeratee_</a>

> You discovered `Iteratee`, then `Enumerator`…  
> And now you come across this…  `Enumeratee`???  
> What is that new stuff in `XXXtee` ?????


### 2nd advice : DON'T PANIC NOW… `Enumeratee` concept is really simple to understand


<div class="well">
<h3><code>Enumeratee</code> is just a <i>pipe adapter</i> between <code>Enumerator</code> and <code>Iteratee</code></h3>
</div>
<br/>
Imagine you have an `Enumerator[Int]` and an `Iteratee[String, Lis[String]]`.  
You can transform an `Int` into a `String`, isn't it?  
So you should be able to transform the chunks of `Int` into chunks of `String` and then inject them into the Iteratee.

Enumeratee is there to save you.

{% codeblock lang:scala %}
val enumerator = Enumerator(123, 345, 456)
val iteratee: Iteratee[String, List[String]] = …

val list: List[String] = enumerator through Enumeratee.map( _.toString ) run iteratee
{% endcodeblock %}

What happened there?

**You just piped `Enumerator[Int]` through and `Enumeratee[Int, String]` into `Iteratee[String, List[String]]`**

In 2 steps:
{% codeblock lang:scala %}
val stringEnumerator: Enumerator[String] = enumerator through Enumeratee.map( _.toString )
val list: List[String] = stringEnumerator run iteratee
{% endcodeblock %}

So, you may understand that `Enumeratee` is a very useful tool to convert your custom `Enumerator` to be used with generic `Iteratee` provided by Play2 API.  
You'll see that this is certainly the tool you will use the most while coding with `Enumerator` / `Iteratee`.

### `Enumeratee` can be applied to an `Enumerator` without `Iteratee`

This is a very useful feature of `Enumeratee`.
You can transform Enumerate[From] into Enumerator[To] with an Enumeratee[From, To]

Signature of `Enumeratee` is quite explicit:
{% codeblock lang:scala %}
Enumeratee[From, To]
{% endcodeblock %}

So you can use it as following:

{% codeblock lang:scala %}
val stringEnumerator: Enumerator[String] = enumerator through Enumeratee.map( _.toString )
{% endcodeblock %}


### `Enumeratee` can transform an `Iteratee`

This is a bit stranger feature because you can transform an `Iteratee[To, A]` to an `Iteratee[From, A]` with `Enumeratee[From, To]`

{% codeblock lang:scala %}
val stringIteratee: Iteratee[String, List[String]] = …

val intIteratee: Iteratee[Int, List[String]] = Enumeratee.map[Int, String]( _.toString ) transform stringIteratee
{% endcodeblock %}


### `Enumeratee` can be composed with an `Enumeratee`

Yes, this is the final very useful feature of `Enumeratee`.

{% codeblock lang:scala %}
val enumeratee1: Enumeratee[Type1, Type2] = …
val enumeratee2: Enumeratee[Type2, Type3] = …

val enumeratee3: Enumeratee[Type1, Type3] = enumeratee1 compose enumeratee2
{% endcodeblock %}

So once again, very easy to see that you can create your generic `Enumeratees` and then compose them into the custom `Enumeratee` you need for your custom `Enumerator` / `Iteratee`.

---------------------------------------

## Conclusion

Now I hope you have a bit more information and are not lost anymore.  
Next step is to use `Iteratee` / `Enumerator` / `Enumeratee` all together.  
I'll write other articles presenting more specific and practical ideas and concepts and samples…  
There are a lot of interesting features that are worth precise explanations.  
Understanding clearly what's an `Iteratee` is important because it helps writing new `Iteratees` but you can also stay superficial and use the many helpers provided by Play2 Iteratee API.  

_Ok, documentation is not yet as complete as it should but we are working on this!!!_

<div class="well">
<h3>Anyway, why should I use <code>Iteratee</code> / <code>Enumerator</code> / <code>Enumeratee</code> ?</h3>
<p>I want to tell you that <code>Iteratee</code> / <code>Enumerator</code> / <code>Enumeratee</code> is not a funny tool for people found of functional constructions.  
They are useful in many domains and once you will understand how they work, I can promise you that you will begin to use it more and more.</p>
<p>Modern web applications are not only dynamically generated pages anymore. Now you manipulate flows of data coming from different sources, in different formats, with different availability timing.
You may have to serve huge amount of data to huge number of clients and to work in distributed environments.</p>
<p><code>Iteratee</code> are made for those cases because there are safe, immutable and very good to deal with data flows in realtime.
Let's tell the buzzword you can see more & more <i>"Realtime WebApp"</i> and <code>Iteratee</code> is associated to that ;)</p>
</div>

> ####Note on weird operators
> You will certainly see lots of those operators in code based on `Iteratee` / `Enumerator` / `Enumeratee` such as `&>`, `|>>`, `|>>>` and the famous fish operator `><>`.
> Don't focus on those operators right now, there are just aliases of real explicit words such as `through`, `apply`, `applyOn` or `compose`.
> I'll try to write an article about those operators to demystify them. With practice, some people will find the code with operators clearer and more compact, some people will prefer words.



Have fun 