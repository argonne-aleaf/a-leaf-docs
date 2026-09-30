# Storage Modeling Reference

This page explains how energy storage (batteries, pumped-storage hydro, and other energy-limited resources) is described in the A-LEAF workbooks and how the model uses those inputs. It covers where each storage input lives, what each column means, and how the case-level storage settings affect the results.

Storage is not driven by a single sheet. It is assembled from:

- existing storage records in the `plant` sheet of the network workbook,
- candidate storage technologies in the `Gen Technology` sheet,
- storage cost and performance assumptions in the `Storage Cost and Performance` sheet,
- case-level switches in the `Simulation Configuration` sheet.

Storage hybridized with a generator or a large load is described separately in [Hybrid Resources Reference](./Hybrid_Resources_Reference.md) and [Large Load and Demand Response Reference](./Large_Load_and_Demand_Response_Reference.md).

## Data surfaces

### Existing storage in the network workbook

Existing storage enters the model through rows of the `plant` sheet with `UNIT_CATEGORY` = `STORAGE`. The columns that define a storage plant are:

| Column | Units | Description |
|---|---|---|
| `CAP` | MW | Discharge (generating) capacity. |
| `Charge_CAP` | MW | Charging capacity. |
| `ES_MWh` | MWh | Stored-energy capacity of the plant. |
| `STOMIN` | MWh | Minimum stored energy (state-of-charge floor). |
| `STOHR_MIN`, `STOHR_MAX` | hours | Minimum and maximum storage duration of the technology. `STOHR_MAX` is used, for example, to derive the energy capacity of a hybrid storage member. |
| `PMAX`, `PMIN` | fraction of `CAP` | Maximum and minimum output. |
| `BATEFF` | fraction | Round-trip efficiency. |
| `AET` | MWh per year | Annual energy throughput limit. |

Most of these columns can hold the text `Gen_Tech`, which takes the value from the `Gen Technology` row with the same `UNITGROUP`. A value entered in the `plant` row overrides the technology default. Round-trip efficiency and throughput can also be set to `ESGC` to read them from `Storage Cost and Performance`.

### Candidate storage in `Gen Technology`

Candidate storage technologies are rows of `Gen Technology`. The storage-related columns are:

| Column | Description |
|---|---|
| `UNITGROUP` | Technology name, also the join key to `Storage Cost and Performance`. |
| `UNIT_CATEGORY` | `STORAGE` for storage technologies. |
| `Hydro_Flag` | `PSH` classifies the technology as pumped-storage hydro. A `STORAGE` row with any other `Hydro_Flag` is treated as a battery. The classification separates pumped-storage candidates from batteries in the pumped-storage analysis. |
| `Storage Commitment` | TRUE adds a binary charge/discharge status (reported as `Storage_Charging_Flag`) that prevents simultaneous charging and discharging. `NA` or FALSE leaves storage as continuous dispatch. |
| `Charge_CAP`, `STOMIN`, `STOHR_MIN`, `STOHR_MAX` | As in the table above, per unit block of the technology. |
| `CAPEX`, `STO_CAPEX` | Power-related capital cost (per kW) and energy-related capital cost (per kWh). Either can be a number or a reference such as `ESGC` or `ATB`. |
| `FOM`, `VOM` | Fixed and variable operation and maintenance cost. |
| `FCR` | Fixed charge rate, the factor that converts a capital cost into an annual charge. |
| `BATEFF`, `AET` | Round-trip efficiency and annual throughput limit; can be `ESGC`. |
| `ES_STO_INVEST_FLAG` | TRUE lets the model choose the storage duration of new units (between `STOHR_MIN` and `STOHR_MAX`) instead of fixing it. |

