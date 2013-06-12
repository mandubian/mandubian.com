---
layout: post
title: "Reactive Json Crafting : JsZipper + ReactiveMongo + multiple Async WS calls"
date: 2013-05-01 17:17
comments: true
external-url: 
categories: [json,scala,play framework,reactivemongo,jszipper,async,reactive,webservice]
keywords: json,scala,play framework,reactivemongo,jszipper,async,reactive,webservice
---

#### EXPERIMENTAL / DRAFT
The sample app can be found on Github [here](https://github.com/mandubian/play-json-zipper/tree/master/samples/multiservices)

<br/>

Hi again folks!

Now, you may certainly have realized I'm [Play2.1](http://www.playframework.org) Json API advocate. But you may also have understood that I'm not interested in Json as an end in itself. What catches my attention is that it's a versatile arborescent data structure that can be used in web server&amp;client, in DB such as [ReactiveMongo](http://reactivemongo.org) and also when communicating between servers with WebServices.

So I keep exploring what can be done with Json (specially in the context of PlayFramework reactive architecture) and building the tools that are required to concretize my ideas.

<div class="well">
<p>My last article introduced <a href="http://www.mandubian.com/2013/05/01/jspath-pattern-matching/">JsPath Pattern Matching</a> and I told you that I needed this tool to use it with JsZipper. It's time to use it...</p>
<p>Here is why I want to do:</p>
<ul>
  <li><b>Build dynamically a Json structure by aggregating data obtained by calling several external WS</b> such as twitterAPI or github API or whatever API.</li>

  <li><b>Build this structure from a Json template stored in MongoDB</b> in which I will find the URL and params of WebServices to call.</li>

  <li><b>Use Play2.1/WS & ReactiveMongo reactive API</b> meaning resulting Json should be built in an asynchronous and non-blocking way.</li>

  <li><b>Use concept of JsZipper</b> introduced in my <a href="http://www.mandubian.com/2013/05/01/JsZipper/">previous article</a> to be able to modify efficiently Play2.1/Json immutable structures.</li>
</ul>
</div>

> Please note that this idea and its implementation is just an exercise of style to study the idea and introduce technical concepts but naturally it might seem a bit fake. Moreover, keep in mind, JsZipper API is still draft...

<br/>
## The idea of Json template

Imagine I want to gather twitter user timeline and github user profile in a single Json object.

I also would like to:

- configure the URL of WS and query parameters to fetch data 
- customize the resulting Json structure

Let's use a _Json template_ such as:

{% codeblock lang:json %}
{
  "streams" : {
    "twitter" : {
      "url" : "http://localhost:9000/twitter/statuses/user_timeline",
      "user_id" : "twitter_nick"
    },
    "github" : {
      "url" : "http://localhost:9000/github/users",
      "user_id" : "github_nick"
    }
  }
}
{% endcodeblock %}

Using the `url` and `user_id` found in `__\streams\twitter, I can call twitter API to fetch the stream of tweets and the same for `__\streams\github`. Finally I replace the content of each node as following:

{% codeblock lang:scala %}
{
  "streams" : {
    "twitter" : {
      // TWITTER USER TIMELINE HERE
    },
    "github" : {
      // GITHUB USER PROFILE HERE
    }
  }
}
{% endcodeblock %}

Moreover, I'd like to store multiple templates like previous sample with multiple `user_id` to be able to retrieve multiple streams at the same time.

<br/>
## Creating Json template in Play/ReactiveMongo (v0.9)

Recently, Stephane Godbillon has released [ReactiveMongo v0.9](http://reactivemongo.org) with corresponding Play plugin. This version really improves and eases the way you can manipulate Json directly with Play & Mongo from Scala.

Let's store a few instance of previous templates using this API:

{% codeblock lang:scala %}
// gets my mongo collection
def coll = db.collection[JSONCollection]("templates")

def provision = Action { Async {
  val values = Enumerator(
    Json.obj(
      "streams" -> Json.obj(
        "twitter" -> Json.obj(
          "url" -> "http://localhost:9000/twitter/statuses/user_timeline",
          "user_id" -> "twitter_nick1"
        ),
        "github" -> Json.obj(
          "url" -> "http://localhost:9000/github/users",
          "user_id" -> "github_nick1"
        )
      )
    ),
    ... more templates
  )

  coll.bulkInsert(values).map{ nb =>
    Ok(Json.obj("nb"->nb))
  }

} }
{% endcodeblock %}

Hard isn't it?

Note that I use `localhost` URL because with real Twitter/Github API I would need OAuth2 tokens and this would be a pain for this sample :)

<br/>
<br/>
## Reactive Json crafting

Now, let's do the real job i.e the following steps:

- retrieve the template(s) from Mongo using ReactiveMongo `JsonCollection`
- call the WebServices to fetch the data using Play Async WS
- update the Json template(s) using Monadic JsZipper `JsZipperM[Future]`

The interesting technical points here are that:

- ReactiveMongo is async so we get `Future[JsValue]`
- Play/WS is Async so we get also `Future[JsValue]`
- We need to call multiple WS so we have a `Seq[Future[JsValue]]`

We could use Play/Json transformers presented in a [previous article](http://www.mandubian.com/2012/10/29/unveiling-play-2-dot-1-json-api-part3-json-transformers/) but knowing that you have to manage Futures and multiple WS calls, it would create quite complicated code.

Here is where Monadic JsZipper becomes interesting: 

- `JsZipper` allows modifying immutable JsValue which is already cool

- `JsZipperM[Future]` allows modifying `JsValue` in the future and it's even better!

Actually the real power of JsZipper (_besides being able to modify/delete/create a node in immutable Json tree_) is to transform a Json tree into a Stream of nodes that it can traverse in depth, in width or whatever you need.

<br/>
### Less code with WS sequential calls
 
Here is the code because you'll see how easy it is:

{% codeblock lang:scala %}
// a helper to call WS
def callWSFromTemplate(value: JsValue): Future[JsValue] = 
  WS.url((value \ "url").as[String])
    .withQueryString( "user_id" -> (value \ "user_id").as[String] )
    .get().map{ resp => resp.json }

// calling WS sequentially
def dataSeq = Action{
  Async{
    for{
      templates <- coll.find(Json.obj()).cursor[JsObject].toList   // retrieves templates from Mongo
      updated   <- Json.toJson(templates).updateAllM{
        case (_ \ "twitter", value) => callWSFromTemplate(value)
        case (_ \ "github", value)  => callWSFromTemplate(value)
        case (_, value)             => Future.successful(value)
      }
    } yield Ok(updated)
  }
}
{% endcodeblock %}

Please note:

- `Json.toJson(templates)` transforms a `List[JsObject]` into `JsArray` because we want to manipulate pure `JsValue` with `JsZipperM[Future]`.

- `.updateAllM( (JsPath, JsValue) => Future[JsValue] )` is a wrapper API hiding the construction of a `JsZipperM[Future]`: once built, the `JsZipperM[Future] traverses the Json tree and for each node, it calls the provided function _flatMapping_ on Futures before going to next node. This makes the calls to WS sequential and not parallel.

- `case (_ \ "twitter", value)` : yes here is the JsPath pattern matching and imagine the crazy stuff you can do mixing Json traversal and pattern matching ;)

