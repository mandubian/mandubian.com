---
layout: post
title: "Play2 Json Interpolation & Pattern Matching"
date: 2013-07-04 08:08
comments: true
external-url:
categories: [play framework,scala,json,string interpolation,pattern matching]
keywords: play framework,scala,json,string interpolation,pattern matching
---

#### EXPERIMENTAL / DRAFT
<br/>

<div class="well">
<p>Do you remember <code>JsPath</code> pattern matching presented in <a href="http://mandubian.com/2013/05/01/jspath-pattern-matching/">this article</a> ?</p>
<p>Let's now go further with something that you should enjoy even more: <b>Json Interpolation &amp; Pattern Matching</b>.</p>

<p>I've had the idea of these features for some time in my mind but let's render unto Caesar what is Caesar's : <a href="http://rapture.io/jsonSupport">Rapture.io</a> proved that it could be done quite easily and I must say I <del>stole</del> got inspired by a few implementation details from them! (<i>specially the @inline implicit conversion for string interpolation class which is required due to a ValueClass limitation that should be removed in further Scala versions</i>)
</div>


First of all, code samples as usual...

## Create JsValue using String interpolation

{% codeblock lang:scala %}
scala> val js = json"""{ "foo" : "bar", "foo2" : 123 }"""
js: play.api.libs.json.JsValue = {"foo":"bar","foo2":123}

scala> js == Json.obj("foo" -> "bar", "foo2" -> 123)
res1: Boolean = true

scala> val js = json"""[ 1, true, "foo", 345.234]"""
js: play.api.libs.json.JsValue = [1,true,"foo",345.234]

scala> js == Json.arr(1, true, "foo", 345.234)
res2: Boolean = true
{% endcodeblock %}

Yes, pure Json in a string...

How does it work? Using [String interpolation](http://docs.scala-lang.org/overviews/core/string-interpolation.html) introduced in Scala 2.10.0 and Jackson for the parsing...

In String interpolation, you can also put Scala variables directly in the interpolated string. You can do the same in Json interpolation.

{% codeblock lang:scala %}
scala> val alpha = "foo"
alpha: String = foo

scala> val beta = 123L
beta: Long = 123

scala> val js = json"""{ "alpha" : "$alpha", "beta" : $beta}"""
js: play.api.libs.json.JsValue = {"alpha":"foo","beta":123}

scala> val gamma = Json.arr(1, 2, 3)
gamma: play.api.libs.json.JsArray = [1,2,3]

scala> val delta = Json.obj("key1" -> "value1", "key2" -> "value2")
delta: play.api.libs.json.JsObject = {"key1":"value1","key2":"value2"}

scala> val js = json"""
     |         {
     |           "alpha" : "$alpha",
     |           "beta" : $beta,
     |           "gamma" : $gamma,
     |           "delta" : $delta,
     |           "eta" : {
     |             "foo" : "bar",
     |             "foo2" : [ "bar21", 123, true, null ]
     |           }
     |         }
     |       """
js: play.api.libs.json.JsValue = {"alpha":"foo","beta":123,"gamma":[1,2,3],"delta":{"key1":"value1","key2":"value2"},"eta":{"foo":"bar","foo2":["bar21",123,true,null]}}

{% endcodeblock %}

_Please note that string variables must be put between `"..."` because without it the parser will complain._

Ok, so now it's really trivial to write Json, isn't it?

String interpolation just replaces the string you write in your code by some Scala code concatenating pieces of strings with variables as you would write yourself. Kind-of: `s"toto ${v1} tata" => "toto + v1 + " tata" + ...`

But at compile-time, it doesn't compile your String into Json: the Json parsing is done at runtime with string interpolation. So using Json interpolation doesn't provide you with compile-time type safety and parsing for now.

>In the future, I may replace String interpolation by a real Macro which will also parse the string at compile-time. Meanwhile, if you want to rely on type-safety, go on using `Json.obj / Json.arr` API.

<br/>
## Json pattern matching

What is one of the first feature that you discover when learning Scala and that makes you say immediately: "Whoaa Cool feature"? **Pattern Matching**.

You can write:
{% codeblock lang:scala %}
scala> val opt = Option("toto")
opt: Option[String] = Some(toto)

scala> opt match {
  case Some(s) => s"not empty option:$s"
  case None    => "empty option"
}
res2: String = not empty option:toto

// or direct variable assignement using pattern matching

scala> val Some(s) = opt
s: String = toto
{% endcodeblock %}

Why not doing this with Json?

And.... Here it is with Json pattern matching!!!

{% codeblock lang:scala %}
scala> val js = Json.obj("foo" -> "bar", "foo2" -> 123L)
js: play.api.libs.json.JsObject = {"foo":"bar","foo2":123}

scala> js match {
  case json"""{ "foo" : $a, "foo2" : $b }""" => Some(a -> b)
  case _ => None
}
res5: Option[(play.api.libs.json.JsValue, play.api.libs.json.JsValue)] = 
Some(("bar",123))

scala> val json"""{ "foo" : $a, "foo2" : $b}""" = json""" { "foo" : "bar", "foo2" : 123 }"""
a: play.api.libs.json.JsValue = "bar"
b: play.api.libs.json.JsValue = 123

scala> val json"[ $v1, 2, $v2, 4 ]" = Json.arr(1, 2, 3, 4)
v1: play.api.libs.json.JsValue = 1
v2: play.api.libs.json.JsValue = 3

{% endcodeblock %}

Magical? 

Not at all... Just `unapplySeq` using the tool that enables this kind of Json manipulation as trees: `JsZipper`... 

>The more I use JsZippers, the more I find fields where I can use them ;)

