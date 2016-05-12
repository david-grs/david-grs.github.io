---
layout: post
title:  "static_any: a low-latency stack-based Boost.Any"
author: David Gross
date:   2016-05-12 21:48
---

When writing some speed-critical code, I wanted to use *Boost.Any* as an attribute in a struct. This struct
was small &mdash; 2 cache lines &mdash; and I knew that all the types that I would store in this *Any*
would fit in 16bytes. 

> I thought "trivial, let's use Boost.Any with a stack-based allocator"!<br />
> ... and then I got disappointed.

Disappointed, because *Boost.Any* does not offer the possibility to use a custom allocator. To be honest, that would 
not have solved the story anyway, as *Boost.Any* does other things that slow down this container, 
but that was enough to start thinking about another &mdash; home-made &mdash; solution.

> Talk is cheap. Show me the code.<br />
> [static\_any project on my github](https://github.com/david-grs/static_any)


Boost.Any implementation
------------------------
So why was *Boost.Any* too slow in my case ? And why was I looking for a stack-based allocator ?

*Boost.Any* is very simple: it erases the type via inheritance, and the few required operations, which are querying the type
and cloning the objects, are done via virtual methods. Here is a skeleton:
{% gist david-grs/8b727b6d2e71361ffa730df4004b449c boost_any_impl %}
<br />
My main issue regarding this implementation was not at all with the costs of calling virtual methods, but with
the memory layout that this would imply to my structure.

I knew that my stored types were small, and I wanted them on the same **cache line**.

To be more concrete, let's take the following code: 
{% gist david-grs/8b727b6d2e71361ffa730df4004b449c mem_layout %}
<br />
The variables *i* and *l* are next to each other in memory, but the integer *1234* will be somewhere else &mdash; due to
the heap-based implementation of *Boost.Any* &mdash; and maybe even on another page.

And of course, you get all the benefits when having these variables located in the same contiguous chunk of memory: better
data-locality means less memory cache misses and a better prefetching.


The benchmark
-------------
As always when talking about speed, we want to see some numbers. So here you are &mdash; it is in nanoseconds, the lower the better:

![Time spent on assignment and get operations in nanoseconds](/assets/images/bench_any_boost_qvariant.png)

The reason *static\_any* is faster on assignment is mainly due to the fact that is does not do a memory allocation. On the *get* operation,
it is due to virtual calls and other implementation details.

But the main thing, the reason I wanted to have such generic container, i.e. a stack-based any, is not shown by these numbers. Because
when you benchmark a piece of code, you run this one in a loop, and you do not get any cache misses... These numbers are just for *raw
sppeed* and most of the time, instructions per cycle for such operations won't be your bottleneck, but memory can be one.


static\_any
-----------
The usage is as simple as *Boost.Any*:
{% gist david-grs/8b727b6d2e71361ffa730df4004b449c static_any_example %}
<br />

At the beginning, I only implemented it for trivially copyable types, as I was only using this container with such objects. This super simple 
and even faster container &mdash; but unsafe, as there is no type checking at runtime &mdash; is in the same header *any.hpp* under the name 
of *static\_any\_t*.

Later, after few discussions with [my workmate Maciek](https://github.com/maciekgajewski), he got the awesome idea about the *gateway function*
that allows *static\_any* to go from the erased type &mdash; the vector of bytes that is used as underlying in *static\_any* &mdash; to the real
type T that is stored.

I will describe that in a later post, meanwhile you can grab the code...

