# RA Metrics, ELCC, and DLOL

This page defines the reliability metrics reported by the Reliability Assessment (RA) model and the two capacity-credit measures derived from it, ELCC and DLOL. It explains how each is weighted, what it means, and how to choose between them. Settings are in [RA Settings and Scenarios Reference](./RA_Settings_and_Scenarios_Reference.md#capacity-credit-elcc-settings). Equations are in [RA Formulation](./RA_Formulation.md#reliability-metrics).

## Reliability metrics

| Metric | Unit | Definition |
|---|---|---|
| `EUE` (Expected Unserved Energy) | MWh per year | Probability-weighted annual energy that cannot be served. |
| `NEUE` (Normalized Expected Unserved Energy) | ppm | `EUE` divided by annual demand, multiplied by 1,000,000. |
| `LOLH` (Loss of Load Hours) | hours per year | Probability-weighted number of hours with unserved energy. |
| `LOLE` (Loss of Load Expectation) | days per year | `LOLH` divided by 24. |

Metrics are reported at annual and day-group levels, and at systemwide and regional levels where applicable. Only hours with unserved energy above a small numerical noise threshold count as loss-of-load events.

### How to interpret them
- `EUE` measures adequacy severity as an expectation, not a worst case. A rare scenario that sheds a large amount of energy contributes in proportion to its probability.
- `NEUE` normalizes by demand, so it is the appropriate metric for comparing systems or regions of different size and is common as a reliability target.
- `LOLH` measures the expected duration of shortfalls. Systemwide `LOLH` counts an hour once even if several buses are short in that hour. Regional `LOLH` counts each region's own shortfall hours.
- `LOLE` restates `LOLH` in days per year by dividing by 24. It is a unit conversion of `LOLH`, not a count of distinct days containing an event.

### Weighting over joint scenarios
Metrics are aggregated over the retained joint scenarios. Each joint scenario has weight equal to its renewable scenario weight divided by `num_risk_scenario`, which gives equal weight to each outage sample and the configured weights to renewable scenarios. Representative-day counts scale each day group to the year. Scenarios removed by risk filtering contribute zero. Weights are applied while accumulating, not by dividing by an outage-only total afterwards.

### Maximum metrics
`Max_Consecutive_Outage_Hours`, `Max_MWh_Loss`, and `Max_MW_Loss` are the largest values observed over the solved scenarios. They describe worst-case severity rather than expected severity, and they depend on the sample size and screening.

## Capacity credit: ELCC and DLOL
ELCC and DLOL are two mutually exclusive ways to express a resource's capacity credit. Choose one per run with `capacity_credit_type` in the `RA Setting` sheet (`ELCC` or `DLOL`) and enable the analysis with `calculate_capacity_credit_flag`. Both reuse the RA workflow, so they share its representative days, renewable scenarios, and metric definitions. Both report capacity credit as a fraction of installed capacity (0 to 1), not a percentage.

| | ELCC | DLOL |
|---|---|---|
| Basis | Reliability with and without the resource | Resource output during system shortfall hours |
| Cost | Iterative search, each step a full RA solve | Computed from the base RA run |
| Interactions | Captures interaction with the rest of the system | Reflects only observed dispatch |
| Best for | Formal accreditation studies | Fast screening |

Both apply to technologies with `ELCC_Flag` set to `true` in the `Gen Technology` sheet, and to buses with `RA_ELCC_Calculation_Flag` set to `true`.

## ELCC

### What it measures
ELCC (Effective Load Carrying Capability) is the additional constant load the system can serve at unchanged reliability because a resource was added. In the deactivation mode, it is the load relief needed to restore reliability after a resource is removed. The capacity credit is that equivalent load as a fraction of the resource's installed capacity.

### What is held constant
ELCC holds one reliability metric fixed at the value of the unperturbed system. `capacity_credit_reference_RA_metric` selects the metric (`EUE`, `NEUE`, `LOLH`, or `LOLE`), and `capacity_credit_reference_RA_metric_spatial_resolution` selects whether it is evaluated for the `Systemwide` system or per `Regional` area.

### What is varied
A constant load is added to demand and adjusted until the metric returns to its reference value, while the candidate resource is physically added to (or removed from) the network. The resolution setting also determines where the load is placed:

- `Systemwide`: spread across all buses in proportion to each bus's demand.
- `Regional`: placed entirely at the resource's bus. For `deactivate_existing`, the removed asset must be at a single bus.

The two assessment modes, set by `capacity_credit_assessment_mode`, are:

- `add_new`: adds a candidate unit at installed capacity (ICAP, the technology's nameplate `CAP` in MW, with no outage derating) and finds the largest constant load the system can then carry at the reference reliability.
- `deactivate_existing`: removes an existing asset (identified by plant name and unit group) and finds the load relief that restores the original reliability. The constant load is negative.

### Search procedure
ELCC is found by an iterative search on the constant load. Each trial load requires a full RA solve, and the gap `metric(constant load) - reference metric` is driven to zero in two phases:

1. **Bracketing.** The search starts with a lower bound of 0 and an upper bound equal to the resource's installed capacity, and doubles the upper bound until the gap changes sign or the iteration limit is reached.
2. **Refinement.** The next trial load is interpolated between the bracket ends (Illinois variant of regula falsi), kept strictly inside the bracket, and the bracket is updated. If interpolation is not usable, the search bisects.

### Convergence
The search stops when either condition holds:

- `|gap| <= abs_tol`, where `abs_tol` is `capacity_credit_abs_tol_value` in the units of the reference metric (MWh for `EUE`, ppm for `NEUE`).
- `|gap| / |reference metric| < tol`, where `tol` is `capacity_credit_rel_tol_value` interpreted as a percentage (`1.0` means 1%).

If `capacity_credit_max_iteration_value` is reached first, the result is flagged `best_effort` and ELCC is taken from the trial load with the smallest gap.

### Result
```
add_new:              ELCC = best constant load     / installed capacity of the added unit
deactivate_existing:  ELCC = equivalent load relief / installed capacity of the removed asset
```

The equivalent load in MW is stored as `equivalent_load_MW` (`add_new`) or `equivalent_load_relief_MW` (`deactivate_existing`). Each target also carries a `status`:

| Status | Meaning |
|---|---|
| `converged` | Tolerance reached. |
| `best_effort` | Iteration limit reached; best trial used. |
| `no_improvement_after_addition` | Adding the resource did not improve the reference metric (`add_new`). |
| `no_degradation_after_deactivation` | Removing the asset did not degrade the reference metric (`deactivate_existing`). |
| `regional_metric_bus_ambiguous` | A `Regional` target spans several buses (`deactivate_existing`). |
| `target_asset_not_found` | No asset matched the requested plant name and unit group (`deactivate_existing`). |

ELCC values are stored in the `capacity_credit_result` entry of the RA result JSON (`<Case_ID>__RA_result_stage_<N>.json`) and are passed to the next expansion round when capacity-credit feedback is enabled. There is no separate ELCC CSV. Dispatch files for ELCC iterations are written only when `export_ELCC_dispatch_results_flag` is `true` (see [RA Execution and Results](./RA_Execution_and_Results.md#what-ra-writes)).

!!! note
    Scenarios without any unserved energy in the base run are dropped before the ELCC search, because they cannot affect the reference metric. This speeds up the search and does not change the result.

## DLOL

### What it measures
DLOL (Direct Loss of Load) is a dispatch-based capacity-credit proxy. For an eligible unit, it is the unit's average output during system loss-of-load hours divided by its installed capacity, that is, its capacity factor during scarcity. It is dimensionless and does not measure a duration or a number of days.

### Calculation
A stress hour is any solved scenario hour with positive systemwide unserved energy, the same event condition as `LOLH`. For each eligible unit `i`, over all stress hours in all solved joint scenarios:

```
AvgOutputStress_i = sum( output_i(hour) * NumDays(hour) ) / sum( NumDays(hour) )
DLOL_i            = AvgOutputStress_i / ICAP_i
```

`output_i` is the unit's dispatched output in MW, `NumDays` is the representative-day count of the hour's day group, and `ICAP_i` is the unit's installed capacity in MW (`CAP` times `EXUNITS`). Unlike `EUE`, `LOLH`, and `LOLE`, DLOL is weighted only by representative-day counts. The joint-scenario weight cancels out of the ratio and is not applied.

### Result
DLOL is stored in the `capacity_credit_result` entry of the RA result JSON at three levels:

- **Unit level:** `ICAP_MW`, `AvgOutputStress_MW`, `DLOL`, and identifying fields.
- **Bus level, by unit group:** `unit_count`, `Avg_DLOL`, `ICAP_Weighted_Avg_DLOL`.
- **System level, by unit group:** `unit_count`, `Avg_DLOL`, `ICAP_Weighted_Avg_DLOL`.

The results are contained in the result JSON and are passed to the next expansion round when capacity-credit feedback is enabled. No separate DLOL CSV is written.

!!! warning "Interpreting DLOL"
    DLOL depends on the sample of shortfall hours. If the system has few or no loss-of-load hours in the sampled scenarios, there are few or no stress hours and DLOL is unreliable. Run with a stress level (`system_peak_scale`) and scenario count that produce enough shortfall hours.

## Practical checks
- Weighted metrics should move in the expected direction when renewable scenarios are added or the peak scale changes.
- Annual and day-group metrics should be consistent.
- Regional metrics should match the buses where the dispatch files show unserved energy.
- ELCC and DLOL results should be consistent with the underlying reliability metrics. Check that ELCC targets show status `converged`.

## Related documentation
- [RA Overview](./RA_Overview.md)
- [RA Formulation](./RA_Formulation.md#capacity-credit-elcc-and-dlol): equations for metric weighting, the ELCC search, and the DLOL roll-up
- [RA Execution and Results](./RA_Execution_and_Results.md)
- [RA Settings and Scenarios Reference](./RA_Settings_and_Scenarios_Reference.md#capacity-credit-elcc-settings)
- [GTEP Overview](../GTEP/GTEP_Overview.md): how capacity credit feeds the next expansion round
