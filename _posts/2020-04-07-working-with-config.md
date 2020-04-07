---
title: "Working With Config"
date: 2020-04-07 00:00:16 +0200
categories: [Config]
tags: [typesafe config]
---

We will first start by adding demo config to our app and reading it manually.  
Add the following to your `conf/application.conf` (not inside, but rather under play's blob):

```conf
name: hotels_best_dishes
host: "https://example.com"
port: 80
hotels: [
  "Club Hotel Lutraky Greece",
  "Four Seasons",
  "Ritz",
  "Waldorf Astoria"
]
min-duration: 2 days
application-id: 00112233-4455-6677-8899-aabbccddeeff
ssh-directory: /home/whoever/.ssh
developer: {
  name: alice,
  age: 20
}
```

We will read values from the config and convert each one to an appropriate built-in type.  
We will also convert the `developer` value to a custom _Person_ class.

In the _project_ tool window (CMD+1), right click on `app` and select `New` -> `Scala Class` and create a case class called `com.example.playground.configuration.Person`.  
Add a name and an age to it. e.g:

```scala
case class Person(name: String, age: Int)
```

Go to `app/controllers/HomeController.scala` and inject an instance of a [Configuration](https://www.playframework.com/documentation/2.8.x/api/scala/play/api/Configuration.html) object to this class by adding `config: Configuration` as a constructor parameter.  
The order of the constructor parameters does not matter - Guice will [create a new instance](https://github.com/google/guice/wiki/JustInTimeBindings) for each of the class parameters using `Eligible Constructors`, `@ImplementedBy` or `@ProvidedBy`.

> Side note: 99% of the time, "injecting a dependency" is just a fancy way of saying "passing it as an argument using the consructor".  
> Your classes will usually be either service classes (that provide behaviour) or value classes (that represent information. such classes are represented in scala with `case class`).  
> As a rule of thumb, while you can create value objects anywhere, you want to inject your services, so that you will be able to inject another service, possibly a mock, when testing or running locally.  
> With object oriented programming, an object also contains its behaviors. Although you will not have a clear separation, you want to inject the object's services.

Change the `index` method so that it will read the config values to the appropriate types.  
Then print them to the sbt console and describe the config in the response.  
Try to do this without copy-paste.

The method should be similar to this:

```scala
import java.net.URL
import java.nio.file.Paths
import java.util.UUID
import scala.concurrent.duration.FiniteDuration
import com.example.playground.configuration.Person

// class definition here

def index() = Action { implicit request: Request[AnyContent] =>
  val name = config.get[String]("name")
  val host = config.get[URL]("host")
  val port = config.get[Int]("port")
  val hotels = config.underlying.getStringList("hotels") // notice that getting a Person list here will be more complex
  val minDuration = config.get[FiniteDuration]("min-duration")
  val applicationId = UUID.fromString(config.get[String]("application-id"))
  val sshDirectory = Paths.get(config.get[String]("ssh-directory"))
  val developer = Person(config.get[String]("developer.name"), config.get[Int]("developer.age"))

  val message =
    s"""
       |name: $name
       |host: $host
       |port: $port
       |hotels: $hotels
       |minDuration: $minDuration
       |applicationId: $applicationId
       |sshDirectory: $sshDirectory
       |developer: $developer
       |""".stripMargin
  println(message)
  Ok(message)
}
```

Run the app by running `run` inside the sbt shell and browse to [http://localhost:9000/](http://localhost:9000/).  
You should see the config.
