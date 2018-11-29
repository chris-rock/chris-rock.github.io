---
title: Implicit JSON conversion with Scala
author: Christoph Hartmann
date: 2014-09-21
tags:
  - scala
aliases:
  - /articles/scala-implicit-conversion/
---

In my last blog posts about Scala, I explained [Scalatra with Bearer Authentication](http://lollyrock.com/articles/scalatra-bearer-authentication/) and [Asynchronous HTTP requests with Scala and Dispatch](http://lollyrock.com/articles/scala-http-requests/). Today I am going to focus on JSON. We will encode data types to JSON and decode JSON into existing data structures.

## JSON Library

[Plenty of options](http://stackoverflow.com/questions/8054018/json-library-for-scala) are available for JSON in Scala. It really depends on your setup and may depend on your web framework decision. I am not going to start an argumentation about the best toolkit available. The following post will stick with [Argonaut](http://argonaut.io), but the general approach works with other JSON frameworks as well. I selected it because it fits my function development approach in Scala and works well with [Scalatra](http://www.scalatra.org/). Be aware that Scalatra itself [promotes](http://scalatra.org/2.3/guides/formats/json.html) [JSON4s](http://json4s.org/) with [Jackson](http://jackson.codehaus.org/), but I found it more difficult to use that Argonaut.

## Conversion

Before we talk about the conversion, we need to define the data model. I use the [slick sample structuree](http://slick.typesafe.com/doc/2.1.0/orm-to-slick.html)

```scala
  case class Person (id: Int,name: String,age: Int,addressId:Int)
  case class Address (id: Int, street: String,city: String)

  // no direct reference, to fit with slick database models
  case class PersonWithAddress(person: Person, address: Address)
```

Now we are able to store a person with address:

```scala
  val person = Person(0, "John Rambo" , 67, 0)
  val address = Address(0, "101 W Main St", "Madison, Kentucky")
  val pa = PersonWithAddress(person, address)
```

If you are developing REST services you may use JSON as input and output data format. It would be a horror if we have to manually construct the conversion for each route. Therefore we want something like:

```scala
  // convert the person to json
  val json = pa.asJson
```

Scala provides a cool feature for [implicit type conversion](http://docs.scala-lang.org/tutorials/FAQ/finding-implicits.html). This is very handy for our use case and fits with [argonaut codec](http://argonaut.io/doc/codec/). For our case, the following definition is appropriate: 

```scala
  // implicit conversion with argonaut
  implicit def PersonAddressEncodeJson: EncodeJson[PersonWithAddress] =
  EncodeJson((p: PersonWithAddress) =>
    ("id" := p.person.id) ->:
    ("name" := p.person.name) ->:
    ("age" := p.person.age) ->:
    ("address" := Json (
      ("id" := p.address.id),
      ("street" := p.address.street),
      ("city" := p.address.city)
    )
  ) ->: jEmptyObject)
```

Now we have all pieces together to convert Scala data types to JSON. The result looks as follows:

```json
{
  "id": 0,
  "name": "John Rambo",
  "age": 67,
  "address": {
    "id": 0,
    "street": "101 W Main St",
    "city": "Madison, Kentucky"
  }
}
```

To consume JSON data we write a decoder:

```scala
implicit def PersonAddressDecodeJson: DecodeJson[PersonWithAddress] =
  DecodeJson(c => for {

    id <- (c --\ "id").as[Int]
    name <- (c --\ "name").as[String]
    age <- (c --\ "age").as[Int]
    address <- (c --\ "address").as[Json]

    // extract data from address
    addressid <- (address.acursor --\ "id").as[Int]
    street <- (address.acursor --\ "street").as[String]
    city <- (address.acursor --\ "city").as[String]

  } yield PersonWithAddress(Person(id, name, age, addressid), Address(addressid, street, city)))
```

As a result we get a valid Scala object with the expected data

```scala
PersonWithAddress(Person(0,John Rambo,67,0),Address(0,101 W Main St,Madison, Kentucky))
```

Compared to Nodejs, this decode is also a type validation. The conversion will fail, if one parameter is missing. In case you would like to make parts optional, you need to adapt the case class and use `Option[Int]` instead of `Int`. Additionally you need to adapt the decode and parse to `Option[Int]`:

```scala
  age <- (c --\ "age").as[Option[Int]]
```

# The complete sample:

```scala
import argonaut._, Argonaut._

object ImplicitConversion {

  // data model based on http://slick.typesafe.com/doc/2.1.0/orm-to-slick.html
  case class Person (id: Int,name: String,age: Int,addressId:Int)
  case class Address (id: Int, street: String,city: String)

  // no direct reference, to fit with slick database models
  case class PersonWithAddress(person: Person, address: Address)

  // implicit conversion with argonaut
  implicit def PersonAddressEncodeJson: EncodeJson[PersonWithAddress] =
  EncodeJson((p: PersonWithAddress) =>
    ("id" := p.person.id) ->:
    ("name" := p.person.name) ->:
    ("age" := p.person.age) ->:
    ("address" := Json (
      ("id" := p.address.id),
      ("street" := p.address.street),
      ("city" := p.address.city)
    )
  ) ->: jEmptyObject)

  implicit def PersonAddressDecodeJson: DecodeJson[PersonWithAddress] =
  DecodeJson(c => for {

    id <- (c --\ "id").as[Int]
    name <- (c --\ "name").as[String]
    age <- (c --\ "age").as[Int]
    address <- (c --\ "address").as[Json]

    // extract data from address
    addressid <- (address.acursor --\ "id").as[Int]
    street <- (address.acursor --\ "street").as[String]
    city <- (address.acursor --\ "city").as[String]

  } yield PersonWithAddress(Person(id, name, age, addressid), Address(addressid, street, city)))

  def main(args: Array[String]) {
    // running a sample
    val person = Person(0, "John Rambo" , 67, 0)
    val address = Address(0, "101 W Main St", "Madison, Kentucky")
    val pa = PersonWithAddress(person, address)

    // convert the person to json
    val json = pa.asJson
    var content = json.toString()

    println (content)

    // we should get a person instance here
    var padecoded : PersonWithAddress = content.decodeOption[PersonWithAddress].get

    println (padecoded)
  }
}
```

## Conclusion

With a few lines of code, we encapsulate the complete JSON encoding and decoding. Its amazing how easy it is. It may not be as intuitive as in Nodejs, but comes with additional type checks, which is generally more complex in Nodejs.

If you have any questions contact me via [Twitter @chri_hartmann](https://twitter.com/chri_hartmann) or [Github](https://github.com/chris-rock)
