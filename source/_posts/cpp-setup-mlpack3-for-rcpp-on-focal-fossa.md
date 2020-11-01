---
title: Setup MLPACK3 For Rcpp on Focal Fossa
top: false
cover: false
toc: true
mathjax: false
date: 2020-10-20 20:02:31
password:
summary:
tags:
  - Rcpp
  - MLPACK
categories:
  - cpp
---


Since 20.04, Ubuntu's official library has introduced binary version of a *cpp* machine learning library, `MLPACK`. This saves the tedious effort of building it from source.

This week I tried to install and setup the library for *Rcpp* by [RcppMLPACK](https://github.com/rcppmlpack/rcppmlpack2). It took me several hours solving those unexpected problems, though it's better than last time when I gave it up. In the end I solved all the problems (on the *R* side), one of them had even been untouched by the author of the library. 

There is almost no reference for this problem. But if you consider the merit of modeling by *cpp* and doing other things flexibly by *R*, the effort still worthed it.

In this post I will go through the pain I suffered recently, and show you all a guild for setting it up without falling into the trap. This is a history of blood and tears (not true).


### Linking Flags

From [the source](https://github.com/rcppmlpack/rcppmlpack2/blob/master/configure.ac#L38-L39) code, we could identify that the linking flags for `MLPACK` when compiling it for `RcppMLPACK` are collected by

```bash
pkg-config --cflags mlpack
pkg-config --libs mlpack
```


When `MLPACK` was freshly installed, the command returned the wrong flag, which looks like `-lBoost::program_options ...`.
At first I thought this line was hidden somewhere in the configuration of `RcppMLPACK`, and it took me two hours to find out that `RcppMLPACK` did nothing wrong. Since the links to `boost` were wrong, the compilation failed at the point of compilation with those links.

The solution is very simple. One line to locate, and one line to amend.

```bash
# tells the location of config file, mlpack.pc
pkg-config --debug mlpack
```

For example, mine looks like */usr/lib/x86_64-linux-gnu/pkgconfig/mlpack.pc*.

Inside the file, the correct version looks like (This is the corrected version).

```pc
  line>Name: mlpack
  line>Description: scalable C++ machine learning library
  line>URL: http://www.mlpack.org/
  line>Version: 3.2.2
  line>Cflags:  -I/usr/include/stb -I/usr/include/
  line>Libs: -larmadillo -lboost_program_options -lboost_unit_test_framework -lboost_serialization -lmlpack
```

All we need to do now is replace the last line with the right links for three `boost` libraries.



### Configure VSCode

The reason remains unknown by VSCode becomes fairly slow, too slow to work properly, when `MLPACK` is included.

I have seen this phenomenon in another computer half a year ago, with `gcc` as compiler and intellisense.

You might wonder why I mention `gcc` out of the blue. Yes, the solution is...

**Replace Compilers and Intellisense By `Clang`!!!**

The problem is still unknown, but `clang` solved it.



### Wrap arma Object Properly in Rcpp::List

The `Rcpp::wrap<>` for `arma` objects is not functioning well if a `Rcpp::List` is returned. The same phenomenon is never encountered in `RcppArmadillo`.

After examining the header files of `RcppMLPACK`, I found this was due to the inappropriate order of including `Rcpp` and `RcppArmadillo` headers.

As suggested in *RcppArmadillo.h*, the declaration of `Rcpp::wrap`, as well as other helper functions shall be included before `Rcpp.h`. 

The following steps solves the abnormal action.

1. Extract declaration part from *RcppArmadilloForward.h*, save it as a separate header file, e.g. *RcppMLPACK/include/RcppArmadilloDeclerations.h*. Don't forget `#ifndef ... #define ... #endif`

2. Save original *RcppMLPACK.h* as another file, e.g. *RcppMLPACK_orig.h*, and replace it with the modified version. The order is not absolute, but this is what I tested to be right. (Other including orders might induce unexpected consequences, like `std::bad_alloc` error from inconsistent configurations in `armadillo` and its *R* version).


RcppMLPACK.h
```h
#ifndef RcppMLPACK__RcppMLPACK__h
#define RcppMLPACK__RcppMLPACK__h

#if _WIN64
#ifndef ARMA_64BIT_WORD
#define ARMA_64BIT_WORD
#endif
#endif

#if defined(__MINGW32__)
#define ARMA_DONT_USE_CXX11
#endif

// Rcpp has its own stream object which cooperates more nicely with R's i/o
// And as of Armadillo 2.4.3, we can use this stream object as well
//
// More recently this changes from ARMA_DEFAULT_OSTREAM to ARMA_COUT_STREAM
// and ARMA_CERR_STREAM
#if !defined(ARMA_COUT_STREAM)
  #define ARMA_COUT_STREAM Rcpp::Rcout
#endif
#if !defined(ARMA_CERR_STREAM)
  #define ARMA_CERR_STREAM Rcpp::Rcerr
#endif


// load Rcpp common and R config
#include <RcppCommon.h>
#include <Rconfig.h>

// load armadillo
#include <mlpack/core.hpp>

// must be included after Rcpp common, R config and armadillo
// to enable proper Rcpp::wrap<> inside Rcpp::List
// see RcppArmadillo.h for reference
#include <RcppArmadilloDeclarations.h>

#include <Rcpp.h>

#undef ARMA_EXTRA_MAT_PROTO
#undef ARMA_EXTRA_MAT_MEAT

// instead of including RcppArmadillo.h -- which re-includes parts
// of Armadillo already brought in by mlpack, we just include pieces
// needed for sugar wrapping etc

// Shall not be included before armadillo (mlpack)
#include <RcppArmadilloConfig.h>
#include <RcppArmadilloWrap.h>
#include <RcppArmadilloAs.h>
#include <RcppArmadilloSugar.h>

// prevent inclusion of Rcpp.h and RcppArmadillo.h via the
// autogenerated RcppExports.cpp
#define Rcpp_hpp
#define RcppArmadillo__RcppArmadillo__h

#endif
```

end.
