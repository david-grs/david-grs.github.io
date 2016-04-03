---
layout: post
title:  "TLS performance overhead and cost on GNU/Linux"
author: David Gross
date:   2016-03-20 18:03:20
redirect_from: "/2016/03/20/tls_performance_overhead_cost_linux.html"
---


When changing the implementation of a time clock, I had to switch from a static method &mdash; think about a classic
call to _clock\_gettime()_ for example &mdash; to a per-thread model. Long story short, it was about a constant TSC that
was not so constant across different sockets. 

Per-thread and static variables: that sounds like a perfect case for Thread-Local Storage (TLS). However this time clock
was widely used in my project and was taking so far a couple of CPU cycles, and I wanted to be sure that the TLS would not add
a consequent overhead.

A quick look on Google only revealed an article from the Intel Developer Zone: [the hidden performance cost of accessing 
thread-local variables](https://software.intel.com/en-us/blogs/2011/05/02/the-hidden-performance-cost-of-accessing-thread-local-variables).
Among pro-tips like "you simply have to buy a Intel&copy;&reg; VTune&trade; licence and don't forget the Parallel Studio XE&copy;&reg;&trade; it
only costs 2k$", the awesome screenshots or the comment section where someone says that the code is slow due to the TLS 
implementation of Windows while another one points out the code was running under Linux, I simply decided to _figure out myself_.

So I did what I prefer: I profiled it.

> #TL;DR
> If you have **statically linked** code, you can use a thread local variable like another one: there is only one TLS block and its
> entire data is known at link-time, which allows the pre-calculation of all offsets.
> <p>&nbsp;</p>
> If you are using **dynamic modules**, there will be a lookup each time you access the thread local variable, but
> you have some options in order to avoid it in speed critical code.


#&micro;-benchmarking the access of TLS data
--------------------------------------------
I wrote a snippet that mimics the code I had in production: one was accessing a static variable, the other one a thread
local variable, through a static method of the class. 

{% gist david-grs/8f8f38b6b63216a97c5c inc.hpp %}
{% gist david-grs/8f8f38b6b63216a97c5c inc.cpp %}
<br />
Note that the _thread\_local_ keyword has been introduced in C++11, but is only accessible from GCC 4.8. Earlier, you still
have to use _\_\_thread_.

To benchmark the code, I am using [geiger](https://github.com/david-grs/geiger "Geiger, a micro benchmark library in C++"), a micro
benchmark library in C++ I developed. The main advantage of geiger is to read [hardware performance counters](https://perf.wiki.kernel.org/index.php/Main_Page "Profiling hardware counters").
I might write a post about it later. 

Here, I simply declare a suite of benchmark and add two tests, one that accesses the static data and the second one the thread local variable. I run twice the suite, the first time I only
do one iteration, and 100 iterations the second run ; I will come back to that later.

{% gist david-grs/8f8f38b6b63216a97c5c tls.cpp %}
<br />
Let's build it with our favorite flags:
{% gist david-grs/8f8f38b6b63216a97c5c CMakelists.txt %}
<br />
The results are quite straightforward: no overhead at all. There is always a bit of jitter with hardware counters, so do not 
guess we have more cycles with the TLS test, there is quite some variance here between several runs.
{% gist david-grs/8f8f38b6b63216a97c5c Output.txt %}
<br />
The assembly code also confirms it. I build with CLang 3.6 but I got the same with GCC 4.7:
{% gist david-grs/8f8f38b6b63216a97c5c Assembly.txt %}
<br />
__NB__: the glibc is using the segment register FS for accessing the TLS


#How about \_\_tls\_get\_addr ?
-------------------------------
Apparently, all is good, I can use TLS as I want. But hang on... how about this function that was in the callstack on the 
Intel blog, _\_\_tls\_get\_addr_ ?

> One thing that always scares me the most when I do micro benchmarks is about the reliability of their results. Indeed, it 
> is very easy to screw up. You have to generate some _realistic_ inputs that will trigger the same behavior that you got
> in production, without too many side effects. And from the D-cache to the BTB, everything is hot...

Here, we have a sneaky case: the compilation is part of the environment of our benchmark. If the thread variable is in a dynamic
module, the linker cannot compute a direct reference to it and we get the much vaunted lookup!

Let's build it again, as a shared library this time:
{% gist david-grs/8f8f38b6b63216a97c5c CMakelists_shared.txt %}
<br />
There is in this case a difference ; the method that accesses the thread local variable takes twice more time. Please not that 
this overhead is still very small, as in this case we compare it to an extremely cheap operation...
{% gist david-grs/8f8f38b6b63216a97c5c Output2.txt %}
<br />
The assembly code shows the same difference:
{% gist david-grs/8f8f38b6b63216a97c5c Assembly2.txt %}


#The TLS implementation in ELF
------------------------------
As global and static variables are stored in _.data_ and _.bss_ sections, thread local variables get stored in equivalent 
_.tdata_ and _.tbss_ sections.

Each thread has its on TCB &mdash; _Thread Control Block_ &mdash; that contains some metadata about the thread itself. One
metadata is the address of the DTV &mdash; Dynamic Thread Vector &mdash; which is a vector that contains pointers to 
thread local variables and other TLS blocks.

![TLS implementation](/assets/images/tls_implementation.jpg)

When using statically linked code, everything is simpler as there is __only one TLS block__. As the offsets to the thread local
variables are known at link-time, the compiler is able to generate a direct access to the thread local variable. 

But when you have dynamic modules, several TLS blocks are present. It is easier to see with the following simplified version 
of _\_\_tls\_get\_addr_. _offset_ is known at build time, then _module\_id_ is the only variable, which is zero when using static
linking.
{% gist david-grs/8f8f38b6b63216a97c5c tls_get_offset.c %}
<br />
This is the actual implementation from the glibc 2.20:
{% gist david-grs/8f8f38b6b63216a97c5c dl-tls.c %}
<br />
The ABI specific parts are implemented in sysdeps ; as for x86_64, we access the TCB from the segment register FS:
{% gist david-grs/8f8f38b6b63216a97c5c x86_64_tls.h %}

#So what?
---------
Accessing thread local variables is cheap in both cases ; the lookup done by the glibc is very fast (and takes a constant time). 
But if your code is &mdash; or could be &mdash; loaded as a dynamic module and is speed critical, then it can be worth spending
some time to __avoid this lookup__.

One easy win could be to access the thread local data from the same block of code. The compiler will be smart enough to call only
once _\_\_tls\_get\_addr_. This could be done by, for example, __inlining__ the function that access the TLS. 

One could think about _keeping the reference_ of the thread local variable, but I found this solution very dodgy. You might have to prepare your arguments to pass the code review :) ...

More information about the TLS and its implementation on GNU/Linux in [ELF Handling for TLS](https://www.akkadia.org/drepper/tls.pdf "ELF Handling for TLS from Ulrich Drepper") from Ulrich Drepper.


