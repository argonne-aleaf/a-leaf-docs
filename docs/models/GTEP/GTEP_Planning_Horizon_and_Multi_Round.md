# GTEP Planning Horizon and Multi-Round

This page explains how the planning horizon of a generation and transmission expansion (GTEP) study is defined, and how the optional multi-round (rolling-horizon) solution breaks that horizon into overlapping subproblems. Read it when you set up the `Planning Design` sheet of the simulation workbook or when a full-horizon solve is too large to run in one piece.

## How the planning horizon is defined

The horizon is set in the `Planning Design` sheet of the simulation workbook. The model divides it into equal-length **planning stages**. Investment and retirement decisions are made once per stage.

| Setting | Meaning |
|---|---|
| `base_year_value` | Year that the input load data represents. Load growth compounds from this year: the growth factor for year *y* is `(1 + growth)^(y - base_year_value)`. It does not select time-series data and does not set the discounting year. |
| `first_stage_year_value` | Calendar year of stage 1. |
| `num_stages_value` | Number of planning stages. |
| `num_years_per_stage_value` | Number of years each stage represents. |
| `horizon_end_year_value` | Information only. The model does not use it and logs a warning if it differs from the derived value. |
| `dollar_year_value` | Dollar year of all cost inputs and the year to which all costs are discounted. |

Stage *k* is labelled with the calendar year

```
first_stage_year_value + (k - 1) * num_years_per_stage_value
```

and the last modeled year is

```
first_stage_year_value + num_stages_value * num_years_per_stage_value - 1
```

Each run logs the horizon and the stage years at the start of the case.

!!! note "Renamed settings"
    Workbooks must use `base_year_value` and `first_stage_year_value`. The settings `current_year_value`, `lead_year_value`, `targetyear_value` and `final_year_value` are not read. Use `base_year_value` where `current_year_value` was used, and `first_stage_year_value` for `current_year_value + lead_year_value`.

### Example

With

- `base_year_value = 2024`
- `first_stage_year_value = 2030`
- `num_stages_value = 4`
- `num_years_per_stage_value = 5`

the model builds four five-year stages:

| Stage | Calendar year of the stage | Years represented |
|---|---|---|
| 1 | 2030 | 2030-2034 |
| 2 | 2035 | 2035-2039 |
| 3 | 2040 | 2040-2044 |
| 4 | 2045 | 2045-2049 |

Load growth compounds from 2024, so stage 1 already carries six years of growth. The last modeled year is 2049.

### How stage length is used

Only the first year of each stage is dispatched in detail, using representative days. The results are scaled to the other years of the stage. The stage length therefore affects:

- **Cost accounting.** Annual cost terms (fixed O&M, operating cost, scarcity cost, carbon cost and policy penalties) are extended over the years the stage represents.
- **Investment recovery.** The number of years over which an investment (or a tax credit) is paid is the shorter of the asset's cost-recovery period and the number of modeled years remaining after the stage year.
- **Annualized reporting.** The annual output files expand each stage over its years. See [GTEP Expansion Outputs](./GTEP_Expansion_Outputs.md).

### Discounting

Present-value cost terms (investment, fixed O&M, fuel and operating cost, tax credits and penalties) are discounted to `dollar_year_value`. The `_real_<year>usd` summary files remove the discount and report costs incurred in each year in constant dollars of that year.

## Standard single-run expansion

When `multi_round_solution_process_flag` is `false`, the model solves the whole horizon as one problem. All stages are optimized jointly, so early investment decisions account for later stages. Reports are written once from the full solution.

## Multi-round expansion

When `multi_round_solution_process_flag` is `true`, the model solves the horizon in sequential **rounds** instead of one large problem. Each round covers a window of consecutive stages. Only the first part of the window is committed. The stages at the end of the window are **look-ahead** stages: they give the committed decisions a view of the near future, but they are solved again in the next round.

Multi-round mode helps when:

