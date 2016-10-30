---
layout: post
title:  "Why you should use Boost.MultiIndex... (Part I)"
author: David Gross
date:   2016-10-30 22:04
---

Although Boost.MultiIndex is a pretty old library &mdash; introduced in Boost 1.32, released in 2004 &mdash; I found it rather unsung and underestimated
across the C++ community in comparison to other non-standard containers. One reason might be due to the declaration of *boost::multi\_index\_container*  
which might have discouraged more than a developer due to its long list of template parameters ; but as we will see later, it is actually very simple 
and clear... if correctly indented!


Multiple views
--------------
In a situation involving multiple views on the same data, there is no doubt: Boost.MultiIndex is what you want and what you should use. Let's take a quick example.

> You have a set of unique integers and do mainly two operations on it: 
>
> + *adding an integer*
> + *iterating over the set in ascending and descending order*

This is achievable quite easily with *boost::multi\_index\_container*:

{% gist david-grs/7519f9b37941626a673926e498317305 mic_ints_asc_desc.cpp %}
<br />
The "hard" part done, the usage of the container itself is trivial. Actually, most of the operations are identical to the ones you can find on a standard container. The only
difference is that you have to get a *view* on the container for some of the operations:

{% gist david-grs/7519f9b37941626a673926e498317305 mic_view.cpp %}


The unfortunate alternative
---------------------------
The alternative and unfortunately most common way to solve this problem is to maintain two _std::set_, one using _std::greater_, the other one _std::less_. 
However, there are few problems with this approach:

1. it is *error prone*: each operation has to be done on the two sets, and if you forget you will have a bad time
2. the *time complexity* is worse on insertion as you have the value twice &mdash; and even if the object is small it will be slower due to the memory allocations of the container
3. the *spatial complexity* will be worse
4. it can be *complicated* if instead of storing twice the value you decide to store a pointer, reference or iterator in one of the two containers: they can be invalidated depending on the container and the operations performed on it
5. it is not very *elegant* from the code perspective
6. if the constructor, destructor, etc are not declared *nothrow*, it will be even more fun!

To sum up, yes, you can isolate the two containers in a separate *class*, wrap all the methods, deal with the exceptions to 
rollback the operation if something throws and code the unit tests. Good luck, though! And it will be slower than Boost.MultiIndex,
and there will be another few hundreds of lines of code to maintain. 


Performance
-----------
When we do C++, performance usually matters: how fast is this double-indexed *boost::multi\_index\_container* compared to the two *std::set*? 

![Speed comparisons between boost::multi_index_container with two ordered indexes and two std::set, time in ms, measured on 1e6 iterations.](/assets/images/speed_boost_multi_index_std_set.jpg)

As we thought, the *multi_index_container* is much faster on insertion (and removal). The solution using two *std::set*'s forces us to duplicate all the operations that modify the container: on insertion
and deletion, both sets need to be updated. This is especially bad as these operations involve memory allocations ; you can then expect this time difference to increase with the size of the value stored.

When doing a lookup or walking in ascending or descending order, performance are almost the same &mdash; as they are done only on *one* std::set and not the two. 

