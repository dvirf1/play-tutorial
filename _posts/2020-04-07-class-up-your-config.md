---
title: "Class-up Your Config"
date: 2020-04-07 00:00:18 +0200
categories: [Config]
tags: [typesafe config, pureconfig]
---

In the previous example there was some repetition.  
Parsing the config to a person was a bit complex, as well as creating a Path instance, UUID instance and so on.

A bigger problem is that it is not straightforward to write a system test that ensures that the config keys are read properly in the code.  
For example, think what would happen if we change `application-id` to `app-id` in the config but not in the code (or vice versa), or change `min-duration` type in the code to Long.

## Add PureConfig as a dependency

We will use [PureConfig](https://pureconfig.github.io/docs/) to class-up our config.  
Go to its website and add the latest version to your build.sbt, e.g:

```scala
libraryDependencies ++= Seq(
  // more dependencies here
  "com.github.pureconfig" %% "pureconfig" % "0.12.0",
)
```
and then `reload` the sbt shell, or `Import Changes` in IntelliJ so it will download the jar for you and help you with code completion (it will also reload the sbt shell behind the scenes).

## Model the config

Create a new `Config` case class under `com.example.playground.configuration`.  
By convention, pure-config expects that each kebab-case-key in `application.conf` will be a camelCaseMember in the config class:

```scala
case class Config(
  name: String,
  host: URL,
  port: Int,
  hotels: List[String],
  minDuration: FiniteDuration,
  applicationId: UUID,
  sshDirectory: Path,
  developer: Person
)
```

## Load the config

create a new injector module `com.example.playground.configuration.Module` that will provide the config using pure-config to load it:

```scala
package com.example.playground.configuration

import com.google.inject.{AbstractModule, Provides, Singleton}
import pureconfig.ConfigSource
import pureconfig.generic.auto._

class Module extends AbstractModule {
  val config: Config = ConfigSource.default.loadOrThrow[Config]

  @Provides()
  @Singleton()
  def configProvider: Config = config
}
```

Enable the module above by adding it to Play's config at `conf/application.conf` under `play.modules.enabled`, e.g your play's blob in the config may look like this:

```
play {
	http.secret.key="MePzCIzgeI8jhPfWg8RPGCFvLobM6K8bnCabNgSdDBc="
	modules.enabled += "com.example.playground.configuration.Module"
}
```

## Use the config

Modify `app/controllers/HomeController.scala` to require the _Config_ VO instead of the _Configuration_ class:

```scala
import com.example.playground.configuration.Config
//..
class HomeController @Inject()(cc: ControllerComponents, config: Config) extends AbstractController(cc) {
  //..
  def index() = Action { implicit request: Request[AnyContent] =>
    val message =
      s"""
         |name: ${config.name}
         |host: ${config.host}
         |port: ${config.port}
         |hotels: ${config.hotels}
         |minDuration: ${config.minDuration}
         |applicationId: ${config.applicationId}
         |sshDirectory: ${config.sshDirectory}
         |developer: ${config.developer}
         |""".stripMargin
    println(message)
    Ok(message)
  }
}
```

Notice that you also get code completion:

![config with code completion]({{ "/assets/img/tutorial/class-up-your-config/config-with-code-completion.gif" | relative_url }}){:style="border:1px solid black;" width='90%'}

Run the app by running `run` inside the sbt shell and browse to [http://localhost:9000/](http://localhost:9000/).  
You should see the config.