---
layout: post
title: "Unveiling Play 2.1 Json API - Part 1 : JsPath & Reads combinators"
date: 2012-09-08 11:10
comments: true
external-url: 
categories: [playframework,play2.1,json,serialization,validation, combinators, applicative]
keywords: play 2.1, json, json serialization, json parsing, json validation, json cumulative errors, play framework, combinators, applicative
---
> **Addendum: recent API refactoring (modified in the articled)** 
> 
> - `Reads[A] provided Reads[B]` has been renamed to `Reads[A] keepAnd Reads[B]` 
> - `Reads[A] andThen Reads[B]` has been renamed to `Reads[A] andKeep Reads[B]`

In incoming `Play2.1` version, a huge **re-thinking** has been done about **`JSON API` provided by `Play2.0.x`** which provides some great features but is clearly just the tip of the iceberg…  
Here is a first presentation of those evolutions aimed at **unleashing your JSON usage in Play2** and **revealing new forms of manipulation of web dataflows from/to external data systems**.   
A usecase of this is manipulating DB structures directly using Json without any class models for document oriented structures such as [MongoDB](http://www.mongodb.org) 

> BTW Don't forget the recent release of new MongoDB async/non-blocking driver [ReactiveMongo](http://reactivemongo.org) ;-)

<div class="well">
<h3><a name="summary">Summary of main new features added in <code>Play2.1</code></a></h3>
<ul>
	<li><b>Simplified Json Objects/Arrays syntax</b> {% codeblock lang:scala %}
Json.obj(
  "key" -> "value", 
  "key2" -> 123, 
  "key3" -> Json.arr("alpha", 143.55, false)
){% endcodeblock %}</li>
	<li><b><code>JsPath</code> for manipulating JSON in <code>XmlPath</code>-style</b> {% codeblock lang:scala %}
(JsPath \ "key" \ "key2" \\ "key3")(3){% endcodeblock %}</li>
	<li><b><code>Reads[T]</code> / <code>Writes[T]</code> / <code>Format[T]</code> combinators</b> based on JsPath and simple logic operators</b>
{% codeblock lang:scala %}	
val customReads: Reads[(String, Float, List[String])] = 
	(JsPath \ "key1").read[String](email keepAnd minLength(5)) and 
	(JsPath \ "key2").read[Float](min(45)) and
	(JsPath \ "key3").read[List[String]] 
	tupled
{% endcodeblock %}
	</li>
	<li><b><code>Reads[T]</code> API now validates JSON</b> by returning a monadic <code>JsResult[T]</code> being a <code>JsSuccess[T]</code> or a <code>JsError</code> aggregating all validation errors {% codeblock lang:scala %}
val js = Json.obj(
  "key1" -> "alpha", 
  "key2" -> 123.345F, 
  "key3" -> Json.arr("alpha", "beta")
)

scala> customReads.reads(js) 
res5: JsSuccess(("alpha", 123.345F, List("alpha", "beta")))

customReads.reads(js).fold(
  valid = { res => 
    val (s, f, l): (String, Float, List[String]) = res
    ...
  },
  invalid = { errors => ... }
)
		
{% endcodeblock %}	
	</li>
</ul>
</div>

**Now let's go in the details ;)**
<br/>
<br/>
# <a name="quick-json-syntax">Very Quick Json syntax</a>

Concerning the new Json syntax, I won't spend time on this, it's quite explicit and you can try it very easily by yourself.

{% codeblock lang:scala %}
val js = Json.obj( 
  "key1" -> "value1", 
  "key2" -> 234, 
  "key3" -> Json.obj( 
    "key31" -> true, 
    "key32" -> Json.arr("alpha", "beta", 234.13),
    "key33" -> Json.obj("key1" -> "value2", "key34" -> "value34")
  )
)
{% endcodeblock %}	

<br/>
<br/>
# <a name="quick-jspath">Quick JsPath</a>

You certainly know `XMLPath` in which you can access a node of an XML AST using a simple syntax based on path.  
JSON is already an AST and we can apply the same kind of syntax to it and logically we called it `JsPath`.  

All following examples use JSON defined in previous paragraph.

### Building JsPath

{% codeblock lang:scala %}
// Simple path
JsPath \ "key1"

// 2-levels path
JsPath \ "key3" \ "key33"
 
// indexed path
(JsPath \ "key3" \ "key32")(2) // 2nd element in a JsArray

// multiple paths
JsPath \\ "key1"
{% endcodeblock %}	

### Alternative syntax

`JsPath` is quite cool but we found this syntax could be made even clearer to highlight `Reads[T]` combinators in the code.  
That's why we provide an alias for `JsPath`: `__` (2 underscores).  
_You can use it or not. This is just a visual facility because with it, you immediately find your JsPath in the code…_

You can write:

{% codeblock lang:scala %}
// Simple path
__ \ "key1"

// 2-levels path
__ \ "key3" \ "key33"
 
// indexed path
(__ \ "key3" \ "key32")(2) // 2nd element in a JsArray

// multiple paths
__ \\ "key1"

// a sample on Reads[T] combinators to show the difference with both syntax
// DON'T TRY TO UNDERSTAND THIS CODE RIGHT NOW… It's explained in next paragraphs
val customReads = 
  (JsPath \ "key1").read(
    (JsPath \ "key11").read[String] and
    (JsPath \ "key11").read[String] and
    (JsPath \ "key11").read[String]
    tupled
  ) and 
  (JsPath \ "key2").read[Float](min(45)) and
  (JsPath \ "key3").read(
    (JsPath \ "key31").read[String] and
    (JsPath \ "key32").read[String] and
    (JsPath \ "key33").read[String]
    tupled
  ) 
  tupled
	
// with __
val customReads = 
  (__ \ "key1").read(
    (__ \ "key11").read[String] and
    (__ \ "key11").read[String] and
    (__ \ "key11").read[String]  
    tupled
  ) and 
  (__ \ "key2").read[Float](min(45)) and
  (__ \ "key3").read[List[String]] (
    (__ \ "key31").read[String] and
    (__ \ "key32").read[String] and
    (__ \ "key33").read[String]
    tupled  
  )
  tupled
	
// You can immediately see the structure of the JSON tree
{% endcodeblock %}	


### Accessing value of JsPath

The important function to retrieve the value at a given JsPath in a JsValue is the following:

{% codeblock lang:scala %}
sealed trait PathNode {
  def apply(json: JsValue): List[JsValue]
  …
}
{% endcodeblock %}	

As you can see, this function retrieves a `List[JsValue]`

You can simply use it like:
{% codeblock lang:scala %}
// build a JsPath
scala> (__ \ "key1")(js) 
res12: List[play.api.libs.json.JsValue] = List("value1")  // actually this is JsString("value1")

// 2-levels path
scala> (__ \ "key3" \ "key33")(js)
res13: List[play.api.libs.json.JsValue] = List({"key":"value2","key34":"value34"})
 
// indexed path
scala> (__ \ "key3" \ "key32")(2)(js)
res14: List[play.api.libs.json.JsValue] = List(234.13)

// multiple paths
scala> (__ \\ "key1")(js)
res17: List[play.api.libs.json.JsValue] = List("value1", "value2")

{% endcodeblock %}	


<br/>
<br/>
# <a name="reads">Reads[T] is now a validator</a>

## <a name="reads-2_0">Reads in Play2.0.x</a>

Do you remember how you had to write a Json `Reads[T]` in `Play2.0.x` ?  
You had to override the `reads` function.  

{% codeblock lang:scala %}
trait Reads[A] {
  self =>
  /**
   * Convert the JsValue into a A
   */
  def reads(json: JsValue): A
}
{% endcodeblock %}	

Take the following simple case class that you want to map on JSON structure:
{% codeblock lang:scala %}
case class Creature(
  name: String, 
  isDead: Boolean, 
  weight: Float
)
{% endcodeblock %}	

In `Play2.0.x`, you would write your reader as following:
{% codeblock lang:scala %}
import play.api.libs.json._

implicit val creatureReads = new Reads[Creature] {
  def reads(js: JsValue): Creature = {
    Creature(
      (js \ "name").as[String],
      (js \ "isDead").as[Boolean],
      (js \ "weight").as[Float]
    )
  }
}

scala> val js = Json.obj( "name" -> "gremlins", "isDead" -> false, "weight" -> 1.0F)
scala> val c = js.as[Creature] 
c: Creature("gremlins", false, 1.0F)
{% endcodeblock %}	

Easy isn't it ?
So what's the problem if it's so easy?

Imagine, you pass the following JSON with a missing field:
{% codeblock lang:scala %}
val js = Json.obj( "name" -> "gremlins", "weight" -> 1.0F)
{% endcodeblock %}	

What happens?
{% codeblock lang:scala %}
java.lang.RuntimeException: Boolean expected
	at play.api.libs.json.DefaultReads$BooleanReads$.reads(Reads.scala:98)
	at play.api.libs.json.DefaultReads$BooleanReads$.reads(Reads.scala:95)
	at play.api.libs.json.JsValue$class.as(JsValue.scala:56)
	at play.api.libs.json.JsUndefined.as(JsValue.scala:70)
{% endcodeblock %}	

**WHATTTTTTT????**
Yes ugly RuntimeException (not even subtyped) but you can work around it using `JsValue.asOpt[T]` :)

