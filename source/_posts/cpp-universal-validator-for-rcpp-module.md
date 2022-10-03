---
title: A Universal Validator for Rcpp Module
top: false
cover: false
toc: true
mathjax: false
date: 2020-10-17 18:05:49
password:
summary:
tags:
  - Rcpp
categories: cpp
---


`Rcpp` module is is a useful interface to access/operate *cpp* classes from *R*. This [official document](https://cran.r-project.org/web/packages/Rcpp/vignettes/Rcpp-modules.pdf) gives many detailed examples of `Rcpp` module.


One feature that is unique in *cpp* is function overload, which makes it possible to distinguish functions by its name along with types of parameters. To enable this feature, `Rcpp` module uses a validator function pointer, which takes the form of `bool (*Rcpp::ValidConstructor)(SEXP*, int)` for constructors (and `bool (*Rcpp::ValidMethod)(SEXP*, int)` for methods) to achieve the overload of functions.

As there is no example of the validator function in the official document, this post intends to give a universal validator function that can be applied to all sorts of scenarios.


### Brief Introduction

The validator function should have following form.

```cpp
bool validator(SEXP* args, int nargs);
```

, where `SEXP* args` is an array of pointers to `RObject` and `int nargs` is the size of the array.

A vital helper function that connects `SEXP` and *cpp* type `T` in this validation process is
```cpp
namespace Rcpp {

  template<T> bool is(SEXP x);

} 
```
. This helper function helps identify whether the pointed `RObject` by `SEXP` (`SEXP` is a pointer to `RObject`) can be cast to type `T`.


As the `bool Rcpp::is<T>(SEXP x)` function only validates the type of one `SEXP`, there is some work need to be done before we can construct a universal validator.


### Variadic Template since cpp11

The variadic template was first introduced in cpp11. It enables template to accept variadic template parameters and process them recursively.


```cpp
template <typename... Types>
bool universal_validator(SEXP* args, int nargs) {
  return universal_validator<Types...>(args, nargs, 0);
}

// default T = void is the key to make the specialization, universal_validator<>(), possible.
template <typename T = void, typename... Types>
bool universal_validator(SEXP* args, int nargs, int idx) {
  // if more types than number of arguments
  if (idx>=nargs) return false;

  // type traits to make the function more general
  typedef typename Rcpp::traits::remove_const_and_reference<T>::type _Tp;

  return Rcpp::is<_Tp>(args[idx]) && universal_validator<Types...>(args, nargs, idx+1);
}

template <>
bool universal_validator<>(SEXP* args, int nargs, int idx) {
  // end condition
  // return false when arguments outnumber types
  return nargs == idx;
}
```

The function checks the types of the arguments one by one and returns false when type or length mismatch occurs. It is also possible the check the length in `bool universal_validator(SEXP* args, int nargs)` without having to check the types if the lengths are discrepant at the beginning. Hint shall be found at at [Rcpp_internal.h](https://github.com/RcppCore/Rcpp/blob/master/src/internal.h#L39). (How to find the length of an array given the address of its first element?)

And the reason why we need to check the length by the validator (not `Rcpp`) lies [here](https://github.com/RcppCore/Rcpp/blob/master/inst/include/Rcpp/module/class.h#L123).

### Example

The example is extended from the [StackOverflow Question](https://stackoverflow.com/questions/42579207/rcpp-modules-validator-function-for-exposed-constructors-with-same-number-of-pa/64289849#64289849)


```cpp
#include <Rcpp.h>
// [[Rcpp::plugins(cpp11)]]

using namespace Rcpp;

class Example {
  public:
    Example(): x(0), y(0) {
      Rcout << __PRETTY_FUNCTION__ << "\n";
    }

    Example(int x_): x(x_), y(0) {
      Rcout << __PRETTY_FUNCTION__ << "\n";
    }

    Example(int x_, int y_): x(x_), y(y_) {
      Rcout << __PRETTY_FUNCTION__ << "\n";
    }

    Example(std::string x_, std::string y_): x(x_.size()), y(y_.size()) {
      Rcout << __PRETTY_FUNCTION__ << "\n";
    }

    Example(std::string x_, int y_): x(x_.size()), y(y_){
      Rcout << __PRETTY_FUNCTION__ << "\n";
    }

    Example(int x_, int y_, int z_): x(x_), y(y_+z_) {
      Rcout << __PRETTY_FUNCTION__ << "\n";
    }

    int add() const { 
      return x + y; 
    }

private:
    int x, y;
};


RCPP_MODULE(example_module) {

  Rcpp::class_<Example>("Example")
  
  .constructor()

  .constructor<int>("(int) constructor", universal_validator<int>)
  .constructor<int, int>("(int, int) constructor", universal_validator<int, int>)
  .constructor<std::string, std::string> ("(string, string) constructor", universal_validator<std::string, std::string>)
  .constructor<std::string, int> ("(string, int) constructor", universal_validator<std::string, int>)
  .constructor<int, int, int>("(int, int, int) constructor", universal_validator<int, int, int>)

  .method("add", &Example::add)
  ;
}


// Following part will run automatically when sourcing the cpp code

/*** R

# default constructor
example_def <- new(Example)
example_def$add()

# int
example_int <- new (Example, 2L)
example_int$add()

# int, int
example_int_int <- new(Example, 1L, 3L) 
example_int_int$add()

# str, str
example_str_str <- new(Example, "abc", "de")
example_str_str$add()

# str, int
example_str_int <- new(Example, "abc", 4L)
example_str_int$add()

# int, int, int
example_iii <- new(Example, 1L, 4L, 3L)
example_iii$add()

*/
```


### Macro to Make Life Easier

In the former example, we repeat ourselves many times. There should be the room the improve the coding efficiency.

Modifying the source header files is possible while might incur unexpected problems (like incompatibility with older versions of cpp).

Following macro simplifies the code without having to touch the source code.


```cpp
#define CTOR(...) \
  constructor<__VA_ARGS__>("(" #__VA_ARGS__ ") constructor", universal_validator<__VA_ARGS__>)

RCPP_MODULE(example_module) {

  Rcpp::class_<Example>("Example")
  
  .constructor()

  /*
  .constructor<int>("(int) constructor", universal_validator<int>)
  .constructor<int, int>("(int, int) constructor", universal_validator<int, int>)
  .constructor<std::string, std::string> ("(string, string) constructor", universal_validator<std::string, std::string>)
  .constructor<std::string, int> ("(string, int) constructor", universal_validator<std::string, int>)
  .constructor<int, int, int>("(int, int, int) constructor", universal_validator<int, int, int>)
  */

  .CTOR(int)
  .CTOR(int, int)
  .CTOR(std::string, std::string)
  .CTOR(std::string, int)
  .CTOR(int, int, int)
  .method("add", &Example::add)
  ;
}
```


### Pointer Version

We need `args+nargs` to mark the end of arguments because the `SEXP` array is initialized like

```cpp
#define MAX_ARGS 65
#define UNPACK_EXTERNAL_ARGS(__CARGS__,__P__)    \
SEXP __CARGS__[MAX_ARGS]; \
// ...
```

The array contains the addresses, `SEXP`, of objects instantiated by default constructors. Therefore `args+nargs` is a valid address (not `nullptr`) of a non-null object (not `R_NilValue`).

If there is a way to make out default constructed `SEXPREC`, the validator can be further simplified.


```cpp
template <typename... Types>
bool universal_validator(SEXP* args, int nargs) {
  return universal_validator<Types...>(args, args + nargs);
}


template <typename T = void, typename... Types>
bool universal_validator(SEXP* args, SEXP* end) {
  if (args == end) 
    return false;

  typedef typename Rcpp::traits::remove_const_and_reference<T>::type _Tp;
  return Rcpp::is<_Tp>(*args) && universal_validator<Types...>(args + 1, end);
}


template <>
bool universal_validator(SEXP* args, SEXP* end) {
  return args == end;
}
```