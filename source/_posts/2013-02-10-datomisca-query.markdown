---
layout: post
title: "Datomisca Delicatessen - Datomic queries coated in a thin layer of Scala"
date: 2013-02-10 13:13
comments: true
external-url: 
categories: [datomic,datomisca,datalog,scala,scala-macro]
keywords: datomic,datomisca,datalog,scala,scala-macro
---


Last week, we have launched [Datomisca](http://pellucidanalytics.github.com/datomisca/index.html), our opensource Scala API trying to enhance [Datomic](http://www.datomic.com) experience for Scala developers. 

Datomic is great in Clojure because it is was made for it. Yet, we believe Scala can provide a very good platform for Datomic too because the functional concepts found in Clojure are also in Scala except that Scala is a compiled and statically typed language whereas Clojure is dynamic. Scala could also bring a few features on top of Clojure based on its features such as static typing, typeclasses, macros…

This article is the first of a serie of articles aiming at describing as shortly as possible specific features provided by Datomisca.
Today, let's present how Datomisca enhances Datomic queries.

<br/>
# <a name="datomic-query">Query in Datomic?</a>

Let's take the same old example of a Person having :

- a name `String`
- a age `Long`
- a birth `Date`

So, how do you write a query in Datomic searching a person by its name?
Like that…

{% codeblock lang:scala %}
(def q [ :find ?e
  :in $ ?name
  :where [ ?e :person/name ?name ] 
])
{% endcodeblock %}

As you can see, this is Clojure using Datalog rules.

In a summary, this query:

- accepts 2 inputs parameters:
    - a datasource `$`
    - a name parameter `?name`
- searches facts respecting datalog rule `[ ?e :person/name ?name ]`: a fact having attribute `:person/name` with value `?name`
- returns the ID of the found facts `?e`

<br/>
## <a name="datomic-query-reminders">Reminders about Datomic queries</a>

### Query is a static data structure

An important aspect of queries to understand in Datomic is that a query is purely a static data structure and not something functional. We could compare it to a prepared statement in SQL: build it once and reuse it as much as you need. 


### Query has input/ouput parameters

In previous example:

- `:in` enumerates input parameters
- `:find` enumerates output parameters

When executing this query, you must provide the right number of input parameters and you will retrieve the given number of output parameters.

<br/>
# <a name="datomisca-query">Query in Datomisca?</a>

So now, how do you write the same query in Datomisca?

{% codeblock lang:scala %}
val q  = Query("""
[ :find ?e
  :in $ ?name
  :where [ ?e :person/name ?name ] 
]""")
{% endcodeblock %}

I see you're a bit disappointed: a query as a string whereas in Clojure, it's a real data structure…

This is actually the way the Java API sends query for now. Moreover, using strings like implies potential bad practices such as building queries by concatenating strings which are often the origin of risks of code injection in SQL for example…

But in Scala we can do a bit better using new Scala 2.10 features : Scala macros.

**So, using Datomisca, when you write this code, in fact, the query string is parsed by a Scala macro:**

- **If there are any error, the compilation breaks showing where the error was detected.**
- **If the query seems valid (with respect to our parser), the String is actually replaced by a AST representing this query as a data structure.**
- **The input/output parameters are infered to determine their numbers.**

> Please note that the compiled query is a simple immutable AST which could be manipulated as a Clojure query and re-used as many times as you want.

### Example OK with single output

{% codeblock lang:scala %}
scala> import datomisca._
import datomisca._

scala> val q  = Query("""
     [ :find ?e
       :in $ ?name
       :where [ ?e :person/name ?name ] 
     ]""")
q: datomisca.TypedQueryAuto2[datomisca.DatomicData,datomisca.DatomicData,datomisca.DatomicData] = [ :find ?e :in $ ?name :where [?e :person/name ?name] ]
{% endcodeblock %}

Without going in deep details, here you can see that the compiled version of `q` isn't a `Query[String]` but a `TypedQueryAuto2[DatomicData, DatomicData, DatomicData]` being an AST representing the query.

`TypedQueryAuto2[DatomicData, DatomicData, DatomicData]` means you have: 

- 2 input parameters `$ ?name` of type `DatomicData` and `DatomicData`
- Last type parameter represents output parameter `?e` of type `DatomicData`
    
*Note : `DatomicData` is explained in next paragraph.*
    
<br/>
### Example OK with several outputs
    
{% codeblock lang:scala %}
scala> import datomisca._
import datomisca._

scala> val q  = Query("""
     [ :find ?e ?age
       :in $ ?name
       :where [ ?e :person/name ?name ] 
              [ ?e :person/age ?age ]  
     ]""")
q: datomisca.TypedQueryAuto2[datomisca.DatomicData,datomisca.DatomicData,(datomisca.DatomicData, datomisca.DatomicData)] = [ :find ?e ?age :in $ ?name :where [?e :person/name ?name] [?e :person/age ?age] ]
{% endcodeblock %}

`TypedQueryAuto2[DatomicData,DatomicData,(DatomicData, DatomicData)]` means you have: 

- 2 input parameters `$ ?name` of type `DatomicData`
- last tupled type parameter represents the 2 output parameters `?e ?age` of type `DatomicData`
    
<br/>
### Examples with syntax-error

{% codeblock lang:scala %}
scala> import datomisca._
import datomisca._

scala> val q  = Query("""
     [ :find ?e
       :in $ ?name
       :where [ ?e :person/name ?name 
     ]""")
<console>:14: error: `]' expected but end of source found
     ]""")
      ^

scala> val q  = Query("""
     [ :find ?e
       :in $ ?name
       :where [ ?e person/name ?name ]
     ]""")
<console>:13: error: `]' expected but `p' found
       :where [ ?e person/name ?name ]
                   ^
{% endcodeblock %}

