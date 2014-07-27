---
title: Scalatra with Bearer Authentication
author: chris
date: 2014-07-26
template: article.jade
---

# Why use Scala over Java?

After I experienced the beauty of Ruby and Node.js for web application development I thought there are not many use cases for developing a Java web application, because they provide:

- easy definition of new routes
- stick to REST API with JSON
- quick development
- easy deployment

Everything can be solved in Java, but it does not necessarily feels right. Java Enterprise Edition 5 and Spring try to make it as easy as possible, but Java has some disadvantages at language level, not on framework level:

- combination of imperative and functional approaches are quite helpful for writing more readable code and getting things done in a shorter time frame
- flexible type system, that is type safe and not verbose (Node.js and Ruby are not type safe)
- [Promises for Javascript](http://www.html5rocks.com/en/tutorials/es6/promises/) replace the concept of callbacks, ease the asynchronous development and establish a clean way to handle errors

Recently I worked on a project where I had the requirement to re-use an existing Java infrastructure. The nature of the existing infrastructure required the use of a JVM-based solution. It was the right time to try Scala and in particular [Scalatra](http://www.scalatra.org) because it is similar to [Sinatra](http://www.sinatrarb.com/) and [Express](http://expressjs.com/). To get started with Scalatra try out these [examples](https://github.com/scalatra/scalatra-website-examples).

I use it with [IntelliJ](http://www.jetbrains.com/idea/), [IntelliJ Scala plugin](http://blog.jetbrains.com/scala/2013/11/18/built-in-sbt-support-in-intellij-idea-13/) and [sbt-idea](https://github.com/mpeltonen/sbt-idea). If you are new to Scala you should also read the [sbt guide](http://www.scala-sbt.org/0.13/tutorial/index.html)

# OAuth 2.0 Bearer Authentication for Scalatra

Once you have a sample Scalatra application up and running, you may want to build your REST API like:

```scala
get("/hello/:name") {
  // Matches "GET /hello/foo" and "GET /hello/bar"
  // params("name") is "foo" or "bar"
  <p>Hello, {params("name")}</p>
}
```

For the command-line requests I am going to use [httpie](httpie http://httpie.org)

```code
http -v http://localhost:8080/hello/john

GET /hello/john HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate, compress
Host: localhost:8080
User-Agent: HTTPie/0.8.0



HTTP/1.1 200 OK
Content-Length: 18
Content-Type: text/html;charset=UTF-8
Server: Jetty(8.1.8.v20121106)

<p>Hello, john</p>
```

Assume you need to restrict the access to routes. In a corporate environment an OAuth server is used quiet often to verify requests. [Bearer tokens](http://tools.ietf.org/html/rfc6750) are very useful for that task. The HTTP header for authentication is well established and used for basic authentication as well. Now the resource provider (our Scala server with routes) need to handle authentication header and verify the received token. 

Salesforce published [Digging Deeper into OAuth 2.0 on Force.com](https://developer.salesforce.com/page/Digging_Deeper_into_OAuth_2.0_on_Force.com) that illustrates the setup:

<img alt="OAuthRoles.png" src="https://s3.amazonaws.com/dfc-wiki/en/images/6/6f/OAuthRoles.png" width="443" height="414">

A client request to the resource provider looks like

```code
http -v http://localhost:8080/hello/john 'Authorization:Bearer Ieg4ahthie'

GET /hello/john HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate, compress
Authorization: Bearer Ieg4ahthie
Host: localhost:8080
User-Agent: HTTPie/0.8.0



HTTP/1.1 200 OK
Content-Length: 18
Content-Type: text/html;charset=UTF-8
Server: Jetty(8.1.8.v20121106)

<p>Hello, john</p>

```

Now, the resource server needs to respect the authentication and only provide access to authenticated users. We need a way to implement a generic authentication strategy to authenticate users via Bearer token. Scalatra provides a way to [hook your own authentication strategy](http://www.scalatra.org/2.3/guides/http/authentication.html) into the framework. The implementation of the `BasicAuthStrategy` is available at [Github](https://github.com/scalatra/scalatra/blob/2.3.x/auth/src/main/scala/org/scalatra/auth/strategy/BasicAuthStrategy.scala)

Based on the BasicAuthStrategy, we are going to implement a `BearerAuthStragegy`


<style type="text/css">
.gist {
  font-size: 12px;
}
</style>
<script src="https://gist.github.com/chris-rock/9cc43202ecbd57ad1f4b.js"></script>

Now you need to wire it into your Servlet:

```scala 
trait HelloAppStack
  extends ScalatraServlet
  with ScalateSupport
  with CorsSupport
  with AuthenticationSupport {
  ...
  }
```

Now you are able to secure routes with:

```scala
get("/hello/:name") {

    val user: User = auth.get

    // Matches "GET /hello/foo" and "GET /hello/bar"
    // params("name") is "foo" or "bar"
    <p>Hello, {params("name")}</p>
}

```

Finally unauthenticated responses are rejected:

```code 
http -v http://localhost:8080/hello/john

GET /hello/john HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate, compress
Host: localhost:8080
User-Agent: HTTPie/0.8.0



HTTP/1.1 401 Unauthorized
Content-Length: 15
Content-Type: text/plain;charset=UTF-8
Server: Jetty(8.1.8.v20121106)

Unauthenticated
```

Transmitting the access token allows you to access the route as before:

```code

http -v http://localhost:8080/hello/john 'Authorization:Bearer Ieg4ahthie'

GET /hello/john HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate, compress
Authorization: Bearer Ieg4ahthie
Host: localhost:8080
User-Agent: HTTPie/0.8.0



HTTP/1.1 200 OK
Content-Length: 18
Content-Type: text/html;charset=UTF-8
Expires: Thu, 01 Jan 1970 00:00:00 GMT
Server: Jetty(8.1.8.v20121106)
Set-Cookie: JSESSIONID=1qxp4ot72v3lmr01n3vesosel;Path=/

<p>Hello, john</p>

```

The example above requires you to implement the `verify` method properly. The current state allows any request with any token access, as long as the token is provided. 

Cheers,

Chris