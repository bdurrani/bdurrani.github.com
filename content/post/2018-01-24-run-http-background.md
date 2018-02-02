---
title: "Running node http in the background"
date: 2018-01-24
tags: ['nodejs', 'linux']
---

This is something simple I needed to do, but had to [dig around](
https://github.com/indexzero/http-server/issues/189) 
a bit to figure it out since I'm a Windows guy.

I just wanted to run `http-server` as a background job in linux.

``` bash
npm run server > http.log 2>&1 &
```

Redirect stderr (the 2) to http.log, referred to by &1. The & at the end just means run 
in the background.
