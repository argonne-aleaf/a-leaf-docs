# GTEP Expansion — Mathematical Formulation

This page is the mathematical reference for the A-LEAF generation and transmission
expansion planning (GTEP) model. It gives the full nomenclature, the expansion objective,
and every expansion-specific constraint family. The sets, parameters, and variables defined
here are shared by the [Operation](../Operation/Operation_Formulation.md) and
[RA](../RA/RA_Formulation.md) formulation pages, which add only their own extra symbols.
Read it when you need to know exactly what the model optimizes, how a setting enters an
equation, or how a reported cost or capacity quantity is defined.

The expansion model minimizes the discounted sum of investment, retirement, fixed O&M,
representative-day dispatch, scarcity, and policy-penalty costs over a multi-stage planning
horizon, subject to capacity accounting, reliability, and policy constraints. It embeds a
representative-day operational sub-problem. That dispatch core (system balance, power-flow
modes, reserves, unit commitment, ramping, storage, large flexible load, and hybrid
resources) is documented once on the
[Operation formulation page](../Operation/Operation_Formulation.md) and is only referenced
here.

**Page outline.** Notation (sets, parameters, variables), the objective function term by
term, then the constraints grouped by topic: capacity and retirement accounting,
transmission expansion, storage, resource limits, reliability and adequacy, policy targets,
and user-defined constraints. The last section describes how the embedded dispatch connects
to the investment variables.

!!! note "Structural features worth knowing before reading the equations"
    - The investment tax credit (ITC) is applied to overnight capital cost before
      annualization, so no explicit `(1−ITC)` factor appears in the objective.
    - The embedded operation cost is scaled by representative-day weights `σ_d`, a sub-hour
      averaging factor `1/|\mathcal T|`, and per-calendar-year discounting, which combine
      into a single weight `χ`.
    - The storage annual energy throughput (AET) limit carries a factor of 2 (a full cycle
      is a charge and a discharge) and efficiency weighting.
    - Per-product reserve flags (`regulation_reserve_flag`, `spinning_reserve_flag`,
      `flexibility_reserve_flag`, `nonspin_reserve_flag`) switch whole cost and constraint
      families on and off together.
    - Transmission losses are modeled as a flat system-wide demand gross-up rather than a
      per-line loss term.

---

## Notation conventions

- **Sets** use calligraphic capitals `\mathcal{·}`; subsets are marked by a superscript
  (for example `\mathcal{I}^{ES}`).
- **Parameters** are upper-case Latin or Greek, with the qualifier as a superscript and
  the indices as subscripts (for example `C^{\mathrm{inv}}_{i,y}`).
- **Variables** are lower-case (for example `g_{i,d,h,t,y}`); investment and retirement
  decisions use `u` with a superscript tag.
- Every constraint states its `\forall` quantifier. Operational quantities are indexed by
  the tuple `(d,h,t,y)`.

!!! info "`h` (hour) versus `t` (sub-hour interval)"
    The operational time axis has two levels. `h \in \mathcal{H}` indexes the **hours** of a
    representative day; `t \in \mathcal{T}` indexes **sub-hour intervals within an hour**.
    Costs summed over `(h,t)` are averaged back to an hourly basis by the factor
    `1/|\mathcal{T}|`. Most runs use `|\mathcal{T}| = 1`, so `t` is degenerate and `(h,t)`
    collapses to hourly; the second level exists for intra-hour (for example 5- or
    10-minute) reserve studies.

    `y \in \mathcal{Y}` is a **planning stage index**, not a calendar year. The calendar
    start year of stage `y` is `Y(y)` and its length in years is `L_y`. See
    [Planning horizon and multi-round](./GTEP_Planning_Horizon_and_Multi_Round.md).

### Sets and indices

| Symbol | Index | Meaning | Defined by |
|---|---|---|---|
| $\mathcal{I}$ | $i$ | Generation and storage unit blocks (candidate and existing) | Gen Technology and network data |
| $\mathcal{I}^{ES}$ | $i$ | Energy-storage units (`UNIT_CATEGORY == "STORAGE"`) | `UNIT_CATEGORY` |
| $\mathcal{I}^{ES,STO}$ | $i$ | Storage units whose duration is an investment decision | `ES_STO_INVEST_FLAG` |
| $\mathcal{I}^{VRE,\mathrm{fix}}$ | $i$ | Variable renewable units on a fixed generation profile | `FUEL_LIMIT == "Fixed Profile"` |
| $\mathcal{I}^{VRE,\mathrm{bud}}$ | $i$ | Variable renewable units governed by an energy budget | budget-constrained units |
| $\mathcal{I}^{ER}$ | $i$ | Economic-retirement-eligible units | `RET_FLAG == true` |
| $\mathcal{I}^{UC}$ | $i$ | Unit-commitment units | dispatch mode |
| $\mathcal{I}^{MR}$ | $i$ | Must-run units | must-run flag |
| $\mathcal{I}^{RP}$ | $i$ | Inter-temporal (ramp-constrained) units | ramp data |
| $\mathcal{I}^{RM}_{m}$ | $i$ | Units consuming raw material $m$ | `Material_Flag` |
| $\mathcal{D}$ | $d$ | Representative day groups | representative-day set |
| $\mathcal{H}$ | $h$ | Hours within a representative day | `run_H` |
| $\mathcal{T}$ | $t$ | Sub-hour intervals within an hour | `run_T` |
| $\mathcal{Y}$ | $y$ | Planning stages | planning stages |
| $\mathcal{K}$ | $k$ | Transmission branches | network branches |
| $\mathcal{K}^{EX}$ | $k$ | Expandable branches | `expansion_flag == true` |
| $\mathcal{K}^{DC}$ | $k$ | Asynchronous DC ties | `dc_line == true` |
| $\mathcal{N}$ | $n$ | Buses (nodes) | network buses |
| $\mathcal{Z}$ | $z$ | Operating-reserve zones | reserve zone table |
| $\mathcal{P}$ | $p$ | Policy zones (RPS, clean-energy standard, emissions, inertia) | policy zone table |
| $\mathcal{P}^{R}$ | $p$ | Planning-reserve zones | planning-reserve zone table |
| $\mathcal{S}$ | $s$ | Resource supply-curve zones | resource supply-curve zone table |
| $\mathcal{M}$ | $m$ | Raw materials | raw-material table |
| $\mathcal{G}$ | $g$ | Generation technology groups | `UNITGROUP` |
| $\mathcal{G}^{SL}$ | $g$ | Technology groups subject to a regional resource-supply-curve limit | `Resource_Limit_Flag` |

