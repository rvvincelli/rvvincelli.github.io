It is a rather common real-life situation one where some piece of information cannot be directly accessed but rather it is inferred by looking at some other information we have, more accessible or
 readily available. And more often than not, this inference has to be repeated step after step to drive some computation. Very popular examples here are speech recognition and robot movement control:

* I don't know *exactly* what word you are spelling out, but by looking at the waveform it *sounds* like...

* our friend the robot does not know where he/she is, though its sensors tell it is quite close to a wall

 And very common applications always have very cool, and established, models. The model of reference here is that of [hidden Markov models](https://en.wikipedia.org/wiki/Hidden_Markov_model) and you
 can find of course pages galore on the topic. In a nutshell:

use sandbox https://en.wikipedia.org/w/index.php?title=Wikipedia:Sandbox&action=submit
inline with ![alt text](http://monosnap.com/image/bOcxxxxLGF.png "Title")

---latex---: ![State at time `t`]({{ site.url }}/assets/imgs/1.png), we define our hidden state at time `t` as a random variable; the process unfolds as an infinite sequence ![States]({{ site.url }}/assets/imgs/2.png) of such variables 
 
---latex---: Markov rule: ![Marco]({{ site.url }}/assets/imgs/3.png). This is the so called *memorylessness*, we just look at the immediately preceding state to pontificate on the one to come

---latex---: Evolution: at every step `t` for every pair of states `(i,j)`, in a possibly continuous space, we pass from state i to state j with probability ![pjs]({{ site.url }}/assets/imgs/4.png)

---latex---: At every step we read a value from the indicator variable, whose response is supposed to be meaningfully linked to the hidden state, so something likep ![]({{ site.url }}/assets/imgs/5.png) for every state-evidence pair (i, e)

In the worst case scenarios (read as: always in real-life) the evolution probabilities may not be easily estimated. In:

---latex---: X_t=f_t(X_t-1,W_t-1)

where W models noise on the states, may be nonlinear and time-dependent. The same in general can be said of a function h_k relating the evidence to the hidden state.

We define the *filtering problem* as the estimation of the current state x_t given the previous state x_t-1 and all of the previous pieces of evidence e_1,...,e_t-1. In a regular
[bayesian](https://en.wikipedia.org/wiki/Bayes_theorem) the estimation of the state is actually a probability distribution:
P(X_t|E_1,...,E_t-1)
computed in two steps:

*prediction, calculate P(X_t|E_1,...,E_t-1), the prior over X_t before receiving the evidence score e_t in the...

*update, the posterior on X_t is obtained by direct application of the Bayes rule

These two steps all boil down to integrals, which we don't expect to be amenable analytically. So what do we do? If we can't snipe the target then let's just shoot around at random - that's the
spirit of the so-called [Monte Carlo](https://en.wikipedia.org/wiki/Monte_Carlo_method) methods! An option is the *bootstrap particle filtering*. The *particle* in it refers to the fact that we
create and maintain a population of random (MC style) particles along the simulation, and [*bootstrap*](https://en.wikipedia.org/wiki/Bootstrapping_(statistics)) because it makes use of sampling
with replacement. The point here is that the particles, at every step, represent the state posterior distribution, in the form of a sample rather than a closed form.

It turns out that the algorithm is pretty simple:

1. Extract n samples from the initial prior X_0, these are our particles P_0={x_00, x_01, ... x_0n}; these particles are realizations of the variable X_0

2. For each particle in P_i sample P(X_t|X_t-1), so basically choose how to evolve each sample according to the f_t above

3. How likely is each evolution? Lookup every score with the evidence function h_j, which is sampling the distribution P(X_t|E_t); these values may be one-normalized and represent a weighthing on
the particles population

4. Bootstrap step, resample with replacement according to the weights; this prepares the population of particles for the next iteration

In the case of discrete distributions, all of the information may be conveniently encoded in matrices and vectors:

* a matrix M_ij where the value is the probability of transitioning from state i into state j

* a matrix E_kl where the value is the probability of the hidden state k given the observable state l

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
