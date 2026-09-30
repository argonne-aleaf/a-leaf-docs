# Network Data Reference

This page describes the two network-related input workbooks of A-LEAF and the sheets and columns of the base network workbook:

- the base network data workbook, which defines the physical system,
- the optional network configuration workbook, which defines how that system is grouped, aggregated and mapped into planning, reserve, policy and resource zones.

Use this page as the schema reference when preparing or editing a network database. The configuration workbook is described in detail in the [Network Configuration Reference](./Network_Configuration_Reference.md).

## Required and optional inputs

### Base network data workbook
The base network data workbook is required. It is selected with `Network_Data_File_ID` in the `Simulation Configuration` sheet and must be named:

`network_<test_system_name>_<Network_Data_File_ID>.xlsx`

It sits in `data/<test_system_name>/` together with the time-series folder. If the file is missing, the run stops with an error naming the expected file.

### Network configuration workbook
The configuration workbook is optional. It is selected with `Network_Configuration_File_ID` in the `Simulation Configuration` sheet and must be named:

`config_<test_system_name>_<Network_Configuration_File_ID>.xlsx`

If the field is blank, missing or `NA`, A-LEAF runs without a configuration workbook. The base workbook is therefore the required physical-system input, and the configuration workbook is an optional overlay for aggregation, zoning and region mapping.

### Run without a configuration workbook
Without a configuration workbook, A-LEAF keeps the original network topology and creates minimal region information. Then:

- every bus remains at its original resolution,
- plants, hybrids and demand entries stay at their original buses,
- the whole system is one planning reserve region, one operating reserve region and one policy region,
- resource limits, cost scaling and capacity credits use neutral system-wide defaults.

A configuration workbook is needed for regional aggregation, multiple planning or reserve zones, policy-region mapping, regional resource limits and capacity credits, and other zone-level overlays.

### Run with a configuration workbook
With a configuration workbook, A-LEAF uses it to define the modeled footprint, map original buses into aggregated regions, create planning, reserve and policy zones, attach regional policy targets and resource information, and define the data regions used for time-series aggregation and capacity-credit lookup.

## 1. Base network data workbook

A-LEAF reads these sheets of the base workbook:

| Sheet | Content |
|---|---|
| `bus` | Network nodes and their base load. |
| `plant` | Existing generation and storage plants. |
| `branch` | Transmission lines and ties. |
| `demand` | Large flexible loads and demand response. |
| `hybrid` | Hybrid generator-plus-storage sites. |
| `File Path` | Locations of the time-series files. |

The workbook may contain other sheets (for example `System Data ->` and `ETC`); they are not read by the model.

For every data sheet, row 1 is a group banner and the column names are in row 2, with data starting in row 3. Keep the column names exactly as documented, including capitalization and spaces.

### `bus`
Defines the network nodes.

| Column | Units | Description |
|---|---|---|
| `bus_i` | | Bus identifier, referenced by `bus_ID` (`plant`, `hybrid`, `demand`), `f_bus` and `t_bus` (`branch`) and by the column names of the time-series files. In the bundled data it encodes the region, for example `ERCO_NCEN_US-TX`. |
| `bus_name` | | Descriptive name. |
| `MW load` | MW | Base (peak) load at the bus. The hourly load is this value scaled by the load profile of the bus. |
| `TimeSeriesStatus` | TRUE/FALSE | Informational flag on the availability of a time series for the bus. |
| `Notes` | | Free text. |
| `full  bus name` | | Long descriptive name (the column name contains two spaces). Informational. |

`Longitude` and `Latitude` columns are optional and informational.

The bus list is the starting point for topology and zonal aggregation. Every `bus_i` must appear in the `sub_area_mapping` sheet of the configuration workbook when a configuration workbook is used.

### `plant`
Defines existing generation and storage plants. One row is one plant.

Plants with `CAP` equal to 0 are removed when the network data is prepared.

#### Identity and location

