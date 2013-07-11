---
layout: post
title: "Play AutoSource & Datomisca: Proofing the concept on Datomic, a Schema DB"
date: 2013-07-11 18:18
comments: true
external-url:
categories: [datasource,rest,crud,scala,play framework,datomic,datomisca,angularjs,async,]
keywords: datasource,rest,crud,scala,play framework,datomic,datomisca,angularjs,async
---

The code for all autosources & sample apps can be found on Github [here](https://github.com/mandubian/play-autosource/tree/master/datomisca)

<div class="well">
  <h3>Brand New Autosources</h3>
  Play AutoSource now have 2 more implementations : 
  <ul>
    <li><b><a href="http://www.datomic.com">Datomic</a> based on <a href="http://pellucidanalytics.github.io/datomisca/">Datomisca</a></b>, the Scala API I developed with Daniel James (<a href="https://twitter.com/dwhjames">@dwhjames</a>) sponsored by Pellucid Analytics & Zenexity which is presented in this article</li>
    <li><b><a href="http://www.couchbase.com/">CouchBase</a></b> contributed by Mathieu Ancelin <a href="https://twitter.com/TrevorReznik">@TrevorReznik</a></li>
  </ul>
</div>

>One month ago, I've demo'ed the concept of Autosource for Play2/Scala with ReactiveMongo in [this article](http://mandubian.com/2013/06/11/play-autosource/). ReactiveMongo was the perfect target for this idea because it accepts Json structures almost natively for both documents manipulation and queries.

>But how does the concept behave when applied on a DB for which data are constrained by a schema and for which queries aren't Json.

<br/>

## Using Datomisca-Autosource in your Play project

Add following lines to your `project/Build.scala`

{% codeblock lang:scala %}
val mandubianRepo = Seq(
  "Mandubian repository snapshots" at "https://github.com/mandubian/mandubian-mvn/raw/master/snapshots/",
  "Mandubian repository releases" at "https://github.com/mandubian/mandubian-mvn/raw/master/releases/"
)

val appDependencies = Seq()

val main = play.Project(appName, appVersion, appDependencies).settings(
  resolvers ++= mandubianRepo,
  libraryDependencies ++= Seq(
    "play-autosource"   %% "datomisca"       % "0.1-SNAPSHOT",
    ...
  )
)
{% endcodeblock %}

<br/>
## Create your Model + Schema

With ReactiveMongo Autosource, you could create a pure blob Autosource using `JsObject` without any supplementary information. But with Datomic, it's not possible because Datomic forces to use a schema for your data.

We could create a schema and manipulate `JsObject` directly with Datomic and some Json validators. But I'm going to focus on the static models because this is the way people traditionally interact with a Schema-constrained DB.

Let's create our model and schema.

{% codeblock lang:scala %}

// The Model (with characters pointing on Datomic named entities)
case class Person(name: String, age: Long, characters: Set[DRef])

// The Schema written with Datomisca
object Person {
  // Namespaces
  val person = new Namespace("person") {
    val characters = Namespace("person.characters")
  }

  // Attributes
  val name       = Attribute(person / "name",       SchemaType.string, Cardinality.one) .withDoc("Person's name")
  val age        = Attribute(person / "age",        SchemaType.long,   Cardinality.one) .withDoc("Person's age")
  val characters = Attribute(person / "characters", SchemaType.ref,    Cardinality.many).withDoc("Person's characterS")

  // Characters named entities
  val violent = AddIdent(person.characters / "violent")
  val weak    = AddIdent(person.characters / "weak")
  val clever  = AddIdent(person.characters / "clever")
  val dumb    = AddIdent(person.characters / "dumb")
  val stupid  = AddIdent(person.characters / "stupid")

  // Schema
  val schema = Seq(
    name, age, characters,
    violent, weak, clever, dumb, stupid
  )
{% endcodeblock %}

<br/>
## Create Datomisca Autosource

Now that we have our schema, let's write the autosource.

{% codeblock lang:scala %}
import datomisca._
import Datomic._

import play.autosource.datomisca._

import play.modules.datomisca._
import Implicits._

import scala.concurrent.ExecutionContext.Implicits.global
import play.api.Play.current

import models._
import Person._

object Persons extends DatomiscaAutoSourceController[Person] {
  // gets the Datomic URI from application.conf
  val uri = DatomicPlugin.uri("mem")

  // ugly DB initialization ONLY for test purpose
  Datomic.createDatabase(uri)

  // Datomic connection is required
  override implicit val conn = Datomic.connect(uri)
  // Datomic partition in which you store your entities
  override val partition = Partition.USER

  // more than ugly schema provisioning, ONLY for test purpose
  Await.result(
    Datomic.transact(Person.schema),
    Duration("10 seconds")
  )

}
{% endcodeblock %}

## Implementing Json <-> Person <-> Datomic transformers

If you compile previous code, you should have following error:

{% codeblock lang:scala %}
could not find implicit value for parameter datomicReader: datomisca.EntityReader[models.Person]
{% endcodeblock %}

Actually, Datomisca Autosource requires 4 elements to work:

- `Json.Format[Person]` to convert `Person` instances from/to Json (network interface)
- `EntityReader[Person]` to convert `Person` instances from Datomic entities (Datomic interface)
- `PartialAddEntityWriter[Person]` to convert `Person` instances to Datomic entities (Datomic interface)
- `Reads[PartialAddEntity]` to convert Json to `PartialAddEntity` which is actually a simple map of fields/values to partially update an existing entity (one single field for ex).

It might seem more complicated than in ReactiveMongo but there is nothing different. The autosource converts `Person` from/to Json and then converts `Person` from/to Datomic structure ie `PartialAddEntity`. In ReactiveMongo, the only difference is that it understands Json so well that static model becomes unnecessary sometimes ;)...

Let's define those elements in `Person` companion object.

{% codeblock lang:scala %}
object Person {
...
  // Classic Play2 Json Reads/Writes
  implicit val personFormat = Json.format[Person]

  // Partial entity update : Json to PartialAddEntity Reads
  implicit val partialUpdate: Reads[PartialAddEntity] = (
    ((__ \ 'name).read(readAttr[String](Person.name)) orElse Reads.pure(PartialAddEntity(Map.empty))) and
    ((__ \ 'age) .read(readAttr[Long](Person.age)) orElse Reads.pure(PartialAddEntity(Map.empty)))  and
    // need to specify type because a ref/many can be a list of dref or entities so need to tell it explicitly
    (__ \ 'characters).read( readAttr[Set[DRef]](Person.characters) )
    reduce
  )

  // Entity Reads (looks like Json combinators but it's Datomisca combinators)
  implicit val entity2Person: EntityReader[Person] = (
    name      .read[String]   and
    age       .read[Long]     and
    characters.read[Set[DRef]]
  )(Person.apply _)

  // Entity Writes (looks like Json combinators but it's Datomisca combinators)
  implicit val person2Entity: PartialAddEntityWriter[Person] = (
    name      .write[String]   and
    age       .write[Long]     and
    characters.write[Set[DRef]]
  )(DatomicMapping.unlift(Person.unapply))

...
}
{% endcodeblock %}

Now we have everything to work except a few configurations.

### Add AutoSource routes at beginning `conf/routes`

{% codeblock lang:scala %}
->      /person                     controllers.Persons
{% endcodeblock %}

### Create `conf/play.plugins` to initialize Datomisca Plugin

{% codeblock lang:scala %}
400:play.modules.datomisca.DatomicPlugin
{% endcodeblock %}

### Append to `conf/application.conf` to initialize MongoDB connection

{% codeblock lang:scala %}
datomisca.uri.mem="datomic:mem://mem"
{% endcodeblock %}


### Insert your first 2 persons with Curl

{% codeblock lang:scala %}
>curl -X POST -d '{ "name":"bob", "age":25, "characters": ["person.characters/stupid", "person.characters/violent"] }' --header "Content-Type:application/json" http://localhost:9000/persons --include

HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 21

{"id":17592186045423} -> oh a Datomic ID

>curl -X POST -d '{ "name":"john", "age":43, "characters": ["person.characters/clever", "person.characters/weak"] }' --header "Content-Type:application/json" http://localhost:9000/persons --include

HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 21

{"id":17592186045425}
{% endcodeblock %}

### Querying is the biggest difference in Datomic

In Datomic, you can't do a `getAll` without providing a Datomic Query.

But what is a Datomic query? It's inspired by `Datalog` which uses predicates to express the constraints on the searched entities. You can combine predicates together.

With Datomisca Autosource, you can directly send datalog queries in the query parameter `q` for GET or in body for POST with one restriction: your query can't accept input parameters and must return only the entity ID. For ex:

`[ :find ?e :where [ ?e :person/name "john"] ] --> OK`

`[ :find ?e ?name :where [ ?e :person/name ?name] ] --> KO`

Let's use it by finding all persons.

{% codeblock lang:scala %}
>curl -X POST --header "Content-Type:text/plain" -d '[:find ?e :where [?e :person/name]]' 'http://localhost:9000/persons/find' --include

HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 231

[
    {
        "name": "bob",
        "age": 25,
        "characters": [
            ":person.characters/violent",
            ":person.characters/stupid"
        ],
        "id": 17592186045423
    },
    {
        "name": "john",
        "age": 43,
        "characters": [
            ":person.characters/clever",
            ":person.characters/weak"
        ],
        "id": 17592186045425
    }
]
{% endcodeblock %}

> Please note the use of POST here instead of GET because Curl doesn't like `[]` in URL even using `-g` option

Now you can use all other routes provided by Autosource

## Autosource Standard Routes
### Get / Find / Stream

- GET /persons?...    -> Find by query
- GET /persons/ID     -> Find by ID
- GET /persons/stream -> Find by query & stream result by page

### Insert / Batch / Find

- POST /persons + BODY -> Insert
- POST /persons/find + BODY -> find by query (when query is too complex to be in a GET)
- POST /persons/batch  + BODY -> batch insert (multiple)

### Update / batch

- PUT  /persons/ID   + BODY  -> Update by ID
- PUT  /persons/ID/partial   + BODY  -> Update partially by ID
- PUT  /persons/batch   -> batch update (multiple)

### Delete / Batch

- DELETE /persons/ID    -> delete by ID
- DELETE /persons/batch + BODY -> batch delete (multiple)

<br/>
<br/>
## Conclusion

Play-Autosource's ambition was to be DB agnostic (as much as possible) and showing that the concept can be applied to schemaless DB (ReactiveMongo & CouchDB) and schema DB (Datomic) is a good sign it can work. Naturally, there are a few more elements to provide for Datomic than in ReactiveMongo but it's useful anyway.

Thank to [@TrevorReznik](https://twitter.com/TrevorReznik) for his contribution of CouchBase Autosource.

I hope to see soon one for Slick and a few more ;)

Have Autofun!
<br/>
<br/>
<br/>
<br/>