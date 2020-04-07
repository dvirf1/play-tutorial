---
title: "Compile-Time Dependency Injection"
date: 2020-04-07 00:00:20 +0200
categories: [Dependency Injection]
tags: [compile time dependency injection]
---

When you add a dependency to a controller (directly or indirectly) or when you refactor a class - if you forget to wire a dependency in your module, you will encounter this error at runtime.  
We would like to tackle this problem.

The main advantage of runtime dependency injection is that it is simple once you're comfortable with it.  
Most of the times, an injected parameter of type T will either have a default constructor, and then Guice will invoke it at runtime, or it will not, and then Guice will look under a module class for a method annotated with `@Provides` that returns an instance of T, and then Guice will invoke it, again - at runtime.  
Dependencies of dependencies will be instantiated by Guice in the same way, by invoking their default constructor / provider method at runtime.

Scala and Java have a special keyword for compile time dependency injection called `new` &#128521;

## Remove Guice

Let's remove the runtime dependency injection:

- Delete the module `com.example.playground.configuration.Module.scala`
- Remove this module from the registered modules in `application.conf` by deleting the key  `play.modules.enabled` along with its corresponding value
- Remove the guice jar from the project by deleting `libraryDependencies += guice` from `build.sbt`.

## Utilize Play's Built In Components

Add a new class `com.example.playground.configuration.components.AppComponents` that inherits from Play's `BuiltInComponentsFromContext` and initializes the application's components:

```scala
package com.example.playground.configuration.components

import controllers.{AssetsComponents, HomeController}
import play.api.ApplicationLoader.Context
import play.api.BuiltInComponentsFromContext
import play.api.routing.Router
import play.filters.HttpFiltersComponents
import pureconfig.ConfigSource
import pureconfig.generic.auto._
import router.Routes
import com.example.playground.configuration.Config

class AppComponents(context: Context) extends BuiltInComponentsFromContext(context)
  with HttpFiltersComponents
  with AssetsComponents {

  lazy val config: Config = ConfigSource.default.loadOrThrow[Config]
  lazy val homeController: HomeController = new HomeController(controllerComponents, config)
  override def router: Router = new Routes(httpErrorHandler, homeController, assets)

}
```

Play's application loader, which we will create shortly, should return an application ([play.api.Application](https://www.playframework.com/documentation/2.8.x/api/scala/play/api/Application.html), to be precise).  
In order to initialize the application, you should initialize its dependencies, such as its environment, request handler, error handler, etc.  
Luckily, play provides an abstract class called `BuiltInComponentsFromContext` that initializes many of these components.  
You may mix-in other SomeComponents traits to get more components, like we did with `HttpFiltersComponents` and `AssetsComponents`.

`BuiltInComponentsFromContext` leaves 2 unimplemented members:
1. `httpFilters`: a sequence of filters that run on the request headers for every request.  
You can implement it as an empty sequence as a start, or get play's recommended filters by mixing-in `HttpFiltersComponents` like we did.
2. `router`: routes the requests to their designated controller.  
The `Routes` class is built from our `conf/routes` file upon compilation.  
You can peek at its constructor and see your HomeController for example.  
In order to instantiate `Routes` we need to create HomeController and `Assets` instances.  
We can get a default implementation for `Assets` by mixing-in `AssetsComponents`, and in order to create HomeController we need to create `Config`.

## Load the application

Create a class `com.example.playground.configuration.AppLoader` that instantiates the components we just created, and returns a new play application:

```scala
package com.example.playground.configuration

import play.api._
import play.api.ApplicationLoader.Context
import com.example.playground.configuration.components.AppComponents

class AppLoader extends ApplicationLoader {
  def load(context: Context): Application = {
    new AppComponents(context).application
  }
}
```

Set the AppLoader as the entry point in `application.conf` by adding `play.application.loader` setting. e.g your play's configuration blob may look like this:

```conf
play {
	http.secret.key="MePzCIzgeI8jhPfWg8RPGCFvLobM6K8bnCabNgSdDBc="
	application.loader = com.example.playground.configuration.AppLoader
}
```

Run the app by running `run` inside the sbt shell and browse to [http://localhost:9000/](http://localhost:9000/).  
You should see the config.