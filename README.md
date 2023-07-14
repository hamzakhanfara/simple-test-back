# SKF Digital Twin PoC Entry Point

Here you can use the model AMB Axial.

- Goal: reduce ratio of (runtime / simulation time) to <= 0.1
- Hardware: JuliaHub machine with 8vCPUs and 8GB memory per vCPU.

First, data is loaded, model parameters and ODE problem defined.

````julia
ModelDataName = "Demo_E300-30V2_ax_cauer.mat"
exct = 4000
exct_t = 1
simulation_time = 5
include("Model_palier_axial_interpolations.jl");
````

## Original Solve

The solver and keyword arguments are defined in order to capture dynamics of the stiff system.
However, RK4 is not a stiff solver so while the following solve produces accurate results,
it sacrifices performance in order to achieve it.

The first solve task ~40 seconds to complete.

````julia
@time sol = solve(prob, RK4(), adaptive=false, dt=2.3809523666666669e-5);
bsol = minimum(@benchmark solve(prob, RK4(), adaptive=false, dt=2.3809523666666669e-5))
````

````
BenchmarkTools.TrialEstimate: 
  time:             42.367 s
  gctime:           7.288 s (17.20%)
  memory:           113.12 GiB
  allocs:           310801901
````

Calculate runtime-to-simtime ratio for starting point.

````julia
ns2sec(t) = t / 1_000_000_000
ratio(ns2sec(bsol.time), simulation_time)
````

````
8.4734560558
````

## New Solve

- Changing to the CVode Backward Differentiation Formula (BDF) solver provides better performance.
- Using the `saveat` keyword allows us to retain the same level of granularity as in the original.
  There is no need to use the `adaptive` option because the solver will interpolate each `saveat`
  point if it is ever overshot in a step.
- When defining the ODE problem, we define the sparcity pattern.
  To learn more about the default and other options, please see
  [Sparsity Handling](https://docs.sciml.ai/SciMLBase/stable/interfaces/SciMLFunctions/#Sparsity-Handling).

The new problem is defined as follows in `Model_palier_axial_interpolations.jl`:

```julia
ODEProblem(connected_simp, x0, (0,simulation_time), P; sparse=true, sparsity=true)
```

The first solve task ~5 seconds which is already a nice improvement.

````julia
using Sundials
@time sol2 = solve(prob, CVODE_BDF(), saveat=2.3809523666666669e-5);
bsol2 = minimum(@benchmark solve(prob, CVODE_BDF(), saveat=2.3809523666666669e-5))
````

````
BenchmarkTools.TrialEstimate: 
  time:             221.398 ms
  gctime:           0.000 ns (0.00%)
  memory:           482.91 MiB
  allocs:           1665760
````

Calculate runtime-to-simtime ratio for simulation time of 5.

````julia
ratio(ns2sec(bsol2.time), simulation_time)
````

````
0.0442795674
````

Calculate the speedup.

````julia
ratio(ns2sec(bsol.time), ns2sec(bsol2.time))
````

````
191.36266574727196
````

## Validating Results

To ensure the new solve produces the same results, let's plot each solution.

````julia
dict = Dict(
    "Current" => sol[amb.electricModel.Cu[1]],
    "Force" => sol[amb.force.F],
    "Flux1" => sol[amb.flux.Flux[1]],
    "Flux2" => sol[amb.flux.Flux[2]],
    "Position" => sol[rotor.Pos],
    "Tension" => sol[cable.Uc[1]],
    "Time" => sol.t
)
dict2 = Dict(
    "Current" => sol2[amb.electricModel.Cu[1]],
    "Force" => sol2[amb.force.F],
    "Flux1" => sol2[amb.flux.Flux[1]],
    "Flux2" => sol2[amb.flux.Flux[2]],
    "Position" => sol2[rotor.Pos],
    "Tension" => sol2[cable.Uc[1]],
    "Time" => sol2.t
)

# matwrite("test-2023-07.mat", dict)

using Plots
plts = []
for key in keys(dict)
    if key == "Time"
        continue
    end
    plt = plot(
        dict["Time"][1:1000:end],
        dict[key][1:1000:end],
        title=key,
        xlabel="Time",
        label="Old (slow)"
    )
    plot!(
        plt,
        dict2["Time"][1:1000:end],
        dict2[key][1:1000:end],
        label="New (fast)",
        style=:dash
    )
    push!(plts, plt)
end
plot(plts...)
````
![](README-14.svg)

## Goal Comparison with Simulation Time of 30 Seconds

Now that we have confirmed the new solve options achieve the same results,
we remake the ODE problem to represent a longer simulation time.

````julia
prob_new = remake(prob; tspan=(0, 30));
@time sol_new = solve(prob_new, CVODE_BDF(), saveat=2.3809523666666669e-5);
bsol_new = minimum(@benchmark solve(prob_new, CVODE_BDF(), saveat=2.3809523666666669e-5))
````

````
BenchmarkTools.TrialEstimate: 
  time:             815.758 ms
  gctime:           0.000 ns (0.00%)
  memory:           1.10 GiB
  allocs:           5079921
````

Calculate runtime-to-simtime ratio for simulation time of 30.

````julia
ratio(ns2sec(bsol_new.time), 30)
````

````
0.027191940133333332
````

---

Weave document with:

```julia
using Literate
Literate.markdown("main.jl";
    config = Dict(
        "name" => "README",
        "execute" => true,
        "flavor" => Literate.CommonMarkFlavor()
    )
)
```

---

*This page was generated using [Literate.jl](https://github.com/fredrikekre/Literate.jl).*