- the full-horizon problem is too large for memory or solve time;
- the study should represent planners who commit to near-term builds without perfect foresight of later stages;
- capacity credits (ELCC) should be updated between rounds using resource adequacy results.

### Settings

These settings are in the `Planning Design` sheet.

| Setting | Meaning |
|---|---|
| `multi_round_solution_process_flag` | `false`: one optimization over the whole horizon. `true`: rolling multi-round optimization. |
| `num_decision_stages_per_round_value` | Number of stages whose build decisions are committed in each round. Must be at least 1. |
| `num_lookahead_stages_per_round_value` | Number of additional future stages modeled in each round but not committed. Must be 0 or more. `0` gives a purely myopic run. |
| `continue_from_previous_run_flag` | Resume an interrupted multi-round run from the last completed round. |

Each round models `num_decision_stages_per_round_value + num_lookahead_stages_per_round_value` stages. The next round starts after the committed stages, so consecutive rounds overlap by the number of look-ahead stages. The last round commits every stage it contains. Look-ahead stages reduce end-of-horizon effects inside each round, at the cost of larger subproblems.

!!! note "Renamed settings"
    The settings `num_stages_per_simulation_round_value` (equal to decision plus look-ahead stages) and `num_overlaps_between_simulation_rounds_value` (equal to look-ahead stages) are not read. Use the two settings above.

### Worked example

Take a horizon of 6 stages with `num_decision_stages_per_round_value = 2` and `num_lookahead_stages_per_round_value = 1`. Each round models 3 stages:

| Round | Stages modeled | Stages committed | Look-ahead stage |
|---|---|---|---|
| 1 | 1-3 | 1-2 | 3 |
| 2 | 3-5 | 3-4 | 5 |
| 3 | 5-6 | 5-6 | none (last round) |

Round 2 starts from the builds committed in round 1. Round 3 reaches the end of the horizon, so it commits both of its stages.

With `num_decision_stages_per_round_value = 1` and `num_lookahead_stages_per_round_value = 0`, each stage is solved by itself in its own round.

### How decisions carry forward

Builds and retirements committed in earlier rounds are treated as existing capacity in later rounds. This applies to generation, storage and transmission expansion.

