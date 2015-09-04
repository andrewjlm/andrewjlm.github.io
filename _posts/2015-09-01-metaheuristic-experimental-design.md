---
layout: post
title: "Optimal design using metaheuristic algorithms in R"
categories: r statistics
---
A common problem in experimental design is to find the design that is the 'best' (or optimal) according to some criteria. For example, suppose we are designing an experiment aimed at discovering the relationship between exposure to some chemical and the water resistance of a wood board. We have 20 test boards and can expose each of them to the chemical for up to one hour. Symbolically, then, our possible designs are all of the form $$ 0 \leq x_i \leq 1 $$ for $$ i=1,\ldots,20 $$.

Suppose we wanted to model this situation using linear regression. How long should each board be exposed to the chemical? The exact answer to this question depends on what we're trying to optimize. One common optimality criterion is D-optimality, which minimizes the geneneralized variance of our parameter estimates. To find this design, we need to maximize $$ \vert X'X \vert $$, the determinant of the information matrix. Determining the design that maximizes this is fairly straightforward - the commonly known D-optimal design for our example is exposing 10 of the boards to the chemical for the full hour and not exposing the other 10 at all.

However, there are design criteria that are not as easy to directly calculate. For example, we might want to minimize the average variance of our parameter estimates. This is know as A-optimality and can be found by minimizing the trace of the inverse of the information matrix. You're welcome to try and directly calculate the optimal design yourself, but it is nowhere near as simple as finding the D-optimal design.

To attempt to determine what the A-optimal design for this problem is, I've decided to use three off-the-shelf implementations of metaheuristic algorithms that can be found in R packages.

#### Problem Setup

To use these algorithms, we need to create a function in R that finds the trace of the inverse of the information matrix when given a vector of input values. For the sake of completeness and comparison, I've also created a function that we can use to check the known result we have for D-optimality.

{% highlight r %}
aoptmat <- function(mat) {
	mat <- matrix(c(rep(1,20),mat),ncol=2)
	sum(diag(solve(t(mat) %*% mat)))
}

doptmat <- function(mat) {
	mat <- matrix(c(rep(1,20),mat),ncol=2)
	det(solve(t(mat) %*% mat))
}
{% endhighlight %}

#### Simulated Annealing

#### Partical Swarm Optimization

#### Genetic Algorithms