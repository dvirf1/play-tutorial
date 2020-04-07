---
title: "Dockerize the app"
date: 2020-04-07 00:00:12 +0200
categories: [Docker]
tags: [docker, sbt native packager]
---

Instead of creating a binary package as our artifact, we will create a docker image.

Docker use operating-system-level virtualization to deliver software in packages called containers.  
You can run many containers for the same image, just like you can run many processes of the same application (e.g you install a calculator application on your computer and open 5 processes of the calculator app).

Play is already bundled with [sbt native packager](https://sbt-native-packager.readthedocs.io/en/stable/) which can package your application in many ways, including a docker image.
Later on we will see how to add the _sbt native packager_ plugin to other sbt projects.

## Create a docker image

- open sbt shell and run `docker:publishLocal`
- open terminal and run `docker images playground` to see that an image was created with the version specified in `build.sbt`. the size of the image is ~0.5GB. we will deal with it later.
- run a docker container of your application: `docker run --rm -p 9000:9000 playground:0.0.0`
You may see the following error:
```plaintext
Oops, cannot start the server.
java.nio.file.AccessDeniedException: /opt/docker/RUNNING_PID
```
This is because play is trying to create a RUNNING_PID file, but the working directory in docker is not writable.

- Add the following to build.sbt anywhere in the file:
```scala
import com.typesafe.sbt.packager.docker.DockerChmodType
import com.typesafe.sbt.packager.docker.DockerPermissionStrategy
dockerChmodType := DockerChmodType.UserGroupWriteExecute
dockerPermissionStrategy := DockerPermissionStrategy.CopyChown
```
the options above will give read and write permissions to users and groups, and copy the files to the image while inheriting the host machine's file mode.
- run `reload` in the sbt shell for the changes to take effect.
- create a new image by running `docker:publishLocal` in the sbt shell
- run a docker container of your application again: `docker run --rm -p 9000:9000 playground:0.0.0`. The container should run normally this time.
- Browse to [http://localhost:9000](http://localhost:9000) - You should see a "Welcome to Play!" message.
- Go back to the terminal and press ctrl+C to remove the docker container (and shut down the server)

> TIP:  
> sbt generates the docker image for you.  
> However, you can still view the docker file for debugging purposes.  
> Generate a docker file by running `docker:stage` in the sbt shell.  
> It will generate a Dockerfile in your project directory under `target/docker/stage/Dockerfile`.  
> Also, you will be able to see the package's files under `target/docker/stage/opt/docker`.  
> Try it.

## Customizing the image

The size of the image we created was 0.5GB. This is because the image is based on `openjdk:8`, which is based on debian.
We want to change the operating system from debian to [Alpine](https://alpinelinux.org/), a minimal linux distribution of ~5MB.

Also, The tag of our version is `0.0.0`, which is pretty annoying.  
We can simply change the version number, but here we will keep it in case someone wants to build a binary package (or if we have another library as a subproject in our project), and we will set the docker version and name explicitly.

We will also add ports for documentation and allow publishing to a remote docker repository (e.g AWS ECR).  
Add the following to your build.sbt:

```scala
Docker / maintainer := "you@example.com" // TODO: set your info here
Docker / packageName := "playground-api"
Docker / version := sys.env.getOrElse("BUILD_NUMBER", "0")
Docker / daemonUserUid  := None
Docker / daemonUser := "daemon"
dockerExposedPorts := Seq(9000)
dockerBaseImage := "openjdk:8-jre-alpine"
dockerRepository := sys.env.get("ecr_repo")
dockerUpdateLatest := true
```

- Run `reload` in the sbt shell for the changes to take effect.
- Create a new image by running `docker:publishLocal` in the sbt shell.  
This time the image name will be `playground-api`, and the tag will be 0 and latest (thanks to `dockerUpdateLatest`).
The size of this image is now ~130MB, much of it is the JRE.  
We can do even better by [creating a docker image by using _GraalVM Native Image_ to create a standalone executable which is compiled ahead of time (called native-image)](https://i.kym-cdn.com/photos/images/newsfeed/000/000/177/800px-Sup_dawg.jpg).  
It is currently not trivial to do so for our app since Play uses Guice for runtime dependency injection by default.  
We will replace this with compile time dependency injection later on.
- Run a docker container of your application: `docker run --rm -p 9000:9000 playground-api`.  
You should get an error since our container's start script uses bash, but alpine is so minimal that it doesn't have bash pre-installed.
- We can instruct sbt-native-packager [to use Ash script](https://www.scala-sbt.org/sbt-native-packager/archetypes/misc_archetypes.html) instead of bash.  
In `build.sbt`: enable the `AshScriptPlugin` for your root project (which is currently the only project). It should look like this:  
`lazy val root = (project in file(".")).enablePlugins(PlayScala, AshScriptPlugin)`
- Run `reload` in the sbt shell for the changes to take effect.
- Create a new image by running `docker:publishLocal` in the sbt shell.
- Run a docker container of your application: `docker run --rm -p 9000:9000 playground-api`
- Browse to [http://localhost:9000](http://localhost:9000) - You should see a "Welcome to Play!" message.
- Go back to the terminal and press ctrl+C to remove the docker container (and shut down the server)

## Cleaning up

If you run `docker images` in the terminal you will see at least the following images:

- `playground-api`, tagged with 0 and latest. This is the image we created in our final successful attempt to run the app from docker.
- `playground:0.0.0`. This is the old image. remove it with `docker rmi playground:0.0.0`
- many images that were overriden by the latest successful attempt and are now dangling.

In your terminal, run `docker image prune --force`.  
Run `docker images` again to see that dangling images were indeed removed.