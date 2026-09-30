# Getting Started

This page covers installing A-LEAF, running a first case, and finding the results. For what the settings and outputs mean, see the [Simulation Setting File Reference](./configuration/ALEAF_Simulation_Setting_File.md) and the model pages linked from the [Overview](./README.md).

!!! tip "New to the acronyms?"
    The [Glossary](./Glossary.md) gives short definitions of ELCC, DLOL, PRM, RPS, ATB, ESGC, and the other terms used on this site.

## Requirements
- macOS or Linux (Windows is untested)
- Julia `1.12` or newer (tested with `1.12.5`)
- The A-LEAF repository, which includes the bundled input data (about 200 MB in `data/`) and the `setting/` control workbook
- A solver: `HiGHS` is installed with the Julia dependencies and is the default. IBM ILOG CPLEX (licensed) and GPU solvers are optional; see [Solvers](#solvers)

## Installation
A-LEAF is a Julia package that is loaded with `using ALEAF`. The recommended setup is to clone the repository and use it as the active Julia project.

**1. Install Julia.** The simplest way is [juliaup](https://github.com/JuliaLang/juliaup):

```bash
curl -fsSL https://install.julialang.org | sh
juliaup add 1.12
juliaup default 1.12
```

**2. Clone the repository.**

```bash
git clone https://github.com/argonne-aleaf/a-leaf.git a-leaf
cd a-leaf
```

**3. Install the Julia dependencies.**

```bash
julia --project=. -e 'using Pkg; Pkg.instantiate()'
```

This installs the package versions recorded in `Manifest.toml`. The first installation downloads and precompiles about 130 packages and takes several minutes. It is needed only once, or after the dependencies change.

**4. Check the installation.**

```bash
julia --project=. -e 'using ALEAF; println("LOAD OK")'
```

### Alternative: install as a Julia package
```julia
using Pkg
Pkg.add(url="https://github.com/argonne-aleaf/a-leaf.git")
```

`using ALEAF` then loads the installed copy of the package. The bundled `setting/` and `data/` folders are not part of the installed package, so run from a directory that contains them (for example a clone of the repository).

## Running your first case
A run is controlled by one Excel workbook in `setting/`. The workbook determines which models run, which data they use, and which policy and technology assumptions apply. Its structure is described in the [Simulation Setting File Reference](./configuration/ALEAF_Simulation_Setting_File.md).

### 1. Find the setting workbook
Setting workbooks are the `.xlsx` files in the `setting/` folder of the repository. Pass the file name without the `.xlsx` extension. The public release bundles one workbook:

| Setting workbook (`setting/`) | Data (`data/`) | Notes |
|---|---|---|
| `ALEAF_Simulation_Setting_NorthAmerica` | `data/NorthAmerica/` | North America database with sub-state (`Country subdivision`) data. The bundled cases model the Texas interconnection at `BA` resolution; other regions and resolutions are selected in the configuration workbook. Technology costs come from `data/common/ATB_2024.csv`. |

The `test_system_name` entry in the `Simulation Setting` sheet selects the data folder under `data/`. To see which spatial resolutions the bundled data can be run at, see [Network Resolution](./database/Network_Resolution.md).

### 2. Run it
From the repository root:

```bash
julia --project=. execute_ALEAF.jl ALEAF_Simulation_Setting_NorthAmerica
```

The workbook can also be chosen in two other ways. In order of precedence:

1. the command-line argument, as above
2. the `ALEAF_SETTING` environment variable, for example `ALEAF_SETTING=ALEAF_Simulation_Setting_NorthAmerica julia --project=. execute_ALEAF.jl`
3. the `setting` line at the top of `execute_ALEAF.jl`

A-LEAF reads `setting/` and `data/` relative to the current directory, so always start Julia from the repository root (or another directory that contains both folders).

The same run can be started from a Julia session:

```julia
using Pkg
Pkg.activate(".")
using ALEAF

ALEAF.run_ALEAF(; master_setting_file_name = "ALEAF_Simulation_Setting_NorthAmerica")
```

`master_setting_file_name` is the only required argument.

### 3. Understand which cases run
A workbook does not define a single scenario. A-LEAF runs every **case** in the `Simulation Configuration` sheet whose `Run_Flag` is `TRUE`. Each case column sets which model families run (`Run_expansion_flag`, `Run_operation_flag`, `Run_RA_flag`) and which data and policy variant it uses. See the [Simulation Configuration Reference](./configuration/Simulation_Configuration_Reference.md).

The bundled workbook defines these six example cases, and only the first has `Run_Flag` enabled:

| Case_ID | Models |
|---|---|
| `Test_EXP` | Expansion only (`Run_Flag` enabled) |
| `Test_OP` | Operation only |
| `Test_RA` | Reliability assessment only |
| `Test_EXP_OP` | Expansion followed by operation |
| `Test_EXP_OP_RA` | Expansion, operation, and reliability assessment |
| `Test_EXP_RA` | Expansion followed by reliability assessment |

To run several cases in one session, set their `Run_Flag` to `TRUE`. To restrict a run to one of the enabled cases, set the `ALEAF_CASE_ID` environment variable to that case's column number (the case must have `Run_Flag` enabled).

!!! warning "Start small"
    Full-size national cases can take hours to solve. For a first run, keep `Run_Flag` enabled for one case only and choose a coarser network resolution (see [Network Resolution](./database/Network_Resolution.md)) before scaling up. Check the `Simulation Configuration` sheet to confirm which cases are enabled before starting.

## Where to find output
Results are written under the `output/` folder of the directory you ran from:

```text
output/<test_system_name>/case_id_<case_id>_<Case_ID>/
```

`test_system_name` comes from the `Simulation Setting` sheet, `<case_id>` is the case column number, and `<Case_ID>` is the case label in the `Simulation Configuration` sheet. For example, the first bundled case writes to `output/NorthAmerica/case_id_1_Test_EXP/`.

Each case folder contains the CSV report families (and optional JSON output) that the model pages describe:

- [GTEP Expansion Outputs](./models/GTEP/GTEP_Expansion_Outputs.md)
- [Operation Outputs](./models/Operation/Operation_Outputs.md)
- [RA Execution and Results](./models/RA/RA_Execution_and_Results.md)

Which reports are written is controlled by the `report_*` flags in the `Simulation Setting` sheet.

## Solvers
`HiGHS` is the default and needs no additional installation. The solver is chosen by `solver_name` in the `Simulation Setting` sheet, and its parameters are read from the matching solver sheet (for example `HiGHS Setting`).

### Optional: CPLEX
CPLEX is optional and is not needed to run A-LEAF. If you have an IBM ILOG CPLEX Studio license (supported versions 12.10, 20.1, and 22.1.x; 22.1.x recommended):

1. Set the environment variable `CPLEX_STUDIO_BINARIES` to the folder that contains the CPLEX binaries, then add the package. From a clone, run `julia --project=. -e 'using Pkg; Pkg.add("CPLEX")'`. This edits the clone's `Project.toml` and `Manifest.toml`; keep that change local. When A-LEAF is installed as a package, run `Pkg.add("CPLEX")` in your own environment instead.
2. Load the package. `execute_ALEAF.jl` loads CPLEX automatically when it is installed. In your own script or a Julia session, run `using CPLEX` before `ALEAF.run_ALEAF`.
3. Set `solver_name = CPLEX` in the `Simulation Setting` sheet. Solver parameters are read from the `CPLEX Setting` sheet.

If `solver_name = CPLEX` is selected without loading CPLEX, A-LEAF stops with an error that says so.

### Optional: GPU solvers
`cuOpt` (LP only) and `MadNLP` can be selected with `solver_name`. They are optional packages that are not part of the default environment. See [GPU Solvers](./configuration/GPU_Solvers.md).

## Environment variables
| Variable | Purpose |
|---|---|
| `ALEAF_SETTING` | Setting workbook name (no `.xlsx`) used by `execute_ALEAF.jl` when no command-line argument is given. |
| `ALEAF_CASE_ID` | Restricts the run to this case number, which must have `Run_Flag` enabled (one case per Julia process). |
| `CPLEX_STUDIO_BINARIES` | Path to the CPLEX binaries; needed only when building the optional CPLEX package. |

## Troubleshooting
| Symptom | Fix |
|---|---|
| `setting/<name>.xlsx not found` | Start Julia from the repository root. A-LEAF reads `setting/` and `data/` relative to the current directory. |
| Very slow first run | The first `using ALEAF` precompiles the project. Later runs start quickly. |
| Error that CPLEX is not loaded | Load CPLEX with `using CPLEX`, or set `solver_name = HiGHS`. See [Optional: CPLEX](#optional-cplex). |

## Next steps
- Read the [Overview](./README.md) to see how the three model families (GTEP, Operation, RA) relate.
- Read the [Simulation Setting File Reference](./configuration/ALEAF_Simulation_Setting_File.md) and the [Simulation Configuration Reference](./configuration/Simulation_Configuration_Reference.md) to adapt a bundled case.
- Use the [Glossary](./Glossary.md) for the recurring acronyms and notation.