{% codeblock lang:scala %}
scala> val c: Option[Creature] = js.asOpt[Creature]
c: None
{% endcodeblock %}	

**Cool but you only know that the deserialization `Json => Creature` failed but not where or on which field(s)?**

<br/>
<br/>
## <a name="reads-2_1">Reads in Play2.1</a>
We couldn't keep this imperfect API as is and in `Play2.1`, the `Reads[T]` API has changed into :

{% codeblock lang:scala %}
trait Reads[A] {
  self =>
  /**
   * Convert the JsValue into a A
   */
  def reads(json: JsValue): JsResult[A]
}
{% endcodeblock %}	

> Yes you have to refactor all your existing custom Reads but you'll see you'll get lots of new interesting features…

So you remark immediately `JsResult[A]` which is a very simple structure looking a bit like an Either applied to our specific problem.  

**`JsResult[A]` can be of 2 types**:

- **`JsSuccess[A]` when `reads`succeeds**

{% codeblock lang:scala %}
case class JsSuccess[A](
  value: T, // the value retrieved when deserialization JsValue => A worked
  path: JsPath = JsPath() // the root JsPath where this A was read in the JsValue (by default, it's the root of the JsValue)
) extends JsResult[T]

// To create a JsSuccess from a value, simply do:
val success = JsSuccess(Creature("gremlins", false, 1.0))

