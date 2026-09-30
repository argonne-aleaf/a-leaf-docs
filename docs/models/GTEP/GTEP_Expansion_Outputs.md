# GTEP Expansion Outputs

This page is a user's guide to the result files written by the GTEP expansion model. For each file it gives the name, what the file contains, the meaning and units of every column, and how to use it. Files are written to the case output directory. Each file name starts with the case identifier `<Case_ID>`.

A typical review starts with the system and technology summaries, then moves to the build decisions (`gen_expansion_EXP` and `line_expansion_EXP`), prices and scarcity, and finally hourly dispatch. See the [Practical reading guide](#practical-reading-guide) at the end of this page.

## Expansion export workflow

After the expansion solve, the model writes:

- an optional JSON snapshot of the solved model;
- a family of CSV reports for generator and transmission builds, unserved energy and reserve shortfalls, power flow, dispatch, market prices, policy slack, demand response, technology and system summaries, and representative days.

## File List

### Optional JSON snapshot
- `ALEAF_LC_GTEP_EXP_<test_system_name>_<Case_ID>_<timestamp>.json`

Written only when `export_model_reference_json_expansion_flag` is `TRUE` in the `Simulation Setting` sheet.

Contents: the expansion model result, the model system reference, the network data and the settings.

This is the largest and most detailed export. It is useful for reproducibility and troubleshooting, but it is not the main reporting format.

### CSV files
- `<Case_ID>__gen_expansion_EXP.csv`
- `<Case_ID>__line_expansion_EXP.csv`
- `<Case_ID>__unserved_energy_EXP.csv`
- `<Case_ID>__reserve_shortfall_EXP.csv`
- `<Case_ID>__power_flow_EXP.csv`
- `<Case_ID>__dispatch_EXP_year_<stage>.csv`
- `<Case_ID>__market_EXP.csv`
- `<Case_ID>__policy_slack_EXP.csv`
- `<Case_ID>__demand_response_EXP.csv`
- `<Case_ID>__tech_summary_by_stage_EXP.csv`
- `<Case_ID>__tech_summary_by_year_EXP.csv`
- `<Case_ID>__system_summary_by_stage_EXP.csv`
- `<Case_ID>__system_summary_by_year_EXP.csv`
- `<Case_ID>__system_summary_by_year_EXP_real_<dollar_year>usd.csv`
- `<Case_ID>__representative_days_EXP.csv`

### Report control flags
Most of the CSV report families above are controlled by boolean flags in the
`Simulation Setting` sheet. All flags default to on: a missing setting, a blank
cell or `TRUE` writes the report, and only an explicit `FALSE` suppresses it. See the
[Simulation Setting File Reference](../../configuration/ALEAF_Simulation_Setting_File.md#report-control-gtep-operation)
for the full description.

| File | Controlling flag |
|---|---|
| `<Case_ID>__gen_expansion_EXP.csv` | `report_expansion_flag` |
| `<Case_ID>__line_expansion_EXP.csv` | `report_expansion_flag` |
| `<Case_ID>__unserved_energy_EXP.csv` | `report_scarcity_EXP_flag` |
| `<Case_ID>__reserve_shortfall_EXP.csv` | `report_scarcity_EXP_flag` |
| `<Case_ID>__power_flow_EXP.csv` | `report_power_flow_EXP_flag` |
| `<Case_ID>__dispatch_EXP_year_<stage>.csv` | `report_dispatch_EXP_flag` |
| `<Case_ID>__market_EXP.csv` | `report_dispatch_EXP_flag` |
| `<Case_ID>__policy_slack_EXP.csv` | `report_dispatch_EXP_flag` |
| `<Case_ID>__demand_response_EXP.csv` | `report_dispatch_EXP_flag` |
| `<Case_ID>__tech_summary_by_stage_EXP.csv` | `report_summary_EXP_flag` |
| `<Case_ID>__tech_summary_by_year_EXP.csv` | `report_summary_EXP_flag` |
| `<Case_ID>__system_summary_by_stage_EXP.csv` | `report_summary_EXP_flag` |
| `<Case_ID>__system_summary_by_year_EXP.csv` | `report_summary_EXP_flag` |
| `<Case_ID>__system_summary_by_year_EXP_real_<dollar_year>usd.csv` | `report_summary_EXP_flag` (derived from the annual file) |
| `<Case_ID>__representative_days_EXP.csv` | always written (not gated) |
| `GTEP_multi_round_info.json` | `report_multi_round_summary_json_EXP_flag` (kept/deleted after the run; see below) |

!!! note "`GTEP_multi_round_info.json` is a working file, not a standard report"
    In multi-round runs the model writes this JSON after every round because it is the
    resume checkpoint and the hand-off from expansion to the RA model and to operation
    runs that reuse a predefined expansion (`predefined_expansion_data_file_name_for_RA`).
    `report_multi_round_summary_json_EXP_flag` (default `TRUE`) only controls whether
    the file is kept after the whole run finishes: with `FALSE` it is deleted at the
    end of the run. See
    [GTEP Planning Horizon and Multi-Round](./GTEP_Planning_Horizon_and_Multi_Round.md#multi-round-output-artifact)
    for its contents and the
    [Simulation Setting File Reference](../../configuration/ALEAF_Simulation_Setting_File.md#report-control-gtep-operation).

!!! note "Multi-round reports"
    In multi-round mode the CSV files above are assembled from per-round pieces
    written to a temporary `_report_parts/round_<r>/` directory. The final file names,
    columns and values are the same as for a single-run expansion, and the report
    settings apply to every round. See
    [GTEP Planning Horizon and Multi-Round](./GTEP_Planning_Horizon_and_Multi_Round.md#how-multi-round-reporting-is-produced).

!!! note "Summary reports do not require the dispatch report"
    The technology and system summaries are computed independently of the dispatch
    files, so `report_dispatch_EXP_flag = FALSE` together with
    `report_summary_EXP_flag = TRUE` still produces the summaries. In multi-round runs,
    each stage's `ObjectiveValue` is the objective of the round that decided that stage.

## Common Field Conventions

### `Case_ID`
The case name.

### `Stage`
The planning-stage id. It is not a calendar year.

### `Year`
The calendar year of the stage, inserted right after `Stage` in the hourly and stage report files: `first_stage_year_value + (Stage - 1) * num_years_per_stage_value`.

### `Start Year` / `Start_Year`
The actual calendar year at which a planning stage begins (`Start_Year` in the by-stage summary files).

### `Number of Years` / `Years_in_Stage`
The stage length represented by that planning stage (`Years_in_Stage` in the by-stage summary files).

### `Rep_Day`
The representative-day index used in the reduced chronology. Its original calendar day is `Day_of_Year` in the `representative_days` file.

### `Hour`
The hour index inside the representative day.

### `Sub_Period`
The intra-hour time-slice index. It is `1` for hourly runs; the column is always written.

### `Stochastic_Scenario_ID`
The stochastic expansion scenario id when stochastic expansion is enabled. Otherwise it is `0`.

### `Days_Represented`
The representative-day weight: the number of full-year days that the reduced day stands for. Multiply hourly quantities by this weight to annualize them.

!!! note "Column names in earlier releases"
    If you have post-processing scripts written for earlier releases, the mapping from old to current file and column names is in the [Output naming history](#output-naming-history) section at the end of this page.

## 1. Generator Expansion Output

### File
- `<Case_ID>__gen_expansion_EXP.csv`

One row per generating unit and planning stage. Use this file to see what is built and retired, where, and when.

### Columns
- `Case_ID`: case identifier.
- `Stage`: planning-stage id.
- `Year`: calendar year of the stage (`first_stage_year_value + (Stage - 1) * num_years_per_stage_value`).
- `Unit_ID`: internal generator index.
- `Plant_Name`: plant name.
- `Unit_Group`: grouped technology label.
- `Unit_Category`: technology category.
- `Bus_Name`: bus label.
- `Unit_Capacity_MW`: size of one unit in MW.
- `New_Units`: number of new generating units added in that stage.
- `Retired_Units`: number of units retired in that stage.
- `Units_In_Service`: total number of units in service in that stage after additions and retirements.
- `New_Capacity_MW`: `New_Units * Unit_Capacity_MW`.
- `Retired_Capacity_MW`: `Retired_Units * Unit_Capacity_MW`.
- `Total_Capacity_MW`: `Units_In_Service * Unit_Capacity_MW`.
- `New_Storage_Unit_Hours`: storage-duration expansion decision for storage technologies, in units times hours of storage duration.
- `Storage_Energy_MWh`: resulting storage energy in MWh carried by the unit in that stage.

## 2. Transmission Expansion Output

### File
- `<Case_ID>__line_expansion_EXP.csv`

One row per branch and planning stage. Use this file to see which corridors are reinforced and by how much. See [GTEP Transmission Expansion](./GTEP_Transmission_Expansion.md) for the modeling approach.

### Columns
- `Case_ID`: case identifier.
- `Line_ID`: internal branch index.
- `Stage`: planning-stage id.
- `Year`: calendar year of the stage (`first_stage_year_value + (Stage - 1) * num_years_per_stage_value`).
- `From_Bus_ID`: from-bus index used in the model.
- `To_Bus_ID`: to-bus index used in the model.
- `From_Bus_Name`: exported bus label for the from side.
- `To_Bus_Name`: exported bus label for the to side.
- `Merged_Line_UIDs`: original line identifiers merged into the modeled branch, if present.
- `Original_Rating_MW`: transmission rating before expansion, in MW.
- `New_Expansion_Fraction`: expansion added on that branch in that stage, as a fraction of the original rating.
- `Cumulative_Expansion_Fraction`: cumulative expansion through that stage, as a fraction of the original rating.
- `Final_Rating_MW`: post-expansion rating, `(1 + New_Expansion_Fraction) * Original_Rating_MW`.
- `Length`: branch length in miles used in transmission investment cost accounting.

!!! warning "Expansion ceilings are formula-based, not physical interface limits"
    The upper limit on expansion is not a calibrated per-corridor interface limit. The
    ceiling is set by the branch `max_rate_a` and by the global setting
    `transmission_expansion_limit_value` (`Planning Design` sheet). With the value `2`, a
    single corridor can grow to three times its base rating, and the model raises
    `max_rate_a` to `rate_a * (1 + transmission_expansion_limit_value)` if the network
    data value is smaller. In the North America network, about 99% of expandable
    branches have `max_rate_a` equal to `3 * rate_a`.

    Read system-wide new-transmission totals as limited by this ceiling relative to the
    base ratings at the modeled resolution, not as calibrated to a transmission planning
    source. Before relying on absolute transmission-build magnitudes, compare the largest
    base ratings with published interface limits.

## 3. Scarcity ENS Output

### File
- `<Case_ID>__unserved_energy_EXP.csv`

Energy not served by bus, representative day and hour. Use this file to locate reliability shortfalls in space and time.

### Columns
- `Case_ID`: case identifier.
- `Stage`: planning-stage id.
- `Year`: calendar year of the stage (`first_stage_year_value + (Stage - 1) * num_years_per_stage_value`).
- `Rep_Day`: representative-day index (its calendar day is `Day_of_Year` in `representative_days_EXP`).
- `Hour`: hour within the representative day.
- `Sub_Period`: time-slice id (`1` for hourly runs).
- `Bus_ID`: bus index.
- `Unserved_Energy_MW`: energy not served at the bus, representative day, hour, sub-period, and stage, in MW.

## 4. Scarcity Reserve Output

### File
- `<Case_ID>__reserve_shortfall_EXP.csv`

Reserve shortfalls by reserve zone, representative day and hour.

### Columns
- `Case_ID`: case identifier.
- `Stage`: planning-stage id.
- `Year`: calendar year of the stage (`first_stage_year_value + (Stage - 1) * num_years_per_stage_value`).
- `Rep_Day`: representative-day index (its calendar day is `Day_of_Year` in `representative_days_EXP`).
- `Hour`: hour within the representative day.
- `Sub_Period`: time-slice id (`1` for hourly runs).
- `Reserve_Zone_ID`: reserve-zone id.
- `Spin_Shortfall_MW`: reserve shortage for contingency or spinning reserve in that reserve zone.
- `Demand_Reserve_MW`: demand-reserve quantity of the zone.
- `NSpin_Shortfall_MW`: reserve shortage for non-spinning reserve.
- `FlexUp_Shortfall_MW`: upward flexible reserve shortage.
- `FlexDown_Shortfall_MW`: downward flexible reserve shortage.

## 5. Power Flow Output

### File
- `<Case_ID>__power_flow_EXP.csv`

Hourly flow, rating, price and congestion information for every branch and representative-day hour. Use it to identify congested corridors and to compare flows with expanded ratings.

### Columns
- `Case_ID`: case identifier.
- `Stage`: planning-stage id.
- `Year`: calendar year of the stage (`first_stage_year_value + (Stage - 1) * num_years_per_stage_value`).
- `Rep_Day`: representative-day index (its calendar day is `Day_of_Year` in `representative_days_EXP`).
- `Hour`: hour within the representative day.
- `Sub_Period`: time-slice id (`1` for hourly runs).
- `Line_ID`: internal branch index.
- `From_Bus_ID`: from-bus index.
- `To_Bus_ID`: to-bus index.
- `From_Region`: label of the from side (the bus label).
- `To_Region`: label of the to side (the bus label).
- `Original_Rating_MW`: original branch rating before expansion, in MW.
- `Cumulative_Expansion_Fraction`: cumulative expansion of that branch through the stage, as a fraction of the original rating.
- `Final_Rating_MW`: expanded branch capacity in MW after applying the expansion.
- `Flow_MW`: dispatched branch flow in MW. In the enhanced-hybrid formulation this is the base (fixed-susceptance) flow; see `Total_Flow_MW`.
- `LMP_From_Bus_USD_per_MWh`: nodal price at the from bus.
- `LMP_To_Bus_USD_per_MWh`: nodal price at the to bus.
- `Congested_Flag`: indicator showing whether the line is congested.
- `Wheeling_Cost`: congestion price difference between the two ends of the branch, expressed as a wheeling-cost style value.
- `Line_Length`: branch length.
- `Days_Represented`: representative-day weight.
- `Expansion_Flow_MW`: expansion-increment flow in MW under the enhanced-hybrid formulation, that is the extra flow the built headroom carries beyond the base flow `Flow_MW`. It is `0` for corridors without the hybrid treatment (DC ties, or whenever `transmission_expansion_hybrid_flag` is off or the mode is not `B-theta`).
- `Total_Flow_MW`: the true corridor flow in MW, `Flow_MW + Expansion_Flow_MW`. It equals `Flow_MW` for corridors without the hybrid treatment.
- `KVL_Residual_MW`: the relaxation gap of the enhanced-hybrid formulation, `Expansion_Flow_MW − u * Flow_MW` in MW, where `u` is the cumulative expansion behind `Final_Rating_MW`. It is `0` for corridors without the hybrid treatment. On hybrid corridors it is `0` for an unbuilt line and for a fully built line and largest at partial builds. It shows how far the relaxation is from the exact flow of the expanded line.

!!! note "`Expansion_Flow_MW`, `Total_Flow_MW` and `KVL_Residual_MW` are audit columns for the enhanced hybrid"
    These three trailing columns carry information only when the optional enhanced-hybrid
    `B-theta` formulation (`transmission_expansion_hybrid_flag`) is active. In every other
    run, `Expansion_Flow_MW = 0`, `Total_Flow_MW = Flow_MW` and `KVL_Residual_MW = 0`. See
    [GTEP Transmission Expansion](./GTEP_Transmission_Expansion.md#enhanced-hybrid-b-theta-transmission-expansion)
    for the formulation and
    [Simulation Setting File Reference](../../configuration/ALEAF_Simulation_Setting_File.md#transmission_expansion_hybrid_flag)
    for the flag.

!!! note "No per-branch loss column"
    The power flow file has no per-branch loss column, because branch flows are lossless.
    Losses are represented only by the flat, system-wide loss percentage
    (`transmission_loss_percent_value`, applied when `enforce_transmission_loss_flag` is
    `TRUE`), reported as `TnD_Loss` in the system summary. See
    [Simulation Setting File Reference](../../configuration/ALEAF_Simulation_Setting_File.md#transmission_loss_percent_value).

## 6. Dispatch Output

### File pattern
- `<Case_ID>__dispatch_EXP_year_<stage>.csv`

One dispatch CSV is written per planning stage. Each contains hourly generator-level results for the representative days of that stage: output, reserve provision, curtailment, storage operation and fuel use.

### Columns
- `Case_ID`: case identifier.
- `Stage`: planning-stage id.
- `Year`: calendar year of the stage (`first_stage_year_value + (Stage - 1) * num_years_per_stage_value`).
- `Stochastic_Scenario_ID`: stochastic expansion scenario id, or `0`.
- `Rep_Day`: representative-day index (its calendar day is `Day_of_Year` in `representative_days_EXP`).
- `Hour`: hour within the representative day.
- `Sub_Period`: time-slice id (`1` for hourly runs).
- `Unit_ID`: internal generator index.
- `Plant_Name`: generator name used for reporting.
- `Bus_ID`: modeled bus index.
- `Bus_Name`: bus label.
- `Parent_Bus_Name`: parent or higher-level bus label from the region configuration.
- `Region_Name`: configured region name for the bus.
- `Tech_ID`: technology id for the unit.
- `Unit_Group`: grouped technology label such as `wind_ons`, `pv`, `gas_cc`, or storage groups.
- `Unit_Category`: broad category such as thermal, renewable, or storage.
- `Unit_Report_Label_1`: free-text reporting label (first slot).
- `Unit_Report_Label_2`: free-text reporting label (second slot).
- `Units_In_Service`: total unit count in service in that stage.
- `Storage_Energy_MWh`: storage energy-duration quantity for storage technologies.
- `Installed_Capacity_MW`: installed capacity in MW for the reporting row.
- `Storage_Charging_Flag`: commitment-state style storage status variable when present.
- `Units_Committed`: unit commitment on/off state when commitment is modeled.
- `Units_Started`: startup indicator when commitment is modeled.
- `Generation_MW`: generation output in MW.
- `Reserve_RegUp_MW`: regulation-up provision.
- `Reserve_RegDn_MW`: regulation-down provision.
- `Reserve_Spin_MW`: spinning reserve provision.
- `Reserve_NSpin_MW`: non-spinning reserve provision.
- `Reserve_FlexUp_MW`: upward flexible reserve provision.
- `Reserve_FlexDn_MW`: downward flexible reserve provision.
- `Curtailment_MW`: curtailment output.
- `Charge_MW`: storage charging power.
- `SOC_MWh`: storage state of charge.
- `Hybrid_Charge_MW`: charging or energy-transfer term for hybrid resources.
- `Hybrid_Type`: hybrid resource label used by the generator index.
- `Fuel`: fuel label.
- `Fuel_Consumption_MMBtu`: implied fuel use from dispatched generation and heat rate.
- `Fuel_Cost_USD`: fuel-cost component attributed to the row.
- `Inertia`: inertia contribution based on the unit’s online state and configured inertia constant.
- `Marginal_Cost_USD_per_MWh`: marginal-cost style value used for the generator row.
- `Days_Represented`: representative-day weight.
- `VRE_Capacity_Factor`: the variable-renewable availability profile behind `Curtailment_MW`, so reported curtailment can be checked against the input profile.

## 7. Market Output

### File
- `<Case_ID>__market_EXP.csv`

Hourly load, scarcity indicators and prices by bus. Use it for locational marginal prices (LMPs), reserve prices and the timing of unserved energy.

### Columns
- `Case_ID`: case identifier.
- `Stage`: planning-stage id.
- `Year`: calendar year of the stage (`first_stage_year_value + (Stage - 1) * num_years_per_stage_value`).
- `Stochastic_Scenario_ID`: stochastic expansion scenario id, or `0`.
- `Rep_Day`: representative-day index (its calendar day is `Day_of_Year` in `representative_days_EXP`).
- `Hour`: hour within the representative day.
- `Sub_Period`: time-slice id (`1` for hourly runs).
- `Bus_ID`: modeled bus index.
- `Bus_Name`: bus label.
- `Parent_Bus_Name`: parent or higher-level bus label.
- `Region_Name`: configured region name.
- `Load_MW`: represented load at the bus and hour.
- `Unserved_Energy_MW`: energy-scarcity quantity at that bus and hour.
- `Spin_Shortfall_MW`: spinning-reserve scarcity quantity attached to the bus.
- `Demand_Reserve_MW`: demand-reserve quantity.
- `NSpin_Shortfall_MW`: non-spinning reserve scarcity quantity attached to the bus.
- `FlexUp_Shortfall_MW`: upward flexible reserve scarcity quantity attached to the bus.
- `FlexDown_Shortfall_MW`: downward flexible reserve scarcity quantity attached to the bus.
- `LMP_USD_per_MWh`: locational marginal price.
- `Price_RegUp_USD_per_MW`: regulation-up reserve clearing price.
- `Price_RegDn_USD_per_MW`: regulation-down reserve clearing price.
- `Price_Spin_USD_per_MW`: spinning reserve clearing price.
- `Price_NSpin_USD_per_MW`: non-spinning reserve clearing price.
- `Price_FlexUp_USD_per_MW`: upward flexible reserve clearing price.
- `Price_FlexDn_USD_per_MW`: downward flexible reserve clearing price.
- `Days_Represented`: representative-day weight.
- `ENS_Flag`: indicator that the bus-hour experienced ENS.

!!! note "Prices under the cuOpt GPU solver"
    The price columns (`LMP_USD_per_MWh`, `Price_RegUp_USD_per_MW`, `Price_RegDn_USD_per_MW`, `Price_Spin_USD_per_MW`, `Price_NSpin_USD_per_MW`,
    `Price_FlexUp_USD_per_MW`, `Price_FlexDn_USD_per_MW`) are constraint dual values. They
    are also available when `solver_name = cuOpt`. Two caveats apply. First, cuOpt duals
    come from PDLP, a first-order method, so the prices are **approximate** relative to a
    simplex solve. Validate against a run with another solver such as HiGHS, or tighten
    the `cuOpt Setting` tolerances, if price accuracy matters. Second, duals exist only
    for **pure-LP** models, so any integer build decision must be relaxed
    (`Integrality = FALSE`) for a priced cuOpt run. See
    [GPU Solvers](../../configuration/GPU_Solvers.md) for setup and tolerance options.

## 8. Policy Output

### File
- `<Case_ID>__policy_slack_EXP.csv`

Slack in the clean-energy and renewable portfolio targets. A nonzero slack means the target was not fully met and the corresponding penalty was paid.

### Columns
- `Case_ID`: case identifier.
- `Stage`: planning-stage id.
- `Year`: calendar year of the stage (`first_stage_year_value + (Stage - 1) * num_years_per_stage_value`).
- `Rep_Day`: representative-day index (its calendar day is `Day_of_Year` in `representative_days_EXP`).
- `Policy_Zone_ID`: modeled bus or policy-zone id.
- `Clean_Energy_Target_Slack`: clean-energy-generation-target slack quantity.
- `RPS_Target_Slack`: renewable portfolio standard slack quantity.

## 9. Demand Response Output

### File
- `<Case_ID>__demand_response_EXP.csv`

Hourly results for demand-response and large flexible load resources, including their onsite generation and storage. See [Large Load and Demand Response Reference](../../database/Large_Load_and_Demand_Response_Reference.md) for how these resources are defined.

### Columns
- `Case_ID`: case identifier.
- `Stage`: planning-stage id.
- `Year`: calendar year of the stage (`first_stage_year_value + (Stage - 1) * num_years_per_stage_value`).
- `Rep_Day`: representative-day index (its calendar day is `Day_of_Year` in `representative_days_EXP`).
- `Hour`: hour within the representative day.
- `Sub_Period`: time-slice id (`1` for hourly runs).
- `Plant_Name`: demand response resource name.
- `Bus_ID`: bus index.
- `Bus_Name`: bus label.
- `Region_Name`: configured region name.
- `Parent_Bus_Name`: parent or higher-level bus label.
- `Online_Year`: online year for the demand response resource.
- `Unit_Group`: demand-response group label.
- `Unit_Category`: category label.
- `Unit_Report_Label_1`: free-text reporting label (first slot).
- `Unit_Report_Label_2`: free-text reporting label (second slot).
- `Unit_Capacity_MW`: MW capacity.
- `Interconnection_Limit_MW`: interconnection limit.
- `Integer_Flag`: whether the demand-response resource is modeled with integer structure.
- `Daily_DR_Limit_MWh`: daily energy limit for the resource.
- `Num_DR_Segments`: number of demand-response segments.
- `Pct_MW_1` to `Pct_MW_5`: MW size of each demand-response segment.
- `Price_1` to `Price_5`: segment prices after economic-base conversion.
- `Hybrid_Gen`: hybrid generation linkage flag or identifier.
- `Hybrid_Gen_CAP`: linked hybrid generation capacity.
- `Hybrid_ES`: linked hybrid storage flag or identifier.
- `Hybrid_ES_CAP`: linked hybrid storage capacity.
- `LFL_Load_MW`: baseline large flexible load level.
- `LFL_DR_MW`: dispatched demand-response amount.
- `LFL_DR_Segment_1_MW` to `LFL_DR_Segment_5_MW`: segment-level dispatched DR quantities (MW).
- `LFL_DR_Segment_1_Active` to `LFL_DR_Segment_5_Active`: segment activation indicators.
- `LFL_Gen_to_Load_MW`: onsite generation serving the flexible load directly.
- `LFL_Gen_to_Grid_MW`: onsite generation exported to the grid.
- `LFL_Gen_to_Storage_MW`: onsite generation charging the onsite storage.
- `LFL_Storage_to_Load_MW`: storage discharge serving the flexible load.
- `LFL_Storage_to_Grid_MW`: storage discharge exported to the grid.
- `LFL_Grid_to_Storage_MW`: grid energy charging the onsite storage.
- `LFL_Storage_SOC_MWh`: flexible-load storage state of charge.

## 10. Technology Summary Output

### Files
- `<Case_ID>__tech_summary_by_stage_EXP.csv`
- `<Case_ID>__tech_summary_by_year_EXP.csv`

Technology-level (unit-level) capacity, generation, cost and revenue results. The planning-stage file reports stage-level totals. The annual file expands stage results into annualized rows. Use these files to compare technologies, regions and units, for example capacity mix by stage or unit profit.

### Planning-stage file columns

!!! note "Noise cleaning"
    Units whose `ICAP`, `ICap_New` and `ICap_Ret` are all below `1e-3` MW are solver noise and are written with every metric column (from `TotalUnits` onward, `CAPCRED` excepted) set to `0`. See [Numerical-noise rule](#numerical-noise-rule).

- `Case_ID`: case identifier.
- `Stage`: planning-stage id.
- `Start_Year`: actual starting calendar year of the stage.
- `Years_in_Stage`: stage length.
- `PLANT_NAME`: plant name.
- `Bus_ID`: bus index.
- `Bus_Name`: bus label.
- `Parent_Bus_Name`: parent or higher-level bus label.
- `Region_Name`: configured region name.
- `Tech_ID`: technology id.
- `UnitGroup`: grouped technology label.
- `Unit_Category`: technology category.
- `Unit_Type`: technology type.
- `Fuel`: fuel label.
- `TotalUnits`: total units in service in that stage.
- `NewUnits`: units added in that stage.
- `RetUnits`: units retired in that stage.
- `ICAP`: installed capacity.
- `UCAP`: unforced capacity after applying capacity credit.
- `ICap_New`: installed capacity associated with new additions.
- `ICap_Ret`: installed capacity associated with retirements.
- `UCap_New`: unforced capacity associated with additions.
- `UCap_Ret`: unforced capacity associated with retirements.
- `Storage_MWh`: storage energy capacity.
- `Storage_Hr`: storage duration in hours.
- `Generation`: annualized generation attributed to the unit in that stage.
- `Curtail`: annualized curtailment.
- `Storage_Charge_MWh`: annualized storage charging.
- `Reserve_RegUp`: annualized regulation-up provision.
- `Reserve_RegDn`: annualized regulation-down provision.
- `Reserve_Spin`: annualized spinning reserve provision.
- `Reserve_NSpin`: annualized non-spinning reserve provision.
- `Reserve_FlexUp`: annualized upward flexible reserve provision.
- `Reserve_FlexDn`: annualized downward flexible reserve provision.
- `Generation_Cost`: generation operating cost.
- `Charge_Cost`: charging cost.
- `Regulation_Cost`: regulation cost.
- `Spin_Cost`: spinning reserve cost.
- `Nspin_Cost`: non-spinning reserve cost.
- `Flex_Cost`: flexible reserve cost.
- `UnitRevenue_E`: energy-market revenue.
- `UnitRevenue_AS`: ancillary-service revenue.
- `UnitRevenue_CRED`: revenue from policy or credit-style components.
- `UnitRevenue`: total revenue.
- `UnitProfit`: net revenue minus reported cost components.
- `FuelConsumption`: annualized fuel consumption.
- `FuelCost`: annualized fuel cost.
- `FOM`: fixed O&M cost.
- `CAPCRED`: capacity credit used for the unit.
- `Reference_Annual_Gen_Investment_Cost`: annualized reference investment basis before the full discounted build-up across the payment period.
- `Gen_Investment_Cost_Committed`: discounted generator investment cost attributed to the stage that decided the units — the present value of the **whole** payment stream (through the end of the horizon, limited by the cost-recovery period), not only the payments falling inside the stage.
- `Gen_ITC_Committed`: discounted investment tax credit attributed to the stage, on the same whole-stream basis as `Gen_Investment_Cost_Committed`.

### Annual file columns
The annual file keeps the same operational and capacity fields, but replaces the stage-level investment and tax-credit fields with annual cash-flow style fields:
- `Scenario`: case identifier.
- `Stage`: planning-stage id.
- `Start Year`: stage start year.
- `Number of Years`: stage length.
- `Bus_ID`: bus index.
- `Bus_Name`: bus label.
- `Parent_Bus_Name`: parent bus label.
- `Region_Name`: configured region name.
- `Tech_ID`: technology id.
- `UnitGroup`: grouped technology label.
- `Unit_Category`: technology category.
- `Unit_Type`: technology type.
- `Fuel`: fuel label.
- `TotalUnits`: total units in service.
- `NewUnits`: units added in the stage.
- `RetUnits`: units retired in the stage.
- `ICAP`: installed capacity.
- `UCAP`: unforced capacity.
- `ICap_New`: installed capacity added.
- `ICap_Ret`: installed capacity retired.
- `UCap_New`: unforced capacity added.
- `UCap_Ret`: unforced capacity retired.
- `Storage_MWh`: storage energy capacity.
- `Storage_Hr`: storage duration.
- `Generation`: annual generation.
- `Curtail`: annual curtailment.
- `Storage_Charge_MWh`: annual charging.
- `Reserve_RegUp`: annual regulation-up provision.
- `Reserve_RegDn`: annual regulation-down provision.
- `Reserve_Spin`: annual spinning reserve provision.
- `Reserve_NSpin`: annual non-spinning reserve provision.
- `Reserve_FlexUp`: annual upward flexible reserve provision.
- `Reserve_FlexDn`: annual downward flexible reserve provision.
- `Generation_Cost`: annual generation cost.
- `Charge_Cost`: annual charging cost.
- `Regulation_Cost`: annual regulation cost.
- `Spin_Cost`: annual spinning reserve cost.
- `Nspin_Cost`: annual non-spinning reserve cost.
- `Flex_Cost`: annual flexible reserve cost.
- `UnitRevenue_E`: annual energy revenue.
- `UnitRevenue_AS`: annual ancillary-service revenue.
- `UnitRevenue_CRED`: annual credit-related revenue.
- `UnitRevenue`: annual total revenue.
- `UnitProfit`: annual profit.
- `FuelConsumption`: annual fuel use.
- `FuelCost`: annual fuel cost.
- `FOM`: annual fixed O&M cost.
- `CAPCRED`: capacity credit.

## 11. System Summary Output

### Files
- `<Case_ID>__system_summary_by_stage_EXP.csv`
- `<Case_ID>__system_summary_by_year_EXP.csv`
- `<Case_ID>__system_summary_by_year_EXP_real_<dollar_year>usd.csv`

System-wide cost, generation, reliability, emissions and adequacy totals. Use these files for headline results such as total system cost by component, energy not served and the planning reserve margin. The planning-stage file reports stage-level totals. The annual file reports one row per year with present-value costs (`*_PV`). The constant-dollar file is a companion to the annual file that removes the NPV discount from every cost column (see [Constant-dollar annual file](#constant-dollar-annual-file) below). `<dollar_year>` is the `dollar_year_value` from the `Planning Design` sheet (e.g. `2022` → `..._real_2022usd.csv`).

!!! note "Planning reserve margin (`PRM`) definition"
    The `PRM` column is reported as `credited UCAP / coincident peak demand − 1`, where credited UCAP is the sum, over the units in service, of each unit's installed capacity times its capacity credit, in MW. The denominator is the system **coincident** peak demand: the regional hourly load shapes are summed first and the annual maximum is taken from that combined series, with per-region load growth applied. The peak is computed from the full 8760-hour annual load time series, not from the selected representative days, so scenario reduction does not affect it. Because the coincident peak is no larger than the sum of the individual regional peaks, this value is lower than a sum-of-regional-peaks ("non-coincident") basis.

### Planning-stage file columns

!!! note "Paid-in-stage versus committed costs"
    `Gen_Investment_Cost`, `Gen_ITC`, `Trans_Investment_Cost`, `Generation_PTC` and `total_system_cost` hold what is **paid within the stage**: they sum exactly to the corresponding `*_PV` rows of that stage in the annual file. The `*_Committed` columns (`Gen_Investment_Cost_Committed`, `Gen_ITC_Committed`, `Trans_Investment_Cost_Committed`, `Generation_PTC_Committed`, appended at the end of the file) hold the present value of the **whole** payment stream (through the end of the horizon, limited by the cost-recovery period) of the units and lines decided in that stage. Totals over all stages are identical either way.

    Example: with 2 stages of 2 years, stage-1 builds are paid over 2025-2028. The by-stage row of stage 1 shows only the 2025-2026 payments, while the `*_Committed` columns show all four years.

- `Case_ID`: case identifier.
- `Stage`: planning-stage id.
- `Start_Year`: actual starting calendar year of the stage.
- `Years_in_Stage`: stage length.
- `Discount_Year`: the discount anchor year (`dollar_year_value`) to which all discounted costs in the file are referenced.
- `Gen_Investment_Cost`: discounted generation investment cost paid within the stage.
- `Gen_ITC`: discounted generator investment tax credit received within the stage.
- `Trans_Investment_Cost`: discounted transmission investment cost paid within the stage.
- `Trans_FOM_Cost`: discounted transmission fixed O&M cost — annual FOM on the **full
  in-service grid** (existing baseline + expansion), computed per branch as
  `transmission_FOM_percent_value × 0.01 × overnight cost × rate_a × length × (1 + cumulative expansion)`
  (`length = 1` for DC ties), charged every operating year of the
  stage and discounted. It is a real recurring cost and is **included in**
  `total_system_cost`. `0` when `transmission_FOM_percent_value` is `0` (the default). See
  [GTEP Transmission Expansion — Transmission fixed O&M](./GTEP_Transmission_Expansion.md#transmission-fixed-om-fom).
- `Gen_Retirement_Cost`: discounted generator retirement cost.
- `FOM_Cost`: discounted fixed O&M cost.
- `Generation_PTC`: discounted production tax credit value received within the stage.
- `Fuel_Cost`: discounted fuel cost.
- `VOM_Cost`: discounted variable O&M cost.
- `Commitment_Cost`: discounted commitment cost, including no-load and startup cost terms.
- `Regulation_Cost`: discounted regulation reserve cost.
- `Spin_Cost`: discounted spinning reserve cost.
- `Nspin_Cost`: discounted non-spinning reserve cost.
- `Flex_Cost`: discounted flexible reserve cost.
- `ENS_Cost`: discounted value of lost load cost from ENS.
- `RNS_Spin_Cost`: discounted reserve-shortage penalty for spinning reserve.
- `RNS_NSpin_Cost`: discounted reserve-shortage penalty for non-spinning reserve.
- `RNS_Flex_Cost`: discounted reserve-shortage penalty for flexible reserve shortages.
- `CTAX_cost`: discounted carbon-tax cost.
- `CEGT_Penalty`: discounted clean-energy target slack penalty.
- `RPS_Penalty`: discounted renewable-target slack penalty.
- `total_system_cost`: total discounted system cost paid within the stage (sums to the stage's `total_system_cost_PV` rows in the annual file).
- `ObjectiveValue`: objective value associated with the solved model.
- `Generation`: annualized system generation.

!!! warning "`ObjectiveValue` is round-local and normalization-scaled (myopic multi-round)"
    Under myopic multi-round (`multi_round_solution_process_flag = TRUE`), the
    `ObjectiveValue` in each stage row is the objective of **that stage's own round**,
    not a whole-horizon value; each decision stage takes its owning round's objective
    (each decision stage takes the objective of its owning round). It is also
    reported **multiplied by `per_unit_econ_base_value`** (the economic normalization
    base from the `Simulation Setting` sheet), so its magnitude tracks that scaling
    setting, not the economics. Consequences:

    - **Do not compare `ObjectiveValue` across runs that used different
      `per_unit_econ_base_value`.**
    - The `per_unit_econ_base_value` scaling is **economically neutral**: the optimal
      decisions are identical, only the scale of the reported number changes.

!!! warning "`sum(ObjectiveValue)` is NOT a horizon NPV (myopic multi-round)"
    With myopic multi-round (`num_decision_stages_per_round_value = 1`,
    `num_lookahead_stages_per_round_value = 0`), each round amortizes **new**
    investment over only `min(CRP, round window)` years — `CRP` is the asset's cost-recovery
    period (e.g. `transmission_investment_CRP_value`), the number of years over which its
    capital cost is paid off: `remaining_years` collapses to
    a single stage's length and the payment duration is `min(CRP, stage length)`, so the
    tail of long-lived capital (e.g. `transmission_investment_CRP_value = 40` yr vs a 5-yr
    stage) is never charged. **Summing the per-round `ObjectiveValue`s undercounts
    lifetime capital and is not a valid system NPV.** Also note that both `ObjectiveValue`
    and the `total_system_cost` / `*_Cost` / `*_PV` columns are **already discounted** to
    `dollar_year_value` with calendar-correct factors
    (`discount_factor = (1 + discount_rate)^-(future_year − dollar_year_value)`), so they
    must **not** be re-discounted. For a defensible horizon system-cost / NPV, either run
    non-myopic (`multi_round_solution_process_flag = FALSE`, single joint optimization) or
    rebuild capital as a full-CRP annuity in post-processing.
- `TnD_Loss`: transmission & distribution loss energy (MWh) for the stage — for each
  year in the stage, the annual nominal demand (full 8760-hour load series with load
  growth applied, not the representative days) times
  `transmission_loss_percent_value * 0.01`, summed over the stage's years; `0` when
  `enforce_transmission_loss_flag` is not `TRUE`. See
  [Simulation Setting File Reference](../../configuration/ALEAF_Simulation_Setting_File.md#transmission_loss_percent_value).
- `Storage_Charge_MWh`: annualized system charging energy.
- `Reserve_RegUp`: annualized regulation-up provision.
- `Reserve_RegDn`: annualized regulation-down provision.
- `Reserve_Spin`: annualized spinning reserve provision.
- `Reserve_NSpin`: annualized non-spinning reserve provision.
- `Reserve_FlexUp`: annualized upward flexible reserve provision.
- `Reserve_FlexDn`: annualized downward flexible reserve provision.
- `ENS`: annualized energy not served.
- `RNS_Spin`: annualized spinning reserve shortage.
- `RNS_NSpin`: annualized non-spinning reserve shortage.
- `RNS_Flex`: annualized flexible reserve shortage.
- `Emission`: annualized emissions.
- `PRM`: planning reserve margin reported for the stage, `credited UCAP / coincident peak demand − 1` (see the definition note above).
- `Peak_Demand_MW` and `UCAP_MW`: the coincident peak demand and credited UCAP behind `PRM` (`PRM = UCAP_MW / Peak_Demand_MW − 1`).
- `Installed_Capacity_MW`: installed capacity in service.
- `Annual_Input_Demand_MWh`: the annual demand from the input data for the stage's dispatched year, times the stage length.
- `Dispatched_Load_MWh`: the load actually dispatched, i.e. the representative-day load weighted by the number of days each day stands for, times the stage length. It differs from `Annual_Input_Demand_MWh` because the few representative days do not exactly reproduce annual energy (e.g. +5.8% in one test case); generation, costs and emissions follow the dispatched load. The load is not rescaled to the annual demand.
- `Gen_Investment_Cost_Committed`, `Gen_ITC_Committed`, `Trans_Investment_Cost_Committed`, `Generation_PTC_Committed`: whole-payment-stream present values of the units and lines decided in the stage (see the note above).

### Annual file columns
The annual system-summary file keeps the same operational and reliability totals but reports one row per year, with cost fields as present values (`*_PV`). The columns `ObjectiveValue` and `Number of Years` are not written to this file.

!!! note "The annual `*_PV` cost columns are NPV-discounted"
    Every `*_PV` cost column in this file is **discounted to `dollar_year_value`** (recorded in the `Discount_Year` column). The `*_PV` suffix marks a present value, not a cash flow: the
    per-year cost is multiplied by `discount_factor = (1 + discount_rate)^-(year − dollar_year_value)`
    (real `discount_rate` from `discount_rate_value`, both on the `Planning Design` sheet).
    So these are present values anchored at `dollar_year_value`, not the raw cost
    incurred in each year. For the same columns expressed as the real cost incurred each
    year (discount removed), use the companion
    [constant-dollar annual file](#constant-dollar-annual-file).

!!! note "Cost-column semantics"
    Every `*_Cost_PV` column is a full dollar cost (price × quantity, representative-day–weighted and discounted), consistent with the corresponding planning-stage `*_Cost` column — it is **not** a quantity-weighted value. In particular the reserve cost columns (`Regulation_Cost_PV`, `Spin_Cost_PV`, `Nspin_Cost_PV`, `Flex_Cost_PV`) hold dollar costs, not cost × reserve-MW. `total_system_cost_PV` is the sum of all `*_Cost_PV` components (with `Gen_ITC_PV` and `Generation_PTC_PV` entering as credits).

- `Case_ID`: case identifier.
- `Stage`: planning-stage id.
- `Year`: actual year inside the stage.
- `Discount_Year`: the discount anchor year (`dollar_year_value`).
- `Gen_Investment_Cost_PV`: annual generation investment cost.
- `Gen_ITC_PV`: annual investment tax credit.
- `Trans_Investment_Cost_PV`: annual transmission investment cost.
- `Trans_FOM_Cost_PV`: annual transmission fixed O&M cost — the single-year
  form of `Trans_FOM_Cost` (see the planning-stage column above), on the full in-service
  grid (existing baseline + expansion). Included in `total_system_cost_PV`.
- `Gen_Retirement_Cost_PV`: annual retirement cost.
- `FOM_Cost_PV`: annual fixed O&M cost.
- `Generation_PTC_PV`: annual production tax credit.
- `Fuel_Cost_PV`: annual fuel cost.
- `VOM_Cost_PV`: annual variable O&M cost.
- `Commitment_Cost_PV`: annual commitment cost.
- `Regulation_Cost_PV`: annual regulation reserve cost.
- `Spin_Cost_PV`: annual spinning reserve cost.
- `Nspin_Cost_PV`: annual non-spinning reserve cost.
- `Flex_Cost_PV`: annual flexible reserve cost.
- `ENS_Cost_PV`: annual ENS cost.
- `RNS_Spin_Cost_PV`: annual spinning reserve shortage cost.
- `RNS_NSpin_Cost_PV`: annual non-spinning reserve shortage cost.
- `RNS_Flex_Cost_PV`: annual flexible reserve shortage cost.
- `CTAX_cost_PV`: annual carbon-tax cost.
- `CEGT_Penalty_PV`: annual clean-energy target slack penalty.
- `RPS_Penalty_PV`: annual renewable-target slack penalty.
- `total_system_cost_PV`: annual total system cost.
- `Generation`: annual generation.
- `TnD_Loss`: annual transmission & distribution loss energy (MWh) — that year's nominal
  demand (full 8760-hour load series with load growth applied) times
  `transmission_loss_percent_value * 0.01`, or `0` when `enforce_transmission_loss_flag`
  is not `TRUE`. Same basis as the planning-stage column above, but for the single year
  rather than summed over the stage.
- `Storage_Charge_MWh`: annual charging.
- `Reserve_RegUp`: annual regulation-up provision.
- `Reserve_RegDn`: annual regulation-down provision.
- `Reserve_Spin`: annual spinning reserve provision.
- `Reserve_NSpin`: annual non-spinning reserve provision.
- `Reserve_FlexUp`: annual upward flexible reserve provision.
- `Reserve_FlexDn`: annual downward flexible reserve provision.
- `ENS`: annual ENS.
- `RNS_Spin`: annual spinning reserve shortage.
- `RNS_NSpin`: annual non-spinning reserve shortage.
- `RNS_Flex`: annual flexible reserve shortage.
- `Emission`: annual emissions.
- `PRM`: annual planning reserve margin, `credited UCAP / coincident peak demand − 1` (see the definition note above).
- `Peak_Demand_MW` and `UCAP_MW`: the coincident peak demand and credited UCAP behind `PRM`.
- `Installed_Capacity_MW`: installed capacity in service.
- `Annual_Input_Demand_MWh` and `Dispatched_Load_MWh`: input-data annual demand and dispatched (representative-day weighted) load of the year (see the stage-level definitions). Only the first year of a stage is dispatched, so the physical quantities (`Peak_Demand_MW`, `UCAP_MW`, `Installed_Capacity_MW`, `Annual_Input_Demand_MWh`, `Dispatched_Load_MWh`) are replicated across the years of a stage.

### Constant-dollar annual file
- `<Case_ID>__system_summary_by_year_EXP_real_<dollar_year>usd.csv`

This companion to `__system_summary_by_year_EXP.csv` is written automatically on every
expansion run (whenever the annual file is written). `<dollar_year>` is the
`dollar_year_value` from the `Planning Design` sheet (e.g. `2022` →
`..._real_2022usd.csv`). It has the same columns as the annual file, with these differences:

- **Cost columns are undiscounted and labelled `*_real`** (not `*_PV`). Each `*_PV` value of the annual file is multiplied back by
  `(1 + discount_rate)^(year − dollar_year_value)`, exactly cancelling the
  `(1 + discount_rate)^-(year − dollar_year_value)` NPV discount that the annual file
  applies. The result is the **real cost incurred in that year**, expressed in constant
  `dollar_year_value` dollars. `total_system_cost_real` remains the sum of its `*_Cost_real`
  components (with `Gen_ITC_real` and `Generation_PTC_real` as credits).
- **The `Discount_Year` column is dropped**, since the values are no longer discounted.
- **A new last column, `System_Cost_per_MWh_real`**, equals `total_system_cost_real / Dispatched_Load_MWh` (`0` when `Dispatched_Load_MWh` is not positive), so costs and energy come from the same dispatch.
- **All other columns are copied unchanged** — the physical quantities (`Generation`,
  `Emission`, `PRM`, `TnD_Loss`, the `Reserve_*` provisions, `ENS`, `RNS_*`,
  `Storage_Charge_MWh`, `Dispatched_Load_MWh`, etc.) are identical to the annual file.

!!! note "Constant-dollar values are invariant to `dollar_year_value`"
    Because the annual file discounts by `(1 + r)^-(Y − base)` and this file undiscounts by
    `(1 + r)^(Y − base)` using the same `base = dollar_year_value`, the base cancels
    exactly. The `*_real` values here are just the real costs already present in the input
    data, regardless of which anchor year is set — changing `dollar_year_value` does not
    change these numbers (it only changes the annual file's discount level and the
    `<dollar_year>` in the filename).

!!! warning "`dollar_year_value` is a discount anchor, not an inflation deflator"
    `dollar_year_value` only sets the NPV anchor year. It does **not** re-express costs in a
    different year's purchasing power — the input cost tables are assumed to already be in
    `dollar_year_value` dollars. Setting it to, say, `2025` would relabel this file
    `..._real_2025usd.csv` while the values stay in the original purchasing power, which
    would mislabel the dollar year. Change `dollar_year_value` only when the underlying cost
    inputs are genuinely in that year's dollars.

### Numerical-noise rule

In the system and technology summary files (`system_summary_by_stage_EXP`, `system_summary_by_year_EXP`, `tech_summary_by_stage_EXP`) and in `gen_expansion_EXP`, `line_expansion_EXP`, `unserved_energy_EXP` and `reserve_shortfall_EXP`, numeric values with magnitude below `1e-3` are written as `0` (solver noise such as `1e-8` "new units" or `6e-5` MWh of unserved energy). Dollar columns (names containing `Cost`, `Penalty`, `_PV`, `_real` or `_Committed`, and `Gen_ITC`, `Generation_PTC`) use a tolerance of $1 instead, so that `5e-5` MWh of unserved energy at a $9000/MWh penalty does not leave a $0.5 cost. The `PRM` and `CAPCRED` columns are exempt.

## 12. Representative-Day Selection Output

### File
- `<Case_ID>__representative_days_EXP.csv`

### Columns
- `Case_ID`: case identifier.
- `Stage`: planning-stage id.
- `Year`: calendar year of the stage (`first_stage_year_value + (Stage - 1) * num_years_per_stage_value`).
- `Rep_Day`: representative-day index used by the model.
- `Stochastic_Scenario_ID`: stochastic scenario identifier for the representative day when stochastic expansion is enabled. Otherwise `0`.
- `Days_Represented`: number of full-year days the representative day stands for (weight of the day itself).
- `Day_Group_ID`: grouped-day id that the representative day belongs to.
- `Days_in_Group`: total number of full-year days in the representative day's day group. This is a different quantity from `Days_Represented`, which is the weight of this representative day alone.
- `Day_of_Year`: original calendar day of the year used as the representative day.

Use this file to see which days were selected for each stage and how many days each represents. The same columns are reported in single-run and multi-round modes.

## 13. Simulation Run Time Output

### File
- `<Case_ID>__simulation_run_time.csv`

Solve times of the run.

### Columns
- `Case_ID`: case identifier.
- `Expansion_Run_Time_s`: expansion solve time in seconds.
- `Operation_Run_Time_s`: operation solve time in seconds.
- `RA_Run_Time_s`: resource-adequacy time in seconds.

The RA time is reported separately even in multi-round runs: RA calls made inside the rolling-horizon loop are counted as RA time, not as expansion time.

## Practical Reading Guide
If you need a minimal expansion-result review sequence, inspect:
1. `__system_summary_by_stage_EXP.csv`
2. `__tech_summary_by_stage_EXP.csv`
3. `__gen_expansion_EXP.csv`
4. `__line_expansion_EXP.csv`
5. `__market_EXP.csv`
6. `__dispatch_EXP_year_<stage>.csv`
7. `__representative_days_EXP.csv`

## Related Documentation
- [GTEP Overview](./GTEP_Overview.md)
- [GTEP Planning Horizon and Multi-Round](./GTEP_Planning_Horizon_and_Multi_Round.md)
- [Scenario Reduction and Representative-Day Groups](../../configuration/Scenario_Reduction_and_Repday_Groups.md)

## Output naming history

Use this section to translate file and column names from earlier releases, for example in existing post-processing scripts. Only the current names are written.

| Old name | Current name |
|---|---|
| `system_summary_planning_stage_{EXP,OP}` | `system_summary_by_stage_{EXP,OP}` |
| `system_summary_annual_{EXP,OP}` | `system_summary_by_year_{EXP,OP}` |
| `..._annual_{EXP,OP}_constant_<year>usd` | `system_summary_by_year_{EXP,OP}_real_<year>usd` |
| `system_tech_summary_planning_stage_EXP`, `system_tech_summary_OP` | `tech_summary_by_stage_{EXP,OP}` |
| `system_tech_summary_annual_EXP` | `tech_summary_by_year_EXP` |
| `dispatch_{EXP,OP}_of_year_<N>` | `dispatch_{EXP,OP}_year_<N>` |
| `expansion_gen_result_EXP`, `expansion_line_result_EXP` | `gen_expansion_EXP`, `line_expansion_EXP` |
| `scarcity_ens_result_EXP`, `scarcity_reserve_result_EXP` | `unserved_energy_EXP`, `reserve_shortfall_EXP` |
| `repday_selection_result_{EXP,OP}` | `representative_days_{EXP,OP}` |
| `policy_{EXP,OP}` | `policy_slack_{EXP,OP}` |

Column renames in the summary files:

| File | Old column | Current column |
|---|---|---|
| `system_summary_by_{stage,year}_{EXP,OP}` (and `_real_<year>usd`), `tech_summary_by_stage_{EXP,OP}` | `Scenario` | `Case_ID` |
| `system_summary_by_stage_{EXP,OP}`, `tech_summary_by_stage_{EXP,OP}` | `Start Year` | `Start_Year` |
| `system_summary_by_stage_{EXP,OP}`, `tech_summary_by_stage_{EXP,OP}` | `Number of Years` | `Years_in_Stage` |
| `system_summary_by_stage_OP` | `Total_system_cost_OP` | `Operating_Cost` |
| `system_summary_by_year_{EXP,OP}` | `*_CF` (e.g. `total_system_cost_CF`) | `*_PV` (e.g. `total_system_cost_PV`) |
| `system_summary_by_year_OP` | `total_system_cost_OP_CF` | `Operating_Cost_PV` |
| `system_summary_by_year_{EXP,OP}_real_<year>usd` | `*_CF` | `*_real` |
| `system_summary_by_year_{EXP,OP}` | `ObjectiveValue`, `Number of Years` | removed |
| `tech_summary_by_stage_{EXP,OP}` | `Gen_Investment_Cost`, `Gen_ITC` | `Gen_Investment_Cost_Committed`, `Gen_ITC_Committed` |

New columns: `Discount_Year` (system summaries, except the `_real_<year>usd` files), `Peak_Demand_MW`, `UCAP_MW`, `Installed_Capacity_MW`, `Annual_Input_Demand_MWh`, `Dispatched_Load_MWh` (system summaries), `*_Committed` cost columns (`system_summary_by_stage_{EXP,OP}`) and `System_Cost_per_MWh_real` (`_real_<year>usd` files).

Column renames in the hourly and stage report files (the same names apply to the expansion (EXP) and operation (OP) files):

| File | Old column | Current column |
|---|---|---|
| all files below | `Scenario` | `Case_ID` |
| all files below | `year` (the stage number) | `Stage` (a calendar `Year` column is inserted right after it) |
| hourly files (dispatch, market, policy_slack, demand_response, power_flow, unserved_energy, reserve_shortfall) | `day`, `hour`, `time` | `Rep_Day`, `Hour`, `Sub_Period` |
| dispatch, market, `representative_days_EXP` | `Stochastic_scenario_ID` | `Stochastic_Scenario_ID` |
| dispatch, market, power_flow, `representative_days_{EXP,OP}` | `NumDays` | `Days_Represented` |
| `dispatch_{EXP,OP}_year_<N>` | `unit_id` | `Unit_ID` |
| `dispatch_{EXP,OP}_year_<N>` | `PLANT_NAME` | `Plant_Name` |
| `dispatch_{EXP,OP}_year_<N>` | `bus_id` | `Bus_ID` |
| `dispatch_{EXP,OP}_year_<N>` | `UnitGroup` | `Unit_Group` |
| `dispatch_{EXP,OP}_year_<N>` | `u_G_iy` | `Units_In_Service` |
| `dispatch_{EXP,OP}_year_<N>` | `u_ESE_iy` | `Storage_Energy_MWh` |
| `dispatch_{EXP,OP}_year_<N>` | `ICAP` | `Installed_Capacity_MW` |
| `dispatch_{EXP,OP}_year_<N>` | `sto_c_idhty` | `Storage_Charging_Flag` |
| `dispatch_{EXP,OP}_year_<N>` | `c_idhty` | `Units_Committed` |
| `dispatch_{EXP,OP}_year_<N>` | `su_idhty` | `Units_Started` |
| `dispatch_{EXP,OP}_year_<N>` | `g_idhty` | `Generation_MW` |
| `dispatch_{EXP,OP}_year_<N>` | `reg_up_idhty` | `Reserve_RegUp_MW` |
| `dispatch_{EXP,OP}_year_<N>` | `reg_dn_idhty` | `Reserve_RegDn_MW` |
| `dispatch_{EXP,OP}_year_<N>` | `spin_idhty` | `Reserve_Spin_MW` |
| `dispatch_{EXP,OP}_year_<N>` | `nonspin_idhty` | `Reserve_NSpin_MW` |
| `dispatch_{EXP,OP}_year_<N>` | `flex_up_idhty` | `Reserve_FlexUp_MW` |
| `dispatch_{EXP,OP}_year_<N>` | `flex_dn_idhty` | `Reserve_FlexDn_MW` |
| `dispatch_{EXP,OP}_year_<N>` | `curt_idhty` | `Curtailment_MW` |
| `dispatch_{EXP,OP}_year_<N>` | `chg_idhty` | `Charge_MW` |
| `dispatch_{EXP,OP}_year_<N>` | `soc_idhty` | `SOC_MWh` |
| `dispatch_{EXP,OP}_year_<N>` | `hybrid_chg_idhty` | `Hybrid_Charge_MW` |
| `dispatch_{EXP,OP}_year_<N>` | `hybrid_type` | `Hybrid_Type` |
| `dispatch_{EXP,OP}_year_<N>` | `Fuel_Type` | `Fuel` |
| `dispatch_{EXP,OP}_year_<N>` | `Fuel_Cost` | `Fuel_Cost_USD` |
| `dispatch_{EXP,OP}_year_<N>` | `MC` | `Marginal_Cost_USD_per_MWh` |
| `dispatch_{EXP,OP}_year_<N>` | `vre_shape` | `VRE_Capacity_Factor` |
| `market_{EXP,OP}` | `bus_id` | `Bus_ID` |
| `market_{EXP,OP}` | `load` | `Load_MW` |
| `market_{EXP,OP}` | `scarcity_E` | `Unserved_Energy_MW` |
| `market_{EXP,OP}` | `scarcity_SPIN` | `Spin_Shortfall_MW` |
| `market_{EXP,OP}` | `Demand_Reserve` | `Demand_Reserve_MW` |
| `market_{EXP,OP}` | `scarcity_NSPIN` | `NSpin_Shortfall_MW` |
| `market_{EXP,OP}` | `scarcity_FU` | `FlexUp_Shortfall_MW` |
| `market_{EXP,OP}` | `scarcity_FD` | `FlexDown_Shortfall_MW` |
| `market_{EXP,OP}` | `LMP` | `LMP_USD_per_MWh` |
| `market_{EXP,OP}` | `RCP_RU` | `Price_RegUp_USD_per_MW` |
| `market_{EXP,OP}` | `RCP_RD` | `Price_RegDn_USD_per_MW` |
| `market_{EXP,OP}` | `RCP_Spin` | `Price_Spin_USD_per_MW` |
| `market_{EXP,OP}` | `RCP_NSpin` | `Price_NSpin_USD_per_MW` |
| `market_{EXP,OP}` | `RCP_FU` | `Price_FlexUp_USD_per_MW` |
| `market_{EXP,OP}` | `RCP_FD` | `Price_FlexDn_USD_per_MW` |
| `market_{EXP,OP}` | `ens_indicator_ndhty` | `ENS_Flag` |
| `policy_slack_{EXP,OP}` | `bus_id` | `Policy_Zone_ID` |
| `policy_slack_{EXP,OP}` | `slack_CEG_ndy` | `Clean_Energy_Target_Slack` |
| `policy_slack_{EXP,OP}` | `slack_RPS_ny` | `RPS_Target_Slack` |
| `demand_response_{EXP,OP}` | `PLANT_NAME` | `Plant_Name` |
| `demand_response_{EXP,OP}` | `bus_idx` | `Bus_ID` |
| `demand_response_{EXP,OP}` | `bus_name` | `Bus_Name` |
| `demand_response_{EXP,OP}` | `UNITGROUP` | `Unit_Group` |
| `demand_response_{EXP,OP}` | `UNIT_CATEGORY` | `Unit_Category` |
| `demand_response_{EXP,OP}` | `UNIT_REPORT_LABEL_1` | `Unit_Report_Label_1` |
| `demand_response_{EXP,OP}` | `UNIT_REPORT_LABEL_2` | `Unit_Report_Label_2` |
| `demand_response_{EXP,OP}` | `CAP` | `Unit_Capacity_MW` |
| `demand_response_{EXP,OP}` | `INTERCON_LIM` | `Interconnection_Limit_MW` |
| `demand_response_{EXP,OP}` | `lfl_lt` | `LFL_Load_MW` |
| `demand_response_{EXP,OP}` | `lfl_DR_lt` | `LFL_DR_MW` |
| `demand_response_{EXP,OP}` | `lfl_seg_lt_<k>` | `LFL_DR_Segment_<k>_MW` |
| `demand_response_{EXP,OP}` | `lfl_ind_lt_<k>` | `LFL_DR_Segment_<k>_Active` |
| `demand_response_{EXP,OP}` | `lfl_g_G_LFL_lt` | `LFL_Gen_to_Load_MW` |
| `demand_response_{EXP,OP}` | `lfl_g_G_Grid_lt` | `LFL_Gen_to_Grid_MW` |
| `demand_response_{EXP,OP}` | `lfl_g_G_ES_lt` | `LFL_Gen_to_Storage_MW` |
| `demand_response_{EXP,OP}` | `lfl_g_ES_LFL_lt` | `LFL_Storage_to_Load_MW` |
| `demand_response_{EXP,OP}` | `lfl_g_ES_Grid_lt` | `LFL_Storage_to_Grid_MW` |
| `demand_response_{EXP,OP}` | `lfl_chg_Grid_ES_lt` | `LFL_Grid_to_Storage_MW` |
| `demand_response_{EXP,OP}` | `lfl_soc_lt` | `LFL_Storage_SOC_MWh` |
| `power_flow_{EXP,OP}` | `line_id` | `Line_ID` |
| `power_flow_{EXP,OP}` | `f_bus` | `From_Bus_ID` |
| `power_flow_{EXP,OP}` | `t_bus` | `To_Bus_ID` |
| `power_flow_{EXP,OP}` | `f_region` | `From_Region` |
| `power_flow_{EXP,OP}` | `t_region` | `To_Region` |
| `power_flow_{EXP,OP}` | `orinal_rate` | `Original_Rating_MW` |
| `power_flow_{EXP,OP}` | `expansion` | `Cumulative_Expansion_Fraction` |
| `power_flow_{EXP,OP}` | `final_rate` | `Final_Rating_MW` |
| `power_flow_{EXP,OP}` | `flow` | `Flow_MW` |
| `power_flow_{EXP,OP}` | `LMP_from_bus` | `LMP_From_Bus_USD_per_MWh` |
| `power_flow_{EXP,OP}` | `LMP_to_bus` | `LMP_To_Bus_USD_per_MWh` |
| `power_flow_{EXP,OP}` | `congestion_flag` | `Congested_Flag` |
| `power_flow_{EXP,OP}` | `wheeling_cost` | `Wheeling_Cost` |
| `power_flow_{EXP,OP}` | `line_length` | `Line_Length` |
| `power_flow_{EXP,OP}` | `f_exp` | `Expansion_Flow_MW` |
| `power_flow_{EXP,OP}` | `flow_total` | `Total_Flow_MW` |
| `power_flow_{EXP,OP}` | `kvl_residual` | `KVL_Residual_MW` |
| `unserved_energy_EXP` | `node` | `Bus_ID` |
| `unserved_energy_EXP` | `ens_ndhty` | `Unserved_Energy_MW` |
| `reserve_shortfall_EXP` | `zone` | `Reserve_Zone_ID` |
| `reserve_shortfall_EXP` | `rns_spin_zdhty` | `Spin_Shortfall_MW` |
| `reserve_shortfall_EXP` | `demand_reserve_zdhty` | `Demand_Reserve_MW` |
| `reserve_shortfall_EXP` | `rns_nonspin_zdhty` | `NSpin_Shortfall_MW` |
| `reserve_shortfall_EXP` | `rns_flex_up_zdhty` | `FlexUp_Shortfall_MW` |
| `reserve_shortfall_EXP` | `rns_flex_dn_zdhty` | `FlexDown_Shortfall_MW` |
| `line_expansion_EXP` | `line_id` | `Line_ID` |
| `line_expansion_EXP` | `f_bus` | `From_Bus_ID` |
| `line_expansion_EXP` | `t_bus` | `To_Bus_ID` |
| `line_expansion_EXP` | `f_bus_name` | `From_Bus_Name` |
| `line_expansion_EXP` | `t_bus_name` | `To_Bus_Name` |
| `line_expansion_EXP` | `merged_line_UIDs` | `Merged_Line_UIDs` |
| `line_expansion_EXP` | `original_rate` | `Original_Rating_MW` |
| `line_expansion_EXP` | `u_new_T_ky` | `New_Expansion_Fraction` |
| `line_expansion_EXP` | `u_T_ky` | `Cumulative_Expansion_Fraction` |
| `line_expansion_EXP` | `final_rate` | `Final_Rating_MW` |
| `line_expansion_EXP` | `length` | `Length` |
| `gen_expansion_EXP` | `unit_id` | `Unit_ID` |
| `gen_expansion_EXP` | `PLANT_NAME` | `Plant_Name` |
| `gen_expansion_EXP` | `UnitGroup` | `Unit_Group` |
| `gen_expansion_EXP` | `u_new_G_iy` | `New_Units` |
| `gen_expansion_EXP` | `u_ret_G_iy` | `Retired_Units` |
| `gen_expansion_EXP` | `u_G_iy` | `Units_In_Service` |
| `gen_expansion_EXP` | `u_new_ESH_iy` | `New_Storage_Unit_Hours` |
| `gen_expansion_EXP` | `u_ESE_iy` | `Storage_Energy_MWh` |
| `representative_days_{EXP,OP}` | `RepDay_id` | `Rep_Day` |
| `representative_days_{EXP,OP}` | `Daygroup_ID` | `Day_Group_ID` |
| `representative_days_{EXP,OP}` | `Numdays_Daygroup` | `Days_in_Group` |
| `representative_days_{EXP,OP}` | `Reference_Day` | `Day_of_Year` |
| `simulation_run_time` | `Expansion Run Time`, `Operation Run Time`, `RA Run Time` | `Expansion_Run_Time_s`, `Operation_Run_Time_s`, `RA_Run_Time_s` |

Notes: `bus_id` and `bus_idx` both become `Bus_ID` (`bus_id` in the policy files becomes `Policy_Zone_ID`); `original_rate` and the misspelled `orinal_rate` both become `Original_Rating_MW`; `u_T_ky` and `expansion` both become `Cumulative_Expansion_Fraction`; `ens_ndhty` and `scarcity_E` both become `Unserved_Energy_MW`, and the reserve-shortage columns share the names `Spin_Shortfall_MW`, `NSpin_Shortfall_MW`, `FlexUp_Shortfall_MW`, `FlexDown_Shortfall_MW`, `Demand_Reserve_MW` between the market and `reserve_shortfall_EXP` files. `Unit_Type` is no longer written in the dispatch files; `Unit_Report_Label_1` and `Unit_Report_Label_2` replace it. `gen_expansion_EXP` also gained `Year`, `Plant_Name`, `Unit_Category`, `Bus_Name`, `Unit_Capacity_MW`, `New_Capacity_MW`, `Retired_Capacity_MW` and `Total_Capacity_MW`. `representative_days` keeps `Days_Represented` (days the representative day stands for) and `Days_in_Group` (days in its day group) as separate quantities.
