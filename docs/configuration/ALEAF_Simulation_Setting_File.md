# ALEAF Simulation Setting File Reference

`ALEAF_Simulation_Setting_NorthAmerica.xlsx` (the bundled example workbook) is the main control file for A-LEAF. It defines:

- which system is studied and how its network is represented
- which model families run (expansion, operation, reliability assessment)
- how the simulation is sequenced over the planning horizon
- the technology, policy, financial and reliability assumptions
- which scenario tables and external inputs are used

This page describes the workbook sheet by sheet. For every setting it gives the meaning, allowed values, and the value in the bundled workbook. Case-level options are on the `Simulation Configuration` sheet, documented in the [Simulation Configuration Reference](./Simulation_Configuration_Reference.md).

## How the workbook is organized

The workbook contains two kinds of sheets:

- **Setting sheets** (`Simulation Setting`, `Planning Design`, `RA Setting`, `Scenario Reduction Setting`) list one named setting per row with a `Value` and a `Note`. Setting names are case-sensitive.
- **Table sheets** (for example `Gen Technology`, `RA Scenarios`, `ATB Setting`) hold one record per row (a technology, a scenario, a constraint).

The workbook contains these sheets:

| Sheet | Purpose |
|---|---|
| `Simulation Setting` | Model-wide behavior: system, network representation, solver, reporting |
| `Planning Design` | Planning horizon, rolling-horizon simulation, financial and transmission cost settings |
| `Gen Technology` | Master technology table |
| `RA Setting` | Reliability assessment (RA) settings |
| `Simulation Configuration` | Case-by-case options (one column per case) |
| `RA Scenarios` | Renewable scenarios used by RA |
| `Scenario Reduction Setting` | Representative-day selection |
| `ATB Setting` | Mapping of technology groups to ATB cost and performance assumptions |
| `Storage Cost and Performance` | Storage cost and performance assumptions |
| `ITC`, `PTC` | Investment and production tax credit values |
| `HiGHS Setting`, `CPLEX Setting`, `cuOpt Setting`, `MadNLP Setting` | Solver parameters |
| `Raw Materials`, `Gen Technology Raw Materials` | Raw-material limits and technology material intensities |
| `External Constraints` | User-defined additional constraints |
| `ATB Params` | Supporting ATB data |

Tabs named `Scenario ->`, `Financial Params ->`, `Solver Settings ->`, `Ancillary Tabs->` and similar are section dividers and hold no inputs.

!!! note "Boolean values"
    Flags accept `TRUE` or `FALSE`. Keep them as Boolean cells rather than text.

## 1. `Simulation Setting`

This sheet defines model-wide behavior, the solver, and output control.

### Test system

| Setting | Bundled value | Meaning |
|---|---|---|
| `test_system_name` | `NorthAmerica` | Name of the test-system folder under `data/`. The network database, time series and other inputs are read from `data/<test_system_name>/`, and the name labels the run outputs. |

### Topology settings

The network is always built from the database workbooks of the selected test system; there is no separate network-source setting.

#### `generation_representation_option`
Bundled value: `Aggregated_by_Tech`. Allowed values (case-sensitive):

- `Aggregated_by_Tech`: one generating unit per technology and bus. Smaller models.
- `Individual_Plants`: each plant is modeled explicitly. Larger models with plant-level detail.

#### `generation_parameter_source_option`
Bundled value: `Gen Technology`. Applies when `generation_representation_option = Individual_Plants`. Allowed values:

- `Gen Technology`: plants inherit parameters from the matching row of the `Gen Technology` sheet.
- `Individual Plant`: plant-specific values from the plant data are used directly.

#### `power_flow_mode_flag`
Bundled value: `B-theta`. Selects the transmission representation used in the optimization. Allowed values:

