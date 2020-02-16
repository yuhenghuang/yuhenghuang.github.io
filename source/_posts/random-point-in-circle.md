---
title: Draw a random point in a circle uniformly 
date: 2020-02-15 13:16:03
tags: Mathematics
category: Algorithm
mathjax: true
summary:
top: false
cover: true
---


## Introduction   

This post is inspired by the problem from [LeetCode](https://leetcode.com/problems/generate-random-point-in-a-circle/), where one is asked to write a program to draw any point uniformly in a circle given its center coordinate and radius. The natural approach would be sampling two uniform random variables, which form a square geometrically, and rejecting them if they happen to fall out of the circle. This algorithm is high effecient in terms of expectation as the rejection area is only about \( \frac{1}{4} \) of the square.

The time and space complexity are now both \( Op(1) \), not the general \( O(1) \). The uncertainty is not a big problem in the context, but it still leaves a problem unsolved, that is, is there any other approach that could tackle the problem in a deterministic way?

## Sample rejection approach


The most straight forward algorithm would be sampling rejection.
Following codes briefly explain the algorithem.

A class is initialized with the information of the circle, and a random point is produced after the last draw falls into the circle.


```java
public class RandomPointInCircle {
	double r, xC, yC;
	public RandomPointInCircle(double radius, double x_center, double y_center) {
		r=radius;
		xC=x_center;
		yC=y_center;
	}
	
	public double[] randPoint() {
	while(True) {
	//generate x and y by (Math.random()-0.5)*2
	//if they are inside the unit circle, center at (0,0) and radius=1, break
	//else continue
	}
	return new double[]{xC+r*x, yC+r*y};
	}
}
```

The process is quite clear, but a little bit slower than an \( O(1) \) algorithm.


## Mathematical approaches

### Catesian coordinates

Why don't we try to transform the problem into a mathematical problem.

Let's start from the conditions.

To simply the notation, The circle is normalized to be centered at \( (0,0) \) with radius being 1.Thus, \( x \) and \( y \) lie between -1 and 1.

Uniformality means the probability of draw any point is identical, that is, probability density function \( f(x,y) \) is a constant on the support.

\\[ 1 = \iint_\Omega f(x,y) \mathrm{d}x\mathrm{d}y \\]
\\[ 1 = f(x,y) \iint_\Omega \mathrm{d}x\mathrm{d}y \\]
as \( f(x,y) \) is a constant.

Since the integration is the area of the circle, we get
\\[ f(x, y)=\frac{1}{\pi} \\]

Generally in an algorithmic setting, one can only generate random variables uniformly distributed between 0 and 1 independently. 

And obviously \( x \) and \( y \) are not independent of each other. At least the support is not independent from first look.

The following theorem explains the relationship between joint distribution and marginal distribution.

\\[ f(x, y)=f(x) \times f(y|x) \\]

LHS is constant and has already been obtained, and the support of \( f(y|x) \) can also be derived as \( \[-\sqrt{1-x^2}, \sqrt{1-x^2} \] \).

Let's assume that \( f(y|x) \) follows a uniform distribution on its support. This would not hurt us as long it does not violate the conditions.

Proving condition is out of the scope of the post, but the assumption can be proved to be true by contradiction.

Given this new condition, the density of \( x \) is 

\\[ f(x) = \frac{2\sqrt{1-x^2}}{\pi} \\]

Thus its cumulative density function is derived by integration.

\\[ F(x) = \frac{1}{\pi}(x\sqrt{1-x^2} + \mathrm{arcsin}(x)) + \frac{1}{2} \\]

Now we can use the [inverse transform sampling-method](https://en.wikipedia.org/wiki/Random_number_generation) to generate \( x \) from a uniform variable between 0 and 1.

It is a pity that the inverse of \( F(x) \) does not have a close form, but we can still find it using binary search in a reasonably short time.

The code is exhibited as following.

```java
public class RandomPointInCircle {
	double r, xC, yC;
	public RandomPointInCircle(double radius, double x_center, double y_center) {
		r=radius;
		xC=x_center;
		yC=y_center;
	}
	
  public double[] randPoint_xy() {
    double tempX=Math.random()-0.5, tempY=(Math.random()-0.5)*2;
    double x=binarySearch(tempX);
    double y=Math.sqrt(1-x*x)*tempY;
    double[] return new double[]{xC+r*x, yC+r*y};
  }

  private double binarySearch(double target) {
    double left=-1, right=1, err=1, x=0;
    while (Math.abs(err)>1e-8) {
      x=(left+right)/2;
      err=cdf(x)-target;
      if (err>0)
        right=x;
      else
        left=x;
    }
    return x;
  }

  private double cdf(double x) {
    return  (x*Math.sqrt(1-x*x)+Math.asin(x))/Math.PI;
  }
```


### Polar coordinates

The only downside of the cartesian coordinate approach is the binary search part. Because of it, the time complexity remains to be \( Op(1) \), though now space complexity is \( O(1) \).

A natural thought would be to find a close-form CDF. But how?

In this subsection, we challenge the problem by another representation of circles.

Let's represent any point  in a circle by its angle to x-axis and distance to \( (0,0) \). Using the same normalization, the angle \( \theta \) and distance \( d \) should be transformed to \( (x,y) \) by
\\[ x = \mathrm{cos}(\theta)d \\]
\\[ y = \mathrm{sin}(\theta)d \\]
, where \( \theta \in \[-\pi, \pi \] \) and \( d \in  \[0, 1\] \)

And the next equation holds on their support
\\[  f(\theta, d) = f(x, y)|J| \\]
, where
\\[ J = \begin{bmatrix} \frac{\partial x}{\partial \theta} & \frac{\partial y}{\partial \theta} \\\\ \frac{\partial x}{\partial d} & \frac{\partial y}{\partial d} \end{bmatrix} \\]

and \( |\cdot| \) represents the determinant and absolute operation.

Thus,
\\[ |J| = | - \mathrm{sin}^2(\theta)d - \mathrm{cos}^2(\theta)d | = d \\]

Now we have
\\[  f(\theta, d) = \frac{1}{\pi} \times d \\]

Like in previous subsection, \( f(\theta) \) follows a uniform distribution by contradiction, but the conditional distribution of \( d \) is simpler
\\[ f(d|\theta)=2d=f(d) \\]
Moreover, the CDF of \( d \) is 
\\[ F(d) = d^2 \\]
, which just has a close-form inverse function.

The idea furtherly simplified the binary search part.

```java
public class RandomPointInCircle {
	double r, xC, yC;
	public RandomPointInCircle(double radius, double x_center, double y_center) {
		r=radius;
		xC=x_center;
		yC=y_center;
	}
	
	  public double[] randPoint() {
    double theta=2*Math.PI*(Math.random()-0.5), dist=Math.sqrt(Math.random());
    double x=Math.cos(theta)*dist, y=Math.sin(theta)*dist;
    return new double[]{xC+r*x, yC+r*y};
  }
}
```

Finally we removed all the \( p \) from time complexity and space complexity. Dragon is defeated!
