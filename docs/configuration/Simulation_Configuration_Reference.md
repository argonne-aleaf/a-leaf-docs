# Simulation Configuration Reference

The `Simulation Configuration` sheet of the setting workbook is the case-level control surface of A-LEAF. Each case occupies one column. A case determines:

- which models run (expansion, operation, reliability assessment);
- which input data variants are used;
- how representative days are selected;
- which dispatch, reserve, policy, and reliability options apply.

Use this page to look up the meaning, allowed values, and dependencies of each row. Global (run-wide) settings are documented in the [Simulation Setting File Reference](./ALEAF_Simulation_Setting_File.md).

## Sheet structure

The sheet is a case matrix:

- column `A` holds a category label;
- column `B` holds the setting name;
- columns `C`, `D`, `E`, and beyond hold the values for individual cases.

Setting names are case-sensitive and must be spelled exactly as shown below. A case is executed only when its `Run_Flag` is `TRUE`.

## Running linked workflows

The three model run flags are `Run_expansion_flag`, `Run_operation_flag`, and `Run_RA_flag`. A single case can enable more than one of them, which defines a linked workflow. The models run in the fixed order expansion, operation, reliability assessment.

- If `Run_expansion_flag` is on, expansion runs first. A subsequent operation or RA run in the same case uses the expansion result just produced, unless the corresponding `Use_predefined_expansion_data_for_*_flag` is on.
- If `Use_predefined_expansion_data_for_OP_flag` (or `..._RA_flag`) is on, the operation (or RA) model starts from a saved expansion result named in `predefined_expansion_data_file_name_for_OP` (or `..._RA`) instead.

## 1. Case definition

| Setting | Meaning | Values |
|---|---|---|
| `Run_Flag` | Turns the case on or off. | `TRUE` / `FALSE` |
| `Case_ID` | Case name used in output folders, file names, and logs. | Text |

## 2. Data file selection and representative days

The `*_File_ID` settings select which input-file variant a case reads. The value `Base` selects the default file; any other value selects a scenario variant of the corresponding file (see the Network Data and Network Configuration references).

| Setting | Meaning |
|---|---|
| `Network_Data_File_ID` | Selects the network data variant. |
| `Network_Configuration_File_ID` | Selects the network configuration variant (for example regional aggregation or topology configuration). |
| `Repday_File_ID` | Selects a predefined representative-day file. Used when the selection mode is `Manual`. |
| `Repday_Selection_Mode_EXP` | How representative days are obtained for the expansion model. |
| `Repday_Selection_Mode_OP` | Same, for the operation model. |
| `Repday_Selection_Mode_RA` | Same, for the reliability model. |

The selection-mode settings accept `Manual` (read the file named by `Repday_File_ID`) or `Scenario Reduction` (compute representative days from the full-year time series). When the manual file is missing or does not match the requested number of day groups and days per group, A-LEAF falls back to scenario reduction. See [Scenario Reduction and Representative-Day Groups](./Scenario_Reduction_and_Repday_Groups.md) for the resulting structure.

