---
title: Asynchronous HTTP requests with Scala and Dispatch
author: chris
date: 2014-08-11
template: article.jade
---

Today, we use REST APIs everywhere. Quite often this requires the implementation of SDKs for specific languages. If you are going to write a SDK or you need to call a REST backend without the availability of a SDK, you need a framework to send HTTP requests. The cool thing about Scala is the fact that it has native support for [Futures (aka Promises)](http://docs.scala-lang.org/overviews/core/futures.html). By using futures, you simplify your life:

 - the application does not block
 - the application can handle more parallel requests
 - you do not need a complex threading model

In Scala, [Dispatch](http://dispatch.databinder.net/Dispatch.html) is an asynchronous http library. Let's do a simple request with Futures.

## Plain HTTP request

```scala
import dispatch._, Defaults._
import scala.util.{Success, Failure}

object DispatchTest {

  def main (args: Array[String]) {
    val svc = url("http://www.wikipedia.org/");
    val response : Future[String] = Http(svc OK as.String)

    response onComplete {
      case Success(content) => {
        println("Successful response" + content)
      }
      case Failure(t) => {
        println("An error has occurred: " + t.getMessage)
      }
    }
  }
}
```

## HTTP request with redirect

It is common that HTTP endpoints use redirects. By default `Dispatch` does not follow these redirects. You need to configure the Http instance to enable the redirect handling by using `Http.configure(_ setFollowRedirects true)(svc OK as.String)`. The previous example with redirect looks like:

```scala
import dispatch._, Defaults._
import scala.util.{Success, Failure}

object DispatchTest {

  def main (args: Array[String]) {
    
    val svc = url("http://www.wikipedia.com");
    val response : Future[String] = Http.configure(_ setFollowRedirects true)(svc OK as.String)

    response onComplete {
      case Success(content) => {
        println("Successful response" + content)
      }
      case Failure(t) => {
        println("An error has occured: " + t.getMessage)
      }
    }
  }
}
```

## HTTP request with basic authentication

If you require basic authentication for your http requests, use `.as_!()`

```scala
val svc = url("http://www.wikipedia.com").as_!("user", "password")
```

## Parse JSON response

Nowadays nearly all REST endpoints use JSON responses. [Argonaut](http://argonaut.io/) is a Scala toolkit for HTTP handling. It uses many functional features of the Scala and integrates very well into the language.

For quick parsing or where a predefined structure is not available, you can parse specific fields:

```scala
import scalaz._, Scalaz._
import argonaut._, Argonaut._

val json = """
  { "name" : "Toddler", "age" : 2, "greeting": "gurgle!" }
"""

// extract a simple field
val greeting1: String =
Parse.parseWith(jsonString, _.field("greeting").flatMap(_.string).getOrElse(null), msg => msg)
```

This works very well at places where you have to deal with changing data structures. Quite often REST APIs deliver a well-defined structure, that can be used for parsing. Assume you get user data like the following:

```json
{
  "dn":"uid=chris,ou=Users,dc=lollyrock,dc=com",
  "controls":[],
  "cn":"Chris Rock",
  "givenName":"Chris",
  "l":"Berlin",
  "mail":"chris@lollyrock.com",
  "uid": "chris" ,
  "displayName":"ch.hartmann",
  "o":"Rock Inc."
}
```

At first we need a structure to store the parsed values. In Scala `Case Classes` are perfect for this need:

```scala
case class User(
   dn: String,
   cn: String,
   givenName: String,
   l: String,
   mail: String,
   displayName: String,
   o: String)
```

The following example demonstrates the parsing of JSON data into a predefined case class.

```
import scalaz._, Scalaz._
import argonaut._, Argonaut._

// data structure, json will be converted into this class
case class User(
   dn: String,
   cn: String,
   givenName: String,
   l: String,
   mail: String,
   displayName: String,
   o: String)


object UserParser {
  
  // use the implicit json conversion of Argonaut
  // more information at http://argonaut.io/doc/parsing/
  implicit def UserCodecJson: CodecJson[User] =
    // the 9 represents the amount of arguments
    casecodec9(User.apply, User.unapply)("dn", "cn", "givenName","l", "mail", "uid", "displayName", "o", "plan")

  // method to use argonaut parse
  def parse(data: String) : Option[User] = {
    Parse.decodeOption[User](data)
  }

  // simple app to test JSON parsing
  def main(args: Array[String]) {

    // json input data
    var jsonString =
      """
        | {"dn":"uid=chris,ou=Users,dc=lollyrock,dc=com","controls":[],"cn":"Chris Rock","givenName":"Chris","l":"Berlin","mail":"chris@lollyrock.com","uid": "chris" ,"displayName":"ch.hartmann","o":"Rock Inc."}
      """.stripMargin

    // parse json content
    val userdata: Option[User] = parse(jsonString)

    // print specific values
    val usr = userdata.get
    println (usr.dn)
    println (usr.displayName)

  }

}
```

The combination of `Dispatch` and `Argonout` provides an efficient way to do HTTP calls and evaluate the response in Scala. Since we are using Futures, the environment can handle more requests at the same time and the code is easier to read than a complex threading system.

Cheers

Chris 

If you have any questions contact me via [Twitter @chri_hartmann](https://twitter.com/chri_hartmann) or [Github](https://github.com/chris-rock)
