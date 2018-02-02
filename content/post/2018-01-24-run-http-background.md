---
title: "Running node http in the background"
date: 2018-01-24
tags: ['nodejs']
draft: true
---

This is something simple I needed to do, but had to [dig around](
https://github.com/indexzero/http-server/issues/189) 
a bit to figure it out since I'm a Windows guy.

``` bash
npm run server > http.log 2>&1 &
```
Redirect stderr to http.log, referred to by &1. The & at the end just means run 
in the background.
