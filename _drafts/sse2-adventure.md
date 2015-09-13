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

### Learning assembly with a compilers help

64-bit assembly is a wide topic by itself. I had some a bit of x86 assembly back in my
university days, but it's been a while.
I was surprised to learn about the fact that there is only one calling convention for
64-bit code, which is how **fastcall** was with x86 assembly. 
You can read more about it [here][1], but the jist of it is that **fastcall** uses registers
as the first four arguments, and the stack frame for the rest.

So I started writing bits of sample code and looking at the generated assembly. All the samples I show
were tested using Visual Studio 2013.

Here is a simple example, just to see how the calling convention works
{% highlight c %}

#include <stdlib.h>     // calloc

void Test1(int one, double two, int* three, double* four)
{
    *three = one + 1;
    *four = two + one;
}

int main(int argc, _TCHAR* argv[])
{
    int one = 1;
    double two = 2.0;

    int* three = (int*)calloc(1, sizeof(int));
    *three = 3;
    double* four = (double*)calloc(1, sizeof(double));
    *four = 4; 

    printf("one: %x two: %x three %x *three: %x four: %x *four: %x\n", (unsigned long)one,
        (unsigned long)two, (unsigned long)three, *three, (unsigned long)four, *four);

    Test1(one, two, three, four);

    printf("one: %x two: %x three %x *three: %x four: %x *four: %x\n", (unsigned long)one,
        (unsigned long)two, (unsigned long)three, *three, (unsigned long)four, *four);

    free(three);
    free(four);

    return 0;
}

{% endhighlight %} 

By itself a pretty useless program, but it will tell me how values and pointers are passed around
and set up.

I changed the build configuration to x64, Debug. What follows is the disassembly for
the Test1

{% highlight c %}
void Test1(int one, double two, int* three, double* four)
     8: {
000000013F321020 4C 89 4C 24 20       mov         qword ptr [rsp+20h],r9        // double* four
000000013F321025 4C 89 44 24 18       mov         qword ptr [rsp+18h],r8        // int* three
000000013F32102A F2 0F 11 4C 24 10    movsd       mmword ptr [rsp+10h],xmm1     // double two
000000013F321030 89 4C 24 08          mov         dword ptr [rsp+8],ecx         // int one
000000013F321034 57                   push        rdi                           // non-optimized code artifact
     9:     *three = one + 1;
000000013F321035 8B 44 24 10          mov         eax,dword ptr [rsp+10h]       // load up one
000000013F321039 FF C0                inc         eax                           // and add 1 to it
000000013F32103B 48 8B 4C 24 20       mov         rcx,qword ptr [rsp+20h]       
000000013F321040 89 01                mov         dword ptr [rcx],eax           // move that value to three
    10:     *four = two + one;
000000013F321042 F2 0F 2A 44 24 10    cvtsi2sd    xmm0,dword ptr [rsp+10h]      // convert int to double. 
                                                                                // Again, this is not optimized
000000013F321048 F2 0F 10 4C 24 18    movsd       xmm1,mmword ptr [rsp+18h]     // move one
000000013F32104E F2 0F 58 C8          addsd       xmm1,xmm0                     // add them both
000000013F321052 0F 28 C1             movaps      xmm0,xmm1                     // move result to xmm0
000000013F321055 48 8B 44 24 28       mov         rax,qword ptr [rsp+28h]       // set the result to *four          
000000013F32105A F2 0F 11 00          movsd       mmword ptr [rax],xmm0  
    11: }
000000013F32105E 5F                   pop         rdi                           // non-optimized code artifact
000000013F32105F C3  
{% endhighlight %} 

The pointers are passed around via *r9* and *r8*. The 32-bit int was passed in 
via *ecx* and the double via *XMM1*
The use of `movsd` was new for me and looked into how those ops work.

Let's take a look at what the optimized looks like. I have to modify the code a little
to fight against inlining by modifying the function prototype a bit by specifying `noinline`.

{% highlight c %}
_declspec(noinline) void Test1(int one, double two, int* three, double* four) 
{% endhighlight %} 

Here is what you get 

{% highlight c %}

     9:     *three = one + 1;
    10:     *four = two + one;
000000013FEC1000 F2 0F 58 0D F8 11 00 00 addsd       xmm1,mmword ptr [3FEC2200h]  
000000013FEC1008 41 C7 00 02 00 00 00 mov         dword ptr [r8],2  
000000013FEC100F F2 41 0F 11 09       movsd       mmword ptr [r9],xmm1  
    11: }
000000013FEC1014 C3                   ret  

{% endhighlight %}

Small, compact and to the point, which is what you would expect from a optimizing compiler.
This is the result of whole program optimization. 

If I did want to generate some assembly code for use via C#, I would have to use a modified version of
the non-optimized code for it.

Maybe I should look at something similar to my original problem, which
is manipulating arrays of data.

How about something like this

{% highlight c %}
_declspec(noinline) double Sum(double* input, size_t size)
{
    double temp = 0;
    for (int i = 0; i < size; i++)
    {
        temp += input[i];
    }
    return temp;
}
{% endhighlight %}

[1]:https://msdn.microsoft.com/en-us/library/7kcdt6fy.aspx 
