---
layout:     post
title:      "Understanding Machine Learning"
date:       2018-10-20 08:10:58 +0530
comments:   true
---

Machine Learning is tough if you haven't been there from beginning because the ecosystem is rapidly developing and growing. In a world with access to a lot of data and a lot of compute power, Machine Learning, an idea to learn from examples and experience - without being explicitly programmed, perfectly makes sense.

When I started learning and reading about machine learning I came across a definition:
> Machine learning is a field of artificial intelligence that uses statistical techniques to give computer systems the ability to "learn" (e.g., progressively improve performance on a specific task) from data, without being explicitly programmed.

I always struggled to understand "progressively improve performance on a specific task". Also, of course, there is programing involved. How is a computer algorithm be generic enough to "learn" from any data. What about the deterministic behaviour of computer algorithms?

The goal of this article is to provide a little context on:
- why machine learning works (e.g., why a machine learns)
- how the modern machine learning algorithms have grown
- why machine learning is a good technique for many different applications and domains

We will go through the toy example of curve fitting problem for the introduction to machine learning. We will first approach this using only algebra, the "old school" way, then we will solve it using machine learning with two different algorithms.

### Curve Fitting

We want to learn the function _f(x)_ which maps the input _x_ to the output _y_. For simplicity in this example let _x_,_y_ from set of real number **R**.

Let say we are given sample observations as follows:

|_x_|_y = f(x)_|
|-----|-----|
|1|21|
|2|53|
|3|101|
|4|165|


Our goal is to lean the mapping of this function fitting above points as best we can. Once we learn that we would be able to predict _ŷ_ for any new input _x_.

Learning in this context means:

- to be able to fit the data to find relationships with various inputs
- to be able to predict output for new input value

If it was a step function like mapping, we could learn what is the best threshold value to always be on one side of the function. Predicting output for a given input is a powerful tool, because we can apply that to many functions in different applications.

Of course, here we assumed that the function accepts single real number as an input and produces a single real number output. (We will discuss about this later.)

### Solving with Algebra

The method below is described from an online lecture. For more info you can check [here](https://www.essie.ufl.edu/~kgurl/Classes/Lect3421/Fall_01/NM5_curve_f01.pdf)

Let's assume that the function _f(x)_ is a polynomial of degree _j_:

![polynomial]({{ site.url }}assets/images/polynomial.png)

![unknowns]({{ site.url }}assets/images/unknowns.png)


Let's plot the data points we have:

![points]({{ site.url }}assets/images/bare-points.png)

When we try to fit these with line: we want to know the function of line which would look like this:

![1st-order-curve]({{ site.url }}assets/images/1st-order-curve.png)

Because these are only 4 data points we get the feeling that the data is linear and we may be able to fit in the first order polynomial. But in fact the data is generated from the second order function. Original points belonged to the [equation](https://www.wolframalpha.com/input/?i=8*x%5E2+%2B+8*x+%2B+5) _y = 8*x^2 + 8*x + 5_

So these would fit nicely when we try to with to second order polynomial:

![2nd-order-curve]({{ site.url }}assets/images/2nd-order-curve.png)


We can compute the function based on the formula of least squared approach - and minimize that function with respect to the co-officiants we want to find to best "fit" our data points.

![solution]({{ site.url }}assets/images/solution.png)

If last part is complex then ignore - it just sums up one metric about the function we know from all the data points we have. That computed metric is known as the error. (In this particular case computed using sum of squared of differences.)

After this to find the parameters which minimises this error metric; we take derivatives of error with respect to the co-efficient. We can put those derivative equations in matrix and solve using linear algebra. The code for this looks like below:

<script src="https://gist.github.com/yogin16/3582b476210cfcf732e6d4f8985d18fe.js"></script>

So from this approach we are able to get the co-officiants of the polynomial. And able to fit the data to the function. Once we learned these co-officiants we would also be able to predict output for new input.

However, there are issues with this approach:
- If you looked in the code snipped - at one point it had to compute matrix inverse in order to solve the linear algebra system.
...That is computationally in-efficient and also not guaranteed to be possible in all cases. And becomes exponentially hard with more number of co-officiants.
- We have figure out value of the order _j_.
- Always assumes that the relationship of function is linear. (Which is ok, based on how we defined our curve fitting problem, but actually not all functions are having linear relationship - we want to remove this assumption now.)

Lets try to approach above issues with two different approach in machine learning, applying to the same problem.

1. https://www.essie.ufl.edu/~kgurl/Classes/Lect3421/Fall_01/NM5_curve_f01.pdf