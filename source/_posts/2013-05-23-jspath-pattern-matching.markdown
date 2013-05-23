---
layout: post
title: "Play2 Json Path Pattern Matching"
date: 2013-05-01 17:17
comments: true
external-url: 
categories: [play framework,scala,json,pattern matching]
keywords: play framework,scala,json,pattern matching
---

#### EXPERIMENTAL / DRAFT
<br/>

<div class="well">
<p>While experimenting <b>Play21/Json Zipper</b> in my <a href="http://www.mandubian.com/2013/05/01/JsZipper/">previous article</a>, I needed to match patterns on <code>JsPath</code> and decided to explore a bit this topic.</p>
<p>This article just presents my experimentations on <code>JsPath</code> pattern matching so that people interested in the topic can tell me if they like it or not and what they would add or remove. So don't hesitate to let comments about it.</p>
<p>If the result is satisfying, I'll propose it to Play team ;)</p>
</div>

Let's go to samples as usual.

## Very simple pattern matching

### `match/scale`-style
{% codeblock lang:scala %}
scala> __ \ "toto" match {
  case __ \ key => Some(key)
  case _ => None
}
res0: Option[String] = Some(toto)

{% endcodeblock %}

### `val`-style

{% codeblock lang:scala %}
scala> val _ \ toto = __ \ "toto"
toto: String = toto
{% endcodeblock %}

Note that I don't write `val __ \ toto = __ \ "toto"` _(2x Underscore)_ as you would expect.

Why? Let's write it:

{% codeblock lang:scala %}
scala> val __ \ toto = __ \ "toto"
<console>:20: error: recursive value x$1 needs type
val __ \ toto = __ \ "toto"
{% endcodeblock %}

Actually, 1st `__` is considered as a variable to be affected by Scala compiler. Then the variable `__` appears on left and right side which is not good.

So I use `_` to ignore its value because I know it's `__`. If I absolutely wanted to match with `__`, you would have written:

{% codeblock lang:scala %}
scala> val JsPath \ toto = __ \ "toto"
toto: String = toto
{% endcodeblock %}

<br/>
## Pattern matching with indexed path

{% codeblock lang:scala %}
scala> val (_ \ toto)@@idx = (__ \ "toto")(2)
toto: String = toto
idx: Int = 2

scala> (__ \ "toto")(2) match {
  case (__ \ "toto")@@idx => Some(idx)
  case _      => None
}
res1: Option[Int] = Some(2)
{% endcodeblock %}

Note the usage of `@@` operator that you can dislike. _I didn't find anything better for now but if anyone has a better idea, please give it to me ;)_

<br/>
## Pattern matching the last element of a JsPath

{% codeblock lang:scala %}
scala> val _ \ last = __ \ "alpha" \ "beta" \ "delta" \ "gamma"
last: String = gamma
{% endcodeblock %}

Using `_`, I ignore everything before `gamma` node.

<br/>
## Matching only the first element and the last one

{% codeblock lang:scala %}
scala> val _ \ first \?\ last = __ \ "alpha" \ "beta" \ "gamma" \ "delta"
first: String = alpha
last: String = delta

scala> val (_ \ first)@@idx \?\ last = (__ \ "alpha")(2) \ "beta" \ "gamma" \ "delta"
first: String = alpha
idx: Int = 2
last: String = delta
{% endcodeblock %}

Note the `\?\` operator which is also a temporary choice: I didn't want to choose `\\` ause `\?\` operator only works in the case where you match between the first and the last element of the path and not between anything and anything...

<br/>
## A few more complex cases

{% codeblock lang:scala %}
scala> val (_ \ alpha)@@idx \ beta \ gamma \ delta = (__ \ "alpha")(2) \ "beta" \ "gamma" \ "delta"
alpha: String = alpha
idx: Int = 2
beta: String = beta
gamma: String = gamma
delta: String = delta

scala> val (_ \ alpha)@@idx \ _ \ _ \ delta = (__ \ "alpha")(2) \ "beta" \ "gamma" \ "delta"
alpha: String = alpha
idx: Int = 2
delta: String = delta

scala> val _@@idx \?\ gamma \ delta = (__ \ "alpha")(2) \ "beta" \ "gamma" \ "delta"
idx: Int = 2
gamma: String = gamma
delta: String = delta

scala> (__ \ "alpha")(2) \ "beta" \ "gamma" \ "delta" match {
  case _@@2 \ "beta" \ "gamma" \ _ => true
  case _ => false
}
res4: Boolean = true
{% endcodeblock %}

<br/>
## And finally using regex?

{% codeblock lang:scala %}
scala> val pattern = """al(\d)*pha""".r
pattern: scala.util.matching.Regex = al(\d)*pha

scala> (__ \ "foo")(2) \ "al1234pha" \ "bar" match {
  case (__ \ "foo")@@idx \ pattern(_) \ "bar" => true
  case _ => false
}
res6: Boolean = true
{% endcodeblock %}

So, I think we can provide more features and now I'm going to use it with my `JsZipper` stuff in my next article ;)

If you like it, tell it!

Have fun!

<br/>
<br/>
<br/>
<br/>