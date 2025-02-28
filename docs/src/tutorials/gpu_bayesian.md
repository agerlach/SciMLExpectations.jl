# GPU-Accelerated Data-Driven Bayesian Uncertainty Quantification with Koopman Operators

What if you have data and a general model and would like to evaluate the
probability that the fitted model outcomes would have had a given behavior?
The purpose of this tutorial is to demonstrate a fast workflow for doing exactly
this. It composes together a few different pieces of the SciML ecosystem:

1. Parameter estimation with uncertainty with Bayesian differential equations by
   integrating the differentiable differential equation solvers with the
   [Turing.jl library](https://turing.ml/dev/).
2. Fast calculation of probabilistic estimates of differential equation solutions
   with parametric uncertainty using the Koopman expectation.
3. GPU-acceleration of batched differential equation solves.

Let's dive right in.

## Bayesian Parameter Estimation with Uncertainty

Let's start by importing all of the necessary libraries:

```julia
using Turing, Distributions, DifferentialEquations
using MCMCChains, Plots, StatsPlots
using Random
using SciMLExpectations
using KernelDensity, SciMLExpectations
using Cuba, DiffEqGPU

Random.seed!(1);
```

For this tutorial we will use the Lotka-Volterra equation:

```julia
function lotka_volterra(du,u,p,t)
  @inbounds begin
      x = u[1]
      y = u[2]
      α = p[1]
      β = p[2]
      γ = p[3]
      δ = p[4]
      du[1] = (α - β*y)*x
      du[2] = (δ*x - γ)*y
  end
end
p = [1.5, 1.0, 3.0, 1.0]
u0 = [1.0,1.0]
prob1 = ODEProblem(lotka_volterra,u0,(0.0,10.0),p)
sol = solve(prob1,Tsit5())
plot(sol)
```

From the Lotka-Volterra equation we will generate a dataset with known parameters:

```julia
sol1 = solve(prob1,Tsit5(),saveat=0.1)
```

Now let's assume our dataset should have noise. We can add this noise in and
plot the noisy data against the generating set:

```julia
odedata = Array(sol1) + 0.8 * randn(size(Array(sol1)))
plot(sol1, alpha = 0.3, legend = false); scatter!(sol1.t, odedata')
```

Now let's assume that all we know is the data `odedata` and the model form.
What we want to do is use the data to inform us of the parameters, but also
get a probabilistic sense of the uncertainty around our parameter estimate. This
is done via Bayesian estimation. For a full look at Bayesian estimation of
differential equations, look at the [Bayesian differential equation](https://turing.ml/dev/tutorials/10-bayesiandiffeq/)
tutorial from Turing.jl.

Following that tutorial, we choose a set of priors and perform `NUTS` sampling
to arrive at the MCMC chain:

```julia
Turing.setadbackend(:forwarddiff)

@model function fitlv(data, prob1)
    σ ~ InverseGamma(2, 3) # ~ is the tilde character
    α ~ truncated(Normal(1.5,0.5),1.0,2.0)
    β ~ truncated(Normal(1.2,0.5),0.5,1.5)
    γ ~ truncated(Normal(3.0,0.5),2,4)
    δ ~ truncated(Normal(1.0,0.5),0.5,1.5)

    p = [α,β,γ,δ]
    prob = remake(prob1, p=p)
    predicted = solve(prob,Tsit5(),saveat=0.1)

    for i = 1:length(predicted)
        data[:,i] ~ MvNormal(predicted[i], σ)
    end
end

model = fitlv(odedata, prob1)

# This next command runs 3 independent chains without using multithreading.
chain = mapreduce(c -> sample(model, NUTS(.45),1000), chainscat, 1:3)
```

This chain gives a discrete approximation to the probability distribution of our
desired quantites. We can plot the chains to see this distributions in action:

```julia
plot(chain)
```

Great! From our data we have arrived at a probability distribution for the
our parameter values.

## Evaluating Model Hypotheses with the Koopman Expectation

Now let's try and ask a question: what is the expected value of `x` (the first
term in the differential equation) at time `t=10` given the known uncertainties
in our parameters? This is a good tutorial question because all other probabilistic
statements can be phrased similarly. Asking a question like, "what is the probability
that `x(T) > 1` at the final time `T`?", can similarly be phrased as an expected
value (probability statements are expected values of characteristic functions
which are 1 if true 0 if false). So in general, the kinds of questions we want
to ask and answer are expectations about the solutions of the differential equation.

The trivial to solve this problem is to sample 100,000 sets of parameters from
our parameter distribution given by the Bayesian estimation, solve the ODE
100,000 times, and then take the average. But is 100,000 ODE solves enough?
Well it's hard to tell, and even then, the convergence of this approach is slow.
This is the Monte Carlo approach and it converges to the correct answer by
`sqrt(N)`. Slow.

However, the [Koopman expectation](https://arxiv.org/abs/2008.08737) can converge
with much fewer points, allowing the use of higher order quadrature methods to
converge exponentially faster in many cases. To use the Koopman expectation
functionality provided by [SciMLExpectations.jl](https://github.com/SciML/SciMLExpectations.jl),
we first need to define our observable function `g`. This function designates the
thing about the solution we wish to calculate the expectation of. Thus for our
question "what is the expected value of `x`at time `t=10`?", we would simply use:

```julia
function g(sol)
    sol[1,end]
end
```

Now we need to use the `expectation` call, where we need to provide our initial
condition and parameters as probability distirbutions. For this case, we will use
the same constant `u0` as before. But, let's turn our Bayesian MCMC chains into
distributions through [kernel density estimation](https://github.com/JuliaStats/KernelDensity.jl)
(the plots of the distribution above are just KDE plots!).

```julia
p_kde = [kde(vec(Array(chain[:α]))),kde(vec(Array(chain[:β]))),
         kde(vec(Array(chain[:γ]))),kde(vec(Array(chain[:δ])))]
```

Now that we have our observable and our uncertainty distributions, let's calculate
the expected value:

```julia
expect = expectation(g, prob1, u0, p_kde, Koopman(), Tsit5(), quadalg = CubaCuhre())
```

Note how that gives the expectation and a residual for the error bound!

```julia
expect.resid
```

### GPU-Accelerated Expectations

Are we done? No, we need to add some GPUs! As mentioned earlier, probability
calculations can take quite a bit of ODE solves, so let's parallelize across
the parameters. [DiffEqGPU.jl](https://github.com/SciML/DiffEqGPU.jl) allows you
to GPU-parallelize across parameters by using the
[Ensemble interface](https://diffeq.sciml.ai/stable/features/ensemble/). Note that
you do not have to do any of the heavy lifting: all of the conversion to GPU
kernels is done automaticaly by simply specifying `EnsembleGPUArray` as the
ensembling method. For example:

```julia
function lotka_volterra(du,u,p,t)
  @inbounds begin
      x = u[1]
      y = u[2]
      α = p[1]
      β = p[2]
      γ = p[3]
      δ = p[4]
      du[1] = (α - β*y)*x
      du[2] = (δ*x - γ)*y
  end
end
p = [1.5, 1.0, 3.0, 1.0]
u0 = [1.0,1.0]
prob = ODEProblem(lotka_volterra,u0,(0.0,10.0),p)
prob_func = (prob,i,repeat) -> remake(prob,p=rand(Float64,4).*p)
monteprob = EnsembleProblem(prob, prob_func = prob_func, safetycopy=false)
@time sol = solve(monteprob,Tsit5(),EnsembleGPUArray(),trajectories=10_000,saveat=1.0f0)
```

Let's now use this in the ensembling method. We need to specify a `batch` for the
number of ODEs solved at the same time, and pass in our enembling method. The
following is a GPU-accelerated uncertainty quanitified estimate of the expectation
of the solution:

```julia
expectation(g, prob1, u0, p_kde, Koopman(), Tsit5(), EnsembleGPUArray(), batch=100, quadalg = CubaCuhre())
```
