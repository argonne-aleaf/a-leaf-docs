# Gen Technology Reference

The `Gen Technology` sheet of the simulation setting workbook is the master technology-definition table in A-LEAF. Each row describes one technology, either an existing technology that plants in the network workbook refer to, or a candidate technology that the expansion model may build. The columns control how the technology is represented in:

- expansion planning,
- production cost simulation,
- reliability assessment,
- ELCC and other post-processing workflows.

!!! note "ELCC, in one line"
    ELCC (Effective Load Carrying Capability) is the reliability-adjusted capacity credit assigned to a resource: roughly, how much of its nameplate capacity can be counted on to help meet peak and reliability needs. It is computed by the RA model and can feed back into GTEP's investment accounting; see [RA Metrics, ELCC, and DLOL](../models/RA/RA_Metrics_ELCC_and_DLOL.md).

The sheet defines technology identity, cost structure, operating behavior, reliability treatment, and investment options, so it is one of the most important input tables in the workbook. Read a row as a technology contract: the left columns say what the technology is, the middle columns define how it behaves operationally and financially, and the right columns define planning, policy and reliability behavior.

## How the sheet is organized

Row 1 holds group banners and row 2 holds the column names; technology rows start in row 3. The groups are:

| Group | Purpose |
|---|---|
| Category | Identity and fuel |
| Dispatch and Commitment Options | Dispatch type, unit commitment, fuel limit, must-run |
| Capacity and Energy | Size, storage duration, output limits |
| Financial | Capital, fixed and variable cost, fuel, start-up and reserve costs |
| Technical Characteristics | Life, heat rate, ramping, efficiency, emissions, inertia |
| Classification, Market and Policy | Variable renewable and hydro classification, profile type, policy eligibility |
| Reliability | Capacity credit, ELCC, forced outage |
| Investment Options | Build and retirement decisions and limits |
| Resource Limits | Regional supply-curve limits and cost scaling |
| ETC | Technology type (`Existing` or `New`) |

The `plant` sheet of the network workbook can refer to a technology row: a `plant` cell that contains the text `Gen_Tech` takes its value from the `Gen Technology` row with the same `UNITGROUP`, and any other value in the plant row overrides the technology default for that plant. See [Network Data Reference](./Network_Data_Reference.md).

### Cost values that point to other tables

Several financial columns accept a number or a reference to another table:

| Reference | Meaning |
|---|---|
| `ATB` | Read the value from the technology cost data selected in the `ATB Setting` sheet (`data/common/ATB_2024.csv`). |
| `ESGC` | Read the value from the `Storage Cost and Performance` sheet for the selected `ESGC_Setting_ID`. |
| `Fuel` | Use the system-wide fuel price of the technology's `FUEL` from the fuel-price file. |
| `Fuel-Regional` | Use the regional fuel price of the technology's `FUEL`. |
| `Water Value` | Use the water value of the reservoir (hydro with a water budget). |

A number is used as entered.

## Technology identity

### `Tech_ID`
Unique identifier of the technology row (for example `New005` for candidates and `Existing138` for existing technologies).

### `UNITGROUP`
Core technology group name used throughout A-LEAF. It is the main cross-sheet join key, linking the technology to:

- the `ATB Setting` sheet (`ATB` is the NREL Annual Technology Baseline cost and performance data set),
- the `Storage Cost and Performance` sheet,
- the `ITC` and `PTC` policy tables,
- capacity-credit and ELCC logic in reliability assessment,
- the `plant`, `hybrid` and `demand` sheets of the network workbook, which refer to technologies by `UNITGROUP`.

Use identical spelling of each `UNITGROUP` in every sheet. A mismatch can break cost lookups, storage assumptions, policy application, ELCC eligibility, and resource limits.

### `UNIT_CATEGORY`
High-level technology class. Examples from the bundled data are `THERMAL`, `NUCLEAR`, `STORAGE`, `PV`, `WIND_ONS`, `WIND_OFS`, `CSP`, and `OTHER`. The category affects reporting groups and some model logic (for example, storage rows use `STORAGE`).