| Column | Description |
|---|---|
| `PLANT_NAME` | Plant name. |
| `bus_ID` | Bus (`bus_i`) where the plant connects. |
| `bus_name` | Bus name. |
| `RetireYear` | Last year of operation. Use a far-future year (such as 2099) for no planned retirement. |
| `Online_Year` | First year of operation. |
| `UNITGROUP` | Technology group; must match a `UNITGROUP` in the `Gen Technology` sheet. |
| `UNIT_CATEGORY` | Category, for example `THERMAL`, `NUCLEAR`, `PV`, `WIND_ONS`, `WIND_OFS`, `CSP`, `HYDRO_ROR`, `HYDRO_IMPOUNDMENT`, `STORAGE`, `OTHER`. |
| `UNIT_REPORT_LABEL_1`, `UNIT_REPORT_LABEL_2` | Free-text labels copied to the outputs. They have no effect on the model. |
| `FUEL` | Fuel code. |
| `PLANT_ORIS_ID`, `Unit_Code`, `Generator_ID` | Optional identifiers. |
| `Longitude`, `Latitude` | Optional location. |

#### Capacity and energy

| Column | Units | Description |
|---|---|---|
| `CAP` | MW | Installed capacity of the plant. |
| `Charge_CAP` | MW | Charging capacity (storage). |
| `ES_MWh` | MWh | Stored-energy capacity (storage). |
| `STOMIN` | MWh | Minimum stored energy (storage). |
| `STOHR_MIN`, `STOHR_MAX` | hours | Minimum and maximum duration. |
| `PMAX`, `PMIN` | fraction of `CAP` | Maximum and minimum output. |
| `FOR` | fraction | Forced outage rate. |

The remaining columns (`Must_Run_Flag`, `Must_Run_Level`, `RETC`, `DECC`, `FOM`, `VOM`, `FC`, `NLC`, `SUC`, `SDC`, `reg_cost`, `spin_cost`, `nspin_cost`, `flex_cost`, `Life`, `HR`, `Ramp`, `RUL`, `RDL`, `MAXR`, `MAXC`, `EMSFAC`, `BATEFF`, `AET`, `Emission_CO2`, `Emission_1`, `Emission_2`, `Emission_3`, `Power_Factor`, `Inertia_Constant`, `VRE_Flag`, `Hydro_Flag`, `Profile_Type`, `Clean_Energy_Flag`, `ITC Flag`, `PTC Flag`, `CAPCRED`, `RA_FOR`) have the same meaning and units as in the [Gen Technology Reference](./Gen_Technology_Reference.md).

#### The `Gen_Tech` placeholder

Most technology columns in the `plant` sheet contain the text `Gen_Tech`. The value is then taken from the `Gen Technology` row with the same `UNITGROUP`. Entering a number or other value in a plant cell overrides the technology default for that plant only.

Worked example: a battery plant row with `UNITGROUP` = `MWH_BA_LIB`, `CAP` = 2, `Charge_CAP` = 2, `ES_MWh` = 12 and `BATEFF` = `Gen_Tech` is a 2 MW, 6-hour battery whose efficiency comes from the `MWH_BA_LIB` technology row. Setting `BATEFF` to 0.9 in the plant row would override the efficiency for this plant only.

Cost columns can also reference `ATB`, `ESGC`, `Fuel`, `Fuel-Regional` or `Water Value`, with the same meaning as in the technology sheet. Existing storage is described in more detail in [Storage Modeling Reference](./Storage_Modeling_Reference.md).

### `branch`
Defines transmission elements between buses.

