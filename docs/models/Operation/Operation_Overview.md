# Operation Overview

The operation model dispatches a given power system, hour by hour, over representative days in each planning stage. It determines unit commitment, generation, storage cycling, reserve provision, power flows, and the resulting costs and prices. The model takes the system as given: it does not decide what to build or retire.

## What the operation model does
Depending on the case settings, the operation model includes:

- economic dispatch and unit commitment
- storage operation
- operating-reserve provision
- large-load and demand-response behavior
- hybrid plant operation
- policy tracking and market outputs (locational prices, reserve prices)

## When to use Operation
Use the operation model to answer questions such as:

- How does a fixed fleet, whether specified by hand or produced by a GTEP solve, dispatch hour by hour, including commitment, storage cycling, and reserve provision?
- What locational marginal prices (LMPs) and reserve clearing prices result from that dispatch?
- Does a candidate or expanded system meet demand and reserve requirements without re-optimizing investment and retirement?
- How do operating costs, curtailment, and emissions compare across scenarios that share one expansion plan?

If the question is what to build or retire, use [GTEP](../GTEP/GTEP_Overview.md). If it is about reliability under outages, use [RA](../RA/RA_Overview.md).

## Run modes
The operation model runs in one of three modes, set per case in the `Simulation Configuration` sheet:

- **Standalone:** dispatch the system exactly as defined in the input workbooks.
- **After expansion:** solve GTEP, then dispatch the expanded system in the same case.
- **Using predefined expansion data:** dispatch a system built from a saved expansion result.

