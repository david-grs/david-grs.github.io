---
layout: post
title:  "Heap fragmentation or how my micro-benchmark went wrong"
author: David Gross
date:   2016-12-28 16:56
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
This gives us the output: `100000 iterations of unordered_set<int>::emplace() took 565ms`. Awesome. This is exactly what we wanted to know.

A few hours later, we added a lot of other small functions like this one. We are now measuring the time of the *emplace* operation for different containers,
with different contained type &mdash; to change their size and the cost of the move, copy, construction and destruction operations &mdash; and with different
number of elements in the container. We even added a layer of template because it looks better!

Our *benchmark()* function is now something like this:
{% gist david-grs/38b359aa9f20d7d27444c16c9e411855 even_more_awesome_benchmark.cc %} 
<br />


The tragedy
-----------
*At some point*, you decide to re-order the function calls &mdash; just for convenience. And boom. You didn't 
expect *anything* to change because you have been at staring these numbers for hours... but your initial benchmark
is now *twice slower*: 

`100000 iterations of unordered_set<int>::emplace() took 1143ms`

How come? You run again and get the same results. You checkout the previous commit and test again: it is fixed. How can a simple re-ordering of calls 
slow down a 5-lines function?

You decide to run the benchmark twice, as first benchmark and last one:

{% gist david-grs/38b359aa9f20d7d27444c16c9e411855 benchmark_cmp.cc %} 
<br />
... which outputs:

    first run:   100000 iterations of unordered_set<int>::emplace() took 565ms
    second run: 100000 iterations of unordered_set<int>::emplace() took 1143ms



Heap fragmentation
------------------
As you guessed because it is the title of this article, the difference is due to the heap.

First of all, let's confirm our hypothesis by actually measuring the time spent during the heap operations &mdash; *malloc*, *free* and *realloc*.

I updated for that an old [C++ wrapper around the malloc hooks](https://github.com/david-grs/mtrace) that I developed a while ago to follow each memory allocation 
&mdash; by printing them. Here we don't want to print them (there are millions!), but measure the time spent in these functions:

{% gist david-grs/38b359aa9f20d7d27444c16c9e411855 benchmark_mtrace.cc %} 
<br />
And this confirms what we thought:

    first run: benchmark_unordered_set_emplace() took 565ms
    time spent in malloc(): 231ms
    time spent in free(): 1ms
    time spent in realloc(): 0ms

    second run: benchmark_unordered_set_emplace() took 1143ms
    time spent in malloc(): 916ms
    time spent in free(): 94ms
    time spent in realloc(): 0ms

<br />
90% of the CPU time is spent in *malloc* and *free* on the second run of the benchmark! To go even more in depth, the GNU extension offers *malloc_info*. This 
function returns the list of bins &mdash; chunks that have been freed &mdash; by category (size, unsorted, fast). Let's insert one call to *malloc_info* before 
each benchmark run:


    before the first run:
    <sizes>
    </sizes>
    <total type="fast" count="0" size="0"/>
    <total type="rest" count="0" size="0"/>

    before the second run:
    <sizes>
    <size from="33" to="48" total="383999952" count="7999999"/>
    <size from="49" to="64" total="64" count="1"/>
    <size from="33" to="33" total="99" count="3"/>
    <unsorted from="145" to="1009" total="1156000346" count="3999994"/>
    </sizes>
    <total type="fast" count="8000000" size="384000016"/>
    <total type="rest" count="3999997" size="1156000445"/>

<br />
What is very particular in this micro-benchmark is that we have millions of successive memory allocations: small allocated chunks from *std::unordered_set&lt;int&gt;::emplace*,
big ones from *boost::container::flat_set*, etc. Then, the benchmark is done, the block ends and the several containers go out of scope and are *destroyed*.

This leads to a very specific heap state: 

 * Millions of chunks are successively destroyed
 * They were also allocated successively, so quite likely to be contiguous and in the same pages
 * These freed chunks represent almost all our memory allocations

All freed chunks are merged together, the bins linked lists are cleared, the memory released. And this takes *forever*.

Remember our code:
{% gist david-grs/38b359aa9f20d7d27444c16c9e411855 benchmark_cmp2.cc %} 
<br />
If we introduce one *malloc(1)* call before our second run of the benchmark and run *malloc_info* again, we now get a heap in a clean state, and the time difference 
is gone. However, this is not a clean or portable solution.

The unfortunate part, is that there is no nice and clean solution. All the possible solutions I can think of are very dependent to the *malloc* implementation, and are made on several assumptions. 
For example, you could be simply rearrange the code to not destroy any of the benchmarked containers before 
the end of all benchmarks. This means that you only allocate, but don't free any memory, which will prevent from the symptom we got earlier, but maybe not from another side effet. 

In our simple case, we could simply change the code by removing the blocks and leave the inlined function calls in the single *benchmark* function:
{% gist david-grs/38b359aa9f20d7d27444c16c9e411855 benchmark_cmp3.cc %} 
<br />
The only good solution &mdash; i.e. not based on any assumptions regarding the *malloc* implementation &mdash; would be to separate the different benchmarks between different applications
or application runs. 

That's all! I ran all my tests on my Ubuntu 14.04, based on the GLibC 2.19 (yes, this setup is getting old...). You can find a simplified version of the benchmark code 
[on my github](https://github.com/david-grs/heap_frag). Don't be surprised if you get different results, everything in this article is implementation dependant.

Enjoy your holidays, and your micro-benchmarks!

