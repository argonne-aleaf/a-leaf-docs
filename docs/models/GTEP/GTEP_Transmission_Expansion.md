# GTEP Transmission Expansion

This page describes how the GTEP model expands the transmission network: what the model can build, how expansion is priced, how the benefit of expansion depends on the power-flow mode, and how corridor aggregation and asynchronous DC ties are treated. Read it when a study allows transmission investment, or when you interpret `line_expansion_EXP` results.

---

## How transmission expansion is modeled

Transmission expansion **reinforces existing corridors only**. The model scales the rating of a branch that already exists. It does not create new corridors between bus pairs that are not already connected, and a branch with `rate_a = 0` gains nothing from expansion.

Expansion is described by two continuous, dimensionless quantities per branch and planning stage:

- the **new expansion** added on the branch in that stage (reported as `New_Expansion_Fraction`);
- the **cumulative expansion** through that stage (reported as `Cumulative_Expansion_Fraction`).

The transfer capacity of a branch is scaled multiplicatively:

```
rate_a * (1 + cumulative expansion)
```

A cumulative expansion of 1 doubles the branch. The build is continuous (a branch can grow by any fraction), and it is limited per branch by

```
cumulative expansion  <=  max_rate_a / rate_a - 1
```

If `max_rate_a` in the network data is smaller than `rate_a * (1 + transmission_expansion_limit_value)`, the model raises it to that value. With `transmission_expansion_limit_value = 2` (`Planning Design` sheet), every eligible branch can grow to about three times its base rating.

Only branches with `expansion_flag = True` in the network `branch` sheet are eligible. For all other branches the expansion is fixed at 0.

