---
title: "DB evolutions"
date: 2020-04-07 00:00:30 +0200
categories: [Database]
tags: [db evolutions, db api]
---

You can create your schema manually and the app will work.  
However, each developer that clones the code will have to do so too, and it results in a poor development experience.  
While we have many repositories that suffer from this problem, we can fix it with little effort by using db evolutions.

## Add evolutions support to our build

Add evolutions support to your project in `build.sbt`:

```scala
libraryDependencies ++= Seq(
  evolutions,
  // more dependencies here
)
```

Reload the sbt-shell (or reimport sbt projects in IntelliJ to see changes in the IDE).

## Define the evolutions

Open a new directory under `evolutions` in the conf directory: right click on `conf` in the _project_ pane (CMD+1) -> New -> Directory -> `evolutions/dish_db` -> Enter.

This directory will contain files like `<number>.sql`, starting from 1.
create a file `1.sql` in `evolutions/dish_db`:

```sql
# --- !Ups

CREATE TABLE IF NOT EXISTS Dish (
 DishName varchar(255) NOT NULL,
 Description varchar(255) NOT NULL,
 Price INT NOT NULL,
 Id INT NOT NULL AUTO_INCREMENT,
 PRIMARY KEY (Id)
) AUTO_INCREMENT=0;

# --- !Downs
DROP TABLE Dish;
```

Create a second file `2.sql`:

```sql
# --- !Ups

INSERT
  INTO Dish (DishName, Description, Price)
  VALUES ('Avocado Sandwich', 'Whole grain bread with Brie cheese, tomatoes and avocado', 8);

# --- !Downs

DELETE FROM Dish WHERE DishName = 'Avocado Sandwich'
```

The `!Ups` and `!Downs` comments are special and contain required transformations and how to revert them, respectively.  
The downs evolutions will not be applied in Prod mode (more on modes later).

## Run the evolutions

In `AppComponents`, mix-in `EvolutionsComponents` and add a method to run the evolutions:

```scala
import play.api.db.evolutions.{ApplicationEvolutions, EvolutionsComponents}

class AppComponents(context: Context) extends BuiltInComponentsFromContext(context)
/* with other components */
  with EvolutionsComponents {
  // some code
  def runEvolutions(): ApplicationEvolutions = applicationEvolutions
}
```

We need to call `runEvolutions()` when we start the application, so they would run automatically.  
Change `AppLoader.scala` to do so:

```scala
def load(context: Context): Application = {
  val components = new AppComponents(context)
  components.runEvolutions() // or simply components.applicationEvolutions
  components.application
}
```

Notice that `applicationEvolutions` is a lazy val, and will be instantiated when you access it.  
Therefore, we could have avoid defining `runEvolutions()` in `AppComponents` and just access this val in the loader, but we chose to be more explicit here.

## Run the app

You can now run the server by running `run` in the sbt-shell.  
Invoking the dishes API (e.g by browsing to [http://localhost:9000/dishes](http://localhost:9000/dishes)) will show
>Database 'dish_db' needs evolution!

When evolutions are activated, Play will check your database schema state before each request in DEV mode, or before starting the application in PROD mode.  
In DEV mode, if your database schema is not up to date, an error page will suggest that you synchronize your database schema by running the appropriate SQL script.
Again, more on modes later.

You can click the button to run the evolutions.  
The dishes API will now work as expected.

## Apply the evolutions automatically

Lets stop the server and configure the app to auto apply the evolutions.  
Add a configuration object for `play.evolutions.db.<your_db>` In `conf/application.conf`:
 ```conf
play {
    http.secret.key="MePzCIzgeI8jhPfWg8RPGCFvLobM6K8bnCabNgSdDBc="
    application.loader = com.example.playground.configuration.AppLoader
    evolutions.db {
        dish_db {
            enabled = true
            schema = "dish_db"
            useLocks = false
            autoApply = true
            autocommit = false
        }
    }
}
 ```

Now the evolutions will run automatically in production +
developers that clone the repository will not have to click the `apply` button when running the app.  
They will only have to use the `sbt run` command.

**Important**:
If you need to run more than a single command (usually `sbt run`, `sbt test`, `mvn run`, etc.) in order to run the code locally or test it after you clone the repository, it should raise a *red flag*{: style="color: red"} regarding the development experience.

You can now run the server by running `run` in the sbt-shell.  
The dishes API will now work as expected.