{% endcodeblock %}	

- **`JsError[A]` when `reads` fails**

> Please note the greatest advantage of JsError is that it's a cumulative error which can store several errors discovered in the Json at different JsPath


{% codeblock lang:scala %}
case class JsError(
  errors: Seq[(JsPath, Seq[ValidationError])]  
  // the errors is a sequence of JsPath locating the path 
  // where there was an error in the JsValue and validation 
  // errors for this path
) extends JsResult[Nothing]

// ValidationError is a simple message with arguments (which can be mapped on localized messages)
case class ValidationError(message: String, args: Any*)

// To create a JsError, there are a few helpers and for ex:
val errors1 = JsError( __ \ 'isDead, ValidationError("validate.error.missing", "isDead") )
val errors2 = JsError( __ \ 'name, ValidationError("validate.error.missing", "name") )

// Errors are cumulative which is really interesting
scala> val errors = errors1 ++ errors2
errors: JsError(List((/isDead,List(ValidationError(validate.error.missing,WrappedArray(isDead)))), (/name,List(ValidationError(validate.error.missing,WrappedArray(name))))))
{% endcodeblock %}	

So what's interesting there is that `JsResult[A]` is a monadic structure and can be used with classic functions of such structures: 

- `flatMap[X](f: A => JsResult[X]): JsResult[X]`  
- `fold[X](invalid: Seq[(JsPath, Seq[ValidationError])] => X, valid: A => X)`  
- `map[X](f: A => X): JsResult[X]`  
- `filter(p: A => Boolean)`  
- `collect[B](otherwise:ValidationError)(p:PartialFunction[A,B]): JsResult[B]` 
- `get: A`

And some sugar such :

- `asOpt`  
- `asEither`
- … 


### Reads[A] has become a validator

As you may understand, using the new `Reads[A]`, you don't only deserialize a JsValue into another structure but you really **validate** the JsValue and retrieve all the validation errors.  
BTW, in `JsValue`, a new function called `validate` has appeared:

{% codeblock lang:scala %}
trait JsValue {
…
  def validate[T](implicit _reads: Reads[T]): JsResult[T] = _reads.reads(this)
  
  // same behavior but it throws a specific RuntimeException JsResultException now
  def as[T](implicit fjs: Reads[T]): T
  // exactly the same behavior has before
  def asOpt[T](implicit fjs: Reads[T]): Option[T]
…
}

// You can now write if you got the right implicit in your scope
val res: JsResult[Creature] = js.validate[Creature])
{% endcodeblock %}	

<br/>
### Manipulating a JsResult[A]

So when manipulating a `JsResult`, you don't access the value directly and it's preferable to use `map/flatmap/fold` to modify the value.

{% codeblock lang:scala %}
val res: JsResult[Creature] = js.validate[Creature]

// managing the success/error and potentially return something
res.fold(
  valid = { c => println( c ); c.name },
  invalid = { e => println( e ); e }
)

// getting the name directly (using the get can throw a NoSuchElementException if it's a JsError)
val name: JsSuccess[String] = res.map( creature => creature.name ).get

// filtering the result
val name: JsSuccess[String] = res.filter( creature => creature.name == "gremlins" ).get

// a classic Play action
def getNameOnly = Action(parse.json) { request =>
  val json = request.body
  json.validate[Creature].fold(
    valid = ( res => Ok(res.name) ),
    invalid = ( e => BadRequest(e.toString) )
  )
}
{% endcodeblock %}	


### Rewriting the Reads[T] with JsResult[A]

The `Reads[A]` API returning a JsResult, you can't write your `Reads[A]` as before as you must return a JsResult gathering all found errors.  
You could imagine simply compose Reads[T] with flatMap :

{% codeblock lang:scala %}
// DO NOT USE, WRONG CODE
implicit val creatureReads = new Reads[Creature] {
  def reads(js: JsValue): JsResult[Creature] = {
    (js \ "name").validate[String].flatMap{ name => 
      (js \ "isDead").validate[Boolean].flatMap { isDead =>
	    (js \ "weight").validate[Float].map { weight =>
	      Creature(name, isDead, weight)
	    }
	  }
	}
  }
}
{% endcodeblock %}	

>Remember the main purpose of `JsResult` is to gather all found errors while validating the JsValue.

`JsResult.flatMap` is pure monadic function (if you don't know what it is, don't care about it, you can understand without it) implying that the function that you pass to `flatMap()` is called only if the result is a `JsSuccess` else it just returns the `JsError`.  
This means the previous code won't aggregate all errors found during validation and will stop at first error which is exactly what we don't want.

Actually, Monad pattern is not good in our case because we are not just composing Reads but we expect combining them following the schema:
{% codeblock lang:scala %}
Reads[String] AND Reads[Boolean] AND Reads[Float] 
  => Reads[(String, Boolean, Float)] 
     => Reads[Creature]
{% endcodeblock %}	

> So we need something else to be able to combine our Reads and this is the greatest new feature that `Play2.1` brings for JSON :  
> **THE READS combinators with JsPath**
<br/>

> If you want more theoretical aspects about the way it was implemented based on generic functional structures adapted to our needs, you can read this post ["Applicatives are too restrictive, breaking Applicatives and introducing Functional Builders"](http://sadache.tumblr.com/post/30955704987/applicatives-are-too-restrictive-breaking-applicativesfrom) written by [@sadache](http://www.github.com/sadache) 

### Rewriting the Reads[T] with combinators

Go directly to the example as practice is often the best :
{% codeblock lang:scala %}
// IMPORTANT import this to have the required tools in your scope
import play.api.libs.json.util._

implicit val creatureReads = (
  (__ \ "name").read[String] and
  (__ \ "isDead").read[Boolean] and
  (__ \ "weight").read[Float]
)(Creature.apply _)  

// or in a simpler way as case class has a companion object with an apply function
implicit val creatureReads = (
  (__ \ "name").read[String] and
  (__ \ "isDead").read[Boolean] and
  (__ \ "weight").read[Float]
)(Creature)  

// or using the operators inspired by Scala parser combinators for those who know them
implicit val creatureReads = (
  (__ \ "name").read[String] ~
  (__ \ "isDead").read[Boolean] ~
  (__ \ "weight").read[Float]
)(Creature)  

{% endcodeblock %}	


So there is nothing quite complicated, isn't it?

#####`(__ \ "name")` is the `JsPath` where you gonna apply `read[String]`
<br/>
#####`and` is just an operator meaning `Reads[A] and Reads[B] => Builder[Reads[A ~ B]]`
  - `A ~ B` just means `Combine A and B` but it doesn't suppose the way it is combined (can be a tuple, an object, whatever…)
  - `Builder` is not a real type but I introduce it just to tell that the operator `and` doesn't create directly a `Reads[A ~ B]` but an intermediate structure that is able to build a `Reads[A ~ B]` or to combine with another `Reads[C]`
<br/>   
#####`(…)(Creature)` builds a `Reads[Creature]`
  - `(__ \ "name").read[String] and (__ \ "isDead").read[Boolean] and (__ \ "weight").read[Float]` builds a `Builder[Reads[String ~ Boolean ~ Float])]` but you want a `Reads[Creature]`.  
  - So you apply the `Builder[Reads[String ~ Boolean ~ String])]` to the function `Creature.apply = (String, Boolean, Float) => Creature` constructor to finally obtain a `Reads[Creature]`

Try it:

{% codeblock lang:scala %}
scala> val js = Json.obj( "name" -> "gremlins", "isDead" -> false, "weight" -> 1.0F)
scala> js.validate[Creature] 
res1: play.api.libs.json.JsResult[Creature] = JsSuccess(Creature(gremlins,false,1.0),) // nothing after last comma because the JsPath is ROOT by default

{% endcodeblock %}	


Now what happens if you have an error now?

{% codeblock lang:scala %}
scala> val js = Json.obj( "name" -> "gremlins", "weight" -> 1.0F)
scala> js.validate[Creature] 
res2: play.api.libs.json.JsResult[Creature] = JsError(List((/isDead,List(ValidationError(validate.error.missing-path,WrappedArray())))))

{% endcodeblock %}	

Explicit, isn't it?

### Complexifying the case

Ok, I see what you think : what about more complex cases where you have several constraints on a field and embedded Json in Json and recursive classes and whatever…

Let's imagine our creature:

- is a relatively modern creature having an email and hating email addresses having less than 5 characters for a reason only known by the creature itself. 
- may have 2 favorites data: 
    - 1 String (called "string" in JSON) which shall not be "ni" (because it loves Monty Python too much to accept this) and then to skip the first 2 chars
    - 1 Int (called "number" in JSON) which can be less than 86 or more than 875 (don't ask why, this is creature with a different logic than ours)
- may have friend creatures
- may have an optional social account because many creatures are not very social so this is quite mandatory

Now the class looks like:

{% codeblock lang:scala %}
case class Creature(
  name: String, 
  isDead: Boolean, 
  weight: Float,
  email: String, // email format and minLength(5)
  favorites: (String, Int), // the stupid favorites
  friends: List[Creature] = Nil, // yes by default it has no friend
  social: Option[String] = None // by default, it's not social
)
{% endcodeblock %}	

`Play2.1` provide lots of generic Reads helpers:

- `JsPath.read[A](implicit reads:Reads[A])` can be passed a custom `Reads[A]` which is applied to the JSON content at this JsPath. So with this property, you can compose hierarchically `Reads[T]` which corresponds to JSON tree structure.
- `JsPath.readOpt` allows `Reads[Option[T]]`
- `Reads.email` which validates the String has email format  
- `Reads.minLength(nb)` validates the minimum length of a String
- `Reads[A] or Reads[A] => Reads[A]` operator is a classic `OR` logic operator
- `Reads[A] keepAnd Reads[B] => Reads[A]` is an operator that tries `Reads[A]` and `Reads[B]` but only keeps the result of `Reads[A]` (For those who know Scala parser combinators `keepAnd == <~` )
- `Reads[A] andKeep Reads[B] => Reads[B]` is an operator that tries `Reads[A]` and `Reads[B]` but only keeps the result of `Reads[B]` (For those who know Scala parser combinators `andKeep == ~>` )

{% codeblock lang:scala %}
// import just Reads helpers in scope
import play.api.libs.json._
import play.api.libs.json.util._
import play.api.libs.json.Reads._
import play.api.data.validation.ValidationError

// defines a custom reads to be reused
// a reads that verifies your value is not equal to a give value
def notEqualReads[T](v: T)(implicit r: Reads[T]): Reads[T] = Reads.filterNot(ValidationError("validate.error.unexpected.value", v))( _ == v )

def skipReads(implicit r: Reads[String]): Reads[String] = r.map( _.substring(2) )

implicit val creatureReads: Reads[Creature] = (
  (__ \ "name").read[String] and
  (__ \ "isDead").read[Boolean] and
  (__ \ "weight").read[Float] and
  (__ \ "email").read(email keepAnd minLength[String](5)) and
  (__ \ "favorites").read( 
  	(__ \ "string").read[String]( notEqualReads("ni") andKeep skipReads ) and
  	(__ \ "number").read[Int]( max(86) or min(875) )
  	tupled
  ) and
  (__ \ "friends").lazyRead( list[Creature](creatureReads) ) and
  (__ \ "social").readOpt[String]
)(Creature)  
{% endcodeblock %}

Many things above can be understood very logically but let's explain a bit:

##### `(__ \ "email").read(email keepAnd minLength[String](5))`

- As explained previously, `(__ \ "email").read(…)` applies the `Reads[T]` passed in parameter to function `read` at the given JsPath `(__ \ "email")`
- `email keepAnd minLength[String](5) => Reads[String]` is a Js validator that verifies JsValue:
    1. is a String : `email: Reads[String]` so no need to specify type here
    2. has email format
    3. has min length of 5 (we precise the type here because minLength is a generic `Reads[T]`)
- Why not `email and minLength[String](5)`? because, as explained previously, it would generate a `Builder[Reads[(String, String)]]` whereas you expect a `Reads[String]`. The `keepAnd` operator (aka `<~`) does what we expect: it validates both sides but if succeeded, it keeps only the result on left side.
<br/>
##### `notEqualReads("ni") andKeep skipReads`

- No need to write `notEqualReads[String]("ni")` because `String` type is inferred from `(__ \ "knight").read[String]` (the power of Scala typing engine)
- `skipReads` is a customReads that skips the first 2 chars
- `andKeep` operator (aka `~>`) is simple to undestand : it validates the left and right side and if both succeeds, only keeps the result on right side. In our case, only the result of `skipReads` is kept and not the result of `notEqualReads`.
<br/>
##### `max(86) or min(875)`

- Nothing to explain there, isn't it ? `or` is the classic `OR`logic operator, nothing else
<br/>
##### `(__ \ "favorites").read(…)`

    (__ \ "string").read[String]( notEqualReads("ni") andKeep notEqualReads("swallow") ) and
    (__ \ "number").read[Int]( max(86) or min(875) )
    tupled

- Remember that `(__ \ "string").read[String](…) and (__ \ "number").read[Int](…) => Builder[Reads[(String, Int)]]`  
- What means `tupled` ? 
`Builder[Reads[(String, Int)]]` can be used with a case class `apply` function to build the `Reads[Creature]` for ex. But it provides also `tupled` which is quite easy to understand : it _"tuplizes"_ your Builder: `Builder[Reads[(A, B)]].tupled => Reads[(A, B)]` 
- Finally `(__ \ "favorites").read(Reads[(String, Int)]` will validate a `(String, Int)` which is the expected type for field `favorites`
<br/>
##### `(__ \ "friend").lazyRead( list[Creature](creatureReads) )`

This is the most complicated line in this code. But you can understand why: the `friend` field is recursive on the `Creature` class itself so it requires a special treatment.

- `list[Creature](…)` creates a `Reads[List[Creature]]`  
- `list[Creature](creatureReads)` passes explicitly `creatureReads` as an argument because it's recursive and Scala requires that to resolve it. Nothing too complicated…
- `(__ \ "friend").lazyRead[A](r: => Reads[A]))` : `lazyRead` expects a `Reads[A]` value _passed by name_ to allow the type recursive construction. This is the only refinement that you must keep in mind in this very special recursive case.
<br/>
##### `(__ \ "social").readOpt[String]`

Nothing quite complicated to understand: we need to read an option and `readOpt` helps in doing this.

Now we can use this `Reads[Creature]`

{% codeblock lang:scala %}
val gizmojs = Json.obj( 
  "name" -> "gremlins", 
  "isDead" -> false, 
  "weight" -> 1.0F,
  "email" -> "gizmo@midnight.com",
  "favorites" -> Json.obj("string" -> "alpha", "number" -> 85),
  "friends" -> Json.arr(),
  "social" -> "@gizmo"
)

scala> val gizmo = gizmojs.validate[Creature] 
gizmo: play.api.libs.json.JsResult[Creature] = JsSuccess(Creature(gremlins,false,1.0,gizmo@midnight.com,(pha,85),List(),Some(@gizmo)),)

val shaunjs = Json.obj( 
  "name" -> "zombie", 
  "isDead" -> true, 
  "weight" -> 100.0F,
  "email" -> "shaun@dead.com",
  "favorites" -> Json.obj("string" -> "brain", "number" -> 2),
  "friends" -> Json.arr( gizmojs))

scala> val shaun = shaunjs.validate[Creature] 
shaun: play.api.libs.json.JsResult[Creature] = JsSuccess(Creature(zombie,true,100.0,shaun@dead.com,(ain,2),List(Creature(gremlins,false,1.0,gizmo@midnight.com,(alpha,85),List(),Some(@gizmo))),None),)

val errorjs = Json.obj( 
  "name" -> "gremlins", 
  "isDead" -> false, 
  "weight" -> 1.0F,
  "email" -> "rrhh",
  "favorites" -> Json.obj("string" -> "ni", "number" -> 500),
  "friends" -> Json.arr()
)

scala> errorjs.validate[Creature] 
res0: play.api.libs.json.JsResult[Creature] = JsError(List((/favorites/string,List(ValidationError(validate.error.unexpected.value,WrappedArray(ni)))), (/email,List(ValidationError(validate.error.email,WrappedArray()), ValidationError(validate.error.minlength,WrappedArray(5)))), (/favorites/number,List(ValidationError(validate.error.max,WrappedArray(86)), ValidationError(validate.error.min,WrappedArray(875))))))
{% endcodeblock %}	

<br/>
<br/>
## <a name="conclusion">Conclusion & much more…</a>

So here is the end of this 1st part not to make a too long article.
But there are so many other features to explain and demonstrate and there will other parts very soon!!!

> `Reads[T]` combinators are cool but you know what? `Writes[T]` & `Format[T]` can be combined too!

Let's tease a bit: 

{% codeblock lang:scala %}
implicit val creatureWrites = (
  (__ \ "name").write[String] and
  (__ \ "isDead").write[Boolean] and
  (__ \ "weight").write[Float]
)(Creature) 
{% endcodeblock %}	


> Next parts about `Writes[T]` combinators and another intriguing feature that I'll name `Json Generators` ;)
 
Have fun... 