- `B-theta`: DC optimal power flow with bus voltage angles (Kirchhoff's voltage law is enforced).
- `PTDF`: flows computed from power transfer distribution factors.
- `Network_Flow`: transport model; flows are limited by line ratings only.

The choice affects solve time and the network data required. See the [Network Configuration Reference](../database/Network_Configuration_Reference.md).

!!! note "Asynchronous DC ties depend on the power flow mode"
    Branches flagged `dc_line = True` (back-to-back HVDC and variable-frequency transformers) are treated differently by mode. In `B-theta` they are excluded from the angle equations and modeled as bounded controllable transfers, with one reference bus per synchronous island. In `Network_Flow` they are ordinary transfer limits. `PTDF` does not support DC ties. See [GTEP Transmission Expansion](../models/GTEP/GTEP_Transmission_Expansion.md#asynchronous-dc-tie-modeling-b-theta).

#### `transmission_expansion_hybrid_flag`
Bundled value: `TRUE`. Default when the row is absent: `FALSE`. Enables the enhanced-hybrid `B-theta` transmission expansion formulation. In a meshed grid, expanding a line changes both its thermal rating and its susceptance. The standard relaxation scales only the rating; the hybrid formulation also captures the susceptance effect while remaining a linear program.

- Applies only with `power_flow_mode_flag = B-theta`. With `Network_Flow` or `PTDF` the flag is ignored and a warning is logged.
- Applies to expandable AC corridors. DC ties keep the standard bounded-transfer cap.
- The built capacity is carried into the operation and RA models through the branch susceptance.
- Adds three audit columns to the expansion `*__power_flow_EXP.csv` output.

See [GTEP Transmission Expansion](../models/GTEP/GTEP_Transmission_Expansion.md#enhanced-hybrid-b-theta-transmission-expansion) for the formulation and [GTEP Expansion Outputs](../models/GTEP/GTEP_Expansion_Outputs.md#5-power-flow-output) for the audit columns.

### Solution method settings

#### `solver_name`
Cell `Simulation Setting!B11`. Bundled value: `HiGHS` (the default). Allowed values:

| Value | Description |
|---|---|
| `HiGHS` | Default open-source LP/MIP solver. No extra setup. |
| `CPLEX` | Optional commercial solver. Requires a licensed CPLEX installation and `using CPLEX` (see [Getting Started](../Getting_Started.md#optional-cplex)). Selecting it without loading the package stops the run with an error. |
| `cuOpt` | Optional NVIDIA GPU solver (LP only). See [GPU Solvers](./GPU_Solvers.md). |
| `MadNLP` | Optional GPU interior-point solver. See [GPU Solvers](./GPU_Solvers.md). |

The parameters of the selected solver are read from its own sheet (`HiGHS Setting`, `CPLEX Setting`, `cuOpt Setting`, `MadNLP Setting`); see [Section 11](#11-solver-sheets).

!!! note "cuOpt market prices are approximate and LP-only"
    With `cuOpt`, `LMP` and the reserve clearing prices (`RCP_*`) are computed from first-order (PDLP) duals. They are approximate compared with a simplex solve and are available only for pure LP models. Relax any `Integrality` flags for priced runs. See [GPU Solvers](./GPU_Solvers.md).

#### `run_GTEP_in_parallel_flag`
Bundled value: `FALSE`. Runs expansion cases in parallel when multiple worker processes are available. Do not enable it when running the RA model.

#### `run_operation_in_parallel_flag`
Bundled value: `TRUE`. Distributes operation simulations across worker processes. Each worker solves its assigned representative-day groups, and the per-group results are merged into the standard output files after the run. See [Serial vs distributed operation solves](../models/Operation/Operation_Execution_and_Run_Modes.md#serial-vs-distributed-operation-solves) for output layout, memory and failure-handling behavior. Set it to `FALSE` for a single-process run.

### Report control (GTEP / Operation)

These flags select which CSV report families the expansion (EXP) and operation (OP) models write. They apply to the expansion and operation models only; RA has its own export flags (see [RA Settings and Scenarios Reference](../models/RA/RA_Settings_and_Scenarios_Reference.md)). All flags are on the `Simulation Setting` sheet.

Bundled value is `TRUE` for every flag except `report_multi_round_summary_json_EXP_flag`. A missing row or blank cell also writes the report; an explicit `FALSE` (also accepted: `0`, `no`, `n`, `f`, case-insensitive) suppresses it.

| Flag | Model | Report files controlled |
|---|---|---|
| `report_summary_EXP_flag` | EXP | `*__tech_summary_by_stage_EXP.csv`, `*__tech_summary_by_year_EXP.csv`, `*__system_summary_by_stage_EXP.csv`, `*__system_summary_by_year_EXP.csv` |
| `report_dispatch_EXP_flag` | EXP | `*__dispatch_EXP_year_*.csv`, `*__market_EXP.csv`, `*__policy_slack_EXP.csv`, `*__demand_response_EXP.csv` |
| `report_power_flow_EXP_flag` | EXP | `*__power_flow_EXP.csv` |
| `report_scarcity_EXP_flag` | EXP | `*__unserved_energy_EXP.csv`, `*__reserve_shortfall_EXP.csv` |
| `report_expansion_flag` | EXP | `*__gen_expansion_EXP.csv`, `*__line_expansion_EXP.csv` |
| `report_summary_OP_flag` | OP | `*__tech_summary_by_stage_OP.csv`, `*__system_summary_by_stage_OP.csv`, `*__system_summary_by_year_OP.csv` |
| `report_dispatch_OP_flag` | OP | `*__dispatch_OP_year_*.csv`, `*__market_OP.csv`, `*__policy_slack_OP.csv`, `*__demand_response_OP.csv` |
| `report_power_flow_OP_flag` | OP | `*__power_flow_OP.csv` |
| `report_multi_round_summary_json_EXP_flag` | EXP | `GTEP_multi_round_info.json` (see below) |

The representative-day files (`*_representative_days_EXP.csv`, `*_representative_days_OP.csv`) and `*__simulation_run_time.csv` are always written.

!!! warning "`report_multi_round_summary_json_EXP_flag` controls retention, not creation"
    `GTEP_multi_round_info.json` is always written during a run, because it carries the expansion result to the RA model, serves as the multi-round resume checkpoint, and can be reused through `predefined_expansion_data_file_name_for_RA` and `predefined_expansion_data_file_name_for_OP`. The flag only decides whether the file is kept after the run finishes. With `TRUE` it is kept. With `FALSE` (the bundled value) it is deleted at the end of the run. Set it to `TRUE` if a later run needs to reuse the file.

### Other simulation settings

| Setting | Bundled value | Meaning |
|---|---|---|
| `logging_level_value` | `simple` | Console and log verbosity. Allowed values: `simple`, `detailed` (case-insensitive). |
| `PTDF_threshold_value` | `0.0001` | Shift factors with absolute value below this threshold are dropped to sparsify the flow constraints. Used only when `power_flow_mode_flag = PTDF`. |
| `per_unit_base_value` | `100` | Per-unit power base (MVA). |
| `per_unit_econ_base_value` | `10000` | Per-unit economic base used to scale cost coefficients for numerical conditioning. Results are reported in physical units. |
| `const_name_flag` | `FALSE` | Keeps explicit constraint names in the optimization model. Increases memory; useful for debugging and LP export. |
| `export_model_reference_json_expansion_flag` | `FALSE` | Exports the expansion model input data as JSON. |
| `export_model_reference_json_operation_flag` | `FALSE` | Exports the operation model input data as JSON. |
| `export_model_lp_expansion_flag` | `FALSE` | Exports the expansion model in `.lp` format. |
| `export_model_lp_operation_flag` | `FALSE` | Exports the operation models in `.lp` format. |
| `model_lp_file_name_value` | `ALEAF_LC_GTEP_model_instance.lp` | File name for the exported `.lp` model. |
| `MIP_relaxed_solution_bounds_flag` | `FALSE` | Speeds up mixed-integer investment problems: the LP relaxation is solved first, and its solution is used to tighten the bounds of the integer investment variables. |
| `MIP_relaxed_solution_bounds_step_value` | `1` | Step size (plus or minus, in integer units) used when redefining the integer investment bounds. |

#### `export_model_reference_json_operation_flag`
Setting either `export_model_reference_json_expansion_flag` or `export_model_reference_json_operation_flag` to `TRUE` has a memory side effect for the operation model. In the default configuration, the operation model keeps a light in-memory data set that omits the per-representative-day hourly data and aggregations needed only for the JSON export. When either flag is `TRUE`, the full data set is built instead. On large (nodal) systems this can substantially increase memory use. Leave both flags `FALSE` unless the JSON export is required. See [Light master reference](../models/Operation/Operation_Execution_and_Run_Modes.md#light-master-reference-nodal-out-of-memory-behavior).

## 2. `Planning Design`

This sheet defines the planning horizon, the rolling-horizon simulation, financial assumptions, transmission cost and loss settings, and other planning-wide parameters.

### Planning horizon

| Setting | Bundled value | Meaning |
|---|---|---|
| `dollar_year_value` | `2022` | Dollar year of all cost inputs. Costs are discounted to this year, so present values are in this year's dollars. Fuel and storage cost data must be expressed in this dollar year. |
| `base_year_value` | `2021` | Year that the input load data represents. Load growth compounds from this year. It does not affect time-series data or discounting. |
| `first_stage_year_value` | `2025` | Calendar year of the first planning stage. Stage k is labeled `first_stage_year_value + (k - 1) x num_years_per_stage_value`. |
| `num_stages_value` | `2` | Number of planning stages. Investment decisions are made once per stage. |
| `num_years_per_stage_value` | `2` | Years represented by each stage. Only the first year of a stage is simulated with representative days; its results are scaled to the remaining years. |
| `horizon_end_year_value` | `2028` | Optional, information only. The last modeled year is `first_stage_year_value + num_stages_value x num_years_per_stage_value - 1`. The model does not use this value and logs a warning if it disagrees with the computed year. |

### Rolling-horizon (multi-round) simulation

In a multi-round run, the planning horizon is solved in a sequence of rounds. Each round makes binding build decisions for its decision stages and can also model look-ahead stages whose decisions are not committed.

| Setting | Bundled value | Meaning |
|---|---|---|
| `multi_round_solution_process_flag` | `TRUE` | `FALSE`: one optimization over the whole horizon. `TRUE`: rolling multi-round optimization. |
| `num_decision_stages_per_round_value` | `1` | Number of stages whose build decisions are committed in each round (at least 1). |
| `num_lookahead_stages_per_round_value` | `0` | Number of additional future stages modeled in each round but not committed (0 or more). `0` gives a myopic run that considers only the decision stages of the current round. |
| `continue_from_previous_run_flag` | `FALSE` | Resumes a multi-round simulation from the last completed round of a prior run. |

Look-ahead stages give the model foresight about later conditions (for example policy tightening) at the cost of a larger problem in each round.

### Financial settings

| Setting | Bundled value | Meaning |
|---|---|---|
| `WACC_value` | `0.054` | Weighted average cost of capital (real). |
| `discount_rate_value` | `0.054` | Social discount rate. Set equal to `WACC_value` when the ATB capital recovery factors are already discounted at the WACC. |
| `project_finance_factor_value` | `1.052` | Project-finance adjustment factor applied to capital costs (ATB uses 1.052). |
| `reserve_cost_type_flag` | `percentage` | Interpretation of the reserve cost columns. `percentage`: percent of marginal cost. `absolute`: value in $/MWh. |
| `min_regulation_cost_value` | `2` | Floor on regulation reserve cost, in $/MWh, applied to regulation-up plus regulation-down procurement in the objective. It prevents free regulation-down procurement. `0` disables the floor; when the row is absent, no floor is applied. |

### Transmission expansion and losses

#### `transmission_expansion_limit_value`
Bundled value: `2`. Upper bound on transmission expansion on a corridor, as a multiple of its existing capacity.

#### `transmission_investment_CRP_value`
Bundled value: `40`. Capital recovery period, in years, for transmission investments.

#### `transmission_route_length_adder_value`
Optional; not present in the bundled workbook. Default when absent: `1.0` (no adder). Multiplier applied to each branch's straight-line length to approximate a real route, which detours around terrain, water and land use. It scales the AC per-MW-mile expansion cost and the AC approach-line cost of DC ties. It does not scale the DC converter-station cost, which is per MW and independent of length. For example, `1.3` applies a 30% routing adder.

#### `transmission_FOM_percent_value`
Optional; not present in the bundled workbook. Default when absent: `0` (no transmission fixed O&M). Annual transmission fixed operations-and-maintenance (FOM) cost as a percent of overnight capital (`3.31` means 3.31%). It is a recurring cost charged in every operating year and is not annualized with the capital recovery factor. It enters the expansion objective for new lines, and appears in the `Trans_FOM_Cost` and `Trans_FOM_Cost_PV` system-summary columns for the whole in-service grid. See [GTEP Transmission Expansion](../models/GTEP/GTEP_Transmission_Expansion.md#transmission-fixed-om-fom) and [GTEP Expansion Outputs](../models/GTEP/GTEP_Expansion_Outputs.md#11-system-summary-output).

#### `enforce_transmission_loss_flag`
Bundled value: `TRUE`. Controls whether the flat, system-wide transmission and distribution (T&D) loss adder is applied in the demand balance. With `FALSE`, the model is lossless. With `TRUE`, demand at every bus is increased by `transmission_loss_percent_value` in all three `power_flow_mode_flag` modes and in the expansion, operation and RA models.

!!! warning "Required row"
    The row must be present on the `Planning Design` sheet with an explicit `TRUE` or `FALSE`. A workbook without it fails when the model is built.

#### `transmission_loss_percent_value`
Bundled value: `5`. Flat, system-wide T&D loss percentage (`3` means 3%). Used only when `enforce_transmission_loss_flag = TRUE`. The model serves `demand x (1 + percent / 100)` at every bus and hour, so generation and imports cover both delivered load and the loss adder.

This is a demand gross-up, not a flow-dependent loss model: losses do not vary with line loading, topology or distance. The resulting annual loss energy is reported in the `TnD_Loss` column of the system summaries ([GTEP Expansion Outputs](../models/GTEP/GTEP_Expansion_Outputs.md#11-system-summary-output)). See also the power flow outputs in [GTEP Expansion Outputs](../models/GTEP/GTEP_Expansion_Outputs.md#5-power-flow-output) and [Operation Outputs](../models/Operation/Operation_Outputs.md#5-power-flow-output).

!!! note "Per-MW-mile and per-MW transmission expansion costs are case-level settings"
    `transmission_cost_dollar_per_MW_mile_value` (AC lines, $/MW-mile) and `dc_tie_expansion_cost_dollar_per_MW_value` (DC-tie converters, $/MW) are set per case on the `Simulation Configuration` sheet, not on `Planning Design`. They are annualized with `WACC_value` and `transmission_investment_CRP_value` from this sheet, and the overnight basis is modified by `transmission_route_length_adder_value` and `transmission_FOM_percent_value`. See the [Simulation Configuration Reference](./Simulation_Configuration_Reference.md#transmission_cost_dollar_per_mw_mile_value).

### Inertia and materials

| Setting | Bundled value | Meaning |
|---|---|---|
| `enforce_rotational_inertia_constraints_flag` | `FALSE` | Applies interconnection-level inertia constraints. Ignored unless unit commitment is enabled. |
| `enforce_material_constraints_flag` | `FALSE` | Applies the annual raw-material limits from the `Raw Materials` and `Gen Technology Raw Materials` sheets in the expansion model. |

### Time resolution and other parameters

| Setting | Bundled value | Meaning |
|---|---|---|
| `num_hours_per_day_value` | `24` | Hours per representative day (24 for hourly resolution). |
| `num_sub_period_value` | `1` | Sub-periods per hour. `1` means hourly. |
| `reference_carbon_emission_level_value` | `981` | Reference system emissions (megaton) noted for the CO2 reduction target. The target's actual baseline is taken from the policy-zone data (`Ref_Emission_m_ton`) in the network database; this row is not used to compute the target. |

## 3. `Gen Technology`

The master technology table. Each row defines one generation or storage technology option. Columns cover:

- category and unit labels
- dispatch and commitment options
- cost inputs
- technical characteristics
- market and policy attributes
- reliability parameters
- investment options (including the `Integrality` column)
- resource limits

Each technology has a `Tech_ID`. Many cost fields can be tied to the National Renewable Energy Laboratory [Annual Technology Baseline (ATB)](https://atb.nrel.gov/) through the `ATB Setting` sheet (Section 8) or to other sources. Profile shape and variable-renewable and hydro treatment are set by the `Profile_Type`, `VRE_Flag` and `Hydro_Flag` columns. See the [Gen Technology Reference](../database/Gen_Technology_Reference.md) for all columns.

## 4. `RA Setting`

Controls the reliability assessment (RA) workflow. See the [RA Overview](../models/RA/RA_Overview.md) and the [RA Settings and Scenarios Reference](../models/RA/RA_Settings_and_Scenarios_Reference.md).

!!! note "Reusing saved expansion output for RA"
    Whether RA reads saved expansion output instead of the current run is set per case on the `Simulation Configuration` sheet (`Use_predefined_expansion_data_for_RA_flag`; see the [Simulation Configuration Reference](./Simulation_Configuration_Reference.md#use_predefined_expansion_data_for_ra_flag)), with the file named in `predefined_expansion_data_file_name_for_RA`.

### Network settings

| Setting | Bundled value | Meaning |
|---|---|---|
| `generate_gen_index_with_investment_options_flag` | `FALSE` | Builds a generator index that includes the investment-option mapping, keeping unit indexing consistent across scenarios and reused inputs. |
| `round_id_to_run_RA` | `1` | Expansion round whose investment state is used for the RA run. |

### Outage sampling and dispatch

| Setting | Bundled value | Meaning |
|---|---|---|
| `num_risk_scenario` | `50` | Number of outage scenarios sampled per representative-day group. |
| `RA_method` | `Sequential Economic Dispatch` | RA simulation method: `Economic Dispatch` or `Sequential Economic Dispatch`. |
| `system_peak_scale` | `1.2` | Multiplier applied to all loads in RA runs (`1.2` = load 20% above the database). |
| `risk_filtering_flag` | `TRUE` | Filters sampled outage scenarios using `risk_tol_value` and the minimum-retention rule. |
| `risk_tol_value` | `0.3` | Filtering threshold based on outage severity relative to the available installed-capacity margin of each day group. |
| `min_num_risk_in_each_day_value` | `0` | Minimum retained scenarios per day group, as a fraction of `num_risk_scenario`. When greater than 0, at least 10 scenarios are retained per day group. `0` applies no minimum. |
| `repair_time_average_hours` | `32` | Mean repair duration (hours) used when sampling restoration windows. |
| `repair_time_bound` | `95 percentile` | Truncation of sampled repair durations. Allowed values: `average`, `95 percentile`, `90 percentile`, `NA`. |
| `reference_temp` | `18.3` | Reference temperature (degrees C) for temperature-dependent forced-outage sampling. |
| `sequential_horizon_hours_value` | `6` | For `Sequential Economic Dispatch`, the number of hours solved together at each step (look-ahead window). `1` is a single-hour snapshot; larger values allow charging look-ahead. |
| `min_redispatch_mc_value` | `1e-05` | Minimum redispatch cost, in per-unit cost, that prevents free units from cycling in post-contingency dispatch. Approximately `value x per_unit_econ_base_value` in $/MWh, so `1e-05` is about $0.10/MWh. The default when the row is absent is `0.0001`. |

### Storage during contingencies

| Setting | Bundled value | Meaning |
|---|---|---|
| `Post_Contingency_Discharge_method` | `Optimal` | Storage discharge policy in post-contingency dispatch. |
| `Post_Contingency_Charge_method` | `Optimal` | Storage charging policy in post-contingency dispatch. |

### Capacity accreditation

| Setting | Bundled value | Meaning |
|---|---|---|
| `calculate_capacity_credit_flag` | `FALSE` | Runs capacity accreditation after the baseline RA simulation. |
| `capacity_credit_type` | `ELCC` | Accreditation method: `ELCC`, `DLOL` or `Lookup Table`. |
| `capacity_credit_assessment_mode` | `add_new` | ELCC evaluation mode. `add_new` adds a candidate resource (also used if the value is not recognized); `deactivate_existing` removes an existing resource. |
| `capacity_credit_RA_simulation_method` | `Sequential Economic Dispatch` | RA simulation method used inside accreditation iterations. |
| `capacity_credit_reference_RA_metric` | `NEUE` | Reliability metric used as the accreditation target, for example `EUE` or `NEUE`. |
| `capacity_credit_reference_RA_metric_spatial_resolution` | `Regional` | Scope of the target metric: `Systemwide` or `Regional`. |
| `capacity_credit_max_iteration_value` | `15` | Maximum iterations of the ELCC solution loop. |
| `capacity_credit_abs_tol_value` | `0.1` | Absolute convergence tolerance in metric units (MWh for `EUE`, ppm for `NEUE`). |
| `capacity_credit_rel_tol_value` | `0.001` | Relative convergence tolerance in percent (`0.001` means 0.001%). |

### Execution and output

| Setting | Bundled value | Meaning |
|---|---|---|
| `distributed_run_flag` | `TRUE` | Runs RA scenario batches on multiple worker processes. |
| `num_distributed_scenarios_per_worker_value` | `200` | Cap on the scenario batch size per worker; the effective size may be smaller. |
| `export_dispatch_results_threshold_value` | `50` | Minimum peak unserved energy (MW) that an outage scenario must exceed to have its dispatch files written. Default when the row is absent: no scenario files are exported. |
| `export_reference_dispatch_results_flag` | `FALSE` | Writes dispatch and system files for the reference (no-outage) dispatch of each day group and renewable scenario. |
| `export_baseline_dispatch_results_flag` | `FALSE` | Writes dispatch and system files for outage scenarios of the baseline RA run, subject to the threshold above. See [RA Settings and Scenarios Reference](../models/RA/RA_Settings_and_Scenarios_Reference.md#export_baseline_dispatch_results_flag). |
| `export_ELCC_dispatch_results_flag` | `FALSE` | Same as the two flags above, for runs inside capacity-accreditation iterations. Can produce many files. |
| `preselected_days_list` | `[]` | Optional whitelist of day groups for RA. An empty list means all eligible day groups. |
| `output_verbose_level` | `compact` | RA output detail: `compact` or `detailed` (keeps per-scenario and risk solution content). |

## 5. `Simulation Configuration`

This sheet holds case-level settings: one column per case, with a category column and a setting-name column preceding the case columns. It determines whether a case runs, which model family it uses, which input data variants apply, representative-day settings, dispatch modes, policy flags, reliability and reserve options, and scenario overrides for load, wind, PV, fuel, hydro and ATB selections. It has its own page: see the [Simulation Configuration Reference](./Simulation_Configuration_Reference.md).

The environment variable `ALEAF_CASE_ID` restricts a run to the single case whose column number in the `Simulation Configuration` sheet matches it (the case must have `Run_Flag` enabled); see [GPU Solvers](./GPU_Solvers.md#running-one-case-per-process).

## 6. `RA Scenarios`

Defines the renewable scenarios used by RA. Enabled scenarios are combined with outage samples to form the joint scenario space.

| Column | Meaning |
|---|---|
| `Scenario_ID` | Renewable scenario name (bundled: `BASE`) |
| `Enabled` | Whether the scenario participates in the RA run |
| `Weight` | Probability weight used in annual RA metrics |
| `Wind_Ons_File_ID` | Identifier of the onshore wind time series |
| `PV_File_ID` | Identifier of the PV time series |

## 7. `Scenario Reduction Setting`

Controls representative-day selection when representative days are generated from full-year time series. See [Scenario Reduction and Repday Groups](./Scenario_Reduction_and_Repday_Groups.md) for the method.

| Setting | Bundled value | Meaning |
|---|---|---|
| `time_resolution` | `Hourly` | Time resolution for scenario reduction. Only hourly is supported. |
| `repday_selection_resolution` | `system` | Spatial scope of selection: `system` (one set of days for the whole system) or `regional` (one set per data region). Unrecognized values fall back to selection at the run resolution. See [selection resolution vs run resolution](./Scenario_Reduction_and_Repday_Groups.md#selection-resolution-vs-run-resolution). |
| `generate_input_data_flag` | `TRUE` | Generates the input matrices for the scenario-reduction algorithm. |
| `input_type_load_shape_flag` | `FALSE` | Uses load shape as a clustering feature. |
| `input_type_load_MWh_flag` | `TRUE` | Uses load energy (MWh) as a clustering feature. |
| `input_type_wind_shape_flag` | `FALSE` | Uses wind shape as a clustering feature. |
| `input_type_wind_MWh_flag` | `TRUE` | Uses wind energy (MWh) as a clustering feature. |
| `input_type_solar_shape_flag` | `FALSE` | Uses solar shape as a clustering feature. |
| `input_type_solar_MWh_flag` | `TRUE` | Uses solar energy (MWh) as a clustering feature. |
| `input_type_net_load_MWh_flag` | `TRUE` | Uses net-load energy (MWh) as a clustering feature. |
| `fixing_extreme_days_flag` | `TRUE` | Forces retention of extreme days. |
| `fix_peak_demand_day_flag` | `TRUE` | Keeps the peak-demand day. |
| `fix_peak_net_demand_day_flag` | `FALSE` | Keeps the peak net-demand (load minus VRE) day. |
| `fix_peak_solar_generation_day_flag` | `FALSE` | Keeps the maximum solar generation day. |
| `fix_peak_wind_generation_day_flag` | `FALSE` | Keeps the maximum wind generation day. |
| `fix_least_solar_generation_day_flag` | `TRUE` | Keeps the minimum solar generation day. |
| `fix_least_wind_generation_day_flag` | `TRUE` | Keeps the minimum wind generation day. |
| `allow_repday_overlap_flag` | `FALSE` | Allows the same day to appear in more than one representative-day group. |
| `preselected_extreme_days_list` | `[]` | Day-group identifiers forced into the selection, for example `[1;2;3]`. `[]` means none. |

The number of enabled `input_type_*` flags must equal the number of data sets configured for the selection.

## 8. `ATB Setting`

Maps A-LEAF technology groups to assumptions from NREL's Annual Technology Baseline. Columns include `ATB_Setting_ID`, `UNITGROUP`, fuel and technology classifications, and the ATB case and scenario selections.

## 9. `Storage Cost and Performance`

Storage cost and performance assumptions by setting identifier and storage duration and type: power and duration definitions, round-trip efficiency, cycle life, calendar life, annual energy throughput, fixed and variable O&M, and year-by-year cost trajectories. The sheet is used when storage cost and performance are selected through the case configuration and technology mappings. See the [Storage Modeling Reference](../database/Storage_Modeling_Reference.md).

## 10. `ITC` and `PTC`

Year-by-year investment tax credit (`ITC`) and production tax credit (`PTC`) values by technology. They apply when the corresponding credits are activated in the case configuration. See [Policy and Financial Settings](./Policy_and_Financial_Settings.md).

## 11. Solver sheets

Each solver sheet has columns `Parameter`, `Flag`, `Value` and `Note`. Set `Flag` to `TRUE` for a row to apply its `Value` to the solver; rows with `Flag = FALSE` keep the solver default. Only the sheet of the selected `solver_name` is used.

Every solver sheet begins with `solver_direct_mode_flag` (bundled value `FALSE`). Direct mode is only available for CPLEX; keep it `FALSE` for HiGHS, cuOpt and MadNLP. For cuOpt in particular, `TRUE` prevents the solver parameters from being applied.

### `HiGHS Setting`
Default solver.

| Parameter | Bundled value | Meaning |
|---|---|---|
| `time_limit` | `6000` | Maximum solver time in seconds. |
| `presolve` | `on` | Enables presolve. |
| `threads` | `1` | Number of solver threads. |
| `output_flag` | `TRUE` | Enables solver log output on the console. |

### `CPLEX Setting`
Used only when `solver_name = CPLEX`. Bundled parameters:

| Parameter | Flag | Value | Meaning |
|---|---|---|---|
| `CPX_PARAM_EPGAP` | `TRUE` | `0.001` | Relative MIP gap tolerance. |
| `CPXPARAM_TimeLimit` | `TRUE` | `86400` | Wall-clock time limit (seconds). |
| `CPX_PARAM_THREADS` | `TRUE` | `36` | Number of solver threads. Adjust to the machine. |
| `CPX_PARAM_LPMETHOD` | `TRUE` | `4` | LP algorithm: 0 auto, 1 primal simplex, 2 dual simplex, 3 network, 4 barrier, 5 sifting, 6 concurrent. |
| `CPX_PARAM_SCRIND` | `TRUE` | `1` | Displays solver messages on screen (0 or 1). |
| `CPX_PARAM_PARAMDISPLAY`, `CPX_PARAM_MIPDISPLAY`, `CPX_PARAM_EPRHS` | `FALSE` | `1`, `2`, `1e-06` | Display and feasibility-tolerance parameters; not applied unless the flag is `TRUE`. |

### `cuOpt Setting` and `MadNLP Setting`
Parameter sheets for the optional GPU solvers. `cuOpt Setting` provides the PDLP algorithm (`method`), `crossover`, and primal, dual and gap tolerances. `MadNLP Setting` provides the KKT linear-system form (`kkt_system`, bundled `SparseCondensed`) and the convergence tolerance (`tol`, bundled `1e-06`). See [GPU Solvers](./GPU_Solvers.md) for values and caveats.

## 12. Raw material sheets

| Sheet | Contents |
|---|---|
| `Raw Materials` | Material identifiers, types and annual limits |
| `Gen Technology Raw Materials` | Material intensity of technologies (links technology identifiers to material quantities) |

These sheets matter only when `enforce_material_constraints_flag = TRUE`.

## 13. `External Constraints`

User-defined constraints applied to selected models. Typical columns are the constraint identifier, an apply flag, the target model, the resource unit group, the parameter, the region identifier, the operator and the value. Use this sheet for additional restrictions such as investment caps, retirement requirements or regional build limits.

## Practical guidance

- Start with `Simulation Setting`, `Planning Design` and `Simulation Configuration` to control overall run behavior.
- Use `Gen Technology`, `ATB Setting` and `Storage Cost and Performance` for technology assumptions.
- Use `RA Setting` and `RA Scenarios` for reliability assessment.
- Use `Scenario Reduction Setting` when representative days are generated from full-year time series.
- Use `External Constraints` only for custom user-defined limits.
- Change one group of settings at a time and compare results against the bundled configuration.

## Related documentation

- [A-LEAF Documentation](../README.md)
- [Simulation Configuration Reference](./Simulation_Configuration_Reference.md)
- [Policy and Financial Settings](./Policy_and_Financial_Settings.md)
- [Scenario Reduction and Repday Groups](./Scenario_Reduction_and_Repday_Groups.md)
- [RA Overview](../models/RA/RA_Overview.md)
- [RA Settings and Scenarios Reference](../models/RA/RA_Settings_and_Scenarios_Reference.md)
