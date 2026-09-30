# Operation Execution and Run Modes

This page explains how to run the operation (production-cost) model in A-LEAF: which run modes exist, how to choose among them, how the serial and parallel solves differ, and what to expect in terms of solver choice, run time, and memory. Read it before setting up an operation study, especially a large or multi-stage one.

## What the operation model does

The operation model evaluates dispatch over the reduced representative-day chronology for each planning stage. It can be used in three ways:

- as a standalone operation run that does not depend on prior expansion results,
- as a follow-on to an expansion run in the same case, or
- as an operation run that uses expansion results loaded from a saved file.

In every case, results are organized by planning stage, representative-day group, and the day, hour, and sub-period indices inside the reduced chronology. The operation model does not perform a single continuous one-year dispatch. It evaluates each stage separately, one representative-day group at a time.

## Choosing a run mode

Three flags on the `Simulation Configuration` sheet determine the run mode for each case. The operation model runs only when `Run_operation_flag` is `TRUE`. Given that, the mode follows from the other two flags:

| `Run_expansion_flag` | `Use_predefined_expansion_data_for_OP_flag` | Mode |
|---|---|---|
| `TRUE` | (not used) | 2. Operation after expansion |
| `FALSE` | `TRUE` | 3. Operation using predefined expansion data |
| `FALSE` | `FALSE` | 1. Standalone operation run |

| If your study needs to... | Use |
|---|---|
| Evaluate an existing system as specified in the input workbooks, with no new builds or retirements | Mode 1 |
| Optimize the long-term build plan and then test it operationally in one run | Mode 2 |
| Re-run or vary operations (dispatch mode, solver, day groups, reserves) on a fixed build plan produced earlier | Mode 3 |

### 1. Standalone operation run

The model builds operation network data directly from the workbook inputs. No investment or retirement decisions are taken from an earlier solution, and the model evaluates dispatch on the configured planning stages and day groups. Use this mode to study the system as defined in the input data.

### 2. Operation after expansion

The model solves the expansion problem and then reuses the solved investment, retirement, and transmission build-out decisions when building the operation problem. Total renewable capacity by bus (`Wind_TotalMW`, `PV_TotalMW`, and `RTPV_TotalMW`, covering onshore wind and utility PV plus rooftop PV, but not CSP) is also carried from each stage of the expansion result into the operation network data.

This is the most common coupled workflow, because it yields both the long-term expansion decisions and operational results under the chosen plan.

### 3. Operation using predefined expansion data

The model reads a saved expansion result instead of solving expansion in the current run. Set:

- `Use_predefined_expansion_data_for_OP_flag` to `TRUE`, and
- `predefined_expansion_data_file_name_for_OP` to the saved file name on the `Simulation Configuration` sheet.

The file is read from `data/<test_system_name>/expansion_results/`. The operation model uses the expansion decisions and the recorded renewable investment totals of the first round in the file, so the file must be a single-round expansion record and not a multi-round history. The operation model is built on that fixed expansion state.

This mode is convenient for sensitivity studies: run expansion once, then repeat the operation model with different dispatch settings.

## What is fixed from the expansion result

When expansion results are available (modes 2 and 3), the operation model does not re-optimize long-term decisions. It fixes:

- the number of units in service in each stage, which already nets the expansion run's new builds and retirements, and
- the storage energy-duration quantity for storage technologies.

It also reads the expansion result's transmission build-out to set branch flow limits. Expansion-only controls are switched off in the operation solve: `multi_round_solution_process_flag` (`Planning Design` sheet) and `transmission_expansion_flag` (`Simulation Configuration` sheet) are treated as `FALSE`. The operation run therefore uses the expanded network and asset state without performing a new transmission expansion optimization.

## Execution flow

For each operation run, A-LEAF performs these steps:

