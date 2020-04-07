---
title: About

# The About page
# v2.0
# https://github.com/cotes2020/jekyll-theme-chirpy
# © 2017-2019 Cotes Chung
# MIT License
---

In this tutorial we will create a new _Play Framework_ application from scratch and play around with its build.  
We will see how to work with config, use compile-time dependency injection,
how to dockerize the app, work with json, access the DB, construct the logic so we will
have easier time to update _Play Framework_ and so on.

We assume that you already read (parts of) the [Play Framework documentation](https://www.playframework.com/documentation),
and that you want to play around with some code yourself.

I wrote this tutorial originally as an onboarding task for new developers at my company,
and therefore it may be opinionated (e.g towards using certain IDE/libraries that we like).  
Although a production app will be implemented differently, this tutorial tries to show how to work with _Play Framework_
in the day to day - not only when developing new projects, but also when working with existing projects.  

For educational purposes, we sometimes do something one way and change it later on.  
For example, we first use runtime dependency injection. After a few chapters, when the reader
understands the advantages and disadvantages of it and how to work with it in existing repositories that already utilize runtime
dependency injection, we refactor our app to use compile-time dependency injection.