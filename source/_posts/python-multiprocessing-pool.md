---
title: An Example of Library Multiprocessing in Python
top: false
cover: false
toc: true
mathjax: false
date: 2020-06-27 10:38:31
password:
summary:
tags: Parallel Computing
categories: Python
---

By default, `python` programs only utilize one CPU. This is not much of a problem for most cases, but when it comes to heavy computations, whether or not using computational resources to their fullest could make a big difference.

This post exhibits a simple example of the library `multiprocessing`'s `Pool` module.

### Settings

For *Windows* users to test the code in *.ipynb*, the functions **must** be imported from other *.py* scripts. e.g.

```python
# in file utils.py
def f(x):
  return x*x

# in main file
from utils import *
from multiprocessing import Pool

with Pool(processes=2) as pool:
  pool.map(f, [1,2,3])

```

It seems that the default settings for MacOS has changed since *Python* 3.8. There are chances MacOS users must follow this suggestion as well. See the [link](https://docs.python.org/3/library/multiprocessing.html#contexts-and-start-methods) for details.

### pool.map(func, iterable, chunksize=None)

The simplest way of parallelizing computing given arguments in a list-like object.

The performance of the `pool.map()` is very close to the built-in `map()` in *Python*, except that it only accepts **ONE** argument. A trivial example is showed in previous section.

For the function to accept multiple arguments, one possible solution is to wrap the original function by a function that accepts one `dict` object as argument. Then unpack the arguments inside of the wrapper as pass them to the original function.

```python
def g(a, b):
  return a+b

def h(args):
  a = args['a']
  b = args['b']
  return g(a, b)

with Pool(processes=2) as pool:
  # set proper chunksize for better performance
  ret = pool.map(h, [{'a':1, 'b':2}, {'a':3, 'b':4}], chunksize=1)
  print(ret)

```

There is also an iterator version of the function, which is `imap()` and its variant `imap_unordered()`. They are highly memory efficient, as we expect from iterators, and the latter does not maintain the order of the input list-like object.

### pool.starmap(func, iterable, chunksize=None)

The method `pool.starmap()` is recently added to `Pool`, to tackle the multiple arguments problem. The input arguments are packed in `tuple` in the iterable parameter. For the same *g(a, b)* function previously, instead of wrapping up parameters in *dict* manually, it is possible to do the following.

```python
with Pool(processes=2) as pool:
  ret = pool.starmap(g, [(1, 2), (3, 4)])
  print(ret)
  # print 

```

### functools.partial(func, *args, **kwargs)

The `partial()` function (adaptor) help bind unchanged parameters to the input function. It should be quite useful in practice along with `multiprocessing`.

```python
from functools import partial

with Pool(process=2) as pool:
  # bind 2 to the parameter a
  func = partial(g, 2)
  ret = pool.map(func, range(3))
  print(ret)
  # print 2, 3, 4

  # bind 3 to the parameter b
  func = partial(g, b=3)
  res = pool.map(func, range(3))
  print(ret)
  # print 3, 4, 5
```

There is some ambiguous behavior of `partial` and `Pool`. This is caused by the `pool.map()` which always passes the argument by `*args`.


```python
def sub(a, b):
  return a-b

with Pool(processes=2) as pool:
  func = partial(sub, a=2)
  print(pool.map(func, range(3)))

```
An error will occur. 
> sub() got multiple values for argument 'a'


### An alternative: concurrent.futures.ProcessPoolExecuter

The `ProcessPoolExecuter` from `concurrent.futures` acts similar to `multiprocessing`, and contains less methods.

The `map` method does NOT execute lazily even if it returns a generator.

```python
from concurrent.futures import ProcessPoolExecutor

with ProcessPoolExecutor() as pool:
  ret = pool.map(f, range(3))
  print(list(ret))
```

### References

[Official Documents](https://docs.python.org/3/library/multiprocessing.html)