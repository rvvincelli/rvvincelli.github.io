---
layout: post
title: Hide that Markow
---

{% include google_analytics.html %}

##Carrying particles around

###Particle filtering for HMMs

It is a rather common real-life situation one where some piece of information cannot be directly accessed but rather it is inferred by looking at some other information we have, more accessible or
 readily available. And more often than not, this inference has to be repeated step after step to drive some computation. Very popular examples here are speech recognition and robot movement control:

* I don't know *exactly* what word you are spelling out, but by looking at the waveform it *sounds* like...

* our friend the robot does not know where he/she is, though the sensors tell it is quite close to a wall

And very common applications always have very cool, and established, models. The model of reference here is that of [hidden Markov models](https://en.wikipedia.org/wiki/Hidden_Markov_model) and you
 can find of course pages galore on the topic. In a nutshell:

* ![](../assets/imgs/1.png?raw=true), we define our hidden state at time ![](../assets/imgs/t.png?raw=true) as a random variable; the process unfolds as an infinite sequence ![](../assets/imgs/2.png?raw=true) of such variables 
 
* Markov rule: ![](../assets/imgs/3.png?raw=true). This is the so called *memorylessness*, we just look at the immediately preceding state to pontificate on the one to come

* Evolution: at every step ![](../assets/imgs/t.png?raw=true) for every pair of states ![](../assets/imgs/ij.png?raw=true), in a possibly continuous space, we pass from state ![](../assets/imgs/i.png?raw=true) to state ![](../assets/imgs/j.png?raw=true) with probability ![](../assets/imgs/4.png?raw=true)

* At every step we read a value from the indicator variable, whose response is supposed to be meaningfully linked to the hidden state, so something like ![](../assets/imgs/5.png?raw=true) for every state-evidence pair ![](../assets/imgs/ie.png?raw=true) 

In the worst case scenarios (read as: always in real-life problems) the evolution probabilities may not be easily estimated:

![](../assets/imgs/6.png?raw=true)

where ![](../assets/imgs/7.png?raw=true) models noise on the states, may be nonlinear and time-dependent. The same in general can be said of a function ![](../assets/imgs/8.png?raw=true) relating the evidence to the hidden state.

We define the *filtering problem* as the estimation of the current state value ![](../assets/imgs/xt.png?raw=true) given the previous state ![](../assets/imgs/xtmin.png?raw=true) and all of the previous pieces of evidence ![](../assets/imgs/es.png?raw=true). In a regular
[bayesian context](https://en.wikipedia.org/wiki/Bayes_theorem) the estimation of the state is actually a probability distribution:

![](../assets/imgs/superbeis.png?raw=true)

computed in two steps:

* prediction, calculate ![](../assets/imgs/superbeis2.png?raw=true), the prior over ![](../assets/imgs/1.png?raw=true) before receiving the evidence score ![](../assets/imgs/et.png?raw=true) in the...

* update, the posterior on ![](../assets/imgs/1.png?raw=true) is obtained by direct application of the Bayes rule

These two steps all boil down to integrals, which we don't expect to be amenable analytically. So what do we do? If we can't snipe the target then let's just shoot around at random - that's the
spirit of the so-called [Monte Carlo](https://en.wikipedia.org/wiki/Monte_Carlo_method) methods! An option is the *bootstrap particle filtering*. The *particle* in it refers to the fact that we
create and maintain a population of random (MC style) particles along the simulation, and [*bootstrap*](https://en.wikipedia.org/wiki/Bootstrapping_(statistics)) because it makes use of sampling
with replacement. The point here is that the particles, at every step, represent the state posterior distribution, in the form of a sample rather than a closed form.

It turns out that the algorithm is pretty simple:

1. Extract ![](../assets/imgs/n.png?raw=true) samples from the initial prior ![](../assets/imgs/X0.png?raw=true), these are our particles ![](../assets/imgs/parts.png?raw=true); these particles are realizations of the variable ![](../assets/imgs/X0.png?raw=true)

2. For each particle in ![](../assets/imgs/pt.png?raw=true) sample ![](../assets/imgs/tobesampled.png?raw=true), so basically choose how to evolve each sample according to the ![](../assets/imgs/ft.png?raw=true) above

3. How likely is each evolution? Lookup every score with the evidence function ![](../assets/imgs/8.png?raw=true), which is sampling the distribution ![](../assets/imgs/distro.png?raw=true); these values may be one-normalized and represent a weighthing on
the particles population

4. Bootstrap step, resample with replacement according to the weights; this prepares the population of particles for the next iteration

In the case of discrete distributions, all of the information may be conveniently encoded in matrices and vectors:

* a matrix ![](../assets/imgs/mij.png?raw=true) where the value is the probability of transitioning from state ![](../assets/imgs/i.png?raw=true) into state ![](../assets/imgs/j.png?raw=true)

* a matrix ![](../assets/imgs/ekl.png?raw=true) where the value is the probability of the hidden state ![](../assets/imgs/k.png?raw=true) given the observable state ![](../assets/imgs/l.png?raw=true)

The code is pretty simple, let's have a look around with [Mathematica](https://www.wolfram.com/mathematica). What we want is a program to create and evolve a family of particles, representing the
target distribution, as described above. Let's stick to an all-discrete world. The signature may simply be:

```mathematica
ParticleFiltering[X_, M_, E_, n_, e_]
```

where:

* `X` is the initial distribution; we are going to generate the first batch of particles from it; if no assumptions may be made then a uniform is always a good prior, but in the particular case of
'nice' (non-reducible, aperiodic and equipped with an invariant distribution) Markov chains the relevance initial distribution fades away as time goes on 

* `M` is our transition matrix, in particular `M[[i,j]]` is the probability of evolving from state `i` to state `j`

* `E` is the evidence matrix, `E[[k,l]]` gives us the probability of the hidden state `k` given that we have seen `l` as evidence

* `n` the number of particles; it should be greater than 30 please!

* `e` the vector of recorded evidence scores - this is the information driving our guess for the variable `X_t` step after step

To simplify things, lets take the support set for the hidden state to be that of natural numbers, so that we can index the matrices easily.

Here is the first generation of particles:

```mathematica
particles = RandomChoice[X -> values, n]
```

where `values` is just `{0,1,...m}`, so we are just randomly choosing from this set enough particles, using the scores of `X` as weights, we are sampling `X`

The iteration looks like:

```mathematica
For[i = 1, i <= Length[e], i++,
 For[j = 1, j <= n, j++,
  particles[[j]] = RandomChoice[M[[particles[[j]], All]] -> values];
  weights[[j]] = E[[e[[i]], particles[[j]]]]
 ]
]
```

so let's take the row of the `M` matrix corresponding to the particle value -  easy, as the outcome already defines the index - and sample the corresponding line distribution. Rephrased:

1. extract a particle
2. check what realization is it and seek to the corresponding row on `M`
3. evolve the particle, thus select a new value at the next instant *t+1* based on the row itself, which represents the possible translations

So now say we have evolved from state `i` to state `j`, how likely is this to happen given that we know the current evidence sample `e_t`? We define a number of weights for our newbreed of particles,
right from the evidences:

```mathematica
weights[[j]] = E[[e[[i]], particles[[j]]]]
```

Now we have a new population of particles, and weights telling us how legitimate they are. Let's resample it:

```mathematica
particles = RandomChoice[weights -> particles, Length[particles]]
```

Now repeat!

In general a bootstrapping step is always due in genetic-like algorithms as we know. In the particular procedure here the set of weights might degenerate - a lot of particles might get small weights
rather soon, and we want to get rid of them. On the other side you also run the risk of sample impoverishment - the set is dominated by a few large-weighted particles and we might get steady too soon, missing out on a more fitting particle set.

For a very well written document on this, discussing variations of the basic algorithm too - check out [this](http://www.cns.nyu.edu/~eorhan/notes/particle-filtering.pdf).

Find the code and some little graphical insights [here](https://github.com/rvvincelli/pdm/blob/master/ParticleFiltering.nb).


