---
layout: post
title: "Unveiling Play 2.1 Json API - Bonus : JSON Inception (based on Scala 2.10 Macros)"
date: 2012-11-11 11:10
comments: true
external-url: 
categories: [playframework,play2.1,json,serialization,validation,combinators,applicative, scala macro]
keywords: play 2.1, json, json serialization, json transform, json validation, json cumulative errors, play framework, combinators, applicative, scala macro
---

> A relatively short article, this time, to present an experimental feature developed by Sadek Drobi ([@sadache](http://www.twitter.com/sadache)) & I ([@mandubian](http://www.twitter.com/mandubian)) and that we've decided to integrate into Play-2.1 because we think it's really interesting and useful. 

<br/>
<br/>
# <a name="wtf-inception">WTF is JSON Inception???</a>

## <a name="wtf-inception-boring">Writing a default case class Reads/Writes/Format is so boring!!!</a>

Remember how you write a `Reads[T]` for a case class.

{% codeblock lang:json %}
import play.api.libs.json._
import play.api.libs.functional.syntax._

case class Person(name: String, age: Int, lovesChocolate: Boolean)

implicit val personReads = (
	(__ \ 'name).reads[String] and
	(__ \ 'age).reads[Int] and
	(__ \ 'lovesChocolate).reads[Boolean]
)(Person)
{% endcodeblock %}	

So you write 4 lines for this case class.  
You know what? We have had a few complaints from some people who think it's not cool to write a `Reads[TheirClass]` because usually Java JSON frameworks like Jackson or Gson do it behind the curtain without writing anything.  
We tried to argue that Play2.1 JSON serializers/deserializers were:

- completely typesafe, 
- fully compiled,
- nothing was performed using introspection/reflection at runtime.  

But it was not enough.

We believe this is a really good approach so we persisted and proposed:

- JSON simplified syntax
- JSON combinators
- JSON transformers

But it was still not enough for those people because they still had to write those 4 lines.

## <a name="wtf-inception-minimalist">Let's be minimalist</a>
As we are perfectionnist, now we propose a new way of writing the same code:

{% codeblock lang:json %}
import play.api.libs.json._
import play.api.libs.functional.syntax._

case class Person(name: String, age: Int, lovesChocolate: Boolean)

implicit val personReads = Json.reads[Person]
{% endcodeblock %}	

1 line only.  
Questions you may ask immediately:

> Does it use runtime bytecode enhancement? -> NO

> Does it use runtime introspection? -> NO

> Does it break type-safety? -> NO

**So what???**

> As I'm currently in a mood of creating new expressions, after creating **JSON coast-to-coast design**, let's call it **JSON INCEPTION** (cool word, quite puzzling isn't it? ;)
<br/>
<br/>
# <a name="json-incept">JSON Inception</a>

## <a name="json-incept-eq">Code Equivalence</a>
As explained just before:

{% codeblock lang:json %}
implicit val personReads = Json.reads[Person]

// IS STRICTLY EQUIVALENT TO writing

implicit val personReads = (
	(__ \ 'name).reads[String] and
	(__ \ 'age).reads[Int] and
	(__ \ 'lovesChocolate).reads[Boolean]
)(Person)
{% endcodeblock %}	

## <a name="json-incept">Inception equation</a>

Here is the equation describing the windy _Inception_ concept:

{% codeblock lang:json %}
(Case Class INSPECTION) + (Code INJECTION) + (COMPILE Time) = INCEPTION
{% endcodeblock %}	

<br/>
####Case Class Inspection
As you may deduce by yourself, in order to ensure preceding code equivalence, we need :

- to inspect `Person` case class, 
- to extract the 3 fields `name`, `age`, `lovesChocolate` and their types,
- to resolve typeclasses implicits,
- to find `Person.apply`.


<br/>
####INJECTION??? Injjjjjectiiiiion….??? injectionnnnnnnn????
No I stop you immediately…  

>**Code injection is not dependency injection…**  
>No Spring behind inception… No IOC, No DI… No No No ;)  
  
I used this term on purpose because I know that injection is now linked immediately to IOC and Spring. But I'd like to re-establish this word with its real meaning.  
Here code injection just means that **we inject code at compile-time into the compiled scala AST** (Abstract Syntax Tree).

So `Json.reads[Person]` is compiled and replaced in the compile AST by:

{% codeblock lang:json %}
(
	(__ \ 'name).reads[String] and
	(__ \ 'age).reads[Int] and
	(__ \ 'lovesChocolate).reads[Boolean]
)(Person)
{% endcodeblock %}	

Nothing less, nothing more…

<br/>
####COMPILE-TIME
Yes everything is performed at compile-time.  
No runtime bytecode enhancement.  
No runtime introspection.  

> As everything is resolved at compile-time, you will have a compile error if you did not import the required implicits for all the types of the fields.

<br/>
# <a name="scala-macros">Json inception is Scala 2.10 Macros</a>

We needed a Scala feature enabling:

- compile-time code enhancement
- compile-time class/implicits inspection
- compile-time code injection

This is enabled by a new experimental feature introduced in Scala 2.10: [Scala Macros](http://scalamacros.org/)  

Scala macros is a new feature (still experimental) with a huge potential. You can :

- introspect code at compile-time based on Scala reflection API, 
- access all imports, implicits in the current compile context
- create new code expressions, generate compiling errors and inject them into compile chain.

Please note that:

- **We use Scala Macros because it corresponds exactly to our requirements.**
- **We use Scala macros as an enabler, not as an end in itself.**
- **The macro is a helper that generates the code you could write by yourself.**
- **It doesn't add, hide unexpected code behind the curtain.**

As you may discover, writing a macro is not a trivial process since your macro code executes in the compiler runtime (or universe).  

    So you write macro code 
      that is compiled and executed 
      in a runtime that manipulates your code 
         to be compiled and executed 
         in a future runtime…           
**That's also certainly why we called it *Inception* ;)**

So it requires some mental exercises to follow exactly what you do. The API is also quite complex and not fully documented yet. Therefore, you must persevere when you begin using macros.

I'll certainly write other articles about Scala macros because there are lots of things to say.  
This article is also meant **to begin the reflection about the right way to use Scala Macros**.  
Great power means greater responsabilities so it's better to discuss all together and establish a few good manners…

<br/>
# <a name="writes-format">Writes[T] & Format[T]</a>

>Please remark that JSON inception just works for case class having `unapply/apply` functions.

Naturally, you can also _incept_ `Writes[T]`and `Format[T]`.

## <a name="writes">Writes[T]</a>

{% codeblock lang:json %}
import play.api.libs.json._
import play.api.libs.functional.syntax._
 
implicit val personWrites = Json.writes[Person]
{% endcodeblock %}	

## <a name="format">Format[T]</a>

{% codeblock lang:json %}
import play.api.libs.json._
import play.api.libs.json.util._
import play.api.libs.functional.syntax._

implicit val personWrites = Json.format[Person]
{% endcodeblock %}	

# <a name="conclusion">Conclusion</a>

> With the so-called **JSON inception**, we have added a **helper** providing a trivial way to define your **default typesafe `Reads[T]/Writes[T]/Format[T]` for case classes**.  

If anyone tells me there is still 1 line to write, I think I might become unpolite ;)

Have Macrofun ;););)