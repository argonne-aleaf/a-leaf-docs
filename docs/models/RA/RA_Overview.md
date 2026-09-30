# RA Overview

The Resource Adequacy (RA) model evaluates how reliably a given power system serves load under generator outage uncertainty and renewable availability uncertainty. It samples outage and renewable scenarios, re-dispatches the system after each outage, and summarizes the unserved energy as reliability metrics (EUE, NEUE, LOLH, LOLE). It also computes capacity-credit values (ELCC and DLOL) for resources. Like Operation, it takes the fleet as given and does not decide what to build.

## When to use RA
Use RA to answer questions such as:

- Can the system maintain adequacy under forced outage conditions?
- How do renewable availability assumptions change reliability outcomes?
- How does reliability differ across buses, day groups, and the system as a whole?
- What is the capacity credit of a candidate resource in an ELCC-style study? ELCC (Effective Load Carrying Capability) is defined in [RA Metrics, ELCC, and DLOL](./RA_Metrics_ELCC_and_DLOL.md#elcc).

If the question is about routine dispatch and prices rather than rare outage events, use [Operation](../Operation/Operation_Overview.md). If it is about what to build, use [GTEP](../GTEP/GTEP_Overview.md).

## Run modes
RA supports the following uses:

- **Perfect-foresight post-contingency dispatch** (`RA_method = "Economic Dispatch"`).
- **Sequential post-contingency dispatch** (`RA_method = "Sequential Economic Dispatch"`).
- **ELCC studies** that reuse the same scenario and dispatch machinery.
- **DLOL reporting.** DLOL (Direct Loss of Load) is a dispatch-based capacity-credit proxy derived from RA outputs; see [RA Metrics, ELCC, and DLOL](./RA_Metrics_ELCC_and_DLOL.md#dlol).
- **Capacity-credit calculation.** Enable it with `calculate_capacity_credit_flag` and choose ELCC or DLOL with `capacity_credit_type`; see [RA Metrics, ELCC, and DLOL](./RA_Metrics_ELCC_and_DLOL.md).

The perfect-foresight and sequential modes share the same scenario and metric definitions. They differ in how redispatch is simulated once a scenario is selected (see [RA Execution and Results](./RA_Execution_and_Results.md#perfect-foresight-and-sequential-dispatch)).

## Inputs
| Input | Where it is set | Reference |
|---|---|---|
| RA method, capacity-credit options, sample sizes, screening, stress level, export flags | `RA Setting` sheet | [RA Settings and Scenarios Reference](./RA_Settings_and_Scenarios_Reference.md) |
| Renewable scenarios | `RA Scenarios` sheet | [RA Settings and Scenarios Reference](./RA_Settings_and_Scenarios_Reference.md) |
| Outage statistics and durations, optional fixed-risk inputs | `RA_data.xlsx` in the case data folder (for example `data/NorthAmerica/RA_data.xlsx`) and saved samples | [RA Scenarios and Data](./RA_Scenarios_and_Data.md) |
| Network, load, and time-series data | `data/` folder | [Network Data Reference](../../database/Network_Data_Reference.md) |
| Representative days | `Scenario Reduction Setting` sheet or a supplied day file | [Scenario Reduction and Representative-Day Groups](../../configuration/Scenario_Reduction_and_Repday_Groups.md) |
| Generation mix to test | Saved `GTEP_multi_round_info.json`, or an in-memory expansion result | [Expansion-record input contract](./RA_Execution_and_Results.md#expansion-record-input-contract) |

The representative-day structure and the resource data determine the simulation horizon and the assets exposed to outage and renewable uncertainty.

!!! note "RA supports nodal resolution"
    RA runs at bus (nodal) resolution as well as at aggregated regional resolution.

## How RA works
1. Build the RA-ready network and representative-day structure.
2. Parse renewable scenarios and load renewable time series.
3. Generate outage samples and combine them with renewable scenarios into joint scenarios.
4. Filter the joint scenarios to those that will be simulated.
5. Solve a reference dispatch for each retained renewable scenario and day group.
6. Solve a post-contingency dispatch for each retained joint scenario.
7. Aggregate annual and day-group reliability metrics and write the results.

### Representative days and renewable scenarios
RA runs on representative-day groups rather than the full 8,760-hour chronology. Annual renewable time series for each scenario are first mapped to the representative-day structure. This mapping matters in two places:

- Filtering, because renewable availability changes the generation margin used to identify critical scenarios.
- Simulation, because both the reference and the post-contingency dispatch must use the renewable scenario belonging to the joint scenario being solved.

Reliability is therefore evaluated over joint scenarios that combine outage uncertainty with renewable availability uncertainty.

## Outputs
RA writes the following, per planning stage, to the case output folder:

- a full JSON result (`<Case_ID>__RA_result_stage_<N>.json`) with settings, metrics, and capacity-credit results
- a metrics CSV (`<Case_ID>__RA_metrics_stage_<N>.csv`) with EUE, NEUE, LOLH, LOLE, and maximum-loss metrics at annual, day-group, regional, and systemwide levels
- optional dispatch and system CSV files for reference and post-contingency runs
- optional ELCC and DLOL results when those modes are enabled

See [RA Execution and Results](./RA_Execution_and_Results.md) for file names, columns, and how to check that a run is valid.

## How RA links to the other models
- **GTEP.** RA can test the fleet produced by a GTEP run. In multi-round GTEP runs, each round's ELCC or DLOL capacity credits feed the next round's expansion decisions; see [Relationship to RA and capacity credit](../GTEP/GTEP_Planning_Horizon_and_Multi_Round.md#relationship-to-ra-and-capacity-credit).
- **Operation.** RA uses the same fleet, network, and power-flow representation as Operation. Unlike Operation, RA has no operating reserves; only unserved energy (priced at VOLL) is penalized in its redispatch.

## Transmission representation and asynchronous DC ties
The RA dispatch models use the same power-flow representation as the other model families, selected by `power_flow_mode_flag`. In `B-theta` mode, branches flagged `dc_line = True` (asynchronous back-to-back HVDC and variable-frequency-transformer ties) are excluded from the DC angle equation and modeled as bounded controllable transfers, and the model fixes one reference (slack) bus per synchronous island. `Network_Flow` treats ties as transfer limits. `PTDF` does not support DC ties. See [GTEP Transmission Expansion](../GTEP/GTEP_Transmission_Expansion.md#asynchronous-dc-tie-modeling-b-theta) for the shared formulation.

## Transmission and distribution losses
RA applies the same flat, system-wide loss gross-up as GTEP and Operation. When `enforce_transmission_loss_flag` is `TRUE` in the `Planning Design` sheet, bus demand in the reference and post-contingency dispatch is multiplied by `1 + transmission_loss_percent_value * 0.01`. Losses are not flow-dependent; the gross-up is uniform across buses and hours. See the [Simulation Setting File Reference](../../configuration/ALEAF_Simulation_Setting_File.md#enforce_transmission_loss_flag).

## Recommended reading order
1. [RA Scenarios and Data](./RA_Scenarios_and_Data.md): outage samples, renewable scenarios, joint scenarios, and fixed-risk inputs
2. [RA Settings and Scenarios Reference](./RA_Settings_and_Scenarios_Reference.md): the `RA Setting` and `RA Scenarios` sheets
3. [RA Execution and Results](./RA_Execution_and_Results.md): reference and post-contingency dispatch, run modes, and exported results
4. [RA Metrics, ELCC, and DLOL](./RA_Metrics_ELCC_and_DLOL.md): reliability metrics, weighting, ELCC, and DLOL
5. [RA Formulation](./RA_Formulation.md): the equation-level reference, for readers who need the underlying mathematics

## Related documentation
- [A-LEAF Documentation](../../README.md)
- [Getting Started](../../Getting_Started.md)
- [GTEP Overview](../GTEP/GTEP_Overview.md)
- [Operation Overview](../Operation/Operation_Overview.md)
- [Simulation Setting File Reference](../../configuration/ALEAF_Simulation_Setting_File.md)
