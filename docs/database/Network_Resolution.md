# Network Resolution

This page explains the spatial resolutions at which A-LEAF can run the bundled North America data and how a run selects one. It builds on the configuration workbook described in the [Network Configuration Reference](./Network_Configuration_Reference.md); read that page first for how the `Network Setting` sheet drives aggregation in general.

## The resolution ladder

The spatial levels available to a run are defined by the `network_resolution_level` sheet of the configuration workbook (see [Network Configuration Reference: `network_resolution_level`](./Network_Configuration_Reference.md#network_resolution_level)). The bundled `data/NorthAmerica/config_NorthAmerica_Base.xlsx` defines five levels, from finest to coarsest:

| Level | `Network_Resolution_ID` | Description |
|---|---|---|
| 1 | `Country subdivision` | Finest bundled level: the sub-state grouping used by the North America data sets (335 areas). This is what "subdivision" means on this page. |
| 2 | `BA zone` | Zones within a balancing authority. |
| 3 | `BA` | Balancing authority. |
| 4 | `Interconnection` | Interconnection (for example Eastern, Western, Texas). |
| 5 | `Country` | Country. |

The bundled config has no `ISO` or `State` level. The ladder is defined per configuration workbook, so a user-supplied workbook can define other levels, including finer ones, by editing `network_resolution_level`, `sub_area_list`, `sub_area_mapping` and the `Network Data Level <n>` sheets. The bundled data does not include a resolution finer than `Country subdivision`.

## Selecting resolution

Three groups of fields in the `Network Setting` sheet of the configuration workbook select the part of the ladder that a run uses. Each group has a `_type` field (the `Network_Resolution_ID` from the table above) and a `_level` field (the matching level number).

- **`network_boundary_type` with `network_boundary_level`.** The geographic footprint in scope. The modeled regions are those with `Model` = TRUE in the `Network Data Level <n>` sheet of the chosen level. In the bundled configuration the boundary is `Interconnection` (level 4), and only Texas has `Model` = TRUE, so the default run models Texas.
- **`regional_aggregation_resolution_type` with `regional_aggregation_resolution_level`.** The main modeled network resolution. The bundled configuration sets `BA` (level 3), so the default run aggregates the sub-state buses of Texas to balancing-authority buses. Set `Country subdivision` (level 1) for the finest bundled network, or `BA zone`, `BA`, `Interconnection` or `Country` for progressively coarser aggregation.
- **`subregional_aggregation_resolution_type` with `subregional_aggregation_resolution_level`.** The lower-level grouping that rolls up into each modeled bus. When it equals the regional type, the sub-regional step is skipped (see the note under `sub_area_list` in the configuration page).

Reserve zones, policy zones, fuel zones, cost scaling, capacity credits and supply curves have their own resolution fields. Reserve and policy zones must be equal to or coarser than the modeled network, and the fuel-zone resolution must match the granularity of the fuel-price file (see the [Network Configuration Reference](./Network_Configuration_Reference.md#fuel-zone-resolution-must-match-the-fuel-file)).

!!! note "Choosing a resolution"
    Finer resolution represents transmission constraints in more detail but produces a larger optimization problem. For interconnection-wide studies, `BA` is a common compromise; use `Country subdivision` for regional studies where sub-state congestion matters.

## Aggregating branches

When the selected resolution is coarser than the finest, parallel lines between the same pair of modeled regions are merged into one corridor whose rating is the sum of its members. AC lines and DC ties are never merged together. See [GTEP Transmission Expansion: Corridor aggregation](../models/GTEP/GTEP_Transmission_Expansion.md#corridor-aggregation) for the rules.

In a user-supplied database that defines finer levels, lines whose two ends are both at the finest level keep their own rating and reactance as separate physical lines, while corridors that touch an aggregated region are merged as usual.

## PTDF / snapshot network reduction

When aggregation merges AC lines and the run uses the `B-theta` or `PTDF` power-flow mode, the reactances of the aggregated corridors are re-estimated from synthetic operating snapshots so that the aggregated network reproduces the cross-border flows of the detailed network. DC ties are excluded from this estimation; in `PTDF` mode they receive zero shift factors and keep their own transfer limits.

The power-flow mode is set with `power_flow_mode_flag` in the `Simulation Setting` sheet. For the estimator, its inputs and its fallback behavior, see [GTEP Transmission Expansion: PTDF / snapshot network reduction](../models/GTEP/GTEP_Transmission_Expansion.md#ptdf-snapshot-network-reduction).

## Per-profile data resolution

A network does not need one time-series column per modeled bus. A-LEAF can read one shape per data region at a coarser resolution and apply it to all finer buses that roll up into that region. The data resolution is set separately for each profile in the `Network Setting` sheet:

- The profiles are `load`, `pv`, `wind_ons`, `wind_ofs`, `rtpv`, `csp`, `hydro` and `load_growth`.
- For each profile, `<profile>_data_resolution_type` and `<profile>_data_resolution_level` give the resolution, for example `load_data_resolution_type` and `pv_data_resolution_type`.
- When a profile's field is absent, the profile is read at the finest resolution, so every bus has its own column.
- The column names of the time-series file must be the region identifiers of the selected resolution. The mapping from each finest area to its data region comes from `sub_area_mapping`.

For example, load can be read at `BA` resolution while wind is read at `Country subdivision` resolution. In the bundled configuration, all profiles are supplied at `Country subdivision` (level 1).

## Related documentation
- [Network Configuration Reference](./Network_Configuration_Reference.md)
- [Network Data Reference](./Network_Data_Reference.md)
- [GTEP Transmission Expansion](../models/GTEP/GTEP_Transmission_Expansion.md)
