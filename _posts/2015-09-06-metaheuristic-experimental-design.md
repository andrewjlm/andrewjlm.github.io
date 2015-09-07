---
layout: post
title: "Optimal design using metaheuristic algorithms in R"
categories: r statistics
---
A common problem in experimental design is to find the design that is the 'best' (or optimal) according to some criteria. For example, suppose we are designing an experiment aimed at discovering the relationship between exposure to some chemical and the water resistance of a wood board. We have 20 test boards and can expose each of them to the chemical for up to one hour. Symbolically, then, our possible designs are all of the form $$ 0 \leq x_i \leq 1 $$ for $$ i=1,\ldots,20 $$.

Suppose we wanted to model this situation using linear regression. How long should each board be exposed to the chemical? The exact answer to this question depends on what we're trying to optimize. One common optimality criterion is D-optimality, which minimizes the geneneralized variance of our parameter estimates. To find this design, we need to maximize $$ \vert X'X \vert $$, the determinant of the information matrix. Determining the design that maximizes this is fairly straightforward - the commonly known D-optimal design for our example is exposing 10 of the boards to the chemical for the full hour and not exposing the other 10 at all.

However, there are design criteria that are not as easy to directly calculate. For example, we might want to minimize the average variance of our parameter estimates. This is know as A-optimality and can be found by minimizing the trace of the inverse of the information matrix. You're welcome to try and directly calculate the optimal design yourself, but it is nowhere near as simple as finding the D-optimal design.

To attempt to determine what the A-optimal design for this problem is, I've decided to use three off-the-shelf implementations of metaheuristic algorithms that can be found in R packages. I'll discuss what I consider the pros and cons of each method and provide theoretical support for the answers they come up with.

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

# Number of times to run each algorithm
runs <- 10
{% endhighlight %}

For all of these metaheuristic algorithms there are a number of optional control parameters, but for simplicity's sake I've chosen to leave them all at their default settings. If you intend to use these algorithms for anything serious, I'd strongly encourage you to check out the documentation because these control parameters can have a huge impact on the usefulness of the algorithm.

#### Simulated Annealing

Annealing is a metallurgical technique that involves heating and slowly cooling a piece of metal with the goal of decreasing its hardness. Simulated annealing takes this method as inspiration to find a global optimum for a particular function. The basic idea is that early in the search (when the temperature is high) the algorithm can make large jumps that prevent it from being stuck in local maxima. As the temperature cools, the 'jumps' within the parameter space become smaller in size, which reduces the probability that we'll end up back at a less-than-optimal solution.

There are a number of different implementations of this method in R but I've chosen to use the one found in the `GenSA` package. The `GenSA` function takes a few arguments - the initial values (which I let be random), the function to optimize and the upper and lower bounds for each of our parameters.

{% highlight r %}
# Simulated annealing for A and D optimality
library(GenSA)

SA.Aresults <- rep(NA, runs)
SA.Dresults <- rep(NA, runs)

set.seed(1234)

for (i in 1:runs) {
  Asadef <- GenSA(NULL, aoptmat, lower = rep(0,20), upper = rep(1,20))
  Dsadef <- GenSA(NULL, doptmat, lower = rep(0,20), upper = rep(1,20))
  SA.AOpt <- sort(Asadef$par)
  SA.DOpt <- sort(Dsadef$par)
  
  SA.Aresults[i] <- toString(table(SA.AOpt))
  SA.Dresults[i] <- toString(table(SA.DOpt))
}
{% endhighlight %}

Here are the results for our A- and D-optimal designs:

{% highlight r %}
c(table(SA.Aresults))

## 12, 8 
##    10

c(table(SA.Dresults))

## 10, 10 
##     10
{% endhighlight %}

In each case, all 10 runs of our algorithm agreed on an optimal design. For A-optimality, we get that exposing 12 of the boards to none of the chemical and 8 boards to a full hour of the chemical produces the optimal design. For D-optimality, we get our known result that the optimal design is half the boards at zero exposure and the other half at maximum exposure. Simulated annealing is my personal favorite metaheuristic as in my experience it performs well at the default settings. However, (anecdotally) it is slower than the following methods.

#### Particle Swarm Optimization

Particle swarm optimization simulates a swarm of particles in $$n$$-dimensional space each with some initial velocity. This velocity can either remain constant or decrease over time. As the particles move through the space we keep track of both the best position encountered by each particle and the best position encountered globally by all particles. The directions that each particle moves in is based on some blend of both that particle's best and the global best. These numbers can technically be user-specified but in practice are almost always both set to 2. The main optional parameters that are used to control the behavior of the swarm are the number of particles and the number of iterations the swarm moves through.

{% highlight r %}
# Particle swarm optimization for A and D optimality
library(pso)

PSO.Aresults <- rep(NA, runs)
PSO.Dresults <- rep(NA, runs)

set.seed(2341)

