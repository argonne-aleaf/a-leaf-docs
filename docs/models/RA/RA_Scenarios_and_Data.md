# RA Scenarios and Data

The Reliability Assessment (RA) model evaluates adequacy over a set of joint scenarios, each combining a sampled generator outage pattern with a renewable availability scenario. This page describes where scenario inputs come from, how they are combined and screened, and how to add scenarios. Setting names are defined in [RA Settings and Scenarios Reference](./RA_Settings_and_Scenarios_Reference.md).

## Scenario inputs at a glance

| Input | Source | Controlled by |
|---|---|---|
| Generation mix and year | Expansion results of the selected round, or the case's existing fleet | `round_id_to_run_RA` |
| Outage samples | Sampled by the model from temperature-dependent forced-outage probabilities | `num_risk_scenario`, `reference_temp`, `repair_time_average_hours`, `repair_time_bound` |
| Outage statistics | `RA_data.xlsx` in the case data folder (for example `data/NorthAmerica/RA_data.xlsx`) | Fixed data file |
| Renewable scenarios | `RA Scenarios` sheet and the timeseries files it references | `Scenario_ID`, `Enabled`, `Weight`, `Wind_Ons_File_ID`, `PV_File_ID` |
| Load stress | Case load data | `system_peak_scale` |
| Representative days | Day groups defined for the RA run (`NDAY_Groups_RA` in the `Simulation Configuration` sheet) | Scenario reduction settings |

## Outage samples
For each representative-day group, the model draws `num_risk_scenario` outage samples. Each sample is a set of hourly availability states for every generator. Outage probabilities depend on local temperature: they rise as temperature moves away from `reference_temp`, following technology-specific regression curves. Repair durations are sampled with a mean of `repair_time_average_hours` and truncated at the statistic chosen by `repair_time_bound`.

Outage samples are generated once and reused across renewable scenarios, so outage sampling is independent of renewable scenario configuration.

### The `RA_data.xlsx` workbook
`RA_data.xlsx` supplies the statistical data behind outage sampling. It contains three sheets:

| Sheet | Content |
|---|---|
| `RegressionParametersAD` | Regression coefficients (rows `Hi` and `Lo`, one column per technology: `GasCCAD`, `GasSTAD`, `GasCTAD`, `CoalSTAD`, `NuclearAD`) for the temperature-dependent probability of an outage beginning, on the high and low temperature side of `reference_temp`. |
| `RegressionParametersDD` | The corresponding coefficients (`GasCCDD`, `GasSTDD`, `GasCTDD`, `CoalSTDD`, `NuclearDD`) for outage duration, together with the repair-time statistics (average, 95th percentile, 90th percentile) used by `repair_time_bound`. |
| `OutageDuration` | Historical repair-duration data by technology, with the summary statistics computed from it. |

The file must be in the case data folder.

### Hybrid assets are treated as one unit
A hybrid asset is a co-located renewable plus battery pair. After outage sampling, the two components are coupled: in any hour when either the renewable or the battery is on outage, the whole asset is offline. This applies consistently to scenario screening, dispatch, and all reliability metrics.

- Scenario screening is slightly more conservative, because a single-component failure counts the whole asset's firm capacity as lost.
- In such an hour the renewable output is zero and the battery holds a constant state of charge.
- Metric definitions are unchanged; only the effective availability of the hybrid asset changes.

## Renewable scenarios
Renewable uncertainty is defined in the `RA Scenarios` sheet. Each enabled row is one discrete scenario with an identifier, a weight, and file identifiers for onshore wind and utility-scale PV timeseries. Scenarios apply to `wind_ons` and `pv`; other technologies use their base profiles. Annual profiles are read for each scenario and mapped to the representative-day groups before screening and simulation.

### Zonal profiles for `LOCAL` units
A generator whose `Timeseries_Tag` is `LOCAL` does not use a single sub-region profile. It uses the zone-average profile of the resource type (hydro, CSP, onshore wind, offshore wind, rooftop PV, or PV) in its bus. When a renewable scenario substitutes wind or PV profiles, the zone average is recomputed for that scenario.

The average is taken over the resource-bearing sub-regions of the bus only. A sub-region is resource-bearing for a shape if its profile is non-zero in at least one representative-day hour. Sub-regions with no resource in the whole dataset, or absent from the profile data, are excluded. Hours in which a resource-bearing sub-region produces nothing (wind below cut-in, PV at night) still count in the average, so the zone's availability is not overstated during the low-renewable conditions that drive adequacy risk. Screening and dispatch use the same averaging.

## Joint scenarios
A joint scenario pairs one outage sample with one renewable scenario. It is the unit of screening, simulation, and metric aggregation. Joint scenarios matter because low-renewable conditions can create risk even when the outage pattern is unchanged.

- Space size: `num_risk_scenario` times the number of enabled renewable scenarios, per day group.
- Weight: the renewable scenario's normalized weight divided by `num_risk_scenario`.

### Screening
When `risk_filtering_flag` is `true`, the model screens joint scenarios using `risk_tol_value` and `min_num_risk_in_each_day_value` (see [`risk_tol_value`](./RA_Settings_and_Scenarios_Reference.md#risk_tol_value)). The renewable contribution to each score depends on the renewable scenario of the joint scenario. Scenarios that are screened out are not simulated and contribute zero to expected-value metrics. The screened list is the workload that RA simulates; its size is reported in the run log and, per day group, in the metrics file as `Scenarios_Sampled` and `Scenarios_Solved`.

## Adding a renewable scenario
1. Prepare an hourly onshore wind file and an hourly PV file for the scenario, in the same format as the base timeseries files.
2. Save them in the case data folder as
   `timeseries_data_files/0_additional_scenarios/WIND/timeseries_wind_ons_hourly_<ID>.csv` and
   `timeseries_data_files/0_additional_scenarios/PV/timeseries_pv_hourly_<ID>.csv`.
3. In the `RA Scenarios` sheet, add a row with a unique `Scenario_ID`, `Enabled` set to `true`, a `Weight`, and the file identifiers `Wind_Ons_File_ID` and `PV_File_ID` (the `<ID>` parts of the file names). The two files can use different identifiers.
4. Keep the `BASE` row. It uses the default timeseries paths from the `File Path` sheet.
5. Run RA. The joint scenario space grows by a factor equal to the number of enabled scenarios, so consider lowering `num_risk_scenario` or using risk filtering to control run time.

!!! note
    Weights are relative. They are re-normalized across enabled rows, so they do not need to sum to 1. Disabling a row raises the effective weight of the others.

## Reusing sampled scenarios
A run can reuse a stored outage sample set and its screened joint scenario list, so that repeated runs (for example, comparing generation mixes) use identical scenarios. The stored data must contain both the outage samples and the screened joint scenario list. The run then reproduces both the outage states and the retained workload.

## Modeling assumptions
- Outage samples are generated independently of renewable scenarios.
- Renewable scenarios are discrete weighted cases, not a continuous stochastic process.
- Screened-out joint scenarios contribute zero to expected-value metrics.
- Representative-day weights are preserved when scenarios are mapped and when metrics are aggregated.

## Related documentation
- [RA Overview](./RA_Overview.md)
- [RA Formulation](./RA_Formulation.md): equation-level treatment of scenario generation and screening
- [RA Settings and Scenarios Reference](./RA_Settings_and_Scenarios_Reference.md): the `RA Setting` and `RA Scenarios` sheet columns
- [RA Execution and Results](./RA_Execution_and_Results.md)
