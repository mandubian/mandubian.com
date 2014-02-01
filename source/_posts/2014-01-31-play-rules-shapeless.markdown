---
layout: post
title: "New Play 2.3 Validation API : breaking the 22 limits with Shapeless"
date: 2014-01-31 08:08
comments: true
external-url:
categories: [scala,play framework,shapeless,validation,json,functional programming,typesafe]
keywords: scala,play framework,shapeless,validation,json,functional programming,typesafe
---

The code & sample apps can be found on [Github](https://github.com/mandubian/shapeless-rules)

After 5 months studying theories deeper & deeper on my free-time and preparing 3 talks for [scala.io](http://www.scala.io) & [ping-conf](http://www.ping-conf.com) with my friend [Julien Tournay aka @skaalf](http://www.twitter.com/skaalf), I'm back blogging and I've got a few more ideas of articles to come...

>If you're interested in those talks, you can find pingconf videos here:
>
>- [Entropic history & Play2.3 new validation API](http://www.ustream.tv/recorded/42778238)
>- [Play2, scalaz-stream & SciFi](http://www.ustream.tv/recorded/42802705)

Let's go back to our today's subject : **Incoming Play2.3/Scala generic validation API & more**.

[Julien Tournay aka @skaalf](http://www.twitter.com/skaalf) has been working a lot for a few months developing this new API and has just published an article previewing [Play 2.3 generic validation API](http://jto.github.io/articles/play_new_validation_api/).

This new API is just the logical extension of play2/Scala Json API (that I've been working & promoting those 2 last years) pushing its principles far further by allowing validation on any data types.

This new API is a real step further as it will progressively propose a common API for all validations in Play2/Scala (Form/Json/XML/...). It proposes an even more robust design relying on very strong theoretical ground making it very reliable & typesafe.

Julien has written his [article](http://jto.github.io/articles/play_new_validation_api/) presenting the new API basics and he also found time to write great documentation for this new validation API. I must confess Json API doc was quite messy but I've never found freetime (and courage) to do better. So I'm not going to spend time on basic features of this new API and I'm going to target advanced features to open your minds about the power of this new API.

> Let's have fun with this new APi & [Shapeless](https://github.com/milessabin/shapeless), this fantastic tool for higher-rank polymorphism & type-safety!

<br/>
## Warm-up with Higher-kind Zipping of Rules

A really cool & new feature of Play2.3 generic validation API is its ability to compose validation Rules in chains like:

{% codeblock lang:scala %}
val rule1: Rule[A, B] = ...
val rule2: Rule[B, C] = ...

val rule3: Rule[A, C] = rule1 compose rule2
{% endcodeblock %}

_In Play2.1 Json API, you couldn't do that (you could only map on Reads)._

Moreover, with new validation API, as in Json API, you can use macros to create basic validators from case-classes.

{% codeblock lang:scala %}
case class FooBar(foo: String, bar: Int, foo2: Long)

val rule = Rule.gen[JsValue, FooBar]

/** Action to validate Json:
  * { foo: "toto", bar: 5, foo2: 2 }
  */
def action = Action(parse.json) { request =>
  rule.validate(request.body) map { foobar =>
    Ok(foobar.toString)
  } recoverTotal { errors =>
    BadRequest(errors.toString)
  }
}
{% endcodeblock %}

Great but sometimes not enough as you would like to add custom validations on your class.
For example, you want to verify :

- `foo` isn't empty
- `bar` is >5
- `foo2` is <10

For that you can't use the macro and must write your caseclass Rule yourself.

{% codeblock lang:scala %}
case class FooBar(foo: String, bar: Int, foo2: Long)

import play.api.data.mapping.json.Rules
import Rules._

val rule = From[JsValue] { __ =>
  (
    (__ \ "foo").read[String](notEmpty) ~
    (__ \ "bar").read[Int](min(5)) ~
    (__ \ "foo2").read[Long](max(10))
  )(FooBar.apply _)
}
{% endcodeblock %}

_Please note the new `From[JsValue]`: if it were Xml, it would be `From[Xml]`, genericity requires some more info._

Ok that's not too hard but sometimes you would like to use first the macro and after those primary type validations, you want to refine with custom validations. Something like:

{% codeblock lang:scala %}
Rule.gen[JsValue, FooBar] +?+?+ ( (notEmpty:Rule[String, String]) +: (min(5):Rule[Int, Int]) +: (min(10L):Rule[Long,Long]) )
// +?+?+ is a non-existing operator meaning "compose"
{% endcodeblock %}

As you may know, you can't do use this `+:` from Scala `Sequence[T]` as this list of Rules is typed heterogenously and `Rule[I, O]` is invariant.

So we are going to use Shapeless heterogenous Hlist for that:

{% codeblock lang:scala %}
val customRules = 
  (notEmpty:Rule[String, String]) :: 
  (min(5):Rule[Int, Int]) :: 
  (min(10L):Rule[Long,Long]) ::
  HNil
// customRules is inferred Rule[String, String]) :: Rule[Int, Int] :: Rule[Long,Long]
{% endcodeblock %}

<br/>
<br/>
#### How to compose `Rule[JsValue, FooBar]` with `Rule[String, String]) :: Rule[Int, Int] :: Rule[Long,Long]` ?
<br/>
We need to convert `Rule[JsValue, FooBar]` to something like `Rule[JsValue, T <: HList]`.

Based on Shapeless `Generic[T]`, we can provide a nice little new conversion API `.hlisted`:

{% codeblock lang:scala %}
val rule: Rule[JsValue, String :: Int :: Long :: HNil] = Rule.gen[JsValue, FooBar].hlisted
{% endcodeblock %}

`Generic[T]` is able to convert any caseclass from Scala from/to Shapeless HList (& CoProduct). 

So we can validate a case class with the macro and get a `Rule[JsValue, T <: HList]` from it.

<br/>
<br/>
#### How to compose `Rule[JsValue, String :: Int :: Long :: HNil]` with `Rule[String, String]) :: Rule[Int, Int] :: Rule[Long,Long]`?
<br/>
Again, using Shapeless Polymorphic and HList RightFolder, we can implement a function :

{% codeblock lang:scala %}
Rule[String, String]) :: Rule[Int, Int] :: Rule[Long,Long] :: HNil) => 
    Rule[String :: Int :: Long :: HNil, String :: Int :: Long :: HNil]
{% endcodeblock %}

> This looks like some higher-kind zip function, let's call it `HZIP`.

<br/>
<br/>
#### Now, we can compose them...
<br/>
{% codeblock lang:scala %}
val ruleZip = Rule.gen[JsValue, FooBar].hlisted compose hzip(
  (notEmpty:Rule[String, String]) :: 
  (min(5):Rule[Int, Int]) :: 
  (min(10L):Rule[Long,Long]) :: 
  HNil
)
{% endcodeblock %}

#### Finally, let's wire all together in a Play action:

{% codeblock lang:scala %}
def hzipper = Action(parse.json) { request =>
  ruleZip.validate(request.body) map { foobar =>
    Ok(foobar.toString)
  } recoverTotal { errors =>
    BadRequest(errors.toString)
  }
}

// OK case
{
  "foo" : "toto",
  "bar" : 5,
  "foo2" : 5
} => toto :: 5 :: 8 :: HNil

// KO case
{
  "foo" : "",
  "bar" : 2,
  "foo2" : 12
} => Failure(List(
  ([0],List(ValidationError(error.max,WrappedArray(10)))), 
  ([1],List(ValidationError(error.min,WrappedArray(5)))), 
  ([2],List(ValidationError(error.required,WrappedArray())))
))
{% endcodeblock %}

>As you can see, the problem in this approach is that we lose the path of Json.
> Anyway, this can give you a few ideas!
> Now let's do something really useful...

<br/>
<br/>
## Higher-kind Fold of Rules to break the 22 limits

As in Play2.1 Json API, the new validation API provides an applicative builder which allows the following:

{% codeblock lang:scala %}
(Rule[I, A] ~ Rule[I, B] ~ Rule[I, C]).tupled => Rule[I, (A, B, C)]

(Rule[I, A] ~ Rule[I, B] ~ Rule[I, C])(MyClass.apply _) => Rule[I, MyClass]
{% endcodeblock %}

But, in Play2.1 Json API and also in new validation API, all functional combinators are limited by the famous Scala 22 limits.

In Scala, you **CAN'T** write :

- a case-class with >22 fields
- a `Tuple23`

**So you can't do `Rule[JsValue, A] ~ Rule[JsValue, B] ~ ...` more than 22 times.**

Nevertheless, sometimes you receive huge JSON with much more than 22 fields in it. Then you have to build more complex models like case-classes embedding case-classes... Shameful, isn't it...

Let's be shameless with Shapeless HList which enables to have unlimited heterogenously typed lists!

So, with HList, we can write :

{% codeblock lang:scala %}
val bigRule = 
  (__  \ "foo1").read[String] ::
  (__  \ "foo2").read[String] ::
  (__  \ "foo3").read[String] ::
  (__  \ "foo4").read[String] ::
  (__  \ "foo5").read[String] ::
  (__  \ "foo6").read[String] ::
  (__  \ "foo7").read[String] ::
  (__  \ "foo8").read[String] ::
  (__  \ "foo9").read[String] ::
  (__  \ "foo10").read[Int] ::
  (__  \ "foo11").read[Int] ::
  (__  \ "foo12").read[Int] ::
  (__  \ "foo13").read[Int] ::
  (__  \ "foo14").read[Int] ::
  (__  \ "foo15").read[Int] ::
  (__  \ "foo16").read[Int] ::
  (__  \ "foo17").read[Int] ::
  (__  \ "foo18").read[Int] ::
  (__  \ "foo19").read[Int] ::
  (__  \ "foo20").read[Boolean] ::
  (__  \ "foo21").read[Boolean] ::
  (__  \ "foo22").read[Boolean] ::
  (__  \ "foo23").read[Boolean] ::
  (__  \ "foo25").read[Boolean] ::
  (__  \ "foo26").read[Boolean] ::
  (__  \ "foo27").read[Boolean] ::
  (__  \ "foo28").read[Boolean] ::
  (__  \ "foo29").read[Boolean] ::
  (__  \ "foo30").read[Float] ::
  (__  \ "foo31").read[Float] ::
  (__  \ "foo32").read[Float] ::
  (__  \ "foo33").read[Float] ::
  (__  \ "foo34").read[Float] ::
  (__  \ "foo35").read[Float] ::
  (__  \ "foo36").read[Float] ::
  (__  \ "foo37").read[Float] ::
  (__  \ "foo38").read[Float] ::
  (__  \ "foo39").read[Float] ::
  (__  \ "foo40").read[List[Long]] ::
  (__  \ "foo41").read[List[Long]] ::
  (__  \ "foo42").read[List[Long]] ::
  (__  \ "foo43").read[List[Long]] ::
  (__  \ "foo44").read[List[Long]] ::
  (__  \ "foo45").read[List[Long]] ::
  (__  \ "foo46").read[List[Long]] ::
  (__  \ "foo47").read[List[Long]] ::
  (__  \ "foo48").read[List[Long]] ::
  (__  \ "foo49").read[List[Long]] ::
  (__  \ "foo50").read[JsNull.type] ::
  HNil

// inferred as Rule[JsValue, String] :: Rule[JsValue, String] :: ... :: Rule[JsValue, List[Long]] :: HNil
{% endcodeblock %}

That's cool but we want the `::` operator to have the same `applicative builder behavior as the `~/and` operator:

{% codeblock lang:scala %}
Rule[JsValue, String] :: Rule[JsValue, Long] :: Rule[JsValue, Float] :: HNil =>
  Rule[JsValue, String :: Long :: Float :: HNil]
{% endcodeblock %}

> This looks like a higher-kind fold so let's call that `HFOLD`.

We can build this `hfold` using Shapeless polymorphic functions & RighFolder.

_In a next article, I may write about coding such shapeless feature. Meanwhile, you'll have to discover the code on [Github](https://github.com/mandubian/shapeless-rules) as it's a bit hairy but very interesting ;)_

Gathering everything, we obtain the following:

{% codeblock lang:scala %}
/* Rules Folding */
val ruleFold = From[JsValue]{ __ =>
  hfold[JsValue](
    (__  \ "foo1").read[String] ::
    (__  \ "foo2").read[String] ::
    (__  \ "foo3").read[String] ::
    (__  \ "foo4").read[String] ::
    (__  \ "foo5").read[String] ::
    (__  \ "foo6").read[String] ::
    (__  \ "foo7").read[String] ::
    (__  \ "foo8").read[String] ::
    (__  \ "foo9").read[String] ::
    (__  \ "foo10").read[Int] ::
    (__  \ "foo11").read[Int] ::
    (__  \ "foo12").read[Int] ::
    (__  \ "foo13").read[Int] ::
    (__  \ "foo14").read[Int] ::
    (__  \ "foo15").read[Int] ::
    (__  \ "foo16").read[Int] ::
    (__  \ "foo17").read[Int] ::
    (__  \ "foo18").read[Int] ::
    (__  \ "foo19").read[Int] ::
    (__  \ "foo20").read[Boolean] ::
    (__  \ "foo21").read[Boolean] ::
    (__  \ "foo22").read[Boolean] ::
    (__  \ "foo23").read[Boolean] ::
    (__  \ "foo25").read[Boolean] ::
    (__  \ "foo26").read[Boolean] ::
    (__  \ "foo27").read[Boolean] ::
    (__  \ "foo28").read[Boolean] ::
    (__  \ "foo29").read[Boolean] ::
    (__  \ "foo30").read[Float] ::
    (__  \ "foo31").read[Float] ::
    (__  \ "foo32").read[Float] ::
    (__  \ "foo33").read[Float] ::
    (__  \ "foo34").read[Float] ::
    (__  \ "foo35").read[Float] ::
    (__  \ "foo36").read[Float] ::
    (__  \ "foo37").read[Float] ::
    (__  \ "foo38").read[Float] ::
    (__  \ "foo39").read[Float] ::
    (__  \ "foo40").read[List[Long]] ::
    (__  \ "foo41").read[List[Long]] ::
    (__  \ "foo42").read[List[Long]] ::
    (__  \ "foo43").read[List[Long]] ::
    (__  \ "foo44").read[List[Long]] ::
    (__  \ "foo45").read[List[Long]] ::
    (__  \ "foo46").read[List[Long]] ::
    (__  \ "foo47").read[List[Long]] ::
    (__  \ "foo48").read[List[Long]] ::
    (__  \ "foo49").read[List[Long]] ::
    (__  \ "foo50").read[JsNull.type] ::
    HNil
  )
}
{% endcodeblock %}

Let's write a play action using this rule:

{% codeblock lang:scala %}
def hfolder = Action(parse.json) { request =>
  ruleFold.validate(request.body) map { hl =>
    Ok(hl.toString)
  } recoverTotal { errors =>
    BadRequest(errors.toString)
  }
}

// OK
{
  "foo1" : "toto1",
  "foo2" : "toto2",
  "foo3" : "toto3",
  "foo4" : "toto4",
  "foo5" : "toto5",
  "foo6" : "toto6",
  "foo7" : "toto7",
  "foo8" : "toto8",
  "foo9" : "toto9",
  "foo10" : 10,
  "foo11" : 11,
  "foo12" : 12,
  "foo13" : 13,
  "foo14" : 14,
  "foo15" : 15,
  "foo16" : 16,
  "foo17" : 17,
  "foo18" : 18,
  "foo19" : 19,
  "foo20" : true,
  "foo21" : false,
  "foo22" : true,
  "foo23" : false,
  "foo24" : true,
  "foo25" : false,
  "foo26" : true,
  "foo27" : false,
  "foo28" : true,
  "foo29" : false,
  "foo30" : 3.0,
  "foo31" : 3.1,
  "foo32" : 3.2,
  "foo33" : 3.3,
  "foo34" : 3.4,
  "foo35" : 3.5,
  "foo36" : 3.6,
  "foo37" : 3.7,
  "foo38" : 3.8,
  "foo39" : 3.9,
  "foo40" : [1,2,3],
  "foo41" : [11,21,31],
  "foo42" : [12,22,32],
  "foo43" : [13,23,33],
  "foo44" : [14,24,34],
  "foo45" : [15,25,35],
  "foo46" : [16,26,36],
  "foo47" : [17,27,37],
  "foo48" : [18,28,38],
  "foo49" : [19,29,39],
  "foo50" : null
} => toto1 :: toto2 :: toto3 :: toto4 :: toto5 :: toto6 :: toto7 :: toto8 :: toto9 :: 
  10 :: 11 :: 12 :: 13 :: 14 :: 15 :: 16 :: 17 :: 18 :: 19 :: 
  true :: false :: true :: false :: false :: true :: false :: true :: false :: 
  3.0 :: 3.1 :: 3.2 :: 3.3 :: 3.4 :: 3.5 :: 3.6 :: 3.7 :: 3.8 :: 3.9 :: 
  List(1, 2, 3) :: List(11, 21, 31) :: List(12, 22, 32) :: List(13, 23, 33) :: 
  List(14, 24, 34) :: List(15, 25, 35) :: List(16, 26, 36) :: List(17, 27, 37) :: 
  List(18, 28, 38) :: List(19, 29, 39) :: null :: 
  HNil


// KO
{
  "foo1" : "toto1",
  "foo2" : "toto2",
  "foo3" : "toto3",
  "foo4" : "toto4",
  "foo5" : "toto5",
  "foo6" : "toto6",
  "foo7" : 50,
  "foo8" : "toto8",
  "foo9" : "toto9",
  "foo10" : 10,
  "foo11" : 11,
  "foo12" : 12,
  "foo13" : 13,
  "foo14" : 14,
  "foo15" : true,
  "foo16" : 16,
  "foo17" : 17,
  "foo18" : 18,
  "foo19" : 19,
  "foo20" : true,
  "foo21" : false,
  "foo22" : true,
  "foo23" : false,
  "foo24" : true,
  "foo25" : false,
  "foo26" : true,
  "foo27" : "chboing",
  "foo28" : true,
  "foo29" : false,
  "foo30" : 3.0,
  "foo31" : 3.1,
  "foo32" : 3.2,
  "foo33" : 3.3,
  "foo34" : 3.4,
  "foo35" : 3.5,
  "foo36" : 3.6,
  "foo37" : 3.7,
  "foo38" : 3.8,
  "foo39" : 3.9,
  "foo40" : [1,2,3],
  "foo41" : [11,21,31],
  "foo42" : [12,22,32],
  "foo43" : [13,23,33],
  "foo44" : [14,24,34],
  "foo45" : [15,25,35],
  "foo46" : [16,26,"blabla"],
  "foo47" : [17,27,37],
  "foo48" : [18,28,38],
  "foo49" : [19,29,39],
  "foo50" : "toto"
} => Failure(List(
  (/foo50,List(ValidationError(error.invalid,WrappedArray(null)))),
  (/foo46[2],List(ValidationError(error.number,WrappedArray(Long)))),
  (/foo27,List(ValidationError(error.invalid,WrappedArray(Boolean)))),
  (/foo15,List(ValidationError(error.number,WrappedArray(Int)))),
  (/foo7,List(ValidationError(error.invalid,WrappedArray(String))
))))

{% endcodeblock %}

Awesome... now, nobody can say 22 limits is still a problem ;)

Have a look at the code on [Github](https://github.com/mandubian/shapeless-rules)
Have fun x 50!

<br/>
<br/>
<br/>
<br/>