### `UNIT_REPORT_LABEL_1` and `UNIT_REPORT_LABEL_2`
Free-text labels for reporting. They are copied to the output tables (`Unit_Report_Label_1` and `Unit_Report_Label_2` in the dispatch output; `Unit_Report_Label_1` and `Unit_Report_Label_2` in the demand-response output) and are used for both generator rows and `demand` (large-load) rows. No model behavior depends on them.

!!! warning "Do not use these for logic"
    Technology behavior is driven by the explicit classification columns `Profile_Type`, `VRE_Flag` and `Hydro_Flag` described below, not by the label text. Changing a label changes only how a technology is named in the results.

### `FUEL`
Primary fuel or resource label used for cost, emissions and policy mapping. The bundled data uses codes such as `NG`, `BIT`, `SUB`, `DFO`, `KER`, `NUC`, `WAT`, and `MWH`. Fuel prices are looked up by this code.

## Classification columns

These columns explicitly tell the model how to treat a technology.

### `Profile_Type`
Selects the hourly availability shape of the technology. Recognized values:

| `Profile_Type` | Hourly shape used |
|---|---|
| `wind_ons` | Onshore wind profile |
| `wind_ofs` | Offshore wind profile |
| `pv` | Utility-scale solar PV profile |
| `rtpv` | Rooftop solar PV profile |
| `hydro` | Hydro profile |
| `csp` | Concentrating solar power profile |
| `NA` | No profile; the technology is not shaped |

The profiles are read from the time-series files listed in the `File Path` sheet of the network workbook. The same `Profile_Type` field selects the shape of the onsite generator of hybrid resources and large flexible loads.

### `VRE_Flag`
TRUE classifies the technology as variable renewable. For a variable renewable technology the model allows curtailment, and for `FUEL_LIMIT` = `Fixed Profile` it limits the hourly output to the capacity multiplied by the hourly shape. This flag, not `UNIT_CATEGORY` or any label, distinguishes variable renewable technologies in the operation, expansion and reliability models.

### `Hydro_Flag`
Classifies hydro and pumped-storage behavior. Use FALSE for non-hydro technologies.

| `Hydro_Flag` | Effect |
|---|---|
| `IMPOUNDMENT` | Reservoir hydro. The `FUEL_LIMIT` of the technology is set by the case-level `Reservoir_Hydro_Operation_Option`, and hydro flexibility applies. |
| `ROR` | Run-of-river hydro. Hydro flexibility applies. |
| `PSH` | Pumped-storage hydro. Separates pumped-storage candidates from batteries in the storage and pumped-storage analysis. |

For `ROR` and `IMPOUNDMENT`, the hydro flexibility rules apply only when `Hydro_Flexibility_Flag` in `Simulation Configuration` is TRUE; `Hydro_Flexibility_Percent` sets the band of output around the profile.

### `Clean_Energy_Flag`
TRUE counts the technology's generation as clean energy in the Clean Energy Generation Target. See [Policy and Financial Settings](../configuration/Policy_and_Financial_Settings.md).

### `ITC Flag` and `PTC Flag`
Technology-level eligibility for the investment tax credit and production tax credit. A technology receives a credit only when both the case-level flag (`ITC_Flag` or `PTC_Flag` in `Simulation Configuration`) and the technology-level flag are TRUE. The credit values come from the `ITC` and `PTC` sheets by `UNITGROUP` and year.

## Dispatch and commitment options