!!! note "Optional: fuller B-theta benefit via the enhanced hybrid"
    The multiplicative scaling above raises only the **thermal rating** of a branch, not its electrical parameters. In `B-theta` mode, upgrading a line in a meshed grid therefore does not let it carry proportionally more power (see [Behavior across power-flow modes](#behavior-across-power-flow-modes)). An opt-in **enhanced-hybrid** formulation removes this limitation in `B-theta` mode while the model remains a linear program. See [Enhanced-hybrid B-theta transmission expansion](#enhanced-hybrid-b-theta-transmission-expansion).

---

## Cost and units

Transmission investment is priced per MW of added capacity and per mile of corridor length:

- `transmission_cost_dollar_per_MW_mile_value` (per case, in the `Simulation Configuration` sheet) is the overnight cost in dollars per MW-mile.
- `transmission_route_length_adder_value` (`Planning Design` sheet, default `1.0`) scales straight-line length to an approximate routed length.
- `transmission_investment_CRP_value` (`Planning Design` sheet) is the capital recovery period in years, and `WACC_value` is the cost of capital.

The overnight cost is annualized with the same capital recovery factor as generation,

```
CRF = WACC * (1 + WACC)^CRP / ((1 + WACC)^CRP - 1)
```

and paid over `min(transmission_investment_CRP_value, remaining modeled years)`, discounted to `dollar_year_value`. Because generation and transmission investments are annualized and discounted the same way, their costs are directly comparable in the objective. Line length is used in miles. The network `Length` column is stored in kilometers and the model converts it to miles (multiplied by 0.621371), which matches the `$/MW-mile` cost unit.

**Marginal cost.** Adding 1 MW of transfer capacity over an AC corridor costs `transmission_cost_dollar_per_MW_mile_value * length * route adder` overnight. The model selects a line only when the congestion cost it relieves, summed over the modeled hours and years, exceeds that. Asynchronous DC ties are priced differently: see [Asynchronous DC-tie modeling](#asynchronous-dc-tie-modeling-b-theta).

---

## Transmission fixed O&M (FOM)

Transmission carries an annual fixed operations-and-maintenance charge, set by `transmission_FOM_percent_value` in the `Planning Design` sheet. The default is `0.0`, which turns the charge off.

The charge for each branch is a percentage of the branch's overnight investment basis (the same basis as the investment cost, before annualization):

```
transmission_FOM_percent_value * 0.01 * overnight cost * rate_a * length * (1 + cumulative expansion)
```

with `length = 1` for DC ties. The charge is applied in every operating year and discounted.

- **In the optimization**, only the FOM on the expansion increment is included. The model cannot retire existing transmission, so the FOM of the existing grid is a constant that cannot change any decision. Leaving it out does not change the optimal build.
- **In the reported costs**, the FOM covers the full in-service grid (existing capacity plus expansion). It appears in the `Trans_FOM_Cost` and `Trans_FOM_Cost_PV` columns and is included in `total_system_cost`. In the operation model it is reported but excluded from `Operating_Cost`.

See [GTEP Expansion Outputs](./GTEP_Expansion_Outputs.md#11-system-summary-output) and [Operation Outputs](../Operation/Operation_Outputs.md#7-system-summary-outputs).

---

## Behavior across power-flow modes

The power-flow mode (`power_flow_mode_flag`, `Simulation Setting` sheet) determines how much benefit a line gets from expansion, because only the thermal limit is scaled by `(1 + expansion)`, not the electrical parameters of the line.

| Mode | Expansion enabled? | Benefit of expansion | Flow definition |
|---|---|---|---|
| `Network_Flow` | Yes (`expansion_flag`) | **Full.** The thermal limit `\|f\| <= rate_a * (1 + expansion)` is the only limit. | Transport model: flows are limited only by branch ratings. |
| `B-theta` | Yes | **Partial** by default: the thermal limit grows but the susceptance stays fixed. **Full** with the optional [enhanced hybrid](#enhanced-hybrid-b-theta-transmission-expansion). | `f = (1/br_x_pu) * (theta_from - theta_to)`. For aggregated AC corridors, `br_x_pu` is estimated from network flows (see [Corridor aggregation](#corridor-aggregation)). |
| `PTDF` | Yes | **Partial:** the thermal limit grows but the shift factors stay fixed. | `f = sum(ptdf * injection)`. PTDF rows are built from the final branch list, so `PTDF` and `B-theta` describe the same network. |

Branch flows are lossless in every mode. The only loss representation is the optional flat, system-wide transmission and distribution demand gross-up (`enforce_transmission_loss_flag` and `transmission_loss_percent_value`), which does not vary with line loading or distance. See [GTEP Overview](./GTEP_Overview.md#transmission-and-distribution-losses).

!!! warning "DC modes under-value expansion by default"
    In `B-theta` and `PTDF` modes, expanding a line raises its thermal rating but does not lower its impedance or change its shift factors. In a meshed grid, the upgraded line cannot then carry proportionally more power, so these modes build less transmission than `Network_Flow` does for the same congestion. Scaling the susceptance with the continuous expansion (`b * (1 + expansion)`) would make the flow equation bilinear and non-convex, which is why the standard model relaxes only the thermal limit.

    In `B-theta` mode, `transmission_expansion_hybrid_flag` enables the [enhanced-hybrid formulation](#enhanced-hybrid-b-theta-transmission-expansion), which restores the full expansion benefit while keeping the model a linear program. `PTDF` mode has no equivalent option, so `PTDF` runs under-value expansion.

---

## Enhanced-hybrid B-theta transmission expansion

The enhanced-hybrid formulation lets an expanded AC line in a `B-theta` model carry more angle-driven flow, not just have a higher rating. The model stays a linear program (no integer variables). The formulation is opt-in. With the flag off, the standard formulation applies.

### What to set

Set `transmission_expansion_hybrid_flag` to `TRUE` in the `Simulation Setting` sheet (see the [flag reference](../../configuration/ALEAF_Simulation_Setting_File.md#transmission_expansion_hybrid_flag)) and use `power_flow_mode_flag = B-theta`. With `Network_Flow` (which has no angle law to relax) or `PTDF`, the flag is ignored, the model logs a warning and the standard formulation runs.

### Which corridors are affected

The treatment applies to a corridor when the hybrid is active, the branch has `expansion_flag = True`, and the branch is an AC line (`dc_line` is not `True`). Non-expandable AC lines and all DC ties keep the standard formulation (see [DC ties keep the standard limit](#dc-ties-keep-the-standard-limit)).

### Split flow: base channel plus expansion increment

For an eligible corridor, the flow is split into two parts:

- The **base flow** \(f\) follows the usual angle law \(f = b_0(\theta_f - \theta_t)\), where \(b_0 = 1/\texttt{br\_x\_pu}\). Its thermal limit is fixed at `rate_a`, that is \(|f| \le \texttt{rate\_a}\). This also limits the angle difference to \(|\Delta\theta| \le \texttt{rate\_a}/b_0\).
- The **expansion increment** \(f_{exp}\) is the extra flow carried by the built headroom. It is signed, so counterflow within the unbuilt headroom is allowed.

The total corridor flow is \(f + f_{exp}\), and this total enters the nodal power balance at both ends.

In exact physics, a line upgraded to admittance \(b_0(1+u)\), with build \(u\), carries \(f_{exp} = b_0\,u\,\Delta\theta = u\,f\). Because both \(u\) and \(f\) are variables, this product is non-convex. The model replaces it with its **McCormick convex hull** over \(u \in [0, \bar u]\), where \(\bar u = \texttt{max\_rate\_a}/\texttt{rate\_a} - 1\) is the same per-corridor cap that limits the expansion:

\[
|f| \le \texttt{rate\_a}
\]

\[
|f_{exp}| \le \texttt{rate\_a}\cdot u
\]

\[
\bar u\, f - \texttt{rate\_a}(\bar u - u) \;\le\; f_{exp} \;\le\; \bar u\, f + \texttt{rate\_a}(\bar u - u)
\]

!!! note "Properties of the relaxation"
    - **Exact for an unbuilt line** (\(u = 0\)): the second constraint forces \(f_{exp} = 0\), so the corridor carries only the base flow.
    - **Exact for a fully built line** (\(u = \bar u\)): the third constraint collapses to \(f_{exp} = \bar u\, f\), the exact flow of the line with admittance \(b_0(1+\bar u)\).
    - **Tightest linear relaxation** of \(f_{exp} = u f\) over the box, given \(f \in [-\texttt{rate\_a}, \texttt{rate\_a}]\).
    - **Largest gap at partial builds.** The gap \(f_{exp} - u f\) is zero at both endpoints and largest at mid-build. It is reported per corridor-hour as `KVL_Residual_MW` in `power_flow_EXP.csv` (see [GTEP Expansion Outputs](./GTEP_Expansion_Outputs.md#5-power-flow-output)), together with `Expansion_Flow_MW` and `Total_Flow_MW`.

### DC ties keep the standard limit

An asynchronous DC tie is angle-decoupled: it is already exact as a bounded transfer and has no impedance to scale. It is therefore excluded from the hybrid split. Its expansion uses the standard limit `rate_a * (1 + expansion)`. See [Asynchronous DC-tie modeling](#asynchronous-dc-tie-modeling-b-theta).

### Operation and RA runs that use a fixed expansion

When an expansion result is handed to a later Operation or RA run, the build is a fixed number rather than a variable, so the model applies the physics exactly: for every AC branch it divides the reactance by `(1 + expansion)`, that is `br_x_pu / (1 + expansion)`, and it keeps the thermal limit `rate_a * (1 + expansion)`. DC ties are not modified. No split flow is used in these runs.

### Expected results

The hybrid formulation typically builds more transmission than the standard `B-theta` model, because expansion now relieves angle-driven congestion. Compare `Total_Flow_MW` with `Final_Rating_MW` in `power_flow_EXP.csv` to see how much of a corridor's expanded capacity is used.

---

## Corridor aggregation

When the network is modeled at a coarser resolution than the database (see [Network Resolution](../../database/Network_Resolution.md)), the branches between two aggregated buses are combined into a single corridor. For a corridor of `N` constituent circuits, the model computes:

- **Capacity** `rate_a` as the sum of the member ratings, and `max_rate_a` as the sum of the `max_rate_a` of the expandable members.
- **Effective length** as the capacity-weighted average of the member lengths for multi-circuit corridors (`N > 1`), so a bundle of parallel circuits is priced at about its true MW-mile total rather than at length times capacity of the whole bundle. A single-circuit corridor uses its own length.
- **Reactance** `br_x_pu` initially as the parallel combination `1 / sum(1/x_i)`. For aggregated AC corridors this initial value is then replaced by an estimate (see below).

!!! note "Why `rate_a` is the plain capacity sum"
    A DC-consistent alternative would limit a corridor to `b0 * min_i(rate_a_i * x_i)`, as if the members were electrically parallel. The model does not use it. Region corridors bundle lines with different nodal endpoints, whose true deliverability is close to the sum of the member ratings and not the weakest single member. The alternative would cut multi-leg imports to a fraction of their capacity and produce large amounts of unserved energy.

### Corridor reactance for aggregated AC corridors

When aggregation merges AC lines and the power-flow mode is `B-theta` or `PTDF`, the model estimates one susceptance for each aggregated corridor so that the reduced network reproduces the cross-border flows of the detailed network. The estimate uses synthetic operating snapshots built from bus peak loads and plant capacities. It uses no time series or dispatch. See [PTDF / snapshot network reduction](#ptdf-snapshot-network-reduction).

The parallel-combination reactance is kept for:

- DC ties, which are angle-decoupled and excluded from the estimate;
- lines kept separate at the finest resolution (see below);
- runs at the database resolution, where no AC lines are merged;
- islands in which the estimate does not improve on the parallel combination for held-out snapshots. The whole estimation is skipped if it fails.

### Mixed resolution: kept lines and merged corridors

A single network can mix fine and aggregated regions. When both endpoints of a bus pair are modeled at the finest level of a finer-resolution (nodal) database, every physical line between them is kept separate. Each keeps its own `rate_a`, `br_x_pu` and thermal limit, and the lines share angles through the ordinary angle law. A corridor that touches at least one aggregated bus is merged as described above. See [Network Resolution](../../database/Network_Resolution.md).

---

## PTDF / snapshot network reduction

The snapshot reduction is used to estimate corridor reactances after aggregation (see above) and to build shift factors for `PTDF` mode. It never reads time series. It fits corridor susceptances from synthetic operating snapshots solved on the detailed network.

- **Fit.** The reduction estimates one susceptance per region-pair corridor so that the reduced network reproduces the detailed network's cross-border flows over the snapshots. Lines kept at the finest resolution enter as fixed edges. It also produces diagnostics such as the held-out flow error and the number of islands and corridors.
- **Snapshots.** In each snapshot, bus load is drawn as a multiple of its peak load, wind and solar output as a multiple of plant capacity, and dispatchable capacity covers the residual. Plant classes are taken from `UNIT_CATEGORY` (wind, solar, dispatchable). Storage injects nothing.
- **Reproducibility.** The random draws are keyed by bus id, so the same corridor reactances are produced on every process, platform and Julia version.
- **Fallback.** If the estimate does not beat the parallel combination on held-out snapshots, that island keeps the parallel combination.
- **PTDF rows.** In `PTDF` mode, each branch's shift-factor row is computed from the final branch list, so `PTDF` and `B-theta` describe the same network. DC ties get zero rows.

This procedure is also described in [Network Resolution](../../database/Network_Resolution.md#ptdf-snapshot-network-reduction).

---

## Asynchronous DC-tie modeling (B-theta)

A-LEAF represents **asynchronous DC ties**, that is back-to-back HVDC links and variable-frequency transformers (VFTs), which transfer power between two interconnections without synchronizing their AC angles. A branch is a DC tie when its `dc_line` column is `True` in the network `branch` sheet (see [Network Data Reference](../../database/Network_Data_Reference.md#dc_line-asynchronous-dc-vft-ties)).

### Identifying a DC tie

When parallel circuits between a bus pair are merged into one corridor, the corridor is a DC tie only if **every** constituent line is a DC tie. Any AC line in the bundle makes the corridor AC.

### Power-flow treatment by mode

- **`B-theta`:** a DC tie is excluded from the angle equation, so it does not synchronize the two interconnections. It acts as a **controllable, bounded, lossless transfer** with `-rate_a <= f <= rate_a * (1 + expansion)` that enters the nodal balance at both ends.
- **`Network_Flow`:** ties are transfer limits, as for any branch.
- **`PTDF`:** DC ties have zero shift factors, matching their angle-decoupled treatment in `B-theta`. Their bounded-transfer limits still apply.

### Per-island reference bus

Because DC ties do not couple the interconnections, each connected AC island needs its own angle reference. The model fixes one reference angle (angle = 0) per connected component of the AC-only network, at the bus with the lowest bus id in each component. When the whole network is one AC component, this is the usual single reference bus. The same rule applies in expansion, operation and RA runs.

### Expansion cost — two converter stations plus AC approach-line distance

DC-tie expansion is priced as two converter stations, one on each side, plus the AC lines that connect each side to the converter station. A back-to-back or VFT facility has essentially zero DC distance, but the AC approach lines have real length.

The overnight cost per MW is

```
2 * dc_tie_expansion_cost_dollar_per_MW_value
  + transmission_cost_dollar_per_MW_mile_value * length_miles * route adder
```

where `dc_tie_expansion_cost_dollar_per_MW_value` (per case, `Simulation Configuration` sheet) is the cost of a single converter station in dollars per MW, and `length_miles` is the branch `Length` converted from kilometers to miles. The route adder (`transmission_route_length_adder_value`) multiplies the per-mile term only, not the converter cost. The result is annualized with the same capital recovery factor as AC lines and discounted like any other transmission investment. The length is already part of this per-MW cost, so it is not applied a second time in the objective.

!!! example "Worked example"
    Consider a tie with a raw `Length` of 22.8 km (14.17 miles), `dc_tie_expansion_cost_dollar_per_MW_value = 150000`, `transmission_cost_dollar_per_MW_mile_value = 1666`, a route adder of 1.0, `WACC_value = 0.054` and `transmission_investment_CRP_value = 40`:

    ```
    overnight cost = 2 * 150000 + 1666 * 14.1673 = about 323,603 $/MW
    CRF            = 0.054 * 1.054^40 / (1.054^40 - 1) = about 0.0615
    annual cost    = 323,603 * 0.0615 = about 19,903 $/MW-year
    ```

!!! note "Network data for DC ties"
    The `Length` of a DC tie in the network data is the route distance of its AC approach lines, in kilometers, like every other branch. When you add or edit DC ties, use real route distances rather than a placeholder, because the length enters the expansion cost. Under coarse aggregation, DC ties that connect the same bus pair merge into a single corridor, so fewer corridors than raw ties can appear.

### Limitations

!!! warning "Converter economic life follows the AC line assumption"
    DC converter cost is amortized over the same `transmission_investment_CRP_value` and `WACC_value` as AC lines. A converter-specific recovery period is not available.

!!! note "The DC-tie cost applies only to expandable ties"
    The DC-tie cost enters the optimization only when the tie has `expansion_flag = True` and `max_rate_a > rate_a`. With `expansion_flag = False` the tie has fixed capacity. To make a tie expandable, set `expansion_flag = True` and `max_rate_a > rate_a` in the network `branch` sheet.

---

## Limitations

- **Existing corridors only.** The model does not build new corridors between unconnected buses.
- **Formula-based expansion ceiling.** The per-branch expansion ceiling comes from `max_rate_a` and the global `transmission_expansion_limit_value`, not from a calibrated interface limit. Interpret absolute build totals accordingly.
- **No transmission retirement.** The model has no transmission-retirement decision.
- **Lossless branches.** Line losses are not modeled per branch.
- **`PTDF` mode.** Expansion is under-valued, and the hybrid formulation is not available.
- **Continuous build.** Expansion is continuous and has no block size or economies of scale.

## Related documentation
- [GTEP Overview](./GTEP_Overview.md)
- [GTEP Formulation](./GTEP_Formulation.md)
- [GTEP Planning Horizon and Multi-Round](./GTEP_Planning_Horizon_and_Multi_Round.md)
- [GTEP Expansion Outputs](./GTEP_Expansion_Outputs.md)
- [Network Data Reference](../../database/Network_Data_Reference.md)
