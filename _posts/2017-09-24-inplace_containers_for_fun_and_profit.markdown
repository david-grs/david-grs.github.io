---
layout: post
title:  "In-place containers for fun and profit"
author: David Gross
date:   2017-09-24 19:31
---

Last year I posted about [static_any](http://david-grs.github.io/low_latency_stack_based_boost_any/) and I am back with a similar container that I implemented
recently, [inplace_string](https://github.com/david-grs/inplace_string). 

> For the record, `static_any` should have been be called `inplace_any`, but at this time I chose the prefix *static* as *Boost.StaticVector* was popular and it 
> was clear to everybody what to expect from such container.



inplace_string
--------------
The idea behind `inplace_string` is to get a full replacement of C++17's `std::string`, with an in-place memory storage:

{% highlight cpp %}
using Name = inplace_string<15>; // 16 bytes on stack, size included
Name name = "foo"; 

auto it = name.find("r");
assert(it == Name::npos);

name += "bar";
  
std::string str(name); // implicit string_view construction
{% endhighlight cpp %}
<br />
What you get as benefits:

  * Faster string operations: string construction and destruction are much faster since no memory allocation takes place. Other operations are also slightly faster: as its buffer
does not grow, `inplace_string` benefits from a simpler code with less branches.
  * Cache-friendly: it stays close to your other class members, your cache likes it.
  * Simplicity: it allows you to avoid dynamic allocations without implementing your own memory allocator.

As for the implementation, no surprises: the underlying container is `std::array<CharT, N>` and it uses the famous trick &mdash; made popular by fbstring &mdash; of storing the remaining size within the last byte of the string &mdash; 
if you don't know it yet [look here](https://github.com/david-grs/inplace_string/blob/81defa14a0650fa804b9c83624b7735084258a3b/inplace_string.h#L287) and think how the value will 
change when the string reaches the maximum capacity.

One of the only difference with `std::string` is the presence of a constructor from `const CharT(&)[M])`, allowing a compile-time error it the input exceeds the maximum capacity:

{% highlight cpp %}
inplace_string<5> too_small;
too_small = "foobar"; // error: static_assert failed "basic_inplace_string: size exceeds maximum capacity"
{% endhighlight cpp %}
<br />


std::string's SSO
-----------------
At first glance, it might seem that `inplace_string` overlaps with `std::string`'s SSO (Short String Optimization), although it does not.

One of the reason is that SSO is implementation dependent:

  * 15 bytes with gcc 5
  * 22 bytes with clang 5
  * 15 bytes with VS 2015 and 2017

Another reason is that even though everybody thinks that using gcc 5 is enough to get SSO, it is wrong: as of this writing, most of the popular Linux distributions (CentOS, Debian, RedHat) patched the glibc/gcc 
to keep the previous `std::string` implementation without SSO (and with COW!). This has been done to not break `std::string`'s ABI when C++11 has been release, cf this [post on RedHat developers' blog](https://developers.redhat.com/blog/2015/02/05/gcc5-and-the-c11-abi/).

More importantly, `inplace_string` *guarantees* that your code will never allocate. If, for any reason, your string grows more than you expected, it will throw... and you will know that you were wrong, as 
your application will likely crash! On the other hand, crossing fingers and relying on `std::string`'s SSO is somewhat a bet, and you will never know if one day, strings get bigger and start allocating.




string_view
-----------
In the current C++ world, `std::string` has a monopoly and the following code is common practice:

{% highlight cpp %}
void foo(const std::string& bar)
{
   ...
}
{% endhighlight cpp %}
<br />
In a code base with `std::string` as unique string class, this code is reasonable &mdash; it is not when multiple string classes live together, as it defeats the
purpose of not using `std::string` if you need to do conversions all over the place. 

{% highlight cpp %}
foo("foobar"); // compiles, but allocates

inplace_string<15> ss = "foobar";
foo(ss); // does not compile, std::string does not know inplace_string

foo(ss.c_str()); // compiles, but allocates and copies the string
{% endhighlight cpp %}
<br />
To solve this issue, C++17 brings us `std::string_view`: `std::string`, `inplace_string` and friends implement an implicit cast operator to `std::string_view`, and mixing 
all of them work nicely:

{% highlight cpp %}
void foo(std::string_view bar)
{
   ...
}

foo("foobar"); // compiles, does not allocate

inplace_string<15> ss = "foobar";
foo(ss); // compiles, building string_view is cheap (taking pointer + size)

std::string s = "foobar";
foo(s); // compiles
{% endhighlight cpp %}
<br />



Mixing in-place containers 
--------------------------
Let's consider the following code:

{% highlight cpp %}
using MetadataTag = std::string;
using MetadataValue = std::experimental::any;

using Metadata = std::pair<MetadataTag, MetadataValue>;

struct MetadataTree 
{
    template <typename StringT, typename ValueT>
    void Add(StringT&& str, ValueT&& value)
    {
        _metadata.emplace_back(std::forward<StringT>(str), std::forward<ValueT>(value));
    }

    std::vector<Metadata> _metadata;
};
{% endhighlight cpp %}
<br />
This object is very similar to `QVariantMap`, which is a `QMap<QString, QVariant>` &mdash; `QVariant` is actually not like `std::variant`, but similar to `std::any`. I use this pattern a lot, 
as it is convenient to use a string to index various objects. One issue with this was speed: having to allocate memory due to many `std::string` is a major obstacle.

Its equivalent with in-place containers *could be*:
{% highlight cpp %}
using MetadataTag = inplace_string<15>;
using MetadataValue = static_any<16>;

using Metadata = std::pair<MetadataTag, MetadataValue>;

struct MetadataTree 
{
    template <typename StringT, typename ValueT>
    void Add(StringT&& str, ValueT&& value)
    {
        _metadata.emplace_back(std::forward<StringT>(str), std::forward<ValueT>(value));
    }

    static constexpr MaxCapacity = 8;
    boost::container::static_vector<Metadata, MaxCapacity> _metadata;
};
{% endhighlight cpp %}

Now, let's benchmark these two guys &mdash; I know, it is unfair, but we all love looking at numbers! For that, let's take a simple usage of 
our `MetadataTree` class:

{% highlight cpp %}
template <typename TreeT>
TreeT GetTree(int i, double d, bool b)
{
  TreeT tree;
  tree.Add("metadata1", i);
  tree.Add("metadata2", d);
  tree.Add("metadata3", b);
  return tree;
}
{% endhighlight cpp %}

... and the result (time, instructions and cycles per iteration):

{% highlight cpp %}
Test                        Time (ns)          INS          CYC 
---------------------------------------------------------------
MetadaTree                        536        2,205        1,010 
in-place MetadaTree                 3            3            2 
{% endhighlight cpp %}
<br />

You can find the benchmark on [my github](https://github.com/david-grs/inplace_examples). Here is the full output of `perf`:

{% highlight cpp %}
$ perf stat -e cycles,instructions,cache-misses ./inplace_examples inplace
inplace3000000

 Performance counter stats for './inplace_examples inplace':

         2 540 791 cycles                   
         2 384 036 instructions              #    0,94  insns per cycle        
            10 970 cache-misses                                                

       0,001584907 seconds time elapsed

$ perf stat -e cycles,instructions,cache-misses ./inplace_examples notinplace
notinplace3000000

 Performance counter stats for './inplace_examples notinplace':

     1 005 689 341 cycles                   
     2 209 217 853 instructions              #    2,20  insns per cycle        
            20 059 cache-misses                                                

       0,531293829 seconds time elapsed
{% endhighlight cpp %}