| Column | Values | Description |
|---|---|---|
| `Dispatch` | `Dispatchable`, `Non-dispatchable` | Dispatchable units can provide up-reserves and are dispatched within their limits. Non-dispatchable units follow their availability (within the hydro flexibility band) and do not provide up-reserves. |
| `Commitment` | TRUE, FALSE, `NA` | TRUE models unit commitment (on/off status, start-up, shut-down, no-load cost). FALSE dispatches the technology continuously. Use `NA` for storage. |
| `Storage Commitment` | TRUE, FALSE, `NA` | For storage. TRUE adds a binary charge/discharge status that prevents simultaneous charging and discharging; the status is reported as `Storage_Charging_Flag`. |
| `FUEL_LIMIT` | `Fixed Profile`, `Budget (day groups)`, `Budget (annual)`, `NA` | Energy limitation. `Fixed Profile` limits hourly output to capacity times the hourly shape (variable renewables and run-of-river). The budget options limit energy over day groups or over the year (reservoir hydro); the hourly bound is then nameplate times `PMAX`. `NA` means no energy limit. |
| `Must_Run_Flag` | TRUE/FALSE | TRUE forces the technology to operate at or above the must-run level whenever it is in service. |
| `Must_Run_Level` | fraction of `CAP` | Minimum output of a must-run technology. The level is capped at `PMAX`. |

## Capacity and energy

| Column | Units | Description |
|---|---|---|
| `CAP` | MW | Nameplate capacity of one unit block. New investment is a number of blocks of this size (see `MAXINVEST`). |
| `Charge_CAP` | MW | Charging capacity of a storage unit block. |
| `STOMIN` | MWh | Minimum stored energy (state-of-charge floor) of a storage unit. |
| `STOHR_MIN`, `STOHR_MAX` | hours | Minimum and maximum storage duration. For existing and hybrid storage, `STOHR_MAX` gives the energy capacity as `Charge_CAP` times `STOHR_MAX`. For candidate storage with `ES_STO_INVEST_FLAG` TRUE, the model chooses a duration between these bounds. |
| `PMAX` | fraction of `CAP` | Maximum output. |
| `PMIN` | fraction of `CAP` | Minimum stable output when the unit is committed. |
| `FOR` | fraction | Forced outage rate. It is not used directly by the model; reliability assessment uses `RA_FOR`. |

## Financial

Cost values are in the dollar year set by `dollar_year_value` in the `Planning Design` sheet.

| Column | Units | Description |
|---|---|---|
| `CAPEX` | $/kW | Overnight capital cost of power capacity. Number, `ATB` or `ESGC`. |
| `STO_CAPEX` | $/kWh | Capital cost of storage energy capacity. Number, `ATB` or `ESGC`. |
| `CAPEX_Scale` | multiplier | Present in the schema. The scaling of `ATB` and `ESGC` capital costs is taken from the `CAPEX_Scale` column of the `ATB Setting` and `Storage Cost and Performance` sheets. |
| `FCR` | fraction | Fixed charge rate, the factor that converts capital cost into an annual charge. Number, `ATB` or `NA`. |
| `CRP` | years | Capital recovery period. Number or `ATB`. |
| `RETC` | | Present in the schema. It is not used by the current model. |
| `DECC` | k$/MW | Decommissioning cost applied to retired capacity. |
| `FOM` | $/kW-year | Fixed operation and maintenance cost. Number, `ATB` or `ESGC`. |
| `VOM` | $/MWh | Variable operation and maintenance cost. Number, `ATB` or `ESGC`. |
| `FC` | $/MMBtu or $/MWh | Fuel cost. Number, `Fuel`, `Fuel-Regional`, `Water Value` or `ATB`. The marginal cost of a thermal unit is the heat rate multiplied by the fuel price plus `VOM`. |
| `NLC`, `SUC`, `SDC` | $/MW | No-load, start-up and shut-down costs (used with `Commitment` TRUE). |
| `reg_cost`, `spin_cost`, `nspin_cost`, `flex_cost` | $/MWh or fraction of marginal cost | Cost of providing regulation, spinning, non-spinning and flexibility reserves. The interpretation follows `reserve_cost_type_flag` in `Planning Design` (`absolute` uses dollars). |

## Technical characteristics

