# RA Settings and Scenarios Reference

This page documents every setting in the `RA Setting` sheet and every column of the `RA Scenarios` sheet of the simulation setting workbook (for example `ALEAF_Simulation_Setting_NorthAmerica.xlsx`). For each setting it gives the meaning, units, allowed values, the workbook default, and the settings it depends on. For the concepts behind the settings, see [RA Overview](./RA_Overview.md) and [RA Scenarios and Data](./RA_Scenarios_and_Data.md).

The `RA Setting` sheet has three columns: `Setting`, `Value`, and `Note`. Section header rows (for example `[ RA Setting]`) are for readability only.

## `RA Setting`

### Network setting

| Setting | Default | Meaning |
|---|---|---|
| `generate_gen_index_with_investment_options_flag` | `false` | Controls how RA sizes its generator index. When `false`, each unit is sized from its existing unit count only. When `true`, the index also reserves room for the technology's `MAXINVEST` limit, so candidate (not yet built) investment units are represented consistently across scenarios and runs. |
| `round_id_to_run_RA` | `1` | Expansion round whose investment decisions define the generation mix and year of the RA run. Used when RA is driven by expansion-planning results. |

### Scenario sampling and screening

| Setting | Default | Units / values | Meaning |
|---|---|---|---|
| `num_risk_scenario` | `50` | count | Number of outage samples drawn for each representative-day group. The full scenario space is `num_risk_scenario` times the number of enabled renewable scenarios. |
| `RA_method` | `Sequential Economic Dispatch` | `Economic Dispatch`, `Sequential Economic Dispatch` | Post-contingency dispatch mode. `Economic Dispatch` solves each day group once with perfect foresight of the outage. `Sequential Economic Dispatch` rolls forward hour by hour without foresight of outage duration. See [RA Execution and Results](./RA_Execution_and_Results.md#perfect-foresight-and-sequential-dispatch). |
| `system_peak_scale` | `1.2` | multiplier | Multiplies all load in the RA run (1.2 means load 20% above the database). It changes dispatch, scarcity, and every demand-normalized metric, so it is the main stress-level control. |
| `risk_filtering_flag` | `true` | `true`, `false` | When `true`, sampled scenarios are screened with `risk_tol_value` and `min_num_risk_in_each_day_value` and only the retained scenarios are simulated. When `false`, all scenarios are retained (subject to `preselected_days_list`). |
| `risk_tol_value` | `0.3` | ratio | Screening threshold. |
| `min_num_risk_in_each_day_value` | `0` | fraction of `num_risk_scenario` | Minimum number of scenarios retained per day group. |
| `repair_time_average_hours` | `32` | hours | Mean repair duration used when sampling restoration times of forced outages. |
| `repair_time_bound` | `95 percentile` | `average`, `95 percentile`, `90 percentile` | Truncates the sampled repair-time distribution at the selected statistic of the historical repair-time data, so restoration windows have no unbounded tail. |
| `reference_temp` | `18.3` | degrees Celsius | Reference temperature for the temperature-dependent forced-outage model. Outage probabilities increase as local temperature departs from this value. |
| `preselected_days_list` | `[]` | list of day-group IDs, for example `[3,7]` | Restricts RA to the listed day groups. An empty list makes all day groups eligible. |

#### `risk_tol_value`
Each candidate scenario receives a severity score: the maximum over its hours of the firm capacity lost to outages divided by the available generation margin. A scenario is retained when its score exceeds `risk_tol_value`. Higher values screen more aggressively and reduce run time, at the cost of discarding more marginal scenarios (which contribute zero to expected-value metrics). Applies only when `risk_filtering_flag` is `true`.

#### `min_num_risk_in_each_day_value`
A retention floor expressed as a fraction of `num_risk_scenario`. The effective floor per day group is the larger of 10 and `min_num_risk_in_each_day_value * num_risk_scenario`. If the tolerance-based screen retains fewer scenarios, the highest-scoring remaining ones are added back until the floor is met. A value of `0` disables the floor, so retention depends only on `risk_tol_value`.

### Capacity Credit / ELCC Settings

These settings apply after the base RA run and take effect only when `calculate_capacity_credit_flag` is `true`. See [RA Metrics, ELCC, and DLOL](./RA_Metrics_ELCC_and_DLOL.md) for the methods.

| Setting | Default | Allowed values | Meaning |
|---|---|---|---|
| `calculate_capacity_credit_flag` | `false` | `true`, `false` | Turns capacity-credit post-processing on or off. |
| `capacity_credit_type` | `ELCC` | `ELCC`, `DLOL` | Capacity-credit method. The two are mutually exclusive within a run. |
| `capacity_credit_assessment_mode` | `add_new` | `add_new`, `deactivate_existing` | ELCC only. `add_new` adds a candidate unit of each resource in the ELCC list. `deactivate_existing` removes an existing asset and measures the load relief needed to restore reliability. An unrecognized value falls back to `add_new` with a warning. |
| `capacity_credit_RA_simulation_method` | `Sequential Economic Dispatch` | `Economic Dispatch`, `Sequential Economic Dispatch` | Dispatch mode used inside the capacity-credit iterations, independent of `RA_method`. It allows, for example, a sequential base pass with perfect-foresight ELCC solves. |
| `capacity_credit_reference_RA_metric` | `NEUE` | `EUE`, `NEUE`, `LOLH`, `LOLE` | Reliability metric that ELCC holds constant. |
| `capacity_credit_reference_RA_metric_spatial_resolution` | `Regional` | `Systemwide`, `Regional` | Whether the target metric is evaluated for the whole system or per region. It also determines where the added constant load is placed. |
| `capacity_credit_max_iteration_value` | `15` | integer | Maximum number of ELCC search iterations. |
| `capacity_credit_abs_tol_value` | `0.1` | metric units (MWh for `EUE`, ppm for `NEUE`) | Absolute convergence tolerance on the metric gap. |
| `capacity_credit_rel_tol_value` | `0.001` | percent (`0.001` means 0.001%) | Relative convergence tolerance on the metric gap. |

#### Notes on `capacity_credit_assessment_mode`
- `add_new` inserts a unit sized at the technology's `CAP` at every bus whose `RA_ELCC_Calculation_Flag` is `true`, for each technology with `ELCC_Flag` set to `true` in the `Gen Technology` sheet.
- `deactivate_existing` removes the assets listed for assessment (matched by plant name and unit group). If `capacity_credit_reference_RA_metric_spatial_resolution` is `Regional` and a matched asset spans more than one bus, that target is skipped with status `regional_metric_bus_ambiguous`. Narrow the target or use `Systemwide`.

### Forced-outage and storage settings

| Setting | Default | Allowed values | Meaning |
|---|---|---|---|
| `Post_Contingency_Discharge_method` | `Optimal` | `Optimal`, `Same as Reference`, `Not Allowed` | Storage discharge policy after an outage begins. `Not Allowed` forces discharge to zero. `Same as Reference` pins discharge to the no-outage reference dispatch. Any other value, including `Optimal`, lets the model re-optimize discharge. |
| `Post_Contingency_Charge_method` | `Optimal` | `Optimal`, `Same as Reference`, `Not Allowed` | The same three policies applied to storage charging. Under `Optimal`, charging is free up to the installed storage capacity. |

See [Post-contingency storage](./RA_Formulation.md#post-contingency-storage) for the formulation.

### Execution and output settings

| Setting | Default | Units / values | Meaning |
|---|---|---|---|
| `distributed_run_flag` | `true` | `true`, `false` | Splits outage-sample generation across available worker processes. Post-contingency redispatch uses all available workers regardless of this flag. |
| `num_distributed_scenarios_per_worker_value` | `200` | count | Upper bound on the batch of scenarios sent to a worker in redispatch. The model may use a smaller batch on a worker to stay within available memory. |
| `output_verbose_level` | `compact` | `compact`, `detailed` | Amount of detail kept in the RA result JSON. `compact` (also used for any value other than `detailed`) removes per-scenario solutions, reference risk data, and renewable scenario data. `detailed` keeps the per-scenario solution content and the filtered joint scenario map. |
| `export_dispatch_results_threshold_value` | `50` | MW | Dispatch and system files of a scenario are written only if its peak unserved energy exceeds this value. It has an effect only when an export flag below is `true`. |
| `export_reference_dispatch_results_flag` | `false` | `true`, `false` | Writes dispatch and system files for the no-outage reference dispatch of each retained (day group, renewable scenario). |
| `export_baseline_dispatch_results_flag` | `false` | `true`, `false` | Writes dispatch and system files for outage scenarios in the base RA run that pass `export_dispatch_results_threshold_value`. |
| `export_ELCC_dispatch_results_flag` | `false` | `true`, `false` | Same as the baseline flag, for the runs inside the ELCC iterations. Can produce a large number of files. |
| `sequential_horizon_hours_value` | `6` | hours | Look-ahead window of each step under `Sequential Economic Dispatch`. Only the first hour of each solve is committed. A value of `1` gives a single-hour snapshot with no look-ahead. |
| `min_redispatch_mc_value` | `1e-05` | per-unit cost | Minimum redispatch cost applied in post-contingency dispatch so that free units do not cycle. The cost in USD/MWh is approximately the value multiplied by `per_unit_econ_base_value`, so `1e-05` is about 0.10 USD/MWh. |

#### `export_baseline_dispatch_results_flag`
This flag is independent of `export_reference_dispatch_results_flag`. The reference flag controls no-outage reference dispatch files only. The baseline flag controls post-contingency files and is further limited by `export_dispatch_results_threshold_value`. File names are listed in [RA Execution and Results](./RA_Execution_and_Results.md#what-ra-writes).

!!! note "Choosing dispatch export settings"
    Exporting every scenario produces many files. Keep both flags `false` for routine runs, and set the threshold above zero when exporting so that only severe scenarios are written.

## `RA Scenarios`

### Purpose
The `RA Scenarios` sheet defines the discrete, weighted renewable scenarios that are combined with outage samples. One row defines one scenario.

### Columns

| Column | Type | Meaning |
|---|---|---|
| `Scenario_ID` | text | Unique scenario name. It labels the scenario in reference dispatch, joint scenarios, and metric weighting. The base scenario must be named `BASE` (case-insensitive) and is always placed first. |
| `Enabled` | `true` / `false` | Only enabled rows participate. Disabled rows are ignored. |
| `Weight` | number | Relative probability weight of the scenario. Weights of enabled scenarios are re-normalized to sum to 1, so disabling a scenario raises the effective weight of the remaining ones. |
| `Wind_Ons_File_ID` | text | Identifier of the onshore wind timeseries file for the scenario. |
| `PV_File_ID` | text | Identifier of the utility-scale PV timeseries file for the scenario. |

The workbook default contains a single row: `BASE`, enabled, weight `1`, `Wind_Ons_File_ID` `Base`, `PV_File_ID` `Base`.

### File resolution
- For the `BASE` scenario, RA uses the onshore wind and PV timeseries files set in the `File Path` sheet (`timeseries_data_wind_ons_path` and `timeseries_data_pv_path`). The file IDs of the `BASE` row are not used to locate files.
- For any other scenario, RA reads `timeseries_data_files/0_additional_scenarios/WIND/timeseries_wind_ons_hourly_<Wind_Ons_File_ID>.csv` and `timeseries_data_files/0_additional_scenarios/PV/timeseries_pv_hourly_<PV_File_ID>.csv` under the case's data folder.

See [RA Scenarios and Data](./RA_Scenarios_and_Data.md#adding-a-renewable-scenario) for the procedure.

## Related documentation
- [RA Overview](./RA_Overview.md)
- [RA Formulation](./RA_Formulation.md): equations for the screening score, retention floor, and weighting
- [RA Scenarios and Data](./RA_Scenarios_and_Data.md)
- [RA Execution and Results](./RA_Execution_and_Results.md)
- [RA Metrics, ELCC, and DLOL](./RA_Metrics_ELCC_and_DLOL.md)
