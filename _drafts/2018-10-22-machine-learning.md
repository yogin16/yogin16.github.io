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

$$ 5+5 $$