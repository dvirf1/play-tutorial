---
title: "Creating a skeleton project"
date: 2020-04-07 00:00:04 +0200
categories: [Giter8]
tags: [giter8, skeleton]
---

## Generate a skeleton with giter8

In terminal, in your `repos` directory, create a new project using a giter8 template for [play-scala-seed](https://github.com/playframework/play-scala-seed.g8):

```bash
cd ~/repos
sbt new playframework/play-scala-seed.g8
```
You will be asked for more information.  
Fill out the following:

|Field|Value|
|:----|:----|
|name | playground
|organization | com.example

This will create a new project skeleton with the latest play version inside `~/repos/playground` based on Play's gitter8 template, which sets up your project folders, build structure, and development environment - all with one command.

## Use sbt to build the project

We will switch to IntelliJ soon, but for now let's open an sbt shell inside the project dir:

```bash
cd playground
sbt
```

_sbt_ is a  build tool - you can compile, test, package, run your app, and define other tasks using sbt.
At your company, you will probably use sbt in your [CI](https://en.wikipedia.org/wiki/Continuous_integration) server (e.g [Travis CI](https://travis-ci.org/)) to do such tasks.

let's compile (first time may take a few minutes):

```sbt
compile
```

![sbt compile]({{ "/assets/img/tutorial/creating-a-skeleton-project/sbt-compile.png" | relative_url }}){:style="border:1px solid black;" width='100%'}

The first time should take a few minutes since sbt downloads the dependencies of your application.

>Side note: The command above runs all the sbt tasks up to and including compile.  
>This includes many other tasks, such as updating the dependencies.  
>Verify with:
>
>```sbt
>inspect tree compile
>```

Tasks can be incremental, and therefore invoking `compile` again without changing any source files should take less than a second.

Running `update` will update our library dependencies (if necessary).

Let's run the app by typing `run` in sbt shell.  
It will serve the app in _Dev_ mode (more on modes later) on port 9000.  
Browse to [http://localhost:9000](http://localhost:9000) - You should see a "Welcome to Play!" message.  
Go back to the sbt console and press `Enter` to stop the server.
Press ctrl+D to shut down sbt.