1. Establish the expansion state to use (none, the just-solved expansion, or the saved file).
2. Generate operation network data for each planning stage.
3. Build the operation model for each representative-day group.
4. Solve the day-group problems.
5. Recover dual values (prices), which requires an additional pass for non-LP models.
6. Write the JSON and CSV reports described in [Operation Outputs](./Operation_Outputs.md).

### Planning stages

The operation model uses the same planning stages as the expansion model. For each stage, A-LEAF generates stage-specific network data, then solves every configured day group for that stage. Results are stored by stage and day group.

### Representative-day groups

The number of operation day groups is set by `NDAY_Groups_OP` on the `Simulation Configuration` sheet, and `NDAYS_in_Single_Group_OP` sets the number of days per group. Each day group maps to a block of representative days. Using more groups or days improves the chronological coverage of the year and increases run time and memory roughly in proportion. See [Scenario Reduction and Representative-Day Groups](../../configuration/Scenario_Reduction_and_Repday_Groups.md) for how days are selected and weighted.

## Dispatch model type and duals

The operation dispatch style is chosen with `Dispatch_Mode_in_OP` (`Economic Dispatch` or `Unit Commitment`).

- Linear (LP) operation runs, with no integer or binary variables, produce prices directly from the solved problem.
- Runs with integer variables (for example, unit commitment) are solved first with integers, and then a second problem with the integer variables fixed is solved to recover dual values. This adds solve time.