!!! note "Selection resolution is a global setting"
    The spatial resolution used to select representative days is not a per-case column. It is set by the global key `repday_selection_resolution` in the `Scenario Reduction Setting` sheet. See [Selection resolution versus run resolution](./Scenario_Reduction_and_Repday_Groups.md#selection-resolution-vs-run-resolution).

## 3. Expansion model settings

| Setting | Meaning | Values / units |
|---|---|---|
| `Run_expansion_flag` | Runs the expansion (GTEP) model. | `TRUE` / `FALSE` |
| `Dispatch_Mode_in_EXP` | Operational representation inside the expansion model. It changes the variables and constraints that are built, not only a label. | `Economic Dispatch`, `Unit Commitment` |
| `transmission_expansion_flag` | Allows the expansion model to build transmission capacity. | `TRUE` / `FALSE` |
| `operating_reserve_modeling_option` | How reserves are represented: reserve provision aggregated over units in a reserve zone, or tracked per individual unit. | `aggregated`, `individual` |
| `include_dispatch_ramping_flag` | Enforces generator ramping limits. | `TRUE` / `FALSE` |
| `dispatch_ramping_modeling_option` | Ramping formulation used when ramping is enabled and dispatch is not handled through full unit commitment. | `individual` (see the workbook for other options) |
| `NDAY_Groups` | Number of representative-day groups in the expansion model. | Integer |
| `NDAYS_in_Single_Group` | Number of consecutive days in each group. | Integer |
| `Load_Shed_in_EXP_Flag` | Allows unserved energy (load shedding, priced at `VOLL`). | `TRUE` / `FALSE` |
| `OR_Shortage_in_EXP_Flag` | Allows operating-reserve shortfalls (priced by the reserve penalties in section 10). | `TRUE` / `FALSE` |

`NDAY_Groups` and `NDAYS_in_Single_Group` define the size of the reduced calendar: the model uses `NDAY_Groups x NDAYS_in_Single_Group` representative days.

### `transmission_cost_dollar_per_MW_mile_value`

Overnight cost of **AC** transmission expansion, in $/MW-mile. The cost of a candidate line build is

`cost x rate_a x length x transmission_route_length_adder_value`

and is annualized with the transmission capital recovery factor (from `transmission_investment_CRP_value` and `WACC_value`). The optional route-length adder `transmission_route_length_adder_value` (`Planning Design` sheet; default `1.0` when the row is absent) scales the straight-line length up to an approximate routed length. The optional `transmission_FOM_percent_value` (default `0` when absent) adds an annual fixed O&M charge.

Because the cost is a case-level setting, transmission-cost sensitivities can be defined as separate cases (for example a base value and plus/minus 30 percent variants). The bundled workbook uses `1666` $/MW-mile.

### `dc_tie_expansion_cost_dollar_per_MW_value`

Overnight cost of a **single** converter station for expanding an asynchronous DC tie (back-to-back HVDC or variable-frequency transformer), in $/MW. A tie needs a converter on each side, so the cost of a tie expansion is

`(2 x dc_tie_expansion_cost_dollar_per_MW_value + transmission_cost_dollar_per_MW_mile_value x length x transmission_route_length_adder_value) x rate_a`

annualized with the same capital recovery factor as AC lines. The route-length adder applies only to the AC approach-line term. A branch is treated as a DC tie when its `dc_line` column is `True`. The bundled workbook uses `150000` $/MW. See [Network Data Reference](../database/Network_Data_Reference.md#dc_line-asynchronous-dc-vft-ties) and [GTEP Transmission Expansion](../models/GTEP/GTEP_Transmission_Expansion.md#asynchronous-dc-tie-modeling-b-theta).

!!! note "Other transmission parameters are global"
    `transmission_expansion_limit_value`, `transmission_investment_CRP_value`, `enforce_transmission_loss_flag`, `transmission_loss_percent_value`, `transmission_route_length_adder_value`, and `transmission_FOM_percent_value` are set once per run on the `Planning Design` sheet. Only the two costs above are per case. `transmission_route_length_adder_value` and `transmission_FOM_percent_value` are optional rows that the bundled workbook does not include; they default to `1.0` and `0` when absent; see the [Simulation Setting File Reference](./ALEAF_Simulation_Setting_File.md#transmission_route_length_adder_value).

### Per-product operating-reserve flags

Four per-case flags switch individual operating-reserve products on or off in the expansion and operation models.

| Flag | Reserve products covered |
|---|---|
| `regulation_reserve_flag` | Regulation up and regulation down |
| `spinning_reserve_flag` | Spinning |
| `flexibility_reserve_flag` | Flexibility up and flexibility down |
| `nonspin_reserve_flag` | Non-spinning |

Each flag defaults to `TRUE` when its row is absent, so workbooks without these rows keep all products enabled. When a flag is `FALSE`, the model has no provision variables, zonal requirements, or cost terms for that product, and result files report `0` for it.

!!! warning "Scope: expansion and operation only"
    These flags apply only to the expansion and operation models. The reliability assessment model has no operating reserves and is unaffected.

### `demand_reserve_provision_fraction`

Allows flexible demand to supply part of the contingency reserve requirement. The demand-side contribution in each zone and hour is limited to this fraction of the combined spinning plus non-spinning requirement. The value is dimensionless, between `0` and `1`. A value of `0` (also the default when the row is absent) disables demand-side reserve provision. The bundled workbook uses `0.3`.

## 4. Stochastic expansion settings

These settings apply only when stochastic expansion is enabled.

| Setting | Meaning |
|---|---|
| `Stochastic_Expansion_Flag` | Turns stochastic expansion on or off. |
| `Stochastic_File_ID` | Selects the stochastic scenario file bundle. |
| `Stochastic_Load_Flag` | Treats load as stochastic. |
| `Stochastic_Wind_Ons_Flag` | Treats onshore wind as stochastic. |
| `Stochastic_PV_Flag` | Treats PV as stochastic. |
| `Num_Sto_Scenarios` | Number of stochastic scenarios (integer). |

## 5. Operation model settings

| Setting | Meaning | Values / units |
|---|---|---|
| `Run_operation_flag` | Runs the operation (production-cost) model. | `TRUE` / `FALSE` |
| `Use_predefined_expansion_data_for_OP_flag` | Starts operation from a saved expansion result instead of the expansion run in the same case. | `TRUE` / `FALSE` |
| `predefined_expansion_data_file_name_for_OP` | Saved expansion-result file used when the flag above is on, for example `GTEP_multi_round_info.json`. | File name |
| `Dispatch_Mode_in_OP` | Operation dispatch formulation. | `Economic Dispatch`, `Unit Commitment` |
| `NDAY_Groups_OP` | Number of representative-day groups in operation. | Integer |
| `NDAYS_in_Single_Group_OP` | Days per group in operation. | Integer |
| `Load_Shed_in_OP_Flag` | Allows unserved energy in operation. | `TRUE` / `FALSE` |
| `OR_Shortage_in_OP_Flag` | Allows reserve shortfalls in operation. | `TRUE` / `FALSE` |

## 6. Reliability assessment settings

### `Run_RA_flag`

Runs the reliability assessment model. How the simulation itself is executed is controlled by the `RA Setting` and `RA Scenarios` sheets.

### `Use_predefined_expansion_data_for_RA_flag`

Uses a saved expansion result as the system state under study instead of the expansion run in the same case. It is the RA counterpart of `Use_predefined_expansion_data_for_OP_flag`.

### `predefined_expansion_data_file_name_for_RA`

Saved expansion-result file used when the flag above is on.

### Other RA case settings

| Setting | Meaning |
|---|---|
| `update_CAPCRED_in_each_round_of_Expansion_Flag` | Updates capacity-credit values in each expansion round. Relevant when expansion and RA are linked in a multi-round workflow. |
| `NDAY_Groups_RA` | Number of representative-day groups in the RA model. |
| `NDAYS_in_Single_Group_RA` | Days per group in the RA model. |

## 7. Planning design overrides

### Load growth

| Setting | Meaning |
|---|---|
| `load_increase_rate_mode` | How annual load growth is built. Accepts `systemwide` or `regional` (case-insensitive); any other value stops the run with an error. |
| `load_increase_rate_file_ID` | Used only with `regional`. Identifies the load-growth file. |
| `load_increase_rate_value` | Used only with `systemwide`. Annual growth rate, for example `0.02` for 2 percent per year. |

- `systemwide`: every load region receives the same growth factor `(1 + load_increase_rate_value) ^ (year - base_year_value)`. Growth compounds from `base_year_value` (`Planning Design` sheet); it is not a flat annual increment.
- `regional`: growth factors are read per year and region from the CSV `timeseries_data_files/0_additional_scenarios/Load/timeseries_load_growth_<ID>.csv` (or `timeseries_load_growth_rate_<ID>.csv` if the first does not exist). The file needs `Year` and `Region` columns and either a `Load Growth Factor` or a `Load Growth Rate` column. An empty ID or a missing file stops the run with an error.

### Planning reserve margin

| Setting | Meaning |
|---|---|
| `enforce_min_reserve_margin_flag` | Enforces the planning reserve margin (PRM) constraint. When `FALSE`, the margin is reported but not enforced. |
| `planning_reserve_margin_type` | `minimum`: credited capacity must be at least `(1 + margin)` times peak demand. `maximum`: credited capacity is capped at that multiple. |
| `planning_reserve_margin_value` | Margin as a fraction of peak demand (for example `0.13` for 13 percent). |

!!! note "Coincident peak demand basis"
    For each planning-reserve zone, credited capacity (capacity times capacity credit) is compared with the zone's coincident peak demand. Regional hourly load shapes are summed first, per-region load growth is applied, and the annual maximum is taken from the combined 8760-hour series (not from the selected representative days). The coincident peak is never larger than the sum of the regional peaks, so a `minimum` requirement is less conservative than one based on summed regional peaks, and a `maximum` requirement is tighter. The `PRM` column in the GTEP and Operation system-summary outputs uses the same basis.

## 8. Storage operation

| Setting | Meaning |
|---|---|
| `contingency_reserve_min_duration_value` | Minimum duration, in hours, for which a resource must sustain contingency reserve. |
| `storage initialization option` | Initial state of charge of storage at the start of each modeled period: `Minimum`, `Middle`, or `Maximum` (case-sensitive). |

## 9. Financial options and linked tables

### `Fuel_ID`

Selects the fuel-price time series for the case.

- `Base` uses the file given by `timeseries_data_fuel_price_path` on the `File Path` sheet (for the North America database, `timeseries_data_files/Fuel/timeseries_fuel_price.csv`).
- Any other value selects `timeseries_data_files/0_additional_scenarios/Fuel/timeseries_fuel_price_<Fuel_ID>.csv`.

The fuel-price CSV must have the columns `Year, Fuel, Scenario, Region, Type, Unit, Month, Price`. The fuel-type header must be spelled `Fuel` (title case); a file with a `FUEL` header is rejected.

!!! note "Fuel-price files are keyed by region"
    Each row's `Region` must match a network `bus_i` value (for the North America BA-keyed files, for example `ERCO_NCEN_US-TX`). A complete file provides a full grid of region, fuel, year, and month. The region granularity must match `regional_fuel_zone_resolution_type` on the `Network Setting` sheet; see [Network Configuration Reference](../database/Network_Configuration_Reference.md#fuel-zone-resolution-must-match-the-fuel-file).

The default North America `timeseries_fuel_price.csv` is the AEO2026 Counterfactual Baseline projection (335 regions, years 2025-2050). The AEO2026 High and Low Oil-and-Gas-Supply variants are provided as `timeseries_fuel_price_AEO2026_High_Oil_and_Gas_Supply.csv` and `timeseries_fuel_price_AEO2026_Low_Oil_and_Gas_Supply.csv` under `0_additional_scenarios/Fuel/` and are selected through `Fuel_ID`.

### Technology cost selectors

| Setting | Meaning |
|---|---|
| `ESGC_Setting_ID` | Selects the scenario of the `Storage Cost and Performance` sheet (see [Storage cost and performance](./Policy_and_Financial_Settings.md#storage-cost-and-performance)). |
| `ATB_Year` | Year of the technology-cost dataset used for technology assumptions. |
| `ATB_Setting_ID` | Selects the mapping in the `ATB Setting` sheet that assigns cost and performance assumptions to technologies. |

## 10. Market parameters

Penalty prices are in $/MWh (VOLL) or $/MW (reserve shortfalls) and apply only when the corresponding shortfall option is enabled.

| Setting | Meaning |
|---|---|
| `VOLL` | Value of lost load: price of unserved energy in the operation and RA objectives. |
| `RegRSP` | Shortage penalty for regulation reserve. |
| `SRSP` | Shortage penalty for spinning reserve. |
| `NSRSP` | Shortage penalty for non-spinning reserve. |
| `FLEXRSP` | Shortage penalty for flexibility reserve. |

## 11. Reliability and scarcity caps in expansion

Optional annual adequacy limits in expansion planning. Each cap has an on/off flag and a limit value.

| Flag | Value | Limit |
|---|---|---|
| `Total_ENS_MWh_Cap_Flag` | `Total_ENS_MWh` | Annual total energy not served (MWh). |
| `ENS_Hours_Cap_Flag` | `ENS_Hours` | Number of hours with energy not served. |
| `Max_ENS_MWh_Cap_Flag` | `Max_ENS_MWh` | Largest energy not served in a single period (MWh). |

## 12. Policy and regulation

The behavior of these settings is described in [Policy and Financial Settings](./Policy_and_Financial_Settings.md).

| Setting | Meaning |
|---|---|
| `ITC_Flag` | Enables investment tax credits. |
| `PTC_Flag` | Enables production tax credits. |
| `CTAX` | Carbon tax applied to emissions from generation. `0` means no tax. |
| `RPS_Flag` | Enforces the renewable portfolio standard. |
| `Allow_Alternative_RPS_Compliance_Flag` | Makes the RPS a soft requirement, with shortfalls priced at `RPS_Penalty`. |
| `RPS_Penalty` | Price of an RPS shortfall when compliance is soft. |
| `RPS_Global_Target_Value` | System-wide RPS target (fraction of energy). |
| `Clean_Energy_Generation_Target_EXP_Flag` | Enforces the clean-energy generation target in expansion. |
| `Clean_Energy_Generation_Target_EXP_Type` | `Annual` (target met over the full year) or `Daygroup` (target met within each representative-day group). Case-sensitive. |
| `Clean_Energy_Generation_Target_OP_Flag` | Enforces the clean-energy generation target in operation. |
| `Clean_Energy_Generation_Target_Start_Year` | First year in which the target applies. |
| `Clean_Energy_Generation_Global_Target_Value` | System-wide clean-energy target (fraction of energy). |
| `Clean_Energy_Generation_Penalty` | Price of a clean-energy target shortfall when the target is soft. |
| `Carbon_Emission_Reduction_Target_Flag` | Enforces the carbon emission reduction target. |
| `Carbon_Emission_Reduction_Global_Target_Value` | System-wide reduction relative to the reference emission level (fraction). |
| `Carbon_Emission_Reduction_Target_Start_Year` | First year in which the target applies. |

## 13. Regional resource limits

| Setting | Meaning |
|---|---|
| `Regional_resource_limits_flag` | Enforces regional resource limits (supply-curve based limits on new capacity). |
| `Resource_limit_level_value` | Which limit column is used. `Low` selects the low-capacity limits; any other value, including `High`, selects the high-capacity limits. Case-sensitive. |
| `Regional_CAPAX_scaling_flag` | Applies regional scaling to capacity-expansion limits. |

## 14. Time-series data selectors

| Setting | Meaning |
|---|---|
| `Load_File_ID` | Load time series variant. |
| `Wind_Ons_File_ID` | Onshore wind time series variant. |
| `PV_File_ID` | PV time series variant. |
| `Temperature_File_ID` | Temperature time series variant. |

These selectors change operational and reliability results without changing the network data.

## 15. Hydro options

| Setting | Meaning |
|---|---|
| `Hydro_Budget_File_ID` | Hydro energy-budget input variant. |
| `Hydro_Value_File_ID` | Hydro water-value input variant. |
| `Hydro_Budget_Flag` | Applies hydro energy-budget constraints. |
| `Hydro_Flexibility_Flag` | Allows hourly hydro output to deviate from its availability profile by plus or minus `Hydro_Flexibility_Percent`. |
| `Hydro_Flexibility_Percent` | Allowed deviation as a fraction of the profile-based output (for example `0.1` for 10 percent). |
| `Reservoir_Hydro_Operation_Option` | `Budget (day groups)` (budget enforced within each representative-day group) or `Budget (annual)` (budget enforced over the full year). |

## 16. Other case-level settings

| Setting | Meaning |
|---|---|
| `Energy_Storage_AET_Limit_Flag` | Enforces annual energy-throughput (`AET`) limits on storage. |
| `External_Constraints_ID` | Selects the constraint set from the `External Constraints` sheet that applies to the case. `NA` applies none. |
| `HFREQ` | Time-step length in hours used in ramping, storage state transitions, and other intertemporal constraints. `1` means hourly. |
| `FIVEMIN` | Set to `1` to enable 5-minute treatment in the expansion model where supported; `0` disables. |
| `FIVEMIN_OP` | Set to `1` to enable 5-minute treatment in the operation model; `0` disables. |
| `PD` | Total system load in MW, computed by A-LEAF at run time as the sum of bus loads. Leave at `0`; it is informational and is not an input to any constraint. |

## How this sheet connects to other sheets

`Simulation Configuration` links the case to the rest of the workbook: `Simulation Setting`, `Planning Design`, `RA Setting`, `RA Scenarios`, `Scenario Reduction Setting`, `ATB Setting`, `Storage Cost and Performance`, `ITC`, `PTC`, and `External Constraints`.

## Checklist for a new case

1. Set `Run_Flag` and `Case_ID`.
2. Choose which model families run and, for chained runs, whether saved expansion results are used.
3. Confirm the representative-day settings for each active model.
4. Check the data selectors for load, fuel, wind, PV, hydro, and the technology-cost settings.
5. Review reserve, penalty, and policy settings.
6. For RA cases, confirm that the case-level RA settings agree with the `RA Setting` sheet.

## Related documentation

- [ALEAF Simulation Setting File Reference](./ALEAF_Simulation_Setting_File.md)
- [Policy and Financial Settings](./Policy_and_Financial_Settings.md)
- [Scenario Reduction and Representative-Day Groups](./Scenario_Reduction_and_Repday_Groups.md)
- [A-LEAF Documentation](../README.md)
- [RA Overview](../models/RA/RA_Overview.md)