> The entire code of this benchmark is available on [my github](https://github.com/david-grs/boost_multi_index_container/blob/master/integers.cc) 

We see that the lookup is slightly slower, probably due to the overhead added by the multiple indexes. What can we do about that? Well, adding a third index, not *ordered* this time, but *hashed*! 

{% gist david-grs/7519f9b37941626a673926e498317305 mic_ints_asc_desc_hashed.cpp %}
<br />
... and the lookup is now 10times faster. What is almost free with Boost.MultiIndex &mdash; adding an index &mdash; has definitely a cost in our alternative solution: if we add a *std::unorered_set*,
the time of insertion and removal grows significantly (and the complexity of the code that manages this additional container too).

![Speed comparisons between boost::multi_index_container with three indexes (two ordered, one hashed) and two std::set plus std::unordered_set, time in ms, measured on 1e6 iterations.](/assets/images/speed_boost_multi_index_std_set_and_unordered_set.jpg)


Implementation
--------------
Behind the scene, *boost::multi\_index\_container* is using a system of headers in order to do its job. Each node consists in the stored object, plus a sequence of headers, depending on the indexes
that are used.

There is a specific header for each index type.

### Ordered
The ordered index is implemented as a red-black tree, thus the header is composed by 3 pointers: one to the parent and the two children pointers. The interesting point here is that the color of the node 
is &mdash; on most platforms &mdash; encoded in the LSB of the parent pointer, resulting in a header of 24 bytes instead of 32 bytes (on a 64-bit systems).

As *std::map* does not perform such space optimization, this can explain the result we got in the previous benchmark, where the lookup was faster on  *boost::multi\_index\_container* than *std::set*.


### Unordered
The implementation of the hash map requires two pointers for the bucket management.

Again, *std::unordered_map* takes more space (3 pointers on GCC 6.2, x86-64), which can explain why the lookup was ~20% slower.


### Sequenced
The sequenced index is implemented as a double-linked list: the header is composed by 2 pointers, one pointing to the next node, the other one to the previous node.


### Random access
The random access index is implemented with an array of pointers to the nodes. In the node itself, the header is only composed by one pointer, which points back to its corresponding element in the main array of pointers.


### Summary

index type               | associated header&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;| size on 64-bit systems
-------------------------|---------------------------------------
ordered                  | 3 pointers | 24 bytes
unordered (hashed)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|2 pointers | 16 bytes          
sequenced                | 2 pointers | 16 bytes
random                   | 1 pointer | 8 bytes

<br />
Then, in the previous double-ordered and hashed container, the overhead was *24+24+16=64 bytes*. If we take *sizeof(int) == 4 bytes*, each value is taking in memory *68 bytes*.

In the other solution, each instance of *std::set* and *std::unordered_set* is using *40 bytes* (GCC 6.2, x86-64 Linux), resulting in a usage of *120 bytes* per value.








I like how self-describing is the declaration of the container. 



1. when using multiple index: obviously you _have_ to use it to avoid: 1. performance 1. redundancy 2. error (state where one struct is updated and not the other ones... rollback etc) 

2. type robustnes: because your key might not be strongly typed - boost.mic makes it clean on what is the view

  struct employee
  {
    std::string first_name;
    std::string last_name;
    int age; 
    double salary;
  };

  std::map<std::string, employee> 

  or using first_name = std::string 
       std::map<first_name, employee>

  without going further, just by the look of the type, we can only guess about the lookup. first name? last name? both...?

  while :

    boost::multi_index_container<
      employee,
      indexed_by<
        hashed_unique<
            tag<by_name>,
            BOOST_MULTI_INDEX_MEMBER(employee, std::string, first_name)
        >
      >
    >

   ... the type describes entirely what it does: it contains an employee, which is indexed by only one index, unique and hashable, composed by the first name. 

3. same example, lets imagine now that the actual key on your employee is a full name. 

     jjust replace
   
            BOOST_MULTI_INDEX_MEMBER(employee, std::string, first_name)

     by

          composite_key<
            employee,
            BOOST_MULTI_INDEX_MEMBER(employee, std::string, first_name),
            BOOST_MULTI_INDEX_MEMBER(employee, std::string, last_name)
          >

     on the other side, with map, things are now much complicated..... :

     what can you do?
        1. std::map<std::pair<std::string, std::string>, employee> m;
           -> this is the quickest and easiest solution, as you don't have to deal with operator< or == + hash. the problem is: it is unclear, error prone (which field is the first?), and ugly.

           it would be a bit better with : using FullName = std::pair<std::string, std::string> but still.

        2. std::map<full_name, employee>: the least bad solution here imho if you don;t caere about performance, but you have to write operator< /==/hash. 

        3. std::set<employee, employee_by_name> is a solution too, but now you have to const_cast all the time (as employee is the key), and also be very careful on when you do it... you still have to write an operator< / == / hash though.

     so in any cases, you have to write quite some code, so here, we see that the small overhead we had with boost.mic is already gone (with all the benefits we have from it for free!)



  ===>>> in a more general way, you can use function !


4. in general, it will be more efficient, depending on the key size:
      - on lookup, you have to build the key. 
                     1. if it is a string, don't think about it, don't use it, cf point 4
                     2. 
            struct Employee { std::string name; std::string address; int age; etc }; 
            
            map<std::string, Employee> mEmployeeByName - you will build a string, see later

            an alternative would be to do set<Employee, ByName> - but you usually don't want that, because 1. Employee will need a ctor from the field used as key (which can break your API just for that) 2. Employee will be const, as it is the key. 


      - on insert, you have to copy if you use map, you have to copy the key
        its the same roughly here - because you can move in map but not un boost.mic

      - on lookup, if you use map, it is bad for cache locality, as you make hot the area of the key, not your object - depending on the side of the key this could be annoying


4. whenever your key is a STRING: because it is safer, but also FASTER:
    - string as a key is bad, for 2 reasons:
         - on lookup, you have to build a string (heap alloc if not a small string). this is annoying because you often get string from shared memory or socket buffer, and they are 
           not string but _view_. and with std::unordered_map/map/set/etc you cannot perform a lookup from a type different that the key. 

         - on insert, because you have to copy it, again a heap alloc and bytes copies... which is quite bad.

   - so lets use string_view as a key! yes, but you have to do it right. 
        cf 

        std::experimental::string_view view{s.market_ref};
        auto it = m_stocks.emplace(view, s).first;

        but even...

        auto it = m_stocks.emplace(s.market_ref, s).first;
        auto it = m_stocks.insert({s.market_ref, s}).first;

        as string_view ctor is not explicit! 

        and all these things are very bad because sometimes, due to compiler optimizations, it will work. (sometimes.....)

    -> the way to go is to move the string ..... 

         but is...   auto it = m_stocks.emplace(ss.market_ref, std::move(ss)).first;   
         ...correct? ss could be moved before we take market_ref...

         how? well.....
     
        auto ss(s);
        std::experimental::string_view v(ss.market_ref);
        auto it = m_stocks.emplace(v, std::move(ss)).first;

        as simple as that. but it is still very slow at insertion, because you have to copy the entire struct before its insertion.

 
5. it is SAFER from an exception PoV: references are never invalidated, even on re-hash.

<insert table of references / iterator invalidation: map, unordered_map, boost.mic, google dense hash map>

