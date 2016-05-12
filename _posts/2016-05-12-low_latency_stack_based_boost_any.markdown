---
layout: post
title:  "static\_any: a low-latency stack-based Boost.Any"
author: David Gross
date:   2016-05-12 21:48
---


> #Talk is cheap. Show me the code.
> [static\_any project on my github](https://github.com/david-grs/static_any)


When writing some speed-critical code, I wanted to use *Boost.Any* as an attribute in a struct. This struct
was small &mdash; 2 cache lines &mdash; and I knew that all the types that I would store in this *Any*
would fit in 16bytes. 

> I thought "trivial, let's use Boost.Any with a stack-based allocator"!
> ... and then I got disappointed.

Disappointed, because *Boost.Any* does not offer the possibility to use a custom allocator. To be honest, that would 
not have solved the story anyway, as *Boost.Any* does other things that slow down this container, 
but that was enough to start thinking about a another &mdash; home-made &mdash; solution.


Boost.Any implementation
------------------------
So why was *Boost.Any* too slow in my case ? And why was I looking for a stack-based allocator ?

*Boost.Any* is very simple: it erases the type via inheritance, and the few required operations, which are querying the type
and cloning the objects, are done via virtual methods. Here is a skeleton:
{% gist david-grs/8f8f38b6b63216a97c5c boost_any_impl %}
<br />
My main issue regarding this implementation was not at all with the costs of calling virtual methods, but with
the memory layout that this would imply to my structure.

I knew that my stored types were small, and I wanted them on the same **cache line**.


static\_any
-----------
This is why I started developing static\_any. At the very beginning, I only implemented it for trivially copyable types, 
as I was only using the container with this kind of objects. This super simple and fast container is in the same header 
*any.hpp* under the name of static\_any\_t. 


