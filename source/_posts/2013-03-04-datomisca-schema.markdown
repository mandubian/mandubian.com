---
layout: post
title: "Datomisca Delicatessen -  Emulsion of Scala type-safety in Datomic Schema"
date: 2013-03-04 00:00
comments: true
external-url: 
categories: [datomic,datomisca,datalog,datom,database,scala,schema]
keywords: datomic,datomisca,datalog,datom,database,scala,schema
---

One more step in our progressive unveiling of [Datomisca](http://pellucidanalytics.github.com/datomisca/index.html), our opensource Scala API (sponsored by [Pellucid](http://www.pellucidanalytics.com) & [Zenexity](http://www.zenexity.com)) trying to enhance [Datomic](http://www.datomic.com) experience for Scala developers...

After evoking [queries compiled by Scala macros in previous article](./2013-02-10-datomisca-query.html) and then [reactive transaction & fact operation API](./2013-02-18-datomisca-fact-operations.html), let's explain **how _Datomisca_ manages Datomic schema attributes**.

<br/>
# Datomic Schema Reminders

As explained in previous articles, Datomic stores lots of atomic facts called `datoms` which are constituted of `entity-id`, `attribute`, `value` and `transaction-id`. 

An attribute is just a namespaced keyword `:<namespace>.<nested-namespace>/<name> such as `:person.address/street`:

- `person.address` is just a hierarchical namespace `person` -> `address`
- `street` is the name of the attribute

It's cool to provision all thoses atomic pieces of information but what if we provision non existing attribute with bad format, type, ...? Is there a way to control the format of data in Datomic?

> In a less strict way than SQL, Datomic provides schema facility allowing to constrain the accepted attributes and their type values.

## Schema attribute definition

Datomic schema just defines the accepted attributes and some constraints on those attributes. Each schema attribute can be defined by following fields:

### value type

- **basic types** : `string`, `long`, `float`, `bigint`, `bigdec`, `boolean`, `instant`, `uuid`, `uri`, `bytes` (yes NO `int`).
- **reference** : in Datomic you can reference other entities (these are lazy relations not as strict as the ones in RDBMS)

### cardinality 

- **one** : one-to-one relation if you want an analogy with RDBMS
- **many** : one-to-many relation

> Please note that in Datomic, all relations are bidirectional even for one-to-many.

### optional constraints:

- unicity
- index creation
- fulltext indexation
- a few more exotic ones that you'll find in [Datomic doc about schema](http://docs.datomic.com/schema.html)

## Schema attributes are entities

The schema validation is applied at fact insertion and allows to prevent from inserting unknown attributes or bad value types. But how are schema attributes defined?

**Actually, schema attributes are themselves entities. **

Remember, in previous article, I had introduced entities as being just loose aggregation of datoms just identified by the same entity ID (the first attribute of a datom). 

So a schema attribute is just an entity stored in a special partition `:db.part/db` and defined by a few specific fields corresponding to the ones in previous paragraph. Here are the fields used to define a Datomic schema attribute technically speaking:

### mandatory fields
    
- `:db/ident` : specifies unique name of the attribute
- `:db/valueType` : specifies one the previous types - _Please note that even those types are not hard-coded in Datomic and in the future, adding new types could be a new feature._
- `:db/cardinality` : specifies the cardinality `one` or `many` of the attribute - a many attribute is just a set of values and type `Set` is important because Datomic only manages sets of unique values as it won't return multiple times the same value when querying.

### optional fields
     
- `:db/unique`
- `:db/doc` (_useful to document your schema_)
- `:db/index`
- `:db/fulltext`
- `:db/isComponent`
- `:db/noHistory`

Here is an example of schema attribute declaration written in Clojure:

{% codeblock lang:clojure %}
{:db/id #db/id[:db.part/db]
 :db/ident :person/name
 :db/valueType :db.type/string
 :db/cardinality :db.cardinality/one
 :db/doc "A person's name"}
{% endcodeblock %}

> As you can see, creating schema attributes just means creating new entities in the right partition. So, to add new attributes to Datomic, you just have to add new facts.

<br/>
# Schema sample

Let's create a schema defining a Koala living in an eucalyptus.

<div class="well">
  <p>Yes I'm a super-Koala fan! Don't ask me why, this is a long story not linked at all to Australia :D... But saving Koalas is important to me so I put this little banner for them...<span style="float: right"><a href="http://www.savethekoala.com"><img src="/images/mandubian/buttondonate.gif" /></a></span>
  </p>
</div>

Let's define a koala by following attributes:

- a name `String`
- an age `Long`
- a sex which can be `male` or `female
- a few eucalyptus trees in which to feed defined by:

    - a species being a `reference` to one of the possible species of eucalyptus trees
    - a row `Long` (_let's imagine those trees are planted in rows/columns_)
    - a column `Long`

Here is the Datomic schema for this:

{% codeblock lang:clojure %}
[
{:db/id #db/id[:db.part/db]
 :db/ident :koala/name
 :db/valueType :db.type/string
 :db/unique :db.unique/value
 :db/cardinality :db.cardinality/one
 :db/doc "A koala's name"}

{:db/id #db/id[:db.part/db]
 :db/ident :koala/age
 :db/valueType :db.type/long
 :db/cardinality :db.cardinality/one
 :db/doc "A koala's age"}

{:db/id #db/id[:db.part/db]
 :db/ident :koala/sex
 :db/valueType :db.type/ref
 :db/cardinality :db.cardinality/one
 :db/doc "A koala's sex"}

{:db/id #db/id[:db.part/db]
 :db/ident :koala/eucalyptus
 :db/valueType :db.type/ref
 :db/cardinality :db.cardinality/many
 :db/doc "A koala's eucalyptus trees"}

{:db/id #db/id[:db.part/db]
 :db/ident :eucalyptus/species
 :db/valueType :db.type/ref
 :db/cardinality :db.cardinality/one
 :db/doc "A eucalyptus specie"}

{:db/id #db/id[:db.part/db]
 :db/ident :eucalyptus/row
 :db/valueType :db.type/long
 :db/cardinality :db.cardinality/one
 :db/doc "A eucalyptus row"}

{:db/id #db/id[:db.part/db]
 :db/ident :eucalyptus/col
 :db/valueType :db.type/long
 :db/cardinality :db.cardinality/one
 :db/doc "A eucalyptus column"}

;; koala sexes as keywords
[:db/add #db/id[:db.part/user] :db/ident :sex/male]
[:db/add #db/id[:db.part/user] :db/ident :sex/female]

;; eucalyptus species
[:db/add #db/id[:db.part/user] :db/ident :eucalyptus.species/manna_gum]
[:db/add #db/id[:db.part/user] :db/ident :eucalyptus.species/tasmanian_blue_gum]
[:db/add #db/id[:db.part/user] :db/ident :eucalyptus.species/swamp_gum]
[:db/add #db/id[:db.part/user] :db/ident :eucalyptus.species/grey_gum]
[:db/add #db/id[:db.part/user] :db/ident :eucalyptus.species/river_red_gum]
[:db/add #db/id[:db.part/user] :db/ident :eucalyptus.species/tallowwood]

]
{% endcodeblock %}

In this sample, you can see that we have defined 4 namespaces:

- `koala` used to logically regroup koala entity fields
- `eucalyptus` used to logically regroup eucalyptus entity fields
- `sex` used to identify koala sex male or female as unique keywords
- `eucalyptus.species` to identify eucalyptus species as unique keywords

Remark also:

- `:koala/name` field is uniquely valued meaning no koala can have the same name
- `:koala/eucalyptus` field is a _one-to-many_ reference to eucalyptus entities

<br/>
# Datomisca way of declaring schema

## First of all, initialize your Datomic DB

{% codeblock lang:scala %}
import scala.concurrent.ExecutionContext.Implicits.global

import datomisca._
import Datomic._

val uri = "datomic:mem://koala-db"

Datomic.createDatabase(uri)
implicit val conn = Datomic.connect(uri)
{% endcodeblock %}


## The NOT-preferred way

Now, you must know it but Datomisca intensively uses Scala 2.10 macros to provide compile-time parsing and validation of Datomic queries or operations written in Clojure.

Previous Schema attributes definition is just a set of classic operations so you can ask Datomisca to parse them at compile-time as following:

{% codeblock lang:scala %}

val ops = Datomic.ops("""[
{:db/id #db/id[:db.part/db]
 :db/ident :koala/name
 :db/valueType :db.type/string
 :db/unique :db.unique/value
 :db/cardinality :db.cardinality/one
 :db/doc "A koala's name"}
...
]""")
{% endcodeblock %}

Then you can provision the schema into Datomic using:

{% codeblock lang:scala %}
Datomic.transact(ops) map { tx =>
  ...
  // do something
  //
}
{% endcodeblock %}

## The preferred way

Ok the previous is cool as you can validate and provision a clojure schema using Datomisca.
But Datomisca provides a programmatic way of writing schema in Scala. This brings :

- **scala idiomatic** way of manipulating schema 
- **Type-safety** to Datomic schema attributes.

Let's see the code directly:

{% codeblock lang:scala %}
// Sex Schema
object SexSchema {
  // First create your namespace
  object ns {
    val sex = Namespace("sex")
  }

  // enumerated values
  val FEMALE  = AddIdent(ns.sex / "female") // :sex/female
  val MALE    = AddIdent(ns.sex / "male")   // :sex/male

  // facts representing the schema to be provisioned
  val txData = Seq(FEMALE, MALE)
}

// Eucalyptus Schema
object EucalyptusSchema {
  object ns {
    val eucalyptus  = new Namespace("eucalyptus") { // new is just here to allow structural construction
      val species   = Namespace("species")
    }
  }

  // different species
  val MANNA_GUM           = AddIdent(ns.eucalyptus.species / "manna_gum")
  val TASMANIAN_BLUE_GUM  = AddIdent(ns.eucalyptus.species / "tasmanian_blue_gum")
  val SWAMP_GUM           = AddIdent(ns.eucalyptus.species / "swamp_gum")
  val GRY_GUM             = AddIdent(ns.eucalyptus.species / "grey_gum")
  val RIVER_RED_GUM       = AddIdent(ns.eucalyptus.species / "river_red_gum")
  val TALLOWWOOD          = AddIdent(ns.eucalyptus.species / "tallowwood")

  // schema attributes
  val species  = Attribute(ns.eucalyptus / "species", SchemaType.ref, Cardinality.one).withDoc("Eucalyptus's species")
  val row      = Attribute(ns.eucalyptus / "row", SchemaType.long, Cardinality.one).withDoc("Eucalyptus's row")
  val col      = Attribute(ns.eucalyptus / "col", SchemaType.long, Cardinality.one).withDoc("Eucalyptus's column")
  
  // facts representing the schema to be provisioned
  val txData = Seq(
    species, row, col,
    MANNA_GUM, TASMANIAN_BLUE_GUM, SWAMP_GUM,
    GRY_GUM, RIVER_RED_GUM, TALLOWWOOD
  )
}

// Koala Schema
object KoalaSchema {
  object ns {
    val koala = Namespace("koala")
  }

  // schema attributes
  val name         = Attribute(ns.koala / "name", SchemaType.string, Cardinality.one).withDoc("Koala's name").withUnique(Unique.value)
  val age          = Attribute(ns.koala / "age", SchemaType.long, Cardinality.one).withDoc("Koala's age")
  val sex          = Attribute(ns.koala / "sex", SchemaType.ref, Cardinality.one).withDoc("Koala's sex")
  val eucalyptus   = Attribute(ns.koala / "eucalyptus", SchemaType.ref, Cardinality.many).withDoc("Koala's trees")

  // facts representing the schema to be provisioned
  val txData = Seq(name, age, sex, eucalyptus)
}


// Provision Schema by just accumulating all txData
Datomic.transact(
  SexSchema.txData ++
  EucalyptusSchema.txData ++
  KoalaSchema.txData
) map { tx =>
  ...  
}

{% endcodeblock %}

Nothing complicated, isn't it?

Exactly the same as writing Clojure schema but in Scala...

<br/>
# Datomisca type-safe schema

Datomisca takes advantage of Scala type-safety to enhance Datomic schema attribute and make them static-typed. Have a look at Datomisca `Attribute` definition:

{% codeblock lang:scala %}
sealed trait Attribute[DD <: DatomicData, Card <: Cardinality]
{% endcodeblock %}

So an `Attribute` is typed by 2 parameters:

- a `DatomicData` type
- a `Cardinality` type

So when you define a schema attribute using _Datomisca_ API, the compiler also infers those types.

Take this example:

{% codeblock lang:scala %}
  val name  = Attribute(ns / "name", SchemaType.string, Cardinality.one).withDoc("Koala's name").withUnique(Unique.value)
{% endcodeblock %}

- `SchemaType.string` implies this is a `Attribute[DString, _]`
- `Cardinality.one` implies this is a `Attribute[_, Cardinality.one]

So `name` is a `Attribute[DString, Cardinality.one]`

In the same way:

- `age` is `Attribute[DLong, Cardinality.one]`
- `sex` is `Attribute[DRef, Cardinality.one]`
- `eucalyptus` is `Attribute[DRef, Cardinality.many]`

> As you can imagine, using this type-safe schema attributes, Datomisca can ensure consistency between the Datomic schema and the types manipulated in Scala.

<br/>
## Taking advantage of type-safe schema

### Checking types when creating facts

> Based on the typed attribute, the compiler can help us a lot to validate that we give the right type for the right attribute.

Schema facilities are extensions of basic Datomisca so you must import following to use them:

{% codeblock lang:scala %}
import DatomicMapping._
{% endcodeblock %}

Here is a code sample:

{% codeblock lang:scala %}
//////////////////////////////////////////////////////////////////////
// correct tree with right types
scala> val tree58 = SchemaEntity.add(DId(Partition.USER))(Props() +
  (EucalyptusSchema.species -> EucalyptusSchema.SWAMP_GUM.ref) +
  (EucalyptusSchema.row     -> 5L) +
  (EucalyptusSchema.col     -> 8L)
)
tree58: datomisca.AddEntity =
{
  :eucalyptus/species :species/swamp_gum
  :eucalyptus/row 5
  :eucalyptus/col 8
  :db/id #db/id[:db.part/user -1000000]
}

//////////////////////////////////////////////////////////////////////
// incorrect tree with a string instead of a long for row
scala> val tree58 = SchemaEntity.add(DId(Partition.USER))(Props() +
  (EucalyptusSchema.species -> EucalyptusSchema.SWAMP_GUM.ref) +
  (EucalyptusSchema.row     -> "toto") +
  (EucalyptusSchema.col     -> 8L)
)
<console>:18: error: could not find implicit value for parameter attrC: 
  datomisca.Attribute2PartialAddEntityWriter[datomisca.DLong,datomisca.CardinalityOne.type,String]
         (EucalyptusSchema.species -> EucalyptusSchema.SWAMP_GUM.ref) +         
{% endcodeblock %}

In second case, compiling fails because `DLong => String` doesn't exist.

In first case, it works because `DLong => Long` is valid.

<br/>
### Checking types when getting fields from Datomic entities

First of all, let's create our first little Koala named _Rose_ which loves feeding from 2 eucalyptus trees.

{% codeblock lang:scala %}
scala> val tree58 = SchemaEntity.add(DId(Partition.USER))(Props() +
  (EucalyptusSchema.species -> EucalyptusSchema.SWAMP_GUM.ref) +
  (EucalyptusSchema.row     -> 5L) +
  (EucalyptusSchema.col     -> 8L)
)
tree74: datomisca.AddEntity =
{
  :eucalyptus/species :species/swamp_gum
  :eucalyptus/row 5
  :eucalyptus/col 8
  :db/id #db/id[:db.part/user -1000002]
}

scala> val tree74 = SchemaEntity.add(DId(Partition.USER))(Props() +
  (EucalyptusSchema.species -> EucalyptusSchema.RIVER_RED_GUM.ref) +
  (EucalyptusSchema.row     -> 7L) +
  (EucalyptusSchema.col     -> 4L)
)
tree74: datomisca.AddEntity =
{
  :eucalyptus/species :species/river_red_gum
  :eucalyptus/row 7
  :eucalyptus/col 4
  :db/id #db/id[:db.part/user -1000004]
}

scala> val rose = SchemaEntity.add(DId(Partition.USER))(Props() +
  (KoalaSchema.name        -> "rose" ) +
  (KoalaSchema.age         -> 3L ) +
  (KoalaSchema.sex         -> SexSchema.FEMALE.ref ) +
  (KoalaSchema.eucalyptus  -> Set(DRef(tree58.id), DRef(tree74.id)) )
)
rose: datomisca.AddEntity =
{
  :koala/eucalyptus [#db/id[:db.part/user -1000001], #db/id[:db.part/user -1000002]]
  :koala/name "rose"
  :db/id #db/id[:db.part/user -1000003]
  :koala/sex :sex/female
  :koala/age 3
}
{% endcodeblock %}

Now let's provision those koala & trees into Datomic and retrieve real entity corresponding to our little Rose kitty.

{% codeblock lang:scala %}
Datomic.transact(tree58, tree74, rose) map { tx =>
  val realRose = Datomic.resolveEntity(tx, rose.id)
  ...  
}
{% endcodeblock %}

Finally let's take advantage of typed schema attribute to access safely to fiels of the entity:

{% codeblock lang:scala %}
scala> val maybeRose = Datomic.transact(tree58, tree74, rose) map { tx =>
  val realRose = Datomic.resolveEntity(tx, rose.id)

  val name = realRose(KoalaSchema.name)
  val age = realRose(KoalaSchema.age)
  val sex = realRose(KoalaSchema.sex)
  val eucalyptus = realRose(KoalaSchema.eucalyptus)

  (name, age, sex, eucalyptus)
}
maybeRose: scala.concurrent.Future[(String, Long, Long, Set[Long])] = scala.concurrent.impl.Promise$DefaultPromise@49f454d6
{% endcodeblock %}

What's important here is that you get a `(String, Long, Long, Set[Long])` which means the compiler was able to infer the right types from the Schema Attribute...

Greattt!!!

Ok that's all for today!

Next article about an extension Datomisca provides for convenience : mapping Datomic entities to Scala structures such as case-classes or tuples. We don't believe this is really the philosophy of Datomic in which atomic operations are much more interesting. But sometimes it's convenient when you want to have data abstraction layer...

Have KoalaFun!
