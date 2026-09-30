# Network Configuration Reference

This page explains the optional network configuration workbook used by A-LEAF and, in particular, the `Network Setting` sheet that controls it. Use this workbook when a study needs to aggregate the network, define several reserve or policy regions, or use regional overlays such as capacity credits and resource limits.

The key idea is:

- the base network workbook defines the physical system,
- the configuration workbook defines how that system is grouped, bounded and interpreted for modeling.

The `Network Setting` sheet is the control layer. It tells A-LEAF:

- which geographic footprint to model,
- which spatial resolution to use for the network itself,
- which spatial resolution to use for reserve zones, policy zones and regional overlays.

## When this workbook is used

The configuration workbook is optional. It is used only when `Network_Configuration_File_ID` is set to a real value in the `Simulation Configuration` sheet. The file is named `config_<test_system_name>_<Network_Configuration_File_ID>.xlsx` and sits in the same data folder as the network workbook.

If the field is missing, blank or `NA`, A-LEAF:

- keeps the original network topology,
- keeps buses at their original resolution,
- builds one system-wide planning reserve zone, one system-wide operating reserve zone and one system-wide policy zone,
- uses neutral system-wide defaults for cost scaling and capacity credit.

The workbook is therefore not required for every run, but it is required whenever the study needs flexible network construction.

## What `Network Setting` controls

The `Network Setting` sheet (columns `Setting`, `Value`, `Note`) is a compact set of switches that determine how the physical network becomes the modeled network. Most switches come in pairs: a `_type` field naming a level of the resolution hierarchy (for example `BA`) and a `_level` field giving that level's number. The bundled `config_NorthAmerica_Base.xlsx` groups them as follows.

### Regional model setting

| Setting | Bundled value | Purpose |
|---|---|---|
| `network_boundary_type`, `network_boundary_level` | `Interconnection`, 4 | Footprint in scope. The modeled regions are those flagged `Model` = TRUE in the `Network Data Level <n>` sheet of the chosen level. |
| `regional_aggregation_resolution_type`, `regional_aggregation_resolution_level` | `BA`, 3 | Main modeled network resolution. |
| `subregional_aggregation_resolution_type`, `subregional_aggregation_resolution_level` | `BA`, 3 | Lower-level grouping that rolls up into each modeled bus. |

### Reserve setting

| Setting | Bundled value | Purpose |
|---|---|---|
| `planning_reserve_zone_boundary_type`, `planning_reserve_zone_boundary_level` | `BA`, 3 | Planning reserve zones. |
| `operating_reserve_zone_boundary_type`, `operating_reserve_zone_boundary_level` | `BA`, 3 | Operating reserve zones. |
| `operating_reserve_data_resolution_type`, `operating_reserve_data_resolution_level` | `BA`, 3 | Resolution of the operating reserve requirement data. |

### Policy setting

| Setting | Bundled value | Purpose |
|---|---|---|
| `policy_zone_boundary_type`, `policy_zone_boundary_level` | `Country`, 5 | Policy zones for RPS, CEGT and CERT. |
| `policy_data_resolution_type`, `policy_data_resolution_level` | `Country`, 5 | Resolution of the policy target data. |

### Additional regional overlays

| Setting | Bundled value | Purpose |
|---|---|---|
| `regional_fuel_zone_resolution_type`, `..._level` | `Country subdivision`, 1 | Resolution of fuel price regions. |
| `regional_resource_cost_scaling_resolution_type`, `..._level` | `BA`, 3 | Resolution of regional capital-cost scaling. |
| `regional_capacity_credits_resolution_type`, `..._level` | `BA`, 3 | Resolution of regional capacity credits. |
| `regional_resource_supply_curve_resolution_type`, `..._level` | `BA`, 3 | Resolution of regional resource limits. |

### Profile data resolution

