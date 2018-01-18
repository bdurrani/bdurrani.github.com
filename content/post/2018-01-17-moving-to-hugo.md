---
title: "Moving to Hugo"
date: 2018-01-18
tags: ["hugo"]
---

I'm trying to bring this blog back to life.
I realized one of the things that was preventing me from updating more often was the fact
that Jekyll is really complicated to set up on Windows. 
I would take a break from writing anything, 
come back and try to set up the toolchain, 
only to discover that something had updated and the existing toolchain broke on Windows.

Hugo is easy. There is a single executable I can download annd run.
The only dependencies I have is [pygments](http://pygments.org/), 
a python module that is a quick `pip install` away. 
No building anything from any SDK.

I've set up the blog to auto-deploy via Netlify, and added Cloudfront to get `https` 
set up on my custom domain. 

By minimizing the amount of set up work, my goal is to try and write more often,
even if it's something small and simple.