!!! warning "Myopic runs do not give a horizon net present value"
    In a round with no look-ahead, new investment is paid over only the years the round window covers, not over the full cost-recovery period. Summing the per-round `ObjectiveValue` values therefore understates lifetime capital cost. See [GTEP Expansion Outputs](./GTEP_Expansion_Outputs.md#11-system-summary-output). For a horizon-wide cost comparison, use a single-run expansion.

## Relationship to RA and capacity credit

Multi-round mode can update capacity credits between rounds using the resource adequacy (RA) model. When `update_CAPCRED_in_each_round_of_Expansion_Flag` is `TRUE` in the `Simulation Configuration` sheet for the case, the model runs RA after a round and applies the resulting ELCC (Effective Load Carrying Capability, the reliability-based capacity credit RA computes for a technology; see [RA Metrics, ELCC, and DLOL](../RA/RA_Metrics_ELCC_and_DLOL.md)) values to eligible technologies in the next round. The first round uses the capacity credits from the input data.

## How multi-round reporting is produced

The model writes the expansion reports per round and assembles them at the end of the run. As each round completes, its decision-stage report pieces are written to a temporary `_report_parts/round_<r>/` directory in the case output directory. At the end of the run the pieces are concatenated into the standard expansion CSV files (the per-stage dispatch files are moved), and the temporary directory is removed. The final file names, columns and values are the same as for a single-run expansion.

Each round honors the same report settings from the `Simulation Setting` sheet:

- `report_expansion_flag`
- `report_scarcity_EXP_flag`
- `report_power_flow_EXP_flag`
- `report_dispatch_EXP_flag`
- `report_summary_EXP_flag`

For the file-by-file description, see [GTEP Expansion Outputs](./GTEP_Expansion_Outputs.md).

## Continuing a prior multi-round run

Set `continue_from_previous_run_flag` to `true` to resume an interrupted run. The model looks for `GTEP_multi_round_info.json` in the case output directory and validates it before doing anything else.

- **File integrity.** A corrupt or truncated file stops the run with an error.
- **Compatibility.** The file must have been written by the same case with the same number of stages per round, the same overlap (look-ahead) and the same horizon. It must also use the current checkpoint format. Any mismatch stops the run and asks you to delete the stale file and start fresh. Files from older versions cannot be resumed.

If validation passes, the recorded status decides what happens:

| Status | Behavior |
|---|---|
| `completed` | The model warns that the run already finished and does nothing. Delete `GTEP_multi_round_info.json` to start over. |
| `in progress` | The model restores the committed decisions, per-stage generation and cost aggregates, round objectives, RA information and the `_report_parts/` pieces from the interrupted run, then continues from the round after the last completed one. Completed rounds are not solved again. |

!!! note "Resumed runs can differ slightly in objective value"
    Rounds solved after the resume point run in a fresh solver process. The `ObjectiveValue` of those stages can differ from an uninterrupted run by about 1e-6 in relative terms. All other outputs match.

## Multi-round output artifact

The multi-round run writes `GTEP_multi_round_info.json` in the case output directory after every round. The file is the resume checkpoint and also the hand-off from the expansion model to the RA model and to operation runs that reuse a predefined expansion (`predefined_expansion_data_file_name_for_RA`). It is a working file, not a results report. Each round is written atomically, so an interrupted write cannot leave a partial checkpoint.

For every round (keys `"1"`, `"2"`, ...) the file stores:

| Entry | Content |
|---|---|
| `solution` | Investment and retirement decisions of the round (categories `expansion` and `expansion_line`). Dispatch and dual values are not stored. |
| `recorded_investment_decisions` | Accumulated decisions entering the round. |
| `updated_investment_decisions` | Accumulated decisions after the round commits. |
| `RA_Info` | RA metrics, ELCC and capacity-credit results linked to the round. |
| `annual_gen_info` | Generation and cost aggregates of the decision stages. |
| `objective` | Objective value of the round. |
| `repday_meta` | Representative-day metadata of the decision stages, used to rebuild the representative-day report on resume. |
| `round_ids_y`, `round_ids_y_decision`, `round_first_y`, `round_last_y` | Stages modeled in the round, stages committed, and the first and last stage of the window. |

At the top level the file records the checkpoint format version, `case_id`, the round configuration, the last planning stage, `status` (`in progress` or `completed`) and the index of the last completed round.

When `report_multi_round_summary_json_EXP_flag` is `FALSE`, the file is deleted after the whole run completes successfully. Keep it (the default is `TRUE`) if you plan to run RA or operation from the expansion result later, or to resume the run.

## Relationship to representative days

Planning stages define the long-term year blocks. Representative days define the reduced chronology inside each stage. In multi-round mode, the round logic changes which stages are active in a solve, but each active stage keeps its own representative days. See [Scenario Reduction and Representative-Day Groups](../../configuration/Scenario_Reduction_and_Repday_Groups.md).

## Checklist

When reviewing a case, check the horizon in this order:

1. `base_year_value`, `first_stage_year_value`, `num_stages_value` and `num_years_per_stage_value` in `Planning Design`.
2. Whether `multi_round_solution_process_flag` is enabled.
3. The number of decision and look-ahead stages per round.
4. Whether capacity credits are updated between rounds.
5. Whether `continue_from_previous_run_flag` is set, and whether an existing checkpoint is intended to be reused.

## Related documentation

- [GTEP Overview](./GTEP_Overview.md)
- [Scenario Reduction and Representative-Day Groups](../../configuration/Scenario_Reduction_and_Repday_Groups.md)
- [Simulation Configuration Reference](../../configuration/Simulation_Configuration_Reference.md)
- [Policy and Financial Settings](../../configuration/Policy_and_Financial_Settings.md)
