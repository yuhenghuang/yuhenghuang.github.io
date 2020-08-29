---
title: Clustered Standard Errors in Linear Regression Model
top: false
cover: false
toc: true
mathjax: true
date: 2020-08-24 17:04:29
password:
summary:
tags:
  - Mathematics
  - Armadillo
categories: Algorithm
---


Source code can be found [here](https://github.com/yuhenghuang/Rcpp/blob/master/ClusteredSE.cpp).


### Problem Setting

Consider a model of

$$ y_i = x_i'\beta + \epsilon_i $$

, where $y_i$ and $'\epsilon_i$ are scalars, $x_i$ and $\beta$ are both $k\times1$ vectors for $i = 1, 2, 3, ..., n$.

Other assumptions include

* $\mathbb{E}\left[\epsilon_i | x_i\right] = 0$

* $\mathcal{Var}\left(\epsilon_i\right) = \sigma^2$

* $\mathcal{Cov}\left(\epsilon_i, \epsilon_j\right) = 0$ for $i \neq j$

Actually the previous assumptions indicate the *i.i.d* property of the error term $\epsilon_i$.

All those assumptions are way too strong for real world data, thus we need to relax them to a certain extent. Introduce a new variable $\{g_i\}^n_{i=1}$ that indicates the group of observations.

New assumptions are

* $\mathbb{E}\left[\epsilon_i | x_i\right] = 0$

* $\mathcal{Var}\left(\epsilon_i\right) = \sigma_i^2$

* $\mathcal{Cov}\left(\epsilon_i, \epsilon_j\right) = 0$ for $g_i \neq g_j$

The former homoskedasticity is now heteroscedasticity, and correlation of error terms within group is allowed.

### Derivation and Inference

For convenience, a matrix formula of the problem is presented as 

$$ \underset{n \times 1}{Y} = \underset{n \times k}{X}\beta + \underset{n \times 1}{\epsilon} $$ (1)

with $\underset{n \times 1}{G}$.


As the assumptions do not affect the result of OLS estimator, the estimate of $\beta$ is

$$\hat{\beta} = (X'X)^{-1}X'Y$$ (2)

and the estimate of $\epsilon$ is

$$e = Y - X\hat{\beta}$$ (3)

The variance of $\hat{\beta}$ can be shown to be

$$\mathcal{Var}(\hat{\beta}) = (X'X)^{-1}X'\epsilon \epsilon' X(X'X)^{-1}$$

Using the assumption $\mathbb{E}\left[\epsilon_i | x_i\right] = 0$, we can derive that $\mathcal{Var}(\epsilon) = \epsilon \epsilon'$. Denote $\mathcal{Var}(\epsilon)$ as $\underset{n \times n}{\Omega}$,

$$\mathcal{Var}(\hat{\beta}) = (X'X)^{-1}X' \Omega X(X'X)^{-1}$$

Applying the rest of assumption we can obtain

$$\mathcal{Var}(\hat{\beta}) = \left(X'X\right)^{-1}\sum\limits^g\left[X_g'\epsilon_g\epsilon_g'X_g\right]\left(X'X\right)^{-1}$$

, where $\underset{n_g \times k}{X_g}$ and $\underset{n_g \times 1}{\epsilon_g}$ are the $X$ and $\epsilon$ in group $g$, and $n_g$ is the sample size of the group.

Since $\epsilon$ is unknown to us, in the computation it is replaced by its estimate $e$.


### Algorithm

1. Precompute $(X'X)^{-1}$

2. Use bucket sort to construct $X_g$ and $\epsilon_g$ correctly in $O(2n)$ time.

  * the first loop is to determine $n_g$ for the purpose of allocating places for $X_g$ and $\epsilon_g$
  * the second loop put $X_g$ and $\epsilon_g$ in the allocated place correctly in terms of value and order.

3. Compute $\sum\limits^g\left[X_g'\epsilon_g\epsilon_g'X_g\right]$

4. Compute $\mathcal{Var}(\hat{\beta})$


For more information, please see the source code link in the beginning of the post.