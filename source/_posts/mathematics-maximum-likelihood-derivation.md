---
title: Informal Derivation of Properties of MLE
top: false
cover: false
toc: true
mathjax: true
date: 2020-12-09 21:42:35
password:
summary:
tags:
  - Mathematics
categories: Proof
---


Suppose $L(x;\theta)$ is the log-likelihood function, where $x$ is the data and $\theta$ is the parameter to be estimated, this post exhibits an informal derivation of properties of Maximum Likelihood Estimator.


Given the maximization problem,


$$0 = \frac{1}{n} \sum^n_{i=1} \left.\frac{\partial L(x_i;\theta)}{\partial \theta} \right|_{\theta = \hat{\theta}}$$


is the condition where $\hat{\theta}$ is the optimum.


By Taylor expansion for some $\bar{\theta}\in [\theta _0, \hat{\theta}]$,


$$0 = \frac{1}{n} \sum^n_{i=1} \left.\frac{\partial L(x_i;\theta)}{\partial \theta} \right|_{\theta = \theta_0}$$

$$+(\hat{\theta} - \theta_0) \frac{1}{n} \sum^n_{i=1} \left.\frac{\partial^2 L(x_i;\theta)}{\partial \theta \partial \theta'} \right|_{\theta = \bar{\theta}}$$


Under some regular assumptions/conditions, the following holds


$$\frac{1}{n} \sum^n_{i=1} \left.\frac{\partial L(x_i;\theta)}{\partial \theta} \right|_{\theta = \theta_0}$$ 

$$\underset{p}{\to} \mathbb{E}\left( \left.\frac{\partial L(x;\theta)}{\partial \theta} \right|_{\theta = \theta_0} \right) = 0$$

. The last equality comes from the maximization settings. Further three other conditions can be proved.

$$\hat{\theta}\underset{p}{\to}\theta_0$$

$$\bar{\theta}\underset{p}{\to}\theta_0$$


$$\frac{1}{n} \sum^n_{i=1} \left.\frac{\partial^2 L(x_i;\theta)}{\partial \theta \partial \theta'} \right|_{\theta = \bar{\theta}}$$ 

$$\underset{p}{\to} \mathbb{E}\left( \left.\frac{\partial^2 L(x_i;\theta)}{\partial \theta \partial \theta'} \right|_{\theta = \theta_0} \right)$$


Denote 

$$H(\theta) \equiv \mathbb{E}\left( \left.\frac{\partial^2 L(x_i;\theta)}{\partial \theta \partial \theta'} \right|_{\theta} \right)$$

, by applying CLT


$$\sqrt{n}(\hat{\theta} - \theta_0) \underset{d}{\to} \mathrm{N}\left(0, H(\theta_0)^{-1} \mathbb{E}\left( \left.\frac{\partial L(x;\theta)}{\partial \theta} \frac{\partial L(x;\theta)}{\partial \theta'}  \right|_{\theta = \theta_0} \right) H(\theta_0)^{-1}\right)$$

. The variance part could be simplified by proving


$$H(\theta_0) = -\mathbb{E}\left( \left.\frac{\partial L(x;\theta)}{\partial \theta} \frac{\partial L(x;\theta)}{\partial \theta'}  \right|_{\theta = \theta_0} \right)$$
.


$$H(\theta) = \int_\Omega \left[\frac{1}{f(x;\theta)}\frac{\partial^2 f(x;\theta)}{\partial\theta\partial\theta'} - \frac{1}{f^2(x;\theta)}\frac{\partial f(x;\theta)}{\partial \theta}\frac{\partial f(x;\theta)}{\partial \theta'}\right]f(x;\theta)\mathrm{d}x$$


Since the first order partial derivative of $L(x;\theta)$ w.r.t. $\theta$ is

$$\frac{\partial L(x;\theta)}{\partial \theta} = \frac{1}{f(x;\theta)}\frac{\partial f(x;\theta)}{\partial \theta}$$

and by the properties of density function,

$$1 = \int_\Omega f(x;\theta)\mathrm{d}x$$

$$0 = \int_\Omega \frac{\partial f(x;\theta)}{\partial \theta}\mathrm{d}x$$

$$0 = \int_\Omega \frac{\partial^2 f(x;\theta)}{\partial\theta\partial\theta'}\mathrm{d}x$$

, where $\Omega$ is the support of $x$.

Then we have

$$H(\theta) = \int_\Omega \frac{\partial^2 f(x;\theta)}{\partial\theta\partial\theta'}\mathrm{d}x - \int_\Omega\frac{1}{f^2(x;\theta)}\frac{\partial f(x;\theta)}{\partial \theta}\frac{\partial f(x;\theta)}{\partial \theta'}f(x;\theta)\mathrm{d}x$$


$$H(\theta_0) = 0 - \mathbb{E}\left( \left.\frac{\partial L(x;\theta)}{\partial \theta} \frac{\partial L(x;\theta)}{\partial \theta'}  \right|_{\theta = \theta_0} \right)$$

.

In the end,

$$\sqrt{n}(\hat{\theta} - \theta_0) \underset{d}{\to} \mathrm{N}\left(0, -H(\theta_0)^{-1}\right)$$
