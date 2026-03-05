# Kinetica.jl

[![CC BY-NC-SA 4.0][cc-by-nc-sa-shield]][cc-by-nc-sa]

Kinetica.jl is a Julia package for building and simulating **chemical reaction networks (CRNs)**.

If you are new to Julia, this README is designed as a practical “start here” guide:

- what Kinetica does,
- how to install it,
- how the package is organized,
- and the basic workflow you will follow in scripts.

For full tutorials and API docs, use the documentation site:  
<https://kinetica-jl.github.io/Kinetica.jl/stable/>

---

## 1) What Kinetica does

Kinetica helps you:

1. Define experimental conditions (e.g., temperature profiles).
2. Explore reaction space to build a CRN (via CDE).
3. Solve the CRN kinetics over time with SciML ODE solvers.
4. Save, reload, and analyze simulation outputs.

Under the hood it uses Julia/SciML packages such as:

- [Catalyst.jl](https://github.com/SciML/Catalyst.jl)
- [ModelingToolkit.jl](https://github.com/SciML/ModelingToolkit.jl)
- [DifferentialEquations.jl](https://github.com/SciML/DifferentialEquations.jl)

It also uses Python tooling (via `PythonCall` + `CondaPkg`) for chemistry interfaces like Open Babel, RDKit, ASE, and autodE.

---

## 2) Installation (beginner-friendly)

### Option A: Install released package (recommended)

In a Julia REPL:

```julia
using Pkg
Pkg.Registry.add(Pkg.RegistrySpec(url="https://github.com/Kinetica-jl/KineticaRegistry"))
Pkg.add("Kinetica")
```

Then test import:

```julia
using Kinetica
```

> First import may take longer because Python dependencies are resolved for the project environment.

### Option B: Work on local source checkout

From Julia REPL, in your own project environment:

```julia
using Pkg
Pkg.develop(path="/path/to/Kinetica.jl")
```

---

## 3) Julia basics you need first

If you are brand new to Julia, the essentials for using Kinetica are:

- **Packages/environment:** `] activate .` and `] add ...` in the package manager.
- **Running scripts:** `julia --project=. your_script.jl`
- **Dictionaries and symbols:** Kinetica uses `Dict(:T => ..., :P => ...)` patterns heavily.
- **Keyword constructors:** Many Kinetica config types are created with named arguments.

---

## 4) Quick workflow mental model

Most Kinetica scripts follow this shape:

1. Build a `ConditionSet`.
2. Build `ODESimulationParams`.
3. Build exploration method (`DirectExplore` or `IterativeExplore`) **or** load an existing CRN.
4. Build a calculator (e.g., `PrecalculatedArrheniusCalculator`).
5. Build solve method (`StaticODESolve` or `VariableODESolve`).
6. Run `explore_network(...)` (exploration + solve) or `solve_network(...)` (solve existing CRN).
7. Analyze/save with `plot`, `save_output`, `load_output`.

---

## 5) Package walkthrough (where things live)

### `src/conditions/`

Defines experimental condition profiles and `ConditionSet`.

- Static condition: constant value.
- Variable condition: profile over time (e.g., linear ramps).
- Optional discrete update timestep (`ts_update`) for rate updates.

### `src/exploration/`

Defines CRN exploration logic.

- `DirectExplore`: radius-limited direct exploration.
- `IterativeExplore`: kinetics-guided level-by-level exploration.
- `CDE`: configuration wrapper for CDE sampling runs.

### `src/solving/`

Defines ODE simulation parameters and solve methods.

- `ODESimulationParams`: solver tolerances, chunking, save interval, etc.
- `StaticODESolve` / `VariableODESolve`: bind params + conditions + calculator.
- `solve_network`: run kinetic simulation on existing CRN.

### `src/analysis/`

Defines output containers and I/O.

- `ODESolveOutput`: main result container.
- `save_output` / `load_output`: BSON-based persistence.
- Plot recipes for quick inspection.

### `src/openbabel/`, `src/rdkit/`, `src/ase/`, `src/autode/`

Python-backed chemistry interfaces used by exploration and calculator workflows.

---

## 6) Your first script pattern

Use this as a template and fill in your own exploration/calculator details.

```julia
using Kinetica
using OrdinaryDiffEq

# 1) Conditions
conditions = ConditionSet(Dict(
	:T => LinearGradientProfile(
		rate = 50.0,
		X_start = 500.0,
		X_end = 1200.0
	)
))

# 2) ODE simulation parameters
pars = ODESimulationParams(
	tspan = (0.0, get_t_final(conditions)),
	u0 = Dict("C" => 1.0),
	solver = Rosenbrock23()
)

# 3) Calculator and solve method (example calculator shown)
#    Replace Ea/A with your values or loaded arrays.
Ea = [1.0e5, 1.2e5]
A  = [1.0e10, 5.0e9]
calc = PrecalculatedArrheniusCalculator(Ea, A)
solvemethod = VariableODESolve(pars, conditions, calc)

# 4) If you already have sd, rd from an existing network:
# res = solve_network(solvemethod, sd, rd)

# 5) Or run full exploration + solve:
# exploremethod = DirectExplore(...)
# res = explore_network(exploremethod, solvemethod; savedir="./output")

# 6) Save/reload
# save_output(res, "./output/final.bson")
# reloaded = load_output("./output/final.bson")
```

For a complete runnable tutorial with prepared example inputs, see:  
`docs/src/getting-started.md`

---

## 7) Important beginner notes

- **`StaticODESolve` vs `VariableODESolve`:**
  - use `StaticODESolve` when all conditions are constants,
  - use `VariableODESolve` when one or more conditions change over time.
- **Chunkwise solving (`solve_chunks=true`)** is enabled by default to improve stability for long simulations.
- **Time units:** helper `tconvert(...)` is available for unit conversion (e.g. ms → s).
- **First-run setup:** if Python dependencies are not yet resolved in your environment, first import/compile steps can be slow.

---

## 8) Troubleshooting

- If `using Kinetica` fails, ensure your Julia project is activated and dependencies are installed.
- If CDE-related exploration fails, verify your template directory path and run permissions.
- If old BSON outputs fail to load cleanly after version upgrades, regenerate reaction hashes with `get_rhash` (see docs note below).

> **Compatibility note (v0.7+):**
> Kinetica v0.7 updates dependencies including StableHashTraits. Some older BSON CRNs may require re-hashing with `get_rhash`. Raw directory-tree CRNs imported with `import_network` are generally unaffected.

---

## 9) Learn more

- Docs home: <https://kinetica-jl.github.io/Kinetica.jl/stable/>
- Getting started tutorial: `docs/src/getting-started.md`
- ODE solving deep dive: `docs/src/tutorials/ode-solution.md`

---

## Citation

If you use Kinetica.jl in your research, please cite:

> Gilkes, J., Storr, M. T., Maurer, R. J., & Habershon, S. (2024). Predicting Long-Time-Scale Kinetics under Variable Experimental Conditions with Kinetica.jl. *Journal of Chemical Theory and Computation, 20*(12), 5196–5214. https://doi.org/10.1021/acs.jctc.4c00333

## License

This work is licensed under a [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License][cc-by-nc-sa].

[cc-by-nc-sa]: http://creativecommons.org/licenses/by-nc-sa/4.0/
[cc-by-nc-sa-shield]: https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg