---
layout: post
title:  "Why you should use Boost.MultiIndex (Part II)"
author: David Gross
date:   2016-11-23 22:10
---

A few weeks ago, I posted the [first part](/why_boost_multi_index_container-part1/) of this article, where I explained the advantages 
of Boost.MultiIndex over the standard containers when you need to have multiple views on a set of data.

In this second part, I would like to talk about the benefits you can get from using Boost.MultiIndex as a single-index hash table, as a replacement
of *std::unordered_map*.

One interesting and powerful aspect of Boost.MultiIndex is that it allows you to add an index of type *T*, where *T* is different from the stored type. And 
it is more frequent and useful that you could think.


std::string as a key
--------------------
Who never used a *std::unordered_map<std::string, X>*? This is the most straightforward 
way to lookup for an object with a string: it is simple, relatively fast, and convenient as *std::hash&lt;std::string&gt;* is 
provided by the standard library.

However, what if you get another string type as input for your lookup? I already found myself in this situation multiple times:  

 * when reading from shared memory, a socket, or a device, you get a *const char\**
 * when using APIs that provide their own string implementation, like Qt: even if your entire codebase is using *std::string*, you will have some boundary zones where both types are used
 * when using third party libraries: you get most of the time a *const char \** or *std::string*

{% gist david-grs/1f17bafa94b9596ec375532836fccd12 lookup_unordered_map_string.cc %} 
<br />
One would say that you could simply convert your string to *std::string* before the lookup. However this might not be acceptable from a performance point of view, as it will perform a memory allocation. 
Nowadays, many STL implementations use "small string optimization* (SSO) &mdash; up to 15 bytes with g++ 6.2 and 22 chars with clang 3.9 &mdash; in order to avoid the memory allocation. Bytes are still 
copied though, and the performance of your system heavily relies on implementation details. 

The solution here is to use C++17's [*std::string_view*](http://en.cppreference.com/w/cpp/experimental/basic_string_view). 
This is not possible with standard containers, though: with *std::unordered_map* and *std::unordered_set*, you cannot query on another type than *Key* &mdash; and using a *std::unordered_map<std::string_view, X>* 
is a bad choice, for a number of reasons I explain in the [appendix](#appendix).

Easy task with Boost.MultiIndex: simlpy add a method that returns a string view and use it as index!

{% gist david-grs/1f17bafa94b9596ec375532836fccd12 lookup_mic_string_view.cc %} 
<br />
The magic is done on line 23, which defined the hashed unique index as `const_mem_fun<stock, std::experimental::string_view, &stock::ref_view>`: a const member function of *stock*, that returns
a string view, and can be found at *&stock::ref_view*.



unordered\_set&lt;unique\_ptr&lt;T&gt;&gt;
----------------------------
Keeping a set of *unique_ptr&lt;T&gt;* is less common, but sometimes convenient. Such pattern can be used if a class owns a collection of polymorphic *T*, where this owner class does not 
know anything about the concrete type.

{% gist david-grs/1f17bafa94b9596ec375532836fccd12 unordered_set_unique_ptr.cc %} 
<br />
Note: *on_error* is private. If it was public, this would be a poor API, as the client code would have a dangling reference the line right after the call to *node::on_error*. *node* owns 
and does not share *connection* ; otherwise it could simply use a *std::unordered_set&lt;std::shared_ptr&lt;T&gt;&gt;*.

There are a few ways to solve this with unordered STL containers, but...

1. `std::unordered_set<connection*>`: you lose the advantages of RAII
2. `std::unordered_map<connection*, std::unique_ptr<connection>>`: ugly and under-performant solution

Boost.MultiIndex allows you here to store unique_ptr while being able to access it from raw pointers:

{% gist david-grs/1f17bafa94b9596ec375532836fccd12 boost_mic_unique_ptr.cc %} 
<br />



composite keys
--------------
Composite keys is another quite common pattern when storing objects into associative containers. As the name implies, it allows to add an index composed by multiple fields. Let's take a quick example.

{% gist david-grs/1f17bafa94b9596ec375532836fccd12 session.h %}
<br />
This class represents a session of a user on a particular machine. A user can only log once on a host, which allows us to uniquely identify a session by the association
of the *username* and the *hostname*.

What are our options to do that with standard containers?

1. The most straightforward solution is to use a `std::unordered_map<std::pair<std::string, std::string>, session>`, as you don't have to provide any custom 
*std::hash* nor *operator==* implementations. I see a lot of problems with this solution though: it is bad from a performance point of view, because it uses four *std::string* instances
while we only need two and it allocates two *std::string* on each lookup. Also, the code is not very elegant and is error prone as you don't know if *username* is the 
first element of the pair or the second.

2. A better way to do it in my opinion is to have a `std::unordered_map<session_id, session>`, with *session_id* a *struct* with two *std::string*. This gets rid of a few drawbacks that we have in the solution 1, but requires a 
*std::hash* specialization and *operator==* definition. Not dramatic, but this raises more questions: how to implement a *good* hash function that takes two *std::string* as input?
I usually use [Boost.Functional/Hash](http://www.boost.org/doc/libs/1_62_0/doc/html/hash/combine.html) for that, but not every application can support Boost as a dependency. Apparently, 
a good start would be to use [a xor with some bit shifting](http://stackoverflow.com/questions/17016175/c-unordered-map-using-a-custom-class-type-as-the-key) on the two *std::string* hashes, but 
who knows if this is really good?

3. A last solution I could think of is to use `std::unordered_set<session>`, but it is still far from ideal: you have all the problems cited in the solution 2. The only advantage is that
you're using less memory as you don't have the two additional *std::string* instances as key... but now you have to *const_cast* all the time your stored objects, which can be annoying or simply not
desirable depending on your API. 

As you can see, there is no simple and straightforward solution to that (simple!) problem. Here is what is looks like with Boost.MultiIndex:

{% gist david-grs/1f17bafa94b9596ec375532836fccd12 boost_mic_composite_key.cc %}
<br />
The *composite_key* takes in this case two *const_mem_fun* &mdash; which is the argument used to refer to const member functions. For the lookup, you will have to build a Boost.Tuple as a key.

As I wrote in the first paragraph, building *std::string* to do a lookup take a significant time. If you care enough about performance, here is the solution that is going to squeeze the
last bits and nanoseconds: 

{% gist david-grs/1f17bafa94b9596ec375532836fccd12 boost_mic_composite_key_view.cc %}
<br />
Same approach as before: the composite key is still composed by the two methods of your class, that we modify of course in order to return a *std::string_view* and not a *std::string* anymore. The only
annoying bit is that we have to explicitly define the *composite_key_hash* as Boost.MultiIndex doesn't pick the *std::string_view* specialization!



<a name="appendix"></a>Appendix
--------
<br />

>
> Why using *std::unordered_map&lt;std::string_view, X&gt;* is bad design choice?
> 

It is a bad design because it is fragile and error-prone. The key is *const std::string_view*, but as it is a view, it can actually change anytime. The only
acceptable scenario in my mind would be if *X* owns the viewed string as *const std::string*.
<br />


>
> How do you insert an element in such container?
>

1. `m.emplace(x.str, x)`
2. `std::experimental::string_view sv{x.str};`<br />
`m.emplace(sv, x.str); ` 
3. `m.emplace(x.str, std::move(x));`
4. `std::experimental::string_view sv{x.str};`<br />
`m.emplace(sv, std::move(x));`
<br />

... and the answer is:

1. is wrong because *x* is going to be copied, while the key is pointing to *x.str*
2. is wrong for the same reason
3. is wrong because there is no guarantee on the order of evaluation of arguments, thus *x* could be moved before *x.str* is evaluated, resulting in the construction of a string view that points to some undefined memory area
4. is correct: we construct the view, and move *x*