Here is see that the compiler will tell you where it detects syntax errors.  

*The query compiler is not yet complete so don't hesitate to report us when you discover issues.*


<br/>
## <a name="datomicdata">What's `DatomicData` ?</a>

Datomisca wraps completely Datomic API and types. So Datomisca doesn't let any Datomic/Clojure types perspirating into its domain and wraps them all in the so-called `DatomicData` which is the abstract parent trait of all Datomic types seen from Datomisca. For each Datomic type, you have the corresponding specific `DatomicData`:

- DString for String
- DLong for Long
- DatomicFloat for Float
- DSet for Set
- DInstant for Instant
- ...

###Why not using Pure Scala types directly?
Firstly, because type correspondence is not exact between Datomic types and Scala. The best sample is `Instant`: is it a `java.util.Date` or a `jodatime.DateTime`?  

Secondly, we wanted to keep the possibility of converting Datomic types into different Scala types depending on our needs so we have abstracted those types.  

This abstraction also isolates us and we can decide exactly how we want to map Datomic types to Scala. The trade-off is naturally that, if new types appear in Datomic, we must wrap them.

<br/>
###Keep in mind that Datomisca queries accept and return `DatomicData`

All query data used as input and output paremeters shall be `DatomicData`. When getting results, you can convert those generic `DatomicData` into one of the previous specific types (`DString`, `DLong`, … ). 

From `DatomicData`, you can also convert to Scala pure types based on implicit typeclasses:

{% codeblock lang:scala %}
DatomicData.as[T](implicit rd: DDReader[DatomicData, T])

scala> DString("toto").as[String]
res0: String = toto

scala> DString("toto").as[Long]
java.lang.ClassCastException: datomisca.DString cannot be cast to datomisca.DLong
...
{% endcodeblock %}

*Note 1 : that current Scala query compiler is a bit restricted to the specific domain of Datomic queries and doesn't support all Clojure syntax which might create a few limitations when calling Clojure functions in queries. Anyway, a full Clojure syntax Scala compiler is in the TODO list so these limitations will disappear once it's implemented…*
<br/>

*Note 2 : Current macro just infers the number of input/output parameters but, using Schema typed attributes that we will present in a future article, we will provide some deeper features such as parameter type inference.*

<br/>
# <a name="execute">Execute the query</a>

You can create queries independently of any connection to Datomic.
But you need an implicit `DatomicConnection` in your scope to execute it.

{% codeblock lang:scala %}
import datomisca._
import Datomic._

// Creates an implicit connection
val uri = "…"
implicit lazy val conn = Datomic.connect(uri)

// Creates the query
val queryFindByName = Query("""
[ :find ?e ?birth
  :in $ ?name
  :where [ ?e :person/name ?name ]
         [ ?e :person/birth ?birth ]        
]""")
     
