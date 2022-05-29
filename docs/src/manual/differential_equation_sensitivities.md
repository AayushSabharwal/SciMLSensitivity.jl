# [Sensitivity Analysis of Differential Equations](@id sensitivity_diffeq)

DiffEqFlux is capable of training neural networks embedded inside of
differential equations with many different techniques. For all of the
details, see the
[DifferentialEquations.jl local sensitivity analysis](https://diffeq.sciml.ai/latest/analysis/sensitivity/)
documentation. Here we will summarize these methodologies in the
context of neural differential equations and scientific machine learning.


## Sensitivity Algorithms

The following algorithm choices exist for `sensealg`. See
[the sensitivity mathematics page](@ref sensitivity_math) for more details on
the definition of the methods.

- `ForwardSensitivity(;ADKwargs...)`: An implementation of continuous forward
  sensitivity analysis for propagating derivatives by solving the extended ODE.
  Only supports ODEs.
- `ForwardDiffSensitivity(;chunk_size=0,convert_tspan=true)`: An implementation
  of discrete forward sensitivity analysis through ForwardDiff.jl.
- `BacksolveAdjoint(;checkpointing=true,ADKwargs...)`: An implementation of
  adjoint sensitivity analysis using a backwards solution of the ODE. By default
  this algorithm will use the values from the forward pass to perturb the backwards
  solution to the correct spot, allowing reduced memory with stabilization.
  Only supports ODEs and SDEs.
- `InterpolatingAdjoint(;checkpointing=false;ADKwargs...)`: The default. An
  implementation of adjoint sensitivity analysis which uses the interpolation of
  the forward solution for the reverse solve vector-Jacobian products. By
  default it requires a dense solution of the forward pass and will internally
  ignore saving arguments during the gradient calculation. When checkpointing is
  enabled it will only require the memory to interpolate between checkpoints.
  Only supports ODEs and SDEs.
- `QuadratureAdjoint(;abstol=1e-6,reltol=1e-3,compile=false,ADKwargs...)`:
  An implementation of adjoint sensitivity analysis which develops a full
  continuous solution of the reverse solve in order to perform a post-ODE
  quadrature. This method requires the the dense solution and will ignore
  saving arguments during the gradient calculation. The tolerances in the
  constructor control the inner quadrature. The inner quadrature uses a
  ReverseDiff vjp if autojacvec, and `compile=false` by default but can
  compile the tape under the same circumstances as `ReverseDiffVJP`.
  Only supports ODEs.
- `ReverseDiffAdjoint()`: An implementation of discrete adjoint sensitivity analysis
  using the ReverseDiff.jl tracing-based AD. Supports in-place functions through
  an Array of Structs formulation, and supports out of place through struct of
  arrays.
- `TrackerAdjoint()`: An implementation of discrete adjoint sensitivity analysis
  using the Tracker.jl tracing-based AD. Supports in-place functions through
  an Array of Structs formulation, and supports out of place through struct of
  arrays.
- `ZygoteAdjoint()`: An implementation of discrete adjoint sensitivity analysis
  using the Zygote.jl source-to-source AD directly on the differential equation
  solver. Currently fails.
- `SensitivityADPassThrough()`: Ignores all adjoint definitions and
  proceeds to do standard AD through the `solve` functions.
- `ForwardLSS()`, `AdjointLSS()`, `NILSS(nseg,nstep)`, `NILSAS(nseg,nstep,M)`:
  Implementation of shadowing methods for chaotic systems with a long-time averaged
  objective. See the [sensitivity analysis for chaotic systems (shadowing methods)
  section](@ref shadowing_methods) for more details.

The `ReverseDiffAdjoint()`, `TrackerAdjoint()`, `ZygoteAdjoint()`, and `SensitivityADPassThrough()` algorithms all offer differentiate-through-the-solver adjoints, each based on their respective automatic differentiation packages. If you're not sure which to use, `ReverseDiffAdjoint()` is generally a stable and performant best if using the CPU, while `TrackerAdjoint()` is required if you need GPUs. Note that `SensitivityADPassThrough()` is more or less an internal implementation detail. For example, `ReverseDiffAdjoint()` is implemented by invoking `ReverseDiff`'s AD functionality on `solve(...; sensealg=SensitivityADPassThrough())`.

`ForwardDiffSensitivity` can differentiate code with callbacks when `convert_tspan=true`,
but will be faster when `convert_tspan=false`.
All methods based on discrete adjoint sensitivity analysis via automatic differentiation,
like `ReverseDiffAdjoint`, `TrackerAdjoint`, or `QuadratureAdjoint` are fully
compatible with events. This applies to ODEs, SDEs, DAEs, and DDEs.
The continuous adjoint sensitivities `BacksolveAdjoint`, `InterpolatingAdjoint`,
and `QuadratureAdjoint` are compatible with events for ODEs. `BacksolveAdjoint` and
`InterpolatingAdjoint` can also handle events for SDEs. Use `BacksolveAdjoint` if
the event terminates the time evolution and several states are saved. Currently,
the continuous adjoint sensitivities do not support multiple events per time point.
The shadowing methods are not compatible with callbacks.

### Internal Automatic Differentiation Options (ADKwargs)

Many sensitivity algorithms share the same options for controlling internal
use of automatic differentiation. The following arguments constitute the
`ADKwargs`:

* `autodiff`: Use automatic differentiation in the internal sensitivity algorithm
  computations. Default is `true`.
* `chunk_size`: Chunk size for forward mode differentiation if full Jacobians are
  built (`autojacvec=false` and `autodiff=true`). Default is `0` for automatic
  choice of chunk size.
* `autojacvec`: Calculate the Jacobian-vector (forward sensitivity) or
  vector-Jacobian (adjoint sensitivity analysis) product via automatic
  differentiation with special seeding. For adjoint methods this option requires
  `autodiff=true`. If `autojacvec=false`, then a full Jacobian has to be
  computed, and this will default to using a `f.jac` function provided by the
  user from the problem of the forward pass. Otherwise, if `autodiff=true`
  and `autojacvec=false` then it will use forward-mode AD for the Jacobian,
  otherwise it will fall back to using a numerical approximation to the Jacobian.
  Additionally, if the method is an adjoint method, there are three choices
  which can be made explicitly. The default vjp choice is a polyalgorithm that
  uses a compiler analysis to choose the most efficient vjp for a given code.
    - `TrackerVJP`: Uses Tracker.jl for the vjp.
    - `ZygoteVJP`: Uses Zygote.jl for the vjp.
    - `EnzymeVJP`: Uses Enzyme.jl for the vjp.
    - `ReverseDiffVJP(compile=false)`: Uses ReverseDiff.jl for the vjp. `compile`
      is a boolean for whether to precompile the tape, which should only be done
      if there are no branches (`if` or `while` statements) in the `f` function.
      When applicable, `ReverseDiffVJP(true)` is the fastest method, and then
      `ReverseDiffVJP(false)` is the second fastest, but this method is not
      compatible with third party libraries like Flux.jl, FFTW.jl, etc. (only
      linear algebra and basic mathematics is supported) so it should be swapped
      in only as an optimization.

Note that the Jacobian-vector products and vector-Jacobian products can be
directly specified by the user using the [performance overloads](@ref performance_overloads).

### Choosing a Sensitivity Algorithm

For an analysis of which methods will be most efficient for computing the
solution derivatives for a given problem, consult our analysis
[in this arxiv paper](https://arxiv.org/abs/1812.01892). A general rule of thumb
is:

- `ForwardDiffSensitivity` is the fastest for differential equations with small
  numbers of parameters (<100) and can be used on any differential equation
  solver that is native Julia.
- Adjoint senstivity analysis is the fastest when the number of parameters is
  sufficiently large. There are three configurations of note. Using
  `QuadratureAdjoint` is the fastest for small systems, `BacksolveAdjoint`
  uses the least memory but on very stiff problems it may be unstable and
  require a lot of checkpoints, while `InterpolatingAdjoint` is in the middle,
  allowing checkpointing to control total memory use.
- The methods which use automatic differentiation (`ReverseDiffAdjoint`,
  `TrackerAdjoint`, `ForwardDiffSensitivity`, and `ZygoteAdjoint`) support
  the full range of DifferentialEquations.jl features (SDEs, DDEs, events, etc.),
  but only work on native Julia solvers. The methods which utilize altered differential
  equation systems only work on ODEs (without events), but work on any ODE solver.
- For non-ODEs with large numbers of parameters, `TrackerAdjoint` in out-of-place
  form may be the best performer.
- `TrackerAdjoint` is able to use a `TrackedArray` form with out-of-place
  functions `du = f(u,p,t)` but requires an `Array{TrackedReal}` form for
  `f(du,u,p,t)` mutating `du`. The latter has much more overhead, and should be
  avoided if possible. Thus if solving non-ODEs with lots of parameters, using
  `TrackerAdjoint` with an out-of-place definition may be the current best
  option.


## Choosing a sensealg in a Nutshell

By default, a stable adjoint with an auto-adapting vjp choice is used. In many cases, a user can optimize the choice to compute more than an order of magnitude faster than the default. However, given the vast space to explore, use the following decision tree to help guide the choice:

- If you have 100 parameters or less, consider using forward-mode sensititivites. If the `f` function is not ForwardDiff-compatible, use `ForwardSensitivty`, otherwise use `ForwardDiffSensitivty` as its more efficient.
- For larger equations, give `BacksolveAdjoint` and `InterpolatingAdjoint` a try. If the gradient of `BacksolveAdjoint` is correct, many times it's the faster choice so choose that (but it's not always faster!). If your equation is stiff or a DAE, skip this step as `BacksolveAdjoint` is almost certainly unstable.
- If your equation does not use much memory and you're using a stiff solver, consider using `QuadratureAdjoint` as it is asymtopically more computationally efficient by trading off memory cost.
- If the other methods are all unstable (check the gradients against each other!), then `ReverseDiffAdjoint` is a good fallback on CPU, while `TrackerAdjoint` is a good fallback on GPUs.
- After choosing a general sensealg, if the choice is `InterpolatingAdjoint`, `QuadratureAdjoint`, or `BacksolveAdjoint`, then optimize the choice of vjp calculation next:
  - If your function has no branching (no if statements), use `ReverseDiffVJP(true)`.
  - If you're on the CPU and your function is very scalarized in operations but has branches, choose `ReverseDiffVJP()`.
  - If your on the CPU or GPU and your function is very vectorized, choose `ZygoteVJP()`.
  - Else fallback to `TrackerVJP()` if Zygote does not support the function.

## Additional Details

A sensitivity analysis method can be passed to a solver via the `sensealg`
keyword argument. For example:

```julia
solve(prob,Tsit5(),sensealg=BacksolveAdjoint(autojacvec=ZygoteVJP()))
```

sets the adjoint sensitivity analysis so that, when this call is
encountered in the gradient calculation of any of the Julia reverse-mode
AD frameworks, the differentiation will be replaced with the `BacksolveAdjoint`
method where internal vector-Jacobian products are performed using
Zygote.jl. From the [DifferentialEquations.jl local sensitivity analysis](https://diffeq.sciml.ai/latest/analysis/sensitivity/)
page, we note that the following choices for `sensealg` exist:

- `BacksolveAdjoint`
- `InterpolatingAdjoint` (with checkpoints)
- `QuadratureAdjoint`
- `TrackerAdjoint`
- `ReverseDiffAdjoint` (currently requires `using DistributionsAD`)
- `ZygoteAdjoint` (currently limited to special solvers)

Additionally, there are methodologies for forward sensitivity analysis:

- `ForwardSensitivty`
- `ForwardDiffSensitivty`

These methods have very low overhead compared to adjoint methods but
have poor scaling with respect to increased numbers of parameters.
[Our benchmarks demonstrate a cutoff of around 100 parameters](https://arxiv.org/abs/1812.01892),
where for models with less than 100 parameters these techniques are more
efficient, but when there are more than 100 parameters (like in neural ODEs)
these methods are less efficient than the adjoint methods.

## Choices of Vector-Jacobian Products (autojacvec)

With each of these solvers, `autojacvec` can be utilized to choose how
the internal vector-Jacobian products of the `f` function are computed.
The choices are:

- `ReverseDiffVJP(compile::Bool)`: Usually the fastest when scalarized operations exist in the `f` function (like
  in scientific machine learning applications like Universal Differential
  Equations) and the boolean `compile` (i.e. `ReverseDiffVJP(true)`)
  is the absolute fastest but requires that the `f` function of the
  ODE/DAE/SDE/DDE has no branching. Does not support GPUs. 

- `TrackerVJP`: Not as efficient as `ReverseDiffVJP`, but supports GPUs. 

- `ZygoteVJP`: Tends to be the fastest VJP method if the ODE/DAE/SDE/DDE is written with mostly vectorized functions (like neural networks and
  other layers from [Flux.jl](https://fluxml.ai/)). Bear in mind that Zygote does not allow mutation, making the solve more memory expensive and therefore slow.

- `nothing`: Default choice given characteristics of the types in your model.
- `true`: Forward-mode AD Jacobian-vector products. Should only be used on sufficiently small equations

- `false`: Numerical Jacobian-vector products. Should only be used if the `f` function is not differentiable
  (i.e. is a Fortran code).

As other vector-Jacobian product systems become available
in Julia they will be added to this system so that no user code changes
are required to interface with these methodologies. 

## Manual VJPs

Note that when defining your differential equation the vjp can be
manually overwritten by providing a `vjp(u,p,t)` that returns a tuple
`f(u,p,t),v->J*v` in the form of [ChainRules.jl](https://www.juliadiff.org/ChainRulesCore.jl/stable/).
When this is done, the choice of `ZygoteVJP` will utilize your VJP
function during the internal steps of the adjoint. This is useful for
models where automatic differentiation may have trouble producing
optimal code. This can be paired with [ModelingToolkit.jl](https://github.com/SciML/ModelingToolkit.jl)
for producing hyper-optimized, sparse, and parallel VJP functions utilizing
the automated symbolic conversions.

## Optimize-then-Discretize

[The original neural ODE paper](https://arxiv.org/abs/1806.07366)
popularized optimize-then-discretize with O(1) adjoints via backsolve.
This is the methodology `BacksolveAdjoint`
When training non-stiff neural ODEs, `BacksolveAdjoint` with `ZygoteVJP`
is generally the fastest method. Additionally, this method does not
require storing the values of any intermediate points and is thus the
most memory efficient. However, `BacksolveAdjoint` is prone
to instabilities whenever the Lipschitz constant is sufficiently large,
like in stiff equations, PDE discretizations, and many other contexts,
so it is not used by default. When training a neural ODE for machine
learning applications, the user should try `BacksolveAdjoint` and see
if it is sufficiently accurate on their problem.

Note that DiffEqFlux's implementation of `BacksolveAdjoint` includes
an extra feature `BacksolveAdjoint(checkpointing=true)` which mixes
checkpointing with `BacksolveAdjoint`. What this method does is that,
at `saveat` points, values from the forward pass are saved. Since the
reverse solve should numerically be the same as the forward pass, issues
with divergence of the reverse pass are mitigated by restarting the
reverse pass at the `saveat` value from the forward pass. This reduces
the divergence and can lead to better gradients at the cost of higher
memory usage due to having to save some values of the forward pass.
This can stabilize the adjoint in some applications, but for highly
stiff applications the divergence can be too fast for this to work in
practice.

To avoid the issues of backwards solving the ODE, `InterpolatingAdjoint`
and `QuadratureAdjoint` utilize information from the forward pass.
By default these methods utilize the [continuous solution](https://diffeq.sciml.ai/latest/basics/solution/#Interpolations-1)
provided by DifferentialEquations.jl in the calculations of the
adjoint pass. `QuadratureAdjoint` uses this to build a continuous
function for the solution of adjoint equation and then performs an
adaptive quadrature via [Quadrature.jl](https://github.com/SciML/Quadrature.jl),
while `InterpolatingAdjoint` appends the integrand to the ODE so it's
computed simultaneously to the Lagrange multiplier. When memory is
not an issue, we find that the `QuadratureAdjoint` approach tends to
be the most efficient as it has a significantly smaller adjoint
differential equation and the quadrature converges very fast, but this
form requires holding the full continuous solution of the adjoint which
can be a significant burden for large parameter problems. The
`InterpolatingAdjoint` is thus a compromise between memory efficiency
and compute efficiency, and is in the same spirit as [CVODES](https://computing.llnl.gov/projects/sundials).

However, if the memory cost of the `InterpolatingAdjoint` is too high,
checkpointing can be used via `InterpolatingAdjoint(checkpointing=true)`.
When this is used, the checkpoints default to `sol.t` of the forward
pass (i.e. the saved timepoints usually set by `saveat`). Then in the
adjoint, intervals of `sol.t[i-1]` to `sol.t[i]` are re-solved in order
to obtain a short interpolation which can be utilized in the adjoints.
This at most results in two full solves of the forward pass, but
dramatically reduces the computational cost while being a low-memory
format. This is the preferred method for highly stiff equations
when memory is an issue, i.e. stiff PDEs or large neural DAEs.

For forward-mode, the `ForwardSensitivty` is the version that performs
the optimize-then-discretize approach. In this case, `autojacvec` corresponds
to the method for computing `J*v` within the forward sensitivity equations,
which is either `true` or `false` for whether to use Jacobian-free
forward-mode AD (via ForwardDiff.jl) or Jacobian-free numerical
differentiation.

## Discretize-then-Optimize

In this approach the discretization is done first and then optimization
is done on the discretized system. While traditionally this can be
done discrete sensitivity analysis, this is can be equivalently done
by automatic differentiation on the solver itself. `ReverseDiffAdjoint`
performs reverse-mode automatic differentiation on the solver via
[ReverseDiff.jl](https://github.com/JuliaDiff/ReverseDiff.jl),
`ZygoteAdjoint` performs reverse-mode automatic
differentiation on the solver via
[Zygote.jl](https://github.com/FluxML/Zygote.jl), and `TrackerAdjoint`
performs reverse-mode automatic differentiation on the solver via
[Tracker.jl](https://github.com/FluxML/Tracker.jl). In addition,
`ForwardDiffSensitivty` performs forward-mode automatic differentiation
on the solver via [ForwardDiff.jl](https://github.com/JuliaDiff/ForwardDiff.jl).

We note that many studies have suggested that [this approach produces
more accurate gradients than the optimize-than-discretize approach](https://arxiv.org/abs/2005.13420)

# Special Notes on Equation Types

While all of the choices are compatible with ordinary differential
equations, specific notices apply to other forms:

## Differential-Algebraic Equations

We note that while all 3 are compatible with index-1 DAEs via the
[derivation in the universal differential equations paper](https://arxiv.org/abs/2001.04385)
(note the reinitialization), we do not recommend `BacksolveAdjoint`
one DAEs because the stiffness inherent in these problems tends to
cause major difficulties with the accuracy of the backwards solution
due to reinitialization of the algebraic variables.

## Stochastic Differential Equations

We note that all of the adjoints except `QuadratureAdjoint` are applicable
to stochastic differential equations.

## Delay Differential Equations

We note that only the discretize-then-optimize methods are applicable
to delay differential equations. Constant lag and variable lag
delay differential equation parameters can be estimated, but the lag
times themselves are unable to be estimated through these automatic
differentiation techniques.






# Controlling Automatic Differentiation

One of the key features of DiffEqFlux.jl is the fact that it has many modes
of differentiation which are available, allowing neural differential equations
and universal differential equations to be fit in the manner that is most
appropriate.

To use the automatic differentiation overloads, the differential equation
just needs to be solved with `solve`. Thus, for example,

```julia
using DiffEqSensitivity, OrdinaryDiffEq, Zygote

function fiip(du,u,p,t)
  du[1] = dx = p[1]*u[1] - p[2]*u[1]*u[2]
  du[2] = dy = -p[3]*u[2] + p[4]*u[1]*u[2]
end
p = [1.5,1.0,3.0,1.0]; u0 = [1.0;1.0]
prob = ODEProblem(fiip,u0,(0.0,10.0),p)
sol = solve(prob,Tsit5())
loss(u0,p) = sum(solve(prob,Tsit5(),u0=u0,p=p,saveat=0.1))
du0,dp = Zygote.gradient(loss,u0,p)
```

will compute the gradient of the loss function "sum of the values of the
solution to the ODE at timepoints dt=0.1" using an adjoint method, where `du0`
is the derivative of the loss function with respect to the initial condition
and `dp` is the derivative of the loss function with respect to the parameters.

## Choosing a Differentiation Method

The choice of the method for calculating the gradient is made by passing the
keyword argument `sensealg` to `solve`. The default choice is dependent
on the type of differential equation and the choice of neural network architecture.

The full listing of differentiation methods is described in the
[DifferentialEquations.jl documentation](https://diffeq.sciml.ai/latest/analysis/sensitivity/#Sensitivity-Algorithms-1).
That page also has guidelines on how to make the right choice.


### Applicability of Backsolve and Caution

When `BacksolveAdjoint` is applicable it is a fast method and requires the least memory.
However, one must be cautious because not all ODEs are stable under backwards integration
by the majority of ODE solvers. An example of such an equation is the Lorenz equation.
Notice that if one solves the Lorenz equation forward and then in reverse with any
adaptive time step and non-reversible integrator, then the backwards solution diverges
from the forward solution. As a quick demonstration:

```julia
using Sundials
function lorenz(du,u,p,t)
 du[1] = 10.0*(u[2]-u[1])
 du[2] = u[1]*(28.0-u[3]) - u[2]
 du[3] = u[1]*u[2] - (8/3)*u[3]
end
u0 = [1.0;0.0;0.0]
tspan = (0.0,100.0)
prob = ODEProblem(lorenz,u0,tspan)
sol = solve(prob,Tsit5(),reltol=1e-12,abstol=1e-12)
prob2 = ODEProblem(lorenz,sol[end],(100.0,0.0))
sol = solve(prob,Tsit5(),reltol=1e-12,abstol=1e-12)
@show sol[end]-u0 #[-3.22091, -1.49394, 21.3435]
```

Thus one should check the stability of the backsolve on their type of problem before
enabling this method. Additionally, using checkpointing with backsolve can be a
low memory way to stabilize it.
