---
title: R - dplyr::rename automation
top: false
cover: false
toc: true
mathjax: false
date: 2020-05-16 10:31:47
password:
summary:
tags: 
  - Automation
  - Tidyverse
categories: R
---

The introduction of *tidyverse* library has drastically changed the way of data processing in R. Its consistent syntaxes and straightforward pipeline coding style increase the readability of codes to a higher level, thus even beginners of R language could learn basic skills of data processing at a rapid pace. Certainly, there are downsides of this library. Its memory management is obscure to users, which cause a large amount of heavy, unnecessary copy operations implicitly. Discussing this feature is beyond the scope of the article, but in my opinion if one's not dealing with extremely large data set, trading a little more electricity/time for convenience is a pretty good deal. At the end of the day, nobody wants to spend hours of searching on the Internet to improve seconds of runtime.


### dplyr::rename

Compare to the *pandas* library in *Python*, *tidyverse* hides many tedious code block inside its functions and classes. Like today's topic, the `rename` function. The `rename` function, as its function name suggests, it changes columns' names given dataframe and the names of new and old columns. 


```r
library(tidyverse)
# library(dplyr)

# data frame with one column "A"
df <- data.frame(A = 1)

# rename "A" to "B"
# either of the following ways works

df %>%
  rename("B" = "A")

df %>%
  rename(B = "A")

df %>%
  rename(B = A)
```

<sup>* pipe operator, aka `%>%`, pass LHS to the first parameter of RHS function<sup>

### rename given string variable

Someone who knows other general-purpose programming languages might wonder how can the second and third ways work when the column names are supposed to be a *string*. I cannot answer the question myself and reading the source codes of a high-level language is definitely a pain in the ass. But if we accept this syntax at first, it is actually very convenient and readable even without quotes, because now everyone knows the parameters in `rename` (of course in its general version `select`) are all column names.

Nothing is all made of 'good'. This simplification has one greatest downside. It is in the automation of program.

In a general process of an automation, parameters are passed among programs by strings and other forms of data types, while inside a script, the inputs are translated into variables. Under the setting, what we want to do next is `rename` column names of dataframe given a string variable.

This is not as simple as it looks like. See the following attempts.

In the first time, "A" column is renamed to "C". This is definitely what we don't want.

Someone might think of a seemingly right answer: we haven't yet extracted the value of *C*. But the unquoted version fails as well.

```r
df <- data.frame(A = 1)

C = "B"
df %>%
  rename(C = A)
# output:
#   C
# 1 1

df %>%
  rename(!!C = A)
# output:
# Unexpected '=' in "df %>%rename(!!C ="
```
<sup>* `!!` is an unquotation operator<sup>

### assignment operator(s)

The error message shows the R interpreter failed exactly at the `=` right after `!!C`. This is the clue for the right answer but there is a long way to arrive at it with the help of Google.

Here is the write way to `rename` dataframe columns given *string* variable.

```r
df <- data.frame(A = 1)

C = "B"
df %>%
  rename(!!C := A)
# output:
#   B
# 1 1
```
<sup>* `!=` is one of assignment operators in R<sup>

The official documents says: 
> The `:=` is mainly useful to unquote names. Unlike `=`, it supports `!!` on its LHS

It might be unknown to many people, but R is an old language. Though we mostly use it for statistical analysis, we cannot deny that R has accumulated a very large set of operations and syntaxes for general programming. This is just one glimpse of R's long history.

If you search the assignment operators of R on the Internet, you'll only find more than what you have expected. And thinking about its capability of overloading operators, R language looks more and more like an uncharted water to me, when I think I've mastered sufficient R skills.