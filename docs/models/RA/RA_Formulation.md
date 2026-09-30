# RA Resource Adequacy — Mathematical Formulation

!!! tip "Looking for a gentler introduction first?"
    This page is the equation-level reference for every constraint and metric of the RA
    model. To understand the RA workflow, run modes, and outputs in plain language, start with
    [RA Overview](./RA_Overview.md) and return here when the underlying mathematics is needed.

This page is the **canonical mathematical reference** for the A-LEAF Resource Adequacy
(RA) model: a two-stage, scenario-based reliability assessment that (1) solves a
"risk-free" reference unit-commitment / economic dispatch for each representative
day-group and renewable scenario, then (2) re-solves a post-contingency dispatch for
every retained generator-outage × renewable joint scenario, and finally (3) aggregates
the unserved-energy outcomes into [reliability metrics](#reliability-metrics) — EUE
(Expected Unserved Energy), NEUE (Normalized EUE), LOLH (Loss-of-Load Hours), and LOLE
(Loss-of-Load Expectation) — and [capacity-credit measures](#capacity-credit-elcc-and-dlol)
— ELCC (Effective Load Carrying Capability) and DLOL (Direct Loss of Load). Each acronym
is defined, with a worked numeric example, in the linked section.

The page is organised as follows: [scenario generation](#scenario-generation), the
[two-stage structure](#two-stage-structure), the [reference dispatch (Stage 1)](#stage-1-reference-risk-free-uced),
the [post-contingency dispatch (Stage 2)](#stage-2-risk-realization-post-contingency-dispatch),
the [reliability metrics](#reliability-metrics), [capacity accreditation](#capacity-credit-elcc-and-dlol),
and finally the [RA-specific notation](#additional-notation-ra-only).

RA reuses the shared nomenclature defined on the
[GTEP formulation page](../GTEP/GTEP_Formulation.md) (sets, parameters, variables,
and the `(d,h,t,y)` time indexing) and the operational dispatch primitives documented on
the [Operation formulation page](../Operation/Operation_Formulation.md). Only the
RA-specific symbols — the scenario index, the post-contingency dispatch and deviation
variables, and the outage sets — are introduced in the [notation section](#additional-notation-ra-only).

!!! note "Structural properties of the RA model"
    Four properties shape almost every constraint family below.

    - **The reference dispatch carries no operating reserves.** The base dispatch has no
      regulation, spinning, flexibility, or non-spinning reserve variables, requirements, or
      costs. RA asks how much load the fleet's *energy* can serve, not how much load plus a
      reserve margin; reserve adequacy is out of scope.
    - **The post-contingency stage redispatches freely with an L1 deviation penalty rather
      than deploying pre-committed reserves.** A single absolute-deviation variable
      $\delta^{s}$ penalises departure from the reference dispatch, subject to ramp bounds
      and a system-wide "replace-the-lost-MW" cap. This mirrors a dispatcher's response to a
      forced outage: re-optimise generation given the surviving fleet rather than deploy
      reserve blocks booked in advance.
    - **RA models generator outages only.** The outage sampler produces a per-hour
      generator up/down mask; no transmission-line outages are generated, so the line-outage
      set $\mathcal{K}^{LC}_s$ is empty in every run and a transmission contingency (for
      example an N-1 line trip) is not part of the reliability estimate.
    - **RA has no expansion and no operating-reserve scarcity term.** Unlike the
      GTEP and Operation objectives, neither RA stage prices reserve shortfalls; the only
      scarcity term is $\text{VOLL}\cdot ens$, where VOLL (Value of Lost Load, \$/MWh) is
      the same parameter used to price unserved energy throughout A-LEAF.

---

## Scenario generation

The randomness in RA's reliability estimate is created here. Everything downstream
(Stage 1 and Stage 2 dispatch, metric aggregation) is deterministic given a set of
scenarios: the outage draws, the renewable scenarios, and the choice of which combinations
are simulated determine what risk RA measures.

### Outage samples ($\mathcal I^{GC}_s$)

Each generator's hourly up/down status is drawn from a two-state semi-Markov process over
the day-group horizon, carried across day-group boundaries: a generator still under repair
at the end of one day-group's simulated hours stays down into the next day-group rather than
being reset to "available." This reflects real forced-outage behavior (repairs often take
longer than a day) and avoids the optimistic bias of starting every representative day fully
available. Per hour the process uses a **temperature-dependent forced-outage rate**
(available-to-down transition probability, `rateAD`) and a **repair distribution**
(down-to-down transition probability, `rateDD`):

- For a string forced-outage-rate (FOR) class, both rates are logistic in the local
  temperature deviation $|\,\bar\tau_{n,t}-\tau^{ref}\,|$ from the reference temperature,
  with separate high-side and low-side quadratic regressions. This captures
  weather-correlated forced-outage risk, for example gas units tripping disproportionately
  during a cold snap, rather than assuming a temperature-independent failure probability.
  The reference temperature $\tau^{ref}$ (`reference_temp`) is the point at which the
  regression's baseline (non-stress) rate applies; deviations above and below it use
  separate quadratic fits because temperature sensitivity is typically asymmetric.
- For a numeric FOR, $\texttt{rateDD}=1-1/\text{MTTR}$ and
  $\texttt{rateAD}=\text{FOR}/(\text{MTTR}\,(1-\text{FOR}))$ give the hourly transition
  probabilities from the two quantities a planning dataset typically supplies: a
  mean-time-to-repair in hours and a steady-state forced-outage rate. This is the classical
  two-state Markov identity
  $\text{FOR}=\texttt{rateAD}/(\texttt{rateAD}+1/\text{MTTR})$ solved for `rateAD`, so the
  long-run fraction of time the unit is unavailable converges to the input FOR.

An online unit fails with probability `rateAD`; on failure a repair time is drawn from
$\mathrm{Geometric}(1-\texttt{rateDD})$, truncated at the `repair_time_bound`
(`average` / `95` / `90` percentile / `NA`), and the unit is held down for that many hours,
spilling into subsequent day-groups. `repair_time_bound` is a risk-conservatism setting:
`average` produces the typical (mean) outage duration, while `95 percentile` and
`90 percentile` cap a drawn repair time at that percentile of the repair-time distribution,
which prevents a single unusually long draw from dominating one scenario while still
allowing meaningfully long outages. The result is a per-scenario hours-by-generators
matrix of up/down states (`true` = online), which becomes the availability mask
$a^{s}_{i,d,h,t}$.

A hybrid asset's storage and its co-located generator are forced to a **single** shared
state (logical AND of the two draws), so the hybrid outages as one unit. The two typically
share point-of-interconnection equipment, so a failure of either takes the combined asset
offline for reliability counting.

### Renewable scenarios ($\mathcal R$)

Where the outage sampler captures *generator* uncertainty, renewable scenarios capture
*weather and resource* uncertainty: the same wind and solar fleet can produce very different
output across historical years or synthetic weather draws, which in a VRE-heavy system is
at least as important to reliability as generator failures. Renewable scenarios come from
the `RA Scenarios` sheet (`Scenario_ID`, `Enabled`, `Weight`, `Wind_Ons_File_ID`,
`PV_File_ID`; base = `BASE`). Enabled rows are read and their weights **normalised to sum to
one**; each supplies a wind-onshore and a PV hourly time series that is mapped onto the
representative day-group hours. With three enabled rows, for example `BASE` (weight 0.5),
`LowWind` (0.25), and `HighWind` (0.25), every retained outage sample is solved three times
(once per renewable scenario), each carrying that scenario's normalized weight into
$\pi_s=w_{r(s)}/N^r$. Setting `Enabled=false` removes a row from both the solve set and the
weight normalisation, so the remaining weights are rescaled to sum to one.

### Joint scenarios and filtering ($\mathcal S^{sc}$, $\mathcal S^{sc,\ast}$)

The joint set of a day-group is the Cartesian product of the $N^{r}$ outage samples and the
renewable scenarios, so $|\mathcal S^{sc}_d|=N^{r}\,|\mathcal R|$ and each joint scenario has
weight $\pi_s=w_{r(s)}/N^{r}$. This construction combines renewable and generator-outage
uncertainty multiplicatively without a bespoke joint sampler. For example, with $N^r=500$
outage samples and 3 renewable scenarios, the unfiltered joint set has 1,500 scenarios per
day-group, each solvable in Stage 2.

Solving every scenario is expensive, and most outage draws are uninformative: they either do
not stress the system or stress it like many other draws. When `risk_filtering_flag == true`,
joint scenarios are therefore **screened** to those that stress the system. For each
scenario the risk score is the maximum over hours of

$$
\text{risk}(s) = \max_{d,h}\ \frac{\sum_{i\in\mathcal I^{GC}_s} \text{FirmCap}_{i,d,h}}
                                   {\text{gen margin}_{d,h}}
$$

This is the outaged firm capacity relative to the margin left after subtracting
deterministic demand from total firm capacity. Maximising over the hours of the day-group
means a sample that stresses the system only briefly, for example in a single peak hour, is
still judged by its worst moment rather than diluted by calmer hours. The gen-margin
denominator is floored at 100 MW whenever any hour's margin is at or below zero, which avoids
division by zero or a negative margin.

Screening is performed at the **joint** (outage sample × renewable scenario) granularity,
because $\text{FirmCap}$ for VRE units depends on the paired renewable scenario: the same
outage draw can be retained under one renewable pairing and dropped under another.

A joint scenario is retained if $\text{risk}(s)>\texttt{risk\_tol\_value}$, with a per-day-group
floor of $\max\!\big(10,\ \texttt{min\_num\_risk\_in\_each\_day\_value}\times |\mathcal S^{sc}_d|\big)$
scenarios kept by rank when the tolerance filter alone would keep fewer. The floor is scaled
by the full joint-scenario count $|\mathcal S^{sc}_d| = N^r\times|\mathcal R|$ of day-group
$d$, not by $N^r$ alone. It ensures that a day-group whose demand and capacity profile makes
every joint scenario score below `risk_tol_value` (for example a very low-stress
representative day) still contributes retained scenarios to the metrics; without it, that
day-group could contribute nothing to annual EUE and LOLH and understate risk. Setting
`risk_filtering_flag` off simulates every joint scenario, which is the most accurate option at
the cost of solving all $N^r\times|\mathcal R|$ scenarios in Stage 2.

The retained scenarios form the set $\mathcal S^{sc,\ast}_d$ that Stage 2 simulates.
Filtered-out scenarios contribute zero to every metric. This treats them as approximately
zero-ENS draws rather than removing them from the probability space, so the retained
scenarios' weights are *not* re-normalised.

---

## Two-stage structure

RA does **not** re-solve a full mixed-integer unit-commitment problem for each of the
(potentially thousands of) generator-outage draws. That would be computationally infeasible,
and conceptually wrong: an outage at hour 14 does not lead the operator to revisit which
units were committed at hour 1. RA instead factors the problem into a one-time
commitment and baseline solve per day-group and renewable scenario (Stage 1), and a much
cheaper, commitment-fixed, continuous LP re-dispatch for each retained outage draw (Stage 2).
The redispatch penalty $\delta^s$ in Stage 2 is therefore meaningful: it measures deviation
from a *specific, already-optimised* reference trajectory.

For each planning stage $y$ RA is solved **day-group by day-group**. Within a day-group
$d$ the workflow is:

1. **Stage 1 — reference (risk-free) UC/ED.** For each *retained* renewable scenario
   $r$, solve a full chronological unit-commitment / economic dispatch over the
   day-group's hours with **no outages**. This fixes the baseline dispatch
   $(g^{\mathrm{ref}},chg^{\mathrm{ref}},soc^{\mathrm{ref}})$ and the commitment schedule.
2. **Stage 2 — risk realization (post-contingency ED).** For each **filtered joint
   scenario** $s\in\mathcal{S}^{sc,\ast}_{d}$ (one outage sample × one renewable scenario),
   apply the sampled hourly outage mask, hold commitment fixed at the Stage 1 schedule, and
   redispatch the surviving fleet to minimise unserved energy plus a redispatch penalty.
3. **Metric aggregation.** Probability-weight the scenario ENS outcomes into EUE / NEUE /
   LOLH / LOLE and the max-extrema.

Stage 2 depends on Stage 1: the deviation penalty and the ramp bounds are defined relative to
$g^{\mathrm{ref}}$, and the "replace-the-lost-MW" cap in
[Stage 2](#stage-2-risk-realization-post-contingency-dispatch) is stated as the reference
output of exactly the units that went on outage. Metric aggregation turns the per-scenario
ENS values into annualised EUE and LOLH through the day-group weights `NumDays` and the
joint-scenario weights $\pi_s$.

Stage 2 runs in one of two dispatch modes, selected by `RA_method`:

- **`Economic Dispatch` (perfect foresight).** A single LP over the whole day-group. Hours
  before the first contingency ("pre-window") are pinned to the reference solution; from
  the first outage hour onward, a **continuous redispatch window** re-optimises with SOC
  chained hour to hour. Because the whole window is solved jointly, storage and
  flexible or hydro units see the *entire* remaining outage duration when shaping their
  output; for example a battery can save energy in hour 1 of a two-hour outage knowing it
  will need it in hour 2. This is the cheaper, more optimistic mode: it effectively assumes
  the operator knows when every unit will return to service.
- **`Sequential Economic Dispatch` (imperfect foresight).** A rolling-horizon sequential
  ED advancing hour by hour: each event hour solves a short look-ahead LP of length
  $k=$ `sequential_horizon_hours_value` (default 6), commits its first hour, and carries the
  committed SOC forward. Later look-ahead hours use a projected (not truly foreseen) outage
  mask. This mode approximates what a real operator can see: a storage unit reacting to an
  outage does not know exactly how long the outage will last, so it can over- or
  under-commit its state of charge relative to the perfect-foresight solution. Sequential
  mode is more representative but computationally heavier (one LP solve per event hour
  instead of one per scenario).

Both modes share the Stage 1 reference solve, the outage and renewable scenarios, and the
metric definitions; only the mechanics of solving the post-contingency window differ.
Sequential-mode ENS is generally equal to or (weakly) larger than perfect-foresight ENS for
the same scenario set because it has strictly less information, so EUE and LOLH from the two
modes are directly comparable only with that caveat.

---

## Stage 1 — reference (risk-free) UC/ED

### Objective

The reference stage answers a simple question in isolation from any contingency: *given
this renewable scenario and no forced outages, what is the least-cost way to serve load
over the day-group, and how much (if any) load is structurally unservable even before any
generator fails?* A system with insufficient total capacity or transmission shows positive
ENS in the risk-free reference dispatch itself. That is a data or build problem rather than a
reliability-risk result, and separating it here keeps it visible instead of folding it into
the outage-driven Stage 2 metrics.

The objective minimises production cost + carbon tax + value of lost load, with
start-up and no-load charges under unit commitment. There are **no reserve terms**:

$$
\begin{aligned}
\min\ \ &\sum_{i}\sum_{d,h,t}\Big(MC_{i,y,d}\, g^{\mathrm{ref}}_{i,d,h,t}
        + CTAX\cdot EF_i\,\tfrac{1}{|\mathcal T|}\, g^{\mathrm{ref}}_{i,d,h,t}\Big)\\[2pt]
  +\ &\sum_{i\in\mathcal I^{UC}}\sum_{d,h}\Big(C^{SU}_i\, su_{i,d,h,1}
        + C^{NL}_i\,\tfrac{1}{|\mathcal T|}\, c_{i,d,h,1}\Big)
  \ +\ \sum_{n}\sum_{d,h,t}\text{VOLL}\,\tfrac{1}{|\mathcal T|}\, ens_{n,d,h,t}
\end{aligned}
$$

$C^{SU}_i,C^{NL}_i$ are scaled by $CAP_i$. A negligible random perturbation breaks dispatch
ties, and new units under ELCC evaluation receive a discounted marginal cost so they are
dispatched first. Large flexible load contributes its demand-response value with a negative
coefficient.

Term by term:

- **Generation cost** $MC_{i,y,d}\,g^{\mathrm{ref}}_{i,d,h,t}$ is the variable (fuel + variable
  O&M) cost of running the fleet and drives merit-order dispatch. It makes the reference
  dispatch the least-cost baseline around which Stage 2 redispatches.
- **Carbon tax** $CTAX\cdot EF_i\,g^{\mathrm{ref}}_{i,d,h,t}/|\mathcal T|$ folds a \$/tonne
  emissions price into marginal cost so that, when `CTAX>0` (`Simulation Configuration`
  sheet), the reference dispatch, and hence which units are on the margin during a
  contingency, reflects the same carbon-cost signal as GTEP and Operation. If `CTAX=0` the
  term vanishes and RA reduces to a pure economic dispatch.
- **Start-up and no-load cost** ($C^{SU}_i\,su_{i,d,h,1}+C^{NL}_i\,c_{i,d,h,1}/|\mathcal T|$,
  active only under `Dispatch_Mode_in_OP == "Unit Commitment"`) captures the fixed cost of
  cycling a thermal unit and of keeping it synchronized at its minimum stable level. Without
  these terms the dispatch could cycle units on and off every hour to save a fraction of a
  dollar, giving unrealistic commitment behavior and a distorted set of units "available to
  fail" in the outage scenarios. The no-load cost is the same parameter GTEP and Operation use.
- **VOLL·ENS** is the reliability signal itself. It prices unserved energy at the value of
  lost load so the solver sheds load only when every other option, including starting an
  expensive peaker, is exhausted. Because $\text{VOLL}$ is far above any realistic
  $MC_i$, nonzero $ens_{n,d,h,t}$ in the *reference* stage indicates insufficient installed
  capacity or transmission for that day-group and scenario; no generator failure explains it.
- **Tie-breaking perturbation** (of order $10^{-7}$, seeded so results are reproducible run to
  run). Many units share an identical marginal cost (for example a block of identical new gas
  units), which would otherwise leave multiple equally cheap optimal dispatches and let the
  solver's choice flip between re-solves for no economic reason. The perturbation is far below
  any real cost difference and never changes which unit is truly cheaper.
- **New-unit marginal-cost discount** (applied to units added while searching for a capacity
  credit) ensures the candidate resource is dispatched ahead of otherwise tied incumbents, so
  its contribution to reliability is not lost to tie-breaking noise.
- **Large flexible load (LFL) demand response** enters with a *negative* coefficient because
  serving load through a higher-priced demand-response segment ($s\ge 2$) is economically
  equivalent to a negative-cost supply source (the load is willing to pay the segment `Price`
  to keep consuming). Higher segments are called on only once cheaper generation is exhausted.
- **Additional $10^{-7}$ tie-breaking penalties** negligible relative to any real cost break
  degenerate ties for quantities the perturbation above does not reach: grid charging of an
  LFL's paired onsite battery, storage-commitment cycling when the unit's `Storage Commitment`
  flag is true, and the charge power of hybrid storage.

Because the reference dispatch holds no headroom, every MW of installed capacity is available
to the least-cost merit order in Stage 1, and any reserve-like behavior appears only once
Stage 2 applies an actual outage.

### System balance and power flow

The bus balance is the Kirchhoff current-law analog: every MW consumed at a bus must be
supplied by local generation, net transmission inflow, storage discharge, unserved energy, or
demand response, in every hour of every retained day. It determines whether the system is
short of energy at all, and it is where the outage mask (Stage 2) or renewable shape (both
stages) takes effect: removing a generator's output, absent enough import or substitute
generation, forces $ens_n$ positive. This is the mechanism by which an outage becomes
"unserved energy."

The bus balance holds generation minus storage charging, plus net imports and unserved
energy, equal to the (T&D-grossed-up, peak-scaled) demand. In `Network_Flow` and
`B-theta` modes:

$$
\sum_{i\in\mathcal I_n} g^{\mathrm{ref}}_{i}
-\sum_{i\in\mathcal I^{ES}_n} chg^{\mathrm{ref}}_{i}
+\!\!\sum_{k:\,t\text{-bus}=n}\!\! f_{k}
-\!\!\sum_{k:\,f\text{-bus}=n}\!\! f_{k}
+ ens_{n} - \sum_{l\in\mathcal L_n} lfl_{l}
= D_{n,d,h,t},
\qquad 0\le ens_{n}\le D_{n,d,h,t}
$$

for all $n,d,h,t$, with the demand gross-up $D_{n,d,h,t}=(1+\rho)\,\Phi\,\widehat D_{n,d,h,t}$.
The ENS upper bound is the bus demand.

The demand in this constraint is not the raw hourly load timeseries. It is grossed up by
$(1+\rho)$ to reflect transmission-and-distribution losses (so generation covers losses as
well as delivered energy) and scaled by $\Phi$ (`system_peak_scale`), which stress-tests the
system at, for example, 105% of its historical peak without re-scaling every input timeseries.
The bound $0\le ens_n\le D_{n,d,h,t}$ prevents negative unserved energy (which would act as
free supply and understate EUE) and caps the shortfall at any bus and hour at that bus's own
demand, which keeps the metric physically interpretable at the regional and bus level used in
the [NEUE](#neue-normalized-eue-ppm) and DLOL rollups. The $lfl_l$ term subtracts the demand
response actually delivered by large flexible loads, so a bus that voluntarily curtails part
of an industrial load (at a price, per the objective's demand-response segments) does not also
register as unserved energy.

**Branch limits (DC modes).** $-F_k\le f_k\le F_k$ for every branch, using the branch
`rate_a` rating, regardless of transmission mode. Without them a `Network_Flow` or `PTDF`
solve could route unlimited power over one corridor and mask transmission-constrained
reliability problems, such as an import-dependent load pocket.

**`B-theta` DC-OPF.** For every non-DC-tie branch, flow follows the angle difference:

$$
f_{k,d,h,t} = b_k\big(\theta_{f(k),d,h,t}-\theta_{t(k),d,h,t}\big),
\qquad b_k = 1/\texttt{br\_x\_pu},\quad \forall\,k\notin\mathcal K^{DC}
$$

This is a full linearised DC optimal power flow: flow on a branch is proportional to the
angle difference across it, weighted by its susceptance $b_k$, so power routes according to
network impedance rather than being freely dispatchable between any two buses. This mode can
expose loop-flow-driven congestion that `Network_Flow` mode cannot. DC ties
(`dc_line == true`) are **excluded from the angle equation** because an asynchronous HVDC link
has no physical AC angle relationship; its flow is a controllable set-point bounded only by
the branch's thermal rating. One reference (slack) bus is fixed **per synchronous island**, as
in GTEP and Operation, so each AC-connected region has its own angle reference and DC ties do
not couple angles across islands.

**`PTDF` mode.** The delivered-demand balance is
$demand_n + ens_n - lfl_n = D_n$, flows are
$f_k=\sum_n \text{PTDF}_{k,n}\,p^{\mathrm{inj}}_n$, and injections sum to zero,
$\sum_n p^{\mathrm{inj}}_n = 0$. PTDF mode is the computationally cheapest of the three: the
linear sensitivity of each branch flow to each bus's net injection is pre-computed once
outside the RA solve, which reduces the power-flow physics to a single linear expression per
branch, at the cost of not modeling angle variables explicitly. The zero-sum constraint is
required because PTDF factors are only meaningful relative to a reference bus.

### Unit commitment, dispatch, and ramping

These constraints keep the reference dispatch physically realistic for thermal and nuclear
units: a coal or CCGT unit cannot run below its minimum stable level while online, cannot
change output arbitrarily fast, and its start-up is tied to its commitment history. Without
them the reference stage could dispatch a baseload unit anywhere between 0 and nameplate in
any hour, which misrepresents plant behavior and gives Stage 2 an unrealistically flexible
baseline (a recovery that "backs off" a coal unit to near zero in one hour is not physically
achievable).

Under `Dispatch_Mode_in_OP == "Unit Commitment"` the reference dispatch uses the standard UC
relations of the Operation model, without reserve headroom:

$$
\underline{P}_i\,CAP_i\, c_{i} \ \le\ g^{\mathrm{ref}}_{i}\ \le\ CAP_i\, c_{i},
\qquad c_{i}\le U^{0}_i,\qquad su_{i,h}\ge c_{i,h}-c_{i,h-1}
\qquad \forall\, i\in\mathcal I^{UC}
$$

Thermal and nuclear units also take inter-temporal hourly ramp limits.

The dispatch band $[\underline{P}_i\,CAP_i\,c_i,\ CAP_i\,c_i]$ states that while committed
($c_i=1$) a unit runs at or above its minimum stable level $\underline{P}_i$ and at or below
nameplate capacity; while off ($c_i=0$) both bounds collapse to zero. Because RA carries no
reserve requirement, the upper bound is $CAP_i$ itself rather than $CAP_i$ less a reserve
obligation, so a committed coal unit can in principle be dispatched to nameplate.
$c_i\le U^0_i$ ties commitment to the unit's exogenous availability for that day (a unit under
planned maintenance has $U^0_i=0$ and cannot be committed), and $su_{i,h}\ge c_{i,h}-c_{i,h-1}$
records a start-up event whenever commitment turns on, which the objective's start-up cost
bills against.

Without commitment, a dispatchable unit is capped at its installed capacity and ramp-limited:

$$
g^{\mathrm{ref}}_{i} \le U^{0}_i\, CAP_i
\qquad \forall\, i\notin\mathcal I^{UC}
$$

This branch covers units the workbook does not model with discrete commitment, such as
peakers, technologies run under `Dispatch_Mode_in_OP` set to `"Economic Dispatch"` rather than
`"Unit Commitment"`, or other non-UC unit categories. They are capacity- and ramp-bounded with
no minimum-stable-level floor and no start-up or no-load cost. Both the UC and non-UC
inter-temporal ramp constraints link hour $h$ to $h-1$ within a day and chain across the
day-group's calendar-day boundary, so a unit cannot jump output at midnight. At the
day-group's first hour there is no previous hour to ramp from, so the constraint is anchored
differently there (see [Storage](#storage), which follows the same pattern).

### Renewables

Fixed-profile VRE (and run-of-river or impoundment hydro) must meet its hourly shape, with
curtailment:

$$
g^{\mathrm{ref}}_{i} + curt_{i} = S^{r}_{i,d,h}\, U^{0}_i\, CAP_i\,\overline{P}_i
\qquad \forall\, i\in\mathcal I^{VRE,\mathrm{fix}}
$$

This is an **equality**, not an upper bound, because a fixed-profile resource (wind, solar
PV, run-of-river hydro) has no fuel decision: its as-available output at hour $(d,h)$ is
exogenous, set by the shape $S^r_{i,d,h}$ of the active renewable scenario $r$. The
curtailment variable $curt_i$ keeps the constraint feasible when the system does not need, or
the network cannot absorb, all of that output. For example, a mid-day PV shape of 0.9 at a
plant with weak local transmission may be partly curtailed. Without $curt_i$, an oversized VRE
fleet in a low-load hour would make the reference dispatch **infeasible**.

In RA the output-limit parameter $\overline{P}_i$ is set to 1, so output is read directly
from nameplate $CAP_i$ times the shape. Elsewhere in A-LEAF `PMAX` acts as an additional
per-unit derate on top of the shape (for example for inverter-loading-ratio effects in hybrid
PV+storage), and RA disables it in both stages so capacity-credit calculations are not
affected by a parameter defined for the Operation model.

Budget-limited hydro instead caps day-group energy by its water budget rather than an hourly
shape. It represents a reservoir that can be dispatched flexibly hour to hour (subject to
`Hydro_Flexibility_Flag` and `Hydro_Flexibility_Percent`, see
[Stage 2](#redispatch-bounds-and-the-contingency-deployment-cap)) provided its *total* energy
across the day-group does not exceed the available water. In the sequential
(imperfect-foresight) mode the budget is not re-imposed within each short look-ahead
window, since a several-hour window has a negligible effect on the day-group's total reservoir
budget; the full-window solve of the perfect-foresight mode enforces it once per day-group.

### Storage

State of charge chains across the day-group with one-way efficiency $\eta_i=\sqrt{\texttt{BATEFF}}$,
within energy bounds:

$$
soc^{\mathrm{ref}}_{i,d,h,t} = soc^{\mathrm{ref}}_{i,d,h-1,t}
  + \tfrac{1}{|\mathcal T|}\Big(\eta_i\, chg^{\mathrm{ref}}_{i} - \tfrac{1}{\eta_i}\, g^{\mathrm{ref}}_{i}\Big),
\qquad \underline{E}_i \le soc^{\mathrm{ref}}_{i}\le \overline{E}_i
$$

This is the energy accounting for a battery or other storage device. The round trip is split
into a charging efficiency $\eta_i$ (less than 1 MWh lands in the battery per MWh drawn from
the grid) and a discharging efficiency $1/\eta_i$ (more than 1 MWh must leave storage to
deliver 1 MWh), with $\eta_i=\sqrt{\texttt{BATEFF}}$ splitting the round-trip efficiency
symmetrically between the two legs. Without it, storage would act as a free lossless energy
source, overstating its contribution during a contingency and understating reliability risk.
The bounds $[\underline{E}_i,\overline{E}_i]$ enforce the physical energy capacity of the
device (a 4-hour, 100 MW battery has $\overline{E}_i=400$ MWh) and a minimum floor (`STOMIN`)
below which the unit cannot be discharged.

For all $i\in\mathcal I^{ES}$, day-group SOC-neutrality is enforced at the closing day: the
storage device must end the representative day-group at the level at which it started. Without
neutrality, a battery that ends every summer-peak day-group fuller than it started would
implicitly import energy from outside the simulated system, since each representative day
recurs `NumDays` times per year.

The day-group's first hour has no previous hour within the group, so its SOC balance is
anchored by the storage initialization option (`Simulation Configuration` sheet:
`Minimum`/`Middle`/`Maximum`, meaning start at `STOMIN`, half of `ES_MWh`, or full). This
choice can matter for a short, high-stress day-group in which the battery never fills through
normal cycling before the first event hour. A storage unit added as a **SATA**
(storage-added-as-transmission) asset instead reserves a `dual_use_percent` share of its
energy capacity for transmission support and starts each day-group at
$\overline{E}_i\,(1-\text{dual\_use\_percent})$, since only part of its capacity is available for
energy arbitrage and reliability service.

Charge power and storage commitment are limited by the device's full nameplate rating, with no
reserve set-aside.

!!! info "Hybrid storage"
    The constraints above apply to **standalone** storage units. For the co-located battery of
    a **hybrid** unit (for example PV+storage), the model uses an equivalent set of storage
    constraints (charge and discharge limits, SOC balance, and SOC neutrality) that also
    limit how much of the battery's charging can come from the grid rather than from its
    co-located generator.

### Large flexible load (LFL) demand response

Beyond the demand-response value in the objective and the $lfl_l$ term in the bus balance, the
reference dispatch includes the **full** LFL constraint family in every hour of every
day-group, identical to that documented on the
[Operation formulation page](../Operation/Operation_Formulation.md): the LFL's own power
balance and interconnection injection and withdrawal limits, its demand-response segment
bounds and relations and the daily demand-response balance and limit, and, when the LFL has an
onsite hybrid generator or storage asset (`Hybrid_Gen` and `Hybrid_ES` columns on the
`Demand` sheet), the onsite thermal-capacity cap and the onsite storage charge, discharge,
SOC-cap, SOC-balance, and neutrality constraints. Nothing here is RA-specific.

### Hybrid on-site generation (GEN side)

Separately from the hybrid storage coupling above and in
[Scenario generation](#scenario-generation), the co-located **generator** side of a hybrid
asset (for example the PV portion of a PV+storage hybrid) has its own thermal-cap and
interconnection constraints in every hour of the reference dispatch: an output cap for the
onsite generator, and limits on how much the shared point of interconnection can inject into
or withdraw from the grid, net of the generator and any co-located storage.

### Policy constraints (storage AET limit, clean-energy generation target)

When enabled in `Simulation Configuration`, the reference dispatch also enforces two
policy constraints defined on the [GTEP formulation page](../GTEP/GTEP_Formulation.md):

- The storage **annual energy-throughput limit**, when
  `Energy_Storage_AET_Limit_Flag == true`.
- The **Clean Energy Generation Target**, when `Clean_Energy_Generation_Target_OP_Flag ==
  true` and the current planning-stage year is at or past
  `Clean_Energy_Generation_Target_Start_Year`.

Neither constraint is re-imposed in Stage 2's post-contingency redispatch (see
[Applying the contingency](#applying-the-contingency)). Both are reference-stage policy
checks on the risk-free baseline.

---

## Stage 2 — risk realization (post-contingency dispatch)

For each retained joint scenario $s\in\mathcal{S}^{sc,\ast}_{d}$, the outage mask
$a^{s}_{i,d,h,t}$ and the reference solution are fixed inputs, and commitment is inherited
from Stage 1. The LP re-optimises dispatch to serve as much load as possible while staying
close to the reference schedule. This block generates RA's reliability signal: it asks, with
this specific set of units unavailable this hour, whether the surviving fleet can still cover
load, and if not, how much load is shed and for how long. Each retained scenario
(potentially hundreds to thousands per day-group) triggers one solve of this LP, and keeping it
a continuous LP with fixed commitment, rather than a fresh MILP, is what makes RA tractable at
scenario-set scale.

### Objective

$$
\min\ \ \sum_{i}\sum_{(d,h,t)\in\mathcal W_s} MC_{i,y,d}\,\delta^{s}_{i,d,h,t}
       \ +\ \sum_{n}\sum_{d,h,t}\text{VOLL}\, ens^{s}_{n,d,h,t}
$$

where $\mathcal W_s$ is the continuous redispatch window (first outage hour to end of
day-group) and $MC_{i,y,d}$ is floored at $\text{MC}^{\min}$. The sequential mode uses the
same two terms in each look-ahead solve.

The two terms play very different roles. **VOLL·ENS** is by orders of magnitude the dominant
term: the LP exhausts every unit's redispatch headroom before accepting any unserved energy,
which is what makes $ens^s$ a meaningful reliability signal. **The deviation penalty**
$MC_{i,y,d}\,\delta^s_i$ is a secondary regulariser. Among the (typically many) redispatch
solutions that achieve the same minimum ENS, it selects the economically cheapest, and it
discourages swinging a free or near-free unit (for example must-take wind, or hydro with
near-zero marginal cost) up and down within the deviation window when that serves no
reliability purpose. Flooring the redispatch marginal cost at $\text{MC}^{\min}$
(`min_redispatch_mc_value`, default $10^{-4}$ pu $\approx$ \$1/MWh) matters for zero and
near-zero-cost units: without a floor the LP is indifferent among infinitely many redispatch
patterns for them, which makes solver behavior noisy and non-reproducible across otherwise
identical scenarios.

!!! info "The redispatch penalty is a tie-breaking regulariser, not a reliability cost"
    The reliability objective is dominated by $\text{VOLL}\cdot ens^{s}$; the
    $MC_i\,\delta^{s}_i$ term never competes with VOLL over whether load is shed and only
    shapes *which* dispatch achieves the minimum-ENS outcome. New units under ELCC evaluation
    receive a $0.9\times$ marginal-cost discount, so a candidate resource is used ahead of
    otherwise tied incumbents. A battery added as transmission (SATA) receives a $-1$
    coefficient, a reward rather than a cost for deviating, so SATA storage is dispatched to
    relieve congestion before other resources are redispatched.

### Applying the contingency

At every hour where unit $i$ is outaged ($a^{s}_{i,d,h,t}=0$, i.e. $i\in\mathcal I^{GC}_{s}(d,h,t)$),
its generation, deviation, and charging are fixed to zero:

$$
g^{s}_{i,d,h,t}=0,\quad \delta^{s}_{i,d,h,t}=0,\quad chg^{s}_{i,d,h,t}=0
\qquad \forall\, i\in\mathcal I^{GC}_{s}(d,h,t)
$$

This represents "unit $i$ is on forced outage this hour." The variables are removed from the
LP's degrees of freedom entirely rather than merely bounded, so the solver cannot "un-outage"
a unit by finding it cheaper to keep dispatching. Fixing $\delta^s_i=0$ alongside $g^s_i=0$ is
necessary because the deviation penalty is defined for every unit in the redispatch window;
otherwise the LP would have to explain a deviation from the reference for a unit that cannot
produce, which would be infeasible or misprice the objective. Storage units on outage also have
their charging fixed to zero, while their SOC is intentionally left free: with $g^s=chg^s=0$
and the hybrid onsite-charge injection also gated to zero, the state of charge holds constant
through the outage without needing to be re-pinned.

Pre-window hours (before the first contingency) fix $g^{s}=g^{\mathrm{ref}}$,
$chg^{s}=chg^{\mathrm{ref}}$, $soc^{s}=soc^{\mathrm{ref}}$. These hours have no contingency
yet, so the scenario replays the reference trajectory, which saves solve effort and guarantees
the redispatch penalty is zero before the event begins.

!!! info "Reduced LFL and hybrid constraints in Stage 2"
    Of the full [LFL constraint family](#large-flexible-load-lfl-demand-response) of Stage 1,
    only a single simplified bound on the LFL dispatch is re-imposed in each retained scenario
    (in both dispatch modes). The demand-response segment mechanics, the daily demand-response
    limit, and the onsite-generator and onsite-storage LFL constraints are not re-built for
    Stage 2, so a post-contingency LFL's flexibility is governed by this one bound rather than
    its full Stage 1 price-segment structure. Of the
    [hybrid GEN-side constraints](#hybrid-on-site-generation-gen-side), only the
    interconnection injection and withdrawal limits are re-imposed; the onsite thermal-capacity
    cap is not, so a hybrid generator's post-contingency output is bounded by the general VRE
    and thermal bounds of [Redispatch
    bounds](#redispatch-bounds-and-the-contingency-deployment-cap).

!!! warning "No line outages are simulated"
    The outage sampler produces only a per-generator up/down mask. There is no line-outage
    draw and no $f^{s}_{k}=0$ contingency, so $\mathcal K^{LC}_s=\varnothing$ in every run, and
    a transmission contingency (line trip, N-1 corridor loss) is not part of RA's reliability
    estimate. The branch flow limits and power-flow equations of Stage 1 are re-imposed
    unchanged in every scenario, so the *rating* of every line is respected; only the
    possibility of a line failing is out of scope. If a case's reliability risk is dominated by
    transmission (for example a single-corridor import-dependent pocket), RA's EUE and LOLH
    understate that risk relative to a study that also samples line outages.

### Redispatch bounds and the contingency-deployment cap

Even a surviving (non-outaged) unit cannot instantaneously take over a lost unit's output: a
coal plant ramping from 60% to 100% of capacity takes time, and a VRE resource can redispatch
only up to what the sun or wind provides that hour. These constraints keep the *response* to
the contingency physically realistic. Without them, one large coal unit tripping could be
"fixed" by an instantaneous, unbounded step change in another unit's output, understating how
disruptive a real contingency is.

Each surviving unit ($a^{s}_i=1$) is bounded by its (VRE-shape-adjusted) capacity and
ramp-limited relative to the reference at the first event hour, or to the previous
post-contingency hour thereafter:

$$
g^{s}_{i} \le CAP_i\, U^{0}_i\, S^{r}_{i,d,h},
\qquad
\big|\,g^{s}_{i} - g^{\mathrm{ref}}_{i}\,\big| \le R^{30}_i\,CAP_i\,U^{0}_i
\ \ (\text{first event hour}),\quad
\big|\,g^{s}_{i,h} - g^{s}_{i,h-1}\,\big| \le R^{60}_i\,CAP_i\,U^{0}_i
$$

The capacity bound $g^s_i\le CAP_i\,U^0_i\,S^r_{i,d,h}(1+\text{flexibility\%})$ re-applies the
unit's VRE shape (for wind, PV, and hydro), so a surviving VRE unit cannot be asked to
redispatch beyond what its weather-driven availability supports that hour. When
`Hydro_Flexibility_Flag` is set, `Hydro_Flexibility_Percent` relaxes the cap, letting
flexible hydro briefly exceed its nominal shape-based output (drawing down the reservoir
faster) to help cover a contingency. The ramp bound distinguishes the **first event hour**,
where the elapsed time since the reference dispatch may be less than a full hour (a mid-hour
contingency) so that a 30-minute ramp fraction $R^{30}_i$ applies, from **subsequent
post-contingency hours**, which use the full 60-minute fraction $R^{60}_i$ because a whole hour
has elapsed between commits. In the sequential (imperfect-foresight) mode, this distinction is
anchored to the *actually committed* prior dispatch of the rolling horizon rather than to the
Stage 1 reference at the same hour, since by a later event hour the system has already
deviated from the reference trajectory.

The absolute-deviation variable linearises $|g^{s}-g^{\mathrm{ref}}|$:

$$
\delta^{s}_{i} \ge g^{s}_{i} - g^{\mathrm{ref}}_{i},
\qquad
\delta^{s}_{i} \ge -\big(g^{s}_{i} - g^{\mathrm{ref}}_{i}\big)
\qquad \forall\, i,\ (d,h,t)\in\mathcal W_s
$$

This is the standard big-M-free formulation for penalising $|x|$ in an LP: two one-sided
inequalities each lower-bound $\delta^s_i$ by $\pm(g^s_i-g^{\mathrm{ref}}_i)$, and because
$\delta^s_i$ carries a strictly positive cost in the objective (after the marginal-cost
floor), the solver pushes it down to exactly $|g^s_i-g^{\mathrm{ref}}_i|$, so no separate
equality is needed.

A **system-wide deployment cap** limits the total upward redispatch of surviving units to
the net output lost to the contingency (net = discharge minus charge):

$$
\sum_{i:\,a^{s}_i=1}\!\big(g^{s}_{i}-chg^{s}_{i}\big)
-\sum_{i:\,a^{s}_i=1}\!\big(g^{\mathrm{ref}}_{i}-chg^{\mathrm{ref}}_{i}\big)
\ \le\
\sum_{i\in\mathcal I^{GC}_{s}}\!\big(g^{\mathrm{ref}}_{i}-chg^{\mathrm{ref}}_{i}\big)
\qquad \forall\, d,h,t
$$

The cap binds in both dispatch modes: the whole-day-group LP of perfect foresight, the
look-ahead LPs of sequential mode, and the short committed-hour LP that sequential mode solves
at each event hour, which also enforces each unit's own capacity and ramp bounds. The
committed dispatch at each event hour is therefore subject to both the system-wide cap and
each unit's limits, consistent with the other solves.

The cap keeps redispatch consistent with *why* ENS occurs: the surviving fleet's total net
output increase over the reference cannot exceed the net output the outaged units *would
have* produced in the reference dispatch. If two 200 MW gas units are outaged and together
produced 250 MW net in the reference dispatch at some hour, the rest of the fleet may
collectively ramp up by at most 250 MW. It cannot manufacture additional MW merely because it
is now cheaper on the margin. Without the cap the LP could over-redispatch surviving cheap
units far beyond what the contingency motivates, effectively re-optimising the whole system
against a different supply and demand balance than the contingency created, which would
distort the redispatch penalty and could mask true unserved energy through an implausible
reallocation. As a small example, if a day-group's reference dispatch has Unit A (100 MW gas,
MC \$30) and Unit B (150 MW gas, MC \$40) running, and a scenario outages Unit A, the surviving
fleet (including B and everything else) may increase its *combined* net output by at most
100 MW that hour.

### Renewable-scenario adjustment

The renewable scenario $r(s)$ enters through the shape $S^{r}_{i,d,h}$, which (i) drives the
Stage 1 reference dispatch for that scenario and (ii) caps post-contingency VRE output in the
capacity bound above. Each retained joint scenario is therefore solved against **its own** VRE
availability: pairing outage sample $k$ with a "low wind" renewable scenario gives a
*different* reference baseline and a *different* Stage 2 VRE cap than pairing the same sample
with `BASE`. No separate constraint balances a VRE-shape deviation against reserve
deployment. The effect of a renewable scenario differing from the base case is carried
entirely by (a) the Stage 1 reference dispatch being re-optimised against that scenario's
shape and (b) the same shape re-appearing as the Stage 2 capacity cap. A low-wind scenario
appears as a *lower ceiling* on wind output in both stages, and any consequent unserved energy
is attributed to renewable variability in the same way a generator outage is attributed to
$\mathcal I^{GC}_s$.

### Post-contingency storage

Whether a battery may *charge* during a contingency response is a policy and data question
rather than a physical one: an operator might reserve all storage energy for discharge during
an event, or let it optimise freely, including charging from surplus VRE, if that is cheaper.
RA exposes this choice through the `Post_Contingency_Charge_method` setting, because the
appropriate answer depends on the operating philosophy being modeled and can materially change
how much a storage fleet contributes to reliability. Storage charging in the window follows
`Post_Contingency_Charge_method`:

$$
chg^{s}_{i}=
\begin{cases}
0 & \texttt{"Not Allowed"}\\
chg^{\mathrm{ref}}_{i} & \texttt{"Same as Reference"}\\
\le CAP_i\,U^{0}_i & \texttt{"Optimal"}
\end{cases}
\qquad \forall\, i\in\mathcal I^{ES},\ a^{s}_i=1
$$

The three modes:

- `"Not Allowed"` forces $chg^s_i=0$. It is the most conservative assumption, useful to credit
  storage purely as an emergency discharge resource that cannot "reset" itself mid-event.
- `"Same as Reference"` pins charging to what the risk-free reference dispatch was doing, so a
  battery charging from cheap surplus generation before the contingency keeps doing so unless
  that generation itself was lost.
- `"Optimal"` (the default, also used if the setting is misconfigured) lets the LP choose
  charging up to the device's full rated power, maximizing the battery's contribution to
  minimizing ENS.

!!! info "The discharge side has a symmetric switch"
    The `RA Setting` sheet's `Post_Contingency_Discharge_method` column mirrors the charge-side
    logic:
    $$
    g^{s}_{i}=
    \begin{cases}
    0 & \texttt{"Not Allowed"}\\
    g^{\mathrm{ref}}_{i} & \texttt{"Same as Reference"}\\
    \text{(unbounded here)} & \texttt{"Optimal"}
    \end{cases}
    \qquad \forall\, i\in\mathcal I^{ES},\ a^{s}_i=1
    $$
    `"Not Allowed"` forces $g^s_i=0$ (no discharge during the event), `"Same as Reference"` pins
    discharge to the risk-free reference trajectory, and `"Optimal"` (the default) adds
    no extra bound: the LP may discharge a surviving battery up to the general redispatch
    bounds above (capacity, ramp, SOC floor `STOMIN`).

State of charge chains through the window, with the reference hybrid onsite-charge injection
gated by availability $a^{s}_i$, within the same $[\underline{E}_i,\overline{E}_i]$ bounds:

$$
soc^{s}_{i,d,h,t} = soc^{s}_{i,d,h-1,t}
  + \tfrac{1}{|\mathcal T|}\Big(\eta_i\big(chg^{s}_{i} + a^{s}_i\, g^{G\to ES,\mathrm{ref}}_{i}\big)
      - \tfrac{1}{\eta_i}\, g^{s}_{i}\Big)
$$

Discharge and charge power and storage commitment use the same nameplate-rating limits as
Stage 1.

The availability gate $a^s_i$ on $g^{G\to ES,\mathrm{ref}}_i$ (the hybrid's onsite
renewable-to-battery charging path) matters for **hybrid** assets. A hybrid PV+storage unit's
battery shares the outage state of its co-located generator (see
[Scenario generation](#scenario-generation)), so when the hybrid is on outage the onsite PV
can no longer charge its paired battery. Gating this injection to zero, rather than continuing
to feed it from the reference solution, keeps the battery's SOC from growing during an outage
in which it cannot receive energy. The window's first hour reads its prior SOC from the
Stage 1 reference trajectory (there is no post-contingency history yet); every later hour
chains to the *previous post-contingency* SOC, so a multi-hour outage compounds any
post-contingency charging and discharging decisions rather than repeatedly resetting to the
reference.

---

## Reliability metrics

All metrics are computed from the retained-scenario ENS solutions. Every scenario solved in
Stage 2 produces an $ens^s$ solution for every bus and hour; this section collapses those
scenario-level results into the annual numbers (EUE, NEUE, LOLH, LOLE, and the max-extrema)
that are reported and that feed the ELCC and DLOL capacity-credit calculations. The weighting
makes these numbers probability-weighted expectations rather than arbitrary sums across
scenarios.

Let $\pi_s=w_{r(s)}/N^{r}$ be the joint-scenario weight,
$\sigma_d$ the representative-day count (`NumDays`), and $B$ the per-unit power
base (`per_unit_base_value`, converting per-unit ENS to MW). An **event hour** is any
$(d,h,t)$ with positive unserved energy, thresholded in practice at $B\cdot ens^s>10^{-4}$
MW so that numerical LP noise is not counted as an event.

### EUE — Expected Unserved Energy (MWh/yr)

$$
\mathrm{EUE} = \sum_{s\in\mathcal S^{sc,\ast}}\ \pi_s
      \sum_{n}\sum_{d,h,t}\ \sigma_d\, B\, ens^{s}_{n,d,h,t}
$$

EUE is accumulated systemwide and broken down by region and day-group. Only hours above
the $10^{-4}$ MW noise floor contribute. An LP solution can carry tiny nonzero ENS from solver
tolerance alone even when the true answer is zero load shed, and without the floor a large
scenario set could accumulate a spurious nonzero EUE from numerical noise.

EUE is a *probability-weighted expectation*, not a worst case: a scenario that sheds 500 MWh
with joint weight $\pi_s=0.001$ contributes only $0.5$ MWh to the annual EUE (times its
day-group's $\sigma_d$), reflecting that the outage combination it represents is rare. It
answers "how much energy, on average per year, do we expect to fail to serve" rather than
"how bad can it get," which the
[max-extrema](#max-extrema-scenario-worst-cases-not-expectations) below address.

### NEUE — Normalized EUE (ppm)

$$
\mathrm{NEUE} = \frac{\mathrm{EUE}}{\sum_{n}\sum_{d,h,t}\sigma_d\, B\, D_{n,d,h,t}}\times 10^{6}
$$

NEUE is computed at systemwide, regional, and day-group resolution. The numerator is
probability-weighted and the denominator is deterministic annual demand: weights are applied
during accumulation, not through an outage-only divisor. A raw EUE is not comparable across
systems of different size (100 MWh/yr of unserved energy is a much bigger problem for a 500 MW
island than for a 50 GW interconnection). Dividing by total annual demand and scaling to parts
per million puts every system on one normalized reliability scale, which is why NEUE is the
metric most often used as a target in reliability standards.

### LOLH / LOLE

$$
\mathrm{LOLH} = \sum_{s\in\mathcal S^{sc,\ast}}\ \pi_s
      \!\!\sum_{(d,h,t)\,:\,\text{event}}\!\! \sigma_d,
\qquad
\mathrm{LOLE} = \frac{\mathrm{LOLH}}{24}\ \ (\text{days/yr})
$$

A system event hour is counted **once across all buses** per scenario, while the regional LOLH
counts per-bus event hours. This de-duplication matters: if three buses show unserved energy in
the same hour of the same scenario (a single systemwide shortfall spread across regions), the
systemwide LOLH counts **one** hour of system-level loss of load, not three. Otherwise a
widespread single-cause event would inflate systemwide LOLH by the number of regions it
touched. Regional LOLH does not de-duplicate across buses, since each bus's own loss-of-load
experience is what a regional reliability standard addresses.
$\mathrm{LOLE}=\mathrm{LOLH}/24$ restates expected loss-of-load hours per year as expected
loss-of-load *days* per year on the convention that each loss-of-load hour counts as 1/24 of a
day; it is a unit conversion, not an independent calculation.

### Max-extrema (scenario worst-cases, not expectations)

`Max_Consecutive_Outage_Hours`, `Max_MWh_Loss`, and `Max_MW_Loss` track the largest values
observed over the solved scenarios (using unweighted per-unit ENS $B\cdot ens^{s}$) and record
the day-group where each occurred. They answer a different question from
EUE, NEUE, LOLH, and LOLE: instead of "what do we expect on average," they report how bad the
single worst simulated scenario was. `Max_Consecutive_Outage_Hours` is the longest unbroken run
of hours with unserved energy at any bus, `Max_MWh_Loss` is the largest cumulative MWh lost
in one such uninterrupted streak, and `Max_MW_Loss` is the largest instantaneous MW shortfall
in any hour. Because these are **maxima, not probability-weighted sums**, a scenario with a
tiny joint weight $\pi_s$ can still set the systemwide maximum if it is the worst one drawn.
They are a tail-risk diagnostic alongside the expectation-based metrics, not expected
outcomes.

!!! info "Scenario weights sum to one per day-group"
    Over the *unfiltered* joint set, $\sum_{k=1}^{N^r}\sum_{r\in\mathcal R} w_r/N^{r} = \sum_{r\in\mathcal R} w_r = 1$
    (renewable weights are normalised), so EUE and LOLH are proper expectations. Risk
    filtering (see [Scenario generation](#scenario-generation)) drops low-risk outage samples
    assumed to carry zero ENS. Their omission is a zero-contribution approximation of the
    expectation rather than a re-normalisation of the remaining weights: the filtered
    scenarios' weights are dropped from the sum, not redistributed to the survivors.

---

## Capacity credit — ELCC and DLOL

Both answer the same question, "how much of this resource's nameplate capacity can the system
count on for reliability purposes?", by two structurally different methods. They are
alternative capacity-accreditation approaches selected through `capacity_credit_type`, not
complementary diagnostics (a third value, `"Lookup Table"`, is a non-simulation shortcut
described below). ELCC re-solves RA repeatedly under a search, while DLOL reads capacity
credit directly from the dispatch of a single RA solve. Both report a fraction of installed
capacity (1.0 counts for 100% of nameplate, 0.0 for nothing). Interpretation and rollups are
discussed on the [RA Metrics, ELCC & DLOL](./RA_Metrics_ELCC_and_DLOL.md) page; the
mathematical skeleton is below.

!!! note "A third, non-simulation mode: `capacity_credit_type == \"Lookup Table\"`"
    When `capacity_credit_type` is `"Lookup Table"`, RA skips ELCC and DLOL (and any RA
    dispatch simulation) and looks up each eligible resource's capacity credit from
    precomputed capacity-credit-versus-ICAP (Installed Capacity, MW) curves in the RA data
    workbook (sheets `CACRED_lookup_wind_battery` for wind and 4-hour storage, and
    `CACRED_lookup_pv_storage` for PV keyed jointly on PV and paired-storage ICAP). Each
    resource's *total installed capacity* is rounded to the nearest 1000 MW and used as the
    lookup key, clamped to the largest ICAP row in the table once installed capacity exceeds
    it. The mode is selected through the `capacity_credit_RA_simulation_method` setting.
    Because it assigns capacity credit purely from installed-capacity totals, with no
    reference to any EUE, NEUE, LOLH, or LOLE outcome or dispatch result, it is not a
    reliability calculation in the sense of ELCC and DLOL. It is a fast proxy for cases where
    re-running RA at every capacity increment is too expensive, for example inside a GTEP
    multi-round loop.

### ELCC — Effective Load Carrying Capability

The intuition behind ELCC: a "perfect", always-available resource of $X$ MW lets the system
serve $X$ MW of additional load with no change in reliability. A real resource, subject to
outages and weather variability, typically absorbs *less* additional load than its nameplate
before reliability degrades back to its starting level, and the ratio of the extra load it
carries to its nameplate capacity is its ELCC. ELCC holds one reference metric $M$ (EUE,
NEUE, LOLH, or LOLE, at systemwide or regional resolution) fixed at its unperturbed value
$M^{\mathrm{ref}}$ and finds the largest constant load $\lambda$ the added resource lets the
system carry. The added load enters the RA load balance, and the search drives the gap
$g(\lambda)=M(\lambda)-M^{\mathrm{ref}}$ to zero:

$$
\text{find } \lambda^{\star}:\ M(\lambda^{\star})=M^{\mathrm{ref}},
\qquad
\mathrm{ELCC}=\frac{\lambda^{\star}}{\mathrm{ICAP}}
$$

$\lambda$ enters the load balance as extra constant demand added in every hour of every
scenario, at a single bus or distributed across the system in proportion to load. A candidate
resource is therefore evaluated against the *same* stress hours that produce the reference
metric, which makes the ratio a capacity-value measure rather than an average-output measure
(see the contrast with [DLOL](#dlol-dispatch-based-capacity-credit-proxy) below).

The root is found by a **bracketed search**: geometric doubling to bracket, then Illinois
regula-falsi with bisection fallback, converging when $|g|\le\texttt{abs\_tol}$ **or**
$|g|/|M^{\mathrm{ref}}|\le\texttt{tol}$. The procedure is:

1. Add the resource with **zero** extra load and check whether the metric improved. If it
   did *not* improve at all, for example because the resource's availability profile never
   coincides with the system's stress hours, ELCC is reported as `0.0` with status
   `"no_improvement_after_addition"`, since no positive $\lambda$ could restore the metric.
2. Otherwise start at $\lambda=\mathrm{ICAP}$ (as if the resource were 100% credited) and
   **double** $\lambda$ until the metric gap changes sign, which brackets the root. A resource
   with a large ELCC may require several doublings before the added load pushes the metric back
   up to $M^{\mathrm{ref}}$; bisection from a fixed bracket would need a very large upper bound
   guessed up front or would fail to bracket.
3. Once bracketed, an Illinois-protected regula-falsi (linear interpolation between the bracket
   endpoints, with an anti-stalling adjustment that halves the weight of a repeatedly unmoved
   endpoint) converges faster than plain bisection while remaining guaranteed to converge.
   Each trial point costs a **full RA re-solve** across all retained scenarios, so the search
   minimizes the number of trials.

As a worked example, if a system's reference EUE is 100 MWh/yr, and adding 200 MW of a new
resource with zero extra load drops EUE to 40 MWh/yr, the search adds constant load (starting
at 200 MW and doubling toward, say, 260 MW) until EUE is pushed back up to 100 MWh/yr. If that
occurs at $\lambda^\star=180$ MW, ELCC $=180/200=90\%$. The same procedure is available for
existing assets, which are deactivated and evaluated in the same way.

### DLOL — dispatch-based capacity-credit proxy

DLOL stands for **Direct Loss of Load**. Despite the name it is not a duration or day-count
metric (that role belongs to LOLH and LOLE); it is a capacity-credit *fraction*, like ELCC.
DLOL is a much cheaper shortcut: instead of re-solving RA under a search over added load, it
reads capacity credit from the dispatch of a **single** RA solve, asking "during the hours when
the system was short of energy, how much did this unit average output, as a fraction of its
nameplate capacity?" A wind farm producing near its full rating during every stress hour gets
a DLOL near 1.0; one that is becalmed during most stress hours (for example wind in a
summer-peaking system where wind is a winter resource) gets a DLOL near 0. For each eligible
unit $i$ (its `UNITGROUP` has `ELCC_Flag == true` in the `Gen Technology` sheet), DLOL is its
representative-day-weighted average output over system stress hours (positive systemwide ENS),
normalised by installed capacity:

$$
\mathrm{DLOL}_i = \frac{1}{\mathrm{ICAP}_i}\cdot
\frac{\sum_{\text{stress hours}} \sigma_d\, B\, g^{s}_{i}}
     {\sum_{\text{stress hours}} \sigma_d}
$$

Unlike EUE and LOLH, the joint-scenario weight $\pi_s$ cancels from the ratio and is not
applied: every scenario and hour with positive systemwide ENS is weighted only by its
representative-day count $\sigma_d$, regardless of how likely that outage draw was. DLOL
answers "conditional on the system being in distress, what does this unit typically
contribute" rather than "what is this unit's unconditional expected contribution."

The trade-off relative to ELCC: DLOL is far cheaper (one RA solve instead of many) and gives a
per-unit result for every eligible unit simultaneously, but it is an
*average-output-during-stress* proxy rather than a rigorous measure of how much load the
resource lets the system carry at unchanged reliability. It does not capture diminishing
returns from adding many correlated units of the same resource: each additional wind unit at
the same site sees the same stress-hour output, so DLOL does not decline with penetration the
way a marginal ELCC would. Results are rolled up to bus and system level using both a simple
average and an ICAP-weighted average across units sharing a `UNIT_GROUP`, so a bus's reported
DLOL for "wind_ons" reflects the capacity-weighted average across every onshore wind unit at
that bus.

---

## Additional notation (RA-only)

RA reuses the [GTEP nomenclature](../GTEP/GTEP_Formulation.md#notation-conventions):
generators $\mathcal{I}$ (storage $\mathcal{I}^{ES}$, VRE $\mathcal{I}^{VRE}$,
unit-commitment $\mathcal{I}^{UC}$), days-in-a-group and hours/sub-hours
$(d,h,t)$, planning stage $y$, branches $\mathcal{K}$, buses $\mathcal{N}$; parameters
$CAP_i$, $\overline{P}_i/\underline{P}_i$ (`PMAX`/`PMIN`), $MC_{i,y,d}$ (`Annual_MC`),
$EF_i$, $F_k$ (`rate_a`), $\eta_i=\sqrt{\texttt{BATEFF}}$, the representative-day weight
$\sigma_d$ (`NumDays`), and $\text{VOLL}$, plus the transmission-and-distribution
(T&D) loss fraction $\rho$, defined on the
[Operation formulation page](../Operation/Operation_Formulation.md#additional-notation-operational-only).
The tables below add only the RA-specific symbols.

!!! info "Time indexing and the planning stage in RA"
    RA solves one planning stage $y$ at a time. $\mathcal{D}$ is the set of calendar days that
    make up a **representative day-group**, over which storage state of charge chains
    chronologically. As in the other model families, $h\in\mathcal{H}$ is the hour,
    $t\in\mathcal{T}$ the sub-hour interval, and hourly costs are averaged by $1/|\mathcal{T}|$.

### Sets and indices

| Symbol | Index | Meaning |
|---|---|---|
| $\mathcal{S}^{sc}_{d}$ | $s$ | Joint risk scenarios of day-group $d$: each pairs one outage sample with one renewable scenario |
| $\mathcal{S}^{sc,\ast}_{d}$ | $s$ | **Filtered** (retained, to-be-simulated) joint scenarios of $d$ |
| $\mathcal{R}$ | $r$ | Renewable scenarios (`RA Scenarios` sheet; base = `BASE`) |
| $\mathcal{I}^{GC}_{s}(d,h,t)$ | $i$ | Generators **forced out** in scenario $s$ at hour $(d,h,t)$ (the down entries of the outage mask; a hybrid storage unit and its main generator share one state) |
| $\mathcal{K}^{LC}_{s}$ | $k$ | Line-outage set of $s$; **empty in every run** ($=\varnothing$) because no line outages are sampled |

### Parameters

| Symbol | Meaning (unit) |
|---|---|
| $N^{r}$ | Number of outage samples per day-group (`num_risk_scenario`) |
| $w_r$ | Renewable-scenario weight, normalised so $\sum_{r} w_r = 1$ (`Weight` column of `RA Scenarios`) |
| $\pi_s$ | Joint-scenario probability weight $= w_{r(s)}/N^{r}$ |
| $a^{s}_{i,d,h,t}$ | Unit availability indicator in $s$ (`1` = online, `0` = outaged) |
| $S^{r}_{i,d,h}$ | Hourly VRE availability shape of unit $i$ under renewable scenario $r$ |
| $R^{30}_i,\,R^{60}_i$ | 30- and 60-minute ramp fraction, $\min(1,\,m\cdot\texttt{Ramp}_i)$ |
| $\Phi$ | System peak scaling factor (`system_peak_scale`) |
| $\lambda$ | Constant load swept in ELCC (MW, per-unit) |
| $\underline{E}_i$ | Minimum state of charge (`STOMIN`) |
| $\overline{E}_i$ | Storage energy capacity, MWh (`ES_MWh`). The GTEP formulation calls this quantity $E^{0}_i$, the initial storage energy capacity from which cumulative energy-capacity investment grows in later stages |
| $\text{MC}^{\min}$ | Redispatch marginal-cost floor (`min_redispatch_mc_value`, default $10^{-4}$) |
| $T^{r}$ | Repair-time bound (`repair_time_bound`: `average` / `95` / `90` percentile / `NA`) |
| $\tau^{ref}$ | Reference temperature for the forced-outage-rate regression (`reference_temp`; see [Scenario generation](#scenario-generation)) |

!!! note "$\pi_s$ here is a probability weight, not a reserve-scarcity price"
    GTEP and Operation use $\pi^{\mathrm{reg}},\pi^{\mathrm S},\pi^{\mathrm{NS}},\pi^{\mathrm{FLEX}}$
    for operating-reserve **scarcity prices** (\$/MWh), always with a superscript qualifier.
    RA's subscripted $\pi_s$, the joint-scenario probability weight used throughout
    [Reliability metrics](#reliability-metrics), shares the base glyph only by index-naming
    convention ($s$ for scenario). RA's objective carries no reserve-scarcity term, so an
    unqualified $\pi$ in a metric formula is a different quantity that shares the letter.

### Variables

RA uses the same dispatch symbols as Operation ($g$, $chg$, $soc$, $curt$, $ens$, $f$,
$\theta_n$, $p^{\mathrm{inj}}$, commitment $c$, start-up $su$). A superscript $s$ marks a
**post-contingency** quantity solved for scenario $s$; unsuperscripted symbols are the
reference-stage solution $g^{\mathrm{ref}}$, $chg^{\mathrm{ref}}$, $soc^{\mathrm{ref}}$.

| Symbol | Meaning |
|---|---|
| $g^{\mathrm{ref}}_{i,d,h,t}$ | Reference (risk-free) generation |
| $g^{s}_{i,d,h,t}$ | Post-contingency generation in scenario $s$ |
| $\delta^{s}_{i,d,h,t}$ | Absolute redispatch deviation $\lvert g^{s}-g^{\mathrm{ref}}\rvert$ |
| $chg^{s}_{i,d,h,t},\ soc^{s}_{i,d,h,t}$ | Post-contingency storage charge / state of charge |
| $ens^{s}_{n,d,h,t}$ | Post-contingency unserved energy (MW) |

!!! warning "There is no reserve-deployment variable"
    The reference stage carries **no reserves**, so post-contingency response cannot be
    modeled as deploying a reserve block held for that purpose. The role is filled by the
    free redispatch $g^{s}$ together with the deviation penalty $\delta^{s}$, the ramp bounds,
    and the system-wide contingency cap of
    [Stage 2](#stage-2-risk-realization-post-contingency-dispatch). RA's redispatch is an
    *economic* re-optimisation bounded by physical ramp rates and capacity, not a lookup of a
    pre-qualified reserve product. This distinction matters when reconciling RA's implied
    reserve need against a separate reserve-adequacy study.

---

## Related pages

- [RA Overview](./RA_Overview.md) — workflow, run modes, transmission representation.
- [RA Scenarios and Data](./RA_Scenarios_and_Data.md) — data objects for outage / renewable / joint scenarios.
- [RA Metrics, ELCC and DLOL](./RA_Metrics_ELCC_and_DLOL.md) — interpretation, ELCC search, DLOL rollups.
- [RA Settings and Scenarios Reference](./RA_Settings_and_Scenarios_Reference.md) — `RA Setting` / `RA Scenarios` sheet columns.
- [GTEP formulation](../GTEP/GTEP_Formulation.md) — shared sets, parameters, variables.
- [Operation formulation](../Operation/Operation_Formulation.md) — the dispatch model RA reuses.
