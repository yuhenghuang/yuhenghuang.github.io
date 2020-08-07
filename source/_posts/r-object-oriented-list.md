---
title: An Object-Oriented List in R
top: false
cover: false
toc: true
mathjax: false
date: 2020-08-01 14:16:34
password:
summary:
tags: Object-Oriented
categories: R
---

In *R*, basically operations follow the principle of so-called **copy-on-modify semantics**. That means the operations modify the copy of a variable/object instead of the variable/object itself. The feature trades convenience for performance. In practice, most of *R* users are from disciplines other than computer science, and it's less likely they prioritize performance over convenience. As a result we could say this is for the benefit of the majority of the community.

While in my opinion, especially when Data Science is being popular year by year, the role that *R* plays as one of the two major languages for DS (the other is *Python*) has been shifting from a statistical tool to a general purpose programming language. In *Java* and *Python*, an Object passed to a function as an argument will be treated as reference without implicitly copying it, whereas *R* copies most types of its objects in this case, following the **copy-on-modify semantics**.

There are some newly emerged OO system that works under **reference semantics** just like other mainstream languages. The `R6class` and `RefClass` are both good choices in general. But they are out of the options for the purpose of the post because they does not support overriding `$`, which is one of the most important operator for a list in *R*. For `R6class` and `RefClass`, `$` is like the `.` in *Java* and *Python* used after an object to access the methods of the object.

Given the two main requirements of this new OO list

* The feature of **reference semantics**
* Able to overriding `$` and of course `[[`

The choice in the post is `R.oo`.


### Constructor

Assuming the class name of the list is `List`, the constructor is like

```r
library(R.oo)
library(rlang)

setConstructorS3("List", function(...) {
  extend(
    Object(),
    "List",
    .ls = list(...)
  )
})
```

### Override `$` and `[[`

My suggestion is to override `[[` first and call it by `$`. Because the argument of `[[` is `character` while `$` is `expression`, there are cases we cannot avoid using `eval()` in the method. That part involves [meta-programming](https://adv-r.hadley.nz/metaprogramming.html) which is not a familiar field for non-developers.

The only thing we do inside the `$` is transform the input argument `name` from `expression` to `string`, then call `[[` to do the rest of the job. The example only shows one way to do the transformation.

```r
setMethodS3("$", "List", function(this, name) {
  key <- as_string(ensym(name))
  return(this[[key]])
})
```

Inside the `[[` there are two cases, accessing the members of the object or accessing the inner `list`. As the Object itself can be treated like an environment, the logic is searching for the variable in the object first. If it is a member of the object, return it. Otherwise, return the element in the list.


```r
setMethodS3("[[", "List", function(this, name) {
  envir = attr(this, ".env")
  if (exists(name, envir = envir, inherits = FALSE)) {
    return(get(name, envir = envir, inherits = FALSE))
  }
  else {
    return(this[[".ls"]][[name, exact=TRUE]])
  }
})
```

The code explains itself fairly well except this line 

```r
    # ---
    return(this[[".ls"]][[name]])
    # ---
```

.The functional form of this line is actually

```r
`[[`(`[[.List`(this, ".ls"), name)
```

, which means the first case of `if else` is already being used in this function.


### Override `$<-` and `[[<-`

A hint here is that in the [source code](https://github.com/HenrikBengtsson/R.oo/blob/develop/R/050.Object.R#L1825) of `R.oo`, the `[[<-` of the base object `Object()` has only one line.

```r
setMethodS3("[[<-", "Object", function(this, name, value) {
  do.call(`$<-`, args = list(this, name, value))
})
```

That is to say we only need to override `$<-` to accomplish the assignment functionality of `list`.

```r
setMethodS3("$<-", "List", function(this, name, value) {
  key <- as_string(ensym(name))
  envir <- attr(this, ".env")
  if (exists(key, envir = envir, inherits = FALSE)) {
    assign(key, value, envir = envir)
  }
  else {
    this[[".ls"]][[key]] <- value
  }
  return(invisible(this))
})
```

The logic of the `if else` is actually the same as `[[` in previous section. In the same way we rewrite the line of assignment to `list` into functional form.

```r
    # ---
    this[[".ls"]][[key]] <- value
    # ---
    # identical to
    # ---
    `[[<-.List`(this, 
                ".ls", 
                `[[<-`(`[[.List`(this, ".ls"), 
                       key, 
                       value))
    # ---
```

Unexpectedly, there are two assignments rather than one. This is due to the **copy-on-modify semantics** feature of `list`, which does not allow in-place modification.


### Autocomplete

To let the autocomplete triggered by `$` work just like `list`, we need one (or two) more method(s).

```r
setMethodS3(".DollarNames", "List", function(this, pattern="", ...) {
  grep(pattern, names(this[[".ls"]]), value = TRUE)
})

setMethodS3("names", "List", function(this, ...) {
  return(names(this[[".ls"]]))
})
```

Actually the second function is sufficient to do the job in RStudio, and the first method is designed for *R* GUI. By the time you're reading the post, the first method should have already been supported by source editor of RStudio.


### Embed API in `$` and `[[`

A good practice is encapsulating `connection` or `access_key` in the Object and call this `List` to retrieve the information implicitly. And as this `List` uses **reference semantics**, not only the modifications made inside/outside of a function will be reflected, but also *R* will not copy anything when passing it as an argument. This feature is particularly helpful if the `connection` or anything inside the object shall not be copied.

Here is a template for this usage. `API_FUNCTION` can be arbitrary function.

```r
setMethodS3("[[", "List", function(this, name) {
  envir = attr(this, ".env")
  if (exists(name, envir = envir, inherits = FALSE)) {
    return(get(name, envir = envir, inherits = FALSE))
  }
  else {
    if (is.null(this[[".ls"]][[name]])) {
      # ---
      this[[".ls"]][[name]] = API_FUNCTION(name)
      # ---
    }
    return(this[[".ls"]][[name, exact=TRUE]])
  }
})
```