| Column | Units | Description |
|---|---|---|
| `Life` | years | Technical life. Sets the retirement year of a new unit (online year plus life minus one) and the horizon of investment accounting. |
| `HR` | MMBtu/MWh | Heat rate. |
| `Ramp` | fraction of `CAP` per minute | Ramp rate. The hourly ramp limit is the smaller of 1 and 60 times this value. |
| `RUL`, `RDL` | fraction of `CAP` | Ten-minute ramp limits up and down, used for reserve headroom. |
| `MAXR` | fraction of `CAP` | Five-minute ramp limit, used for regulation reserve. |
| `MAXC` | fraction of `CAP` | Maximum contingency-reserve contribution. |
| `EMSFAC` | | Present in the schema. It is not used by the current model; emissions are calculated from the `Emission_*` columns. |
| `BATEFF` | fraction | Round-trip efficiency of storage (one-way efficiency is its square root). Number or `ESGC`. |
| `AET` | MWh per year | Annual energy throughput limit of a storage unit. Number or `ESGC`. |
| `Emission_CO2` | tonne/MWh | CO2 emission factor. |
| `Emission_1`, `Emission_2`, `Emission_3` | tonne/MWh | Additional user-defined emission factors. |
| `Power_Factor` | | Power factor used together with `Inertia_Constant` to convert committed capacity into an inertia contribution. |
| `Inertia_Constant` | seconds | Inertia constant used in the system inertia requirement. |

## Reliability

