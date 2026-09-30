# Large Load and Demand Response Reference

Large flexible loads (LFLs) such as data centers, cryptocurrency-mining facilities, or flexible industrial plants are not always fixed demand: many are willing to curtail part of their consumption when the price of power is high enough. A-LEAF represents this willingness explicitly. Each large load is described by a price/quantity curve made of up to five segments, so the optimizer can trade off serving the load fully against other uses of the energy. Optionally, a large load can have its own onsite generation and storage.

Large loads are entered in the `demand` sheet of the network workbook (`network_<test_system_name>_<Network_Data_File_ID>.xlsx`). This page describes the sheet, explains how the model treats each entry, and lists the results that report it. Use this page when a study includes data centers or other flexible demand that should respond to prices, or when a load has onsite supply.

## How large flexible load is modeled

For every entry in the `demand` sheet, the model decides in each hour how much of the load is served. The main ideas are:

- **Served versus curtailed load.** The load can consume at most its capacity `CAP` (in MW). The difference between the maximum consumption and the load actually served is the demand response (curtailment) in that hour.
- **Price/quantity segments.** `CAP` is split into up to five blocks whose sizes are given as shares of `CAP` (`Pct_MW_1` to `Pct_MW_5`). Segment 1 is a mandatory floor that is always served. Each further segment is optional and can be served only if the segments before it are served (segments switch on in order). Serving an optional segment earns the segment price (`Price_2` to `Price_5`, in $/MWh); the optimizer therefore keeps a block online when the value of the energy to the load exceeds the value of that energy elsewhere in the system (its marginal cost or LMP), and curtails it otherwise.
- **Daily curtailment limit.** `Daily_DR_Limit_MWh` caps the total energy the load can curtail below its maximum consumption over one representative day.
- **Interconnection limit.** `INTERCON_LIM` (MW) limits the site's connection to the grid. Grid withdrawals (load plus any storage charging from the grid) and, for sites with onsite supply, injections into the grid are each limited to this value.
- **Online year and hourly profile.** A large load is inactive in planning years before its `Online_Year`. When the row names a `Profile_Tag` and an hourly profile file is provided (see [Optional hourly profile](#optional-hourly-profile)), the maximum consumption follows that 0 to 1 shape instead of staying flat at `CAP`.
- **Optional onsite generation and storage.** A large load can be paired with a generator and a battery behind the same meter. Onsite generation can serve the load, be exported to the grid, or charge the onsite storage. Storage can serve the load or export to the grid, and can charge from the grid or from onsite generation.
- **Cost accounting.** Segment prices enter the objective as a reward for the energy served in optional segments. Segment 1 carries no price because it is always served. The curtailed energy therefore has an opportunity cost equal to the forgone segment price rather than a value-of-lost-load penalty. Because prices directly shape the optimization, they are model inputs and not descriptive metadata.

This block structure gives large flexible load an economic, stepwise response instead of treating it as fixed load.

### Worked example

The bundled North America database contains a data center row with these values:

| Field | Value | Meaning |
|---|---|---|
| `CAP` | 14 | Maximum consumption of 14 MW |
| `INTERCON_LIM` | 14 | Site connection limited to 14 MW |
| `Num_DR_Segments` | 2 | Two segments are active |
| `Pct_MW_1`, `Pct_MW_2` | 0.9, 0.1 | Segment 1 is 12.6 MW (mandatory), segment 2 is 1.4 MW (optional) |
| `Price_2` | 9000 | Segment 2 is served only when the energy is worth more than 9000 $/MWh to the system |
| `Daily_DR_Limit_MWh` | 33.6 | Up to 33.6 MWh per day can be curtailed |

Segment 2 can be curtailed for at most 24 hours at 1.4 MW, which equals 33.6 MWh, so the daily limit in this example does not bind beyond the segment size. Under normal prices the load is served in full at 14 MW; under scarcity it drops to 12.6 MW.

## `demand` sheet columns

Column headers are in row 2 of the sheet (row 1 holds group banners). Each row is one large-load entry.

### Identity and location

| Column | Description |
|---|---|
| `PLANT_NAME` | Name of the load entry. |
| `bus_ID` | Bus identifier (`bus_i` in the `bus` sheet) where the load connects. |
| `bus_name` | Bus name. Informational. |
| `Online_Year` | First year in which the load is active. `NA` keeps the load active in all years. |
| `UNITGROUP` | Group label for the load (for example `DataCenter`). Reported in the outputs. |
| `UNIT_CATEGORY` | Category label (for example `DATA_CENTER`). Reported in the outputs. |
| `UNIT_REPORT_LABEL_1`, `UNIT_REPORT_LABEL_2` | Free-text labels, for example the load type and the facility or operator name. They are copied to the output and have no effect on the model. |

### Size and interconnection

| Column | Units | Description |
|---|---|---|
| `CAP` | MW | Maximum consumption of the load. It is the basis for the segment sizes. |
| `INTERCON_LIM` | MW | Interconnection limit applied to grid withdrawals and, for sites with onsite supply, to grid injections. |

### Demand-response structure

| Column | Units | Description |
|---|---|---|
| `Integer_Flag` | TRUE/FALSE | When TRUE, the on/off decision of each optional segment is binary. When FALSE, the segments can be partially engaged, which keeps the problem continuous. |
| `Daily_DR_Limit_MWh` | MWh/day | Upper limit on the load that can be curtailed below its maximum consumption in one representative day. |
| `Num_DR_Segments` | 1 to 5 | Number of active segments. Segments beyond this number are ignored. |
| `Pct_MW_1` to `Pct_MW_5` | fraction of `CAP` | Size of each segment. Segment 1 is the mandatory floor. The active shares should sum to 1 so that the segments together cover `CAP`. |
| `Price_1` to `Price_5` | $/MWh | Value of serving each segment. `Price_1` is not used because segment 1 is always served. `Price_2` up to the price of the last active segment enter the objective. |

### Optional onsite generation and storage

| Column | Description |
|---|---|
| `Hybrid_Gen` | `UNITGROUP` of the onsite generator, taken from the `Gen Technology` sheet, or `NA` for none. |
| `Hybrid_Gen_CAP` | Capacity of the onsite generator in MW. |
| `Hybrid_ES` | `UNITGROUP` of the onsite storage technology, taken from the `Gen Technology` sheet, or `NA` for none. |
| `Hybrid_ES_CAP` | Power capacity of the onsite storage in MW. |

The onsite generator and storage inherit their technical parameters (efficiency, minimum state of charge, profile type) from the referenced `Gen Technology` rows. If a named `UNITGROUP` is not found in `Gen Technology`, the run stops with an error identifying the load entry. The `demand` sheet has no `Grid_Charge` column: onsite storage may charge from the grid within `INTERCON_LIM`.

!!! note "Onsite storage energy capacity"
    For a large load, the power rating of the onsite storage is `Hybrid_ES_CAP` (MW), and its energy capacity is that rating multiplied by the maximum storage duration (`STOHR_MAX`) of the referenced `Gen Technology` storage row, as for a hybrid plant (see [Hybrid Resources Reference](./Hybrid_Resources_Reference.md)). The state of charge is limited to this energy capacity, and the `Maximum` and `Middle` initialization options start from it in full or at half.

### Optional columns

| Column | Description |
|---|---|
| `Profile_Tag` | Name of a column in the hourly profile file (see below). `NA` or blank keeps the load flat at `CAP`. |
| `status`, `county`, `developer`, `latitude`, `longitude`, `site_dist_km` | Descriptive columns carried in the bundled database. They do not affect the model. |
| `PMAX`, `PMIN`, `FOR` | Columns present in some databases for consistency with the `plant` and `hybrid` sheets. They do not affect the model for large loads. |

### Optional hourly profile

To give large loads an hourly shape, add the key `timeseries_data_dc_path` to the `File Path` sheet of the network workbook, pointing to a CSV file relative to the data folder (for example `timeseries_data_files/DC/timeseries_dc_hourly.csv`). The file has the columns `Year`, `Month`, `Day`, `Period` followed by one column per profile, with values between 0 and 1. The `Profile_Tag` of each load names its column (for example `DC0001`). The maximum consumption in each hour is `CAP` multiplied by that value. If the key or the file is absent, all loads are flat.

## Interaction with other sheets

- `Gen Technology` provides the technology definitions of onsite generation and storage through `Hybrid_Gen` and `Hybrid_ES`.
- The `bus` sheet defines the bus in `bus_ID`. When a configuration workbook aggregates the network, the load is assigned to the aggregated bus that contains its `bus_ID`.
- The load in the `demand` sheet is separate from the fixed load in the `bus` sheet. Make sure that a facility is represented in only one of them to avoid double counting.
- The onsite generator uses the `Profile_Type` of its `Gen Technology` row to select its hourly availability shape (see [Gen Technology Reference](./Gen_Technology_Reference.md#profile_type)). A generator with `Profile_Type = NA` is available at its full capacity in every hour.
- The `storage initialization option` in `Simulation Configuration` (`Minimum`, `Middle`, or `Maximum`) sets the initial state of charge of onsite storage, as for other storage.

## Outputs to inspect

Large-load results are written to the demand-response file of the expansion and operation runs:

- `<Case_ID>__demand_response_EXP.csv` (expansion run)
- `<Case_ID>__demand_response_OP.csv` (operation run)

Both are controlled by the dispatch report flags in the `Simulation Setting` sheet (`report_dispatch_EXP_flag`, `report_dispatch_OP_flag`). The main columns are:

- **Identity and inputs:** `Unit_Group`, `Unit_Category`, `Unit_Report_Label_1`, `Unit_Report_Label_2`, `Unit_Capacity_MW`, `Interconnection_Limit_MW`, `Integer_Flag`, `Daily_DR_Limit_MWh`, `Num_DR_Segments`, `Pct_MW_1` to `Pct_MW_5`, `Price_1` to `Price_5`, `Hybrid_Gen`, `Hybrid_Gen_CAP`, `Hybrid_ES`, `Hybrid_ES_CAP`.
- **Load results:** `LFL_Load_MW` (load served), `LFL_DR_MW` (curtailed load), `LFL_DR_Segment_1_MW` to `LFL_DR_Segment_5_MW` (served quantity per segment), `LFL_DR_Segment_1_Active` to `LFL_DR_Segment_5_Active` (segment engagement).
- **Onsite resources:** `LFL_Gen_to_Load_MW`, `LFL_Gen_to_Grid_MW`, `LFL_Gen_to_Storage_MW`, `LFL_Storage_to_Load_MW`, `LFL_Storage_to_Grid_MW`, `LFL_Grid_to_Storage_MW`, `LFL_Storage_SOC_MWh`.

See [Operation Outputs](../models/Operation/Operation_Outputs.md) and [GTEP Expansion Outputs](../models/GTEP/GTEP_Expansion_Outputs.md) for the complete column lists, and [Operation Formulation](../models/Operation/Operation_Formulation.md) for the equations.

## Practical notes

- Set `Pct_MW_1` to the share of the load that must always be served. A value of 1 with a single segment represents a fixed load that never curtails.
- A high `Price_2` (such as the 9000 $/MWh in the example) makes the optional segment behave almost like firm load, because it is curtailed only in extreme scarcity. Lower prices produce more price-responsive load.
- Keep the `Pct_MW_*` shares of the active segments summing to 1. Shares that sum to less than 1 reduce the maximum consumption below `CAP`.
- Grid withdrawals (served load plus storage charging from the grid) share one `INTERCON_LIM`. When the limit equals the load, the onsite battery cannot charge from the grid while the load is fully served.

## Checklist for a new entry

1. Add a row to the `demand` sheet with a valid `bus_ID` and `Online_Year`.
2. Set `CAP` and `INTERCON_LIM`.
3. Set `Num_DR_Segments`, the segment shares `Pct_MW_1` to `Pct_MW_5`, and the prices `Price_1` to `Price_5`.
4. Set `Daily_DR_Limit_MWh` consistent with the optional segments (the optional share of `CAP` multiplied by 24 hours is the largest useful value).
5. For onsite resources, fill `Hybrid_Gen`, `Hybrid_Gen_CAP`, `Hybrid_ES`, `Hybrid_ES_CAP`; otherwise use `NA` and 0.
6. Optionally set `Profile_Tag` and the `timeseries_data_dc_path` file.

## Related documentation
- [Network Data Reference](./Network_Data_Reference.md)
- [Hybrid Resources Reference](./Hybrid_Resources_Reference.md)
- [Gen Technology Reference](./Gen_Technology_Reference.md)
- [Operation Outputs](../models/Operation/Operation_Outputs.md)
