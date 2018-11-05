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

We will go through the toy example of curve fitting problem for the introduction to machine learning starting with bare algebra and then doing same thing in the ML way.

### Curve Fitting
We want to learn the function _f(x)_ which maps the input _x_ to the output _y_. For simplicity in this example let _x_,_y_ from set of real number **R**.

Let say we are given sample observations as follows:

|x|y=f(x)|
|-----|-----|
|1|21|
|2|53|
|3|101|
|4|165|


Our goal is to lean the mapping of this function fitting above points as best we can. And once we learn that we would be able to predict _ŷ_ for any new input _x_.
So, learning in this context means:
- to be able to fit the data to find relationships with various inputs (for example if it was a step function like mapping, how to learn what is the best threshold value to always be on one side of the function)
- to be able to predict output for new input value (this is a powerful tool. because a system is set of complex functions. and if we able to predict output for given input for any particular function we are able to learn how the system would behave)

Of course, here we assumed that the function accepts single real number as an input and produces a single real number output. But we will discuss about this later.