| Column | Units | Description |
|---|---|---|
| `UID` | | Unique branch identifier. |
| `f_bus`, `t_bus` | | From and to bus (`bus_i`). |
| `rate_a` | MW | Thermal rating used as the flow limit. |
| `rate_c` | MW | Emergency rating. Not used by the model; the flow limit is `rate_a`. |
| `Length` | km | Branch length in kilometers. It scales the cost of transmission expansion (see below); the model converts it to miles before applying `transmission_cost_dollar_per_MW_mile_value`. |
| `model_flag` | TRUE/FALSE | TRUE includes the branch in the model. FALSE keeps the row in the file but removes it from the run. |
| `expansion_flag` | TRUE/FALSE | TRUE allows the expansion model to increase the rating of the branch. |
| `max_rate_a` | MW | Highest rating the branch can reach through expansion. It must exceed `rate_a` for expansion to be possible. |
| `br_x_pu` | per unit | Series reactance. Required for the `B-theta` and `PTDF` power-flow modes. |
| `br_r_pu` | per unit | Series resistance. Not used by the model, which has no line losses in the power-flow constraints. |
| `transformer` | TRUE/FALSE | Marks transformers. Informational. |
| `dc_line` | TRUE/FALSE | TRUE marks an asynchronous DC tie (see below). Optional: a sheet without this column treats every branch as an AC line. |
| `Notes` | | Free text. |

This sheet is used for both the operation and the expansion models. See [GTEP Transmission Expansion](../models/GTEP/GTEP_Transmission_Expansion.md) for how ratings and expansion are modeled in each power-flow mode.

