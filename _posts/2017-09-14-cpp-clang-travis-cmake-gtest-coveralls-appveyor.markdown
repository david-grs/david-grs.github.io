---
layout: post
title:  "C++ CI: Travis, CMake, GTest, Coveralls & Appveyor"
author: David Gross
date:   2017-09-14 21:57
---

Last week, Jason Turner presented [an intro to Travis CI](https://www.youtube.com/watch?v=3ulKzD2cmSw). I've never used it but have wanted to try for a while, so I
gave it a shot. Here is a short summary of this small adventure...

First of all, what are these tools we are talking about?

  * Travis CI: a build farm for GNU/Linux and Mac OS X that also runs your unit tests
  * Coveralls: it works on the top of Travis CI, by generating coverage report (*lcov* on Linux)
  * Appveyor: like Travis CI, but for Windows

> #TL;DR
> You can checkout all the configuration files on [this github demo project](https://github.com/david-grs/clang_travis_cmake_gtest_coveralls_example)

To start, simply sign in with your GitHub account on [Travis CI](https://travis-ci.org/), this will import all your repositories. From there, just enable one of them 
and add the following *.travis.yml* to your repo:

{% highlight yaml %}
dist: trusty
sudo: false
language: cpp
  
script:
  - cmake .
  - cmake --build .
{% endhighlight  %}

On *git push*, it will automatically trigger your first Travis build. 
  


GCC 6 & Clang 5
---------------
By default, Travis CI is using a quite old version of GCC &mdash; 4.8, so no C++14 support. Switching to GCC 5 is quite simple and Jason explains in his video how to change the *.travis.yml*. The one for GCC 6 is almost identical:

{% highlight yaml %}
dist: trusty
sudo: false
language: cpp

addons:
  apt:
    sources:
      - ubuntu-toolchain-r-test
    packages:
      - g++-6 

script:
  - CXX=/usr/bin/g++-6 CC=/usr/bin/gcc-6 cmake .
  - cmake --build . 
{% endhighlight  %}

Here, we are using the *apt* add-on to install the *g++-6* package. You can add any packages or libraries on which your project depends. Alternatively, 
you can also use the last *clang* release to build your project:

{% highlight yaml %}
dist: trusty
sudo: false
language: cpp

addons:
  apt:
    sources:
      - llvm-toolchain-trusty-5.0
    packages:
      - clang-5.0

script:
  - CXX=/usr/bin/clang++-5.0 CC=/usr/bin/clang-5.0 cmake .
  - cmake --build . 
{% endhighlight  %}
<br />



Google Test
-----------
GTest is not handled by default &mdash; and we cannot blame Travis for that, but more Ubuntu that decided to stop distributing the library package. This means
that your CMake *FindPackage* will fail:

{% highlight yaml %}
CMake Error at /usr/share/cmake-3.2/Modules/FindPackageHandleStandardArgs.cmake:138 (message):
  Could NOT find GTest (missing: GTEST_LIBRARY GTEST_MAIN_LIBRARY)
Call Stack (most recent call first):
  /usr/share/cmake-3.2/Modules/FindPackageHandleStandardArgs.cmake:374 (_FPHSA_FAILURE_MESSAGE)
  /usr/share/cmake-3.2/Modules/FindGTest.cmake:204 (FIND_PACKAGE_HANDLE_STANDARD_ARGS)

  CMakeLists.txt:21 (find_package)
-- Configuring incomplete, errors occurred!
{% endhighlight  %}

I looked around and found that people solve this in various ways:

 1. With a pre-build bash script that installs the *libgtest-dev* package, builds it and copies the libraries &mdash; this is quite hacky and fragile as it depends on the various system paths and library names.
 2. With the GTest source in the repo &mdash; importing the whole source tree isn't necessary
 3. By adding GTest as a submodule &mdash; this is the way to go


Add GTest as a submodule: 
```git submodule add git@github.com:google/googletest.git gtest```

Then create a *gtest.cmake* file with this content:

{% highlight yaml %}
set(GOOGLETEST_ROOT gtest/googletest CACHE STRING "Google Test source root")

include_directories(SYSTEM
    ${PROJECT_SOURCE_DIR}/${GOOGLETEST_ROOT}
    ${PROJECT_SOURCE_DIR}/${GOOGLETEST_ROOT}/include
    )

set(GOOGLETEST_SOURCES
    ${PROJECT_SOURCE_DIR}/${GOOGLETEST_ROOT}/src/gtest-all.cc
    ${PROJECT_SOURCE_DIR}/${GOOGLETEST_ROOT}/src/gtest_main.cc
    )

foreach(_source ${GOOGLETEST_SOURCES})
    set_source_files_properties(${_source} PROPERTIES GENERATED 1)
endforeach()

add_library(gtest ${GOOGLETEST_SOURCES})
{% endhighlight  %}

Note the *SYSTEM* keyword in *include_directories*. Without it, you will get spammed on every build by all the warnings from GTest, especially if you build with *-Wall -Wextra etc*. 

Last step, include it in your main CMakeLists.txt and link to *gtest*:

{% highlight yaml %}
cmake_minimum_required(VERSION ...)

project(...)
include(gtest.cmake)
...

# GTest needs threading support
find_package (Threads)
target_link_libraries(... gtest ${CMAKE_THREAD_LIBS_INIT})
{% endhighlight  %}
<br />


Coveralls
---------
As you did for Travis CI, you need to sign in with your GitHub account on [Coveralls](http://coveralls.io/) and enable your project. 

Now edit your *.travis.yml* to install and run *cpp-coveralls*:

{% highlight yaml %}
dist: trusty
sudo: false
language: cpp

addons:
  apt:
    sources:
      - ubuntu-toolchain-r-test
    packages:
      - g++-6

before_install:
  - pip install --user cpp-coveralls

script:
  - CXX=/usr/bin/g++-6 CC=/usr/bin/gcc-6 cmake -DCOVERAGE=1 .
  - cmake --build . 
  - ./tests

after_success:
  - coveralls --root . -E ".*gtest.*" -E ".*CMakeFiles.*" 
{% endhighlight  %}
<br />
A few things to note here:

   1. We excluded Google Test from the coverage with *-E ".\*gtest.\*"*
   2. *-DCOVERAGE=1* is passed to CMake &mdash; locally, you might not want to enable coverage 
   3. Don't forget to run your unit tests if you want them in the coverage report


Finally, we need to enable the *gcda* files generation &mdash; needed by *lcov* &mdash; in our toolchain. This is done by adding
the *--coverage* flag to both compiler and linker:

{% highlight yaml %}
SET(COVERAGE OFF CACHE BOOL "Coverage")
...
if (COVERAGE)
    target_compile_options(tests PRIVATE --coverage)
    target_link_libraries(tests PRIVATE --coverage)
endif()
{% endhighlight  %}
<br />


Appveyor
--------
As a Linux user, Appveyor was a bit tricky as I couldn't build locally. Be sure to remember:
 
  * Appveyor will NOT clone your project with *git clone --recursive*, so you need to run manually *git submodule update --init --recursive* after the checkout
  * It is not possible to build directly with CMake &mdash; Appveyor expects a sln file, you can generate it via CMake, e.g. *cmake -G "Visual Studio 15 2017 Win64"*
  * The last 1.8.0 release of GTest does not build with Visual Studio 2017 &mdash; you need to add the definition *GTEST_HAS_TR1_TUPLE=0* to tell GTest not using ::tr1 stuff

Here is my *appveyor.yml* file:

{% highlight yaml %}
version: '1.0.{build}'

image: Visual Studio 2017

platform:
  - x64
 
configuration:
  - Release
  - Debug

install:
    - git submodule update --init --recursive

before_build:
    - cmake -G "Visual Studio 15 2017 Win64" .

build:
  project: $(APPVEYOR_BUILD_FOLDER)\$(APPVEYOR_PROJECT_NAME).sln

test_script:
  - '%APPVEYOR_BUILD_FOLDER%\%CONFIGURATION%\tests.exe'
{% endhighlight %}



