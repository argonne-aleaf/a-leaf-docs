# RA Execution and Results

This page explains how to run the Reliability Assessment (RA) model, what happens during a run, which files it writes, and how to check and interpret the results. Settings are described in [RA Settings and Scenarios Reference](./RA_Settings_and_Scenarios_Reference.md), scenarios in [RA Scenarios and Data](./RA_Scenarios_and_Data.md), and metric definitions in [RA Metrics, ELCC, and DLOL](./RA_Metrics_ELCC_and_DLOL.md).

## Running RA
RA runs as a model family of a case in the `Simulation Configuration` sheet, like expansion and operation.

1. In the `Simulation Configuration` sheet, set `Run_RA_flag` to `TRUE` for the case and `Run_Flag` to `TRUE`. Bundled examples include `Test_RA` (RA only) and `Test_EXP_OP_RA` (expansion, operation, and RA).
2. Review the `RA Setting` and `RA Scenarios` sheets. At a minimum, check `num_risk_scenario`, `RA_method`, `system_peak_scale`, and the enabled renewable scenarios.
3. Start A-LEAF as described in [Getting Started](../../Getting_Started.md).

### Which generation mix RA studies
RA evaluates the generation fleet produced by expansion planning:

- If the same case runs expansion first, RA uses that result. In multi-round expansion, RA runs once per round and `round_id_to_run_RA` selects the round when a single round is needed.
- If `Use_predefined_expansion_data_for_RA_flag` is on, RA reads the saved expansion result named in `predefined_expansion_data_file_name_for_RA` instead. See [Simulation Configuration Reference](../../configuration/Simulation_Configuration_Reference.md).

### Expansion-record input contract
RA accepts a saved expansion result in either of two forms:

- the multi-round record written by a multi-round expansion run (`GTEP_multi_round_info`, keyed by round), or
- the single-round expansion result record written by a standalone expansion solve. It is recognized by containing both an `expansion model result` and an `expansion model system reference` entry, and is converted to the multi-round form automatically.

A single-round record can therefore drive an RA run directly, without being reshaped first. When no external record is supplied, RA reads the saved `GTEP_multi_round_info.json` from the case output folder.

## What RA does in a run
1. Build the network and representative-day structure for the selected fleet and year.
2. Sample outages for every day group and combine them with the enabled renewable scenarios into joint scenarios.
3. Screen the joint scenarios if `risk_filtering_flag` is `true`.
4. Solve a **reference dispatch** for each retained day group and renewable scenario.
5. Solve a **post-contingency dispatch** for each retained joint scenario, and aggregate unserved energy into reliability metrics.
6. If `calculate_capacity_credit_flag` is `true`, compute ELCC or DLOL.
7. Write the result files.

### Reference dispatch
The reference dispatch is the no-outage economic dispatch for a day group under one renewable scenario. It defines the operating state, including commitment exposure, storage position, and congestion, before outages occur. Because renewable availability changes that state, a separate reference dispatch exists for each renewable scenario retained for the day group.

### Post-contingency dispatch
For each retained joint scenario, RA applies the outage sample and redispatches the system under the matching renewable scenario. It answers a conditional question: given this outage state and this renewable availability, how much load cannot be served after redispatch?

### Perfect foresight and sequential dispatch
`RA_method` selects how redispatch is represented once a scenario is chosen. The scenario definition and reference dispatch are identical in both modes.

| Mode | Behavior |
|---|---|
| `Economic Dispatch` (perfect foresight) | One optimization per day group. Hours before the first outage follow the reference solution. From the first outage hour onward, the model re-optimizes all remaining hours together, so storage and hydro can plan around the whole outage. |
| `Sequential Economic Dispatch` | A rolling-horizon dispatch that advances hour by hour. Each outage hour solves a look-ahead of `sequential_horizon_hours_value` hours (default 6), commits only the first hour, and carries the resulting storage state forward. Later look-ahead hours use a projected outage state, not the true one. |