!!! info "Four independent zonal partitions of the same buses"
    $\mathcal P$ (policy), $\mathcal P^{R}$ (planning reserve), $\mathcal Z$ (operating
    reserve), and $\mathcal S$ (resource supply curve) each partition the bus set
    $\mathcal N$ independently. A bus's RPS or clean-energy zone, planning-reserve
    (capacity-accreditation) zone, reserve-sharing zone, and resource-supply-curve zone are
    configured in separate zone tables in the network workbook and need not coincide. A
    single-region case can collapse all four to the whole system, but a multi-region case
    (for example a state-level RPS nested inside a wider reserve-sharing pool) needs four
    different aggregations, which is why they remain distinct index sets.

    $\mathcal I^{VRE,\mathrm{fix}}$ and $\mathcal I^{VRE,\mathrm{bud}}$ partition variable
    generation by modeling choice, not by technology name. A fixed-profile unit is pinned to
    an hourly capacity factor with curtailment as its only degree of freedom. A
    budget-constrained unit (for example a hydro plant with a monthly or annual energy
    allocation) can reshape when within a representative day it generates, subject to an
    energy cap over the day or the year; see the
    [Operation page](../Operation/Operation_Formulation.md). Membership in
    $\mathcal I^{VRE,\mathrm{fix}}$ is independent of up-reserve eligibility: a
    non-dispatchable fixed-profile unit cannot carry up-reserves, but a dispatchable
    fixed-profile unit can, because eligibility is a per-unit technology attribute. See the
    [Operation page's renewable-scheduling section](../Operation/Operation_Formulation.md#fixed-profile-availability).

    $\mathcal K^{EX}$ and $\mathcal K^{DC}$ are not mutually exclusive: a branch can be both
    an asynchronous DC tie and eligible for expansion.

### Parameters

| Symbol | Meaning (unit) | Workbook or setting name |
|---|---|---|
| $CAP_i$ | Nameplate power capacity of one unit block (MW; per-unit scaled internally) | `CAP` |
| $U^{0}_i$ | Initial number of installed unit blocks | `EXUNITS` |
| $\overline{U}^{\mathrm{inv}}_i,\ \underline{U}^{\mathrm{inv}}_i$ | Maximum and minimum investment blocks per unit | `MAXINVEST`, `MININVEST` |
| $\overline{P}_i,\ \underline{P}_i$ | Maximum and minimum generation as a fraction of $CAP_i$ | `PMAX`, `PMIN` |
| $C^{\mathrm{inv}}_{i,y}$ | Annualized generation investment cost, ITC already embedded ($/kW-yr) | derived from capital cost, `ITC`, and `crpyears` |
| $C^{\mathrm{inv,STO}}_{i,y}$ | Annualized storage-duration cost, ITC already embedded ($/kWh-yr) | derived from storage capital cost |
| $FOM_{i,y}$ | Fixed O&M ($/kW-yr) | fixed O&M cost |
| $DECC_i$ | Decommissioning cost (k$/MW) | `DECC` |
| $MC_{i,y,d}$ | Marginal (fuel plus variable O&M) cost per stage and day group ($/MWh) | derived from fuel price and variable O&M |
| $EF_i$ | CO$_2$ emission factor (tonne/MWh) | `Emission_CO2` |
| $CTAX$ | Carbon tax ($/tonne) | `CTAX` |
| $C^{\mathrm{reg}}_i,C^{\mathrm{spin}}_i,C^{\mathrm{nspin}}_i,C^{\mathrm{flex}}_i$ | Reserve provision costs ($/MWh, or fraction of $MC$) | `reg_cost`, `spin_cost`, `nspin_cost`, `flex_cost` |
| $C^{SU}_i,C^{SD}_i,C^{NL}_i$ | Start-up, shut-down, and no-load cost ($/MW) | `SUC`, `SDC`, `NLC` |
| $PTC_{i,y},ITC_{i,y}$ | Production and investment tax credit | `PTC`, `ITC` |
| $\eta_i$ | One-way storage efficiency $=\sqrt{\texttt{BATEFF}}$ | `BATEFF` |
| $CAP^{\mathrm{chg}}_i$ | Storage charge capacity (MW) | `Charge_CAP` |
| $E^{0}_i$ | Initial storage energy capacity (MWh) | `ES_MWh` |
| $\underline{D}_i,\overline{D}_i$ | Minimum and maximum storage duration (h) | `STOHR_MIN`, `STOHR_MAX` |
| $AET_i$ | Annual energy-throughput limit | `AET` |
| $\kappa_i$ | Capacity credit ($UCAP_i = CAP_i\,\kappa_i$) | `CAPCRED` |
| $PRM$ | Planning reserve margin | `planning_reserve_margin_value` |
| $F_k$ | Branch rating (MW; per-unit scaled internally) | `rate_a` |
| $\overline{F}_k$ | Maximum branch rating | `max_rate_a` |
| $\ell_k$ | Branch length (mile) | `length` |
| $C^{\mathrm{inv},T}_k$ | Annualized transmission cost ($/MW-mile-yr, or $/MW-yr for DC ties) | `transmission_expansion_cost` |
| $\overline{\Delta}^{T}$ | Transmission expansion limit | `transmission_expansion_limit_value` |
| $VOLL$ | Value of lost load ($/MWh) | `VOLL` |
| $\pi^{\mathrm{reg}},\pi^{\mathrm{S}},\pi^{\mathrm{NS}},\pi^{\mathrm{FLEX}}$ | Reserve scarcity prices | `RegRSP`, `SRSP`, `NSRSP`, `FLEXRSP` |
| $CET_{p,y}$ | Clean-energy generation target (fraction) | clean-energy generation target table |
| $RPS_{p,y}$ | Renewable portfolio standard (fraction of annual demand) | RPS target table |
| $EMT_{p,y}$ | Emission reference level and reduction target | carbon-emission reduction target table |
| $\rho^{CEG}$ | Clean-energy compliance-slack penalty price | `Clean_Energy_Generation_Penalty` |
| $\rho^{RPS}$ | RPS compliance-slack penalty price | `RPS_Penalty` |
| $RS_{g,s}$ | Regional resource limit (MW) | `Resource_Limit_MW` |
| $M^{+}_{m,y},\ \alpha_{m,i}$ | Annual material limit and material intensity | raw-material limit and `GenTech_Raw_Materials` tables |
| $\sigma_d$ | Number of days represented by group $d$ | `NumDays` |
| $\Lambda$ | Horizon day-scale $=\tfrac{1}{365}\sum_d\sigma_d$ | derived |
| $\omega(\tau)$ | Discount factor $(1+r)^{-(\tau-y_0)}$ to dollar year $y_0$ | discount rate and `dollar_year_value` |
| $L_y,\ Y(y)$ | Stage length (yr) and stage calendar start year | `stage_length` and planning-stage table |

!!! info "Parameters worth a second look"
    - $C^{\mathrm{inv}}_{i,y}$ and $C^{\mathrm{inv,STO}}_{i,y}$ can vary **by planning
      stage**, so a technology commissioned in stage 3 can be cheaper than the same
      technology commissioned in stage 1. The cost-curve inputs in the workbook determine
      whether the curve is flat or declining.
    - $MC_{i,y,d}$ is indexed by both stage $y$ and representative day group $d$, so fuel
      and variable O&M cost can vary seasonally (for example winter versus summer gas
      prices) within a single stage. This lets $Z^{\mathrm{op}}$ reproduce a
      seasonal-price signal even though the year is collapsed into a few representative
      days.
    - $CRP_i$ (`crpyears`) is the number of years for which a unit's investment annuity is
      paid. It differs from `Life_i`, which governs forced end-of-life retirement (see
      [Retirement accounting](#retirement-accounting)). A unit can have a 20-year
      cost-recovery period and a 40-year physical life.
    - $\kappa_i$ (`CAPCRED`) is either a numeric derate or a string key into a data-region
      capacity-credit table. The table path lets resource-adequacy results (ELCC, the
      *Effective Load Carrying Capability*, a technology's derated contribution to peak
      reliability; see [RA Metrics, ELCC, and DLOL](../RA/RA_Metrics_ELCC_and_DLOL.md))
      feed back into the planning-reserve-margin constraint across multi-round GTEP and RA
      iterations (see
      [Planning horizon and multi-round](./GTEP_Planning_Horizon_and_Multi_Round.md)).
    - $\Lambda$ equals 1 whenever the representative-day set sums to 365 days. When scenario
      reduction selects a partial day set (fewer than 365 represented days; see
      [Scenario Reduction and Repday Groups](../../configuration/Scenario_Reduction_and_Repday_Groups.md)),
      $\Lambda<1$ scales every fixed annual charge in the objective (investment, storage
      duration, transmission investment, fixed O&M, decommissioning) down to the fraction of
      the year actually represented, keeping fixed-cost accounting consistent with the
      truncated operational sample. $\Lambda$ does not appear in the raw-material limit,
      whose right-hand side is a physical annual quantity rather than a cost.

### Variables

| Symbol | Meaning | Output column or note |
|---|---|---|
| $u^{\mathrm{new},G}_{i,y}$ | New generation investment (unit blocks) | expansion outputs |
| $u^{\mathrm{ret},G}_{i,y}$ | Generation retirement (unit blocks) | expansion outputs |
| $u^{G}_{i,y}$ | Cumulative installed unit blocks | expansion outputs |
| $u^{\mathrm{new},ESH}_{i,y}$ | New storage-duration investment (h) | expansion outputs |
| $u^{ESE}_{i,y}$ | Cumulative storage energy capacity (MWh) | expansion outputs |
| $u^{\mathrm{new},T}_{k,y}$ | Transmission expansion increment (dimensionless multiplier) | expansion outputs |
| $u^{T}_{k,y}$ | Cumulative transmission multiplier | expansion outputs |
| $g_{i,d,h,t,y}$ | Generation (MW) | dispatch outputs |
| $chg_{i,d,h,t,y}$ | Storage charging (MW) | dispatch outputs |
| $soc_{i,d,h,t,y}$ | State of charge (MWh) | dispatch outputs |
| $curt_{i,d,h,t,y}$ | Curtailment (MW) | dispatch outputs |
| $ens_{n,d,h,t,y}$ | Energy not served (MW) | dispatch outputs |
| $f_{k,d,h,t,y}$ | Branch flow (MW) | dispatch outputs |
| $\theta_{n,d,h,t,y}$ | Bus voltage angle (B-θ mode only) | |
| $p^{\mathrm{inj}}_{n,d,h,t,y}$ | Net nodal injection (PTDF mode only) | |
| $c_{i,d,h,t,y}$ | Unit commitment (integer) | |
| $su_{i,d,h,t,y},\ sd_{i,d,h,t,y}$ | Start-up and shut-down (integer) | |
| $r^{\mathrm{regUp}},r^{\mathrm{regDn}},r^{\mathrm{spin}},r^{\mathrm{nspin}},r^{\mathrm{flexUp}},r^{\mathrm{flexDn}}_{i,d,h,t,y}$ | Reserve provision by unit | |
| $rns^{\mathrm{regUp}},rns^{\mathrm{regDn}},rns^{\mathrm{cont}},rns^{\mathrm{nspin}},rns^{\mathrm{flexUp}},rns^{\mathrm{flexDn}}_{z,d,h,t,y}$ | Reserve scarcity (shortfall) by zone | |
| $c^{STO}_{i,d,h,t,y}$ | Storage commitment (binary) | |
| $\varsigma^{RPS}_{p,y},\ \varsigma^{CEG}_{p,y}\,(\text{or }{}_{p,d,y})$ | RPS and clean-energy compliance slack | |

Result files report these quantities under their own column names; see
[Expansion outputs](./GTEP_Expansion_Outputs.md) and
[Operation outputs](../Operation/Operation_Outputs.md).

!!! info "Variable pairs that are easy to conflate"
    - $g$ versus $curt$: $g$ is what is dispatched onto the network; $curt$ is potential
      generation (mostly variable renewable) that is given up. It is tracked as its own
      variable so it can be reported without perturbing the energy balance that $g$
      satisfies.
    - $ens$ versus $curt$: $ens$ is a demand-side scarcity variable (bus-level unmet load,
      priced at $VOLL$ in the objective); $curt$ is a supply-side one (a unit's unused
      available energy). They enter through different mechanisms and never substitute for
      one another in the balance equations.
    - $\theta$ exists only when `power_flow_mode_flag == "B-theta"`; $p^{\mathrm{inj}}$
      exists only under `"PTDF"`. The two are never decision variables in the same solve;
      see the [Operation formulation page](../Operation/Operation_Formulation.md).
    - $c,\ su,\ sd$ exist only when `Dispatch_Mode_in_EXP == "Unit Commitment"`. Under
      continuous (economic) dispatch there is no commitment variable, and $g$ is bounded
      directly by $u^{G}_{i,y}\,CAP_i\,\overline P_i$.
    - $r^{(\cdot)}_{i,d,h,t,y}$ (per-unit reserve provision, a real decision with real
      cost) versus $rns^{(\cdot)}_{z,d,h,t,y}$ (zonal reserve shortfall, a slack): the
      shortfall takes a positive value only when no combination of unit-level $r$ across
      the zone can meet the zone's aggregate requirement. It is the reserve-market
      analogue of $ens$.

---

## Objective — expansion

The model minimizes the sum of the term groups below:

$$
\min\; \underbrace{Z^{\mathrm{inv}}}_{\text{gen invest}}
     + \underbrace{Z^{\mathrm{STO}}}_{\text{storage duration}}
     + \underbrace{Z^{T}}_{\text{tx invest}}
     + \underbrace{Z^{T,\mathrm{FOM}}}_{\text{tx fixed O\&M}}
     + \underbrace{Z^{\mathrm{ret}}}_{\text{decommission}}
     + \underbrace{Z^{\mathrm{FOM}}}_{\text{fixed O\&M}}
     + \underbrace{Z^{\mathrm{op}}}_{\text{embedded dispatch}}
     - \underbrace{Z^{\mathrm{PTC}}}_{\text{prod. tax credit}}
     + \underbrace{Z^{\mathrm{scar}}}_{\text{scarcity}}
     + \underbrace{Z^{\mathrm{pol}}}_{\text{policy penalties}}
$$

Throughout, $\omega(\tau)=(1+r)^{-(\tau-y_0)}$ is the discount factor to the dollar year
$y_0$ (`dollar_year_value`), and each stage's annual costs are spread over the calendar
years the stage spans by summing $\tau = Y(y)+f-1$ for $f=1,\dots,L_y$ (or over the
investment payment horizon). $\Lambda=\tfrac1{365}\sum_d\sigma_d$ scales annual fixed
costs when the horizon is represented by a partial day set.

The ten-term decomposition matches how the results break the solved objective into cost
columns (see [Expansion outputs](./GTEP_Expansion_Outputs.md)), so each $Z$ term maps to
a reported cost line. Economically, the terms separate decisions with different time
profiles: one-time capital outlays ($Z^{\mathrm{inv}}$, $Z^{\mathrm{STO}}$, $Z^T$), a
one-time exit cost ($Z^{\mathrm{ret}}$), recurring ownership costs independent of use
($Z^{\mathrm{FOM}}$), recurring costs proportional to use ($Z^{\mathrm{op}}$), a recurring
subsidy ($Z^{\mathrm{PTC}}$, subtracted), and two backstop terms that become active only
when the core constraints cannot otherwise be met economically ($Z^{\mathrm{scar}}$,
$Z^{\mathrm{pol}}$).

### Generation investment $Z^{\mathrm{inv}}$

$$
Z^{\mathrm{inv}} = \Lambda \sum_{y\in\mathcal Y}\ \sum_{i\in\mathcal I}\
\sum_{f=1}^{\Pi_{i,y}} \omega\!\big(Y(y)+f-1\big)\;
\big(1000\, C^{\mathrm{inv}}_{i,y}\, CAP_i\big)\; u^{\mathrm{new},G}_{i,y}
$$

with payment horizon $\Pi_{i,y}=\min\!\big(CRP_i,\ (N{+}1{-}y)L_y\big)$, where $N$ is the
number of stages. The factor 1000 converts $/kW capital cost to $/MW against MW capacity.

This is the annualized overnight capital cost of new capacity. It stands in for
construction financing (interest during construction, debt service, equity return) folded
into a single $/kW-yr charge, and it is what makes building a plant costly as opposed to
paying only fixed O&M once it exists. Without it, nothing would stop the model from
over-building free capacity to relax every other constraint (planning reserve margin, RPS,
reserves).

The payment horizon $\Pi_{i,y}$ is a truncation, not a discount: a unit built with
`crpyears`$=20$ but only 6 calendar years remaining in the study horizon pays 6 years of
annuity, not 20. The model carries no terminal or salvage value for the unrecovered
remainder of the capital cost, so a unit built in the last stage pays only what falls
inside the horizon. This can bias the model toward front-loading retirements and
back-loading new investment near the end of the study; late-stage investment decisions are
not on a level annuity footing with early-stage ones.

!!! info "ITC is not an explicit objective factor"
    The investment term has no explicit $(1-ITC)$ factor. The ITC reduces the overnight
    capital cost before annualization, so it is already contained in
    $C^{\mathrm{inv}}_{i,y}$. A technology with `ITC == 0` (or `ITC_Flag == false`) has an
    annualized cost equal to its full, undiscounted capital cost.

### Storage-duration investment $Z^{\mathrm{STO}}$

$$
Z^{\mathrm{STO}} = \Lambda \sum_{y}\ \sum_{i\in\mathcal I^{ES}}\
\sum_{f=1}^{\Pi_{i,y}} \omega\!\big(Y(y)+f-1\big)\;
\big(1000\, C^{\mathrm{inv,STO}}_{i,y}\, CAP_i\big)\; u^{\mathrm{new},ESH}_{i,y}
\qquad \forall\, i\in\mathcal I^{ES}
$$

Storage (batteries, pumped-storage hydropower) has two nearly independent cost drivers:
power-conversion hardware ($/kW, priced by $Z^{\mathrm{inv}}$ through $u^{\mathrm{new},G}$
like any other technology) and energy media (cells, reservoir volume), priced per kWh of
duration by this term. Separating the two lets the model answer how many hours of storage
a resource should have, not only how many MW. The cost is charged against
$u^{\mathrm{new},ESH}$ (hours of new duration), so the $/kWh parameter applies whether or
not `ES_STO_INVEST_FLAG` lets the unit choose its own duration; a unit whose duration is
pinned to a fixed ratio (see [Storage duration and investment link](#storage-duration-and-investment-link))
still pays for the energy hardware that ratio implies.

### Transmission investment $Z^{T}$

$$
Z^{T} = \Lambda \sum_{y}\ \sum_{k\in\mathcal K^{EX}}\
\sum_{f=1}^{\Pi^{T}_{y}} \omega\!\big(Y(y)+f-1\big)\;
C^{\mathrm{inv},T}_k\, F_k\, \ell^{\ast}_k\; u^{\mathrm{new},T}_{k,y},
\qquad
\ell^{\ast}_k=\begin{cases}1 & k\in\mathcal K^{DC}\\ \ell_k & \text{otherwise}\end{cases}
$$

with $\Pi^{T}_y=\min\!\big(CRP^{T},(N{+}1{-}y)L_y\big)$. There is no factor of 1000
because transmission cost is already expressed in $/MW-mile. Cost units, per-line caps,
DC-tie treatment, and power-flow-mode dependence are described on the
[Transmission Expansion](./GTEP_Transmission_Expansion.md) page.

Without a cost on transmission expansion, every existing corridor would be freely
widenable, which erases locational price signals and pushes the system toward a
copper-plate dispatch. This term prices the tension between building cheap remote
generation and paying to move its energy to load. DC ties are charged per MW rather than
per MW-mile because a DC tie upgrade is typically a converter-station capacity increase
whose cost does not scale with corridor length. Expansion only reinforces an existing
corridor (see [Transmission expansion](#transmission-expansion-multiplier-and-per-line-cap)),
so this term cannot create a connection between two buses that have no existing branch.

### Transmission fixed O&M $Z^{T,\mathrm{FOM}}$

$$
Z^{T,\mathrm{FOM}} = \Lambda \sum_{y}\ \sum_{k\in\mathcal K^{EX}}\
\sum_{f=1}^{L_y} \omega\!\big(Y(y)+f-1\big)\;
C^{\mathrm{FOM},T}_k\, F_k\, \ell^{\ast}_k\; u^{T}_{k,y}
$$

where $C^{\mathrm{FOM},T}_k$ is the branch's fixed O&M cost basis (overnight capital cost
times `transmission_FOM_percent_value` times 0.01), $F_k$ is its thermal rating, and
$\ell^{\ast}_k$ is the same length factor as in $Z^T$. Like generation fixed O&M, it runs
over every year of the stage ($f=1,\dots,L_y$), is not truncated by a cost-recovery
period, and carries no factor of 1000.

!!! info "Fixed O&M is charged on the expansion increment only"
    The term multiplies the cumulative transmission multiplier $u^{T}_{k,y}$ for
    expansion-eligible branches $\mathcal K^{EX}$, so it charges the expansion increment
    and not the existing grid. The model has no transmission-retirement decision, so the
    fixed O&M of already-installed transmission is a constant that no decision can change,
    and adding a constant to a minimization does not move the optimum. It is therefore left
    out of the objective and added back in cost reporting. Generation fixed O&M
    $Z^{\mathrm{FOM}}$, by contrast, is in the objective because generators can retire
    ($u^{G}_{i,y}$ falls), which makes existing-fleet fixed O&M decision-relevant.

!!! warning "Objective term versus reported `Trans_FOM_Cost`"
    The system-summary columns `Trans_FOM_Cost` and `Trans_FOM_Cost_PV` report fixed O&M on
    the full in-service grid, every branch, at
    $C^{\mathrm{FOM},T}_k\,F_k\,\ell^{\ast}_k\,(1+u^{T}_{k,y})$. The reported cost therefore
    equals this objective term plus the constant existing-grid fixed O&M that the objective
    omits. The [Operation model](../Operation/Operation_Overview.md) adds no transmission
    fixed O&M term to its objective; it fixes $u^{T}_{k,y}$ to the expansion solution and
    dispatches a fixed network, so this cost appears only in its report (and is excluded
    from its operation-only total). See
    [GTEP Expansion Outputs](./GTEP_Expansion_Outputs.md#11-system-summary-output) and
    [GTEP Transmission Expansion](./GTEP_Transmission_Expansion.md#transmission-fixed-om-fom).

### Decommissioning $Z^{\mathrm{ret}}$

$$
Z^{\mathrm{ret}} = \Lambda \sum_{y}\ \sum_{i:\,DECC_i>0}
\omega\!\big(Y(y)\big)\; \big(1000\, DECC_i\, CAP_i\big)\; u^{\mathrm{ret},G}_{i,y}
$$

Decommissioning is discounted at the stage start year $Y(y)$: it is a one-off charge, not
spread over the stage. It represents the cash cost of physically removing a unit
(demolition, site remediation, interconnection removal), distinct from lost future
revenue, which the model captures by no longer dispatching the unit. Technologies with
`DECC == 0` are excluded from the sum and retire at no cost; their retirement is governed
only by the [Retirement accounting](#retirement-accounting) constraint.

### Fixed O&M $Z^{\mathrm{FOM}}$

$$
Z^{\mathrm{FOM}} = \Lambda \sum_{y}\ \sum_{i}\
\sum_{f=1}^{L_y} \omega\!\big(Y(y)+f-1\big)\;
\big(1000\, FOM_{i,y}\, CAP_i\big)\; u^{G}_{i,y}
$$

Fixed O&M is the recurring cost of keeping a built unit staffed, insured, and maintained
regardless of how much energy it produces, the fixed-cost counterpart to the variable
dispatch cost inside $Z^{\mathrm{op}}$. It is charged on the cumulative installed fleet
$u^{G}_{i,y}$, not on new investment, because it applies equally to inherited capacity
(`EXUNITS`) and to anything built in an earlier stage that is still in service. Unlike the
investment terms it is not truncated by a cost-recovery period; it runs for every calendar
year the unit exists. Fixed O&M is often the deciding factor in economic retirement: a
unit with a thin positive energy margin can be retired early if its fixed O&M outweighs
that margin.

### Embedded operation cost $Z^{\mathrm{op}}$

The representative-day dispatch cost is scaled by day weight $\sigma_d$, sub-hour
averaging $1/|\mathcal T|$, and per-year discounting. With the common weight
$\chi_{d,y,f}=\dfrac{\sigma_d\,\omega(Y(y)+f-1)}{|\mathcal T|}$:

$$
\begin{aligned}
Z^{\mathrm{op}} = \sum_{y}\sum_{f=1}^{L_y}\sum_{d}\chi_{d,y,f}\Bigg[\;
&\sum_{i}\sum_{h,t}\Big( MC_{i,y,d}\, g_{i,d,h,t,y}
      + CTAX\cdot EF_i\, g_{i,d,h,t,y}\Big) \\[2pt]
+\ &\sum_{i}\sum_{h,t}\Big(
   C^{\mathrm{reg}}_i(r^{\mathrm{regUp}}_{i}+r^{\mathrm{regDn}}_{i})
 + C^{\mathrm{spin}}_i r^{\mathrm{spin}}_{i}
 + C^{\mathrm{nspin}}_i r^{\mathrm{nspin}}_{i}
 + C^{\mathrm{flex}}_i(r^{\mathrm{flexUp}}_{i}+r^{\mathrm{flexDn}}_{i})\Big) \\[2pt]
+\ &\sum_{i\in\mathcal I^{UC}}\sum_{h}\Big( C^{SU}_i\, su_{i,d,h,1,y} + C^{NL}_i\, c_{i,d,h,1,y}\Big)
\;\Bigg]
\end{aligned}
$$

- **Generation and carbon.** The carbon-tax term applies only where $EF_i>0$ and
  $CTAX>0$.
- **Reserve costs.** Each product term appears only when its reserve flag is on, and
  up-products only for dispatchable units. If a product's flag is `false`, its variables,
  zonal requirement, and cost are all absent from the model. When
  `reserve_cost_type_flag == "percentage"`, $C^{\mathrm{reg/spin/nspin/flex}}_i$ are read
  as fractions of $MC_{i,y,d}$, and the floor `min_regulation_cost_value` keeps regulation
  from being free. Non-spinning reserve cost enters only under `Unit Commitment` dispatch.
- **Unit commitment.** Start-up cost applies to each start-up and no-load cost to each
  committed interval. Shut-down cost $C^{SD}_i$ does not enter the objective; $sd$ carries
  no cost.
- **Large flexible load and hybrid resources.** The demand-response value of large flexible
  load enters with a negative coefficient (value of served load), and hybrid storage
  grid-charging carries a small tie-breaker penalty. These belong to the large-flexible-load
  and hybrid formulation; see the [Operation formulation](../Operation/Operation_Formulation.md).

This term is where the dispatch sub-problem's cost enters the expansion objective. An
investment decision pays off only through the fuel, carbon, reserve, and start-up costs it
lets the embedded dispatch avoid or incur, which is why expansion and operation are solved
together. The compound weight $\chi_{d,y,f}$ does three separate jobs:

- $\sigma_d$ scales one representative day's dispatch cost up to what that day type costs
  over the year (a group standing for 60 winter days contributes 60 times one winter day).
- $1/|\mathcal T|$ prevents finer sub-hourly resolution from inflating total cost merely
  because more $(h,t)$ terms are summed. It equals 1 when a single interval per hour is
  used, the common case.
- $\omega(Y(y)+f-1)$ discounts each calendar year of the stage back to the dollar year.

!!! example "Worked example of the $\chi$ weight"
    A day group $d$ with $\sigma_d=50$ represented days, in a stage starting
    $Y(y)=2035$, evaluated at $f=1$ (calendar year 2035), with `dollar_year_value`$=2025$
    and a 5% discount rate: $\chi_{d,y,1} = 50 \times (1.05)^{-10} / 1 \approx 50 \times
    0.6139 \approx 30.7$. A coal unit dispatched at $g=100$ MW for all 24 hours of that
    representative day at $MC=\$25$/MWh contributes
    $\chi_{d,y,1}\times MC \times \sum_h g \approx 30.7\times25\times2{,}400 \approx
    \$1.84\text{M}$ to $Z^{\mathrm{op}}$ from that one day group and stage-year; the full
    term sums this contribution over every $(d,h,t,f)$ and every unit.

Turning off a reserve flag (for example `nonspin_reserve_flag`) removes the product's
variable and zonal requirement from the model entirely, so the case behaves as if the
product were never modeled, not merely free. The `"percentage"` cost type reflects the
common heuristic that a unit's opportunity cost of holding reserve scales with its own
fuel cost. Without the `min_regulation_cost_value` floor, a very low-$MC$ unit (for
example a nuclear plant) would supply regulation at essentially zero cost.

### Production tax credit $Z^{\mathrm{PTC}}$ (subtracted)

The credit applies only when `PTC_Flag == true` and the unit's `PTC Flag` is set. It enters
the objective as a negative contribution, with $ptc = \texttt{PTC}[\tau]\times 10$
converting cents/kWh to $/MWh:

- **Existing, non-nuclear units:** subtract $ptc\cdot g$ for the first 5 eligible years.
- **Existing nuclear units:** subtract $ptc\cdot g$ for all stage-years.
- **New units:** subtract $ptc$ times an estimated annual generation (capacity-factor shape
  times $CAP\cdot PMAX$) applied to $u^{\mathrm{new},G}$ over $\min(10,\text{remaining})$
  years, with the credit fixed at the investment-year value.

A production tax credit is a per-MWh subsidy, so a cost-minimizing objective models it as
a negative cost: each qualifying MWh reduces net system cost, which makes credit-eligible
technologies more competitive in the investment decision. The three branches reflect
different eligibility windows by asset vintage. Existing non-nuclear assets typically
qualify for a short remaining window (modeled as 5 years), existing nuclear can qualify
throughout, and a new asset earns the credit for up to 10 years from its own commissioning
year regardless of when in the horizon it is built.

The new-asset branch departs from "credit times actual dispatch". Investment
$u^{\mathrm{new},G}_{i,y}$ is a stage-level decision, so the credit multiplies the
investment variable by an estimated annual generation built from the technology's annual
capacity-factor shape (the same shapes used by the [RPS constraint](#policies-renewable-portfolio-standard-rps))
times nameplate capacity. This fixes the credit at the rate in effect when the project is
placed in service. A new asset's estimated credit therefore does not respond to
curtailment or low realized capacity factor in the embedded dispatch; it is set at
investment time by the technology's typical shape.

### Scarcity $Z^{\mathrm{scar}}$

$$
\begin{aligned}
Z^{\mathrm{scar}} = \sum_{y}\sum_{f=1}^{L_y}\sum_{d}\chi_{d,y,f}\Bigg[\;
&\sum_{n} VOLL\cdot ens_{n,d,h,t,y}
+ \sum_{z}\Big( \pi^{\mathrm{reg}}\,(rns^{\mathrm{regUp}}_{z}+rns^{\mathrm{regDn}}_{z})
+ \pi^{\mathrm{S}}\, rns^{\mathrm{cont}}_{z} \\
&+\ \pi^{\mathrm{NS}}\, rns^{\mathrm{nspin}}_{z}
+ \pi^{\mathrm{NS}}\, rns^{\mathrm{flexUp}}_{z}
+ \pi^{\mathrm{FLEX}}\, rns^{\mathrm{flexDn}}_{z}\Big)\Bigg]
\end{aligned}
$$

Each reserve-shortfall term is active only when the corresponding reserve flag is on.

The term $VOLL\cdot ens$ is the model's backstop against infeasibility. Instead of making
the whole problem infeasible in any hour where installed capacity, imports, and reserves
cannot cover load, $ens$ absorbs the shortfall at a steep but finite price, so the solve
returns a usable answer that also quantifies the shortfall. This matters most in early
stages (before enough capacity has been built) and in stress scenarios (aggressive
retirements, tight policy targets). $VOLL$ is normally set far above any dispatch cost so
that $ens>0$ occurs only when every cheaper alternative (generation, transmission,
reserves, storage cycling) is exhausted. If $VOLL$ is set below the marginal cost of the
most expensive dispatchable unit, the model can choose to shed load rather than run that
unit, which distorts both the reported unserved energy and the implied reliability of the
resulting plan.

The reserve-shortfall terms apply the same backstop to each operating reserve product. They
allow a zone's regulation, spinning, or flexibility requirement to go unmet at a scarcity
price approximating the cost of degraded reliability, instead of forcing infeasibility when
ramp limits, minimum up-times, or commitment constraints make the exact requirement
unreachable in an hour.

!!! info "Scarcity price assignments"
    Non-spinning-reserve shortfall $rns^{\mathrm{nspin}}$ is priced at $\pi^{\mathrm{NS}}$
    (`NSRSP`). Two products share a price column: flexibility-up shortfall
    $rns^{\mathrm{flexUp}}$ is also priced at $\pi^{\mathrm{NS}}$, while flexibility-down
    uses $\pi^{\mathrm{FLEX}}$ (`FLEXRSP`); the single contingency shortfall
    $rns^{\mathrm{cont}}$ is priced at $\pi^{\mathrm{S}}$ (`SRSP`). Regulation up and down
    shortfalls are priced at $\pi^{\mathrm{reg}}$ (`RegRSP`).

### Policy penalties $Z^{\mathrm{pol}}$

Compliance-slack penalties keep the RPS and clean-energy targets feasible:

$$
Z^{\mathrm{pol}} = \sum_{y}\sum_{f=1}^{L_y}\dfrac{\omega(\cdot)}{|\mathcal T|}\Bigg[
\underbrace{\rho^{CEG}\!\!\sum_{p}\varsigma^{CEG}_{p,y}}_{\text{annual}}
\ \text{or}\ \underbrace{\rho^{CEG}\!\!\sum_{d}\sigma_d\!\sum_{p}\varsigma^{CEG}_{p,d,y}}_{\text{day-group}}
\;\Bigg]
\ +\ \sum_{y}\sum_{f=1}^{L_y}\omega(\cdot)\,\rho^{RPS}\!\sum_{p}\varsigma^{RPS}_{p,y}
$$

The clean-energy slack uses the annual or the day-group form, matching the clean-energy
constraint variant in use. The RPS slack term is present only when
`Allow_Alternative_RPS_Compliance_Flag == true`. A negligible tie-breaker penalty
($10^{-6}$) is also placed on storage commitment and hybrid grid-charge variables to break
degeneracy without perturbing dispatch.

!!! info "The clean-energy slack coefficient carries a $1/|\mathcal T|$ factor"
    The clean-energy slack coefficient includes the sub-hour averaging factor
    $1/|\mathcal T|$ even though $\varsigma^{CEG}$ is not indexed by $(h,t)$ and no
    $(h,t)$ summation occurs in the term. When $|\mathcal T|=1$, the usual case, this has no
    effect. The RPS slack term carries no such factor.

These slack penalties keep an ambitious clean-energy or RPS target from turning an
otherwise solvable case into an infeasible one. Policy targets are sometimes set ahead of
confirmed resource availability or transmission build-out, and an analyst often wants to
know how far short of a target the least-cost system falls. The solved slack value is that
compliance gap, and the penalty price keeps the slack from being used unless necessary.
The RPS slack is fixed to zero unless `Allow_Alternative_RPS_Compliance_Flag` is set, so an
unreachable RPS target by default appears as solver infeasibility. The flag is an explicit
opt-in to soften that behavior, comparable to alternative compliance payments in some
jurisdictions.

The $10^{-6}$ tie-breaker penalties address a numerical issue. In the LP relaxation, a
storage unit's commitment status or a hybrid asset's grid-charging path can be degenerate:
many equally cheap solutions satisfy every dispatch and energy-balance constraint while
leaving these variables free in a flat region of the objective. Without a small penalty,
the solver could return an arbitrary, non-reproducible pattern of commitment or
grid-charging with no cost effect. The penalty is about three orders of magnitude below the
cheapest real per-unit marginal cost, so it nudges the solver toward a canonical
minimal-usage solution without measurably changing economic dispatch.

---

## Expansion constraints

This section covers investment limits, capacity and retirement accounting, storage sizing,
transmission expansion, and regional resource limits. Reliability and policy constraints
follow in later sections.

### Investment cap (per-unit)

Investment is bounded directly on the variable; `INVEST_FLAG == false` fixes it to 0:

$$
\underline{U}^{\mathrm{inv}}_i \le u^{\mathrm{new},G}_{i,y} \le \overline{U}^{\mathrm{inv}}_i
\qquad \forall\, i,y
$$

`Integrality == true` makes the variable integer. Storage-duration investment is bounded by

$$
0 \le u^{\mathrm{new},ESH}_{i,y} \le \overline{D}_i\,\overline{U}^{\mathrm{inv}}_i
$$

`MININVEST` and `MAXINVEST` typically capture a site-level build constraint: a candidate
project has a maximum buildable size (land, interconnection queue position, permit), or the
workbook forces a minimum build of a technology at a site by a given stage. Without an
upper bound, nothing else prevents the solver from building an unlimited amount of the
cheapest technology in a single stage; construction, permitting, and supply-chain pacing
are represented through these bounds. `INVEST_FLAG == false` disables a technology in a
specific case (for example no new coal) while keeping its row in the technology catalog for
other cases that share the workbook. `Integrality` distinguishes discretely sized plants (a
nuclear reactor is one indivisible unit block) from continuously sized ones (a wind or
solar farm can be built to any MW), which keeps most of the model a linear program with a
small integer footprint.

The storage-duration bound $\overline D_i\,\overline U^{\mathrm{inv}}_i$ is a consistency
cap rather than an independent limit. Even where `ES_STO_INVEST_FLAG` lets a unit choose
its duration, that duration cannot imply more energy capacity than the largest possible
power build (`MAXINVEST` blocks) times the technology's maximum duration (`STOHR_MAX`).

### System-wide investment window (per technology)

$$
\underline{U}^{G}_{g} \ \le\ \sum_{i:\,\mathrm{grp}(i)=g}\sum_{y} u^{\mathrm{new},G}_{i,y}
\ \le\ \overline{U}^{G}_{g}
\qquad \forall\, g\in\mathcal G
$$

Each bound is active only when the corresponding `SYSTEM_MININVEST` or `SYSTEM_MAXINVEST`
value is greater than 0.

This window is independent of the per-unit cap. `MAXINVEST` limits one candidate site,
whereas `SYSTEM_MAXINVEST` sums new investment across every candidate block and every stage
for a technology group (`UNITGROUP`). It is the appropriate tool when the workbook
represents one technology as many parallel site candidates (one row per bus or resource
zone) and the analyst wants an aggregate system-wide cap, for example a manufacturing or
interconnection-queue limit on total new nuclear capacity. `SYSTEM_MININVEST` serves the
opposite case, a policy mandate for at least some build of a technology (for example a
state-level offshore wind requirement) expressed at the technology-group level. The two
bounds are gated independently, so a workbook can specify a floor, a ceiling, both, or
neither.

### Capacity bookkeeping (cumulative fleet)

$$
u^{G}_{i,y} =
\begin{cases}
U^{0}_i + u^{\mathrm{new},G}_{i,y} - u^{\mathrm{ret},G}_{i,y} & y = y_{\text{first}}\\[3pt]
u^{G}_{i,y-1} + u^{\mathrm{new},G}_{i,y} - u^{\mathrm{ret},G}_{i,y} & y > y_{\text{first}}
\end{cases}
\qquad \forall\, i,y
$$

In multi-round mode, the first-stage equation also adds the net new-minus-retired unit
count recorded from earlier rounds, in addition to $U^0_i$.

This is the running stock variable that every other capacity-facing quantity reads: fixed
O&M, the planning reserve margin, the RPS proxy, and the resource supply-curve limit all use
$u^{G}_{i,y}$, so each sees the same installed capacity. The recursive form treats capacity
as a multi-stage stock: a unit built in stage 2 is still present, and still incurs fixed
O&M, in stage 5 unless retired in between. In multi-round mode, adding the recorded
decisions at the first stage lets a later round retain everything earlier rounds committed;
without it, each round would reset every unit's fleet to its pre-study `EXUNITS`.

*Worked example.* A unit starts with `EXUNITS`$=2$ (two existing 500 MW blocks). Stage 1
builds $u^{\mathrm{new}}=1$ and retires none, so $u^{G}_{i,1}=3$ (1,500 MW). Stage 2 builds
nothing and retires $u^{\mathrm{ret}}=1$, so $u^{G}_{i,2}=2$ (1,000 MW). Every use of
$u^G_{i,2}$, including fixed O&M, capacity credit, and the dispatch upper bound, sees
exactly two blocks.

### Retirement accounting

For units without economic retirement (`RET_FLAG == false`), retirement is fixed to the
exogenous planned schedule plus any end-of-life forced retirement of new assets:

$$
CAP_i\, u^{\mathrm{ret},G}_{i,y}
= U^{\mathrm{ret,plan}}_{i,y} \;\big[+\, CAP_i\, u^{\mathrm{new},G}_{i,\,y-\Theta_i+1}\big]
\qquad \forall\, i\notin\mathcal I^{ER},y
$$

For economic-retirement units (`RET_FLAG == true`), cumulative retirement is bounded below
by the planned and forced schedule, and the model may retire more endogenously:

$$
\sum_{q\le y} CAP_i\, u^{\mathrm{ret},G}_{i,q}
\ \ge\ \sum_{q\le y} U^{\mathrm{ret,plan}}_{i,q}
\;+\; \sum_{q=\Theta_i}^{y} CAP_i\, u^{\mathrm{new},G}_{i,\,q-\Theta_i+1}
\qquad \forall\, i\in\mathcal I^{ER},y
$$

Here $\Theta_i=\lceil \text{Life}_i/L_y\rceil+1$ is the service life in stages, and
$U^{\mathrm{ret,plan}}$ is the planned retirement for the stage's year from the
`Planned_Retirement` data (`Ret_{year}`).

Real fleets mix two retirement drivers. An exogenous schedule (an announced utility
retirement plan, a license expiration, a coal-phaseout law) is a workbook input independent
of economics. For units where `RET_FLAG` allows it, the model can also retire earlier than
planned when the unit becomes uneconomic (fixed O&M plus going-forward costs exceed the
value it earns). The equality for non-economic units forces exact compliance with the plan.
The inequality for economic units gives the model a one-sided lever: it can retire at least
as much as the plan requires and more if that minimizes cost, but never less. This asymmetry
is what allows GTEP to produce early, economics-driven coal or nuclear retirements.

$\Theta_i$ (forced end-of-life retirement of new assets) appears in both branches. A
vintage that has been in service past `Life_i` years, converted to whole planning stages
rounded up plus one stage of buffer, is forced out of the fleet regardless of `RET_FLAG`.
Without this term, a plant built early could operate for the whole horizon simply because
it remains economic. The one-stage buffer handles life values that do not divide evenly
into stage lengths (a 25-year life in 10-year stages forces retirement in the third stage
of service, not the second).

In multi-round mode, retirement already committed in earlier rounds is subtracted from the
cumulative sum re-checked in the current round, so that retirement recorded earlier is not
counted twice when the round boundary shifts.

### Storage energy bookkeeping

$$
u^{ESE}_{i,y} =
\begin{cases}
E^{0}_i + CAP^{\mathrm{chg}}_i\,\Delta^{rec,ESH}_i + CAP^{\mathrm{chg}}_i\,u^{\mathrm{new},ESH}_{i,y} - CAP^{\mathrm{chg}}_i\,u^{\mathrm{new},ESH}_{i,\,y-\Theta_i+1} & y=y_{\text{first}}\\[3pt]
u^{ESE}_{i,y-1} + CAP^{\mathrm{chg}}_i\,u^{\mathrm{new},ESH}_{i,y} - CAP^{\mathrm{chg}}_i\,u^{\mathrm{new},ESH}_{i,\,y-\Theta_i+1} & y>y_{\text{first}}
\end{cases}
\qquad \forall\, i\in\mathcal I^{ES},y
$$

The retirement term is the asset's own original investment-year variable, lagged by
$\Theta_i-1$ stages, $u^{\mathrm{new},ESH}_{i,\,y-\Theta_i+1}$. It is not a separate
retirement decision and activates only once $y$ reaches the new asset's forced-retirement
stage, using the same lagged-vintage rule as forced generation retirement above. In
multi-round mode, the first-stage equation also adds $\Delta^{rec,ESH}_i$, the net new
storage-duration investment carried forward from earlier rounds, in addition to $E^0_i$.

This is the energy-capacity (MWh) counterpart of the capacity bookkeeping, and it bounds
the state of charge in the embedded dispatch. New duration is multiplied by
$CAP^{\mathrm{chg}}_i$, the unit's charge power rating in MW, rather than the discharge
rating $CAP_i$, because $u^{\mathrm{new},ESH}_{i,y}$ is denominated in hours of duration
(hours times charge power equals energy added). Using $CAP_i$ would misstate the energy
addition for storage whose charge and discharge ratings differ. Because the retirement term
is the same investment variable read at a lagged stage, a storage vintage's power and energy
capacity retire together.

### Storage duration and investment link

$$
u^{\mathrm{new},ESH}_{i,y} - \underline{D}_i\, u^{\mathrm{new},G}_{i,y}
\begin{cases}\ \ge 0 & i \in \mathcal I^{ES,STO}\ (\texttt{ES\_STO\_INVEST\_FLAG})\\ \ = 0 & i \in \mathcal I^{ES}\setminus\mathcal I^{ES,STO}\end{cases}
\qquad \forall\, y
$$

For units flagged `ES_STO_INVEST_FLAG`, this is a floor rather than a fixed ratio. A
developer can install more duration per MW than the minimum (for example a 6-hour battery
where 4 hours is required); duration is a separately priced decision (through
$Z^{\mathrm{STO}}$), but it can never fall below $\underline D_i$ (`STOHR_MIN`), a physical
or regulatory minimum for the resource type (for example a duration threshold tied to
storage ITC eligibility). For units without the flag, the equality pins duration to a
fixed multiple of the power capacity built, which is how a fixed-duration battery product
is represented without adding a free duration decision per candidate.

### Storage annual energy throughput (AET)

$$
\sum_{d}\sigma_d\sum_{h,t}\Big(
\eta_i\, chg_{i} + \tfrac{1}{\eta_i} g_{i}
+ \tfrac{0.15}{\eta_i} r^{\mathrm{regUp}}_{i}
+ 0.15\,\eta_i\, r^{\mathrm{regDn}}_{i}\Big)
\ \le\ 2\,AET_i\,\Lambda_{AET}\, u^{G}_{i,y}
\qquad \forall\, i\in\mathcal I^{ES},y
$$

with $\Lambda_{AET}=\tfrac1{365}\sum_d\sigma_d$. Reserve terms appear only if their flag is
on. The constraint is active only for storage technologies whose `AET` value is greater
than 0; a technology left blank or zero has no throughput cap.

This constraint caps a storage unit's annual cycling independently of its power and energy
limits, representing the calendar-aging or warranty throughput limit that battery
chemistry imposes. Without it, hourly state-of-charge balance alone could dispatch a
battery through many more full cycles per year than its rating tolerates by exploiting
hour-to-hour price spreads. Several details of the right-hand side:

- The factor of 2 arises because one full cycle is a charge and a discharge.
- Charge is weighted by $\eta_i$ and discharge and up-regulation by $1/\eta_i$, so a lossy
  round trip consumes slightly more of the annual throughput budget on the charge side than
  the discharge side delivers.
- A fixed 0.15 regulation-deployment factor applies to reserve holding. Regulation is a
  fast, small-magnitude, largely self-canceling service, so counting the full reserved MW
  against throughput every hour would overstate real cycling. The value 0.15 is a
  rule-of-thumb average deployment fraction, not a measured quantity.
- $\Lambda_{AET}$ scales the annual limit for a partial-horizon representative-day set,
  consistent with $\Lambda$ elsewhere in the objective.

The Operation model applies an equivalent throughput limit to the fixed post-expansion
fleet when Operation runs without an embedded expansion solve; see the
[Operation formulation](../Operation/Operation_Formulation.md).

### Transmission expansion (multiplier and per-line cap)

Cumulative multiplier accounting with a per-line headroom cap; non-expandable lines are
fixed to 0:

$$
u^{T}_{k,y} = u^{T}_{k,y-1} + u^{\mathrm{new},T}_{k,y},
\qquad
0 \le u^{T}_{k,y} \le \frac{\overline{F}_k}{F_k}-1
\qquad \forall\, k\in\mathcal K^{EX},y;
\qquad u^{\mathrm{new},T}_{k,y}=u^{T}_{k,y}=0\ \ \forall\, k\notin\mathcal K^{EX}
$$

with $\overline{F}_k$ raised to $F_k(1+\overline{\Delta}^{T})$ when the workbook value is
smaller. A line's transfer limit is relaxed multiplicatively to $F_k(1+u^{T}_{k,y})$ in the
power-flow constraints. Mode-by-mode behavior is described on the
[Transmission Expansion](./GTEP_Transmission_Expansion.md) page.

Relaxing a line's rating multiplicatively rather than by a fixed MW amount resembles a
reconductoring or thermal uprate of an existing corridor, and it keeps the per-line cap
$\overline F_k/F_k-1$ a single dimensionless number comparable across lines with very
different ratings. Because expansion only reinforces an existing corridor, a need for a new
bus-to-bus connection cannot be met by this mechanism: the network file must already
contain a (possibly small-rated) branch between the two buses for expansion to scale.

### Resource supply-curve limit

Cumulative installed capacity of a supply-curve resource in a zone $s$ cannot exceed its
regional limit (existing capacity raises the limit if it already exceeds it):

$$
\sum_{i:\,\mathrm{grp}(i)=g,\ n_i\in s} CAP_i\, u^{G}_{i,y} \ \le\ RS_{g,s}
\qquad \forall\, g\in\mathcal G^{SL},\,s,\,y
$$

For pumped hydro (`psh`) the limit applies to new capacity only
($\sum CAP_i\,u^{\mathrm{new},G}_{i,y}\le RS$). The constraint family is active only when
`Regional_resource_limits_flag == true` (`Simulation Configuration`).

This represents a finite regional resource, such as buildable land in a wind zone,
rooftop area for distributed PV, or suitable pumped-hydro reservoir sites, that several
candidate unit rows can draw from simultaneously. If a workbook splits one resource pool
into multiple candidate projects at different buses (a common way to model a supply curve
with rising cost per tranche), nothing else would stop the model from building every
tranche at once, beyond what the resource supports. Raising $RS_{g,s}$ to existing capacity
where it is already exceeded is a data-consistency safeguard: if a resource assessment
undercounts capacity that is already built, the constraint would otherwise be infeasible
before any new investment. Pumped hydro is limited on new capacity only because existing
PSH assets occupy historical reservoir sites that do not compete with new candidates for
the same site pool.

---

## Reliability and adequacy constraints

### Planning reserve margin (reliability)

$$
\sum_{i:\,n_i\in p} CAP_i\,\kappa_i\, u^{G}_{i,y}
\ \begin{cases}\ \ge\ PD_{p,y}\,(1+PRM) & \texttt{planning\_reserve\_margin\_type == "minimum"}\\ \ \le\ PD_{p,y}\,(1+PRM) & \text{"maximum"}\end{cases}
\qquad \forall\, p\in\mathcal P^{R},y
$$

Here $UCAP_{i}=CAP_i\,\kappa_i$ is the capacity-credit-derated capacity, and $\kappa_i$ may
be a data-region ELCC lookup (resource-adequacy feedback). $PD_{p,y}$ is the coincident
zone peak demand with load growth. The constraint is active only when
`enforce_min_reserve_margin_flag == true` (`Simulation Configuration`).

This is the conventional capacity-adequacy proxy planners use in place of a full
probabilistic reliability study inside the expansion problem. Evaluating a loss-of-load
metric at every candidate solution would be too expensive to embed, so the model requires
derated installed capacity to exceed coincident peak demand by a margin $PRM$ (for example
15%). Without it, nothing in the objective guarantees resource adequacy in the conventional
sense; the model would build only enough capacity to serve expected dispatch economically,
leaving no margin against forecast error, forced outages, or extreme events.

$\kappa_i$ takes two forms. A numeric derate expresses an exogenously assumed capacity value
(for example a flat outage-derated factor for thermal units). A string key into the
data-region lookup is the channel through which a resource-adequacy (ELCC) study feeds back
a technology- and region-specific capacity credit: when
`update_CAPCRED_in_each_round_of_Expansion_Flag` is set, a multi-round workflow re-solves RA
between rounds and updates the lookup so that, for example, a battery's credited capacity
reflects its marginal reliability contribution rather than a flat assumption (see
[Planning horizon and multi-round](./GTEP_Planning_Horizon_and_Multi_Round.md)). The
`"minimum"` type, the common case, is a capacity floor. `"maximum"` inverts it into a
ceiling on credited capacity, useful for scenarios that cap over-building of reserve margin.

!!! info "The margin is computed from local generation only"
    The credited capacity sums only units mapped into planning-reserve zone $p$. The
    constraint has no import or tie-line credit term, so a zone that could rely on a
    neighbor's surplus over a transmission tie receives no planning-reserve credit for it.
    Transmission and planning-reserve accounting are not directly linked.

### System inertia requirement (unit commitment only)

$$
\sum_{i\in\mathcal I^{UC}:\,n_i\in p} \frac{CAP_i}{PF_i}\,H_i\,c_{i,d,h,t,y}
\ \ge\ \mathrm{IN}_{p}
\qquad \forall\, p\in\mathcal P,\,d,h,t,y
$$

where $H_i$ (`Inertia_Constant`) and $PF_i$ (`Power_Factor`) convert committed MW capacity
into an inertia proxy (MW$\cdot$s), and $\mathrm{IN}_p$ (the inertia `target`) is policy
zone $p$'s minimum inertia requirement, defined on the same $\mathcal P$ partition as the
clean-energy, RPS, and emissions constraints. The constraint is applied only in zones with
a positive requirement, and only when both
`enforce_rotational_inertia_constraints_flag == true` (`Planning Design`) and
`Dispatch_Mode_in_EXP == "Unit Commitment"` (`Simulation Configuration`) hold. The
standalone Operation model applies the same constraint under `Dispatch_Mode_in_OP`.

This is a system-strength proxy for rotational mass from synchronous generators and some
storage, most relevant in high-renewable systems where inverter-based resources displace
conventional spinning mass. It exists only under unit commitment because it is written
against the commitment variable $c_{i,d,h,t,y}$: inertia depends on what is spinning, not on
how much energy it produces. The requirement is enforced on the committed capacity of the
cumulative, endogenously built fleet, so a zone whose fleet cannot meet the target is
reported as infeasible rather than left unenforced.

### Other reliability caps (optional)

Three optional, independently switchable caps on energy not served are available:

| Cap | Flag and value (`Simulation Configuration`) | Meaning |
|---|---|---|
| Total energy not served | `Total_ENS_MWh_Cap_Flag`, `Total_ENS_MWh` | Caps the total shortfall energy over the represented horizon (an annual-energy ceiling, similar to an expected-unserved-energy limit) |
| Maximum energy not served | `Max_ENS_MWh_Cap_Flag`, `Max_ENS_MWh` | Caps the single worst hour's shortfall (an hourly ceiling) |
| Approximate ENS hours | `ENS_Hours_Cap_Flag` | Caps a continuous proxy for loss-of-load hours |

All three are upper bounds on scaled $ens$, layered on top of the $VOLL$ pricing in
$Z^{\mathrm{scar}}$, because pricing alone does not control how a given amount of scarcity is
distributed. The approximate ENS-hours cap divides each hour's $ens$ by that hour's system
demand and sums the result. This is an approximation: a true indicator of whether any load
was shed in an hour would need a binary variable per $(n,d,h,t,y)$, which the continuous form
avoids for tractability. A small partial shortfall therefore counts as a small fraction of
an hour, so the proxy generally under-counts an integer loss-of-load-hours metric. A case
that sets none of the flags has no cap beyond the $VOLL$ price.

---

## Policy constraints

### Policies — clean-energy generation target

$$
\sum_{i\in p} g^{\mathrm{clean}}_{i,d,h,t,y} + \varsigma^{CEG}_{p,y}
\ \ge\ CET_{p,y}\ \sum_{i\in p} g_{i,d,h,t,y}
\qquad \forall\, p,y\ \ (Y(y)\ge \text{start year})
$$

The sums run over $(d,h,t)$ in the annual variant, or over $(h,t)$ within each day group in
the day-group variant. Storage generation counts toward the denominator only when the unit
is flagged clean.

This models a clean share of actual dispatched energy (for example a state Clean Energy
Standard). It differs from the RPS proxy below because it is measured against the dispatch
$g$ rather than an installed-capacity shape, so it responds to curtailment, economic
dispatch, and transmission congestion as a generation-based standard would. Storage counts
toward the clean numerator, and toward the denominator when it is a net consumer, only if
its `Clean_Energy_Flag` is set. Without that restriction, a battery charging from a fossil
grid and discharging later would appear clean by existing; the flag limits the credit to
storage paired with, or dedicated to, a qualifying clean resource.

The annual variant lets a zone bank clean output from a windy representative day against a
still one, averaging compliance over the year. The day-group variant enforces the ratio
within each representative day, a stricter choice suited to policies with a compliance
window shorter than a year. Unlike the RPS slack, $\varsigma^{CEG}$ always exists once this
constraint is switched on, so the target is always soft; the penalty price and the start-year
gate are the only levers on how binding it is.

### Policies — renewable portfolio standard (RPS)

The constraint uses a capacity-times-annual-shape energy proxy on the installed fleet rather
than hourly dispatch:

$$
\sum_{i:\,n_i\in p} \mathrm{AF}^{\mathrm{ann}}_{g(i),y}\,CAP_i\, u^{G}_{i,y}
+ \varsigma^{RPS}_{p,y}
\ \ge\ RPS_{p,y}\, AD_{p,y}
\qquad \forall\, p,y
$$

where $\mathrm{AF}^{\mathrm{ann}}_{g,y}$ is the technology's annual, zone-aggregated
capacity-factor shape (one value per policy zone, technology, and year, for example the
wind, solar, or hydro shape) and $AD_{p,y}$ is the zone annual demand with growth. This is
a different quantity from the availability factor $AF_{i,d,h,y}$ on the
[Operation formulation page](../Operation/Operation_Formulation.md), which is a per-unit,
per-hour availability drawn from the representative-day chronology. The two are never used
interchangeably. The production tax credit term uses the same aggregated shape, keyed by bus
rather than policy zone, for the estimated generation of new assets. The slack
$\varsigma^{RPS}$ is fixed to 0 unless `Allow_Alternative_RPS_Compliance_Flag` is set.

Real RPS compliance in most US states is measured against actual annual generation and
renewable energy certificate purchases, which would require summing $g$ over the year and
load-serving-entity-level accounting that the model does not otherwise track. The
constraint instead approximates a qualifying unit's expected annual output as its
technology's annual capacity-factor shape times installed nameplate, a proxy that does not
depend on realized dispatch.

!!! warning "RPS compliance is decoupled from actual dispatch"
    The constraint is driven by $u^{G}_{i,y}$ and a fixed annual shape, not by $g$. A wind
    or solar unit that is heavily curtailed in the embedded dispatch (for example because of
    congestion or oversupply) still counts its full shape-based estimate toward RPS
    compliance. Dispatch and RPS accounting read the same installed-capacity variable but
    never the same energy variable. This is a deliberate tractability simplification, but a
    case can appear RPS-compliant on paper while the curtailed energy actually delivered
    would not comply.

With the slack fixed to 0 by default, an unreachable RPS target appears as solver
infeasibility rather than being absorbed silently. `Allow_Alternative_RPS_Compliance_Flag`
opts in to a priced penalty, comparable to alternative compliance payments.

### Policies — emission reduction

$$
\sum_{i\in p}\sum_{d,h,t} \sigma_d\,\tfrac{EF_i}{1000}\, g_{i,d,h,t,y}
\ \le\ E^{\mathrm{ref}}_{p}\,\big(1 - \delta_{p,y}\big)
\qquad \forall\, p,y\ \ (Y(y)\ge \text{start year})
$$

$E^{\mathrm{ref}}_p$ is the reference emission level (million tonnes) and $\delta_{p,y}$ is
the required fractional reduction (the carbon-emission reduction `target`). The constraint
is active when `Carbon_Emission_Reduction_Target_Flag == true`.

Each policy zone $p$ is compared against its own reference level and reduction trajectory
rather than a single system-wide cap. This lets the model represent overlapping
jurisdictions with independent climate targets (different states, or a multi-state
cap-and-trade region), each with its own baseline year and glide path. The start-year gate
lets a policy take effect partway through the horizon, matching a statute's effective date.
Unlike the RPS and clean-energy constraints, there is no policy-specific slack: an
unreachable emissions target appears as solver infeasibility, which suits a hard regulatory
cap but means the case should be checked for feasibility, or the target relaxed, before
concluding that it cannot be met economically.

!!! warning "Check emission units"
    $EF_i$ (`Emission_CO2`) is a per-MWh rate. Dividing by 1000 and summing over MW-hours
    converts the left side to million tonnes, and the reference level is read from the
    workbook in million tons and brought onto the same numeric scale internally. A workbook
    that enters `Emission_CO2` or the reference level in a different unit convention
    produces a wrong absolute target without any error. Check absolute magnitudes against the
    workbook's documented units for any new region or dataset.

### Policies — raw-material limit

$$
\sum_{i\in\mathcal I^{RM}_m} \alpha_{m,i}\, CAP_i\, u^{\mathrm{new},G}_{i,y}
\ \le\ M^{+}_{m,y}
\qquad \forall\, m,y
$$

Capacity $CAP_i$ enters in physical MW here (converted from the internal per-unit scale).
The constraint is enabled by `enforce_material_constraints_flag` (`Planning Design`) and is
off by default.

This represents an upstream supply-chain or critical-mineral limit, for example annual
availability of lithium, enriched uranium fuel assemblies, or biomass feedstock, that gates
how much new capacity of a material-intensive technology can be built in a year,
independent of any dollar-cost bound. It constrains only $u^{\mathrm{new},G}_{i,y}$, not the
existing fleet or its dispatch: $\alpha_{m,i}$ is interpreted as a construction-time
bill of materials (steel, cobalt, and so on to build the plant), not an ongoing fuel-supply
limit on operation. A running fuel-availability limit would need a separate dispatch-level
constraint, which is not provided. The constraint is relevant mainly to studies probing
supply-chain bottlenecks.

---

## User-defined external constraints

Each row of the `External Constraints` workbook sheet with `Model == "Expansion"` and
`Apply_Flag == true` adds one constraint on a single (region, `UNITGROUP`, planning-year)
selection. The row's `Parameter` column selects one of three types.

**Econ Retirement** fixes the unit's retirement at the target year to the planned-retirement
baseline plus a user-specified increment:

$$
CAP_i\, u^{\mathrm{ret},G}_{i,y} = U^{\mathrm{ret,plan}}_{i,y} + \Delta^{ret}
\qquad i:\ \mathrm{grp}(i)=\text{Resource Unit Group},\ n_i\in\text{Region ID},\ y=\text{Planning Year}
$$

**Investment** fixes new investment at the target year to an exact MW value:

$$
CAP_i\, u^{\mathrm{new},G}_{i,y} = \Delta^{inv}
\qquad i:\ \mathrm{grp}(i)=\text{Resource Unit Group},\ n_i\in\text{Region ID},\ y=\text{Planning Year}
$$

**Total ICAP** bounds the net (new minus retired) capacity change of the selected
group and region at the target year against a user-chosen operator and right-hand side:

$$
\sum_{i:\,\mathrm{grp}(i)=\text{Resource Unit Group},\ n_i\in\text{Region ID}}
CAP_i\big(u^{\mathrm{new},G}_{i,y} - u^{\mathrm{ret},G}_{i,y}\big)
\ \lessgtr\ \Delta^{ICAP}
\qquad y=\text{Planning Year}
$$

where $\lessgtr$ is one of $\le$, $\ge$, or $=$ according to the row's `Operator`
(`"less than"`, `"greater than"`, `"equal to"`). The right-hand side $\Delta^{ICAP}$ depends
on the row's `Value_Unit`:

- `Value_Unit == "%"` with `Compared To == "EXCAP"`: a percentage of existing nameplate
  capacity, $\sum_i EXCAPS_i\times\text{Value}\times0.01 - \sum_i CAP_i\,EXUNITS_i$, summed
  over the same selection.
- `Value_Unit == "MW"`: the row's absolute value, converted to the per-unit basis of the
  variable sum ($\Delta^{ICAP}=\text{Value}/\texttt{per\_unit\_base\_value}$).

!!! note "Supported value units"
    `Total ICAP` rows should use `Value_Unit` of `"%"` or `"MW"`. Any other value leaves the
    right-hand side at 0.

Each row applies only in the single stage whose year matches its `Planning Year`. These
constraints are the workbook's mechanism for a one-off restriction that does not belong to
any general constraint family, for example "region X's coal fleet must retire exactly Y MW
by stage 3 regardless of economics" or "region Y may add no more than Z% of its existing gas
capacity in the final stage". Because each row targets one (region, `UNITGROUP`, year)
tuple independently, a workbook can stack several overrides without interaction, at the
cost of authoring and auditing each by hand. Unlike `SYSTEM_MININVEST` and
`SYSTEM_MAXINVEST` (technology-level, summed over the entire horizon, one min/max pair per
technology), these rows are year-specific and allow an explicit choice of comparison
operator.

---

## Embedded operational sub-problem

For every stage $y$ and representative day group $d$, the expansion model builds a full
chronological dispatch over the hours $\mathcal H$ and sub-hour intervals $\mathcal T$ of
that day, using the same formulation as the standalone Operation model. Its cost enters the
expansion objective as $Z^{\mathrm{op}}$ and $Z^{\mathrm{scar}}$, weighted by
$\chi_{d,y,f}=\sigma_d\,\omega(Y(y)+f-1)/|\mathcal T|$ and summed over the calendar years of
the stage.

The dispatch-core equations are documented on the
[Operation formulation page](../Operation/Operation_Formulation.md) and not repeated here:

- nodal and system power balance, and the optional flat transmission-and-distribution
  demand gross-up $D(1+\rho)$;
- the three power-flow modes: Network_Flow (transport), PTDF (shift-factor DC), and B-θ
  DC-OPF (bus angles, DC-tie angle exclusion, per-island slack);
- operating-reserve balances (regulation, spinning, flexibility, non-spinning), each with its
  on/off product flag;
- unit commitment (commit, start-up, shut-down), thermal minimum and maximum dispatch, and
  inter-temporal ramping;
- must-run, fixed-profile renewable, and energy-budget scheduling;
- storage power, commitment, and state-of-charge balance;
- large flexible load and hybrid resources.

The expansion-specific coupling into that core is that each unit's available capacity is the
cumulative fleet $u^{G}_{i,y}$ (and storage energy $u^{ESE}_{i,y}$), and each line's
transfer limit is $F_k(1+u^{T}_{k,y})$. The operational constraints read the investment
variables rather than fixed capacities.

!!! warning "Water-use accounting applies to standalone Operation only"
    Cooling-water availability limits (the thermal generation-to-water-use link and its
    per-day-group and per-segment limits) are enforced in the standalone Operation model.
    They have no effect on the embedded expansion dispatch, so a plant that would be
    water-constrained in a standalone Operation re-solve can still dispatch without a water
    limit inside the expansion problem.

Embedding the chronological dispatch inside the expansion problem, rather than solving it
as a separate sub-problem coordinated by a decomposition scheme, captures the two-way
coupling between investment and operation in a single solve: each candidate plan is
evaluated against its actual least-cost dispatch, not an approximation. The trade-off is
problem size and solve time.

!!! note "Expansion versus standalone Operation"
    When the Operation model runs downstream of an expansion solve (`expansion_fix = true`),
    it disables `multi_round_solution_process_flag` and `transmission_expansion_flag`, fixes
    the investment variables to the expansion result, and re-solves the dispatch
    chronologically. See [Operation run modes](../Operation/Operation_Execution_and_Run_Modes.md).
    This second solve provides a full chronological 8760-hour (or finer) re-evaluation of the
    expansion plan beyond the representative-day approximation used inside the planning
    problem; the Operation formulation and run-modes pages explain why the distinction
    matters for reported dispatch results.