!!! note "`br_x_pu` of aggregated corridors"
    For an aggregated AC corridor (parallel circuits between two aggregated buses merged into one branch), the reactance is first the parallel combination `1 / Σ(1/xᵢ)` of its members. In the `B-theta` and `PTDF` modes it is then replaced by a value estimated from synthetic operating snapshots, so that the aggregated network reproduces the cross-border flows of the detailed network. The input `br_x_pu` is used unchanged for DC ties, for lines kept separate at the finest resolution, and for runs without aggregation. See [GTEP Transmission Expansion: Corridor aggregation](../models/GTEP/GTEP_Transmission_Expansion.md#corridor-aggregation) and [Network Resolution](./Network_Resolution.md).

#### `dc_line` (asynchronous DC / VFT ties)

`dc_line` and its companion reactance column `br_x_pu` are optional branch columns. Where present (for example in `network_NorthAmerica_Base.xlsx`), `dc_line` is a boolean. When TRUE, the branch is an **asynchronous DC tie**: a back-to-back HVDC link or variable-frequency transformer (VFT) that transfers power between two interconnections without synchronizing their AC angles. When FALSE, blank or absent, the branch is a normal synchronous AC line.

A DC tie changes how the branch behaves in the `B-theta` power-flow mode:

- it is excluded from the DC angle equation `f = (1/br_x)·(θ_f − θ_t)`, so it does not force the two ends to a common angle reference,
- it is modeled as a controllable, bounded, lossless transfer (`−rate_a ≤ f ≤ rate_a·(1+u_T)`, where `u_T` is the expansion multiplier) that enters the nodal balance at both ends.

In `Network_Flow` mode, ties are transfer limits and behave as before. In `PTDF` (power transfer distribution factor) mode, a DC tie has a zero row of shift factors, so it injects no shift-factor coupling, the same angle-decoupled treatment it receives in `B-theta`, while keeping its own bounded-transfer limits. For the formulation and the per-island reference-bus consequence, see [GTEP Transmission Expansion](../models/GTEP/GTEP_Transmission_Expansion.md#asynchronous-dc-tie-modeling-b-theta).

!!! note "AC and DC corridors are never merged together"
    When parallel circuits between aggregated buses are merged, AC and DC lines are grouped separately by `dc_line` first. A DC tie running parallel to an AC line between the same bus pair is therefore never folded into the AC corridor, and a bus pair can produce one AC corridor and one DC corridor. This applies only to aggregated bus pairs. In a user-supplied finer-resolution database, lines whose ends are both at the finest level stay separate, each with its own `rate_a` and `br_x_pu`. See [Network Resolution](./Network_Resolution.md).

#### Cost fields and DC-tie expansion

To make a branch expandable, set `expansion_flag` = TRUE and give it headroom with `max_rate_a` greater than `rate_a`. A branch with `expansion_flag` = FALSE stays fixed. The cost basis of expansion differs by branch type and is set per case in the `Simulation Configuration` sheet:

- AC lines use `transmission_cost_dollar_per_MW_mile_value` ($/MW-mile). The overnight cost is `cost · rate_a · Length · transmission_route_length_adder_value`, with `Length` converted from kilometers to miles.
- DC ties use `dc_tie_expansion_cost_dollar_per_MW_value` ($/MW for one converter station) together with `transmission_cost_dollar_per_MW_mile_value`. The overnight cost is `(2 · dc_tie_expansion_cost_dollar_per_MW_value + transmission_cost_dollar_per_MW_mile_value · Length · transmission_route_length_adder_value) · rate_a`: two converter stations (one per side) plus an AC approach-line term that depends on the branch length. See [GTEP Transmission Expansion: DC-tie expansion cost](../models/GTEP/GTEP_Transmission_Expansion.md#expansion-cost-two-converter-stations-plus-ac-approach-line-distance) for the derivation and a worked example.

`transmission_route_length_adder_value` (`Planning Design` sheet, default 1.0) multiplies the straight-line length to approximate real routing. It scales the AC term but not the DC converter-station cost. Both bases are annualized with the same capital recovery factor (`transmission_investment_CRP_value` and `WACC_value`). Each expandable branch also carries an annual fixed O&M charge equal to the overnight cost times `transmission_FOM_percent_value` times 0.01 (default 0.0, meaning fixed O&M is off). See [Simulation Configuration Reference](../configuration/Simulation_Configuration_Reference.md#transmission_cost_dollar_per_mw_mile_value).

### `demand`
Defines large flexible loads and demand-response resources that are represented separately from the fixed bus load. Each entry has a size, an interconnection limit, a price/quantity curve, and optional onsite generation and storage. See [Large Load and Demand Response Reference](./Large_Load_and_Demand_Response_Reference.md) for the columns and their behavior.

### `hybrid`
Defines hybrid sites made of a generator and a storage unit behind one interconnection. See [Hybrid Resources Reference](./Hybrid_Resources_Reference.md) for the columns and their behavior.

### `File Path`
Lists the location of each time-series file, relative to the data folder of the test system. The sheet has the columns `Setting`, `Value` and `Note`, starting in row 2.

| `Setting` | Content |
|---|---|
| `timeseries_data_load_path` | Hourly load shape by bus or data region. |
| `timeseries_data_wind_ons_path`, `timeseries_data_wind_ofs_path`, `timeseries_data_pv_path`, `timeseries_data_rtpv_path`, `timeseries_data_hydro_path`, `timeseries_data_csp_path` | Hourly availability shapes of the corresponding `Profile_Type`. |
| `timeseries_data_temperature_path` | Hourly temperature. |
| `timeseries_data_fuel_price_path` | Fuel prices by region, fuel, year and month. |
| `reserve_requirement_data_reg_up_path`, `..._reg_down_path`, `..._spin_path`, `..._nspin_path`, `..._flex_up_path`, `..._flex_down_path` | Hourly reserve requirements by reserve zone. |
| `timeseries_data_dc_path` | Optional hourly profiles of large flexible loads (see [Large Load and Demand Response Reference](./Large_Load_and_Demand_Response_Reference.md#optional-hourly-profile)). |

Hourly files have the columns `Year`, `Month`, `Day` and `Period` followed by one column per bus or data region, named by the region identifier. Alternative versions of the load, wind, PV and fuel files are selected with `Load_File_ID`, `Wind_Ons_File_ID`, `PV_File_ID` and `Fuel_ID` in `Simulation Configuration`; a value other than `Base` reads the file `timeseries_<type>_hourly_<ID>.csv` (or `timeseries_fuel_price_<ID>.csv`) from `timeseries_data_files/0_additional_scenarios/<type folder>/`.

## 2. Optional network configuration workbook

The configuration workbook provides the region-mapping and zoning layer on top of the physical network. A-LEAF reads these sheets:

| Sheet | Content |
|---|---|
| `Network Setting` | Switches that select the footprint, aggregation and zone resolutions. |
| `network_resolution_level` | Ordered list of spatial levels. |
| `sub_area_list` | Areas that can become modeled buses. |
| `sub_area_mapping` | Mapping of each finest-level area to every coarser level. |
| `Network Data Level <n>` | One sheet per level with regional records and overlays (resource limits, capacity credits, cost scaling). |
| `RPS`, `CEGT`, `CERT` | Regional policy targets by year. |

`CEGT` is the Clean Energy Generation Target (a minimum share of clean generation), `CERT` is the Carbon Emission Reduction Target (an emissions-reduction requirement) and `RPS` is the Renewable Portfolio Standard (a minimum share of renewable generation), each enforced per policy zone. See [Network Configuration Reference](./Network_Configuration_Reference.md) for the columns and [Policy and Financial Settings](../configuration/Policy_and_Financial_Settings.md) for how each target is activated and enforced.

## 3. Zone structures built from the network inputs

From the physical network and the optional configuration data, A-LEAF builds these zone families:

| Zone family | Use |
|---|---|
| Planning reserve zones | Planning reserve margin and adequacy constraints. |
| Reserve zones | Operating reserve requirements. |
| Policy zones | RPS, clean-energy and carbon-related targets. |
| Resource supply curve zones | Regional build limits. |
| Cost scaling zones | Locational scaling of investment cost. |
| Capacity credit zones | Regional capacity credits, used when a technology's `CAPCRED` is a key rather than a number. |

These are the places where the configuration workbook has the largest effect on results.

## 4. Relationship to time-series data

The network structure determines which buses or regions receive load data, how renewable shapes are aggregated, and which regional identifiers are used for time-series allocation. With a configuration workbook, the data-region mapping is used to aggregate local renewable shapes, map policy and adequacy data, and connect zones to time-series columns. See [Network Resolution](./Network_Resolution.md#per-profile-data-resolution).

## 5. Relationship to the model families

All A-LEAF model families use the same network data:

- Expansion planning uses the topology, zone mappings, technology overlays and representative-day data.
- Production cost simulation uses the same network and zones to build dispatch and reserve formulations.
- Reliability assessment reuses the network and adds outage and renewable-uncertainty logic.

Changes to the base workbook or the configuration workbook can therefore affect all downstream models.

## Scope of this page

This page and the other pages in the `database` folder document the workbook schema: which sheets and columns A-LEAF reads and how they are used. They do not document where the values of a specific bundled data set originally came from. For the provenance of a data set, consult the build notes that accompany that data set.

## Practical reading guide

When reviewing a case, check the network inputs in this order:

1. Confirm `Network_Data_File_ID`.
2. Confirm whether `Network_Configuration_File_ID` is blank, `NA`, or a real ID.
3. If a configuration workbook is used, check `Network Setting`, `sub_area_mapping` and the relevant `Network Data Level` sheet first.
4. Check how that structure interacts with the policy, reserve and resource-limit settings.

If a run behaves as one system-wide region when several zones were expected, first check that a valid configuration workbook is selected.

## Related documentation
- [A-LEAF Documentation](../README.md)
- [ALEAF Simulation Setting File Reference](../configuration/ALEAF_Simulation_Setting_File.md)
- [Network Configuration Reference](./Network_Configuration_Reference.md)
- [Network Resolution](./Network_Resolution.md)
- [Simulation Configuration Reference](../configuration/Simulation_Configuration_Reference.md)
- [Gen Technology Reference](./Gen_Technology_Reference.md)
- [Storage Modeling Reference](./Storage_Modeling_Reference.md)
- [Hybrid Resources Reference](./Hybrid_Resources_Reference.md)
- [Large Load and Demand Response Reference](./Large_Load_and_Demand_Response_Reference.md)