The price columns in the market and power-flow outputs (`LMP_USD_per_MWh` and the reserve prices) are dual values. See [Operation Outputs](./Operation_Outputs.md#2-market-output) for how to read them.

## Solver choice

The solver is selected with `solver_name` on the `Simulation Setting` sheet:

| `solver_name` | Notes for operation runs |
|---|---|
| `HiGHS` | Default and open source. Suitable for small and medium systems. |
| `CPLEX` | Optional and licensed. It must be installed and loaded; see [Getting Started](../../Getting_Started.md#optional-cplex). Generally faster on large or unit-commitment problems. |
| `cuOpt` | GPU solver for pure LP problems. Prices are approximate first-order duals. See [GPU Solvers](../../configuration/GPU_Solvers.md). |
| `MadNLP` | Accepted when its package is loaded. |

For studies where prices matter, validate results from an approximate solver such as `cuOpt` against a simplex or barrier solve on a smaller case.

## Serial vs distributed operation solves

`run_operation_in_parallel_flag` on the `Simulation Setting` sheet selects between two solve strategies. It applies to all three run modes.

### Serial operation solve

With the flag `FALSE`, one process builds the network, solves the day groups one after another within each stage, and writes all reports. Use it for small and medium systems, for debugging, and whenever a single machine has enough memory for the full model.

### Distributed operation solve

With the flag `TRUE`, day-group problems within each stage are distributed across available worker processes. Worker processes must be available to the Julia session before the run starts (for example, started with `julia -p <N>` or added with `Distributed.addprocs`). Use it when the number of day groups or the model size makes a serial solve too slow or too large for one process's memory.

Key characteristics:

- **Per-worker day-group solves.** Each planning stage is still processed in turn. Within a stage, each worker builds and solves only its assigned day group, a model roughly `1/NDAY_Groups_OP` of the full-horizon size. Workers use the same representative-day assignment as the main process, so their results line up with the reference.
- **Small main-process footprint.** The main process keeps only aggregates: objective values per day group, summed annual generation information, summed scarcity totals, and the build decisions (which do not vary by day group). It does not hold per-hour solutions.
- **Different intermediate file layout.** Each worker writes its own dispatch, market, policy, demand-response, and power-flow rows to per-day-group part files with a `__y<year>_dg<id>` suffix. When the run completes, these are merged automatically into the same single CSV files a serial run writes, and the part files are deleted. If a run is interrupted before the merge, the leftover `__y*_dg*.csv` files contain the raw per-day-group results.
- **Fault tolerance.** Day groups are handed to whichever worker is free. If a worker dies (for example, because it ran out of memory), its day group is reassigned to another live worker, up to three attempts. If any day group cannot be completed, the run stops with an error and does not return partial results.
- **Multi-stage support.** The distributed solve works with all three run modes on multi-stage horizons. Stages without recorded renewable investment totals (for example, in standalone runs) are built without fixed renewable totals, as in the serial solve.

!!! note "Choose the number of workers with memory in mind"
    Each worker holds a model for one day group. More workers reduce wall-clock time but multiply the total memory in use. If workers are terminated by the operating system, reduce the worker count or increase the number of day groups so that each subproblem is smaller.

### Light master reference (nodal / out-of-memory behavior)

The main process builds a reporting-only reference that it never solves. To keep its footprint small at nodal scale, A-LEAF builds that reference in a light mode whenever neither `export_model_reference_json_operation_flag` nor `export_model_reference_json_expansion_flag` is `TRUE`. In the light mode, the main process does not build the hourly representative-day data or the solve-only aggregations, does not create solver models it will never use, and shares one copy of the generator technology table across all sub-area networks instead of copying it for each bus. The serial solve applies the same discipline by discarding the hourly data and raw time series from the stored network data once the reference is built.

!!! note "Turning on a model-reference JSON export disables the light reference"
    Setting either `export_model_reference_json_operation_flag` or `export_model_reference_json_expansion_flag` to `TRUE` keeps the full reference so it can be written to JSON. This is appropriate for small systems but restores the large memory footprint on nodal systems. Enable it only when the JSON dump is needed. See the [Simulation Setting File Reference](../../configuration/ALEAF_Simulation_Setting_File.md#export_model_reference_json_operation_flag).

## Time and memory expectations

Run time and memory scale with the following factors:

- **Network size.** Nodal and bus-level networks are much larger than zonal or reduced ones. Transmission representation (`Network_Flow`, B-theta, or PTDF-based) also affects model size.
- **Number of stages and day groups.** Each stage and day group is a separate solve. Doubling either doubles the number of solves.
- **Dispatch mode.** `Unit Commitment` adds integer variables and a second solve for duals, and is substantially slower than `Economic Dispatch`.
- **Solver.** Commercial and GPU solvers typically solve large LPs faster than HiGHS.
- **Reports.** Writing hourly reports for large networks adds time and disk use. The report control flags in [Operation Outputs](./Operation_Outputs.md#report-control-flags) allow unneeded families to be suppressed.

Each run writes `<Case_ID>__simulation_run_time.csv` with `Expansion_Run_Time_s`, `Operation_Run_Time_s`, and `RA_Run_Time_s` for the case. Use it to compare the time cost of different settings.

A practical approach is to start with the serial solve on a small number of day groups, confirm results and run time, and then scale up. Move to the distributed solve when the serial run no longer fits in memory or takes too long.

## Checklist for reviewing an operation run

1. Confirm the run mode (standalone, after expansion, or predefined expansion data).
2. Confirm whether the run was serial or distributed.
3. Confirm the number of stages and operation day groups.
4. Confirm that expansion decisions were fixed when expected (modes 2 and 3).
5. Confirm the solver and dispatch mode.

## Relationship to expansion outputs

Operation reports use the same dispatch, market, technology-summary, system-summary, and representative-day structures as the expansion reports, so the two are directly comparable. The difference is that operation outputs reflect fixed expansion decisions and operation-only outcomes, not the integrated expansion objective.

## Related documentation

- [Operation Overview](./Operation_Overview.md)
- [Operation Formulation](./Operation_Formulation.md)
- [Operation Outputs](./Operation_Outputs.md)
- [GTEP Overview](../GTEP/GTEP_Overview.md)
- [GTEP Expansion Outputs](../GTEP/GTEP_Expansion_Outputs.md)
- [Scenario Reduction and Representative-Day Groups](../../configuration/Scenario_Reduction_and_Repday_Groups.md)
