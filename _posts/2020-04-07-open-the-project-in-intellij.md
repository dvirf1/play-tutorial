---
title: "Open the project in IntelliJ"
date: 2020-04-07 00:00:08 +0200
categories: [IntelliJ]
tags: [intellij]
---

## Open the project

In [IntelliJ](https://www.jetbrains.com/idea/), click `File -> Open`, navigate to the project directory (`playground`) and click open.

> TIP:
> tick `v` under:
>
> - Library sources
> - sbt sources
> - use sbt shell for imports
> - use sbt shell for builds
>
> It is especially important to use sbt shell for builds, so your IDE will be as similar as possible to how the application is built on the [CI](https://en.wikipedia.org/wiki/Continuous_integration) server (e.g _[Travis CI](https://travis-ci.org/)_).

Select JDK 1.8+ as the project's JDK and open the project.

![open the project in intellij]({{ "/assets/img/tutorial/open-the-project-in-intellij/intellij.png" | relative_url }}){:style="border:1px solid black;" width='90%'}

IntelliJ will open the project, index the files, and open an sbt shell.  
You can open the sbt shell with Cmd+Shift+S, or by opening the Find Action dialog (Cmd+Shift+A) and typing `sbt shell`.

## sbt loads *.sbt files on startup

- Open the `build.sbt` file and see that the version of your application is `1.0-SNAPSHOT`.
- Open the sbt shell and type `show version`. It will print the same version as above.
- Change the version in the build.sbt file  
from `version := "1.0-SNAPSHOT"`  
to `version := s"0.${sys.env.getOrElse("BUILD_NUMBER", "0")}.0"` which will set the version to 0.0.0 when you create a package locally, and to 0.X.0 when you create a package in TeamCity
- Run `show version` again in the sbt shell. notice that the old version, 1.0-SNAPSHOT, is still displayed.


This is because once sbt starts, it loads all the `*.sbt` files in the project directory
(It is best practice to call the main build file `build.sbt`, but `aaa.sbt` would work just fine).  
Therefore, When we change an sbt file we need to reload the sbt shell,
so sbt would start again, and changes will take effect.  
Similarly, if we want build changes to take effect in IntelliJ
we need to click `Import changes` in the _sbt import_ sticky balloon,
or go to the sbt pane and click the button with the refresh icon, labeled "Reimport all sbt projects".  
By ticking the `use sbt` options when openning the project, we made sure that when we import the changes made to the build, IntelliJ will also reload the sbt shell.

![import changes]({{ "/assets/img/tutorial/open-the-project-in-intellij/intellij-import-sbt-changes.png" | relative_url }}){:style="border:1px solid black;" width='30%'}

- type `reload` in the sbt shell, or press the button with the rounded green arrow, labeled "Restart sbt shell"
- run `show version` again and see that now the correct version is shown.

![sbt shell example]({{ "/assets/img/tutorial/open-the-project-in-intellij/sbt-shell-example.png" | relative_url }}){:style="border:1px solid black;" width='30%'}

### Conclusion

Since sbt loads all the `*.sbt` files on startup, you should reload the sbt shell whenever you change a `*.sbt` file.  
You don't need to reload the sbt shell when you change a `*.scala` / `*.java` source files.