### `CAPCRED`
Capacity credit of the technology: the fraction of `CAP` that counts toward the planning reserve margin. The reliability-derated capacity is `UCAP = CAP × CAPCRED`; see the [planning reserve margin constraint](../models/GTEP/GTEP_Formulation.md#planning-reserve-margin-reliability). The field can be:

- a number between 0 and 1, used directly, or
- a text key such as `pv`, `wind_ons` or `wind_ofs`. The key selects the matching `CAPCRED_<key>` column of the `Network Data Level` sheet of the configuration workbook, so the capacity credit varies by region. This requires a configuration workbook.

Both planning and reliability assessment read this field. From the second planning year onward, when `multi_round_solution_process_flag` (`Planning Design`) and `update_CAPCRED_in_each_round_of_Expansion_Flag` (`Simulation Configuration`) are both TRUE and the technology's `ELCC_Flag` is TRUE, the expansion model replaces `CAPCRED` with the ELCC result calculated by the reliability model for that year.

### `ELCC_Flag`
TRUE makes the technology eligible for ELCC calculation and for capacity-credit updates from the reliability model.

### `RA_FOR`
Forced-outage representation used by the reliability assessment. It can be a number (a forced outage rate) or the name of a unit type defined in the outage data of the reliability model (the bundled data uses `GasCC`, `GasCT`, `GasST`, `CoalST` and `Nuclear`). See [RA Scenarios and Data](../models/RA/RA_Scenarios_and_Data.md).

## Investment options

| Column | Values | Description |
|---|---|---|
| `INVEST_FLAG` | TRUE/FALSE | TRUE allows new investment in the technology. FALSE fixes it to zero, so existing technologies keep FALSE. |
| `ES_STO_INVEST_FLAG` | TRUE/FALSE | For storage: TRUE lets the model choose the duration of new units; FALSE fixes the duration. |
| `MAXINVEST` | unit blocks | Maximum number of unit blocks (of size `CAP`) that can be built at one bus. |
| `MININVEST` | unit blocks | Minimum number of unit blocks per bus when investment occurs. |
| `SYSTEM_MAXINVEST`, `SYSTEM_MININVEST` | unit blocks | Bounds on the total new investment in the technology across all buses and the whole planning horizon. Each bound is active only when its value is greater than 0. |
| `RET_FLAG` | TRUE/FALSE | TRUE allows economic retirement before the scheduled retirement. |
| `MINRET` | | Present in the schema. It is not used by the current model. |
| `Land_Use` | | Present in the schema. It is not used by the current model. |
| `Integrality` | TRUE/FALSE | TRUE makes new investment integer (whole unit blocks). FALSE allows fractional investment and keeps the problem continuous. Duals and prices require a continuous problem. |
| `Material_Flag` | TRUE/FALSE | TRUE includes the technology in raw-material limits; the intensities are in `Gen Technology Raw Materials`. |

## Resource limits

These columns connect technologies to regional build limits and cost variations.

| Column | Values | Description |
|---|---|---|
| `Resource_Limit_Flag` | TRUE/FALSE | TRUE applies a regional capacity limit to the technology. |
| `Resource_Limit_ID` | text | Resource key that links technologies to the limit columns of the `Network Data Level` sheet. The key `pv` selects `Resource_Limit_pv_highcap` and `Resource_Limit_pv_lowcap` (MW). The case setting `Resource_limit_level_value` (`High` or `Low`) chooses which of the two applies. Technologies with the same key share the limit. Use `NA` when no limit applies. |
| `Locational_Scaling_Flag` | TRUE/FALSE | TRUE applies regional capital-cost scaling from the `Network Data Level` sheet. |

The limits work together with the regional data in the configuration workbook and with the case-level settings `Regional_resource_limits_flag`, `Resource_limit_level_value` and `Regional_CAPAX_scaling_flag` in `Simulation Configuration`.

## ETC

| Column | Values | Description |
|---|---|---|
| `Tech_Type` | `Existing`, `New` | Marks whether the row describes an existing technology (referenced by plants) or a candidate technology. Pumped-storage and other analyses use it to identify candidates. |

## Interaction with other sheets

### `ATB Setting`
Maps a `UNITGROUP` to the ATB technology (`Tech`), detail (`TechDetail`), financial case (`Case`), recovery period (`CRP`), scenario, year (`ATB Year`) and `CAPEX_Scale`, for a named `ATB_Setting_ID`. Technologies with cost columns set to `ATB` read capital cost, fixed O&M and, for some technologies, variable O&M from the ATB file for this configuration.

!!! note "`TechDetail` must match the ATB file exactly"
    In the bundled ATB file (`data/common/ATB_2024.csv`), the technology row must match the ATB columns `technology_alias` (from `Tech`), `techdetail` (from `TechDetail`), `core_metric_case` (from `Case`), `crpyears` (from `CRP`), `scenario` and the ATB year. A `TechDetail` that matches no row yields no value, and the corresponding CAPEX, FOM or VOM is `0.0`. Confirm each `TechDetail` against the `techdetail` values in the ATB file.

### `Storage Cost and Performance`
Provides storage cost and performance data by `ESGC_Setting_ID` and `UNITGROUP`. See [Storage Modeling Reference](./Storage_Modeling_Reference.md).

### `ITC` and `PTC`
Provide year-specific tax-credit values for eligible technologies.

### `Simulation Configuration`
Selects the ATB setting, ESGC setting, policy flags and other case options that apply to the technologies in a run.

### `RA Setting` and `RA Scenarios`
Use the reliability fields of this sheet to decide which technologies take part in reliability assessment, ELCC calculation and renewable adequacy treatment.

## Practical reading guide

When reviewing a technology row, check the fields in this order:

1. Identity: `Tech_ID`, `UNITGROUP`, `UNIT_CATEGORY`, `UNIT_REPORT_LABEL_1`, `UNIT_REPORT_LABEL_2`, `FUEL`.
2. Behavior classification: `Profile_Type`, `VRE_Flag`, `Hydro_Flag`.
3. Operating behavior: `Dispatch`, `Commitment`, `FUEL_LIMIT`, ramping and reserve columns.
4. Physical behavior: capacity, duration, efficiency, emissions.
5. Cost sources: numbers versus references such as `ATB`, `ESGC` or `Fuel`.
6. Reliability and policy fields.
7. Investment and resource-limit controls.

## Related documentation
- [ALEAF Simulation Setting File Reference](../configuration/ALEAF_Simulation_Setting_File.md)
- [Simulation Configuration Reference](../configuration/Simulation_Configuration_Reference.md)
- [Storage Modeling Reference](./Storage_Modeling_Reference.md): storage technologies (`UNIT_CATEGORY` = `STORAGE`)
- [Hybrid Resources Reference](./Hybrid_Resources_Reference.md): hybrid generator and storage pairs
- [RA Overview](../models/RA/RA_Overview.md)