<br/>
## More complex Json pattern matching

{% codeblock lang:scala %}
scala> val js = json"""{
    "key1" : "value1",
    "key2" : [
      "alpha",
      { "foo" : "bar",
        "foo2" : {
          "key21" : "value21",
          "key22" : [ "value221", 123, false ]
        }
      },
      true,
      123.45
    ]
  }"""
js: play.api.libs.json.JsValue = {"key1":"value1","key2":["alpha",{"foo":"bar","foo2":{"key21":"value21","key22":["value221",123,false]}},true,123.45]}

scala> val json"""{ "key1" : $v1, "key2" : ["alpha", $v2, true, $v3] }""" = js
v1: play.api.libs.json.JsValue = "value1"
v2: play.api.libs.json.JsValue = {"foo":"bar","foo2":{"key21":"value21","key22":["value221",123,false]}}
v3: play.api.libs.json.JsValue = 123.45

scala> js match {
    case json"""{
      "key1" : "value1",
      "key2" : ["alpha", $v1, true, $v2]
    }"""   => Some(v1, v2)
    case _ => None
  }
res9: Option[(play.api.libs.json.JsValue, play.api.libs.json.JsValue)] = 
Some(({"foo":"bar","foo2":{"key21":"value21","key22":["value221",123,false]}},123.45))

// A non matching example maybe ? ;)
scala>  js match {
    case json"""{
      "key1" : "value1",
      "key2" : ["alpha", $v1, false, $v2]
    }"""   => Some(v1, v2)
    case _ => None
  }
res10: Option[(play.api.libs.json.JsValue, play.api.libs.json.JsValue)] = None
{% endcodeblock %}

If you like that, please tell it so that I know whether it's worth pushing it to Play Framework!

<br/>
## Using these features right now in a Scala/SBT project

These features are part of my experimental project JsZipper presented in [this article](http://mandubian.com/2013/05/01/JsZipper/).

To use it, add following lines to your SBT `Build.scala`:

{% codeblock lang:scala %}
object ApplicationBuild extends Build {
  ...
  val mandubianRepo = Seq(
    "Mandubian repository snapshots" at "https://github.com/mandubian/mandubian-mvn/raw/master/snapshots/",
    "Mandubian repository releases" at "https://github.com/mandubian/mandubian-mvn/raw/master/releases/"
  )
  ...

  val main = play.Project(appName, appVersion, appDependencies).settings(
    resolvers ++= mandubianRepo,
    libraryDependencies ++= Seq(
      ...
      "play-json-zipper"  %% "play-json-zipper"    % "0.1-SNAPSHOT",
      ...
    )
  )
  ...
}
{% endcodeblock %}

In your Scala code, import following packages

{% codeblock lang:scala %}
import play.api.libs.json._
import syntax._
import play.api.libs.functional.syntax._
import play.api.libs.json.extensions._
{% endcodeblock %}

PatternMatch your fun!

<br/>
<br/>
<br/>
<br/>