for (i in 1:runs) {
  Apsodef <- psoptim(rep(NA,20),aoptmat,lower = 0, upper = 1)
  Dpsodef <- psoptim(rep(NA,20),doptmat, lower = 0, upper = 1)
  PSO.AOpt <- sort(Apsodef$par)
  PSO.DOpt <- sort(Dpsodef$par)
  
  PSO.Aresults[i] <- toString(table(PSO.AOpt))
  PSO.Dresults[i] <- toString(table(PSO.DOpt))
}
{% endhighlight %}

How does it perform?

{% highlight r %}
c(table(PSO.Aresults))

## 11, 9 12, 8 13, 7 
##     4     5     1

c(table(PSO.Dresults))

## 10, 10  9, 11 
##      9      1
{% endhighlight %}

One disadvantage of particle swarm optimization is that we're not guaranteed a globally optimal result. That's not to say that it can't be useful. We need to consider if finding the *exact* optimal design is of any practical important for our problem versus finding something that is close to the optimal design. If we don't need an exact result, PSO is still highly appealing because tuning it only requires deciding on the size of the swarm and the number of iterations. Additionally, it seems to run much faster than either simulated annealing or genetic algorithms (again, this is anecdotal).

#### Genetic Algorithms

Genetic algorithms mimic the process of evolution. The general idea is that is we have two or more potential parameter sets that give good results, we should somehow be able to find a better set of parameters. We can think of each combination of factors as a chromosome. Based on how those combinations perform on the function we're trying to optimize, we'll find the best designs and somehow combine/randomly change them. The combination is intended to generate better solutions from a set of already good ones and the random change (or mutation) is an attempt to prevent the algorithm from getting stuck at a local optima.

{% highlight r %}
library(genalg)

set.seed(3421)

Agadef <- rbga(stringMin = rep(0,20), stringMax = rep(1,20),
               evalFunc = aoptmat)
Dgadef <- rbga(stringMin = rep(0,20), stringMax = rep(1,20),
               evalFunc = doptmat)
GA.AOpt <- sort(round(Agadef$population[which.min(Agadef$evaluations),],
                      digits = 3))
GA.DOpt <- sort(round(Dgadef$population[which.min(Dgadef$evaluations),],
                      digits = 3))
{% endhighlight %}

I've only ran this once because I haven't found it incredibly simple to compare the results of the genetic algorithm across runs. Here is what this run came up with:

{% highlight r %}
table(GA.AOpt)

## GA.AOpt
## 0.003 0.004 0.005 0.006 0.009 0.011 0.014 0.015 0.018 0.026 0.982 0.989 
##     1     1     1     1     1     3     1     1     1     1     1     1 
## 0.992 0.993 0.994 0.998 
##     1     1     1     3

table(GA.DOpt)

## 0.005 0.006 0.008  0.01 0.012 0.013 0.014 0.015 0.025 0.984 0.985 0.988 
##     1     1     2     1     1     1     1     1     1     1     1     1 
## 0.992 0.993 0.997 0.999     1 
##     1     1     2     1     2
{% endhighlight %}

Like PSO, convergence to a global optimum is not guaranteed. If you round the above numbers you'll find that the genetic algorithm returns results that are close to what we found using simulated annealing. Again, in your particular application a set of results that is close to the global optimum may suffice and genetic algorithms may be worth considering.

#### Theoretical Support

In this post I've acted as if 12 zero-exposure boards and 8 full-exposure boards is the optimal solution without providing any proof except the output of the simulated annealing runs. There is some theoretical support that this is the true optimal design. This can be found in the excellent book by Martijn Berger and Weng-Kee Wong, *An Introduction to Optimal Designs for Social and Biomedical Research*. On page 71, they indicate that for performing simple linear regression the two-point A-optimal design over an interval $$[a,b]$$, the weight at each point is given by:

$$ w_1 = \frac{\sqrt{(a+b^2)}}{\sqrt{(1+a^2)}+\sqrt{(1+b^2)}} \\
w_2 = 1 - w_1 $$

For our design, this works out to $$w_1=.5773$$ and $$w_2=.4226$$. For a sample of size 20, this would correspond to 11.54 zero exposure boards, which would round to the result found by simulated annealing. That being said, I have not been able to find any evidence that this is the A-optimal design if we consider designs that could have more than two support points. To truly accept this as the optimal result, we would need to prove some result along the lines of 'the optimum number of support points for simple linear regression is 2'. This seems intuitively true, but I haven't found theoretical support for it yet. If you know of any, please contact me via email or twitter!

#### Future Steps

From here, there are a few other questions that I think would be interesting to investigate.

1. Can we tune the PSO method to encourage finding a global optimum? Can we somehow average multiple runs of the PSO to find the global optimum?

2. Are my intuitions about the relative speed of these methods correct? This should be fairly simple to confirm, I just haven't gotten around to doing it yet. One interesting twist is that the fastest algorithm might differ by the particular problem we're trying to apply it to.

3. How can the movement of these algorithms toward solutions be visualized? Seeing how the best solution found by the algorithm evolves as the iterations increases could be very interesting.

If you have any thoughts on these questions or any others, please reach out. I think these metaheuristic algorithms hold great promise in the field of experimental design and I'd love to discuss them with anyone else who is curious.