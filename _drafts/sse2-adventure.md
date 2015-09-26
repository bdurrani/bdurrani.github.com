---
layout: post
title: Adventures in optimizations
comments: false
---

I started out trying to figure out how sqeeze out some more speed for some SNR calculations, which involved the following bit of math.

<div>
$$SumOfSquares = \sum_{i=0}^{N} data_i^2$$
</div>

The language of choice was C# with Visual Studio 2013 for a 64-bit process for a 64-bit process.

One idea was to try and use SSE to improve the throughput of the computation. 
But the .NET framework currently (as of .NET 4.2) does not generate SSE instructions.
One way to do this would be to make a dll with the math function exported, and call that from C# using PInvoke.
The Visual Studio compiler (2012 and above) has the ability to auto-vectorize loops.
But this means that I have to add a C dll as a dependency of my C# application. 
One more file to keep track just for a single calculation.

But what if I could take the assembly code for that bit of code compiled using C and embed that directly into my C# application?

This post is how I went about investigating this.

This meant that I would need to actually write optimized some x64 assembly.
I could just start with writing up some C code and look at what the compiler generates.

### Learning assembly with a compilers help

64-bit assembly is a wide topic by itself. I had some a bit of x86 assembly back in my
university days, but it's been a while.
There is only one calling convention for 64-bit code, which is how **fastcall** was with x86 assembly. 
You can read more about it [here][1], but the jist of it is that **fastcall** uses registers as the first four arguments, and the stack frame for the rest for the functions parameters.
Ofcourse, not of this matters if the compiler inlines your function.

So I started writing bits of sample code and looking at the generated assembly.
All the samples I show were tested using Visual Studio 2013.

Here is a simple example, just to see how the calling convention works. `Test1()` takes a int, a double, and pointers of each kind.

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

By itself a pretty useless program, but it will tell me how values and pointers are passed around and set up.

I build a x64 Debug verson. 
And here is what you get.

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

Let's take a look at what the optimized looks like.
I have to modify the code a little to fight against inlining by modifying the function prototype a bit by specifying `noinline`.

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
the non-optimized code for it to avoid making use of hard-coded memory references for storing data
and instead, loading up data from registers.

Maybe I should look at something similar to my original problem, which
is manipulating arrays of data.

How about something like this

{% highlight c %}
_declspec(noinline) double SumOfSquares(int* input, size_t size)
{
    double temp = 0;
    for (int i = 0; i < size; i++)
    {
        temp += (input[i] * input[i]);
    }
    return temp;
}
{% endhighlight %}

Here is a debug output.