// Executes the query     
val results: List[(DatomicData, DatomicData] = Datomic.q(queryFindByName, database, DString("John"))
// Results type is precised for the example but not required
{% endcodeblock %}

*Please note we made the `database` input parameter mandatory even if it's implicit in when importing `Datomic._` because in Clojure, it's also required and we wanted to stick to it.*

### Compile-error if wrong number of inputs

If you don't provide 2 input parameters, you will get a compile error because the query expects 2 input parameters.

{% codeblock lang:scala %}
// Following would not compile because query expects 2 input parameters
val results: List[(DatomicData, DatomicData] = Datomic.q(queryFindByName, DString("John"))

[info] Compiling 1 Scala source to /Users/pvo/zenexity/workspaces/workspace_pellucid/datomisca/samples/getting-started/target/scala-2.10/classes...
[error] /Users/pvo/zenexity/workspaces/workspace_pellucid/datomisca/samples/getting-started/src/main/scala/GettingStarted.scala:87: overloaded method value q with alternatives:
[error]   [A, R(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)](query: datomisca.TypedQueryAuto1[A,R(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)], a: A)(implicit db: datomisca.DDatabase, implicit ddwa: datomisca.DD2Writer[A], implicit outConv: datomisca.DatomicDataToArgs[R(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)])List[R(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)] <and>
[error]   [R(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)](query: datomisca.TypedQueryAuto0[R(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)], db: datomisca.DDatabase)(implicit outConv: datomisca.DatomicDataToArgs[R(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)])List[R(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)(in method q)] <and>
[error]   [OutArgs <: datomisca.Args, T](q: datomisca.TypedQueryInOut[datomisca.Args1,OutArgs], d1: datomisca.DatomicData)(implicit db: datomisca.DDatabase, implicit outConv: datomisca.DatomicDataToArgs[OutArgs], implicit ott: datomisca.ArgsToTuple[OutArgs,T])List[T] <and>
[error]   [InArgs <: datomisca.Args](query: datomisca.PureQuery, in: InArgs)(implicit db: datomisca.DDatabase)List[List[datomisca.DatomicData]]
[error]  cannot be applied to (datomisca.TypedQueryAuto2[datomisca.DatomicData,datomisca.DatomicData,(datomisca.DatomicData, datomisca.DatomicData)], datomisca.DString)
[error]         val results = Datomic.q(queryFindByName, DString("John"))
[error]                               ^
[error] one error found
[error] (compile:compile) Compilation failed
{% endcodeblock %}

The compile error seems a bit long as the compiler tries a few different version of `Datomic.q` but just remind that when you see `cannot be applied to (datomisca.TypedQueryAuto2[…`, it means you provided the wrong number of input parameters.

<br/>
### Use query results

Query results are `List[DatomicData…]` depending on the output parameters inferred by the Scala macros.

In our case, we have 2 output parameters so we expect a `List[(DatomicData, DatomicData)]`.
Using `List.map` (or `headOption` to get the first one only), you can then use pattern matching to specialize your `(DatomicData, DatomicData)` to `(DLong, DInstant)` as you expect. 

{% codeblock lang:scala %}
results map {
  case (e: DLong, birth: DInstant) => 
    // converts into Scala types
    val eAsLong = e.as[Long]
    val birthAsDate = birth.as[java.util.Date]
}
{% endcodeblock %}

*Note 1: that when you want to convert your `DatomicData`, you can use our converters based on implicit typeclasses as following*

*Note 2: The Scala macro has not way just based on query to infer the real types of output parameters but ther is a TODO in the roadmap: using typed schema attributes presented in a future article, we will be able to do better certainly… Be patient ;)*

<br/>
# <a name="complex">More complex queries</a>

As Datomisca parses the queries, you may wonder what is the level of completeness of the query parser for now?

Here are a few examples showing what can be executed already:

{% codeblock lang:scala %}

////////////////////////////////////////////////////
// using variable number of inputs
val q = Query("""[
 :find ?e
 :in $ [?names ...] 
 :where [?e :person/name ?names]
]""")

Datomic.q(q, database, DSet(DString("toto"), DString("tata")))

////////////////////////////////////////////////////
// using tuple inputs
val q = Query("""[
  :find ?e ?name ?age
  :in $ [[?name ?age]]
  :where [?e :person/name ?name]
         [?e :person/age ?age]
]""")

Datomic.q(q, 
  database, 
  DSet(
    DSet(DString("toto"), DLong(30L)),
    DSet(DString("tutu"), DLong(54L))
  )
)

////////////////////////////////////////////////////
// using function such as fulltext search
val q = Query("""[
  :find ?e ?n
  :where [(fulltext $ :person/name "toto") [[ ?e ?n ]]]
]""")

////////////////////////////////////////////////////
// using rules
val totoRule = Query.rules("""
[ [ [toto ?e]
    [?e :person/name "toto"]
] ]
""")

val q = Query("""[
 :find ?e ?age
 :in $ %
 :where [?e :person/age ?age]
        (toto ?e)
]
""")

////////////////////////////////////////////////////
// using query specifying just the field in fact to be searched
val q = Query("""[:find ?e :where [?e :person/name]]""")
      
{% endcodeblock %}

*Note that currently Datomisca reserializes queries to string when executing because Java API requires it but once Datomic Java API accepts that we pass List[List[Object]] instead of strings for query, the interaction will be more direct…*

<div class="well">Next articles about Datomic operations to insert/retract facts or entities in Datomic using Datomisca.</div>


Have datomiscafun!