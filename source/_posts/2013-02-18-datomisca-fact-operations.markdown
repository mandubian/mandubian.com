---
layout: post
title: "Datomisca Delicatessen - The Scala Reactive Cherry on the Datomic Cake of Facts"
date: 2013-02-18 14:14
comments: true
external-url: 
categories: [datomic,datomisca,datalog,datom,database,scala,reactive,asynchronous,non-blocking,future,execution context]
keywords: datomic,datomisca,datalog,datom,scala,reactive,asynchronous,non-blocking,future,execution context
---


Let's go on unveiling [Datomisca](http://pellucidanalytics.github.com/datomisca/index.html) a bit more. 

Remember Datomisca is an opensource Scala API (sponsored by [Pellucid](http://www.pellucidanalytics.com) and [Zenexity](http://www.zenexity.com)) trying to enhance [Datomic](http://www.datomic.com) experience for Scala developers.

After evoking [queries compiled by Scala macros in previous article](./2013-02-10-datomisca-query.html), I'm going to describe how _Datomisca_ allows to create Datomic fact operations in a programmatic way and sending them to Datomic transactor using asynchronous/non-blocking API based on _Scala 2.10 Future/ExecutionContext_.


<br/>
# <a name="datomic-facts">Facts about Datomic</a>

First, let's remind a few facts about Datomic:

> Datomic is a [immutable fact-oriented distributed schema-constrained database](http://docs.datomic.com/query.html)
   
It means:

#### Datomic stores very small units of data called _facts_
Yes no tables, documents or even columns in Datomic. Everything stored in it is a very small fact.

<br/>
#### _Fact_ is the atomic unit of data
Facts are represented by the following tuple called **Datom**

{% codeblock lang:clojure %}
datom = [entity attribute value tx]
{% endcodeblock %}

- _entity_ is an ID and several facts can share the same ID making them facts of the same entity. **Here you can see that an entity is very loose concept in Datomic.**
- _attribute_ is just a namespaced keyword : `:person/name` which is generally constrained by a typed schema attribute. **The namespace can be used to logically identify an entity like _"person"_ by regrouping several attributes in the same namespace.**
- _value_ is the value of this attribute for this entity at this instant
- _tx_ uniquely identifies the [transaction](http://docs.datomic.com/glossary.html#sec-35) in which this fact was inserted. Naturally a transaction is associated with a time.

<br/>
#### _Facts_ are immutable & temporal
It means that:

- **You can't change the past**  
Facts are immutable ie you can't mutate a fact as other databases generally do: Datomic always creates a new version of the fact with a new value.
- **Datomic always grows**  
If you add more facts, nothing is deleted so the DB grows. Naturally you can truncate a DB, export it and rebuild a new smaller one.
- **You can foresee a possible future**  
From your present, you can temporarily add facts to Datomic without committing them on central storage thus simulating a possible future.

<br/>
#### Reads/writes are distributed across different components
- **One Storage service** storing physically the data (Dynamo DB/Infinispan/Postgres/Riak/...)
- **Multiple Peers** (generally local to your app instances) behaving like high-speed synchronized cache obfuscating all the local data storage and synchro mechanism and providing the _Datalog_ queries.
- **One (or several) transactor(s)** centralizing the write mechanism allowing ACID transactions and notifying peers about those evolutions. 

For more info about architecture, go to [this page](http://docs.datomic.com/architecture.html)

<br/>
#### Immutability means known DB state is always consistent

You might not be up-to-date with central data storage as Datomic is distributed, you can even lose connection with it but the data you know are always consistent because nothing can be mutated.  
> This immutability concept is one of the most important to understand in Datomic.

<br/>
#### Schema contrains entity attributes

Datomic allows to define that a given attribute must :

- be of **given type** : `String` or `Long` or `Instant` etc…
- have **cardinality** (`one` or `many`)
- be **unique** or not
- be **fullsearchable** or not
- be **documented**
- …

It means that if you try to insert a fact with an attribute and a value of the wrong type, Datomic will refuse it.

Datomic entity can also reference other entities in Datomic providing relations in Datomic (even if Datomic is not RDBMS). One interesting thing to know is that **all relations in Datomic are bidirectional.**

> I hope you immediately see the link between these typed schema attributes and potential Scala type-safe features…

<br/>
> **Author's note : Datomic is more about _evolution_ than mutation**  
> _I'll let you meditate this sentence linked to theory of evolution ;)_

<br/>
# <a name="datomic-ops">Datomic operations</a>

When you want to create a new fact in Datomic, you send a write operation request to the `Transactor`.

## <a name="datomic-ops-basic">Basic operations</a>

There are 2 basic operations:

#### Add a Fact

{% codeblock lang:clojure %}
[:db/add entity-id attribute value]
{% endcodeblock %}

Adding a fact for the same entity will _NOT update_ existing fact but create a new fact with same _entity-id_ and a new _tx_.

#### Retract a Fact

{% codeblock lang:clojure %}
[:db/retract entity-id attribute value]
{% endcodeblock %}

Retracting a fact doesn't erase any fact but just tells: _"for this entity-id, from now, there is no more this attribute"_

You might wonder why providing the value when you want to remove a fact? This is because an attribute can have a _MANY_ cardinality in which case you want to remove just a value from the set of values.

## <a name="datomic-ops-entity">Entity operations</a>

In Datomic, you often manipulate groups of facts identifying an entity. An entity has no physical existence in Datomic but is just a group of facts having the same _entity-id_. Generally, the attributes constituting an entity are logically grouped under the same namespace (`:person/name`, `:person/age`…) but this is not mandatory at all.

Datomic provides 2 operations to manipulate entities directly

#### Add Entity

{% codeblock lang:clojure %}
{:db/id #db/id[:db.part/user -1]
  :person/name "Bob"
  :person/spouse #db/id[:db.part/user -2]}
{% endcodeblock %}

Actually this is equivalent to 2 _Add-Fact_ operations:

{% codeblock lang:clojure %}
(def id #db/id[:db.part/user -1])
[:db/add id :person/name "Bob"]
[:db/add id :person/age 30]
{% endcodeblock %}

#### Retract Entity

{% codeblock lang:clojure %}
[:db.fn/retractEntity entity-id]
{% endcodeblock %}

<br/>
## <a name="datomic-ops-ident">Special case of identified values</a>

In Datomic, there are special entities built using the special attribute `:db/ident` of type `Keyword` which are said to be _identified by the given keyword_.

There are created as following:

{% codeblock lang:clojure %}
[:db/add #db/id[:db.part/user] :db/ident :person.characters/clever]
[:db/add #db/id[:db.part/user] :db/ident :person.characters/dumb]
{% endcodeblock %}

If you use `:person.characters/clever` or `:person.characters/dumb`, it references directly one of those 2 entities without using their ID.

You can see those identified entities as enumerated values also.

Now that you know how it works in Datomic, let's go to _Datomisca_!

<br/>
# <a name="datomisca-ops">Datomisca programmatic operations</a>

Datomisca's preferred way to build Fact/Entity operations is programmatic because it provides more flexibility to Scala developers.
Here are the translation of previous operations in Scala:

{% codeblock lang:scala %}
import datomisca._
import Datomic._

// creates a Namespace
val person = Namespace("person")

// creates a add-fact operation 
// It creates the datom (id keyword value _) from
//   - a temporary id (or a final long ID)
//   - the couple `(keyword, value)`
val addFact = Fact.add(DId(Partition.USER))(person / "name" -> "Bob")

// creates a retract-fact operation
val retractFact = Fact.retract(DId(Partition.USER))(person / "age" -> 123L)

// creates identified values
val violent = AddIdent(person.character / "violent")
val dumb = AddIdent(person.character / "dumb")

// creates a add-entity operation
val addEntity = Entity.add(DId(Partition.USER))(
  person / "name" -> "Bob",
  person / "age" -> 30L,
  person / "characters" -> Set(violent.ref, dumb.ref)
)

// creates a retract-entity operation from real Long ID of the entity
val retractEntity = Entity.retract(3L)

val ops = Seq(addFact, retractFact, addEntity, retractEntity)
{% endcodeblock %}

Note that:

- `person / "name"` creates the keyword `:person/name` from namespace `person`
- `DId(Partition.USER)` generates a temporary Datomic Id in Partition `USER`. _Please note that you can create your own partition too_.
- `violent.ref` is used to access the keyword reference of the _identified entity_.
- `ops = Seq(…)` represents a collection of operations to be sent to _transactor_.

<br/>
# <a name="datomisca-macro-ops">Datomisca Macro operations</a>

Remember the way Datomisca dealt with query by parsing/validating Datalog/Clojure queries at compile-time using Scala macros?

You can do the same in Datomisca with operations:

{% codeblock lang:scala %}
val id = DId(Partition.USER)
val weak = AddIdent( person / "character", "weak"))
val dumb = AddIdent( person / "character", "dumb"))

val ops = Datomic.ops("""[
   [:db/add #db/id[:db.part/user] :db/ident :region/n]
   [:db/add \${DId(Partition.USER)} :db/ident :region/n]
   [:db/retract #db/id[:db.part/user] :db/ident :region/n]
   [:db/retractEntity 1234]
   {
      :db/id \${id}
      :person/name "toto"
      :person/age 30
      :person/characters [ \$weak \$dumb ]
   }
]""")
{% endcodeblock %}

It compiles what's between `"""…"""` at compile-time and tells you if there are errors and then it builds Scala corresponding operations.

Ok it's cool but if you look better, you'll see there is some sugar in this Clojure code:

- `\${DId(Partition.USER)}`
- `\$weak`
- `\$dumb`

**You can use Scala variables and inject them into Clojure operations at compile-time as you do for Scala string interpolation**


>For Datomic queries, the compiled way is really natural but we tend to prefer programmatic way to build operations because it feels to be much more "scala-like" after experiencing both methods.


## <a name="datomisca-parse-ops">Datomisca runtime parsing</a>

There is a last way to create operations by parsing at runtime a String and throwing an exception if the syntax is not valid.

{% codeblock lang:scala %}
val ops = Datomic.parseOps(""" … """)
{% endcodeblock %}

> It's very useful if you have existing Datomic Clojure files (containing schema or bootstrap data) that you want to load into Datomic.

<br/>
# <a name="datomisca-transact">Datomisca reactive transactions</a>

Last but not the least, let's send those operations to Datomic Transactor.

In its Java API, Datomic Connection provides a `transact` asynchronous API based on a `ListenableFuture`. This API can be enhanced in Scala because Scala provides much more evolved asynchronous/non-blocking facilities than Java based on Scala 2.10 `Future`/`ExecutionContext`.

`Future` allows to implement your asynchronous call using continuation style based on Scala classic `map/flatMap` methods.
`ExecutionContext` is a great tool allowing to specify in which pool of threads your asynchronous call will be executed making it non-blocking with respect to your current execution context (or thread). 

> This new feature is really important when you work with reactive API such as Datomisca or Play too so don't hesitate to study it further.

Let's look at code directly to show how it works in _Datomisca_:

{% codeblock lang:scala %}
import datomisca._
import Datomic._

// don't forget to bring an ExecutionContext in your scope… 
// Here is default Scala ExecutionContext which is a simple pool of threads with one thread per core by default
import scala.concurrent.ExecutionContext.Implicits.global

// creates an URI
val uri = "datomic:mem://mydatomicdn"
// creates implicit connection
implicit val conn = Datomic.connect(uri)

// a few operations
val ops = Seq(addFact, retractFact, addEntity, retractEntity)

val res: Future[R] = Datomic.transact(ops) map { tx : TxReport => 
   // do something
   …
   
   // return a value of type R (anything you want)
   val res: R = …
   
   res
}

// Another example by building ops directly in the transact call and using flatMap
Datomic.transact(
  Entity.add(id)(
    person / "name"      -> "toto",
    person / "age"       -> 30L,
    person / "character" -> Set(weak.ref, dumb.ref)
  ),
  Entity.add(DId(Partition.USER))(
    person / "name"      -> "tutu",
    person / "age"       -> 54L,
    person / "character" -> Set(violent.ref, clever.ref)
  ),
  Entity.add(DId(Partition.USER))(
    person / "name"      -> "tata",
    person / "age"       -> 23L,
    person / "character" -> Set(weak.ref, clever.ref)
  )
) flatMap { tx =>
  // do something
  … 
  val res: Future[R] = …
  
  res
}
{% endcodeblock %}
      
> Please note the `tx: TxReport` which is a structure returned by Datomic transactor containing information about last transaction.
      
# <a name="datomisca-resolve">Datomisca resolving Real ID</a>
      
In all samples, we create operations based on temporary ID built by Datomic in a given partition.

{% codeblock lang:scala %}
DId(Partition.USER)
{% endcodeblock %}

But once you have inserted a fact or an entity into Datomic, you need to resolve the real final ID to use it further because the temporary ID is no more meaningful.

The final ID is resolved from the `TxReport` send back by Datomic transactor. This `TxReport` contains a map between temporary ID and final ID. Here is how you can use it in Datomisca:

{% codeblock lang:scala %}
val id1 = DId(Partition.USER)
val id2 = DId(Partition.USER)

Datomic.transact(
  Entity.add(id1)(
    person / "name"      -> "toto",
    person / "age"       -> 30L,
    person / "character" -> Set(weak.ref, dumb.ref)
  ),
  Entity.add(id2)(
    person / "name"      -> "tutu",
    person / "age"       -> 54L,
    person / "character" -> Set(violent.ref, clever.ref)
  )
) map { tx =>
  val finalId1: Long = tx.resolve(id1)
  val finalId2: Long = tx.resolve(id2)
  // or
  val List(finalId1, finalId2) = List(id1, id2) map { tx.resolve(_) }
}

{% endcodeblock %}

<div class="well">That's all for now… Next articles about writing programmatic Datomic schema with Datomisca.</div>


Have Promise[Fun]!