# Scenario Reduction and Representative-Day Groups

Solving a full year of hourly data for every study year is usually too large for capacity expansion, and often for operation and reliability studies as well. A-LEAF therefore reduces the full-year load, wind, and solar time series to a small set of representative days. This page explains what the reduction produces, how to control it, and how to read the resulting calendar in the output files.

The same reduced calendar is shared across the framework:

- the expansion (GTEP) model uses representative days to keep the optimization tractable;
- the operation model uses representative days and day groups for dispatch and reporting;
- the reliability assessment (RA) model reuses the calendar and maps its uncertainty inputs onto it.

## Representative days and day groups

The reduced calendar has two layers:

- **Representative days** are the individual days that are actually modeled.
- **Day groups** collect one or more representative days into a logical block. A group of one day is a single independent day; a group of several days is a run of consecutive calendar days (for example days 4, 5, and 6) that is modeled chronologically, so that storage, ramping, and other intertemporal constraints link the days in the block.

The size of the calendar is set per model in `Simulation Configuration`: `NDAY_Groups` and `NDAYS_in_Single_Group` for expansion, `NDAY_Groups_OP` and `NDAYS_in_Single_Group_OP` for operation, and `NDAY_Groups_RA` and `NDAYS_in_Single_Group_RA` for RA. A case with 3 groups of 1 day models 3 representative days; 4 groups of 3 days models 12 representative days in 4 chronological blocks.

## How representative days are obtained

Each model has a selection mode in `Simulation Configuration` (`Repday_Selection_Mode_EXP`, `Repday_Selection_Mode_OP`, `Repday_Selection_Mode_RA`).

**Manual.** A predefined representative-day file (named by `Repday_File_ID`) is loaded directly. This is appropriate when a study needs a fixed calendar, for example to compare cases on identical days. The file must match the requested number of day groups and days per group.

**Scenario Reduction.** A-LEAF selects days from the full-year data. Scenario reduction is also the fallback when the manual file is missing or does not match the requested group structure, so a case still runs, but with a calendar that may differ from the intended fixed one.

### How scenario reduction selects days

Scenario reduction starts from the full-year system load, onshore wind, and PV series (regions are weighted by their original load and installed capacity) and selects the requested number of days or day windows that best represent the year. Each selected group receives a probability reflecting how much of the year it stands for.

The features used for the selection, and the rules for keeping extreme days, are controlled on the `Scenario Reduction Setting` sheet:

| Setting | Purpose |
|---|---|
| `time_resolution` | Time resolution of the reduction. `Hourly` is the default; 5-minute resolution is not supported. |
| `repday_selection_resolution` | Spatial scope of the selection (see the next section). |
| `generate_input_data_flag` | Generates the input matrices for the reduction algorithm. Default `TRUE`. |
| `input_type_load_shape_flag`, `input_type_load_MWh_flag` | Use load shape or load energy (MWh) as a clustering feature. |
| `input_type_wind_shape_flag`, `input_type_wind_MWh_flag` | Use wind shape or wind energy as a clustering feature. |
| `input_type_solar_shape_flag`, `input_type_solar_MWh_flag` | Use solar shape or solar energy as a clustering feature. |
| `input_type_net_load_MWh_flag` | Use net-load energy as a clustering feature. |
| `fixing_extreme_days_flag` | Forces the algorithm to retain extreme days. Default `TRUE`. |
| `fix_peak_demand_day_flag` | Keeps the peak-demand day. |
| `fix_peak_net_demand_day_flag` | Keeps the peak net-demand (load minus VRE) day. |
| `fix_peak_solar_generation_day_flag` | Keeps the day with maximum solar generation. |
| `fix_peak_wind_generation_day_flag` | Keeps the day with maximum wind generation. |
| `fix_least_solar_generation_day_flag` | Keeps the day with minimum solar generation. |
| `fix_least_wind_generation_day_flag` | Keeps the day with minimum wind generation. |
| `allow_repday_overlap_flag` | Allows the same day to appear in more than one group. Default `FALSE`. |
| `preselected_extreme_days_list` | Day-group IDs to force into the selection, for example `[1;2;3]`; `[]` means none. |

The workbook notes that the number of enabled `input_type_*` flags must equal the number of data sets the reduction expects. Retaining extreme days (peak demand, minimum wind, minimum solar) is important for adequacy and scarcity results, because clustering on averages alone tends to drop them.

## Selection resolution vs run resolution

Representative-day selection clusters on a single system-wide aggregate, so the resolution used to select days can be decoupled from the spatial resolution at which the model runs. This keeps pre-processing cheap for large nodal databases and does not change results.

