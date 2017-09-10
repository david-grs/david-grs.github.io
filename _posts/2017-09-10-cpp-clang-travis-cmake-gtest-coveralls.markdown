---
layout: post
title:  "Clang, Travis, CMake, GTest & Coveralls"
author: David Gross
date:   2017-09-10 14:52
---

This week, Jason Turner presented [an intro to Travis CI](https://www.youtube.com/watch?v=3ulKzD2cmSw). I never used Travis CI but wanted to try for a while, so I
gave it a shot. Here is a short summary of this adventure...

To start, simply sign in with your GitHub account on [Travis CI](https://travis-ci.org/), this will import all your repositories. From there, just enable one of them 
and add the following *.travis.yml* to your repo:

```yaml
dist: trusty
sudo: false
language: cpp
  
script:
  - cmake .
  - cmake --build .
```

On *git push*, it will automatically trigger your first Travis build. A few things though:

  1. The default GCC version is quite old &mdash; 4.8, no C++14 support
  2. Google Test won't work out of the box
  3. This doesn't include coverage report



GCC 6 & CLang 5
---------------
Switching to GCC 5 is quite simple &mdash; Jason explains in his video how to change the *.travis.yml*. The one for GCC 6 is almost identical:

```yaml
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
```

Here, we are using the *apt* addon to install the *g++-6* package. You can add any packages or libraries from which your project depends. Alternatively, 
you can also use the last *clang* release to build your project:

```yaml
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
```
<br />



Google Test
-----------
GTest is not handled by default &mdash; and we cannot blame Travis for that, but more Ubuntu that decided to stop distributing the library package. This means that your
CMake *FindPackage* will fail:

```yaml
CMake Error at /usr/share/cmake-3.2/Modules/FindPackageHandleStandardArgs.cmake:138 (message):
  Could NOT find GTest (missing: GTEST_LIBRARY GTEST_MAIN_LIBRARY)
Call Stack (most recent call first):
  /usr/share/cmake-3.2/Modules/FindPackageHandleStandardArgs.cmake:374 (_FPHSA_FAILURE_MESSAGE)
  /usr/share/cmake-3.2/Modules/FindGTest.cmake:204 (FIND_PACKAGE_HANDLE_STANDARD_ARGS)

  CMakeLists.txt:21 (find_package)
-- Configuring incomplete, errors occurred!
```

I looked around and found that people solves this in various ways:

 1. With a pre-build bash script that installs the *libgtest-dev* package, build it and copy the libraries &mdash; this is quite hacky and fragile as it depends on the various system paths and library names.
 2. With the GTest source in the repo &mdash; importing the whole source tree isn't necessary
 3. By adding GTest as a submodule &mdash; this is the way to go


Add Google Test as a submodule: 
```git submodule add git@github.com:google/googletest.git gtest```

Then, create a *gtest.cmake* file with this content:

```yaml
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
```

Note the *SYSTEM* keyword in *include_directories*. Without it, you will get spammed on every build by all the warnings from GTest, especially if you build with *-Wall -Wextra etc*. 

Last step, include it in your main CMakeLists.txt and link to *gtest*:

```yaml
cmake_minimum_required(VERSION ...)

project(...)
include(gtest.cmake)
...
target_link_libraries(tests gtest pthread)
``` 



Coveralls
---------
As you did for Travis CI, you need to sign in with your GitHub account on [Coveralls](http://coveralls.io/) and enable your project. 

Now, edit your *.travis.yml* to install and run *cpp-coveralls* &mdash; after the unit tests passed:

```yaml
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
```

A few things to note here:
  * We excluded Google Test from the coverage with *-E ".\*gtest.\*"*
  * *-DCOVERAGE=1* is passed to CMake &mdash; locally, you might not want to enable coverage all the time
  * Don't forget to run your tests!

Now we need to enable the *gcda* files generation &mdash; needed by *lcov* &mdash; in our toolchain. This is done by adding
the *--coverage* flag to both compiler and linker:

```yaml
SET(COVERAGE OFF CACHE BOOL "Coverage")
...
if (COVERAGE)
    target_compile_options(tests PRIVATE --coverage)
    target_link_libraries(tests PRIVATE --coverage)
endif()
```


