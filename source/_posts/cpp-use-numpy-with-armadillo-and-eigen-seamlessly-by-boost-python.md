---
title: Use Numpy with Armadillo and Eigen Seamlessly by Boost.Python
top: false
cover: false
toc: true
mathjax: false
date: 2021-02-15 12:03:10
password:
summary:
tags:
  - Armadillo
  - Eigen
  - Boost
  - Numpy
categories: cpp
---


`Numpy` is probably the most important library in *python* for scientific computing. As the main data structure of `Numpy`, `numpy.ndarray` is widely used in the library everywhere.

As a script language, *python* is naturally slow in the process of typing at runtime. Although the implementation of `Numpy` is written by *C*, it requires more than enough effort to conquer the slow-down caused by typing in pure *python* part. If there are functions that are not implemented by `Numpy`, a user-defined one is destined to be fairly slow.

A straightforward solution is to rewrite the slowest part by compiled languages. 

`Numba` is an endeavour that is working on pure *python* code and to avoid a large amount of modification to original code while targeting the goal. But in practice, there are many implicit rules that one must bear in mind when coding `Numba` functions. And due to its flexibility, it still sacrificed the speed-gain somehow. In some way `Numba` may be called a programming language that serves partial and limited purposes in specific context.

`Cython` and `CPython` are other alternatives. Those two are facing the same problems with `Numba`. Both requires the user to learn an almost brand new language with limited extensive usage.

The previous three approaches have a common obvious down side of too much effort at preparation stage.


In the post I'd like to introduce an approach by `Boost.Python` which seamlessly integrate *python* and *cpp*, including `Numpy`.

For those who with knowledge about *cpp*, it takes literally little effort to master this approach.


### Headers and Libraries

```cpp
#include <boost/python/numpy.hpp>
#include <eigen3/Eigen/Dense>
#include <armadillo>
#include <iostream>

namespace py = boost::python;
namespace np = boost::python::numpy;
```

Libraries are

* `Boost.Python` - `Numpy` is part of this library, [link](https://github.com/boostorg/python)
* `Eigen3` - header-only, [link](https://eigen.tuxfamily.org/dox/index.html)
* `armadillo` - [link](http://arma.sourceforge.net/)

The latter two are popular linear algebra libraries in *cpp*.
The installations are skipped in the post.

Test cases use 2D double arrays.

### Pass `numpy.ndarray` to *cpp*

- array **MUST** be transposed when passed to *cpp*, because `Numpy` adopts *C* order whereas the other two libraries have *Fortran* in the backend. 


* `Eigen::Map`

This data structure in `Eigen3` is specially used for obtaining data from other source.

Copying `Eigen::Map` does not lose the reference relationship, but the reference can not be transferred to `Eigen::Matrix` by any means.

```cpp
void read_ndarray(np::ndarray& arr) {
  // 2D, double ndarray

  Eigen::Map<Eigen::MatrixXd, Eigen::Unaligned> m(
    reinterpret_cast<double*>(arr.get_data()),
    arr.shape(1), arr.shape(0)
  );

  // this will change the original numpy.ndarray at position (1, 0)
  m(0, 1) = 3.1415926;
}
```

* `arma::mat`

`Armadillo` uses the same data structure to manage data from other sources as original matrices, though `arma::mat` actually has internal flags to mark the source of data.

See the [Source Code](https://gitlab.com/conradsnicta/armadillo-code/-/blob/10.2.x/include/armadillo_bits/Mat_bones.hpp#L37) for reference.

```cpp
void read_ndarray(np::ndarray& arr) {
  // 2D, double ndarray

  arma::mat m(
    reinterpret_cast<double*>(arr.get_data()),
    arr.shape(1), arr.shape(0),
    false, // copy?
    true // strictly keep the shape?
  );

  // this will change the original numpy.ndarray at position (1, 0)
  m(0, 1) = 3.1415926;
}
```


### Return `numpy.ndarray` from *cpp*

Copy the data to `numpy.ndarray` and return it. Transpose should also be done.

* `Eigen::Matrix`

```cpp
np::ndarray ret_ndarray() {
  Eigen::MatrixXd m(3, 2);
  m.fill(2.5);

  // shape and data type
  py::tuple shape = py::make_tuple(2, 3);
  np::dtype dt = np::dtype::get_builtin<double>();

  // create empty numpy ndarray
  np::ndarray out = np::empty(shape, dt);

  // copy data
  std::copy(
    m.data(), m.data() + 2 * 3
    reinterpret_cast<double*>(out.get_data())
  );

  return out;
}
```


* `arma::mat`

```cpp
np::ndarray ret_ndarray() {
  arma::mat m(3, 2, arma::fill::ones);

  // shape and data type
  py::tuple shape = py::make_tuple(2, 3);
  np::dtype dt = np::dtype::get_builtin<double>();

  // create empty numpy ndarray
  np::ndarray out = np::empty(shape, dt);

  // copy data
  std::copy(
    m.data(), m.end()
    reinterpret_cast<double*>(out.get_data())
  );

  return out;
}
```


### `numpy.ndarray` as reference to *cpp*

First construct a class to hold *cpp* data to be referred.

```cpp
class Foo {
  private:
    Eigen::MatrixXd m1;
    arma::mat m2;


  public:
    // m row, n col
    Foo(int m, int n): m1(n, m), m2(n, m, arma::fill::ones) {
      m1.fill(2.5);
    }

    void print_eigen() { std::cout << m2 << std::endl; }
    void print_arma() { std::cout << m1 << std::endl; }

    np::ndarray ref_eigen();
    np::ndarray ref_arma();
};
```

* `Eigen::Matrix`

```cpp
np::ndarray Foo::ref_eigen() {
  // still transpose...
  size_t m = m1.cols(), n = m1.rows();

  std::vector<size_t> shape = {m, n};
  std::vector<size_t> strides = {n * sizeof(double) , sizeof(double)};

  np::dtype dt = np::dtype::get_builtin<double>();

  // speculation: the last argument create a new python object to ensure that the GC works fine.
  return np::ndarray::from_data(m1.data(), dt, shape, strides, py::object());
}
```

Reference for [strides](https://numpy.org/doc/stable/reference/generated/numpy.ndarray.strides.html)


* `arma::mat`

```cpp
np::ndarray Foo::ref_arma() {
  size_t m = m2.n_cols, n = m2.n_rows;

  std::vector<size_t> shape = {m, n};
  std::vector<size_t> strides = {n * sizeof(double) , sizeof(double)};

  np::dtype dt = np::dtype::get_builtin<double>();

  return np::ndarray::from_data(m2.begin(), dt, shape, strides, py::object());
}
```

Try modifying the returned `numpy.ndarray` to see if the data in `Foo` change or not.


## Conclusion

* `numpy.ndarray` can be converted to `arma::mat` and `Eigen::Map` without copying data, literally passing by reference.

* It is convenient to copy data in `arma::mat` and `Eigen::Matrix` to a new `numpy.ndarray` and return it.

* `arma::mat` and `Eigen::Matrix` (or `Eigen::Map` occasionally) can be referred by `numpy.ndarray`, literally return by reference.

* All the conversions above must require TRANSPOSE because of the difference between *C* order and *Fortran* order.

* Caveats

- since `numpy.ndarray` is not friend with `arma::mat` and `Eigen::Matrix`, the memories are managed respectively. Thus there is no moving construction/assignment among them and sometimes unnecessary copy is unavoidable (Don't try to this unless the library authors have decided them to be friends).

- It's the user's responsibility to assert the data type and shape of `numpy.ndarray` at runtime.
