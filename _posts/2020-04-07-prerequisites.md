---
title: "Prerequisites"
date: 2020-04-07 00:00:02 +0200
categories: [Prerequisites]
tags: [prerequisites, mac]
---

- Download and install JDK 8+ from [AdoptOpenJDK's website](https://adoptopenjdk.net/). Use HotSpot JVM (default).

- Download and install [Docker](https://www.docker.com/).

- Download and install SBT and Scala, e.g using Homebrew or SDKMAN:

```bash
# update brew
brew update
# install sbt if missing
brew list sbt || brew install sbt
# optional: install scala if missing
brew list scala || brew install scala
```

There are many Scala versions (2.12.8, 2.12.9, 2.13.0, ..) and sbt will automatically fetch the correct Scala version.  
Therefore, you don't have to install Scala locally, but it is useful when you want to play with it quickly.  
We will see an example later.