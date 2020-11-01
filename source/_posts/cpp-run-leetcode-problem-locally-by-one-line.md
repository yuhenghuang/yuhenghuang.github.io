---
title: How to Run Leetcode Problem Locally By One Line
top: false
cover: false
toc: true
mathjax: false
date: 2020-10-26 21:16:40
password:
summary:
tags:
  - Tuple
  - Variadic Template
categories:
  - cpp
---


This post introduce a way to run leetcode problem by one line in *cpp*, with all the process wrapped up. `c++14` and `c++17` features are widely used to simplify the code.

Without those convenient features, there are also alternative ways, though hose alternatives require a certain understanding of *cpp* templates.


### Helper Functions and Structs


1. Parse necessary types from string.

```cpp
// templated parser for primitive types and pointers
template<typename T>
struct universal_parser {
  T operator()(std::string&);
};

// partial specialization for 1d vector
template<typename T>
struct universal_parser<std::vector<T>> {
  std::vector<T> operator()(std::string&);
};

// partial specialization for 2d vector
template<typename T>
struct universal_parser<std::vector<std::vector<T>>> { 
  std::vector<std::vector<T>> operator()(std::string&);
};

```

2. Print output

```cpp
template<typename T>
void universal_print(const T&);

// overload for 1d vector
template<typename T>
void universal_print(const std::vector<T>&);

// overload for 2d vector
template<typename T>
void universal_print(const std::vector<std::vector<T>>&);

// specialization for tree
template<>
void universal_print(const TreeNode* root);

// specialization for linked list
template<>
void universal_print(const ListNode* root);
```

3. Split input

```cpp
std::vector<str::string> string_split(std::string&);
```


### Basic Flow

Input is one line of a text file for one run, which is `std::string` in the process, and it will be transformed to `std::vector<str::string>`. Another component is the pointer to the member function of class `Solution`. We will learn how to start from these two parts to achieve the goal.

The main function `ufunc` accepts two parameters, function pointer and a line of string.

```cpp

// fill parameters in tuple in-place
template <typename Tuple, std::size_t... Is>
void input_gen(Tuple& params, std::vector<std::string>::iterator iter, std::index_sequence<Is...>);

// run the function
template <class Solution, typename Ret, typename Tuple, typename... Types, std::size_t... Is>
std::enable_if_t<!std::is_void<Ret>::value> 
ufunc_call(Ret (Solution::*fn)(Types...), Tuple& params, std::index_sequence<Is...>);


// modify-in-place version
template <class Solution, typename Ret, typename Tuple, typename... Types, std::size_t... Is>
std::enable_if_t<std::is_void<Ret>::value> 
ufunc_call(Ret (Solution::*fn)(Types...), Tuple& params, std::index_sequence<Is...>);

// main function
template <class Solution, typename Ret, typename... Types>
void ufunc(Ret (Solution::*fn)(Types...), std::string& line) {

  // split string to vector<string>
  std::vector<std::string> args = string_split(line);

  // initialize tuple used to save input parameters
  std::tuple<std::remove_const_t<std::remove_reference_t<Types>> ...> params;

  // parse parameters and save to tuple
  input_gen(params, args.begin(), std::index_sequence_for<Types...>{});

  // run the function given function pointer and parameters(tuple)
  ufunc_call(fn, params, std::index_sequence_for<Types...>{});
}
```


### Fill Parameters in Tuple


The `((Is), ...)` syntax will repeat the expression in the first parenthesis for every element in the variadic template.

`*iter++` is always valid as long as the size of `args` is equal to the size of `Types...` (number of parameters of the method).

```cpp
template <typename Tuple, std::size_t... Is>
void input_gen(Tuple& params, std::vector<std::string>::iterator iter, std::index_sequence<Is...>) {
  (
    (std::get<Is>(params) = 
      universal_parser<std::tuple_element_t<Is,Tuple>>()(*iter++)
    ), 
  ...);
}
```



### Run

```cpp

// print return value
template <class Solution, typename Ret, typename Tuple, typename... Types, std::size_t... Is>
std::enable_if_t<!std::is_void<Ret>::value> 
ufunc_call(Ret (Solution::*fn)(Types...), 
            Tuple& params, 
            std::index_sequence<Is...>) {

  Solution sol;
  Ret res = (sol.*fn)(std::get<Is>(params) ...);
  universal_print(res);
}

// print first parameter
template <class Solution, typename Ret, typename Tuple, typename... Types, std::size_t... Is>
std::enable_if_t<std::is_void<Ret>::value> 
ufunc_call(Ret (Solution::*fn)(Types...), 
            Tuple& params, 
            std::index_sequence<Is...>) {

  Solution sol;
  (sol.*fn)(std::get<Is>(params) ...);
  universal_print(std::get<0>(params));
}
```