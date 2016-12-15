---
layout: post
title:  "Heap fragmentation or how my micro-benchmark went wrong"
author: David Gross
date:   2016-12-15 22:46
---

Micro-benchmarking code always looks simple: a few variables, a small *for* loop and two *std::chrono* calls. I think this simplicity is an illusion.
Micro-benchmarking is either complicated or inaccurate. 

You have to be careful that the compiler does not optimize out the code you are timing, accurately measure the time and know your time resolution,
all the caches are usually hot (as you loop over the same operation) and you will get a totally different cache hit rate than in production, a "production-like" set of inputs 
has to be generated &mdash; without affecting the resuts of the benchmark of course! &mdash;, on each iteration of the benchmark 
the state of your object is changing but should still allow the benchmark to continue, etc. It is really hard.

And this is why I usually don't micro-benchmark: it is too hard for what it is ; you will get numbers and finally be disappointed or puzzled 
because you will not get the same latencies in production.

I prefer measuring the impact of my code changes directly in production &mdash; or in a production-like environment. The initial cost of setting up such timing 
system might be higher, but on the long term you get a lot of benefits from it. Lots of code changes can simply not be micro-benchmarked at all. In his 
talk [*Optimization Tips - Mo' Hustle Mo' Problems*](https://www.youtube.com/watch?v=Qq_WaiwzOtI), Andrei Alexandrescu is presenting the speed gain of 
not inlining the destructor of an object &mdash; by simply defining an empty destructor in the source file instead of the default one. Good luck 
to micro-benchmark that, and if you actually do it *somehow*, I guess you will get the opposite result, as having the *inlined* call will save some instructions
and the instruction cache won't be polluted.

This post is about my last adventure with a micro-benchmark. 



The simplest &micro;-benchmark
------------------------------
This is a very simple &mdash; and the most boring I could find &mdash; micro-benchmark:
{% gist david-grs/38b359aa9f20d7d27444c16c9e411855 awesome_benchmark.cc %} 
<br />
This gives us the output: `100000 iterations of umap<int, int>::emplace() took 565ms`. Awesome. This is exactly what we wanted to know.

A few hours later, we added a lot of other small functions like this one. We are now measuring the time of the *emplace* operation for different containers,
with different contained type &mdash; to change their size and the cost of the move, copy, construction and destruction operations &mdash; and with different
number of elements in the container. We even added a layer of template because it looks better!

Our *main()* is now something like this:
{% gist david-grs/38b359aa9f20d7d27444c16c9e411855 even_more_awesome_benchmark.cc %} 
<br />


The tragedy
-----------
*At some point*, you decide to re-order the function calls. You don't know why &mdash; just for convenience, because it is nicer like that. And boom. You didn't 
expect *anything* to change because you were staring these numbers for hours... but you now get: 

`100000 iterations of umap<int, int>::emplace() took 1143ms`

How come? You run again and get the same results. Nothing changed. You checkout the previous commit and test again: it is fixed. How can a simple re-ordering of function
calls change by more than a factor two the time of this *for* statement?

You decide to run the benchmark twice, as first benchmark and last one:

{% gist david-grs/38b359aa9f20d7d27444c16c9e411855 benchmark_cmp.cc %} 
<br />
and you still get:

    first run:   100000 iterations of umap<int, int>::emplace() took 565ms
    second run: 100000 iterations of umap<int, int>::emplace() took 1143ms
<br />



Hardware counters are your friends 
----------------------------------
In this kind of scenario, you have a quite limited set of options.

First of all, you might think about looking at the disassembly. I know that we all love the [GCC explorer](http://gcc.godbolt.org), but when you are benchmarking 
containers, the assembly code will be very long. Also, reodering function calls is not a small change, so the entire assembly code will have changed and it will hard to compare that by hand.

The most powerful tool you could use in this case if your intuition does not guide you towards the light is a feature offered by your CPU, called hardware counters. *perf* is the simplest way 
to use them, but in this case you would like to only get the counters on a small piece of C++ code, not the entire execution of your program.

In order to use them in C/C++, the reference is the [libpapi](http://icl.cs.utk.edu/papi/). As it is a C library, I wrote a while ago a short C++ header file to make the basic usage of 
the library easier: [libpapipp](https://github.com/david-grs/papipp). Let's use it here to measure the number of instructions and cycles, to give us a bit more an idea on what is going on.

{% gist david-grs/38b359aa9f20d7d27444c16c9e411855 benchmark_papipp.cc %} 
<br />
This allows us to see that the number of instructions executed is actually 25% higher when we reorder the functions. This is a good indication, as the code is identical, we know that actually
*more* code is executed



Heap fragmentation
------------------
As you guessed because it is the title of this article, the difference comes from the heap. At the beginning of the micro-benchmark, the heap is in a *clean* state, while after a lot of 
insertions and deletions in some containers, it is heavily fragmented, and one malloc() call might take much more time than expected. 

Let's confirm that by measuring the time we spend in *malloc()* and *free()*. I updated for that an old [C++ wrapper around the malloc hooks](https://github.com/david-grs/mtrace) that I developed 
once to print the memory allocations. Here we don't want to print them (there are millions!), but measure the time spent in them:

{% gist david-grs/38b359aa9f20d7d27444c16c9e411855 benchmark_mtrace.cc %} 
<br />
... which gives us immediately the answer:
    
    first run:
    benchmark_unordered_map_emplace() took 565ms
    time spent in malloc(): 231ms
    time spent in free(): 1ms
    time spent in realloc(): 0ms

    second run:
    benchmark_unordered_map_emplace() took 1143ms
    time spent in malloc(): 916ms
    time spent in free(): 94ms
    time spent in realloc(): 0ms

We were spending almost 90% of the CPU time in *malloc()* and *free()* the second run of the benchmark! For even more details, we can also call *malloc_info* before our second run:

    <malloc version="1">
    <heap nr="0">
    <sizes>
    <size from="33" to="48" total="383999952" count="7999999"/>
    <size from="49" to="64" total="64" count="1"/>
    <size from="33" to="33" total="99" count="3"/>
    <unsorted from="145" to="1009" total="1156000346" count="3999994"/>
    </sizes>
    <total type="fast" count="8000000" size="384000016"/>
    <total type="rest" count="3999997" size="1156000445"/>


Micro-benchmarking is hard. Be very careful while doing it ; this kind of mistakes or wrong measurements can happen, even if you are using Google Benchmark ;)


