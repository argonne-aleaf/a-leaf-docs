# GTEP Overview

The Generation and Transmission Expansion Planning (GTEP) model finds the least-cost build-out and retirement plan for a power system over a multi-decade horizon. It decides what to build, where, and when, and what to retire, while representing the hourly operation of the system inside each planning stage. GTEP is the model to use when the study question is about investment.

## What GTEP does
A GTEP case solves a long-term planning problem that can include:

- generation expansion
- generator retirement
- storage capacity and duration expansion
- transmission expansion
- representative-day operational dispatch inside each planning stage
- policy, reserve, scarcity, and financial terms in the planning objective

Depending on the case settings, a GTEP case can also trigger an [Operation](../Operation/Operation_Overview.md) run or an [RA](../RA/RA_Overview.md) run on the expanded system.

## When to use GTEP
Use GTEP to answer questions such as:

- What generation and transmission should be built, and when, to meet demand and policy targets at least cost?
- Which existing generators are economic to keep online, and which should retire at each planning stage?
- How sensitive is the optimal build-out to fuel prices, technology costs, policy design, or reliability requirements? Policy levers include renewable and clean-energy targets (RPS, CEGT), carbon pricing, and the investment and production tax credits (ITC, PTC); see [Policy and Financial Settings](../../configuration/Policy_and_Financial_Settings.md).
- What does the resulting system look like operationally, before it is handed to the [Operation](../Operation/Operation_Overview.md) model for detailed dispatch or to the [RA](../RA/RA_Overview.md) model for a reliability check?

If the fleet is already fixed and only its dispatch or reliability is of interest, use the Operation or RA model directly.

## How a GTEP study is organized
A typical case spans several multi-year planning stages (see [GTEP Planning Horizon and Multi-Round](./GTEP_Planning_Horizon_and_Multi_Round.md)). The operation of each stage is represented by a small set of representative days rather than all 8,760 hours, which keeps the combined investment and dispatch problem tractable (see [Scenario Reduction and Representative-Day Groups](../../configuration/Scenario_Reduction_and_Repday_Groups.md)). Very large studies can be split into overlapping subproblems with the multi-round option.

For the equations and their economic interpretation, see [GTEP Formulation](./GTEP_Formulation.md).

## Inputs
| Input | Where it is set | Reference |
|---|---|---|
| Planning horizon (base year, first stage year, number and length of stages, dollar year) | `Planning Design` sheet | [Planning Horizon and Multi-Round](./GTEP_Planning_Horizon_and_Multi_Round.md) |
| Case definition, model switches, reserve flags, transmission expansion options | `Simulation Configuration` sheet | [Simulation Configuration Reference](../../configuration/Simulation_Configuration_Reference.md) |
| Solver, power-flow mode, report flags | `Simulation Setting` sheet | [Simulation Setting File Reference](../../configuration/ALEAF_Simulation_Setting_File.md) |
| Technology costs and parameters, candidate resources | `Gen Technology`, `ATB Setting`, and storage cost sheets | [Gen Technology Reference](../../database/Gen_Technology_Reference.md) |
| Network, load, and time-series data | `data/` folder | [Network Data Reference](../../database/Network_Data_Reference.md) |
| Representative days | `Scenario Reduction Setting` sheet or a supplied day file | [Scenario Reduction and Representative-Day Groups](../../configuration/Scenario_Reduction_and_Repday_Groups.md) |
| Policy targets and tax credits | `Planning Design`, `ITC`, `PTC` sheets | [Policy and Financial Settings](../../configuration/Policy_and_Financial_Settings.md) |

## Outputs
GTEP writes CSV reports for build and retirement decisions (generation and transmission), per-stage and annual technology and system summaries, dispatch, power flow, and scarcity (energy not served and reserve shortfall). An optional JSON summary of multi-round expansion (`GTEP_multi_round_info.json`) can be exported and reused by later runs. Which reports are written is controlled by the `report_*` flags in the `Simulation Setting` sheet. See [GTEP Expansion Outputs](./GTEP_Expansion_Outputs.md).

