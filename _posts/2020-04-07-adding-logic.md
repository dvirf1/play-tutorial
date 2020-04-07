---
title: "Adding Logic"
date: 2020-04-07 00:00:22 +0200
categories: [Controller]
tags: [controller, action, json]
---

We will add some logic and refactor in the next sections.  
We will build a dish menu for our users.

## Defining Routes

First, let's add three new routes to the `conf/routes` file:
```
GET     /dishes            com.example.playground.dish.DishController.allDishes
GET     /dishes/:name      com.example.playground.dish.DishController.findDish(name: String)
POST    /dishes            com.example.playground.dish.DishController.createDish
```

## Creating a Model

Now let's add a VO ([Value Object](https://en.wikipedia.org/wiki/Value_object)) that will represent a dish.  
VOs are represented in Scala with a case class.  
Add a case class `com.example.playground.dish.Dish` with name, description and price:

```scala
case class Dish(
  name: String,
  description: String,
  price: Double
)
```

## Add a controller

Add a controller `com.example.playground.dish.DishController` that implements the API above:

```scala
package com.example.playground.dish

import play.api.mvc.{AbstractController, Action, AnyContent, ControllerComponents}
import scala.collection.mutable

import play.api.libs.json.{Json, OWrites}

class DishController(
  controllerComponents: ControllerComponents,
  dishes: mutable.Set[Dish]
) extends AbstractController(controllerComponents) {

  def allDishes(): Action[AnyContent] = Action {
    implicit val dishWrites: OWrites[Dish] = Json.writes[Dish]
    Ok(Json.toJson(dishes))
  }

  def findDish(name: String): Action[AnyContent] = Action {
    val maybeDish = dishes.find(dish => dish.name == name) // can be shortened to `dishes.find(_.name == name)`
    maybeDish match {
      case Some(dish) =>
        implicit val dishWrites = Json.writes[Dish]
        Ok(Json.toJson(dish))
      case None =>
        NotFound("could not find the specified dish")
    }
  }

  def createDish(): Action[AnyContent] = Action { request =>
    request.body.asJson match {
      case Some(jsValue) =>
        val name = (jsValue \ "name").as[String]
        val description = (jsValue \ "description").as[String]
        val price = (jsValue \ "price").as[Double]
        val dishToCreate = Dish(name, description, price)
        if (dishes.contains(dishToCreate))
          Ok(s"Dish $name already exists")
        else {
          dishes += dishToCreate
          Ok(s"Added dish $name to the dish list")
        }
      case None =>
        BadRequest("Expected json as the body, but got something else")
    }
  }

}
```

The code will be explained shortly.

## Wiring the dependencies

This controller stores the dishes in-memory, in a mutable set which you will now inject:
Try to compile the code by running `compile` in the sbt shell. The compilation will fail, but the `conf/routes` file will now be compile to a new `Routes` class with a constructor that requires `DishController`.  
Create a new instance of the controller in `AppComponents` and provide it to the constructor:

```scala
import scala.collection.mutable
import com.example.playground.dish.{Dish, DishController}
// ..
val dishes = mutable.Set(
  Dish("Avocado Sandwich", "Whole grain bread with Brie cheese, tomatoes and avocado", 10),
  Dish("Ice Cream", "Chocolate + Vanilla Ice Cream", 8)
)
lazy val dishController: DishController = new DishController(controllerComponents, dishes)
override def router: Router = new Routes(httpErrorHandler, homeController, dishController, assets)
```

Lastly, create a new package, `com.example.playground.home` and move `app/HomeController` to this package. Change the package name at `conf/routes` and adjust the import at `com.example.playground.configuration.components.AppComponents`.

## Invoking the API

You can now run the server by running `run` in the sbt-shell.  
Let's invoke the API:

