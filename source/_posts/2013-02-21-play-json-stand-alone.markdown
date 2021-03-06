---
layout: post
title: "Playing with Play2 SCALA JSON API Stand-alone"
date: 2013-02-21 14:14
comments: true
external-url: 
categories: [play framework,json,play 2.1,scala]
keywords: play framework,json,play 2.1,scala
---

> Now _play-json stand-alone_ is officially a stand-alone module in Play2.2. So you don't have to use the dependency given in this article anymore but use Typesafe one like _com.typesafe.play:play-json:2.2_

<br/>
In a very recent [Pull Request](https://github.com/playframework/Play20/pull/754), `play-json has been made a stand-alone module in [Play2.2-SNAPSHOT master](https://github.com/playframework/Play20) as play-iteratees.

It means: 

- You can take Play2 Scala Json API as a stand-alone library and keep using Json philosophy promoted by [Play Framework](http://www.playframework.org) anywhere.
- `play-json` module is stand-alone in terms of dependencies but is a part & parcel of Play2.2 so it will evolve and follow Play2.x releases (and following versions) always ensuring full compatibility with play ecosystem.
- `play-json` module has 3 ultra lightweight dependencies:
     - `play-functional`  
     - `play-datacommons`
     - `play-iteratees`

These are pure Scala generic pieces of code from Play framework so no Netty or whatever dependencies in it.  
You can then import `play-json` in your project without any fear of bringing unwanted deps.

`play-json` will be released with future Play2.2 certainly so meanwhile, I provide:

- a build published in my [Maven Github repository](https://github.com/mandubian/mandubian-mvn/)
- a sample project called [play-json-alone](https://github.com/mandubian/play-json-alone)

<br/>
>Even if the version is _2.2-SNAPSHOT_, be aware that this is the same code as the one released in Play 2.1.0. This API has reached a good stability level. Enhancements and bug corrections will be brought to it but it's production-ready right now.



## Adding play-json 2.2-SNAPSHOT in your dependencies

In your `Build.scala`, add:

{% codeblock lang:scala %}
import sbt._
import Keys._

object ApplicationBuild extends Build {

  val mandubianRepo = Seq(
    "Mandubian repository snapshots" at "https://github.com/mandubian/mandubian-mvn/raw/master/snapshots/",
    "Mandubian repository releases" at "https://github.com/mandubian/mandubian-mvn/raw/master/releases/"
  )

  lazy val playJsonAlone = Project(
    BuildSettings.buildName, file("."),
    settings = BuildSettings.buildSettings ++ Seq(
      resolvers ++= mandubianRepo,
      libraryDependencies ++= Seq(
        "play"        %% "play-json" % "2.2-SNAPSHOT",
        "org.specs2"  %% "specs2" % "1.13" % "test",
        "junit"        % "junit" % "4.8" % "test"
      )
    )
  )
}
{% endcodeblock %}

## Using play-json 2.2-SNAPSHOT in your code:

Just import the following and get everything from Play2.1 Json API:

{% codeblock lang:scala %}
import play.api.libs.json._
import play.api.libs.functional._

case class EucalyptusTree(col:Int, row: Int)

object EucalyptusTree{
  implicit val fmt = Json.format[EucalyptusTree]
}

case class Koala(name: String, home: EucalyptusTree)

object Koala{
  implicit val fmt = Json.format[Koala]
}
  
val kaylee = Koala("kaylee", EucalyptusTree(10, 23))

println(Json.prettyPrint(Json.toJson(kaylee)))

Json.fromJson[Koala](
  Json.obj(
    "name" -> "kaylee", 
    "home" -> Json.obj(
      "col" -> 10, 
      "row" -> 23
    )
)
{% endcodeblock %}


> Using `play-json`, you can get some bits of [Play Framework](http://www.playframework.org) pure Web philosophy.  
> Naturally, to unleash its full power, don't hesitate to dive into [Play Framework](http://www.playframework.org) and discover 100% full Web Reactive Stack ;)

Thanks a lot to Play Framework team for promoting play-json as stand-alone module!  
Lots of interesting features incoming soon ;)

Have fun!

