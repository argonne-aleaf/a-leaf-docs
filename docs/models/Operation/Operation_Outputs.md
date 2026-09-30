# Operation Outputs

This page is a user's guide to the files written by an operation run: what each file contains, what its columns mean and in which units, how to read the results, and which checks to run before relying on them. For how operation runs are set up and executed, see [Operation Execution and Run Modes](./Operation_Execution_and_Run_Modes.md).

## Output overview

Each operation run writes a family of CSV files (and, optionally, a JSON snapshot) to the case output folder. All file names start with the `Case_ID` from the `Simulation Configuration` sheet.

| File | Content | Granularity |
|---|---|---|
| `<Case_ID>__dispatch_OP_year_<stage>.csv` | Unit dispatch, reserves, storage state | One file per stage; unit by representative day and hour |
| `<Case_ID>__market_OP.csv` | Load, scarcity, LMPs, reserve prices | Bus by representative day and hour |
| `<Case_ID>__policy_slack_OP.csv` | Clean-energy target slack | Policy zone by representative day |
| `<Case_ID>__demand_response_OP.csv` | Flexible-load and demand-response dispatch | Resource by representative day and hour |
| `<Case_ID>__power_flow_OP.csv` | Branch flows, ratings, congestion | Branch by representative day and hour |
| `<Case_ID>__tech_summary_by_stage_OP.csv` | Per-unit capacity, energy, cost, and revenue | Unit by stage |
| `<Case_ID>__system_summary_by_stage_OP.csv` | System costs and physical totals | One row per stage |
| `<Case_ID>__system_summary_by_year_OP.csv` | Same, discounted (`*_PV`), by year | One row per year |
| `<Case_ID>__system_summary_by_year_OP_real_<dollar_year>usd.csv` | Same, undiscounted (`*_real`) | One row per year |
| `<Case_ID>__representative_days_OP.csv` | Selected representative days and weights | Representative day |
| `<Case_ID>__water_management_OP.csv` | Segment-level reservoir water use | Only when reservoir water management is modeled |
| `<Case_ID>__simulation_run_time.csv` | Solve times of the case | One row per case |
| `ALEAF_LC_GTEP_OP_<test_system_name>_<Case_ID>_<timestamp>.json` | Full solved-model snapshot | Optional |