Sequential dispatch approximates an operator who does not know how long an outage will last, so it may commit storage too early or too late. Because it has less information, its unserved energy is generally equal to or larger than the perfect-foresight result for the same scenarios, and EUE and LOLH from the two modes are comparable only with that in mind. It solves one optimization per outage hour rather than one per scenario, so it usually takes longer. See [RA Formulation](./RA_Formulation.md#stage-2-risk-realization-post-contingency-dispatch) for the equations.

## What RA writes
Files are written to the case output folder. `<Case_ID>` is the case label from the `Simulation Configuration` sheet and `N` is the planning stage. The stage number is in every file name, so multi-round runs do not overwrite earlier stages.

| File | Written | Content |
|---|---|---|
| `<Case_ID>__RA_result_stage_<N>.json` | Always | Full result for stage `N`, including settings, metrics, capacity-credit results when calculated, and the `stage` and calendar `year`. The amount of scenario-level detail depends on `output_verbose_level`. |
| `<Case_ID>__RA_metrics_stage_<N>.csv` | Always | The metrics in long format (see below). |
| `<Case_ID>__Reference_RA_stage_<N>_day_id_<day_group>_scenario_<renewable_scenario_id>_dispatch.csv` and `_system.csv` | `export_reference_dispatch_results_flag` is `true` | One pair per retained (day group, renewable scenario). |
| `<Case_ID>__RA_stage_<N>_day_id_<day_group>_risk_id_<joint_id>_dispatch.csv` and `_system.csv` | `export_baseline_dispatch_results_flag` is `true` and the scenario's peak unserved energy exceeds `export_dispatch_results_threshold_value` | Post-contingency results of a simulated joint scenario. |
| `<Case_ID>__RA_stage_<N>_ELCC_iteration_<iter>_day_id_<day_group>_risk_id_<joint_id>_dispatch.csv` and `_system.csv` | `export_ELCC_dispatch_results_flag` is `true`, threshold as above | Post-contingency results of scenarios inside ELCC iterations. Written to a per-resource subfolder such as `ELCC_of_<UNIT_REPORT_LABEL_2>_at_bus_<location>` (or `ELCC_existing_asset_<plant>_<bus>` for asset deactivation). |

### Metrics CSV
`<Case_ID>__RA_metrics_stage_<N>.csv` has the columns `Scenario`, `Stage`, `Year`, `Metric`, `Unit`, `Scope`, `Day_Group`, `Region`, and `Value`.

- `Scope` is `Annual`, `Day Group`, or `Setting`.
- `Region` is `Systemwide` or a bus (region) name.
- The metrics are `EUE` (MWh), `NEUE` (ppm), `LOLH` (hours/year), `LOLE` (days/year), `Max_Consecutive_Outage_Hours`, `Max_MW_Loss`, and `Max_MWh_Loss`.
- For each day group, `Scenarios_Sampled` and `Scenarios_Solved` give the sample size before and after screening. Only solved scenarios contribute.
- `System_Peak_Scale` records the stress level of the run.

Reading a metric together with its sample size and stress level avoids over-interpreting a small or heavily screened run.

### Dispatch files
The `_dispatch.csv` files describe generators and storage:

- Identification columns: `Rep_Day`, `Hour`, `Sub_Period`, `Unit_ID`, `Plant_Name`, `Bus_ID`, `Bus_Name`, `Tech_ID`, `Unit_Group`, `Unit_Category`, `Unit_Report_Label_1`, `Unit_Report_Label_2`, `Units_In_Service`, `Storage_Energy_MWh`, `Installed_Capacity_MW`.
- Post-contingency files add `Unit_Available_Flag` (`false` means the unit is on outage), `Generation_MW`, `Charge_MW`, `SOC_MWh`, the same three quantities from the no-outage reference (`Reference_Generation_MW`, `Reference_Charge_MW`, `Reference_SOC_MWh`), and `Outage_Event_Flag` (`true` when at least one unit is out in the scenario).
- Reference files carry `Marginal_Cost_USD_per_MWh` instead of the outage columns.

The `_system.csv` files hold one row per bus and hour: `Rep_Day`, `Hour`, `Sub_Period`, `Bus_ID`, `Bus_Name`, `Unserved_Energy_MW`, `Load_MW`. Post-contingency files add `Outage_Capacity_MW`, the capacity on outage at the bus.

Reference outputs are separated by day group and renewable scenario, so scenarios do not overwrite one another.

## Interpreting results
Start with the metrics CSV.

- Compare the annual `Systemwide` `NEUE`, `EUE`, and `LOLE` with the adequacy target of the study. Use `Regional` rows to find where shortfalls occur.
- Use the `Day Group` rows to see which representative days drive risk. Metrics are weighted by day-group counts, so a day group with a large weight matters more.
- Compare the `Max_*` metrics with the expected-value metrics. They report the worst solved scenario, not an average.
- Check `System_Peak_Scale`. The default of `1.2` stresses the system by 20%, so the metrics are not directly comparable with a run at `1.0`.

### Checks for a valid run
- The number of retained joint scenarios is reasonable: not zero, and not so large that most of the sampled space is being simulated when screening is enabled.
- Reference dispatch exists for every retained renewable scenario.
- Post-contingency results differ between renewable scenarios where they should.
- Metrics and dispatch files agree with the selected `RA_method`. In particular, do not compare `EUE` from `Economic Dispatch` and `Sequential Economic Dispatch` runs without considering the information difference above.

!!! note
    With `output_verbose_level` set to `compact`, the JSON omits the per-scenario solutions. Set it to `detailed`, or enable dispatch export, when scenario-level inspection is needed.

## Related documentation
- [RA Overview](./RA_Overview.md)
- [RA Formulation](./RA_Formulation.md#stage-2-risk-realization-post-contingency-dispatch): the equations for reference and post-contingency dispatch
- [RA Scenarios and Data](./RA_Scenarios_and_Data.md)
- [RA Settings and Scenarios Reference](./RA_Settings_and_Scenarios_Reference.md)
- [RA Metrics, ELCC, and DLOL](./RA_Metrics_ELCC_and_DLOL.md)