- Get all the existing dishes by browsing to [http://localhost:9000/dishes](http://localhost:9000/dishes) . You should see a json with 2 dishes.
- Find a dish by browsing to [http://localhost:9000/dishes/Avocado%20Sandwich](http://localhost:9000/dishes/Avocado%20Sandwich)
- Create a dish, e.g by using your [favorite rest client](https://insomnia.rest/) or using `curl` in your terminal:
```bash
curl --request POST \
  --url http://localhost:9000/dishes \
  --header 'content-type: application/json' \
  --data '{
	"name": "Karaka Spicy Ramen",
	"description": "Pork tonkotsu broth, thin noodles, pork belly chashu",
	"price": 16
}'
```


Experiment by trying to add a dish twice or finding a dish that does not exist.

## How the controller works

### Actions

Each method is implemented as an `Action`, that takes
- A block of code that evaluates to a `Result` (such as `Ok`), like in `allDishes` and `findDish`, or takes:
- A function from a request to a `Result`, like in `createDish`.

### Serializing JSON manually

In `allDishes` we would like to serialize the dishes to json, and a return a result like:
```json
[
  {
    "name": "Ice Cream",
    "description": "Chocolate + Vanilla Ice Cream",
    "price": 8
  },
  {
    "name": "Avocado Sandwich",
    "description": "Whole grain bread with Brie cheese, tomatoes and avocado",
    "price": 10
  }
]
```

To do so, we need to serialize the `dishes` instance, which has a type `mutable.Set[Dish]`.  
Play has json encoders (which serialize classes to json) for basic types, such as String, Int, Boolean.  
The scala compiler can create an encoder for collection types as well, such as List[A], Set[A], mutable.Set[A], etc. **as long as there is an encoder for A in scope**.  

#### Serializing supported types

This means that if we wanted to serialize an object of type `mutable.Set[String]` all we had to do is pass it to the toJson method like so:

```scala
val setOfStrings = mutable.Set("hello", "world")
Json.toJson(setOfStrings)
```

#### Serializing custom types

However, we would like to serialize an object of type `mutable.Set[Dish]` so we need to create an encoder for the `Dish` class.  
We could have created an encoder for dish in the following cumbersome way:

```scala
implicit val dishWritesCumbersome: Writes[Dish] = new Writes[Dish] {
      def writes(dish: Dish) = Json.obj(
        "name" -> dish.name,
        "description" -> dish.description,
        "price" -> dish.price
      )
    }
```

But this requires too much boilerplate, which means that this is too error prone. e.g, we can easily mix up the description field: `"description" -> dish.name`.

### Serializing JSON with macros

Since all the members in the Dish case class have encoders
 we can use the macro `Json.writes[Dish]` to automatically create an encoder for it.  
If a member of Dish didn't have an encoder, but was itself composed of members that have encoders, then we could build an encoder for it in the same way. e.g, Imagine a different version of Dish, that specifies the chef:

```scala
case class Chef(first: String, last: String)
case class DishV2(name: String, description: String, price: Int, chef: Chef)
// later on in the code:
implicit val chefWrites: OWrites[Chef] = Json.writes[Chef]
implicit val dishV2Writes: OWrites[DishV2] = Json.writes[DishV2]
```

Check the signature of `toJson` method by clicking with the cursor on `toJson` and using `ctrl + j`, which will open the quick documentation pane (use `ctrl` and not `cmd`. if you changed your key map, use double `shift`, that is bound in all the key maps to search everywhere, and search for `quick documentation`).  

We can see that the signature is
```scala
def toJson[T](o: T)(implicit tjs: Writes[T]): JsValue
```
toJson has a type parameter T and accepts 2 parameter lists, each containing a single parameter.  
In scala, a method with multiple parameters can have a single parameter list or multiple parameter lists. e.g
```scala
def method1(a: String, b: String): String = a + b // single parameter list that contains 2 parameters
def method2(a: String)(b: String): String = a + b // 2 parameter lists, each containing a single parameter

method1("hello", " world")
method2("hello")(" world")
```

The signature of toJson means that it can accept an object `o` of any type `T` if we (or the scala compiler) also pass it a json seralizer instance `tjs` of type `Writes[T]`.  
The type parameter T in our case is inferred as `mutable.Set[Dish]`, so these 2 calls are equivalent:
 ```scala
Json.toJson(dishes)
Json.toJson[mutable.Set[Dish]](dishes)
 ```

We are not passing this method the parameters for the second parameter list.  
Normally, the Scala compiler emit an error and say that we are missing an argument list:
 ```scala
 method2("hello") // compile time error: missing argument list for method method2
 ```
However, the second argument list for `toJson` is marked with implicit,
which means that `tjs` is an implicit parameter of type `Writes[mutable.Set[Dish]]`,
and before the compiler would give up, it will try to find an implicit value of that type, or to create one recursively.

In our case, the compiler can provide an implicit value of type `Writes[mutable.Set[Dish]]` since it is given an implicit value of type `Writes[Dish]`.

Note: Implicit value just serves as a canonical instance for its type. e.g the implicit value for `Ordering[Int]` just orders integers from low to high.


The `findDish` method searches for a dish with the same name and returns it in a similar manner, with a `Writes[Dish]` instance.  
e.g the following code is equivalent:
```scala
Json.toJson(dish)
Json.toJson[Dish](dish)
Json.toJson[Dish](dish)(dishWrites)
```

The reason that we don't get an error on the first and second call is that before the compiler throws an error that the second argument list was not supplied, it tries to find an implicit value of type `Write[Dish]` in scope, and succeeds since there is such instance that is marked with the implicit keyword.  
Try to delete this keyword and check the error:
```scala
// implicit val dishWrites = Json.writes[Dish]
val dishWrites = Json.writes[Dish]
Json.toJson(dish) // error: No Json serializer found for type com.example.playground.dish.Dish. Try to implement an implicit Writes or Format for this type
```

### Deserializing JSON: explanation of the initial implementation

The `createDish` method is implemented by providing the Action a function from a request to a result.  
All we had to do to achieve this is to add `request =>` in the beginning of the body.  

Click with the cursor on `request` and open quick documentation (ctrl+j). Notice that the type of request is `Request[AnyContent]`, which means that the body can be of any type (we will improve this later. we want the body to be of type Dish, since it contains a json of type Dish).

Here, we are manually creating a Dish instance by traversing the json object, looking for each field by name.  
This is error prone and unfortunately code like this is common.  
Sometimes the logic of deserializing the json can be nested inside several functions, and when we try to understand what a controller is doing by reading its signature we don't even know how the body looks like.  

### Deserializing JSON: take 2

Let's change the code under the first `case` clause to try to class-up the json automatically to Dish:

```scala
def createDish(): Action[AnyContent] = Action { request =>
  request.body.asJson match {
    case Some(jsValue) =>
      implicit val dishReads = Json.reads[Dish]
      Json.fromJson[Dish](jsValue) match {
        case JsSuccess(dishToCreate, path) =>
          if (dishes.contains(dishToCreate))
            Ok(s"Dish ${dishToCreate.name} already exists")
          else {
            dishes += dishToCreate
            Ok(s"Added dish ${dishToCreate.name} to the dish list")
          }
        case JsError(errors) =>
          BadRequest("Expected dish json, but got another json")
      }
    case None =>
      BadRequest("Expected json as the body, but got something else")
  }
}
```

### Deserializing JSON: take 3

Let's use Play to parse the body to json for us. We will do so by providing a json parser as the first argument (and in a new argument list) to Action:
```scala
def createDish(): Action[JsValue] = Action(parse.json) { request =>
  val jsValue: JsValue = request.body
  implicit val dishReads = Json.reads[Dish]
  Json.fromJson[Dish](jsValue) match {
    case JsSuccess(dishToCreate, path) =>
      if (dishes.contains(dishToCreate))
        Ok(s"Dish ${dishToCreate.name} already exists")
      else {
        dishes += dishToCreate
        Ok(s"Added dish ${dishToCreate.name} to the dish list")
      }
    case JsError(errors) =>
      BadRequest("Expected dish json, but got another json")
  }
}
```
Now we no longer need to check if the body can be parsed to json or not.  
Our server will automatically return a 400 Bad Request if the body is not a json.  
Another benefit is that now a new developer that reads the signature of `createDish` can tell that its body has to be json, since it returns `Action[JsValue]`.  
You can also verify (with quick documentation) that the type of request is now `Request[JsValue]`.  
However, we still need to check if the json is a Dish json or another json (e.g it may be a person json:  
`{ "name": "alice", "age": 20 }`).

### Deserializing JSON: take 4

Let's refactor and make the dishReads to be a member of the controller, and pass it to the body parser:
```scala
implicit val dishReads: Reads[Dish] = Json.reads[Dish]

def createDish(): Action[Dish] = Action(parse.json[Dish]) { request =>
  val dishToCreate: Dish = request.body
  if (dishes.contains(dishToCreate))
    Ok(s"Dish ${dishToCreate.name} already exists")
  else {
    dishes += dishToCreate
    Ok(s"Added dish ${dishToCreate.name} to the dish list")
  }
}
```

As a note, again, the following code is equivalent:
```scala
Action(parse.json[Dish]) // dishReads is passed implicitly
Action(parse.json[Dish](dishReads)) // dishReads is passed explicitly, the type parameter is passed explicitly as Dish
Action(parse.json(dishReads)) // dishReads is passed explicitly and the type patameter in inferred
```

Now a developer that reads the signature for `createDish` can immediately tell that the body is expected to be of type `Dish`.  
We have come a long way since the first implementation.  
Now we just deserialize the request body to the desired type, do some logic, and serialize the result.

### Refining the signature of actions that do not read the request

Lastly, we will change the other actions to have a type signature that conveys that they don't use the request body (by returning `Action[Unit]`).  
We can already tell that these methods do not parse the body, since they don't take the request as a parameter, so choose your preferred implementation:

```scala
def allDishes(): Action[Unit] = Action(parse.empty) { _ =>
  /* code */
}
 
def findDish(name: String): Action[Unit] = Action(parse.empty) { _ =>
  /* code */
}
```

We could have named the parameter of the function literal (from `Request[Unit]` to `Result`) that we are passing. e.g  
`Action(parse.empty) { request => ... ` or  
`Action(parse.empty) { requestWithUnitBody => ... `  
We named it `_`, which is useful since we are not using it and we have no reason to name it with a special name.
