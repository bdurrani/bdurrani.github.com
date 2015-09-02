---
layout: post
title: Adventures in optimizations
comments: false
---

I started out trying to figure out how sqeeze out optimizations for 
some SNR calculations, which involved the following bit of math.

<div>
$$norm = \sum_{i=0}^{N} golden_i^2$$
$$distance = \sum_{i=0}^N (golden_i^2 - variance_i^2)$$
</div>

The language of choice was c#, and this was for a 64-bit process.

One idea was to try and use SSE to improve the throughput of the computation. 
Specifically, SSE2, since all inputs involved 64-bit doubles. 
SSE is all about 32-bit floats and ints. 
The Visual Studio compiler (2012 and above) has the ability to auto-vectorize looks
using SSE2. What better way to learn how to best use SSE2 than reading the compiler 
output. With that, I set about reading up on 64-bit assembly code.

64-bit assembly is a wide topic by itself. I had some a bit of x86 assembly back in my
university days, but it's been a while.
I was surprised to learn about the fact that there is only one calling convention for
64-bit code, which is how **fastcall** was with x86 assembly. 
You can read more about it [here][1], but the jist of it is that **fastcall** uses registers
as the first four arguments, and the stack frame for the rest.

So I started writing bits of sample code and looking at the generated assembly.


[1]:https://msdn.microsoft.com/en-us/library/7kcdt6fy.aspx 
