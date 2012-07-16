---
layout: post
title: "xmlsoap-ersatz"
date: 2012-07-14 00:59
comments: true
categories: [ scala, soap, xml, playframework ]
published: true
---
# XML/SOAP to/from scala structures with *xmlsoap-ersatz*

> [xmlsoap-ersatz](https://github.com/mandubian/scala-xmlsoap-ersatz) is a toolset to help people serialize/deserialize XML/SOAP with Scala. It doesn't try to respect any standard but it allows to interprete and generate any XML format with any standard you require.

It was developed to be used with [Play2/Scala](http://www.playframework.org) but it can work standalone as it doesn't depend on any other library.

xmlsoap-ersatz uses the same design as Play2 Json serialization/deserialization based on **implicit typeclasses**. This mechanism is very clean, robust and generic. Please note that implicits are NOT used as implicit CONVERSIONS which are often tricky and sometimes dangerous!

*xmlsoap-ersatz is still draft library so if you discover anything wrong, don't hesitate to tell it*

> xmlsoap-ersatz is an [opensource public github](https://github.com/mandubian/scala-xmlsoap-ersatz) so don't hesitate to contribute and give ideas :)

> Full doc is in [Github Wiki](https://github.com/mandubian/scala-xmlsoap-ersatz/wiki)

<br/>

# XML Scala serialization/deserialization

### Imagine you want to map this XML to a case class

    val fooXml = <foo>
        <id>1234</id>
        <name>brutus</name>
        <age>23</age>
    </foo>

<br/>

### You want to map it to:

    case class Foo(id: Long, name: String, age: Option[Int])
    Foo(id=1234L, name="brutus", age=Some(23))


* case class is a sample but the mechanism also works with any structure in Scala such as tuples
* `age` field is `Option[Int]` meaning it might not appear in the XML (`None` in this case)


<br/>

### So how would you write that with _xmlsoap ersatz_?

    import play2.tools.xml._
    import play2.tools.xml.DefaultImplicits._
    [...]
    val foo = EXML.fromXML[Foo](fooXml)
    assert(foo == Foo(1234L, "brutus", Some(23)))

    val fooXml2 = EXML.toXML(foo)
    assert(fooXml2 == fooXml)

As you may imagine, this is not so simple as `fromXML`/`toXML` signatures are the following:

    object EXML {
        def toXML[T](t: T, base: xml.NodeSeq = xml.NodeSeq.Empty)(implicit w: XMLWriter[T]): xml.NodeSeq
        def fromXML[T](x: xml.NodeSeq)(implicit r: XMLReader[T]): Option[T] 
    }

You can see the **implicit typeclasses `XMLReader`/`XMLWriter`** which define the mapper to/from XML to your case class.
So in order `EXML.fromXML`/`EXML.toXML` to work properly, you should define an implicit XML reader/writer for your specific structure in your scope. 
It can be done at once by extending `XMLFormatter[T]`:

    trait XMLFormatter[T] extends XMLReader[T] with XMLWriter[T] {
        def read(x: xml.NodeSeq): Option[T]
        def write(f: T, base: xml.NodeSeq): xml.NodeSeq
    }

<br/>

### Defining the implicit  for your case class

    implicit object FooXMLF extends XMLFormatter[Foo] {
      def read(x: xml.NodeSeq): Option[Foo] = {
        for( 
          id <- EXML.fromXML[Long](x \ "id");
          name <- EXML.fromXML[String](x \ "name");
          age <- EXML.fromXML[Option[Int]](x \ "age")
        ) yield(Foo(id, name, age))
      }

      def write(f: Foo, base: xml.NodeSeq): xml.NodeSeq = {
        <foo>
          <id>{ f.id }</id>
          <name>{ f.name }</name>
          { EXML.toXML(f.age, <age/>) }
        </foo>
      }
    }


You may think this is a bit tedious to write but this is quite easy after a few tries and the most important:
> This mechanism provides a very precise and simple control on what you want to do.

Please note:

- the `write` function uses Scala XML literals simply.
- the `implicit` is important: you can declare it once in your scope and that's all.
- the `age` field in `write` requires a special syntax:

    `{ EXML.toXML(f.age, <age/>) }` means: `<age/>` is used as the base node and will generate following XML:
    - if age is `Some(23)`: `<age>23</age>`
    - if age is `None`: `<age xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:nil="true" />` (_standards defines this but you can redefine it as you want_)

- the `base` field in `write` function can be used to pass a parent context node to the writer. It is helpful in case you need to add attributes to parent node for ex as in the case of `age: Option[Int]` field. 
>But don't forget that Scala XML nodes are immutable and that you can just copy the nodes you pass to a function. 

- the _for-comprehension_ is just a shortcut but you could write it using flatMap/map also:

For ex:

    def read(x: xml.NodeSeq): Option[Foo] = {
      EXML.fromXML[Long](x \ "id").flatMap{ id =>
        EXML.fromXML[String](x \ "name").flatMap { name =>
          EXML.fromXML[Int](x \ "age").map{ age =>
            Foo(id, name, age)
          }
        }
      }
    }

<br/>

### The complete code

    import play2.tools.xml._
    import play2.tools.xml.DefaultImplicits._

    implicit object FooXMLF extends XMLFormatter[Foo] {
      def read(x: xml.NodeSeq): Option[Foo] = {
        for( 
          id <- EXML.fromXML[Long](x \ "id");
          name <- EXML.fromXML[String](x \ "name");
          age <- EXML.fromXML[Option[Int]](x \ "age");
        ) yield(Foo(id, name, age))
      }

      def write(f: Foo, base: xml.NodeSeq): xml.NodeSeq = {
        <foo>
          <id>{ f.id }</id>
          <name>{ f.name }</name>
          { EXML.toXML(f.age, <age/>) }
        </foo>
      }
    }

    val foo = EXML.fromXML[Foo](fooXml)
    assert(foo == Foo(1234L, "albert", 23)

    val fooXml = EXML.toXML(foo)
    assert(fooXml == fooXml)

# Integrate with Play2/Scala

## Add xmlsoap-ersatz to your configuration (ersatz is deployed as a maven repo on github)

    object ApplicationBuild extends Build {

      val appName         = "play2-xmlsoap"
      val appVersion      = "1.0-SNAPSHOT"

      val appDependencies = Seq(
        "play2.tools.xml" %% "xmlsoap-ersatz" % "0.1-SNAPSHOT"
      )  

      val main = PlayProject(appName, appVersion, appDependencies, mainLang = SCALA).settings(
        resolvers += ("mandubian-mvn snapshots" at "https://github.com/mandubian/mandubian-mvn/raw/master/snapshots")
      )
    }

## Use xmlsoap-ersatz in your controller

    package controllers

    import play.api._
    import play.api.mvc._
    import play2.tools.xml._
    import play2.tools.xml.DefaultImplicits._

    object Application extends Controller {
       case class Foo(id: Long, name: String, age: Option[Int])

       implicit object FooXMLF extends XMLFormatter[Foo] {
          def read(x: xml.NodeSeq): Option[Foo] = {
          for( 
              id <- EXML.fromXML[Long](x \ "id");
              name <- EXML.fromXML[String](x \ "name");
              age <- EXML.fromXML[Option[Int]](x \ "age")
            ) yield(Foo(id, name, age))
          }

          def write(f: Foo, base: xml.NodeSeq): xml.NodeSeq = {
            <foo>
              <id>{ f.id }</id>
              <name>{ f.name }</name>
              { EXML.toXML(f.age, <age/>) }
            </foo>
         }
      }  

      def foo = Action(parse.xml) { request =>
        EXML.fromXML[Foo](request.body).map { foo =>
          Ok(EXML.toXML(foo))
        }.getOrElse{
          BadRequest("Expecting Foo XML data")
        }
      }
  
    }

## Finally the route in `conf/routes`

    POST    /foo    controllers.Application.foo

Have fun and don't hesitate to contribute