See [Gen Technology Reference](./Gen_Technology_Reference.md) for the complete column list. Report labels `UNIT_REPORT_LABEL_1` and `UNIT_REPORT_LABEL_2` do not affect the model (see [Gen Technology Reference](./Gen_Technology_Reference.md#unit_report_label_1-and-unit_report_label_2)).

### `Storage Cost and Performance`

The `Storage Cost and Performance` sheet holds storage cost and performance scenarios. Each row is identified by:

- `ESGC_Setting_ID`, the name of the scenario (for example `Low_Price_Fast_Learning`). The `ESGC_Setting_ID` of the `Simulation Configuration` sheet selects the scenario used by a case. "ESGC" is the label of this data source; a `Gen Technology` or `plant` cell containing the text `ESGC` means "take this value from `Storage Cost and Performance`".
- `UNITGROUP`, the technology the row applies to.

The sheet has two header rows (a description row and a units row) below the column names. Its columns are:

| Column | Units | Description |
|---|---|---|
| `UNIT_CATEGORY` | | Category of the technology. |
| `Capacity` | MW | Reference power capacity of one unit. |
| `Duration` | hours | Storage duration. |
| `RTE` | fraction | Round-trip efficiency. It becomes `BATEFF` when a cell references `ESGC`. |
| `Cycle_Life` | cycles | Number of cycles over the life of the asset. |
| `Calendar_Life` | years | Calendar life. |
| `AET` | MWh per year | Annual energy throughput. In the bundled data it is calculated as capacity multiplied by duration, efficiency, cycle life and 0.8, divided by calendar life. |
| `FOM`, `VOM` | $/kW-year, $/MWh | Fixed and variable operation and maintenance cost. |
| `CAPEX_Scale` | multiplier | Scaling factor applied to the capital cost columns. |
| `Total Capital Cost_kW_<year>` | $/kW | Power-related capital cost by year. |
| `Total Capital Cost_kWh_<year>` | $/kWh | Energy-related capital cost by year. |

A technology referencing `ESGC` for a value must have a row for the selected `ESGC_Setting_ID` and its `UNITGROUP`; otherwise the run stops with an error that names the missing entry.

### Case-level settings

Two settings in `Simulation Configuration` control storage behavior:

| Setting | Values | Description |
|---|---|---|
| `storage initialization option` | `Minimum`, `Middle`, `Maximum` | Initial state of charge at the start of each modeled period: the floor (`STOMIN`), half of the energy capacity, or full. Any other value leaves the initial state of charge free for the optimizer to choose. |
| `Energy_Storage_AET_Limit_Flag` | TRUE/FALSE | Enforces the annual energy throughput limit (`AET`). |

The choice of representative days also affects storage, because the state of charge is tracked across the hours of each representative day and days are weighted by the number of days they represent.

## How storage is modeled

The expansion and operation models use the same storage rules, so storage behaves consistently between planning and operation runs.

- **Power limits.** Charging cannot exceed `Charge_CAP` and discharging cannot exceed `CAP`, each scaled by the number of units in service. Storage that provides reserves must keep enough headroom for the reserve it carries.
- **Energy balance.** The state of charge changes hour to hour by the charged energy multiplied by the one-way efficiency minus the discharged energy divided by the one-way efficiency. The one-way efficiency is the square root of the round-trip efficiency (`BATEFF`).
- **Energy limits.** The state of charge stays between `STOMIN` and the energy capacity. For existing storage the energy capacity is `ES_MWh`. For new storage it is the invested power capacity multiplied by the invested duration.
- **Investment.** Power capacity and, when `ES_STO_INVEST_FLAG` is TRUE, storage duration are separate investment decisions with separate costs (`CAPEX` per kW and `STO_CAPEX` per kWh). Power and energy additions are therefore not always proportional.
- **Daily cycle neutrality.** A representative day cannot create net energy from nothing: storage returns to its starting state of charge over a representative-day group, so storage cannot use the reduced chronology to gain energy.
- **Annual throughput.** When `Energy_Storage_AET_Limit_Flag` is TRUE, the total energy cycled during the year is limited by `AET`, scaled from the representative days to a full year. Reserve provision counts partly toward throughput. See [GTEP Formulation](../models/GTEP/GTEP_Formulation.md#storage-annual-energy-throughput-aet) for the equation.
- **Storage commitment.** With `Storage Commitment` TRUE, a binary status prevents charging and discharging in the same interval.

### Annual energy throughput

`AET` limits the number of MWh a unit can cycle per year, to represent degradation and warranty limits. It is expressed per unit block, so the plant-level limit scales with the number of units in service. The limit is annualized with the representative-day weights, so it applies to a full year and not to a single representative day. A technology with `AET` equal to 0 has no throughput limit.

## Outputs to inspect

The main storage results in the dispatch output are:

- `Charge_MW`: charging power.
- `SOC_MWh`: state of charge.
- `Storage_Energy_MWh`: storage energy capacity in service.
- `Storage_Charging_Flag`: charge/discharge status when storage commitment is active.
- `New_Storage_Unit_Hours`: new storage duration invested, in the expansion outputs.

See [Operation Outputs](../models/Operation/Operation_Outputs.md) and [GTEP Expansion Outputs](../models/GTEP/GTEP_Expansion_Outputs.md) for the complete column lists. As a consistency check, `SOC_MWh` should stay within `Storage_Energy_MWh`, and the charge and discharge totals should be consistent with the round-trip efficiency.

## Practical notes

- Set `STOHR_MIN` and `STOHR_MAX` equal to fix the duration of a technology. Set `ES_STO_INVEST_FLAG` to TRUE and use different bounds to let the model choose the duration.
- Use `Hydro_Flag` = `PSH` only for pumped-storage hydro. Batteries and other storage keep `Hydro_Flag` FALSE.
- Round-trip efficiency in `BATEFF` is a fraction such as 0.85, not a percentage.
- If storage rarely cycles in a run, check the `AET` limit, the reserve requirements (storage can earn its value by holding reserves) and the price spreads of the representative days.

## Checklist for a storage case

1. Existing storage rows in the `plant` sheet: `CAP`, `Charge_CAP`, `ES_MWh`, `STOMIN`.
2. Candidate storage rows in `Gen Technology`: category, `Hydro_Flag`, duration bounds, cost columns.
3. The selected `ESGC_Setting_ID` and the matching rows in `Storage Cost and Performance`.
4. `storage initialization option`.
5. `Energy_Storage_AET_Limit_Flag`.

## Related documentation
- [Gen Technology Reference](./Gen_Technology_Reference.md)
- [Policy and Financial Settings](../configuration/Policy_and_Financial_Settings.md)
- [Network Data Reference](./Network_Data_Reference.md)
- [Operation Outputs](../models/Operation/Operation_Outputs.md)
- [GTEP Expansion Outputs](../models/GTEP/GTEP_Expansion_Outputs.md)
