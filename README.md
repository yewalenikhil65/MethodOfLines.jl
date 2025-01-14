# MethodOfLines.jl

Provides automatic discretization of symbolic PDE systems as defined with ModelingToolkit.jl.

Feature requests and issues welcome.

## Usage:
```
discretization = MOLFiniteDifference(dxs, 
                                      <your choice of continuous variable, usually time>; 
                                      upwind_order = <Currently unstable at any value other than 1>, 
                                      approx_order = <Order of derivative approximation, starting from 2> 
                                      grid_align = <your grid type choice>)
prob = discretize(pdesys, discretization)
```
Where `dxs` is a vector of pairs of parameters to the grid step in this dimension, i.e. `[x=>0.2, y=>0.1]`

Note that the second argument to `MOLFiniteDifference` is optional, all parameters can be discretized if all required boundary conditions are specified.

Currently supported grid types: `center_align` and `edge_align`. Edge align will give better accuracy with Neumann Boundary conditions.

`center_align`: naive grid, starting from lower boundary, ending on upper boundary with step of `dx`

`edge_align`: offset grid, set halfway between the points that would be generated with center_align, with extra points at either end that are above and below the supremum and infimum by `dx/2`. This improves accuracy for neumann BCs.

If you find that your system throws errors, please post an issue with the code and we will endeavor to support it.

## Assumptions
- That the grid is cartesian.
- That periodic boundary conditions are of the simple form `u(t, x_min) ~ u(t, x_max)`, or the same with lhs and rhs reversed. Note that this generalises to higher dimensions
- That boundary conditions do not contain references to derivatives which are not in the direction of the boundary, except in time.
- That initial conditions are of the form `u(...) ~ ...`, and don't reference the initial time derivative.
- That simple derivative terms are purely of a dependant variable, for example `Dx(u(t,x,y))` is allowed but `Dx(u(t,x,y)*v(t,x,y))`, `Dx(u(t,x)+1)` or `Dx(f(u(t,x)))` are not. As a workaround please expand such terms with the product/chain rules and use the linearity of the derivative operator, or define a new dependant variable by adding an equation for it like `eqs = [Differential(x)(w(t,x))~ ... , w(t,x) ~ v(t,x)*u(t,x)]`. An exception to this is if the differential is a nonlinear or spherical laplacian, in which case only the innermost argument should be wrapped.

If any of these limitations are a problem for you please post an issue and we will prioritize removing them.

## Coming soon:
- Fewer Assumptions.
- More robust testing and validation.
- Benchmarks.
## Full Example:
```
## 2D Diffusion

# Dependencies
using ModelingToolkit, MethodOfLines, LinearAlgebra, OrdinaryDiffEq, DomainSets
using ModelingToolkit: Differential

# Variables, parameters, and derivatives
@parameters t x y
@variables u(..)
Dxx = Differential(x)^2
Dyy = Differential(y)^2
Dt = Differential(t)
# Domain edges
t_min= 0.
t_max = 2.0
x_min = 0.
x_max = 2.
y_min = 0.
y_max = 2.

# Discretization parameters
dx = 0.1; dy = 0.2
order = 2

# Analytic solution for boundary conditions
analytic_sol_func(t, x, y) = exp(x + y) * cos(x + y + 4t)

# Equation
eq  = Dt(u(t, x, y)) ~ Dxx(u(t, x, y)) + Dyy(u(t, x, y))

# Initial and boundary conditions
bcs = [u(t_min,x,y) ~ analytic_sol_func(t_min, x, y),
        u(t, x_min, y) ~ analytic_sol_func(t, x_min, y),
        u(t, x_max, y) ~ analytic_sol_func(t, x_max, y),
        u(t, x,y_min) ~ analytic_sol_func(t, x, y_min),
        u(t, x, y_max) ~ analytic_sol_func(t, x, y_max)]

# Space and time domains
domains = [t ∈ Interval(t_min, t_max),
           x ∈ Interval(x_min, x_max),
           y ∈ Interval(y_min, y_max)]

# PDE system
@named pdesys = PDESystem([eq], bcs, domains, [t, x, y], [u(t, x, y)])

# Method of lines discretization
discretization = MOLFiniteDifference([x=>dx, y=>dy], t; approx_order = order)
prob = ModelingToolkit.discretize(pdesys, discretization)

# Solution of the ODE system
sol = solve(prob,Tsit5())
```