{% highlight c %}
 _declspec(noinline) double SumOfSquares(int* input, size_t size)
    14: {
000000013FEB3410 48 89 54 24 10       mov         qword ptr [temp],rdx      // saving registers on the
000000013FEB3415 48 89 4C 24 08       mov         qword ptr [rsp+8],rcx     // stack for debugging easy
000000013FEB341A 57                   push        rdi                       // see [this article][2]
000000013FEB341B 48 83 EC 20          sub         rsp,20h                   // space for local variables 
000000013F22341F 48 8B FC             mov         rdi,rsp  
000000013F223422 B9 08 00 00 00       mov         ecx,8  
000000013F223427 B8 CC CC CC CC       mov         eax,0CCCCCCCCh  
000000013F22342C F3 AB                rep stos    dword ptr [rdi]           // clear the new stack home space 
000000013F22342E 48 8B 4C 24 30       mov         rcx,qword ptr [rsp+30h]   // input
    15:     double temp = 0;
000000013F223433 0F 57 C0             xorps       xmm0,xmm0                 // clear 
000000013F223436 F2 0F 11 44 24 10    movsd       mmword ptr [rsp+10h],xmm0 // temp = 0 
    16:     for (int i = 0; i < size; i++)
000000013F22343C C7 44 24 18 00 00 00 00 mov         dword ptr [rsp+18h],0  // i= 0
000000013F223444 EB 0A                jmp         000000013F223450  
000000013F223446 8B 44 24 18          mov         eax,dword ptr [rsp+18h]  
000000013F22344A FF C0                inc         eax  
000000013F22344C 89 44 24 18          mov         dword ptr [rsp+18h],eax  
000000013F223450 48 63 44 24 18       movsxd      rax,dword ptr [rsp+18h]   // rax = i 
000000013F223455 48 3B 44 24 38       cmp         rax,qword ptr [rsp+38h]   // cmp i with size 
000000013F22345A 73 35                jae         000000013F223491          // jmp if i >= size 
    17:     {
    18:         temp += (input[i] * input[i]);
000000013F22345C 48 63 44 24 18       movsxd      rax,dword ptr [rsp+18h]   // rax = i 
000000013F223461 48 63 4C 24 18       movsxd      rcx,dword ptr [rsp+18h]   // rcx = i 
000000013F223466 48 8B 54 24 30       mov         rdx,qword ptr [rsp+30h]   // rdx = input 
000000013F22346B 4C 8B 44 24 30       mov         r8,qword ptr [rsp+30h]    // r8 = input 
000000013F223470 8B 04 82             mov         eax,dword ptr [rdx+rax*4] // input[i] 
000000013F223473 41 0F AF 04 88       imul        eax,dword ptr [r8+rcx*4]  // res =input[i] * input[i] 
000000013F223478 F2 0F 2A C0          cvtsi2sd    xmm0,eax                  // xmm0 = (double)res 
000000013F22347C F2 0F 10 4C 24 10    movsd       xmm1,mmword ptr [rsp+10h] // xmm1 = temp 
000000013F223482 F2 0F 58 C8          addsd       xmm1,xmm0                 // xmm += res 
000000013F223486 0F 28 C1             movaps      xmm0,xmm1  
000000013F223489 F2 0F 11 44 24 10    movsd       mmword ptr [rsp+10h],xmm0 // save res to temp
    19:     }
000000013F22348F EB B5                jmp         000000013F223446          // repeat the loop 
    20:     return temp;
000000013F223491 F2 0F 10 44 24 10    movsd       xmm0,mmword ptr [rsp+10h] // mov temp to return value 
    21: }
000000013F223497 48 83 C4 20          add         rsp,20h  
000000013F22349B 5F                   pop         rdi  
000000013F22349C C3                   ret  
{% endhighlight %}

There are several things in this code that would make our lives easier if we ever had to debug this code.
For example, copies of the function parameters are saved in the stack.
[Guard bytes][3] (0xCCCCCCCC) are used when initializing the extra stack space for temporary variables on the stack.  

Let's repeat the process now, but for the release version. 
I had to make some changes in the function to prevent it from being inlined.
The release version is build with /Ox (Full optimization) turned on. 
And since the purpose of this exercise is to speed things up, I want to make sure my loop gets vectorized. 
So I turned on the [vectorization report][4] to verify this, and also used switched the floating model to [/fp:fast][5] 

