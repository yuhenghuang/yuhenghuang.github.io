---
title: Mathematic Derivation of An Approach of Computing Multivariate Normal Density
top: false
cover: false
toc: true
mathjax: true
date: 2020-07-29 20:42:43
password:
summary:
tags:
  - Mathematics
  - Armadillo
categories: Algorithm
---

This post gives mathematic explanations for the algorithm on the [Rcpp example site](https://gallery.rcpp.org/articles/dmvnorm_arma/).


The density function for $\underset{k\times1}{X} \sim N(\mu, \Sigma)$ is
$$f(x) = \frac{\exp(-\frac{1}{2}(x-\mu)^T\Sigma^{-1}(x-\mu))}{\sqrt{(2\pi)^k|\Sigma|}}$$


We follow the algorithm in the [.cpp](https://github.com/yuhenghuang/Rcpp/blob/master/Multivariate_Normal_Density.cpp) file to compute the density of $\underset{n \times k}{x}$ for each row and return their density in a vector of size $n$ numerically.

To simplify the computation, instead of the original form, we choose to compute $\log\left(f\left(x\right)\right)$.

$$\log\left(f\left(x\right)\right) = 
-\frac{1}{2}(\Sigma^{-\frac{1}{2}}(x-\mu))^T(\Sigma^{-\frac{1}{2}}(x-\mu))
-\frac{k}{2}\log(2\pi) + \frac{1}{2}log(|\Sigma|^{-1})$$

, where $\Sigma^{-\frac{1}{2}} = L$ such that $\Sigma = L^TL$. <sup>*this definition differs from [Wikipedia](https://en.wikipedia.org/wiki/Cholesky_decomposition) whereas follows [Armadillo](http://arma.sourceforge.net/docs.html#chol), thus $L$ is an upper triangular matrix here.</sup>

### Compute $L^{-1}$, or $\Sigma^{-\frac{1}{2}}$

```cpp
const arma::mat rooti = arma::trimatu(arma::chol(sigma)).i();
```
the variable name `rooti` exactly comes from **inverse** and **root square** of the variance-covariance matrix $\Sigma$.

### Compute $\frac{1}{2}\log(|\Sigma|^{-1})$ and $-\frac{k}{2}\log(2\pi)$

$$|\Sigma|^{-1} = |\Sigma^{-1}| = |(L^TL)^{-1}| = |L^{-1}||(L^T)^{-1}|$$

As $L$ is a triangular matrix, $|L^{-1}|\equiv|(L^T)^{-1}|$ and its determinant is the product of its diagonal elements.

The second constant term only consists of scalars.

```cpp
static const double log2pi = std::log(2. * M_PI);
using arma::uword;

const uword n = x.n_rows, k = x.n_cols;
arma::vec out(n);

const double rootisum = arma::sum(log(rooti.diag())),
             constants = - double(k) / 2. * log2pi,
             other_terms = rootisum + constants;
```


### The first term

* Compute $z = \underset{1 \times k}{x} - \mu$
* Update $z := zL^{-1}$
* Compute $-\frac{1}{2}zz^T$

```cpp
void inplace_trimat_mul(arma::rowvec &x, const arma::mat &trimat) {
  const arma::uword n = trimat.n_cols;

  // be wary of unsigned int...
  for (arma::uword j=n; j-->0;) {
    double temp = 0.;
    for (arma::uword i=0; i<=j; ++i)
      temp += trimat(i, j) * x[i];
    x[j] = temp;
  }
}

arma::rowvec z;
for (uword i=0; i<n; ++i) {
  z = x.row(i) - mu;
  inplace_trimat_mul(z, rooti);
  out(i) = other_terms - 0.5 * arma::dot(z, z);
}
```