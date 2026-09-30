# Hybrid Resources Reference

A hybrid resource is a generator and a battery that share one grid connection, for example a solar-plus-storage site. The battery can charge from the onsite generator and, if allowed, from the grid, and the whole site is limited by a single interconnection capacity rather than by separate limits for each piece. Use the `hybrid` sheet of the network workbook to represent such sites.

A-LEAF models a hybrid as two linked members, a generator and a storage unit, instead of blending them into one technology. This keeps the technical characteristics of each component visible while representing their coupling (shared interconnection, onsite charging).

## How hybrid resources are modeled

- **Two members, one site.** Each `hybrid` row creates a generator member and a storage member at the same bus. Both are reported as separate rows in the dispatch output and are tagged with the hybrid type `GEN` or `ES`.
- **Technology from `Gen Technology`.** The technical and cost parameters of each member come from the `Gen Technology` rows whose `UNITGROUP` equals `Hybrid_Gen` and `Hybrid_ES`. The hybrid row supplies only the sizes (`Hybrid_Gen_CAP`, `Hybrid_ES_CAP`), the site connection limit, and the grid-charging choice. The storage energy capacity is the storage power capacity multiplied by the maximum duration `STOHR_MAX` of the storage technology.
- **Shared interconnection.** The output of the generator plus the discharge of the storage cannot exceed `INTERCON_LIM` at any hour, and the energy the storage charges from the grid cannot exceed `INTERCON_LIM`. Power that the generator sends directly to the storage does not use the interconnection.
- **Onsite generation profile.** The hourly availability of the generator member follows the `Profile_Type` of its `Gen Technology` row. For a variable renewable technology (`VRE_Flag` TRUE) with `FUEL_LIMIT` = `Fixed Profile`, the generator can produce at most its capacity multiplied by the hourly shape; the power sent to storage counts against this bound. For other technologies the hourly bound is the capacity multiplied by `PMAX`. In practice most hybrid generators are solar PV (`Profile_Type` = `pv`), but wind and other shapes are supported (see [Gen Technology Reference](./Gen_Technology_Reference.md#profile_type)).
- **Optional grid charging.** When `Grid_Charge` is FALSE, the storage member can charge only from the onsite generator. When TRUE, it can also charge from the grid within `INTERCON_LIM`.
- **Existing assets.** Hybrid members are treated as installed capacity that operates between `Online_Year` and `RetireYear`. They are not investment candidates.

!!! note "One shape mechanism for all onsite generation"
    Onsite generators of hybrid resources use the same hourly-shape mechanism as ordinary variable renewable and hydro plants. Hybrid generators are not part of the ordinary fixed-profile availability constraint; the site-level rules above apply instead.

### Worked example

A 100 MW PV plant with a 50 MW battery behind a 100 MW connection:

| Column | Value |
|---|---|
| `Hybrid_Gen` | `SUN_PV` (a `UNITGROUP` in `Gen Technology` with `Profile_Type` = `pv`) |
| `Hybrid_Gen_CAP` | 100 |
| `Hybrid_ES` | the `UNITGROUP` of a 4-hour battery |
| `Hybrid_ES_CAP` | 50 |
| `INTERCON_LIM` | 100 |
| `Grid_Charge` | FALSE |

The PV output at each hour is at most 100 MW multiplied by the PV shape. The battery charges only from the PV, can hold 50 MW multiplied by its `STOHR_MAX` in MWh, and the PV output that reaches the grid plus the battery discharge stays below 100 MW.

## `hybrid` sheet columns

Column headers are in row 2 of the sheet (row 1 holds group banners). The bundled `network_NorthAmerica_Base.xlsx` contains the header row only, so the sheet is empty by default.

### Identity and timing

| Column | Description |
|---|---|
| `PLANT_NAME` | Name of the hybrid site. |
| `bus_ID` | Bus identifier (`bus_i` in the `bus` sheet) where the site connects. |
| `bus_name` | Bus name. |
| `RetireYear` | Last year of operation. |
| `Online_Year` | First year of operation. |
| `PLANT_ORIS_ID` | Optional facility identifier. |
| `UNITGROUP`, `UNIT_CATEGORY` | Group and category labels of the hybrid site. |
| `UNIT_REPORT_LABEL_1`, `UNIT_REPORT_LABEL_2` | Free-text labels copied to the output; they have no effect on the model. |

### Site and component sizes

| Column | Units | Description |
|---|---|---|
| `INTERCON_LIM` | MW | Shared interconnection limit for injection into and withdrawal from the grid. |
| `Hybrid_Gen` | `UNITGROUP` or `NA` | Generator technology, taken from `Gen Technology`. `NA` means no generator member. |
| `Hybrid_Gen_CAP` | MW | Capacity of the generator member. |
| `Hybrid_ES` | `UNITGROUP` or `NA` | Storage technology, taken from `Gen Technology`. `NA` means no storage member. |
| `Hybrid_ES_CAP` | MW | Power capacity of the storage member; also its charging capacity. |
| `Grid_Charge` | TRUE/FALSE | Whether the storage member may charge from the grid. |
| `CAP`, `PMAX`, `PMIN`, `FOR` | | Present for consistency with the `plant` sheet. Component sizes and limits come from `Hybrid_Gen_CAP`, `Hybrid_ES_CAP` and the `Gen Technology` rows, not from these columns. |

If a `UNITGROUP` named in `Hybrid_Gen` or `Hybrid_ES` does not exist in `Gen Technology`, the run stops with an error naming the hybrid entry.

## Relationship to large flexible load

The same `Hybrid_Gen`, `Hybrid_Gen_CAP`, `Hybrid_ES` and `Hybrid_ES_CAP` fields appear in the `demand` sheet, where they attach onsite generation and storage to a large flexible load. The two uses differ: a `hybrid` site is a supply resource that sells to the grid, whereas onsite resources on a large load first serve that load. See [Large Load and Demand Response Reference](./Large_Load_and_Demand_Response_Reference.md).

## Outputs to inspect

For a resource defined in the `hybrid` sheet, the plant-level dispatch output (`<Case_ID>__dispatch_OP_year_<stage>.csv` and the expansion equivalent) reports the generator and storage members as separate rows, with two hybrid-specific columns:

- `Hybrid_Type`: `GEN` or `ES`, identifying which side of the pair the row describes.
- `Hybrid_Charge_MW`: power that the generator member sends to the storage member.

The storage member also reports `Charge_MW` and `SOC_MWh` like any other storage unit. The descriptive columns `Hybrid_Gen`, `Hybrid_Gen_CAP`, `Hybrid_ES` and `Hybrid_ES_CAP` are reported only in the demand-response file, for hybrid resources attached to a large load (see [Large Load and Demand Response Reference](./Large_Load_and_Demand_Response_Reference.md#outputs-to-inspect)).

See [Operation Outputs](../models/Operation/Operation_Outputs.md) for the complete column list.

## Checklist for a new hybrid

1. Confirm that the generator and storage technologies exist in `Gen Technology` and note their `UNITGROUP` names.
2. Add a `hybrid` row with the bus, the online and retirement years, `Hybrid_Gen`, `Hybrid_Gen_CAP`, `Hybrid_ES`, `Hybrid_ES_CAP`.
3. Set `INTERCON_LIM` to the site connection limit, usually not larger than the sum of the two component capacities.
4. Set `Grid_Charge` according to whether the battery may charge from the grid.
5. Check the storage technology's `STOHR_MAX`, which determines the energy capacity of the battery.

## Related documentation
- [Network Data Reference](./Network_Data_Reference.md)
- [Gen Technology Reference](./Gen_Technology_Reference.md)
- [Storage Modeling Reference](./Storage_Modeling_Reference.md)
- [Large Load and Demand Response Reference](./Large_Load_and_Demand_Response_Reference.md)
- [Operation Outputs](../models/Operation/Operation_Outputs.md)
- [RA Execution and Results](../models/RA/RA_Execution_and_Results.md)
