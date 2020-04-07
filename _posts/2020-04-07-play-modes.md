---
title: "Play modes"
date: 2020-04-07 00:00:30 +0200
categories: [Modes]
tags: [modes]
---

Play has 3 modes: Dev, Prod and Test.

Whenever a play application is started, it is started using one of the 3 modes above.  
For example, when running Play with `sbt run` the mode will be Dev.  
When running with `sbt test` the mode will be Test.  
When running the app after creating a distribution of it (e.g with `sbt dist` or `sbt docker:publishLocal`) the mode will be Prod.  

It is important to note that these 3 modes are part of Play's API.  
Many companies have environments for staging and production.  
The distributed play application will run in Prod mode in all live environments - including the staging environment.

Using Play's modes allows us to inject different dependencies for each mode, which provides better development experience.  
For instance, we can inject a service implementation that interacts with the DB in Prod mode, and an in-memory implementation in Dev and Test mode.

## Instantiate the components class according to the mode

Inside the package `com.example.playground.configuration.components`, create 3 new classes that inherit from `AppComponents`: `DevComponents`, `ProdComponents` and `TestComponents`.
Here is an example for `DevComponents`:


```scala
package com.example.playground.configuration.components

import play.api.ApplicationLoader.Context

class DevComponents(context: Context) extends AppComponents(context) {
  // wire dependencies specific for dev
}
```

Now we have a components class for each mode, and therefore `AppComponents` can be made abstract:

```scala
abstract class AppComponents(context: Context) extends /* .. */
```
Also, you may print the mode to convince yourself the app is started with the expected mode. To do so, add `println(s"running in ${environment.mode} mode")` in `AppComponents`'s body.

Let's provide a factory that will provide the components class depending on the mode that is given to us in the context by the application loader.  
In the same file (AppComponents.scala), create a companion object for the `AppComponents` class that will do so:

```scala
abstract class AppComponents(context: Context) extends /* .. */ {

} // end of AppComponents class

object AppComponents {
  def apply(context: Context): AppComponents =
    context.environment.mode match {
      case Mode.Dev  => new DevComponents(context)
      case Mode.Prod => new ProdComponents(context)
      case Mode.Test => new TestComponents(context)
    }
}
```

Now in `AppLoader` we no longer want (or can) to create a new AppComponents instance, since we already have a components class for each mode, and since AppComponents is abstract.  
Instead, we will call the `apply` method by omitting the `new` keyword:

```scala
val components = AppComponents(context)
```
`apply` is a special method, and `x.apply(args)` is equivalent to `x(args)`.

## Run the app

Run the app in dev mode with by running `run` in the sbt-shell, or in prod mode, e.g by publishing a docker image and running it:
```bash
sbt docker:publishLocal
docker run --rm -p 9000:9000 playground-api
```
Everything should work the same.

## Inject different implementations for different modes

Lets inject the in-memory dish library in Dev and Test modes, and the db implementation in Prod mode.

First, we'll go to `AppComponents` class and make dish library abstract, by deleting the implementation:

```scala
abstract class AppComponents(context: Context) extends /* .. */ {

  val dishLibrary: DishLibrary
```

We will override `dishLibrary` in `ProdComponents` and provide the db implementation:

```scala
import com.example.playground.dish.{DishLibrary, DishLibraryDb}

class ProdComponents(context: Context) extends AppComponents(context) {
  // wire dependencies specific for prod
  override val dishLibrary: DishLibrary = new DishLibraryDb(dbApi)
}
```

And we will provide the in-memory implementation in `DevComponents`, and similarly in `TestComponents`:

```scala
val dishes = mutable.Set(
  Dish("Avocado Sandwich", "Whole grain bread with Brie cheese, tomatoes and avocado", 10),
  Dish("Ice Cream", "Chocolate + Vanilla Ice Cream", 8)
)
override val dishLibrary: DishLibrary = new DishLibraryInMemory(dishes)
```

Running the app now will provide the appropriate implementation of the dish library according the the given mode.

We will want to do the same for other dependencies as well.  
For example, in the future we might want to emit stats with [java-statsd-client](https://github.com/tim-group/java-statsd-client).  
In that case, we will create an abstract member of type `StatsDClient` in `AppComponents` with:
```scala
val statsDClient: StatsDClient
```
Then in `ProdComponents` we will override it by instantiating a new `NonBlockingStatsDClient`,
while in dev and test modes we will override it by instantiating a new `NoOpStatsDClient`,
which is a [null StatsD client](https://en.wikipedia.org/wiki/Null_object_pattern).

## Allow overriding the config in Prod mode

Lastly, we want to provide the option to override db settings in `conf/application.conf`.  
To do so, we will introduce an optional substitution for each setting that we would like to make overridable.  
Change the `db` object in `application.conf`:
```
db {
    dish_db {
        driver = "org.h2.Driver"
        driver = ${?dish_db_driver}
        url = "jdbc:h2:mem:my_app_db;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE;MODE=MYSQL;INIT=CREATE SCHEMA IF NOT EXISTS dish_db\\;"
        url = ${?dish_db_url}
        username = ""
        username = ${?dish_db_username}
        password = ""
        password = ${?dish_db_password}
    }
}
```
The `driver`, for example, is "org.h2.Driver" by default.  
However, if a substitution is provided under `dish_db_driver`, then it will be used.  
A substitution that is not found in the configuration tree, is searched in the environment variables.  
In practice, it means that if we will provide an environment variable called `dish_db_driver` with the value `com.amazon.redshift.jdbc.Driver`, then it will set Redshift as the driver for the db (Also, we will obviously add [redshift driver](https://mvnrepository.com/artifact/com.amazon.redshift/redshift-jdbc42) to the classpath in such case by adding it as a library dependency in `build.sbt`).  
The values for this config can be obtained with secret management tools such as [HashiCorp Vault](https://www.hashicorp.com/products/vault).

Also, keep in mind that in our case there was no problem in providing the db implementation of the dish library in Dev mode, since we have already made sure that the db will work out-of-the-box upon cloning the repository, by using h2 and evolutions.

Unfortunately, many repos do not work this way, and requires you to install a local db.