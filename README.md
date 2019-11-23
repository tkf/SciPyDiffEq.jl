# SciPyDiffEq.jl

[![Join the chat at https://gitter.im/JuliaDiffEq/Lobby](https://badges.gitter.im/JuliaDiffEq/Lobby.svg)](https://gitter.im/JuliaDiffEq/Lobby?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

SciPyDiffEq.jl is a common interface binding for the
[SciPy solve_ivp module](https://docs.scipy.org/doc/scipy/reference/generated/scipy.integrate.solve_ivp.html#scipy.integrate.solve_ivp)
ordinary differential equation solvers. It uses the
[PyCall.jl](https://github.com/JuliaPy/PyCall.jl) interop in order to
send the differential equation over to Python and solve it.

Note that this package isn't for production use and is mostly just for benchmarking.
For well-developed differential equation package, see
[DifferentialEquations.jl](https://github.com/JuliaDiffEq/DifferentialEquations.jl).

## Installation

To install SciPyDiffEq.jl, use the following:

```julia
Pkg.clone("https://github.com/JuliaDiffEq/SciPyDiffEq.jl")
```

## Using SciPyDiffEq.jl

SciPyDiffEq.jl is simply a solver on the DiffEq common interface, so for details see the [DifferentialEquations.jl documentation](https://juliadiffeq.github.io/DiffEqDocs.jl/dev/).
The available algorithms are:

```julia
SciPyDiffEq.RK45
SciPyDiffEq.RK23
SciPyDiffEq.Radau
SciPyDiffEq.BDF
SciPyDiffEq.LSODA
```

## Example

```julia
using SciPyDiffEq

function lorenz(u,p,t)
 du1 = 10.0(u[2]-u[1])
 du2 = u[1]*(28.0-u[3]) - u[2]
 du3 = u[1]*u[2] - (8/3)*u[3]
 [du1, du2, du3]
end
tspan = (0.0,10.0)
u0 = [1.0,0.0,0.0]
prob = ODEProblem(lorenz,u0,tspan)
sol = solve(prob,SciPyDiffEq.RK45())
```

## Measuring Overhead

In the following we can measure the overhead and show that using SciPy from Julia
is about 38x faster than using SciPy with Numba. Using SciPyDiffEq:

```julia
using SciPyDiffEq, BenchmarkTools

function lorenz(u,p,t)
    du1 = 10.0(u[2]-u[1])
    du2 = u[1]*(28.0-u[3]) - u[2]
    du3 = u[1]*u[2] - (8/3)*u[3]
    [du1, du2, du3]
end
tspan = (0.0,10.0)
u0 = [1.0,0.0,0.0]
prob = ODEProblem(lorenz,u0,tspan)
@btime sol = solve(prob,SciPyDiffEq.RK45(),dense=false, abstol=1e-8,reltol=1e-8) # 211.784 ms (315699 allocations: 13.08 MiB)
```

This gives 211ms. Solving the equivalent problem with SciPy is:

```py
import numpy as np
from scipy.integrate import odeint
import timeit
import numba
def f(u, t):
    x, y, z = u
    return [10.0 * (y - x), x * (28.0 - z) - y, x * y - 2.66 * z]

u0 = [1.0,0.0,0.0]
tspan = (0., 100.)
t = np.linspace(0, 100, 1001)
sol = odeint(f, u0, t)
def time_func():
    odeint(f, u0, t, rtol = 1e-8, atol=1e-8)

_t = timeit.Timer(time_func).timeit(number=100)
print(_t) # 13.898981100000015 seconds
```

which takes 13.89 seconds. Then using Numba JIT with nopython mode is:

```py
numba_f = numba.jit(f,nopython=True)
odeint(numba_f, u0, t,rtol = 1e-8, atol=1e-8)

def time_func():
   odeint(numba_f, u0, t,rtol = 1e-8, atol=1e-8)

_t = timeit.Timer(time_func).timeit(number=100)
print(_t) # 8.05035870000006 seconds
```

which takes 8 seconds. Together, this showcases a 38x speedup by using the Julia
based interface, so overhead concerns in future benchmarks are gone.

## Benchmarks

```julia
using ParameterizedFunctions, MATLABDiffEq, OrdinaryDiffEq, ODEInterface,
      ODEInterfaceDiffEq, Plots, Sundials, SciPyDiffEq
using DiffEqDevTools
using LinearAlgebra

## Non-Stiff Problem 1: Lotka-Volterra

f = @ode_def_bare LotkaVolterra begin
  dx = a*x - b*x*y
  dy = -c*y + d*x*y
end a b c d
p = [1.5,1,3,1]
tspan = (0.0,10.0)
u0 = [1.0,1.0]
prob = ODEProblem(f,u0,tspan,p)
sol = solve(prob,Vern7(),abstol=1/10^14,reltol=1/10^14)
test_sol = TestSolution(sol)

setups = [Dict(:alg=>DP5())
          Dict(:alg=>dopri5())
          Dict(:alg=>Tsit5())
          Dict(:alg=>Vern7())
          Dict(:alg=>MATLABDiffEq.ode45())
          Dict(:alg=>SciPyDiffEq.RK45())
          Dict(:alg=>SciPyDiffEq.LSODA())
          Dict(:alg=>MATLABDiffEq.ode113())
          Dict(:alg=>CVODE_Adams())
  ]
abstols = 1.0 ./ 10.0 .^ (6:13)
reltols = 1.0 ./ 10.0 .^ (3:10)
wp = WorkPrecisionSet(prob,abstols,reltols,setups;appxsol=test_sol,dense=false,
                      save_everystep=false,numruns=100,maxiters=10000000,
                      timeseries_errors=false,verbose=false)
plot(wp,title="Non-stiff 1: Lotka-Volterra")
savefig("benchmark1.png")
```

![benchmark1](https://user-images.githubusercontent.com/1814174/69478405-f9dac480-0dbf-11ea-8f8e-5572b86d2a97.png)

```julia
## Non-Stiff Problem 2: Rigid Body

f = @ode_def_bare RigidBodyBench begin
  dy1  = -2*y2*y3
  dy2  = 1.25*y1*y3
  dy3  = -0.5*y1*y2 + 0.25*sin(t)^2
end
prob = ODEProblem(f,[1.0;0.0;0.9],(0.0,100.0))
sol = solve(prob,Vern7(),abstol=1/10^14,reltol=1/10^14)
test_sol = TestSolution(sol)

setups = [Dict(:alg=>DP5())
          Dict(:alg=>dopri5())
          Dict(:alg=>Tsit5())
          Dict(:alg=>Vern7())
          Dict(:alg=>MATLABDiffEq.ode45())
          Dict(:alg=>SciPyDiffEq.RK45())
          Dict(:alg=>SciPyDiffEq.LSODA())
          Dict(:alg=>MATLABDiffEq.ode113())
          Dict(:alg=>CVODE_Adams())
  ]
abstols = 1.0 ./ 10.0 .^ (6:13)
reltols = 1.0 ./ 10.0 .^ (3:10)
wp = WorkPrecisionSet(prob,abstols,reltols,setups;appxsol=test_sol,dense=false,
                      save_everystep=false,numruns=100,maxiters=10000000,
                      timeseries_errors=false,verbose=false)
plot(wp,title="Non-stiff 2: Rigid-Body")
savefig("benchmark2.png")
```

![benchmark2](https://user-images.githubusercontent.com/1814174/69478406-fe9f7880-0dbf-11ea-8f15-a0a0f3e8717f.png)


```julia
## Stiff Problem 1: ROBER

rober = @ode_def begin
  dy₁ = -k₁*y₁+k₃*y₂*y₃
  dy₂ =  k₁*y₁-k₂*y₂^2-k₃*y₂*y₃
  dy₃ =  k₂*y₂^2
end k₁ k₂ k₃
prob = ODEProblem(rober,[1.0,0.0,0.0],(0.0,1e5),[0.04,3e7,1e4])
sol = solve(prob,CVODE_BDF(),abstol=1/10^14,reltol=1/10^14)
test_sol = TestSolution(sol)

abstols = 1.0 ./ 10.0 .^ (5:8)
reltols = 1.0 ./ 10.0 .^ (1:4);
setups = [Dict(:alg=>Rosenbrock23()),
          Dict(:alg=>TRBDF2()),
          Dict(:alg=>rodas()),
          Dict(:alg=>radau()),
          Dict(:alg=>RadauIIA5()),
          Dict(:alg=>SciPyDiffEq.LSODA())
          Dict(:alg=>SciPyDiffEq.Radau())
          Dict(:alg=>SciPyDiffEq.BDF())
          Dict(:alg=>MATLABDiffEq.ode23s())
          Dict(:alg=>MATLABDiffEq.ode15s())
          ]
          
wp = WorkPrecisionSet(prob,abstols,reltols,setups;
                      save_everystep=false,appxsol=test_sol,maxiters=Int(1e5),numruns=100)
plot(wp,title="Stiff 1: ROBER")
savefig("benchmark3.png")
```

![benchmark3](https://user-images.githubusercontent.com/1814174/69478407-ffd0a580-0dbf-11ea-9311-909bd3938b42.png)


```julia
## Stiff Problem 2: HIRES

f = @ode_def Hires begin
  dy1 = -1.71*y1 + 0.43*y2 + 8.32*y3 + 0.0007
  dy2 = 1.71*y1 - 8.75*y2
  dy3 = -10.03*y3 + 0.43*y4 + 0.035*y5
  dy4 = 8.32*y2 + 1.71*y3 - 1.12*y4
  dy5 = -1.745*y5 + 0.43*y6 + 0.43*y7
  dy6 = -280.0*y6*y8 + 0.69*y4 + 1.71*y5 -
           0.43*y6 + 0.69*y7
  dy7 = 280.0*y6*y8 - 1.81*y7
  dy8 = -280.0*y6*y8 + 1.81*y7
end

u0 = zeros(8)
u0[1] = 1
u0[8] = 0.0057
prob = ODEProblem(f,u0,(0.0,321.8122))

sol = solve(prob,Rodas5(),abstol=1/10^14,reltol=1/10^14)
test_sol = TestSolution(sol)

abstols = 1.0 ./ 10.0 .^ (5:8)
reltols = 1.0 ./ 10.0 .^ (1:4);
setups = [Dict(:alg=>Rosenbrock23()),
          Dict(:alg=>TRBDF2()),
          Dict(:alg=>rodas()),
          Dict(:alg=>radau()),
          Dict(:alg=>RadauIIA5()),
          Dict(:alg=>SciPyDiffEq.LSODA())
          Dict(:alg=>SciPyDiffEq.Radau())
          Dict(:alg=>SciPyDiffEq.BDF())
          Dict(:alg=>MATLABDiffEq.ode23s())
          Dict(:alg=>MATLABDiffEq.ode15s())
          ]

wp = WorkPrecisionSet(prob,abstols,reltols,setups;
                      save_everystep=false,appxsol=test_sol,maxiters=Int(1e5),numruns=100)
plot(wp,title="Stiff 2: Hires")
savefig("benchmark4.png")
```

![benchmark4](https://user-images.githubusercontent.com/1814174/69478409-019a6900-0dc0-11ea-9fce-c11a5e4de9a4.png)