The global key `repday_selection_resolution` in the `Scenario Reduction Setting` sheet controls this. It is not a per-case column. Two values are accepted (case-insensitive):

| Value | Behavior |
|---|---|
| `system` | Builds the selection aggregate as a single system-wide bus over all modeled regions. Cheapest option. |
| `regional` | Builds the selection aggregate at the `network_boundary_type` resolution (for example, one bus per interconnection). |

!!! note "`system` and `regional` produce the same days"
    The selection metric is a system-wide aggregate over the same set of regions for both options, so `system` and `regional` select identical representative days. They differ only in how many small selection buses are built. `regional` is reserved for a possible per-zone clustering; do not choose it expecting a different calendar.

If the key is absent, blank, or unrecognized, selection runs in-line at the run resolution. An unrecognized value writes an informational message to the log and takes the same fallback. Existing setting files that lack the key keep working unchanged.

The setting applies to the operation and expansion models in serial and distributed execution, in single-round, multi-round, and stochastic workflows. It has no effect in `Manual` mode.

!!! note "Recommended value for large nodal databases"
    For large nodal databases, set `repday_selection_resolution` to `system`. It is the cheapest option and gives the same representative days.

## Reading the calendar in the results

### Representative-day fields

Each representative day has these attributes:

| Field | Meaning |
|---|---|
| `Day_Group_ID`, `Day_Group` | The day group the day belongs to. |
| `NumDays_Group` | Number of days of the year the group represents. |
| `NumDays` | Number of days of the year this representative day stands for. It is the weight used for annualized quantities. |
| `Day` | The original day number in the year (1 to 365). |
| `Month` | Month of the original day. |
| `Scenario_ID` | Stochastic scenario, in stochastic workflows only. |

Each day group additionally has `Start_Day_Id`, `End_Day_Id`, `Probability`, and two lists: `Day_List`, the original calendar days it represents, and `Day_Idx_List`, the internal representative-day IDs it contains.

### `Day_of_Year`

The representative-day result files report two identifiers per row:

- `Rep_Day`: the internal representative-day ID;
- `Day_of_Year`: the original full-year day number (the `Day` field).

Use `Day_of_Year` to relate a modeled day back to the annual time series (for example to find its weather or load in the source data). `Rep_Day` only identifies the day inside the reduced model.

For a group of three consecutive days, the result files show:

| `Rep_Day` | `Day_of_Year` | `Day_Group_ID` |
|---|---|---|
| 1 | 4 | same group |
| 2 | 5 | same group |
| 3 | 6 | same group |

Here `Day_List = [4, 5, 6]`, and `Day_Idx_List` holds the three internal IDs.

### Weighting

Selection produces a probability for each day group. A-LEAF converts these probabilities into the day counts `NumDays_Group` and `NumDays`. The representative-day weight `NumDays` then scales objective terms, annualized metrics, policy accounting (targets, penalties, tax credits), and scarcity and energy-not-served reporting. Annual totals in the outputs are therefore estimates of full-year quantities extrapolated from the representative days.

## How each model uses the calendar

- **Expansion (GTEP).** Time series are attached to representative days, costs and policy quantities are weighted by `NumDays`, and multi-day groups keep intertemporal constraints across their days.
- **Operation.** Dispatch and power-flow results map back to day groups, so a multi-day group should be read as one operational block rather than as isolated days.
- **Reliability assessment.** RA reuses the calendar built upstream and does not define its own. Outages are sampled by day group, renewable scenarios are mapped onto the reduced days by `Day_of_Year`, and sequential and joint-scenario workflows run by day group.

## Troubleshooting

When a scenario-reduction run gives unexpected results, check in this order:

1. The selection mode in `Simulation Configuration` (`Manual` or `Scenario Reduction`), and whether a manual file was actually used or the run fell back to scenario reduction.
2. The `Scenario Reduction Setting` sheet: enabled features and extreme-day rules.
3. The representative-day report: `Rep_Day`, `Day_Group_ID`, and `Day_of_Year`.

For grouped-day behavior, confirm that each group covers the intended consecutive days and that `Day_of_Year` matches the original day whose data should be used.

## Related documentation

- [A-LEAF Documentation](../README.md)
- [ALEAF Simulation Setting File Reference](../configuration/ALEAF_Simulation_Setting_File.md)
- [Simulation Configuration Reference](../configuration/Simulation_Configuration_Reference.md)
- [Network Data Reference](../database/Network_Data_Reference.md)
- [RA Overview](../models/RA/RA_Overview.md)