## How GTEP links to the other models
- **Operation.** The expanded fleet can be dispatched in a full operation run, either in the same case or in a later run that reuses the saved expansion results. See [Operation Execution and Run Modes](../Operation/Operation_Execution_and_Run_Modes.md).
- **RA.** The expanded fleet can be tested for reliability. In multi-round runs, the capacity credits from each round feed the next round's expansion decisions; see [Relationship to RA and capacity credit](./GTEP_Planning_Horizon_and_Multi_Round.md#relationship-to-ra-and-capacity-credit).

## Operating reserves
The model schedules operating-reserve products (regulation up and down, spinning, flexibility up and down, and non-spinning) together with energy dispatch. Each product can be switched on or off per case in the `Simulation Configuration` sheet:

- `regulation_reserve_flag` (regulation up and down)
- `spinning_reserve_flag`
- `flexibility_reserve_flag` (flexibility up and down)
- `nonspin_reserve_flag`

When a flag is `FALSE`, the model omits that product's provision, zone requirements, and cost and scarcity terms, and the reports show `0` for it. A flag defaults to `TRUE` when its column is absent. See the [Simulation Configuration Reference](../../configuration/Simulation_Configuration_Reference.md#per-product-operating-reserve-flags).

!!! note "Operating reserves apply to GTEP and Operation only"
    Operating-reserve products and these flags apply to the expansion and operation models. The RA model has no operating reserves.

## Transmission and distribution losses
When `enforce_transmission_loss_flag` is `TRUE` in the `Planning Design` sheet, the model multiplies every bus's hourly demand by `1 + transmission_loss_percent_value * 0.01` in the load balance. Generation, storage discharge, and imports must cover delivered load plus this loss adder. The uplift applies in all three power-flow modes (`PTDF`, `B-theta`, `Network_Flow`). When the flag is `FALSE`, the model is lossless.

!!! warning "Flat gross-up, not flow-dependent losses"
    This is a flat, system-wide demand gross-up, not a per-line I²R loss model. Losses do not vary with line loading, network topology, or distance: the model serves `demand × (1 + pct/100)` at every bus and hour.

The loss energy appears in the `TnD_Loss` column of the system-summary output (see [GTEP Expansion Outputs](./GTEP_Expansion_Outputs.md#11-system-summary-output)). The same gross-up applies in the Operation and RA models. See the [Simulation Setting File Reference](../../configuration/ALEAF_Simulation_Setting_File.md#enforce_transmission_loss_flag) for both settings.

## Numerical treatment of near-zero renewable availability
In hours when a variable renewable unit's available output is effectively zero (below about 0.01 MW), the model sets that unit's generation and up-reserve provision to `0`. The forgone output is negligible, and the treatment improves numerical scaling. It is applied automatically and is not configurable.

## Detailed pages
1. [GTEP Formulation](./GTEP_Formulation.md): objective, variables, and constraints
2. [GTEP Planning Horizon and Multi-Round](./GTEP_Planning_Horizon_and_Multi_Round.md): stages, horizon, and rolling-horizon solves
3. [GTEP Transmission Expansion](./GTEP_Transmission_Expansion.md): transmission investment and DC-tie modeling
4. [GTEP Expansion Outputs](./GTEP_Expansion_Outputs.md): output files and columns

## Related documentation
- [A-LEAF Documentation](../../README.md)
- [Getting Started](../../Getting_Started.md)
- [Simulation Setting File Reference](../../configuration/ALEAF_Simulation_Setting_File.md)
- [Simulation Configuration Reference](../../configuration/Simulation_Configuration_Reference.md)
- [Scenario Reduction and Representative-Day Groups](../../configuration/Scenario_Reduction_and_Repday_Groups.md)
- [Policy and Financial Settings](../../configuration/Policy_and_Financial_Settings.md)
- [Operation Overview](../Operation/Operation_Overview.md)
- [RA Overview](../RA/RA_Overview.md)
