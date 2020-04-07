---
title: "Create a package"
date: 2020-04-07 00:00:10 +0200
categories: [sbt dist]
tags: [package, dist, application secret]
---

## Create a package

Let's create a binary version of our application.  
Open the sbt shell and type `dist`.  
This task will create a new zip in your project directory under `target/universal/` directory.  
The package name will be composed of the project name and version as set in build.sbt.  
You can view it in your _project_ tool window (CMD+1), Safari's finder or the terminal.

Open terminal, go to your project's directory, and unzip the package with `unzip target/universal/playground-0.0.0.zip -d target/universal`.

Open `target/universal/playground-0.0.0` directory.  
This directory contains 4 sub directories:

- `bin`: contains scripts to run the app in linux and windows.
- `conf`: contains the app's configuration from `<project_dir>/conf/`
- `lib`: contains the application's [compile-dependencies](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html#Dependency_Scope) which will be loaded to the classpath
- `share` containing public and other resources.

Open terminal, change directory to the package directory and run the app:
```bash
cd target/universal/playground-0.0.0
bin/playground
```

You should get an error message saying that the application is not secure since the application secret has not been set and we are running in prod mode (e.g running normally from a binary package and not from IntelliJ).  
Play's modes can make the development experience much better. More on that in a later chapter.

## Generate an application secret

Run the app again with a random secret key:  
`bin/playground -Dplay.http.secret.key=aabbccddee`.  
Browse to http://localhost:9000 - You should see a "Welcome to Play!" message.  
Go back to the terminal and press ctrl+C to shut down the server.

We will now set this option permanently in the `application.conf` configuration file.

Open `conf/application.conf` and see that this file is empty.  
Specifically, it does not define a secret key for signing session cookies and CSRF tokens and for built in encryption utilities.

- Let's generate a random secret ourselves.  
Option 1: in the sbt shell, run `playGenerateSecret`  
Option 2: in your terminal, run `head -c 32 /dev/urandom | base64`

- Set the value above as the application secret in the configuration.
Add the following to `application.conf`, and change the secret key to the secret key you generated:
```hocon
play {
	http.secret.key="MePzCIzgeI8jhPfWg8RPGCFvLobM6K8bnCabNgSdDBc="
}
```
> Note that we could have also added:
> ```hocon
> play.http.secret.key="MePzCIzgeI8jhPfWg8RPGCFvLobM6K8bnCabNgSdDBc="
> ```
> but since we are going to set more options under play.* later on, the first option will group all of play's settings for us.

- Delete the old package, create a new one and run it:
In the terminal, from the project directory:
```bash
rm -rf target/universal/playground-0.0.0*
```
In sbt shell - run `dist`.
In the terminal:
```bash
unzip target/universal/playground-0.0.0.zip -d target/universal
cd target/universal/playground-0.0.0/bin/
./playground
```

Browse to [http://localhost:9000](http://localhost:9000) - You should see a "Welcome to Play!" message.  
Go back to the terminal and press ctrl+C to shut down the server.