{% highlight c %}
_declspec(noinline) double SumOfSquares(int* input, size_t size)
    14: {
000000013F881000 48 89 5C 24 08       mov         qword ptr [rsp+8],rbx    // save size
000000013F881005 57                   push        rdi                      // volatile register 
000000013F881006 48 83 EC 20          sub         rsp,20h                  // make space for temp vars 
000000013F88100A 48 8B DA             mov         rbx,rdx  
000000013F88100D 48 8B F9             mov         rdi,rcx  
    15:     // add a rand() call here to prevent inline
    16:     double temp = rand();
000000013F881010 FF 15 0A 11 00 00    call        qword ptr [3F882120h]  
    17:     temp = 0;
    18:     for (int i = 0; i < size; i++)
000000013F881016 33 D2                xor         edx,edx               // clear the result of rand()
000000013F881018 0F 57 D2             xorps       xmm2,xmm2             // clear 
000000013F88101B 44 8B C2             mov         r8d,edx  
000000013F88101E 48 83 FB 04          cmp         rbx,4                     // cmp size and 4 
000000013F881022 72 62                jb          000000013F881086          // jmp if size < 4 
000000013F881024 83 3D F1 1F 00 00 02 cmp         dword ptr [__isa_available],2   // checks to see [if SSE4.2 is supported][6] 
000000013F88102B 7C 59                jl          000000013F881086          // jmp if not supported, which is true in
                                                                            // my case 
000000013F88102D 48 8B C3             mov         rax,rbx  
000000013F881030 48 8B CB             mov         rcx,rbx  
000000013F881033 0F 28 CA             movaps      xmm1,xmm2  
000000013F881036 83 E0 03             and         eax,3  
000000013F881039 44 8D 4A 02          lea         r9d,[rdx+2]  
000000013F88103D 48 2B C8             sub         rcx,rax  
    19:     {
    20:         temp += (input[i] * input[i]);
000000013F881040 F3 0F 7E 04 97       movq        xmm0,mmword ptr [rdi+rdx*4]  
000000013F881045 49 63 C1             movsxd      rax,r9d  
000000013F881048 41 83 C0 04          add         r8d,4  
000000013F88104C 49 63 D0             movsxd      rdx,r8d  
000000013F88104F 41 83 C1 04          add         r9d,4  
000000013F881053 66 0F 38 40 C0       pmulld      xmm0,xmm0  
000000013F881058 F3 0F E6 C0          cvtdq2pd    xmm0,xmm0  
000000013F88105C 66 0F 58 D0          addpd       xmm2,xmm0  
000000013F881060 F3 0F 7E 04 87       movq        xmm0,mmword ptr [rdi+rax*4]  
000000013F881065 66 0F 38 40 C0       pmulld      xmm0,xmm0  
000000013F88106A F3 0F E6 C0          cvtdq2pd    xmm0,xmm0  
000000013F88106E 66 0F 58 C8          addpd       xmm1,xmm0  
000000013F881072 48 3B D1             cmp         rdx,rcx  
000000013F881075 72 C9                jb          000000013F881040  
    17:     temp = 0;
    18:     for (int i = 0; i < size; i++)
000000013F881077 66 0F 58 D1          addpd       xmm2,xmm1  
000000013F88107B 0F 28 C2             movaps      xmm0,xmm2  
000000013F88107E 66 0F 15 C2          unpckhpd    xmm0,xmm2  
000000013F881082 F2 0F 58 D0          addsd       xmm2,xmm0  
000000013F881086 49 63 C0             movsxd      rax,r8d                   // jmp here if SSE4.2
                                                                            // is not supported 
                                                                            // r8d = i, rax = i
000000013F881089 48 3B C3             cmp         rax,rbx                   // compare size and i 
000000013F88108C 73 32                jae         000000013F8810C0          // jmp if above 
000000013F88108E 48 8D 0C 87          lea         rcx,[rdi+rax*4]           // rcx = input[rax] 
000000013F881092 0F 1F 40 00          nop         dword ptr [rax]           // not sure about the 
    17:     temp = 0;                                                       // purpose of all
    18:     for (int i = 0; i < size; i++)                                  // these nops
000000013F881096 66 66 0F 1F 84 00 00 00 00 00 nop         word ptr [rax+rax+00000000h]  
000000013F8810A0 8B 01                mov         eax,dword ptr [rcx]       // eax = input[i]
000000013F8810A2 41 FF C0             inc         r8d                       // i++ 
000000013F8810A5 48 83 C1 04          add         rcx,4  
    19:     {
    20:         temp += (input[i] * input[i]);
000000013F8810A9 0F AF C0             imul        eax,eax                   // signed int multiply
000000013F8810AC 66 0F 6E C8          movd        xmm1,eax                  // cache result 
000000013F8810B0 49 63 C0             movsxd      rax,r8d                   // rax = current i 
000000013F8810B3 F3 0F E6 C9          cvtdq2pd    xmm1,xmm1                 // convert to double 
000000013F8810B7 F2 0F 58 D1          addsd       xmm2,xmm1                 // save to temp 
000000013F8810BB 48 3B C3             cmp         rax,rbx  
000000013F8810BE 72 E0                jb          000000013F8810A0  
    21:     }
    22:     return temp;
000000013F8810C0 0F 28 C2             movaps      xmm0,xmm2                 // save temp result 
    23: }
000000013F8810C3 48 8B 5C 24 30       mov         rbx,qword ptr [rsp+30h]   // rbx = input 
000000013F8810C8 48 83 C4 20          add         rsp,20h                   // clear up stack 
000000013F8810CC 5F                   pop         rdi  
000000013F8810CD C3                   ret  
{% endhighlight %}

My current CPU does not support SSE4.2 so the code path skips all that goodness.
If I have made `temp` an `int` instead of a `double`, the compiler would have generated 
the [SSE2 integer arithmetic][7] instruction set.
I actually did confirm this, but let's continue with what we have so far

### Generate assembly byte code

So I have the assembly code I need to calculate sum of squares. 
Let's see what that looks like

{% highlight nasm %}
# compile using gcc
.intel_syntax noprefix
.data
.text
.global main
main:
# double sum(int* input, size_t size)
#	rcx = input
#	rdx = size
push rdi
mov rbx, rdx
mov rdi, rcx
xor edx, edx
xorps xmm2, xmm2	
mov r8d, edx
movsxd rax, r8d
lea rcx, [rdi+rax*4]
@calcbegin:
mov eax, dword ptr [rcx]
inc r8d
add rcx, 4
imul eax, eax
movd xmm1, eax
movsxd rax, r8d
cvtdq2pd xmm1, xmm1
addsd xmm2, xmm1
cmp rax, rbx
jb @calcbegin
movaps xmm0, xmm2
pop rdi
ret    
{% endhighlight %}

Looks like this should work. There is no SSE here, but it should work
Now all I need is to assemble this, and generate the byte code. I used gcc for this.
I generate the .o file, and dump it's disassembly into a text file

{% highlight bash %}
gcc -g -c simplelooptest.s -m64 && objdump -d -M intel simplelooptest.o > test.out
{% endhighlight %}

Manually parsing the dissassembly for the byte code is a pain, so I wrote up
a quick F# script that does that for me.
You can see the script [here][8].
Just give it the full path to `test.out` and it prints out the byte array to the debugger.

How do you actually use the byte code in C#?
You need to allocate executable memory space using `VirtualAlloc` and copy the byte array there.
Then use `Marshal.GetDelegateForFunctionPointer()` to actually execute the instructions with the
correct method signature.

You can see that below

{% highlight csharp %}
using System;
using System.Diagnostics;
using System.Runtime.InteropServices;

namespace ConsoleTest
{
   internal class Program
   {
      #region Native stuff

      [Flags]
      public enum AllocationType : uint
      {
         COMMIT = 0x1000,
         RESERVE = 0x2000,
         RESET = 0x80000,
         LARGE_PAGES = 0x20000000,
         PHYSICAL = 0x400000,
         TOP_DOWN = 0x100000,
         WRITE_WATCH = 0x200000
      }

      [Flags]
      public enum MemoryProtection : uint
      {
         EXECUTE = 0x10,
         EXECUTE_READ = 0x20,
         EXECUTE_READWRITE = 0x40,
         EXECUTE_WRITECOPY = 0x80,
         NOACCESS = 0x01,
         READONLY = 0x02,
         READWRITE = 0x04,
         WRITECOPY = 0x08,
         GUARD_Modifierflag = 0x100,
         NOCACHE_Modifierflag = 0x200,
         WRITECOMBINE_Modifierflag = 0x400
      }

      public enum MemoryFreeType : uint
      {
         MEM_DECOMMIT = 0x4000,
         MEM_RELEASE = 0x8000
      }

      [DllImport("kernel32.dll", SetLastError = true)]
      private static extern IntPtr VirtualAlloc(IntPtr lpAddress, UIntPtr dwSize,
         AllocationType flAllocationType, MemoryProtection flProtect);

      [DllImport("kernel32.dll", SetLastError = true)]
      private static extern bool VirtualProtect(IntPtr lpAddress, uint dwSize,
         MemoryProtection flNewProtect, out MemoryProtection lpflOldProtect);

      [DllImport("kernel32.dll", SetLastError = true)]
      private static extern bool VirtualFree(IntPtr lpAddress, UIntPtr dwSize,
         MemoryFreeType dwFreeType);

      #endregion

      private delegate double SumOfSquaresDelegate(int[] input, int size);

      /// <summary>
      /// Create the buffer for executable memory and return a ptr to it
      /// </summary>
      /// <param name="data">The byte code</param>
      /// <returns>The ptr to executable memory</returns>
      private static IntPtr SetupBuffer(byte[] data)
      {
         var codeBuffer = VirtualAlloc(IntPtr.Zero, new UIntPtr((uint)data.Length),
            AllocationType.COMMIT | AllocationType.RESERVE,
            MemoryProtection.READWRITE);

         Marshal.Copy(data, 0, codeBuffer, data.Length);
         MemoryProtection oldProtection;
         VirtualProtect(codeBuffer, (uint)data.Length, MemoryProtection.EXECUTE_READ, out oldProtection);
         return codeBuffer;
      }

      /// <summary>
      /// Free memory allocated by VirtualAlloc
      /// </summary>
      /// <param name="ptr">ptr to VirtualAlloc memory</param>
      private static void FreeBuffer(IntPtr ptr)
      {
         VirtualFree(ptr, UIntPtr.Zero, MemoryFreeType.MEM_RELEASE);
      }

      private static void Main(string[] args)
      {
         BenchmarkSumOfSquares();
         Console.ReadLine();
         return;
      }

      public static void BenchmarkSumOfSquares()
      {
         byte[] data = { 0x57, 0x48, 0x89, 0xd3, 0x48, 0x89, 0xcf, 0x31, 0xd2, 0x0f, 0x57, 0xd2, 0x41, 0x89, 0xd0, 0x49, 0x63, 0xc0, 0x48, 0x8d, 0x0c, 0x87, 0x8b, 0x01, 0x41, 0xff, 0xc0, 0x48, 0x83, 0xc1, 0x04, 0x0f, 0xaf, 0xc0, 0x66, 0x0f, 0x6e, 0xc8, 0x49, 0x63, 0xc0, 0xf3, 0x0f, 0xe6, 0xc9, 0xf2, 0x0f, 0x58, 0xd1, 0x48, 0x39, 0xd8, 0x72, 0xe0, 0x0f, 0x28, 0xc2, 0x5f, 0xc3, 0x90, 0x90, 0x90, 0x90, 0x90, };

         var buffer = SetupBuffer(data);
         var sumSquaresDelegate = (SumOfSquaresDelegate)Marshal.GetDelegateForFunctionPointer(buffer, typeof(SumOfSquaresDelegate));
         const int size = 50000;
         int[] input = new int[size];
         var rand = new Random(13);
         for (int i = 0; i < size; i++)
         {
            input[i] = rand.Next(10);
         }

         double res = 0, managedRes = 0;

         Stopwatch sw = new Stopwatch();
         sw.Start();
         for (int i = 0; i < 10000; i++)
         {
            res = sumSquaresDelegate(input, size);
         }
         sw.Stop();
         Console.WriteLine("assembly time: {0}", sw.Elapsed.TotalMilliseconds);

         sw.Restart();
         for (int i = 0; i < 10000; i++)
         {
            managedRes = SumOfSquaresManaged(input);
         }
         sw.Stop();
         Console.WriteLine("managed time: {0}", sw.Elapsed.TotalMilliseconds);

         FreeBuffer(buffer);
         Console.WriteLine("sum of squares managed: {0}", managedRes);
         Console.WriteLine("sum of squares assembly: {0}", res);
      }

      /// <summary>
      /// C# version of sum of squares
      /// </summary>
      /// <param name="input">input data</param>
      /// <returns>sum of squares</returns> 
      public static double SumOfSquaresManaged(int[] input)
      {
         double temp = 0;
         foreach (var item in input)
         {
            temp += (item * item);
         }
         return temp;
      }
   }
}
{% endhighlight %}

This is something I wrote to benchmark the code comparing it to the C# equivalent implementation.
It runs the two versions of the sum of squares a bunch of times and calculates the amount of time each run took.

Here is the output of the console app for a release build.

{% highlight bash %}
assembly time: 1349.824
managed time: 3829.5137
sum of squares managed: 1432310
sum of squares assembly: 1432310
{% endhighlight %}
 
Looks like the assembly version is about 3X faster than the plain C# version.
I didn't expect there to be such a huge improvement.

Next thing I'll try is to use SSE2 intrinsics and embed the byte code for that. 
But that's for another post.


[1]:https://msdn.microsoft.com/en-us/library/7kcdt6fy.aspx 
[2]:http://blogs.msdn.com/b/ntdebugging/archive/2009/01/09/challenges-of-debugging-optimized-x64-code.aspx
[3]:https://en.wikipedia.org/wiki/Magic_number_(programming)#Magic_debug_values
[4]:https://msdn.microsoft.com/en-us/library/jj614596.aspx 
[5]:https://msdn.microsoft.com/en-us/library/e7s85ffb.aspx 
[6]:https://stackoverflow.com/questions/9334548/get-sse-version-without-asm-on-x64
[7]:https://msdn.microsoft.com/en-us/library/vstudio/k87x524b(v=vs.100).aspx 
[8]:https://gist.github.com/bdurrani/2ccb6e89a6156c1dac06 
