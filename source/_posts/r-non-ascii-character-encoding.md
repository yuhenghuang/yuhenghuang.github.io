---
title: R non-ascii character encoding
top: false
cover: false
toc: true
mathjax: true
date: 2020-05-03 22:54:26
summary:
tags: Character/String
categories: R
---

Although English has been a universal language for decades, tasks in real work are still dealing with other languages, which have more variations in characters. Of course, no one would naively expect them to be as simple as ascii characters.

To present those languages in computers smoothly, there are many standards of encoding. Some are international, where as others are designed only for limited number of languages. The encoding and decoding of languages thus became a crucial issue when one is dealing with non-English characters.

Wrong encoding and decoding will only lead to a **new** language on the monitor which no one could understand. Therefore, under most situations, the data should be encoded by international standards that won't be affected by the locale of the computer or other factors, if possible. This is the best practical way to prevent an invention of a new language from happening.

Today's example comes from a famous IDE for R development, [RStudio](https://rstudio.com/).

### Setting

I was working on some data containing Chinese characters in several columns. The data was encoded by an international standard called "utf-8". The goal of this job was to right-strip the regional characters from the variable so that it is comparable to another variable from a different data source.

### Trial

As usual, I loaded the libraries and switched locale to "chinese". And the basic operations of _dplyr_ comes next.

```r
library(openxlsx)
library(haven)
library(stringi)
library(tidyverse)

Sys.setlocale(locale = 'chinese')

df <- ## load df

df_new <- df %>%
  mutate(region = stri_replace_last_regex(region, '[区县镇市]$', ''))

```

The code was supposed to work logically, but nothing changed...

Then I tried to debug why this even happened. After excluding the logic mistakes, I found that the following command returned false.

```r
any(stri_detect_regex(df$region, '[区县镇市]$'))
#FALSE returned...
```

The confusing things is that all input character could be print properly in the console. That's why it took a long time to ascertain the hypothesis.

### Solution

The time I found out the source of the problem could be **encoding**, I started searching for ways to convert the regex patterns to "utf-8".

Here are two approaches.

```r
df_new <- df %>%
  mutate(region = stri_replace_last_regex(region, enc2utf8('[区县镇市]$'), ''))

# or
df_new <- df %>%
  mutate(region = stri_replace_last_regex(region, 
                                          stri_conv('[区县镇市]$', from='GB18030', to='utf-8'), ''))
```

The first approach works without extra parameters while the second one requires a certain level of knowledge about the encodings of the language.