- `Async` means the embedded code will return `Future[Result]` but remember that it DOESN'T mean the `Action` is synchronous/blocking because in Play, everything is Asynchronous/non-blocking by default.

Then you could tell me that this is cool but the WS are not called in parallel but sequentially. Yes it's true but imagine that it's less than 10 lines of code and could even be reduced. Yet, here is the parallelized version...

<br/>
### Parallel WS calls

{% codeblock lang:scala %}
def dataPar = Action{
  Async{
    coll.find(Json.obj()).cursor[JsObject].toList.flatMap{ templates =>
      // converts List[JsObject] into JsArray
      val jsonTemplates = Json.toJson(templates)

      // gathers all nodes that need to be updated
      val nodes = jsonTemplates.findAll{
        case (_ \ "twitter", _) | (_ \ "github", _) => true
        case (_, value) => false
      }

      // launches WS calls in parallel and updates original JsArray
      Future.traverse(nodes){
        case (path@(_ \ "twitter"), value) => callWSFromTemplate(value).map( resp => path -> resp )
        case (path@(_ \ "github"), value)  => callWSFromTemplate(value).map( resp => path -> resp )
      }.map{ pathvalues => Ok(jsonTemplates.set(pathvalues:_*)) }
    }
  }
}
{% endcodeblock %}
 
Note that:

- `jsonTemplates.findAll( filter: (JsPath, JsValue) => Boolean )` traverses the Json tree and returns a `Stream[(JsPath, JsValue)]` containing the filtered nodes. This is not done with `Future` because we want to get all nodes now to be able to launch all WS calls in parallel.

- `Future.traverse(nodes)(T => Future[T])` traverses the filtered values and calls all WS in parallel.

- `case (path@(_ \ "twitter"), value)` is just JsPath pattern matching once again keeping track of full path to be able to return it with the value `path -> resp` for next point.

- `jsonTemplates.set( (JsPath, JsValue)* )` finally updates all values at given path. Note how easy it is to update multiple values at multiple paths.

A bit less elegant than the sequential case but not so much.

<br/>
<br/>
## Conclusion

This sample is a bit stupid but you can see the potential of mixing those different tools together. 

Alone, JsZipper and JsPath pattern matching provides very powerful ways of manipulating Json that Reads/Writes can't do easily. 

When you add reactive API on top of that, JsZipper becomes really interesting and elegant.

The sample app can be found on Github [here](https://github.com/mandubian/play-json-zipper/tree/master/samples/multiservices)

Have JsZipperM[fun]!

<br/>
<br/>
<br/>
<br/>