Each mode can be solved serially or in parallel (`run_operation_in_parallel_flag`). In the parallel option, workers solve individual day groups and write per-day-group output files that are merged afterward. See [Operation Execution and Run Modes](./Operation_Execution_and_Run_Modes.md), including [Serial vs distributed operation solves](./Operation_Execution_and_Run_Modes.md#serial-vs-distributed-operation-solves).

## Inputs
| Input | Where it is set | Reference |
|---|---|---|
| Run mode (`Run_operation_flag`, `Run_expansion_flag`, `Use_predefined_expansion_data_for_OP_flag`, `predefined_expansion_data_file_name_for_OP`) | `Simulation Configuration` sheet | [Operation Execution and Run Modes](./Operation_Execution_and_Run_Modes.md) |
| Dispatch options, reserve flags, power-flow mode | `Simulation Configuration` and `Simulation Setting` sheets | [Simulation Configuration Reference](../../configuration/Simulation_Configuration_Reference.md) |
| Solver, parallel option, report flags | `Simulation Setting` sheet | [Simulation Setting File Reference](../../configuration/ALEAF_Simulation_Setting_File.md) |
| Fleet, storage, hybrid, and large-load data | `Gen Technology` sheet and `data/` folder | [Gen Technology Reference](../../database/Gen_Technology_Reference.md) |
| Network, load, and time-series data | `data/` folder | [Network Data Reference](../../database/Network_Data_Reference.md) |
| Representative days | `Scenario Reduction Setting` sheet or a supplied day file | [Scenario Reduction and Representative-Day Groups](../../configuration/Scenario_Reduction_and_Repday_Groups.md) |
| Saved expansion result (modes 2 and 3 only) | `data/<test_system_name>/expansion_results/` or the same run | [GTEP Expansion Outputs](../GTEP/GTEP_Expansion_Outputs.md) |

## Outputs
The operation model writes CSV reports (dispatch, market prices, policy, power flow, and system summaries) organized by planning stage, representative-day group, day, hour, and sub-period. An optional JSON snapshot is also available. Which reports are written is controlled by the `report_*` flags in the `Simulation Setting` sheet. See [Operation Outputs](./Operation_Outputs.md).

## How Operation links to the other models
- **GTEP.** GTEP solves an operation-style dispatch for representative days inside each planning stage. A standalone operation run uses the same dispatch representation. In the after-expansion and predefined modes, the operation model fixes the units in service, storage duration, and transmission build-out from the expansion result and does not re-optimize them.
- **RA.** The RA model starts from a reference dispatch of the same fleet and then re-dispatches after outages. RA does not use operating reserves, while Operation does.

## Operating reserves
The operation model schedules operating-reserve products (regulation up and down, spinning, flexibility up and down, and non-spinning) together with energy dispatch. Each product can be switched on or off per case in the `Simulation Configuration` sheet:

- `regulation_reserve_flag` (regulation up and down)
- `spinning_reserve_flag`
- `flexibility_reserve_flag` (flexibility up and down)
- `nonspin_reserve_flag`

When a flag is `FALSE`, the model omits that product's provision, zone requirements, and cost and scarcity terms, and the reports show `0` for it. A flag defaults to `TRUE` when its column is absent. See the [Simulation Configuration Reference](../../configuration/Simulation_Configuration_Reference.md#per-product-operating-reserve-flags).

!!! note "Operating reserves apply to GTEP and Operation only"
    Operating-reserve products and these flags apply to the expansion and operation models. The RA model has no operating reserves.

## Transmission representation and asynchronous DC ties
The operation model uses the same power-flow representation as the expansion model, selected by `power_flow_mode_flag`. In `B-theta` mode, branches flagged `dc_line = True` (asynchronous back-to-back HVDC and variable-frequency-transformer ties) are excluded from the DC angle equation and modeled as bounded controllable transfers, and the model fixes one reference (slack) bus per synchronous island. `Network_Flow` treats ties as transfer limits. `PTDF` does not support DC ties. See [GTEP Transmission Expansion](../GTEP/GTEP_Transmission_Expansion.md#asynchronous-dc-tie-modeling-b-theta) for the shared formulation.

## Transmission and distribution losses
The flat, system-wide loss gross-up used in GTEP applies identically. When `enforce_transmission_loss_flag` is `TRUE` in the `Planning Design` sheet, every bus's hourly demand is multiplied by `1 + transmission_loss_percent_value * 0.01` in the load balance, in all three power-flow modes. Losses are not flow-dependent: they do not vary with line loading, topology, or distance. The loss energy is reported in the `TnD_Loss` system-summary column (see [Operation Outputs](./Operation_Outputs.md#7-system-summary-outputs)). See the [Simulation Setting File Reference](../../configuration/ALEAF_Simulation_Setting_File.md#enforce_transmission_loss_flag) for both settings.

## Numerical treatment of near-zero renewable availability
In hours when a variable renewable unit's available output is effectively zero (below about 0.01 MW), the model sets that unit's generation and up-reserve provision to `0`. The forgone output is negligible, and the treatment improves numerical scaling. It is applied automatically and is not configurable.

## Detailed pages
1. [Operation Formulation](./Operation_Formulation.md): objective, variables, and constraints
2. [Operation Execution and Run Modes](./Operation_Execution_and_Run_Modes.md): run modes, serial and parallel solves, solver and memory considerations
3. [Operation Outputs](./Operation_Outputs.md): output files and columns

## Related documentation
- [A-LEAF Documentation](../../README.md)
- [Getting Started](../../Getting_Started.md)
- [Simulation Setting File Reference](../../configuration/ALEAF_Simulation_Setting_File.md)
- [Simulation Configuration Reference](../../configuration/Simulation_Configuration_Reference.md)
- [Scenario Reduction and Representative-Day Groups](../../configuration/Scenario_Reduction_and_Repday_Groups.md)
- [Policy and Financial Settings](../../configuration/Policy_and_Financial_Settings.md)
- [Storage Modeling Reference](../../database/Storage_Modeling_Reference.md)
- [Hybrid Resources Reference](../../database/Hybrid_Resources_Reference.md)
- [Large Load and Demand Response Reference](../../database/Large_Load_and_Demand_Response_Reference.md)
- [GTEP Overview](../GTEP/GTEP_Overview.md)
- [RA Overview](../RA/RA_Overview.md)
