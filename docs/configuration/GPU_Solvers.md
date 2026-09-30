# GPU Solvers (cuOpt and MadNLP)

A-LEAF solves its optimization models with HiGHS by default. Two optional GPU solvers are available for large models:

- **cuOpt**: NVIDIA's PDLP solver, a first-order method for linear programs (LP only, no integer variables).
- **MadNLP**: a GPU interior-point solver, used through ExaModels.

Both are optional. A standard installation runs entirely on the CPU, and a GPU solver is active only when its Julia packages are loaded together with `using ALEAF`. GPU solvers require Julia 1.9 or later and an NVIDIA GPU. CPLEX is a separate optional CPU solver; see [Getting Started](../Getting_Started.md#optional-cplex).

## Selecting the solver

Set `solver_name` in the `Simulation Setting` sheet (cell `B11`) of the setting workbook to `HiGHS` (default), `CPLEX`, `cuOpt` or `MadNLP`. Solver parameters are read from the sheet with the matching name: `HiGHS Setting`, `CPLEX Setting`, `cuOpt Setting` or `MadNLP Setting`. The `solver_direct_mode_flag` row on each of these sheets must stay `FALSE` for HiGHS, cuOpt and MadNLP. See the [Simulation Setting File Reference](./ALEAF_Simulation_Setting_File.md#11-solver-sheets).

## Default solver (HiGHS)

No extra packages are needed:

```
julia --project=. execute_ALEAF.jl ALEAF_Simulation_Setting_NorthAmerica
```

## cuOpt

cuOpt solves linear programs only. The expansion model contains integer build decisions by default, so to run it on cuOpt, set `Integrality = FALSE` for the integer-build technologies in the `Gen Technology` sheet. The operation model is already an LP.

### Setup

Requirements are an NVIDIA GPU and the cuOpt runtime libraries (`libcuopt`) that the `cuOpt` Julia package uses. `cuOpt` is an optional dependency, compatible with version `0.2.1`, and is not part of the default environment.

1. Create an environment that contains A-LEAF and `cuOpt`. From the repository root (`env-cuopt` is an example directory name):
   ```
   julia --project=env-cuopt -e 'using Pkg; Pkg.develop(path="."); Pkg.add("cuOpt")'
   ```
   Make the cuOpt runtime libraries visible to the dynamic loader as described in the `cuOpt` package documentation, for example through `LD_LIBRARY_PATH`.
2. In the setting workbook, set `solver_name = cuOpt` and relax the integer build technologies (`Integrality = FALSE` in `Gen Technology`).
3. Load `cuOpt` before `using ALEAF` and run:
   ```
   julia --project=env-cuopt -e 'using cuOpt; using ALEAF; ALEAF.run_ALEAF(master_setting_file_name="ALEAF_Simulation_Setting_NorthAmerica")'
   ```
   `execute_ALEAF.jl` does not load `cuOpt` itself, so use this command or add `using cuOpt` above `using ALEAF` in your own copy of the script.

### Running one case per process

GPU memory used by cuOpt is returned to the operating system only when the Julia process exits. When a workbook enables many cases, GPU memory can be exhausted in a single process. Setting the environment variable `ALEAF_CASE_ID` to a case's column number in the `Simulation Configuration` sheet restricts the run to that case (the case must have `Run_Flag` enabled). Launch one process per case. When the variable is unset, all enabled cases run in one process.

### `cuOpt Setting` parameters

The bundled sheet uses the PDLP algorithm (`method = 1`), `crossover = FALSE`, and `1e-4` for the absolute and relative primal, dual and gap tolerances (`absolute_primal_tolerance`, `relative_primal_tolerance`, `absolute_dual_tolerance`, `relative_dual_tolerance`, `absolute_gap_tolerance`, `relative_gap_tolerance`).

With `crossover = FALSE`, cuOpt returns an interior (non-vertex) solution, so dual values and prices are approximate. Set `crossover = TRUE` if accurate prices are needed; the solve is slower.

### Prices (LMP and reserve clearing prices)

With cuOpt, A-LEAF writes the `LMP` column and the reserve clearing price columns (`RCP_RU`, `RCP_RD`, `RCP_Spin`, `RCP_NSpin`, `RCP_FU`, `RCP_FD`) to `*__market_EXP.csv` and `*__market_OP.csv`. Keep these caveats in mind:

- **Approximate values.** The duals come from a first-order method at the default `1e-4` tolerance, so prices are approximate compared with a simplex solve. For price-sensitive work, compare against a HiGHS run, and tighten the dual and gap tolerances in `cuOpt Setting`.
- **LP only.** Duals exist only for pure LP models. If any `Integrality` flag is set, cuOpt returns no duals. Relax the integer build decisions for any run where prices matter.
- **Version dependence.** The dual-value support is tied to `cuOpt` version `0.2.1`. After upgrading `cuOpt`, verify prices against a HiGHS run; A-LEAF logs a warning at load time if it detects a change in the `cuOpt` internals it relies on.

## MadNLP

MadNLP requires an NVIDIA GPU and the MadNLP/CUDA package stack. It activates when `ExaModels`, `MadNLP`, `MadNLPGPU`, `CUDA` and `CUDSS` are all loaded.

### Setup

Create an environment once (from the repository root; `env-madnlp` is an example name):

```
julia --project=env-madnlp -e 'using Pkg; Pkg.develop(path="."); Pkg.add(["ExaModels","MadNLP","MadNLPGPU","CUDA","CUDSS"])'
```

Set `solver_name = MadNLP` in the workbook, then load the GPU stack before A-LEAF and unset `LD_LIBRARY_PATH` (see [Using cuOpt and MadNLP together](#cuopt-and-madnlp-cannot-share-a-julia-session)):

```
unset LD_LIBRARY_PATH
julia --project=env-madnlp -e 'using MadNLP, MadNLPGPU, CUDA, CUDSS, ExaModels; using ALEAF; ALEAF.run_ALEAF(master_setting_file_name="ALEAF_Simulation_Setting_NorthAmerica")'
```

### `MadNLP Setting` parameters

Set a row's `Flag` to `TRUE` to apply its `Value`; otherwise the solver default is used. The sheet exposes two parameters.

| Parameter | Bundled value | Meaning |
|---|---|---|
| `kkt_system` | `SparseCondensed` | Form of the KKT linear system. Allowed values: `SparseCondensed` (GPU default, used with cuDSS), `DenseCondensed`, `Sparse`, `ScaledSparse`, `SparseUnreduced`, `Dense`. In GPU testing only `SparseCondensed` and `ScaledSparse` converged; the other forms may fail. |
| `tol` | `1e-06` | Interior-point convergence tolerance. |

## cuOpt and MadNLP cannot share a Julia session

cuOpt ships its own CUDA libraries. When they are on `LD_LIBRARY_PATH`, they conflict with the libraries that `CUDA.jl` and `cuDSS` use for MadNLP and cause illegal-memory errors. Therefore:

- Use separate environments for cuOpt and MadNLP, and never load both in one session.
- For cuOpt, set `LD_LIBRARY_PATH` to the cuOpt libraries.
- For MadNLP, unset `LD_LIBRARY_PATH` so `CUDA.jl` uses its own libraries.

## Notes and limitations

- The default environment is CPU-only. Installing A-LEAF does not download the CUDA stack; add the GPU solver packages as shown above.
- A single solve uses one GPU. Requesting more GPUs does not speed up or enlarge one A-LEAF solve.
- Do not run several cases concurrently on GPUs, and do not enable `run_GTEP_in_parallel_flag` with a GPU solver. Concurrent expansion runs that share one `data/<test_system_name>/` folder write scenario-reduction files to the same location and can overwrite each other.
