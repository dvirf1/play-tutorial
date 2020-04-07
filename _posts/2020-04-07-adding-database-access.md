---
title: "Adding database access"
date: 2020-04-07 00:00:28 +0200
categories: [Database]
tags: [database, db api]
---

We will read the dishes from a DB instead of from a mutable set in memory.

## Add JDBC and DB driver

Add jdbc support and h2 driver to your project. in build.sbt:

```scala
libraryDependencies ++= Seq(
  jdbc,
  "com.h2database" % "h2" % "1.4.197",
  // more dependencies here
)
```

Reload the sbt-shell (or reimport sbt projects in IntelliJ to see changes in the IDE).

## Describe how to connect to the database in the config

Add a new database in `conf/application.conf`:

```
db {
    dish_db {
        driver = "org.h2.Driver"
        url = "jdbc:h2:mem:my_app_db;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE;MODE=MYSQL;INIT=CREATE SCHEMA IF NOT EXISTS dish_db\\;"
        # username = "some_user"
        # password = "some_secret"
    }
}
```

We called the database "dish_db".  
We will get this database using the `DBApi` shortly.  
The database is an h2 database that lives in memory. It creates a schema automatically if it does not exist.

## Implement a service that reads from the database

Lets refactor the DishLibrary as a specific in-memory implementation of a DishLibrary trait, and add another implementation using the DB:

```scala
package com.example.playground.dish

import scala.collection.mutable

import play.api.db.Database

sealed trait DishLibrary {
  /** returns an immutable set of all the available dishes */
  def getAllDishes: Set[Dish]

  /** returns an immutable set of all the available dishes */
  def findDish(name: String): Option[Dish]

  /** adds the dish if it does not exists. does nothing otherwise.
   * returns true if and only if the dish did not exist */
  def create(dishToCreate: Dish): Boolean

}

class DishLibraryInMemory(dishes: mutable.Set[Dish]) extends DishLibrary {

  def getAllDishes: Set[Dish] = dishes.toSet

  def findDish(name: String): Option[Dish] = dishes.find(_.name == name)

  def create(dishToCreate: Dish): Boolean = dishes.add(dishToCreate)

}

class DishLibraryDb(dbApi: play.api.db.DBApi) extends DishLibrary {
  private val database: Database = dbApi.database("dish_db")

  def getAllDishes: Set[Dish] = {
    database.withConnection { connection =>
      val resultSet = connection
        .createStatement()
        .executeQuery("SELECT DishName, Description, Price FROM Dish")

      val dishes = mutable.ListBuffer.empty[Dish]

      while (resultSet.next()) {
        val name: String = resultSet.getString("DishName")
        val description: String = resultSet.getString("Description")
        val price: Double = resultSet.getDouble("Price")
        val dish = Dish(name, description, price)
        dishes.addOne(dish)
      }

      dishes.toSet
    }
  }

  def findDish(name: String): Option[Dish] = {
    database.withConnection { connection =>
      val resultSet =
        connection
          .createStatement()
          .executeQuery(s"SELECT DishName, Description, Price FROM Dish WHERE DishName = '$name'")

      if (resultSet.next()) {
        Some(
          Dish(
            resultSet.getString("DishName"),
            resultSet.getString("Description"),
            resultSet.getInt("Price")
          )
        )
      } else None
    }
  }

  def create(dishToCreate: Dish): Boolean = {
    database.withTransaction { connection =>
      val sql = s"""INSERT INTO Dish (DishName, Description, Price)
                   |SELECT * FROM (SELECT '${dishToCreate.name}', '${dishToCreate.description}', ${dishToCreate.price}) AS tmp
                   |WHERE NOT EXISTS (
                   |    SELECT DishName FROM Dish WHERE DishName = '${dishToCreate.name}'
                   |) LIMIT 1;
                   |""".stripMargin

      val rowsAffected = connection.createStatement().executeUpdate(sql)
      rowsAffected == 1
    }
  }

}
```

We get a reference to the db using the same name as we defined in out application's config: `dbApi.database("dish_db")`.  
Now we can use `withConnection()` and `withTransaction()`, that take a function from a connection to whatever result we want to return.

> TIP: You can verify this by placing the cursor inside the curly braces and invoking the parameter info action with cmd+P, or placing the cursor on the name of the method and invoking quick documentation with ctrl+J.

For example, in `getAllDishes`, a connection is provided at `database.withConnection { connection =>`, and since the last expression in the block returns a `Set[Dish]` this is the result of the method.  
At the end of `withConnection()`, the connection and all the created statements are automatically released.  
`withTransaction()` is similar, ant at the end of it the transaction is automatically committed, unless an exception occurs.

## Inject the service

Inject DishLibraryDb as a service to `DishController`. in `AppComponents`:

```scala
import play.api.db.{DBComponents, HikariCPComponents}
import com.example.playground.dish.{Dish, DishController, DishLibrary, DishLibraryDb}

class AppComponents(context: Context) extends BuiltInComponentsFromContext(context)
  with HttpFiltersComponents
  with AssetsComponents
  with HikariCPComponents
  with DBComponents {
  // ..
  val dishLibrary: DishLibrary = new DishLibraryDb(dbApi)
  // ..
}
```

We mixed-in `DbComponents`, which provides us with an instance `dbApi` of type `DBApi`, which we then send as an argument to `DishLibraryDb`.  
Notice that `DbComponents` decalres an abstract connection of type `ConnectionPool`, which means that you will get a compile-time error if you will not provide an instance of it.  
As in most of Play's components, you don't have to create one yourself. You can get a connection pool by mixing-in `HikariCPComponents`.

You can now run the server by running `run` in the sbt-shell.  
Invoking the dishes API (e.g by browsing to [http://localhost:9000/dishes](http://localhost:9000/dishes)) will fail,
since the app will try to read from the table DISH, which doesn't exist.  
We will use DB Evolutions to create it.