`load_data_resolution_type`, `pv_data_resolution_type`, `wind_ons_data_resolution_type`, `wind_ofs_data_resolution_type`, `rtpv_data_resolution_type`, `csp_data_resolution_type`, `hydro_data_resolution_type` and `load_growth_data_resolution_type`, each with a matching `_level` field, set the resolution of each time-series profile. See [Network Resolution](./Network_Resolution.md#per-profile-data-resolution).

### Build flags

`generate_networkdata_flag` and `network_reduction_flag` appear in the `[ETC]` group of the sheet. They are present for compatibility and do not change the model behavior; leave them at their bundled values.

These settings do not contain the mapping data themselves; they tell A-LEAF which parts of the other sheets in the configuration workbook to use.

## Other sheets in the configuration workbook

`Network Setting` works because the other sheets provide the geographic hierarchy and the regional records. In every sheet below, row 1 is blank or holds group banners and the column names are in row 2.

### `network_resolution_level`
This sheet defines the ordered hierarchy of spatial levels with the columns `Network_Resolution_Level`, `Network_Resolution_ID` and `Note`. In `config_NorthAmerica_Base.xlsx` the hierarchy is:

| `Network_Resolution_Level` | `Network_Resolution_ID` |
|---|---|
| 1 | `Country subdivision` |
| 2 | `BA zone` |
| 3 | `BA` |
| 4 | `Interconnection` |
| 5 | `Country` |

Levels are ordered from finest to coarsest. The numeric `_level` fields in `Network Setting` point into this table, and the `_type` fields use the `Network_Resolution_ID` names.

!!! note "The hierarchy is defined per configuration workbook"
    `Country subdivision` is the finest level in the bundled configuration. A user-supplied workbook can define other levels, including finer ones. See [Network Resolution](./Network_Resolution.md) for the ladder available with the bundled data.

!!! note "Reserve and policy zones must be equal to or coarser than the aggregation resolution"
    A reserve or policy zone cannot be finer than the modeled network. The levels of `planning_reserve_zone_boundary_type`, `operating_reserve_zone_boundary_type` and `policy_zone_boundary_type` must be equal to or coarser than `regional_aggregation_resolution_level`. For example, with `BA`-level aggregation you can define reserve zones at `BA`, `Interconnection` or `Country`, but not below `BA`. A setting that violates this rule is not rejected with an error; it can produce zone structures that do not match the intended design.

For example, in `config_NorthAmerica_Base.xlsx`, `network_boundary_level` = 4 points to the level-4 `Interconnection` records, while `regional_aggregation_resolution_level` = 3 and `regional_capacity_credits_resolution_level` = 3 point to the level-3 `BA` records. Each `type`/`level` pair is set independently.

### `sub_area_mapping`
This sheet translates between spatial levels. It has one column per level (named by `Network_Resolution_ID`), and each row maps one finest-level area to its `BA zone`, `BA`, `Interconnection` and `Country`. For example:

| `Country subdivision` | `BA zone` | `BA` | `Interconnection` | `Country` |
|---|---|---|---|---|
| `AEC_0_US-AL` | `AEC_0` | `AEC` | `Eastern` | `usa` |

A-LEAF uses this table to:

- remap original buses into aggregated buses,
- determine the region membership of each aggregated bus,
- identify which data region belongs to a reserve or policy zone,
- recover all lower-level areas that roll up into a higher-level zone.

It is the sheet that makes the network configurable.

### `sub_area_list`
This sheet lists the areas that can become modeled buses after aggregation, with the same columns as `sub_area_mapping`. Together with `regional_aggregation_resolution_type`, `subregional_aggregation_resolution_type` and `sub_area_mapping`, it defines the aggregated bus list.

!!! note "If regional and sub-regional resolution are the same type, sub-regional aggregation is skipped"
    When `regional_aggregation_resolution_type` and `subregional_aggregation_resolution_type` have the same value, A-LEAF logs a warning ("Detected the same resolution for both regional aggregation and sub-regional aggregation" and "Sub-regional aggregation will not be applied.") and skips the nested grouping step. This is the case in the bundled `config_NorthAmerica_Base.xlsx`, where both fields are `BA`.

### `Network Data Level <n>`
There is one such sheet for each level defined in `network_resolution_level`. It holds the regional records for that level. In the bundled configuration each sheet has these columns:

| Column | Description |
|---|---|
| `Region_ID` | Region identifier, as used in `sub_area_mapping`. |
| `Region_Name` | Descriptive name. |
| `Model` | TRUE marks the region as part of the modeled footprint (used when this level is the network boundary). |
| `Subregional_Aggregation` | Flag used when sub-regional aggregation is applied (FALSE for all regions in the bundled configuration). |
| `RA_ELCC_Calculation_Flag` | Marks whether the region participates in the ELCC calculation; see [RA Metrics, ELCC, and DLOL](../models/RA/RA_Metrics_ELCC_and_DLOL.md). |
| `Resource_Limit_<key>_highcap` and `Resource_Limit_<key>_lowcap` | Regional capacity limits (MW) for resources with `Resource_Limit_ID` equal to `<key>` (bundled keys: `wind_ons`, `wind_ofs`, `pv`, `psh`). The case setting `Resource_limit_level_value` (`High` or `Low`) selects which column applies. |
| `CAPCRED_<key>` | Regional capacity credit (fraction) for technologies whose `CAPCRED` in `Gen Technology` is the text `<key>` (bundled: `wind_ons`, `wind_ofs`, `pv`, `rtpv`, `hydro_ROR`, `hydro_impoundment`, `csp`). |
| `CAPAX_scale_<key>` | Regional capital-cost variation for technologies with `Locational_Scaling_Flag` TRUE. |

These sheets serve two purposes: they define which regions are modeled at a level, and they store the regional overlays tied to that level. The overlay used by a run is read from the sheet whose level number is set in the corresponding `Network Setting` field. This is why `Network Setting` has both a `type` and a `level`: the level selects the `Network Data Level <n>` sheet and the type names the geographic field of that level.

### `RPS`, `CEGT` and `CERT`
These sheets provide regional policy targets:

- `RPS`: Renewable Portfolio Standard (minimum renewable generation share),
- `CEGT`: Clean Energy Generation Target (minimum clean generation share),
- `CERT`: Carbon Emission Reduction Target (emissions-reduction requirement).

Each sheet has a `Region` column (a region of the policy-data resolution) followed by one column per year with the target value. `CERT` has an additional `Ref_Emission_m_ton` column with the reference emissions used for the reduction target. `Network Setting` determines the policy zone boundaries and the resolution at which these values are read. See [Policy and Financial Settings](../configuration/Policy_and_Financial_Settings.md) for how each target is activated and enforced.

## Fuel-zone resolution must match the fuel file

`regional_fuel_zone_resolution_type` selects the geographic level at which A-LEAF looks up fuel prices. This level must match the region granularity of the fuel-price CSV (see [Simulation Configuration Reference](../configuration/Simulation_Configuration_Reference.md#fuel_id) for the file schema).

The fuel-price file is indexed by region, fuel, year and month, and A-LEAF requests a price for each modeled region by name:

- For the bundled North America fuel files, the region values are bus identifiers such as `ERCO_NCEN_US-TX`. `regional_fuel_zone_resolution_type` must then be the finest level of the hierarchy (`Country subdivision` in `config_NorthAmerica_Base.xlsx`).
- If it is set coarser (for example `BA` or `Interconnection`), the requested region name does not exist in the file, and A-LEAF falls back to the `System-wide` row of the file for that fuel, year and month.

!!! warning "A fuel-zone mismatch does not fail immediately"
    A resolution coarser than the granularity of the fuel file makes the model silently use the `System-wide` fuel price instead of the intended regional price. With `logging_level_value` set to `detailed`, the log shows `Failed to load fuel price for type <FUEL> in year <YEAR> in region <REGION>. System-wide price will be used`. If the file has no `System-wide` row for that fuel, year and month, the run terminates with `Failed to load fuel price for type <FUEL> in year <YEAR> in region <REGION>. Terminate the program`. To obtain regional prices, set `regional_fuel_zone_resolution_type` to the level on which the fuel file is keyed (`Country subdivision` for the bundled files).

## How flexible network construction works

The construction proceeds in these steps:

1. Start from the physical network in the base workbook.
2. Select the footprint with `network_boundary_type` and `network_boundary_level`. The footprint is exactly the set of regions with `Model` = TRUE in the corresponding `Network Data Level <n>` sheet. For `network_boundary_type` = `Interconnection` (level 4), it is the set of interconnections flagged in `Network Data Level 4`.
3. Select the modeled network resolution with `regional_aggregation_resolution_type` and `..._level`.
4. Select the lower-level grouping with `subregional_aggregation_resolution_type` and `..._level`.
5. Use `sub_area_mapping` to connect these levels to the original physical areas.
6. Use the `Network Data Level <n>` sheets to decide which regions are modeled and which regional overlay data applies.
7. Build separate zone structures for planning reserve, operating reserve, policy, resource supply curves, cost scaling and capacity credit.

The same physical network workbook can therefore support different modeled systems simply by changing the configuration workbook selections.

### Using different resolutions for different layers

The network, reserve zones, policy zones and overlay data do not have to use the same geography. For example, a study can:

- aggregate the modeled network to `BA` level,
- read operating reserve requirements at `BA` level,
- enforce policy zones at `Country` level,
- look up capacity credits at `Interconnection` level,
- apply cost scaling at `BA` level.

This is why the workbook has separate controls for network aggregation, reserve boundaries and data, policy boundaries and data, and fuel, cost, supply-curve and capacity-credit resolution.

!!! note "Power-flow mode is set in the Simulation Setting sheet"
    The power-flow representation of the modeled network is chosen by `power_flow_mode_flag` in the `Simulation Setting` sheet, with the options `Network_Flow` (transport), `B-theta` (DC angle) and `PTDF` (shift factors). In `B-theta` and `PTDF` modes, the reactances of aggregated AC corridors are re-estimated from synthetic snapshots after aggregation, and the `PTDF` factors are built from the final branch list so that `PTDF` matches `B-theta` on the same network. See [Network Resolution](./Network_Resolution.md) and [GTEP Transmission Expansion: PTDF / snapshot network reduction](../models/GTEP/GTEP_Transmission_Expansion.md#ptdf-snapshot-network-reduction).

## Example: `config_NorthAmerica_Base.xlsx`

The bundled configuration (`data/NorthAmerica/config_NorthAmerica_Base.xlsx`) shows how the pieces fit together. Its `Network Setting` values are:

- `network_boundary_type` = `Interconnection`, `network_boundary_level` = 4
- `regional_aggregation_resolution_type` = `BA`, level 3
- `subregional_aggregation_resolution_type` = `BA`, level 3
- `planning_reserve_zone_boundary_type`, `operating_reserve_zone_boundary_type` and `operating_reserve_data_resolution_type` = `BA`
- `policy_zone_boundary_type` and `policy_data_resolution_type` = `Country`
- `regional_resource_cost_scaling_resolution_type`, `regional_capacity_credits_resolution_type` and `regional_resource_supply_curve_resolution_type` = `BA`
- `regional_fuel_zone_resolution_type` = `Country subdivision`
- all profile data resolutions = `Country subdivision` (level 1)

In practice this means:

- The modeled footprint is chosen by the `Model` flags of `Network Data Level 4`. In the bundled file only Texas is flagged, so the default run models Texas.
- The network resolution is `BA`: the original sub-state buses are aggregated up to BA. Because the regional and sub-regional types are both `BA`, the sub-regional step is skipped.
- Planning and operating reserve zones are BA-level.
- Policy targets are enforced at Country level.
- Capacity credits, resource limits and cost scaling are applied at BA level.
- Fuel prices are looked up per `Country subdivision`.

To model another interconnection, set `Model` = TRUE for it in `Network Data Level 4` (and FALSE for Texas if it should not be included).

## Practical interpretation rules

- A `*_type` field answers "which geographic label is used?", for example `BA`, `Interconnection` or `Country`.
- A `*_level` field answers "which `Network Data Level <n>` sheet and hierarchy level provide the records?".
- Ask what the pair controls: network boundary, aggregation, reserve, policy, cost scaling, capacity credit or resource supply curve.

## What to check first when debugging

1. `Network_Configuration_File_ID` is set in `Simulation Configuration`.
2. The selected configuration workbook exists in the data folder (a missing file stops the run with an error).
3. The `network_resolution_level` hierarchy matches the intended geography.
4. `sub_area_mapping` contains the needed cross-level mappings.
5. The right `Network Data Level <n>` sheets contain modeled regions with `Model` = TRUE.
6. The reserve, policy and overlay resolution fields agree with the intended study design.

Common symptoms of a bad configuration are:

- the run collapses to one system-wide zone,
- policy targets apply at the wrong geography,
- capacity-credit values are missing or unexpected,
- resource limits or cost scaling do not match the intended regional design.

## Related documentation
- [Network Data Reference](./Network_Data_Reference.md)
- [Network Resolution](./Network_Resolution.md)
- [ALEAF Simulation Setting File Reference](../configuration/ALEAF_Simulation_Setting_File.md)
- [Simulation Configuration Reference](../configuration/Simulation_Configuration_Reference.md)
- [A-LEAF Documentation](../README.md)
