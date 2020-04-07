---
title: "Moving the logic to a library"
date: 2020-04-07 00:00:24 +0200
categories: [Library]
tags: [compile time dependency injection, tests]
---

Even though the marshalling and unmarshalling of the request is much simpler now,
our logic is still written in the controller.  
This means that in order to test our logic we need to create an application (a fake one),
which is slow and forces us to use Play's API in our tests,
which in turn may hinder us from upgrading to newer versions of play.  
Unfortunately, we have several repositories in inovid suffer from this problem.

Let's move the logic (as simple as it is in our example) to a separate library, and consume it from the controller.

## Extract the logic to a service

Add a new class `com.example.playground.dish.DishLibrary` that performs the logic that we implemented in the controller:

```scala
package com.example.playground.dish

import scala.collection.mutable

class DishLibrary(dishes: mutable.Set[Dish]) {

  /** returns an immutable set of all the available dishes */
  def getAllDishes: Set[Dish] = dishes.toSet

  /** returns an immutable set of all the available dishes */
  def findDish(name: String): Option[Dish] = dishes.find(_.name == name)

  /** adds the dish if it does not exists. does nothing otherwise.
   * returns true if and only if the dish did not exist */
  def createDish(dishToCreate: Dish): Boolean = dishes.add(dishToCreate)

}
```

## Inject the service

Inject the dish library to the controller instead of the mutable set:
```scala
class DishController(
  controllerComponents: ControllerComponents,
  dishLibrary: DishLibrary
) extends ...
```

change the actions to use the dish library:

```scala
def allDishes(): Action[Unit] = Action(parse.empty) { _ =>
  implicit val dishWrites: Writes[Dish] = Json.writes[Dish]
  Ok(Json.toJson(dishLibrary.getAllDishes))
}

def findDish(name: String): Action[Unit] = Action(parse.empty) { _ =>
  val maybeDish = dishLibrary.findDish(name)
  maybeDish match {
    case Some(dish) =>
      implicit val dishWrites = Json.writes[Dish]
      Ok(Json.toJson(dish))
    case None =>
      NotFound("could not find the specified dish")
  }
}

implicit val dishReads: Reads[Dish] = Json.reads[Dish]
def createDish(): Action[Dish] = Action(parse.json[Dish]) { request =>
  val dishToCreate: Dish = request.body
  if (dishLibrary.create(dishToCreate))
    Ok(s"Added dish ${dishToCreate.name} to the dish list")
  else
    Ok(s"Dish ${dishToCreate.name} already exists")
}
}
```

Instantiate the dish library in `AppComponents` and pass it to the controller:
```scala
val dishLibrary: DishLibrary = new DishLibrary(dishes)
lazy val dishController: DishController = new DishController(controllerComponents, dishLibrary)
```


Run the app and invoke the API again.

Everything still works as before, and this change may appear to be very small, but imagine how would you test the logic if it was more complicated than simple one-liners.  
Now our actions just deserialize the request body to the desired type, call a library to perform the logic, and serialize the result.  
It makes it very easy to test the logic now.