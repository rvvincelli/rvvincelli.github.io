It is a rather common real-life situation one where some piece of information cannot be directly accessed but rather it is inferred by looking at some other information we have, more accessible or
 readily available. And more often than not, this inference has to be repeated step after step to drive some computation. Very popular examples here are speech recognition and robot movement control:

* I don't know *exactly* what word you are spelling out, but by looking at the waveform it *sounds* like...

* our friend the robot does not know where he/she is, though its sensors tell it is quite close to a wall

 And very common applications always have very cool, and established, models. The model of reference here is that of [hidden Markov models](https://en.wikipedia.org/wiki/Hidden_Markov_model) and you
 can find of course pages galore on the topic. In a nutshell:

* ![]({{ site.url }}/assets/imgs/1.png), we define our hidden state at time ![]({{ site.url }}/assets/imgs/t.png) as a random variable; the process unfolds as an infinite sequence ![]({{ site.url }}/assets/imgs/2.png) of such variables 
 
* Markov rule: ![]({{ site.url }}/assets/imgs/3.png). This is the so called *memorylessness*, we just look at the immediately preceding state to pontificate on the one to come

* Evolution: at every step `t` for every pair of states ![]({{ site.url }}/assets/imgs/ij.png), in a possibly continuous space, we pass from state ![]({{ site.url }}/assets/imgs/i.png) to state ![]({{ site.url }}/assets/imgs/j.png) with probability ![]({{ site.url }}/assets/imgs/4.png)

* At every step we read a value from the indicator variable, whose response is supposed to be meaningfully linked to the hidden state, so something likep ![]({{ site.url }}/assets/imgs/5.png) for every state-evidence pair ![]({{ site.url }}/assets/imgs/ie.png) 

In the worst case scenarios (read as: always in real-life) the evolution probabilities may not be easily estimated. In:

![]({{ site.url }}/assets/imgs/6.png)

where ![]({{ site.url }}/assets/imgs/7.png) models noise on the states, may be nonlinear and time-dependent. The same in general can be said of a function ![]({{ site.url }}/assets/imgs/8.png) relating the evidence to the hidden state.

We define the *filtering problem* as the estimation of the current state value ![]({{ site.url }}/assets/imgs/xt.png) given the previous state ![]({{ site.url }}/assets/imgs/xtmin.png) and all of the previous pieces of evidence ![]({{ site.url }}/assets/imgs/es.png). In a regular
[bayesian](https://en.wikipedia.org/wiki/Bayes_theorem) the estimation of the state is actually a probability distribution:

![]({{ site.url }}/assets/imgs/superbeis.png)

computed in two steps:

* prediction, calculate ![](assets/imgs/superbeis2.png?raw=true), the prior over ![]({{ site.url }}/assets/imgs/1.png) before receiving the evidence score ![]({{ site.url }}/assets/imgs/et.png) in the...

* update, the posterior on ![]({{ site.url }}/assets/imgs/1.png) is obtained by direct application of the Bayes rule

These two steps all boil down to integrals, which we don't expect to be amenable analytically. So what do we do? If we can't snipe the target then let's just shoot around at random - that's the
spirit of the so-called [Monte Carlo](https://en.wikipedia.org/wiki/Monte_Carlo_method) methods! An option is the *bootstrap particle filtering*. The *particle* in it refers to the fact that we
create and maintain a population of random (MC style) particles along the simulation, and [*bootstrap*](https://en.wikipedia.org/wiki/Bootstrapping_(statistics)) because it makes use of sampling
with replacement. The point here is that the particles, at every step, represent the state posterior distribution, in the form of a sample rather than a closed form.

It turns out that the algorithm is pretty simple:

1. Extract ![]({{ site.url }}/assets/imgs/n.png) samples from the initial prior ![]({{ site.url }}/assets/imgs/X0.png), these are our particles ![]({{ site.url }}/assets/imgs/parts.png); these particles are realizations of the variable ![]({{ site.url }}/assets/imgs/X0.png)

2. For each particle in ![]({{ site.url }}/assets/imgs/Pt.png) sample ![]({{ site.url }}/assets/imgs/tobesampled.png), so basically choose how to evolve each sample according to the ![]({{ site.url }}/assets/imgs/ft.png) above

3. How likely is each evolution? Lookup every score with the evidence function ![]({{ site.url }}/assets/imgs/8.png), which is sampling the distribution ![]({{ site.url }}/assets/imgs/distro.png); these values may be one-normalized and represent a weighthing on
the particles population

4. Bootstrap step, resample with replacement according to the weights; this prepares the population of particles for the next iteration

In the case of discrete distributions, all of the information may be conveniently encoded in matrices and vectors:

* a matrix ![]({{ site.url }}/assets/imgs/mij.png) where the value is the probability of transitioning from state ![]({{ site.url }}/assets/imgs/i.png) into state ![]({{ site.url }}/assets/imgs/j.png)

* a matrix ![]({{ site.url }}/assets/imgs/ekl.png) where the value is the probability of the hidden state ![]({{ site.url }}/assets/imgs/k.png) given the observable state ![]({{ site.url }}/assets/imgs/l.png)

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


For[i = 1, i <= Length[e], i++,
 For[j = 1, j <= n, j++,
  particles[[j]] = RandomChoice[M[[particles[[j]], All]] -> values];
  weights[[j]] = E[[e[[i]], particles[[j]]]]
 ]
]

so let's take the row of the M matrix corresponding to the particle value, easy, as the outcome already defines the index, and sample the corresponding line distribution. Rephrased:

1. extract a particle
2. check what realization is it and seek to the corresponding row on M
3. evolve the particle, thus select a new value at the next instant t+1 based on the row itself, which represents the possible translations

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
rather soon, and we want to get rid of them. On the other side you also run the risk of sample impoverishment
- the set is dominated by a few large-weighted particles and we might get steady too soon, missing out on a more fitting particle set.

For a very well written document on this, discussing variations of the basic algorithm too - check out [this](http://www.cns.nyu.edu/~eorhan/notes/particle-filtering.pdf).

Find the code [here](https://github.com/rvvincelli/pdm/blob/master/ParticleFiltering.nb).
