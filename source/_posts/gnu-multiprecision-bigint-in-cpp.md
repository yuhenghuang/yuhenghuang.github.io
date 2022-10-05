---
title: GNU MultiPrecision BigInt in Cpp
top: false
cover: false
toc: true
mathjax: true
date: 2022-10-03 15:58:48
password:
summary:
tags:
  - BigInt
  - GNU MultiPrecision
categories: cpp
---

## Introduction

The post introduce a widely used `BigInt` library **GNU MultiPrecision** (GMP for short) for computation in *c++*. After brief explanation of the library, an example is constructed to deepen the understanding and hands-on skills.


## GNU MultiPrecision

The brief introduction of the library from official website.

> GMP is a free library for arbitrary precision arithmetic, operating on signed integers, rational numbers, and floating-point numbers. There is no practical limit to the precision except the ones implied by the available memory in the machine GMP runs on. GMP has a rich set of functions, and the functions have a regular interface.



### Installation

Assume one has set up compilation tools before the installation.

First go to the official website https://gmplib.org/ to download the file and unzip it.

Most of the following part can be found on [Official Installation Instructions](https://gmplib.org/manual/Installing-GMP).

```bash
# by default the library is built for c, not c++
./configure --enable-cxx

make

# Some self-tests can be run with
make check

# And you can install (under /usr/local by default) with
make install
```

The compilation flag of the library should be `-lgmp -lgmpxx` for *cxx* programs. 

### Class

The original `C` interfaces of the library provide functions for procedure oriented programming, and the official page offers a comprehensive documentations for them, which can be found [here](https://gmplib.org/manual/index).


The `C++` support wraps up the data in an object oriented fashion and defines arithmetic operations. Thus, for `C++` users this is the best option without specific requirements. All `C++` classes are defined in header file `gmpxx.h`

There is [one chapter](https://gmplib.org/manual/C_002b_002b-Class-Interface) for the `C++` support in the documentation.



## Example


The example exhibits via code how to evaluate arithmetic expression in string format (e.g. `( 1 + 2 )`).

### operation

- `+`
- `-`
- `*`
- `//` integer division, round by floor (towards -infinite)
- `<<^` left shift first number by 13, then XOR operation


### expression

- `int`
- ([`expr` | `int`] `op` [`expr` | `int`])

The rule dictates that only possible error is *divided by zero*. If the error ever occurs, the program should catch it and break the current loop (and wait for next expression).


### parse symbol (helper class)

Iterates over the entire expression and return `string_view` of each symbol.

Assume spaces between all symbols to simplify parsing.

```cpp
#include <string>
#include <stack>
#include <string_view>
#include <charconv>
#include <iostream>
#include <gmpxx.h>


class SymbolIterator {
  private:
    const char* w;

  public:
    SymbolIterator(const std::string& expr): w(expr.data()) { }

    bool hasNext() const { return *w != '\0'; }

    std::string_view getNext() {
      if (hasNext()) {
        while (*w == ' ')
          ++w;

        // in case trailing spaces exist
        if (hasNext()) {
          const char* ptr = w;
          while ((*ptr != '\0') && (*ptr != ' '))
            ++ptr;

          std::string_view sv(w, ptr - w);

          w = ptr;

          return sv;
        }
      }

      return {};
    }
};
```


### main

BigInt type in GMP is `mpz_class`.

Algorithm

1. Push `mpz_class` (numbers) and `char` (operators) in two separate stacks
2. Pop TWO `mpz_class` and ONE `char` once a right parenthesis is encountered
3. Compute and push the results (`mpz_class`) back to the stack
4. Loop until the end of expression

```cpp
int main() {

  std::string expr;

  while (getline(std::cin, expr)) {
    SymbolIterator iter(expr);

    std::stack<mpz_class> s_int;
    std::stack<char> s_op;
    while (iter.hasNext()) {
      std::string_view sv = iter.getNext();

      // right parenthesis
      if (sv[0] == ')') {

        // BigInt
        mpz_class num2 = std::move(s_int.top());
        s_int.pop();
        
        char op = s_op.top();
        s_op.pop();

        mpz_class num1 = std::move(s_int.top());
        s_int.pop();
          
        try {
          switch (op) {
            case '+':
              num1 += num2; break;
            case '-':
              num1 -= num2; break;
            case '*':
              num1 *= num2; break;
            case '/':
              if (num2 == 0) 
                throw std::runtime_error("Error: divide by zero!");
              else
                num1 /= num2; // truncate to zero
              break;
            case '<':
              (num1 <<= 13) ^= num2; break;
            default:
              abort();
          }
        }
        catch (std::runtime_error& e) {
          std::cout << e.what() << std::endl;
          break;
        }

        s_int.push(std::move(num1));
      }
      // operator, left parenthesis. push only operators
      else if ((sv.size() == 1 && !isdigit(sv[0])) || (sv[0] == '<') || (sv[0] == '/')) {
        if (sv[0] != '(')
          s_op.push(sv[0]);
      }
      else {
        int num;
        auto result = std::from_chars(sv.data(), sv.data() + sv.size(), num);

        if (result.ec != std::errc::invalid_argument)
          s_int.emplace(num);
        else
          abort();
      }
    }

    if (s_int.size() == 1)
      std::cout << s_int.top() << std::endl;
  }

  return 0;
}
```

Then compile the *example.cpp* file by the following command. C++17 is enabled to support `string_view` feature.


```bash
clang++ -std=c++17 example.cpp -o a.out -lgmp -lgmpxx

# execute and input test case 1 and return result
./a.out
( ( ( ( 5 * ( 213456743 * 2134456612 ) ) - 10 ) // -3 ) <<^ 2 )
-6220651949702276628478
```