The JSON snapshot is written only when `export_model_reference_json_operation_flag` is `TRUE` on the `Simulation Setting` sheet. It contains the operation model result, the operation system reference, the network data, and the settings. Writing it forces a full, memory-heavy model reference; see [Operation Execution and Run Modes](./Operation_Execution_and_Run_Modes.md#light-master-reference-nodal-out-of-memory-behavior).

When operation is run in parallel, intermediate per-day-group part files with a `__y<year>_dg<id>` suffix are merged into the files above at the end of the run.

### Report control flags

Report families are controlled by boolean flags on the `Simulation Setting` sheet. All flags are on by default: a missing key, a blank cell, or `TRUE` writes the report, and only an explicit `FALSE` suppresses it. See the [Simulation Setting File Reference](../../configuration/ALEAF_Simulation_Setting_File.md#report-control-gtep-operation) for details.

| File | Controlling flag |
|---|---|
| `<Case_ID>__dispatch_OP_year_<stage>.csv` | `report_dispatch_OP_flag` |
| `<Case_ID>__market_OP.csv` | `report_dispatch_OP_flag` |
| `<Case_ID>__policy_slack_OP.csv` | `report_dispatch_OP_flag` |
| `<Case_ID>__demand_response_OP.csv` | `report_dispatch_OP_flag` |
| `<Case_ID>__power_flow_OP.csv` | `report_power_flow_OP_flag` |
| `<Case_ID>__tech_summary_by_stage_OP.csv` | `report_summary_OP_flag` |
| `<Case_ID>__system_summary_by_stage_OP.csv` | `report_summary_OP_flag` |
| `<Case_ID>__system_summary_by_year_OP.csv` | `report_summary_OP_flag` |
| `<Case_ID>__system_summary_by_year_OP_real_<dollar_year>usd.csv` | `report_summary_OP_flag` (derived from the annual file) |
| `<Case_ID>__representative_days_OP.csv` | Always written |

## Common field conventions

The hourly and stage files share a set of index columns.

| Column | Meaning |
|---|---|
| `Case_ID` | Case name from the `Simulation Configuration` sheet. |
| `Stage` | Planning-stage id. |
| `Year` | Calendar year of the stage, `first_stage_year_value + (Stage - 1) * num_years_per_stage_value`. Appears right after `Stage`. |
| `Rep_Day` | Representative-day index in the reduced chronology. Its calendar day is `Day_of_Year` in `<Case_ID>__representative_days_OP.csv`. |
| `Hour` | Hour index within the representative day. |
| `Sub_Period` | Time-slice index within the hour; `1` for hourly runs. |
| `Days_Represented` | Number of full-year days that the representative day stands for. |

`Days_Represented` is the weight used to scale reduced-day results toward annual values. To compute an annual energy from an hourly file, multiply each hourly value (MW, for one-hour intervals) by `Days_Represented` and sum.

The `Case_ID`, `Stage`, `Year`, `Rep_Day`, `Hour`, `Sub_Period`, and `Days_Represented` columns are described here once and appear with the same meaning in every file that includes them.

## 1. Dispatch output

### File
`<Case_ID>__dispatch_OP_year_<stage>.csv`, one file per planning stage. Each row is one unit in one representative-day hour.

### Columns

| Column | Meaning |
|---|---|
| `Case_ID`, `Stage`, `Year`, `Rep_Day`, `Hour`, `Sub_Period` | Index columns (see above). |
| `Unit_ID` | Generator index in the model. |
| `Plant_Name` | Generator name used for reporting. |
| `Bus_ID` | Bus index. |
| `Bus_Name` | Bus label. |
| `Parent_Bus_Name` | Parent or higher-level bus label. |
| `Region_Name` | Configured region name. |
| `Tech_ID` | Technology id. |
| `Unit_Group` | Grouped technology label. |
| `Unit_Category` | Technology category. |
| `Unit_Report_Label_1`, `Unit_Report_Label_2` | Free-text reporting labels from the generator technology data. |
| `Units_In_Service` | Number of units in service in the stage (can be fractional for aggregated or relaxed units). |
| `Storage_Energy_MWh` | Storage energy quantity for storage technologies (MWh). |
| `Installed_Capacity_MW` | Installed capacity of the unit row (MW). |
| `Storage_Charging_Flag` | Storage charging status variable, when present. |
| `Units_Committed` | Number of units committed, when unit commitment is modeled. |
| `Units_Started` | Startup indicator, when unit commitment is modeled. |
| `Generation_MW` | Generation output (MW). |
| `Reserve_RegUp_MW`, `Reserve_RegDn_MW` | Regulation-up and regulation-down provision (MW). |
| `Reserve_Spin_MW`, `Reserve_NSpin_MW` | Spinning and non-spinning reserve provision (MW). |
| `Reserve_FlexUp_MW`, `Reserve_FlexDn_MW` | Upward and downward flexible reserve provision (MW). |
| `Curtailment_MW` | Curtailed output (MW). |
| `Charge_MW` | Storage charging power (MW). |
| `SOC_MWh` | Storage state of charge (MWh). |
| `Hybrid_Charge_MW` | Hybrid-resource charging or transfer term (MW). |
| `Hybrid_Type` | Hybrid resource type label (`NA` for non-hybrid units). |
| `Inertia` | Inertia contribution of the resource. |
| `Marginal_Cost_USD_per_MWh` | Marginal cost of the unit row (USD/MWh). |
| `Days_Represented` | Representative-day weight. |

### How to use and check

- Sum `Generation_MW * Days_Represented` by `Unit_Category` or `Tech_ID` to obtain annual energy by technology, and compare with `Generation` in the technology summary.
- For storage, check that `SOC_MWh` stays within `Storage_Energy_MWh` and that `Charge_MW` and `Generation_MW` are consistent with the intended round-trip efficiency.
- Reserve columns are provided in MW for the hour. Total reserve provision should be consistent with the reserve requirements set in the input data.
- Very small values (of order `1e-6` MW) are solver tolerance and can be treated as zero.

## 2. Market output

### File
`<Case_ID>__market_OP.csv`. Each row is one bus in one representative-day hour.

### Columns

| Column | Meaning |
|---|---|
| `Case_ID`, `Stage`, `Year`, `Rep_Day`, `Hour`, `Sub_Period` | Index columns. |
| `Bus_ID`, `Bus_Name`, `Parent_Bus_Name`, `Region_Name` | Bus identification. |
| `Load_MW` | Load represented at the bus and hour (MW). |
| `Unserved_Energy_MW` | Energy not served (load shed) at the bus (MW). |
| `Demand_Reserve_MW` | Demand-reserve quantity (MW). |
| `Spin_Shortfall_MW` | Spinning (contingency) reserve shortfall of the reserve zone (MW). |
| `NSpin_Shortfall_MW` | Non-spinning reserve shortfall of the reserve zone (MW). |
| `FlexUp_Shortfall_MW` | Upward flexible reserve shortfall of the reserve zone (MW). |
| `FlexDown_Shortfall_MW` | Downward flexible reserve shortfall of the reserve zone (MW). |
| `LMP_USD_per_MWh` | Locational marginal price (USD/MWh). |
| `Price_RegUp_USD_per_MW`, `Price_RegDn_USD_per_MW` | Regulation-up and regulation-down clearing prices (USD/MW). |
| `Price_Spin_USD_per_MW`, `Price_NSpin_USD_per_MW` | Spinning and non-spinning reserve clearing prices (USD/MW). |
| `Price_FlexUp_USD_per_MW`, `Price_FlexDn_USD_per_MW` | Upward and downward flexible reserve clearing prices (USD/MW). |
| `Days_Represented` | Representative-day weight. |

!!! note "Reserve shortfalls are per reserve zone and reported once"
    The four reserve shortfall columns describe the reserve zone, not the individual bus. For each timestep, a zone's shortfall is written on one representative bus of that zone and is `0` on the zone's other buses. Summing a shortfall column across all bus rows therefore gives the correct system total. Only `Unserved_Energy_MW` is a true per-bus quantity. Reserve clearing prices are the same for every bus in a zone.

!!! note "Prices under the cuOpt GPU solver"
    The price columns are dual values of the model constraints. With `solver_name = cuOpt`, they come from a first-order (PDLP) method and are approximate relative to a simplex solve. Duals are available only for pure LP models, which includes the economic-dispatch operation model. Validate prices against a HiGHS or CPLEX run, or tighten the `cuOpt Setting` tolerances, when price accuracy matters. See [GPU Solvers](../../configuration/GPU_Solvers.md).

### How to use and check

- Average `LMP_USD_per_MWh` weighted by `Load_MW * Days_Represented` for load-weighted prices by bus, region, or stage.
- Differences in LMP between buses in the same hour indicate congestion or losses; cross-check with `Congested_Flag` in the power-flow file.
- `Unserved_Energy_MW` and the shortfall columns should be zero in a well-supplied system. Persistent non-zero values mean the case is short of capacity, transmission, or reserves; their cost appears in `ENS_Cost` and the `RNS_*_Cost` columns of the system summary.
- Prices equal to `0` throughout can indicate that duals were not available (for example, an unsupported solver setting) and should not be interpreted as free energy.

## 3. Policy output

### File
`<Case_ID>__policy_slack_OP.csv`

### Columns

| Column | Meaning |
|---|---|
| `Case_ID`, `Stage`, `Year`, `Rep_Day` | Index columns. |
| `Policy_Zone_ID` | Bus or policy-zone id used for reporting. |
| `Clean_Energy_Target_Slack` | Slack quantity for the clean-energy generation target. |

The operation policy file reports only the clean-energy slack. It does not include the renewable portfolio standard (RPS) slack column that appears in the expansion policy report. A positive slack means that the clean-energy target was not fully met and a penalty applies (see `CEGT_Penalty` in the system summary).

## 4. Demand response output

### File
`<Case_ID>__demand_response_OP.csv`. Each row is one demand-response or flexible-load resource in one representative-day hour. See [Large Load and Demand Response Reference](../../database/Large_Load_and_Demand_Response_Reference.md) for how these resources are defined.

### Columns

| Column | Meaning |
|---|---|
| `Case_ID`, `Stage`, `Year`, `Rep_Day`, `Hour`, `Sub_Period` | Index columns. |
| `Plant_Name` | Demand-response resource name. |
| `Bus_ID`, `Bus_Name`, `Region_Name`, `Parent_Bus_Name` | Bus identification. |
| `Online_Year` | Online year of the resource. |
| `Unit_Group`, `Unit_Category` | Group and category labels. |
| `Unit_Report_Label_1`, `Unit_Report_Label_2` | Free-text reporting labels. |
| `Unit_Capacity_MW` | Resource capacity (MW). |
| `Interconnection_Limit_MW` | Interconnection limit (MW). |
| `Integer_Flag` | Whether the resource is modeled with integer structure. |
| `Daily_DR_Limit_MWh` | Daily demand-response energy limit (MWh). |
| `Num_DR_Segments` | Number of demand-response segments. |
| `Pct_MW_1` to `Pct_MW_5` | Size of each demand-response segment (MW). |
| `Price_1` to `Price_5` | Price of each segment. |
| `Hybrid_Gen`, `Hybrid_Gen_CAP` | Linked hybrid generation identifier and capacity. |
| `Hybrid_ES`, `Hybrid_ES_CAP` | Linked hybrid storage identifier and capacity. |
| `LFL_Load_MW` | Baseline flexible-load level (MW). |
| `LFL_DR_MW` | Dispatched demand-response level (MW). |
| `LFL_DR_Segment_1_MW` to `LFL_DR_Segment_5_MW` | Dispatched demand response by segment (MW). |
| `LFL_DR_Segment_1_Active` to `LFL_DR_Segment_5_Active` | Segment activation indicators. |
| `LFL_Gen_to_Load_MW` | Onsite generation serving the flexible load directly (MW). |
| `LFL_Gen_to_Grid_MW` | Onsite generation exported to the grid (MW). |
| `LFL_Gen_to_Storage_MW` | Onsite generation charging onsite storage (MW). |
| `LFL_Storage_to_Load_MW` | Storage discharge serving the flexible load (MW). |
| `LFL_Storage_to_Grid_MW` | Storage discharge exported to the grid (MW). |
| `LFL_Grid_to_Storage_MW` | Grid energy charging onsite storage (MW). |
| `LFL_Storage_SOC_MWh` | Onsite storage state of charge (MWh). |

### How to use and check

- Daily totals of `LFL_DR_MW` (over the hours of a representative day) should not exceed `Daily_DR_Limit_MWh`.

## 5. Power flow output

### File
`<Case_ID>__power_flow_OP.csv`. Each row is one branch in one representative-day hour.

### Columns

| Column | Meaning |
|---|---|
| `Case_ID`, `Stage`, `Year`, `Rep_Day`, `Hour`, `Sub_Period` | Index columns. |
| `Line_ID` | Branch index. |
| `From_Bus_ID`, `To_Bus_ID` | From-bus and to-bus indices. |
| `From_Region`, `To_Region` | Region labels of the two ends. |
| `Original_Rating_MW` | Branch rating before any expansion (MW). |
| `Cumulative_Expansion_Fraction` | Transmission expansion carried from the expansion result, as a fraction of the original rating. |
| `Final_Rating_MW` | Branch rating after applying the expansion (MW). |
| `Flow_MW` | Dispatched power flow (MW); the sign shows direction relative to from-bus to to-bus. |
| `LMP_From_Bus_USD_per_MWh`, `LMP_To_Bus_USD_per_MWh` | Locational prices at the two ends (USD/MWh). |
| `Congested_Flag` | Set when the branch is at its limit, `abs(Flow_MW) >= Final_Rating_MW`. |
| `Wheeling_Cost` | Always `0.0`; wheeling cost is not computed in the operation report. |
| `Line_Length` | Branch length. |
| `Days_Represented` | Representative-day weight. |

### How to use and check

- The number of congested hours by branch is the sum of `Congested_Flag * Days_Represented`.
- The price difference `LMP_To_Bus_USD_per_MWh - LMP_From_Bus_USD_per_MWh` on a congested branch indicates the marginal value of additional transmission capacity.
- A flow above `Final_Rating_MW` should not occur. If it does, review the transmission settings and the flow representation in use.
- Transmission losses are not reported per branch. A system-wide loss percentage is applied in the demand balance and reported as `TnD_Loss` in the system summaries. See [Simulation Setting File Reference](../../configuration/ALEAF_Simulation_Setting_File.md#transmission_loss_percent_value).

## 6. Technology summary output

### File
`<Case_ID>__tech_summary_by_stage_OP.csv`. Each row is one unit (or aggregated unit) in one stage, with annualized results.

Units whose `ICAP`, `ICap_New`, and `ICap_Ret` are all below `1e-3` MW are treated as solver noise and are written with every metric column (from `TotalUnits` onward, except `CAPCRED`) set to `0`. Other numeric values below `1e-3` (below 1 USD for dollar columns) are also written as `0`; `PRM` and `CAPCRED` are exempt.

### Columns

| Column | Meaning |
|---|---|
| `Case_ID`, `Stage` | Index columns. |
| `Start_Year` | First year of the planning stage. |
| `Years_in_Stage` | Stage length (years). |
| `Plant_Name` | Plant name. |
| `Bus_ID`, `Bus_Name`, `Parent_Bus_Name`, `Region_Name` | Bus identification. |
| `Tech_ID` | Technology id. |
| `Unit_Group`, `Unit_Category` | Group and category labels. |
| `Unit_Report_Label_1`, `Unit_Report_Label_2` | Free-text reporting labels. |
| `Fuel` | Fuel label. |
| `TotalUnits` | Total units in service in the operation run. |
| `NewUnits`, `RetUnits` | Units added and retired in the linked expansion result, if one exists. |
| `ICAP` | Installed capacity (MW). |
| `UCAP` | Unforced capacity: installed capacity de-rated by capacity credit, `UCAP = ICAP * CAPCRED`. See [GTEP Formulation](../GTEP/GTEP_Formulation.md#parameters). |
| `ICap_New`, `ICap_Ret` | Installed capacity of additions and of retirements (MW). |
| `UCap_New`, `UCap_Ret` | Unforced capacity of additions and of retirements (MW). |
| `Storage_MWh` | Storage energy capacity (MWh). |
| `Storage_Hr` | Storage duration (hours). |
| `Generation` | Annualized generation (MWh). |
| `Curtail` | Annualized curtailment (MWh). |
| `Storage_Charge_MWh` | Annualized charging energy (MWh). |
| `Reserve_RegUp`, `Reserve_RegDn`, `Reserve_Spin`, `Reserve_NSpin`, `Reserve_FlexUp`, `Reserve_FlexDn` | Annualized reserve provision by product. |
| `Generation_Cost` | Generation operating cost (USD). |
| `Charge_Cost` | Charging cost (USD). |
| `Regulation_Cost`, `Spin_Cost`, `Nspin_Cost`, `Flex_Cost` | Reserve costs (USD). |
| `UnitRevenue_E` | Energy-market revenue (USD). |
| `UnitRevenue_AS` | Ancillary-service revenue (USD). |
| `UnitRevenue_CRED` | Credit-related or policy-related revenue (USD). |
| `UnitRevenue` | Total revenue (USD). |
| `UnitProfit` | Net revenue minus reported operating costs (USD). |
| `FuelConsumption` | Annualized fuel consumption. |
| `FuelCost` | Annualized fuel cost (USD). |
| `FOM` | Fixed O&M cost (USD). |
| `CAPCRED` | Capacity credit (fraction). |
| `Reference_Annual_Gen_Investment_Cost` | Reference annual generator investment cost basis (USD). |
| `Gen_Investment_Cost_Committed` | Present value of the whole investment payment stream from the linked expansion result (through the end of the horizon, limited by the cost-recovery period). |
| `Gen_ITC_Committed` | Present value of the investment tax credit from the linked expansion result, on the same whole-stream basis. |

### How to use and check

- `UnitProfit` and the revenue columns indicate whether a technology recovers its operating and fixed costs at modeled prices. `UnitProfit` is net of operating costs as reported; compare it with `Reference_Annual_Gen_Investment_Cost` for full cost recovery.
- Sum `Generation` by `Unit_Category` for the generation mix. The total should match `Generation` in the system summary.
- `ICAP` summed over units should match `Installed_Capacity_MW` in the system summary.

## 7. System summary outputs

### Files
- `<Case_ID>__system_summary_by_stage_OP.csv`: one row per planning stage.
- `<Case_ID>__system_summary_by_year_OP.csv`: one row per year, with present-value costs (`*_PV`).
- `<Case_ID>__system_summary_by_year_OP_real_<dollar_year>usd.csv`: one row per year, with undiscounted costs (`*_real`).

`<dollar_year>` is `dollar_year_value` from the `Planning Design` sheet (for example, `2022` gives `..._real_2022usd.csv`). In these files, numeric values below `1e-3` are written as `0`; `PRM` and `CAPCRED` are exempt.

!!! note "Planning reserve margin (`PRM`) definition"
    `PRM` is reported as `credited UCAP / coincident peak demand - 1`. Credited UCAP is the sum, over units in service, of installed capacity times capacity credit, in MW. The denominator is the system coincident peak demand: regional hourly load shapes are summed first and the annual maximum is taken from the combined series, with per-region load growth applied. The peak is taken from the full 8760-hour load series, not from the representative days, so scenario reduction does not affect it. Because the coincident peak is never larger than the sum of the regional peaks, this value is lower than a sum-of-regional-peaks (non-coincident) basis. Within a multi-year stage, the peak is re-evaluated at each year's own load growth, so the annual `PRM` can change from year to year within a stage.

!!! note "`TnD_Loss` is evaluated at the stage start year"
    The demand used for `TnD_Loss` is evaluated once per stage at the stage's `Start_Year` and reused for every year of the stage. In a multi-year stage, the annual `TnD_Loss` therefore repeats for each year, and the stage `TnD_Loss` equals the stage length times that one value.

### Planning-stage summary columns

!!! note "Paid-in-stage versus committed costs"
    `Gen_Investment_Cost`, `Gen_ITC`, `Trans_Investment_Cost`, `Generation_PTC`, and `total_system_cost` hold what is paid within the stage; they sum exactly to the corresponding `*_PV` columns of that stage in the annual file. The `*_Committed` columns (`Gen_Investment_Cost_Committed`, `Gen_ITC_Committed`, `Trans_Investment_Cost_Committed`, `Generation_PTC_Committed`, at the end of the file) hold the present value of the whole payment stream, through the end of the horizon and limited by the cost-recovery period, of the units and lines decided in that stage. Totals over all stages are identical either way.

    Example: with two stages of two years, stage-1 builds are paid over 2025-2028. The stage-1 row shows only the 2025-2026 payments, while the `*_Committed` columns show all four years.

| Column | Meaning |
|---|---|
| `Case_ID`, `Stage` | Index columns. |
| `Start_Year`, `Years_in_Stage` | Stage start year and length. |
| `Discount_Year` | Anchor year (`dollar_year_value`) to which all discounted costs in the file refer. |
| `Gen_Investment_Cost` | Discounted generator investment cost paid within the stage (from the expansion result). |
| `Gen_ITC` | Discounted investment tax credit received within the stage (from the expansion result). |
| `Trans_Investment_Cost` | Discounted transmission investment cost paid within the stage (from the expansion result). |
| `Trans_FOM_Cost` | Discounted transmission fixed O&M on the full in-service grid (existing plus expanded), charged every operating year of the stage. It is `0` when `transmission_FOM_percent_value` is `0` (the default). It is included in `total_system_cost` but not in `Operating_Cost`. See [GTEP Transmission Expansion](../GTEP/GTEP_Transmission_Expansion.md#transmission-fixed-om-fom). |
| `Gen_Retirement_Cost` | Discounted generator retirement cost (from the expansion result). |
| `FOM_Cost` | Discounted fixed O&M cost. |
| `Generation_PTC` | Discounted production tax credit received within the stage. |
| `Fuel_Cost`, `VOM_Cost`, `Commitment_Cost` | Discounted fuel, variable O&M, and commitment costs. |
| `Regulation_Cost`, `Spin_Cost`, `Nspin_Cost`, `Flex_Cost` | Discounted reserve costs. |
| `ENS_Cost` | Discounted cost of energy not served. |
| `RNS_Spin_Cost`, `RNS_NSpin_Cost`, `RNS_Flex_Cost` | Discounted reserve shortage costs. |
| `CTAX_cost` | Discounted carbon-tax cost. |
| `CEGT_Penalty` | Discounted clean-energy target penalty. |
| `RPS_Penalty` | Discounted renewable portfolio standard penalty. |
| `total_system_cost` | Total discounted cost paid within the stage, including carried expansion-related terms. |
| `Operating_Cost` | Operation-only total: total system cost excluding investment, retirement, and all fixed O&M (generation and transmission). |
| `ObjectiveValue` | Solved objective value (see the warning below). |
| `Generation` | Annualized system generation (MWh). |
| `TnD_Loss` | Transmission and distribution loss energy (MWh) for the stage: annual nominal demand (full 8760-hour load series) times `transmission_loss_percent_value * 0.01`, summed over the stage's years. `0` when `enforce_transmission_loss_flag` is not `TRUE`. |
| `Storage_Charge_MWh` | Annualized storage charging energy (MWh). |
| `Reserve_RegUp`, `Reserve_RegDn`, `Reserve_Spin`, `Reserve_NSpin`, `Reserve_FlexUp`, `Reserve_FlexDn` | Annualized reserve provision by product. |
| `ENS` | Annualized energy not served (MWh). |
| `RNS_Spin`, `RNS_NSpin`, `RNS_Flex` | Annualized reserve shortages by product. |
| `Emission` | Annualized emissions. |
| `PRM` | Planning reserve margin (see the definition above). |
| `Peak_Demand_MW`, `UCAP_MW` | Coincident peak demand and credited UCAP behind `PRM` (MW). |
| `Installed_Capacity_MW` | Installed capacity in service (MW). |
| `Annual_Input_Demand_MWh` | Annual demand from the input data for the stage's dispatched year, times the stage length. |
| `Dispatched_Load_MWh` | Load actually dispatched: the representative-day load weighted by days represented, times the stage length. |
| `Gen_Investment_Cost_Committed`, `Gen_ITC_Committed`, `Trans_Investment_Cost_Committed`, `Generation_PTC_Committed` | Whole-payment-stream present values of the units and lines decided in the stage. |

`Dispatched_Load_MWh` differs from `Annual_Input_Demand_MWh` because a small number of representative days does not reproduce annual energy exactly (for example, +5.8% in one test case). Generation, costs, and emissions follow the dispatched load; the load is not rescaled to the annual demand.

!!! warning "`ObjectiveValue` is scaled and carried expansion terms have limits"
    `ObjectiveValue` is reported multiplied by `per_unit_econ_base_value` (`Simulation Setting` sheet), so its magnitude depends on that setting and must not be compared across runs that used different values. The scaling does not change decisions. When the operation run follows a myopic multi-round expansion, the carried `Gen_Investment_Cost`, `Trans_Investment_Cost`, and `Gen_Retirement_Cost` are the round-local expansion figures, so summing them across stages is not a horizon net present value. See the `ObjectiveValue` warnings in [GTEP Expansion Outputs: System Summary](../GTEP/GTEP_Expansion_Outputs.md#11-system-summary-output).

### Annual summary columns

The annual file has one row per year. The columns `ObjectiveValue` and `Number of Years` are not written.

!!! note "The annual `*_PV` cost columns are discounted"
    Every `*_PV` cost column is discounted to `dollar_year_value` (recorded in `Discount_Year`). Each year's cost is multiplied by `(1 + discount_rate)^-(year - dollar_year_value)`, with the real `discount_rate` taken from `discount_rate_value` on the `Planning Design` sheet. The values are present values, not the cash cost incurred in each year. For the cost incurred each year, use the [constant-dollar annual file](#constant-dollar-annual-file).

| Column | Meaning |
|---|---|
| `Case_ID`, `Stage` | Index columns. |
| `Year` | Actual year inside the stage. |
| `Discount_Year` | Discount anchor year (`dollar_year_value`). |
| `Gen_Investment_Cost_PV`, `Gen_ITC_PV`, `Trans_Investment_Cost_PV` | Annual generator investment, investment tax credit, and transmission investment (present value). |
| `Trans_FOM_Cost_PV` | Annual transmission fixed O&M on the full in-service grid. Included in `total_system_cost_PV`, not in `Operating_Cost_PV`. |
| `Gen_Retirement_Cost_PV`, `FOM_Cost_PV`, `Generation_PTC_PV` | Annual retirement cost, fixed O&M, and production tax credit. |
| `Fuel_Cost_PV`, `VOM_Cost_PV`, `Commitment_Cost_PV` | Annual fuel, variable O&M, and commitment costs. |
| `Regulation_Cost_PV`, `Spin_Cost_PV`, `Nspin_Cost_PV`, `Flex_Cost_PV` | Annual reserve costs. |
| `ENS_Cost_PV` | Annual cost of energy not served. |
| `RNS_Spin_Cost_PV`, `RNS_NSpin_Cost_PV`, `RNS_Flex_Cost_PV` | Annual reserve shortage costs. |
| `CTAX_cost_PV`, `CEGT_Penalty_PV`, `RPS_Penalty_PV` | Annual carbon-tax cost and policy penalties. |
| `total_system_cost_PV` | Annual total system cost. |
| `Operating_Cost_PV` | Annual operation-only total system cost. |
| `Generation` | Annual generation (MWh). |
| `TnD_Loss` | Annual transmission and distribution loss energy (MWh), on the same basis as the stage column but for a single year; repeats across the years of a stage. |
| `Storage_Charge_MWh` | Annual storage charging energy (MWh). |
| `Reserve_RegUp`, `Reserve_RegDn`, `Reserve_Spin`, `Reserve_NSpin`, `Reserve_FlexUp`, `Reserve_FlexDn` | Annual reserve provision by product. |
| `ENS` | Annual energy not served (MWh). |
| `RNS_Spin`, `RNS_NSpin`, `RNS_Flex` | Annual reserve shortages by product. |
| `Emission` | Annual emissions. |
| `PRM` | Annual planning reserve margin. |
| `Peak_Demand_MW`, `UCAP_MW`, `Installed_Capacity_MW` | Coincident peak demand, credited UCAP, and installed capacity (MW). |
| `Annual_Input_Demand_MWh`, `Dispatched_Load_MWh` | Input-data annual demand and dispatched (representative-day weighted) load of the year. |

Only the first year of a stage is dispatched, so the physical quantities (`Peak_Demand_MW`, `UCAP_MW`, `Installed_Capacity_MW`, `Annual_Input_Demand_MWh`, `Dispatched_Load_MWh`) are replicated across the years of a stage.

### Constant-dollar annual file

`<Case_ID>__system_summary_by_year_OP_real_<dollar_year>usd.csv` accompanies the annual file and is written whenever the annual file is written. It has the same columns as the annual file, with these differences:

- **Cost columns are undiscounted and labeled `*_real`** instead of `*_PV`. Each `*_PV` value is multiplied back by `(1 + discount_rate)^(year - dollar_year_value)`, giving the real cost incurred in that year in constant `dollar_year_value` dollars. This applies to `total_system_cost_real` and to `Operating_Cost_real`, each of which remains the sum of its `*_real` components (with `Gen_ITC_real` and `Generation_PTC_real` as credits).
- **`Discount_Year` is not written**, because the values are no longer discounted.
- **A new last column, `System_Cost_per_MWh_real`,** equals `total_system_cost_real / Dispatched_Load_MWh` (`0` when `Dispatched_Load_MWh` is not positive), so cost and energy come from the same dispatch.
- **All other columns are unchanged.** Physical quantities (`Generation`, `Emission`, `PRM`, `TnD_Loss`, the `Reserve_*` columns, `ENS`, `RNS_*`, `Storage_Charge_MWh`, `Dispatched_Load_MWh`, and so on) are identical to the annual file.

!!! note "Constant-dollar values do not depend on `dollar_year_value`"
    The annual file discounts by `(1 + r)^-(Y - base)` and this file removes that factor using the same `base = dollar_year_value`, so the base cancels. The `*_real` values equal the real costs in the input data regardless of the anchor year. Changing `dollar_year_value` changes only the discount level of the annual file and the `<dollar_year>` in the file name.

!!! warning "`dollar_year_value` is a discount anchor, not an inflation adjustment"
    `dollar_year_value` sets only the anchor year for discounting. It does not restate costs in another year's purchasing power, because the input cost tables are assumed to be in `dollar_year_value` dollars already. Setting it to `2025`, for example, would label the file `..._real_2025usd.csv` while the values remain in the original purchasing power. Change it only when the cost inputs are genuinely in that year's dollars.

### How to use and check

- Compare `total_system_cost` in the stage file with the sum of `total_system_cost_PV` over that stage's years in the annual file. They should agree.
- `ENS`, `ENS_Cost`, and the `RNS_*` columns should be zero or negligible unless load shedding or reserve shortage is intended. Non-zero values indicate an inadequate system in the case.
- `PRM` gives a quick adequacy screen against the target reserve margin.
- Use `System_Cost_per_MWh_real` to compare average system cost across stages, cases, or years.

## 8. Representative-day selection output

### File
`<Case_ID>__representative_days_OP.csv`. Each row is one representative day.

### Columns

| Column | Meaning |
|---|---|
| `Case_ID`, `Stage`, `Year` | Index columns. |
| `Rep_Day` | Representative-day index. |
| `Days_Represented` | Number of full-year days represented by this representative day. |
| `Day_Group_ID` | Day group to which the representative day belongs. |
| `Days_in_Group` | Total number of full-year days in the day group. This differs from `Days_Represented`, which is the weight of this representative day alone. |
| `Day_of_Year` | Calendar day of the year mapped to the representative day. |

Use this file to translate `Rep_Day` in the hourly files into a calendar day, and to confirm that the weights `Days_Represented` sum to the number of days in the year (365 or 366). See [Scenario Reduction and Representative-Day Groups](../../configuration/Scenario_Reduction_and_Repday_Groups.md).

## 9. Water management output

### File
`<Case_ID>__water_management_OP.csv`, written only when reservoir water management is modeled.

### Columns
`Stage`, `Year`, `Rep_Day`, `Hour`, `Sub_Period`, `Segment`, `Unit_ID`, `Bus_ID`, `Unit_Report_Label_1`, `Unit_Report_Label_2`, `Water_Use`. `Water_Use` is the water use of the reservoir unit in a given segment, in the model's water units.

## 10. Run-time output

### File
`<Case_ID>__simulation_run_time.csv`

### Columns

| Column | Meaning |
|---|---|
| `Case_ID` | Case name. |
| `Expansion_Run_Time_s` | Expansion solve time (seconds). |
| `Operation_Run_Time_s` | Operation solve time (seconds). |
| `RA_Run_Time_s` | Resource adequacy run time (seconds); `0` when not run. |

## Suggested review order

For a compact review of an operation run, inspect these files in order:

1. `<Case_ID>__system_summary_by_stage_OP.csv`: costs, generation, `ENS`, `PRM`.
2. `<Case_ID>__tech_summary_by_stage_OP.csv`: generation and economics by unit.
3. `<Case_ID>__market_OP.csv`: prices and scarcity.
4. `<Case_ID>__dispatch_OP_year_<stage>.csv`: hourly dispatch and storage operation.
5. `<Case_ID>__power_flow_OP.csv`: congestion.
6. `<Case_ID>__representative_days_OP.csv`: day weights and calendar mapping.

## Related documentation

- [Operation Overview](./Operation_Overview.md)
- [Operation Formulation](./Operation_Formulation.md)
- [Operation Execution and Run Modes](./Operation_Execution_and_Run_Modes.md)
- [GTEP Expansion Outputs](../GTEP/GTEP_Expansion_Outputs.md)
