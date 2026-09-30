# Operation (Dispatch) — Mathematical Formulation

This page is the mathematical reference for the A-LEAF operational (chronological dispatch) core. It covers the standalone dispatch objective, system and nodal power balance, the three power-flow modes, operating-reserve balances, unit commitment, inter-temporal ramping, must-run units, renewable scheduling, storage, hydro, and large flexible load and hybrid resources. The same formulation is used by both the standalone Operation model and the representative-day dispatch embedded in the [GTEP expansion model](../GTEP/GTEP_Formulation.md), which links here for the entire dispatch core rather than repeating it.

The Operation model solves a chronological least-cost dispatch over a fixed system. It takes the installed fleet, storage energy and network as given. They are fixed to an expansion solution when the model runs downstream of GTEP (`expansion_fix = true`), or pinned to the data-specified existing fleet when no prior expansion result is supplied (`expansion_fix = false`; see the closing section). The model minimizes fuel, variable O&M, reserve-provision, carbon, start-up, scarcity and flexible-load-value costs subject to balance, flow, reserve, commitment, ramping, storage and flexible-load constraints.

Several modeling choices are worth noting up front.

- **Transmission losses** are a flat demand gross-up $D(1+\rho)$ applied uniformly at every bus. There is no per-line loss term on branch flows.
- **The B-θ DC-OPF mode** uses one reference bus per synchronous island and excludes asynchronous DC ties from the angle equation. Their transfer limits still apply.
- **Per-product reserve flags** switch entire families of variables, requirements and costs on or off. A disabled product leaves no variable, requirement or cost term in the model.
- **Storage state of charge** carries efficiency-weighted charge and discharge and several initialization options.
- **The no-load cost** in the objective is charged to the commitment variable, not to generation.

---

## Notation

All sets, parameters and variables are defined once on the [GTEP formulation page](../GTEP/GTEP_Formulation.md#notation-conventions). This page reuses those symbols and defines only the operational-only additions below. The operational time axis is the tuple $(d,h,t,y)$: representative day $d$, hour $h$, sub-hour interval $t$ and planning stage $y$, with quantifiers stated per equation. In standalone Operation there is a single stage $y = \mathcal{Y}[1]$. In the GTEP-embedded dispatch the same equations apply for every stage $y$.

### Additional notation (operational-only)

| Symbol | Meaning |
|---|---|
| $\theta_{n,d,h,t,y}$ | Bus voltage angle (B-θ DC-OPF) |
| $p^{\mathrm{inj}}_{n,d,h,t,y}$ | Net nodal injection (PTDF mode) |
| $dem_{n,d,h,t,y}$ | Delivered (served) nodal load (PTDF mode) |
| $PTDF_{k,n}$ | Power-transfer distribution factor (branch $k$, bus $n$) |
| $b_k = 1/x_k$ | Branch series susceptance, with $x_k=$ `br_x_pu` |
| $\mathcal{N}^{\mathrm{ref}}$ | Per-island reference buses (angle fixed to 0) |
| $\rho$ | Flat transmission and distribution loss fraction ($=$ `transmission_loss_percent_value` $\times 0.01$, the setting being in percent) |
| $MAXR_i$ | 5-minute ramp limit (fraction of $CAP_i$) |
| $RR_i=\min(1,60\,\texttt{Ramp}_i)$ | Hourly ramp fraction |
| $\kappa^{\mathrm{c}}$ | Contingency-reserve minimum duration (h), `contingency_reserve_min_duration_value` |
| $\mathcal{L}$ | Large flexible loads (index $l$) |
| $lfl_{l,d,h,t,y}$ | Served flexible-load withdrawal (MW) |
| $lfl^{\mathrm{seg}}_{l,s,d,h,t,y}$ | DR segment $s$ served load (MW) |
| $lfl^{\mathrm{ind}}_{l,s,d,h,t,y}$ | DR segment on/off indicator (optional binary) |
| $lfl^{\mathrm{DR}}_{l,d,h,t,y}$ | Demand-response (curtailed) load (MW) |
| $g^{G\text{-}LFL}_{l},g^{G\text{-}Grid}_{l},g^{G\text{-}ES}_{l}$ | On-site generation routed to the load, the grid and the on-site storage (MW) |
| $g^{ES\text{-}LFL}_{l},g^{ES\text{-}Grid}_{l}$ | On-site storage discharge to the load and to the grid (MW) |
| $chg^{Grid\text{-}ES}_{l}$ | On-site storage charge from the grid (MW) |
| $soc^{L}_{l,d,h,t,y}$ | On-site storage state of charge (MWh) |
| $g^{G\text{-}ES}_{i}$ | Hybrid main-generator power routed to the co-located storage (MW) |
| $CAP_l,\ INTERCON_l$ | Flexible-load flat-load cap and interconnection limit (MW) (`INTERCON_LIM`) |
| $LFL^{\mathrm{seg}}_{l,s}=\texttt{Pct\_MW}_{l,s}CAP_l$ | Segment size (MW) |
| $C^{DR}_{l,s}$ | DR segment price ($/MWh), `Price_s` |
| $\overline{DR}_l$ | Daily DR energy cap (MWh), `Daily_DR_Limit_MWh` |
| $\eta^{L}_l=\sqrt{\texttt{BATEFF}}$ | On-site storage one-way efficiency |

Several abbreviations recur as category labels rather than indexed symbols: **VRE** (variable renewable energy, wind and solar), **ES** (energy storage), **UC** (unit commitment), **DR** (demand response), **LFL** (large flexible load, the set $\mathcal L$), **AGC** (automatic generation control, the second-to-second dispatch that regulation reserve supports), **DER** (distributed energy resource, generation or storage behind a load's or hybrid plant's meter) and **MIP** (mixed-integer program, the model class once commitment or indicator binaries are added to an LP dispatch).

Each operating-reserve product is controlled by its own `*_reserve_flag`. When a flag is false, the product's variables, requirement rows and costs are absent from the model. Up-reserves can be carried only by units that are `Dispatchable`.

---

## Standalone dispatch objective

The standalone Operation model minimizes the representative-day-weighted dispatch cost for the single stage $y$. The per-day weight is $w_d=\sigma_d/|\mathcal T|$, where $\sigma_d$ is the number of calendar days the representative day stands for (`NumDays`) and $|\mathcal T|$ is the number of sub-hourly intervals per hour:

$$
\begin{aligned}
\min\ Z^{\mathrm{OP}} =\;
&\sum_{i}\sum_{d}w_d\sum_{h,t}\Big[
   MC_{i,y,d}\,g_{i} + CTAX\cdot EF_i\,g_{i}
 - \mathbb{1}_{\mathrm{PTC}}\,ptc_{i}\,g_{i}\Big] \\[2pt]
+\ &\sum_{i}\sum_{d}w_d\sum_{h,t}\Big[
   C^{\mathrm{reg}}_i(r^{\mathrm{regUp}}_{i}+r^{\mathrm{regDn}}_{i})
 + C^{\mathrm{spin}}_i r^{\mathrm{spin}}_{i}
 + C^{\mathrm{nspin}}_i r^{\mathrm{nspin}}_{i}
 + C^{\mathrm{flex}}_i(r^{\mathrm{flexUp}}_{i}+r^{\mathrm{flexDn}}_{i})\Big] \\[2pt]
+\ &\sum_{i\in\mathcal I^{UC}}\sum_{d}w_d\sum_{h}\Big[
   C^{SU}_i\,su_{i,d,h,1,y} + C^{NL}_i\,c_{i,d,h,1,y}\Big] \\[2pt]
+\ &\sum_{d}w_d\sum_{h,t}\Big[\sum_{n}VOLL\,ens_{n}
 + \sum_{z}\big(\pi^{\mathrm{reg}}(rns^{\mathrm{regUp}}_{z}+rns^{\mathrm{regDn}}_{z})
 + \pi^{\mathrm{S}}rns^{\mathrm{cont}}_{z}
 + \pi^{\mathrm{NS}}rns^{\mathrm{nonspin}}_{z}
 + \pi^{\mathrm{FLEX}}(rns^{\mathrm{flexUp}}_{z}+rns^{\mathrm{flexDn}}_{z})\big)\Big] \\[2pt]
-\ &\sum_{l}\sum_{s\ge 2}\sum_{d}w_d\,C^{DR}_{l,s}\sum_{h,t} lfl^{\mathrm{seg}}_{l,s}
\ +\ \sum_{d}w_d\,\rho^{CEG}\sum_{p}\varsigma^{CEG}_{p,d,y}
\ +\ \epsilon\!\!\sum_{\text{(sto UC, hybrid chg)}}\!\!(\cdot)
\end{aligned}
$$

Each term is weighted by $w_d$, so a representative day's cost is scaled up to the $\sigma_d$ calendar days it represents, and divided by $|\mathcal T|$ so that an hourly cost rate ($/MWh) is charged correctly to each sub-hourly interval without counting the hour more than once. Economically, the objective serves load at least cost with every real operating cost priced in (fuel and variable O&M, carbon, ancillary-service provision, commitment and involuntary shortfalls), plus two revenue-like terms (production tax credit and demand-response value) that make some actions cheaper than their raw cost. Because reserves, start-up and scarcity all compete within the same objective, relative price levels as well as physical limits determine outcomes, for example whether a unit shuts down overnight or stays at $PMIN$ to avoid a start-up cost.

The terms are as follows.

- **Generation and carbon.** $MC_{i,y,d}\,g$ is the fuel and variable O&M marginal cost, the main driver of dispatch order. It is read per stage $y$ and representative day $d$, so it can vary by day (for example with seasonal fuel-price shapes). $CTAX\cdot EF_i\,g$ adds a carbon price proportional to the unit's emission rate $EF_i$ (`Emission_CO2`, in tCO$_2$/MWh). With `CTAX` equal to 0 (the usual case when carbon policy is modeled as a cap in GTEP) this term vanishes and dispatch follows fuel-cost merit order. A unit with $EF_i=0$ (VRE, nuclear, storage) is unaffected by `CTAX`.
- **Production tax credit** (subtracted; applies only when `PTC_Flag` is on and the unit's `PTC Flag` is set). $ptc$ is the `PTC` value for the year $\min(2050,Y(y))$, converted from cents/kWh to $/MWh (multiplied by 10). It is a per-MWh subsidy that can make a unit's effective marginal cost negative, which dispatches PTC-eligible wind and solar ahead of zero-fuel-cost technologies without a credit. The clamp at 2050 holds the credit at its last tabulated year for later stages.
- **Reserve-provision costs.** Each product is included only when its `*_reserve_flag` is on, and up-products only for units eligible to carry them. Pricing reserve provision prevents the model from loading every eligible unit with reserves at no cost. When `reserve_cost_type_flag` is `percentage`, the four reserve unit costs are read as fractions of $MC_{i,y,d}$. For example, `reg_cost = 0.1` prices regulation-up and regulation-down at 10% of that unit's energy marginal cost that day, so reserve prices follow fuel prices without a separate $/MW table. The floor `min_regulation_cost_value` keeps regulation from being free for near-zero-cost units such as hydro and nuclear, which would otherwise create degenerate ties among providers. The non-spinning cost applies only when `Dispatch_Mode_in_OP` is `Unit Commitment`. Under economic dispatch, spinning and non-spinning merge into a single contingency product priced at `spin_cost`.
- **Start-up and no-load** (unit-commitment units only). $C^{SU}_i\,su$ charges the per-MW start-up cost once per off-to-on transition, which discourages cycling a unit within a day for small energy-cost savings and keeps costly-to-cycle units, such as combined-cycle gas, committed through a shallow midday dip. The no-load term charges the no-load cost $C^{NL}_i$ (`NLC`) to the **commitment** variable $c$, not to generation, in every interval the unit is committed. It represents the fixed hourly cost of keeping a thermal unit synchronized (auxiliary load, minimum fuel burn, staffing) and gives the model an incentive to decommit a unit instead of idling it at $PMIN$ when prices are low. The objective has no shut-down cost; only start-up is priced. The embedded operation cost in the [GTEP formulation](../GTEP/GTEP_Formulation.md#embedded-operation-cost-zmathrmop) uses the same convention.
- **Scarcity.** $VOLL\cdot ens$ is the value-of-lost-load penalty on unserved energy. It is set far above any credible marginal generation cost, so the model sheds load only when dispatch, reserves, imports, storage and demand response are exhausted. It keeps the balance equation feasible under any conditions. Zonal reserve scarcity plays the same role for reserve requirements: a shortfall is allowed but penalized, and it appears in the reported reserve-scarcity costs and quantities as a reliability signal. Each reserve scarcity product is priced at its own penalty: regulation at $\pi^{\mathrm{reg}}$ (`RegRSP`), contingency at $\pi^{\mathrm S}$ (`SRSP`), non-spinning at $\pi^{\mathrm{NS}}$ (`NSRSP`) and flexibility up and down at $\pi^{\mathrm{FLEX}}$ (`FLEXRSP`).
- **Large flexible load demand-response value.** DR segments $s\ge 2$ enter with a **negative** coefficient $-w_d\,C^{DR}_{l,s}$. Served flexible load is a benefit, mirroring a demand bid curve. Raising a segment's `Price_s` therefore makes the model more willing to serve that increment, the opposite of how an ordinary cost coefficient behaves.
- **Clean-energy slack penalty** (when `Clean_Energy_Generation_Target_OP_Flag` is on). It penalizes the shortfall $\varsigma^{CEG}_{p,d,y}$ against a clean-energy generation target, in the same soft-constraint pattern as $ens$ and $rns$. The target constraint (described on the [GTEP page](../GTEP/GTEP_Formulation.md)) stays feasible under any dispatch, and the penalty discourages but does not forbid missing it.
- **Tie-breaking terms** ($\epsilon=10^{-6}$). Storage commitment $c^{STO}$, hybrid grid charging $chg$ and hybrid large-load grid charging carry a negligible per-unit penalty. These variables are otherwise unconstrained by cost and can take several values without changing the true dispatch cost (for example, an idle storage unit can have $c^{STO}\in\{0,1\}$). The penalty, several orders of magnitude below the smallest real per-unit marginal cost, selects one solution deterministically without measurably changing the optimal cost, which improves reproducibility.

---

## System and nodal power balance

Every bus $n$ must balance injections, withdrawals, branch flows, unserved energy and any co-located flexible load in every interval. This is Kirchhoff's current law applied to the dispatch: local generation, imports and shortfall must equal local load, storage charging and exports. It is the equation that couples generation, storage, transmission and load into one system problem. Bus demand is the **grossed-up** load

$$
D_{n,d,h,t,y}=\big(1+\rho\big)\,\widehat D_{n,d,h,t,y},
$$

where $\widehat D$ is the growth-scaled coincident bus load and $\rho$ is the flat transmission and distribution loss fraction, applied when `enforce_transmission_loss_flag` is true. A model bus is often an aggregation of several data regions (for example several counties folded into one balancing-area node). The bus load sums each region's hourly load shape times its base-year MW times its own `load_growth_by_region` factor, so regions behind the same bus can grow at different annual rates. $\rho$ is a **single scalar** (`transmission_loss_percent_value`, entered in percent: a value of `3` gives a 3% gross-up) used at every bus and every hour. It is a simplification, not a per-line loss calculation. There is no per-line loss term, so branch flows are lossless in all three power-flow modes: a MW that leaves one end of a line arrives in full at the other. With `enforce_transmission_loss_flag` off, $\rho=0$ and $D=\widehat D$.

### Network_Flow and B-θ nodal balance

For the transport (`Network_Flow`) and DC-OPF (`B-theta`) modes, the balance is written directly on branch flows:

$$
\sum_{k:\,t_k=n} f_{k} - \sum_{k:\,f_k=n} f_{k}
\;+\;\sum_{i\in\mathcal I(n)}\!\big(g_{i} - \mathbb{1}_{i\in\mathcal I^{ES}}\,chg_{i}\big)
\;+\; ens_{n}
\;-\;\sum_{l\in\mathcal L(n)} lfl_{l}
\;-\; D_{n}
\;=\;0
\qquad \forall\, n,d,h,t,y
$$

with $0 \le ens_{n} \le D_{n}$. The first two sums are the net branch injection into bus $n$: flows arriving at $n$ (where $n$ is the branch's "to" bus $t_k$) count positively, and flows leaving $n$ (the "from" bus $f_k$) count negatively. Storage subtracts its charging $chg_i$, and discharge appears through $g_i$ like any other generator. Any large flexible load at $n$ subtracts its served withdrawal $lfl_l$; it is tracked separately from $D_n$ because it can be partly supplied by on-site generation or storage before drawing on the grid (see the large flexible load section).

The unserved-energy variable $ens_n$ keeps the equation feasible under any combination of outages, low renewable output and binding transmission limits. Instead of the model becoming infeasible when local supply plus imports cannot cover $D_n$, the gap is absorbed by $ens_n$ at the value of lost load $VOLL$ from the objective. The bound $ens_n\le D_n$ ensures a bus cannot shed more than it consumes. As an example, bus A has 100 MW of cheap generation and 40 MW of load, bus B has no generation and 80 MW of load, and the tie line has a 50 MW limit. Bus A exports 50 MW (limited by the line), leaving bus B short by 80 - 50 = 30 MW, which appears as $ens_B=30$.

### PTDF nodal balance

Under `PTDF` the balance is split across a served-load variable $dem_n$ and a nodal injection $p^{\mathrm{inj}}_n$, because the PTDF flow equation expresses $f_k$ as a linear function of injections. Introducing $p^{\mathrm{inj}}_n$ lets the same injection appear in both the local balance and the PTDF flow equation:

$$
dem_{n} + ens_{n} - \sum_{l\in\mathcal L(n)} lfl_{l} - D_{n} = 0
\qquad\text{and}\qquad
p^{\mathrm{inj}}_{n} - \sum_{i\in\mathcal I(n)}\!\big(g_{i}-\mathbb{1}_{i\in\mathcal I^{ES}}chg_{i}\big) + dem_{n} = 0
\qquad \forall\, n,d,h,t,y
$$

so $p^{\mathrm{inj}}_{n}=\sum_i(g_i-chg_i)-dem_n$: generation net of served load, positive at a net-exporting bus and negative at a net-importing bus. System-wide injections sum to zero, $\sum_n p^{\mathrm{inj}}_{n}=0$, the analogue of total generation equaling total load once transmission losses are excluded.

---

## Power-flow modes

The active mode is set by `power_flow_mode_flag` (`Simulation Setting`), and each mode requires matching network data. This section gives the dispatch physics of each mode. For the expansion-side interaction (the $F_k(1+u^T_{k,y})$ rating multiplier, per-line caps and DC-tie cost treatment) see [Transmission Expansion](../GTEP/GTEP_Transmission_Expansion.md). The modes trade fidelity against build and solve cost. `Network_Flow` is a pure LP transport model (fastest, no loop-flow physics). `PTDF` linearizes power flow around a base case and captures loop flow without angle variables. `B-theta` is a full DC-OPF (most physically faithful, with one angle variable per bus per interval). Changing `power_flow_mode_flag` without supplying the matching network data (PTDF matrices for `PTDF`, branch reactances `br_x_pu` for `B-theta`) either fails or solves a physically meaningless network, so the setting and the data must be changed together.

In standalone Operation, the transfer limits treat the committed expansion increment $u^T_{k,y}$ as a **constant** taken from the expansion result (zero when no expansion is fixed), so Operation dispatches against the transmission capacity GTEP already decided to build and never adds more. In the GTEP-embedded dispatch, the same limits read the expansion **variable** $u^T_{k,y}$, so the expansion problem optimizes transmission investment against the dispatch cost these limits produce.

### 1. Network_Flow (transport model)

A branch is a pipe with a symmetric MW capacity in each direction, with no angle or shift-factor coupling:

$$
-\,F_k\big(1+u^T_{k,y}\big)\ \le\ f_{k}\ \le\ F_k\big(1+u^T_{k,y}\big)
\qquad \forall\, k,d,h,t,y
$$

This is the cheapest mode to build and solve. It suits network data aggregated to a coarse zonal or balancing-area topology, where a physically resolved flow model would be spurious precision. It cannot represent loop flow, and every corridor is dispatched independently up to its own limit.

### 2. PTDF (shift-factor DC approximation)

Each branch flow is a linear combination of nodal injections, with a threshold that drops negligible shift factors:

$$
f_{k}=\sum_{n:\,|PTDF_{k,n}|\ge \tau} PTDF_{k,n}\,p^{\mathrm{inj}}_{n}
\qquad \forall\, k,d,h,t,y,
$$

where $\tau=$ `PTDF_threshold_value`, subject to the same transfer limits $|f_{k}|\le F_k(1+u^T_{k,y})$. $PTDF_{k,n}$ is the MW flow on branch $k$ per MW injected at bus $n$ and withdrawn at the reference, a linearization of power-flow sensitivity around a base operating point. It is supplied as network data, not derived from `br_x_pu` inside the model. This lets `PTDF` capture loop flow (an injection at one bus can shift flow onto a distant branch) without an angle variable per bus. The threshold $\tau$ (for example `0.01`) exists for model size: a full PTDF matrix is nearly dense, and dropping small entries keeps the constraint sparse. A very small $\tau$ produces a large, slow model, while a $\tau$ that is too large drops real, if minor, loop-flow paths.

### 3. B-θ (full DC-OPF)

Branch flow is the angle difference across the series susceptance $b_k=1/x_k$, where $x_k=$ `br_x_pu`:

$$
f_{k}=b_k\big(\theta_{f_k,d,h,t,y}-\theta_{t_k,d,h,t,y}\big)
\qquad \forall\, k\notin\mathcal K^{DC},\,d,h,t,y,
$$

again with $|f_{k}|\le F_k(1+u^T_{k,y})$. Unlike `PTDF`, the bus angle $\theta_n$ is a decision variable solved jointly with dispatch, so the flow pattern is derived inside the model from the branch susceptances. This makes `B-theta` the most physically transparent mode, but also the most expensive and the most sensitive to data quality, since a wrong or missing `br_x_pu` directly distorts flows. The flow equation is scaled by $s_b=\sqrt{|b_k|}$ (that is, $\tfrac{1}{s_b}f_k-\tfrac{b_k}{s_b}(\theta_{f}-\theta_{t})=0$) to keep susceptances of very different magnitudes numerically well conditioned. The scaling does not change the solution.

!!! info "DC ties are excluded from the angle equation; per-island reference bus"
    Two behaviors differ from a textbook DC-OPF.

    - **DC-tie angle exclusion.** Asynchronous DC ties ($k\in\mathcal K^{DC}$, `dc_line` true) do not couple bus angles. The angle equation is not written for them, but their transfer limits $|f_k|\le F_k(1+u^T_{k,y})$ still apply. A DC tie therefore behaves as an angle-decoupled controllable transfer, like an HVDC converter station between systems that are not in synchronous lock-step.
    - **Per-island reference bus.** One reference bus per synchronous (AC-connected) island has its angle fixed to $\theta=0$. The flow equation constrains only angle differences, so without a reference the angles of each island would be undetermined. Islands are identified from AC branches only; DC ties are excluded, so a system connected to the rest of the network only by a DC tie forms its own island with its own reference bus. The set of reference buses is $\mathcal N^{\mathrm{ref}}$, with $\theta_n=0$ for each $n\in\mathcal N^{\mathrm{ref}}$.

---

## Operating-reserve balance

Each reserve zone $z$ must meet its requirement for every enabled product, with a zonal scarcity (shortfall) variable $rns$ capped at the requirement. Reserves are headroom the system holds in addition to energy dispatch to survive routine variability (regulation), the sudden trip of a generator or line (spinning and non-spinning contingency), and net-load ramps from load and VRE forecast error (flexibility). None of this headroom is delivered as energy in the base case, but it must be physically available on short notice. Without a reserve requirement the model would dispatch every unit to the edge of its capacity, which is unrealistic and would understate the fleet needed to ride through contingencies.

The requirement $R^{\bullet}_{z}$ is summed over the balancing-area regions mapped to zone $z$, using the `reg_up`, `reg_down`, `spin`, `nspin`, `flex_up` and `flex_down` columns of the representative-day data. Reserve zones can span several balancing areas when reserves are shared, or be defined per balancing area when they are not; this is a data choice, not a formulation choice.

In **individual** reserve mode, up-reserve sums skip non-dispatchable units, because a fixed-profile VRE plant cannot be asked to raise output above its available output. In **aggregated** mode the requirement is met by reserve-group proxy variables that represent the zone's fleet in bulk, a coarser but much smaller formulation for large systems. Aggregated mode is available only in the GTEP-embedded dispatch. Standalone Operation requires `operating_reserve_modeling_option` to be `individual` and stops with an error otherwise (see [Standalone Operation vs. GTEP-embedded dispatch](#standalone-operation-vs-gtep-embedded-dispatch)).

$$
\sum_{i\in z} r^{\bullet}_{i,d,h,t,y} + rns^{\bullet}_{z,d,h,t,y}\ \ge\ R^{\bullet}_{z,d,h,t,y},
\qquad 0\le rns^{\bullet}_{z}\le R^{\bullet}_{z}
\qquad \forall\, z,d,h,t,y
$$

for each enabled product $\bullet$. As with the nodal energy balance, the scarcity variable $rns^{\bullet}_z$ keeps this a **soft** constraint. When the fleet cannot provide the required headroom, for example when several large units are already at $PMAX$, the model reports a priced reserve shortfall instead of becoming infeasible. The bound $rns^{\bullet}_z\le R^{\bullet}_z$ limits the shortfall to at most the full requirement.

| Product $\bullet$ | Reserve variable | Scarcity variable | Requirement |
|---|---|---|---|
| Regulation up | $r^{\mathrm{regUp}}$ | $rns^{\mathrm{regUp}}$ | `reg_up` |
| Regulation down | $r^{\mathrm{regDn}}$ | $rns^{\mathrm{regDn}}$ | `reg_down` |
| Spinning (unit commitment) | $r^{\mathrm{spin}}$ | $rns^{\mathrm{cont}}$ | `spin` |
| Non-spinning (unit commitment) | $r^{\mathrm{nspin}}$ | $rns^{\mathrm{nonspin}}$ | `nspin` |
| Contingency (economic dispatch) | $r^{\mathrm{spin}}$ | $rns^{\mathrm{cont}}$ | `spin` + `nspin` |
| Flexibility up | $r^{\mathrm{flexUp}}$ | $rns^{\mathrm{flexUp}}$ | `flex_up` |
| Flexibility down | $r^{\mathrm{flexDn}}$ | $rns^{\mathrm{flexDn}}$ | `flex_down` |

Regulation (up and down) covers second-to-second and minute-to-minute automatic generation control around the scheduled dispatch point. Spinning and non-spinning contingency reserve cover the loss of the largest in-service unit or tie: spinning reserve must already be synchronized and respond within minutes, and non-spinning reserve can start from off within the contingency window. Flexibility (up and down) covers slower, larger swings from net-load forecast error, such as a large solar ramp-down at sunset, which regulation is too small and contingency too event-driven to cover.

Under `Dispatch_Mode_in_OP` set to `Unit Commitment`, spinning and non-spinning products are enforced separately, because only a commitment model can distinguish units that are already spinning from units that could start in time. Under economic dispatch, a single combined **contingency** requirement (`spin` + `nspin`) is used instead. A product whose `*_reserve_flag` (see the [Simulation Configuration Reference](../../configuration/Simulation_Configuration_Reference.md#per-product-operating-reserve-flags)) is off creates no requirement, no provision variable and no scarcity variable, so turning a product off is equivalent to it never having been part of the formulation.
---

## Unit commitment

For unit-commitment units ($i\in\mathcal I^{UC}$, active when `Dispatch_Mode_in_OP` is `Unit Commitment`), the integer commitment $c_i$, start-up $su_i$ and shut-down logic turn the dispatch problem from a continuous LP into a mixed-integer problem. A thermal unit is either synchronized (producing between $PMIN$ and $PMAX$ and incurring an hourly no-load cost) or off (no output and no no-load cost, but a start-up cost the next time it is needed). Without commitment logic, an economic-dispatch LP can run a large coal unit at a few percent of $PMAX$ overnight whenever its marginal cost is competitive, which understates no-load and cycling costs.

$$
\begin{aligned}
c_{i,d,h,t,y} &\le u^{G}_{i,y} && \text{(commit} \le \text{installed units)} \\
su_{i,d,h,t,y} &\le c_{i,d,h,t,y} && \text{(start-up} \le \text{commit)} \\
su_{i,d,h,t,y} &\ \ge\ c_{i,d,h,t,y} - c_{i,\,\mathrm{prev}} && \text{(start-up turns on)} \\
su_{i,d,h,t,y} &\ \le\ u^{G}_{i,y} - c_{i,\,\mathrm{prev}} && \text{(no start-up if already committed)}
\end{aligned}
\qquad \forall\, i\in\mathcal I^{UC},\,d,h,t,y
$$

The first row states that $c_i$ cannot exceed the number of installed units $u^G_{i,y}$. The second states that the start-up indicator can be on only in intervals in which the unit is committed. The third forces $su_i\ge1$ whenever the unit transitions from off to on ($c_i=1$, $c_{i,\mathrm{prev}}=0$), so the start-up cost applies once per start and cannot be avoided. The fourth prevents the model from being forced into a start-up while the unit remains committed: when $c_{i,\mathrm{prev}}=1$, the right-hand side is at most $u^G_{i,y}-1$. For a single unit ($u^G=1$), $c=0$ in hour 1 and $c=1$ in hour 2 forces $su=1$ in hour 2. If $c=1$ in both hours, the third row requires only $su\ge0$ and, because $C^{SU}>0$, the model leaves $su=0$.

The previous state $c_{i,\mathrm{prev}}$ refers to the previous interval, wrapping at the first hour to the last hour of the prior day. At the first day of a day group it wraps **cyclically** to the last hour of the group's last day. This lets a representative day group stand in for many non-consecutive calendar days without forcing an artificial cold start at the group's first modeled hour.

### System inertia

When `enforce_rotational_inertia_constraints_flag` is true **and** `Dispatch_Mode_in_OP` is `Unit Commitment`, each policy zone $p$ must hold enough committed rotational inertia to meet a zone target. This represents the ability of a power system to ride through the first seconds of a frequency event, which comes from the kinetic energy of synchronously spinning generator mass rather than from MW capacity:

$$
\sum_{i\in p}\frac{CAP_i}{PF_i}\,H_i\,c_{i,d,h,t,y}\ \ge\ INERTIA_p
\qquad \forall\, p,d,h,t,y\ \text{with}\ INERTIA_p>0
$$

where $H_i=$ `Inertia_Constant` and $PF_i=$ `Power_Factor` convert a unit's real-power capacity to its rotating-mass inertia contribution, and $INERTIA_p$ is the zone's inertia target. The sum runs over committed capacity: an uncommitted unit contributes no inertia whatever its nameplate size, which is why the constraint applies only under `Unit Commitment`. A zone with a zero target has no inertia constraint. The requirement is evaluated against the model's own commitment and installed capacity, so in GTEP it tracks the cumulative built fleet.

---

## Unit dispatch and inter-temporal ramping

Two dispatch forms exist for each unit: a **unit-commitment form**, in which the integer commitment $c_i$ gates the minimum and maximum output, and an **economic-dispatch (LP) form**, in which the installed units $u^G_i$ gate the maximum. The form is selected per unit by `Dispatch_Mode_in_OP`. These constraints stop the model from treating a generator as an instantaneously responsive MW source: thermal units cannot run below a minimum stable load $PMIN$, cannot exceed nameplate $PMAX$, and cannot change output faster than their ramp rate allows. Each limit also leaves headroom for the reserve products the unit provides, because reserve is a commitment of extra (or reduced) output that must be deliverable on top of the energy schedule.

### Minimum and maximum generation with reserve headroom

**Unit-commitment form:**

$$
PMIN_i\,CAP_i\,c_{i} + r^{\mathrm{regDn}}_{i}+r^{\mathrm{flexDn}}_{i}
\ \le\ g_{i}
\ \le\ c_{i}\,PMAX_i\,CAP_i - \big(r^{\mathrm{regUp}}_{i}+r^{\mathrm{flexUp}}_{i}+r^{\mathrm{spin}}_{i}\big)
\qquad \forall\, i\in\mathcal I^{UC},\,d,h,t,y
$$

A committed unit ($c_i=1$) must run at or above $PMIN$. Any down-reserve it provides raises this floor, because supplying 20 MW of down-reserve requires the unit to be running at least 20 MW above $PMIN$. Symmetrically, every up-reserve product is subtracted from the $PMAX$ ceiling, because up-reserve requires headroom below $PMAX$. If $c_i=0$, both bounds collapse to $g_i=0$: an uncommitted unit produces nothing and carries no reserve. Up-reserve terms appear only for units eligible to carry up-reserves (`Dispatchable` units) and for enabled reserve products.

**Economic-dispatch (LP) form** ($PMIN$ set to 0, headroom against the installed fleet $u^G_i$; up-reserves only in individual mode):

$$
r^{\mathrm{regDn}}_{i}+r^{\mathrm{flexDn}}_{i}
\ \le\ g_{i}
\ \le\ u^{G}_{i,y}\,PMAX_i\,CAP_i - \big(r^{\mathrm{regUp}}_{i}+r^{\mathrm{flexUp}}_{i}+r^{\mathrm{spin}}_{i}\big)
\qquad \forall\, i\notin\mathcal I^{UC},\,d,h,t,y
$$

Without a commitment integer the model cannot represent a decommitted unit. Every installed unit is treated as available, and $PMIN$ is relaxed to 0 so the LP does not force minimum output on units that would be shut down in a low-price hour. This is a stylized simplification made for solve speed. The model can run a large thermal unit at near-zero output rather than incur a start-up cost it does not model.

Reserve provision is also capped separately by 5-minute and commitment headroom. A unit's 5-minute ramp capability can be tighter or looser than the energy headroom, and a reserve product must be deliverable within its own response window. Regulation reserves are limited by $r^{\mathrm{regUp}},r^{\mathrm{regDn}}\le c_i\,PMAX_i\,CAP_i\,MAXR_i$ (with $u^G_{i,y}$ replacing $c_i$ in the economic-dispatch form). In addition, the combined up-products (regulation-up, flexibility-up and spinning) are limited by $\min(RUL_i,MAXC_i)\,PMAX_i\,CAP_i$, and the combined down-products (regulation-down and flexibility-down) by $\min(RDL_i,MAXC_i)\,PMAX_i\,CAP_i$. Each direction uses its own 10-minute ramp rate ($RUL$ up, $RDL$ down), capped by the maximum-contingency percentage. In the unit-commitment form, non-spinning reserve is also limited by the unit's headroom.

### Inter-temporal hourly ramping

**Unit-commitment form.** The previous-interval committed capacity sets the ramp band, with an allowance for start-up and shut-down hours. Here $RR_i=\min(1,60\,\texttt{Ramp}_i)$ is the fraction of nameplate capacity a unit can move in one hour, capped at 100%:

$$
\begin{aligned}
g_{i} - g_{i,\mathrm{prev}} &\ \le\ c_{i,\mathrm{prev}}\,RR_i\,CAP_i\,PMAX_i + su_{i}\,RR_i\,CAP_i\,PMAX_i \\
g_{i,\mathrm{prev}} - g_{i} &\ \le\ c_{i}\,RR_i\,CAP_i\,PMAX_i + RR_i\,CAP_i\,PMAX_i\,(su_{i}-c_{i}+c_{i,\mathrm{prev}})
\end{aligned}
\qquad \forall\, i\in\mathcal I^{UC}\cap\{\text{THERMAL,NUCLEAR,OTHER}\}
$$

This limits how fast a committed unit's output can change from one interval to the next, on top of the minimum/maximum band: commitment status alone would allow a jump from $PMIN$ to $PMAX$ in one hour, which real turbines and boilers cannot make. The $su_i$ term on the ramp-up side gives an extra allowance in the hour a unit starts, so a unit coming online is not double-penalized by both the start-up logic and a from-zero ramp limit. The shut-down side is symmetric: the term $(su_i-c_i+c_{i,\mathrm{prev}})$ equals $+1$ exactly in the hour a unit shuts down ($c_i=0$, $c_{i,\mathrm{prev}}=1$, $su_i=0$) and grants the equivalent allowance to ramp down to zero.

**Economic-dispatch (LP) form.** The ramp band is the installed-fleet ramp capability:

$$
\big|\,g_{i} - g_{i,\mathrm{prev}}\,\big|\ \le\ u^{G}_{i,y}\,RR_i\,CAP_i\,PMAX_i
\qquad \forall\, i\notin\mathcal I^{UC}\cap\{\text{THERMAL,NUCLEAR,OTHER}\}
$$

Without a commitment status there is no start-up or shut-down interval to treat specially, so output moves by at most $RR_i$ of nameplate per hour in either direction.

In both forms $g_{i,\mathrm{prev}}$ wraps cyclically at day-group boundaries, using the same logic as storage state of charge and commitment continuity, so a unit does not receive an unlimited fresh-start ramp at the first hour of every representative day. When `FIVEMIN` is 1 (sub-hourly resolution, $|\mathcal T|>1$), a sub-hourly ramp band also applies. It limits the change between consecutive sub-hourly intervals (for example 5 minutes), so that a unit cannot use its whole hourly ramp allowance in the first interval of the hour.

---

## Must-run

Must-run units ($i\in\mathcal I^{MR}$, `Must_Run_Flag` true) hold at least their must-run level, clamped to $PMAX_i$, on the installed fleet:

$$
g_{i,d,h,t,y}\ \ge\ u^{G}_{i,y}\,\min(\texttt{Must\_Run\_Level}_i,PMAX_i)\,CAP_i
\qquad \forall\, i\in\mathcal I^{MR},\,d,h,t,y
$$

This is a data-driven floor for units whose real operation is not purely economic, such as combined-heat-and-power plants that must keep running to serve a steam host, or units the case author wants held at a minimum output. Without it, a must-run unit with a high marginal cost could be dispatched to zero on a low-load day. The clamp to $PMAX_i$ prevents an infeasible floor if `Must_Run_Level` (a fraction of capacity) is entered above 1.0. A `Must_Run_Level` of 0 is equivalent to no floor.

---

## Renewable scheduling

Variable renewable and hydro units (`UNIT_CATEGORY` of `VRE` or `HYDRO`) are scheduled either by a **fixed hourly profile** or by an **energy budget**, chosen by the unit's `FUEL_LIMIT`. The two resource classes face different physical limits. A wind or solar plant's output in a given interval is set by weather, an exogenous shape rather than a decision. A reservoir or budget-limited hydro unit can choose when to release stored water within a total-energy limit, which makes it suitable for peak shaving or ramp support.

### Fixed-profile availability

The availability $AF_{i,d,h,y}$ (the technology's time-series shape) caps dispatch on the installed fleet. `Dispatchable` VRE units may also carry up-reserves (individual reserve mode); `Non-dispatchable` VRE units cannot:

$$
g_{i} + \big(r^{\mathrm{regUp}}_{i}+r^{\mathrm{flexUp}}_{i}+r^{\mathrm{spin}}_{i}\big)
\ \le\ AF_{i,d,h,y}\,u^{G}_{i,y}\,CAP_i\,PMAX_i
\qquad \forall\, i\in\mathcal I^{VRE,\mathrm{fix}},\,d,h,t,y
$$

$AF_{i,d,h,y}$ is the fraction of nameplate output available in that interval (for example zero at night for solar). It is supplied as an exogenous shape. The model can curtail a VRE unit below $AF\cdot CAP\cdot PMAX$, for instance in a low-load or transmission-constrained hour, but can never dispatch it above that ceiling. A `Dispatchable` unit, such as an inverter-based plant capable of active-power control, may curtail below its available output and hold that headroom as up-reserve. `Non-dispatchable` units never appear in the up-reserve sums.

!!! note "How the zonal availability shape is aggregated"
    When a model bus aggregates several data sub-regions, $AF_{i,d,h,y}$ is the **mean of the sub-regional profiles over the resource-bearing sub-regions only**. A sub-region counts as resource-bearing for a technology if its profile is non-zero in at least one representative-day hour of that stage. Sub-regions that are zero (or absent from the time-series file) in every hour are structural zeros, meaning the resource does not exist there, and are excluded from both numerator and denominator. A zone is therefore not penalized for containing, for example, land-locked sub-regions in its offshore-wind shape. Temporal zeros (night for solar, below-cut-in hours for wind) remain in the denominator, because the resource-bearing set is fixed per stage, zone and technology rather than recomputed each hour. If a zone has no resource-bearing sub-region for a technology, its shape is zero in every hour.

For intervals with near-zero availability ($AF\cdot CAP\cdot PMAX<10^{-4}$), $g_i$ and any up-reserves are fixed to zero, which is equivalent to the constraint above and avoids a badly scaled row.

### Energy budget

Budget-limited hydro caps the summed generation (plus up-reserves for dispatchable units) over a day group by the per-MW water budget:

$$
\sum_{d\in \mathrm{grp}}\sum_{h,t}\!\Big(g_{i}+r^{\mathrm{regUp}}_{i}+r^{\mathrm{flexUp}}_{i}+r^{\mathrm{spin}}_{i}\Big)
\ \le\ B_{i,\mathrm{grp}}\,u^{G}_{i,y}\,CAP_i\,PMAX_i
\qquad \forall\, i\in\mathcal I^{VRE,\mathrm{bud}}
$$

There is no per-hour cap. The model can shape hydro output freely within the day group, for example concentrating release in the peak hours, as long as the total energy released does not exceed the water budget $B_{i,\mathrm{grp}}$. This lets budget-limited hydro act as a dispatchable peaking and flexibility resource without state-of-charge tracking.

An annual variant of the budget sums across every representative day in the year, weighting by $\sigma_d$ and scaling by $\tfrac{1}{365}\sum_d\sigma_d$. It is appropriate when the water limit is genuinely annual, such as an annual acre-foot allocation, rather than resetting at each day-group boundary.

The optional `Hydro_Flexibility_Flag` lets a run-of-river unit shift some output within its daily profile rather than being locked to the hourly shape. `Hydro_Flexibility_Flag` and `Hydro_Budget_Flag` are independent switches, not mutually exclusive alternatives. A unit with both set applies both: the intra-day flexibility on top of the day-group or annual energy cap. If neither is set, impoundment hydro is treated as strict run-of-river with a fixed profile, the least flexible behavior.

### Water-value and reservoir-segment hydro dispatch

A third mechanism applies to hydro units whose `FUEL_LIMIT` is `Budget (day groups)` and whose fuel type is `Water Value`. It models a reservoir whose usable water is divided into discrete elevation or storage **segments** $j$ (from the `Reservoir_Level` values of the water-value table), each with its own share of the day-group water budget. This is richer than the flat energy-budget cap above and is useful when the marginal value of water depends on how full the reservoir is: a segment near the top of the reservoir is cheaper to draw down than one near the bottom. Generation equals the sum of segment-level water-use variables $wu_{i,j}$:

$$
g_{i,d,h,t,y} = \sum_{j} wu_{i,j,d,h,t,y}
\qquad \forall\, i\in\mathcal I^{\mathrm{water}},\,d,h,t,y
$$

Three further limits apply:

- The **day-group total** water use cannot exceed a reservoir-derived share of the day-group water budget, scaled by the unit's `EXCAPS` and applied to installed units $u^G_{i,y}$.
- Each **segment's** cumulative day-group use is capped by that segment's own share of the budget, given by the differences in `Reservoir_Level` between consecutive segments.
- Each segment's **daily** use must be non-increasing from day to day within the group, a monotonic drawdown order that reflects a reservoir generally not refilling within a group.

No simulation setting switches this mechanism: it applies whenever a unit has these data attributes. When `Hydro_Budget_Flag` is also set for the same unit, the ordinary day-group or annual budget cap continues to apply, and the water-value segments shape how the same overall budget is drawn down.

---

## Storage

Storage units ($i\in\mathcal I^{ES}$) discharge through the generation variable $g_i$ and charge through $chg_i$, and track state of charge $soc_i$ with one-way efficiency $\eta_i=\sqrt{\texttt{BATEFF}}$. The round-trip efficiency `BATEFF` is split evenly between the charge and discharge legs, so a 90% round-trip battery loses about 5% going in and about 5% coming out. Unlike a thermal unit's ramp limit, which looks back one hour, a storage unit's dispatch depends on its whole charge history through the state of charge. See [Storage Modeling](../../database/Storage_Modeling_Reference.md) for the data.

### Power limits with reserve headroom

Discharge is limited by the maximum-generation bound above: discharge plus up-reserves cannot exceed $u^G_i\,PMAX_i\,CAP_i$, so storage discharge competes for up-reserve headroom on the same footing as a peaking unit. Charging plus down-reserves (individual reserve mode) is limited by the installed power:

$$
chg_{i} + \big(r^{\mathrm{regDn}}_{i}+r^{\mathrm{flexDn}}_{i}\big)\ \le\ u^{G}_{i,y}\,CAP_i\,PMAX_i
\qquad \forall\, i\in\mathcal I^{ES},\,d,h,t,y
$$

Charging and discharging share the same symmetric MW rating. Providing down-reserve means committing to charge more on request, which requires headroom below the charging ceiling, just as up-reserve on discharge requires headroom below the discharge ceiling. Hybrid storage units use an analogous charging limit.

### Storage commitment (optional binary)

When the setting `Storage Commitment` is true, a binary $c^{STO}_i$ prevents simultaneous charging and discharging through a big-M switch, with $c^{STO}_i\le u^G_i$. Up-reserves are tied to the discharge side (individual mode, eligible units only) and down-reserves to the charge side (individual mode):

$$
g_{i}+\big(r^{\mathrm{regUp}}_{i}+r^{\mathrm{flexUp}}_{i}+r^{\mathrm{spin}}_{i}\big)\ \le\ M\,c^{STO}_{i},\qquad
chg_{i}+\big(r^{\mathrm{regDn}}_{i}+r^{\mathrm{flexDn}}_{i}\big)\ \le\ M\,(1-c^{STO}_{i}),\qquad
c^{STO}_{i}\ \le\ u^{G}_{i,y}
\qquad \forall\, i,d,h,t,y
$$

with $M=\max(1000,\texttt{MAXINVEST}_i)\,PMAX_i\,CAP_i$. The big-M is generous enough not to restrict real dispatch when the switch is on the allowed side.

With $c^{STO}_i=1$ the unit may discharge and provide up-reserve, and charging and down-reserve are forced to zero. With $c^{STO}_i=0$ the reverse holds. Without the switch, a continuous (LP) model can show a battery charging and discharging simultaneously in the same interval, which nets to zero throughput but is not physical operation. The binary adds one integer variable per storage unit per interval, which increases solve time. It is unnecessary when any nonzero round-trip loss already makes simultaneous charging and discharging unattractive economically.

### State-of-charge bounds with reserve reservation

Up-reserves hold stored energy above the floor, and down-reserves hold headroom below the cap (individual mode). Here $\kappa^{\mathrm c}=$ `contingency_reserve_min_duration_value`:

$$
STOMIN_i + \tfrac{1}{\eta_i}\big(r^{\mathrm{regUp}}_{i}+r^{\mathrm{flexUp}}_{i}+\kappa^{\mathrm c}r^{\mathrm{spin}}_{i}\big)
\ \le\ soc_{i}
\ \le\ u^{ESE}_{i,y} - \eta_i\big(r^{\mathrm{regDn}}_{i}+r^{\mathrm{flexDn}}_{i}\big)
\qquad \forall\, i\in\mathcal I^{ES},\,d,h,t,y
$$

Both bounds use the same one-way efficiency $\eta_i=\sqrt{\texttt{BATEFF}}$: the up-reserve headroom is scaled by $1/\eta_i$ and the down-reserve headroom by $\eta_i$.

This is the energy-side counterpart of the power headroom above. Providing up-reserve is a commitment to discharge more on request, which requires enough stored energy above $STOMIN_i$ (divided by $\eta_i$, because discharging draws down stored energy faster than it delivers power). Providing down-reserve is a commitment to absorb more energy, which requires headroom below the energy cap $u^{ESE}_{i,y}$. The contingency term is scaled by $\kappa^{\mathrm c}$, the minimum duration in hours that contingency reserve must be sustainable, so a battery committing $r^{\mathrm{spin}}$ MW must hold $\kappa^{\mathrm c}\,r^{\mathrm{spin}}$ MWh of usable energy. For example, a battery with $STOMIN=1$ MWh, $\eta=0.95$ and $\kappa^{\mathrm c}=0.5$ h that provides $r^{\mathrm{spin}}=2$ MW must keep $soc\ge1+\frac{1}{0.95}(0.5\times2)\approx2.05$ MWh.

In the aggregated reserve mode of the GTEP-embedded dispatch (see the closing section), the state-of-charge bounds use the fourth root $\sqrt[4]{\texttt{BATEFF}}$ on both sides instead of the one-way efficiency.

### State-of-charge balance, initialization and cyclic closure

State of charge evolves by the efficiency-weighted charge and discharge over the sub-hour count $|\mathcal T|$, so that sub-hourly intervals do not multiply the accumulated energy change:

$$
soc_{i,d,h,t,y}=soc_{i,\mathrm{prev}}+\frac{\eta_i\,chg_{i}-\tfrac{1}{\eta_i}g_{i}}{|\mathcal T|}
\qquad \forall\, i\in\mathcal I^{ES},\,d,h,t,y
$$

This balance links a unit's dispatch across time. Because representative days are not necessarily consecutive calendar days, the model needs an explicit rule for $soc_{i,\mathrm{prev}}$ at the first modeled interval of a day group. At that point the previous state is replaced by an initial state of charge chosen with `storage initialization option`:

- `Minimum`: $soc_0=STOMIN_i$. Pessimistic; tests whether the system can meet load with storage essentially empty.
- `Middle`: $soc_0=u^{ESE}_{i,y}/2$. A neutral half-full assumption.
- `Maximum`: $soc_0=u^{ESE}_{i,y}$. Optimistic; storage starts fully charged.
- `Optimal` (default): no fixed initial state. A **cyclic closure** ties the last interval of the group back to its first, so the solver chooses the cheapest starting state subject to ending the group where it started. No energy is created or destroyed at the boundary, and the trajectory could repeat indefinitely without drift:

$$
soc_{i,\mathrm{end}} - soc_{i,\mathrm{begin}} + \frac{\eta_i\,chg_{i,\mathrm{begin}}-\tfrac{1}{\eta_i}g_{i,\mathrm{begin}}}{|\mathcal T|}=0.
$$

`Minimum`, `Middle` and `Maximum` bound the value of storage from one side. For example, a `Minimum` run gives a conservative estimate because storage receives no free initial charge. `Optimal` does not bias the result in either direction and does not require assuming a starting state of charge.

### Annual energy throughput (AET)

When `Energy_Storage_AET_Limit_Flag` is true, the throughput of each storage unit over the horizon is capped. The form is the same efficiency-weighted, factor-of-2 form given on the [GTEP page](../GTEP/GTEP_Formulation.md#storage-annual-energy-throughput-aet). The cap limits total annual cycling (charge plus discharge MWh) rather than instantaneous power, which represents technologies whose degradation or warranty is driven by cumulative cycling. The power and state-of-charge limits above do not constrain how many times per year a unit cycles.
---

## Large flexible load and hybrid resources

A large flexible load (LFL) $l\in\mathcal L$ is a price-responsive, capped demand. It can shed load through piecewise demand-response (DR) segments and may include on-site generation and on-site storage. It represents a large industrial or data-center customer, or an aggregated DR program, whose consumption is a decision the model partly controls in exchange for a price signal rather than a fixed must-serve profile like ordinary bus load $D_n$. The site may sit behind its own interconnection point with co-located generation and storage. Its served withdrawal $lfl_l$ enters the nodal balance as a subtracted load (see above). Data for these resources is described in [Large Load and Demand Response](../../database/Large_Load_and_Demand_Response_Reference.md) and [Hybrid Resources](../../database/Hybrid_Resources_Reference.md).

### Interconnection balance and limits

The served flexible load equals the sum of DR segments minus any on-site supply delivered directly to the load. Grid injection and withdrawal are each bounded by the interconnection limit $INTERCON_l$:

$$
\begin{aligned}
lfl_{l} &= \sum_{s} lfl^{\mathrm{seg}}_{l,s} - g^{G\text{-}LFL}_{l} - g^{ES\text{-}LFL}_{l} && \text{(power balance)}\\
lfl_{l} &\le CAP_l && \text{(flat-load cap)}\\
g^{G\text{-}Grid}_{l} + g^{ES\text{-}Grid}_{l} &\le INTERCON_l && \text{(injection to grid)}\\
lfl_{l} + chg^{Grid\text{-}ES}_{l} &\le INTERCON_l && \text{(withdrawal from grid)}
\end{aligned}
\qquad \forall\, l,d,h,t,y
$$

The first row is a local balance behind the meter. The load's draw from the grid ($lfl_l$) equals the scheduled DR consumption net of whatever the load's own generation or storage delivers directly to it, so on-site supply displaces grid withdrawal MW for MW. A load with a large solar array or battery therefore shows a smaller net grid draw than its gross schedule. The flat-load cap $lfl_l\le CAP_l$ is the nameplate size of the load. The two interconnection rows reflect a single point of interconnection with one MW rating in each direction: on-site generation and on-site storage discharge compete for the same outbound capacity, and the load's grid draw plus any grid charging of on-site storage compete for the same inbound capacity.

### Piecewise DR segments and daily cap

Segment 1 is the fixed flat baseline. Higher segments $s\ge 2$ activate in order through the (optional binary) indicator $lfl^{\mathrm{ind}}$, and the DR (curtailed) load fills the gap to the total baseline, bounded per day by $\overline{DR}_l$:

$$
\begin{aligned}
lfl^{\mathrm{seg}}_{l,1} &= LFL^{\mathrm{seg}}_{l,1}, &
lfl^{\mathrm{seg}}_{l,s} &\le LFL^{\mathrm{seg}}_{l,s}\,lfl^{\mathrm{ind}}_{l,s}\ (s\ge2), &
lfl^{\mathrm{ind}}_{l,s} &\ge lfl^{\mathrm{ind}}_{l,s+1} \\
lfl_{l} + lfl^{\mathrm{DR}}_{l} &= \sum_{s} LFL^{\mathrm{seg}}_{l,s}, &
\sum_{h,t} lfl^{\mathrm{DR}}_{l} &\le \overline{DR}_l &&
\end{aligned}
\qquad \forall\, l,(s,)d,h,t,y
$$

with $LFL^{\mathrm{seg}}_{l,s}=\texttt{Pct\_MW}_{l,s}\,CAP_l$. The DR value of the served segments $s\ge 2$ is credited in the objective at $-C^{DR}_{l,s}$.

This is a piecewise-linear demand bid curve. Segment 1 is always fully served and represents the portion of the load that is not price-responsive. Each higher segment is an additional increment of demand that is served only if it clears its own price $C^{DR}_{l,s}$ against everything else in the system. The ordering constraint $lfl^{\mathrm{ind}}_{l,s}\ge lfl^{\mathrm{ind}}_{l,s+1}$ forces segments to activate in price order: segment $s+1$ cannot be on while segment $s$ is off. The variable $lfl^{\mathrm{DR}}_l$ is the complement, the part of the total baseline demand not served in that interval, and is limited per day by $\overline{DR}_l$ (`Daily_DR_Limit_MWh`), which represents a contractual or operational limit on the MWh of curtailment that can be called in a day.

The indicator is binary when the load's `Integer_Flag` is true and continuous on $[0,1]$ otherwise. A continuous indicator lets a segment activate partially, which approximates a large aggregated DR resource well but not an all-or-nothing curtailment contract. The binary form increases solve time.

### On-site generation

On-site generation (thermal or VRE-shaped) is split among routes to the load, the grid and the on-site storage, and is capped by its shape-scaled capacity:

$$
g^{G\text{-}LFL}_{l} + g^{G\text{-}Grid}_{l} + g^{G\text{-}ES}_{l}\ \le\ CAP^{G}_l\,AF_{l,d,h,y}
\qquad \forall\, l\ \text{with on-site gen},\ d,h,t,y
$$

The three routes share one physical generator: the sum of direct supply to the load, export to the grid (through the shared interconnection above) and charging of the co-located storage cannot exceed what the plant can produce in that interval. $AF=1$ for a thermal unit and equals the technology availability shape for a VRE unit, as in the fixed-profile treatment for grid-connected VRE.

### On-site storage

On-site storage has charge (from the grid and from on-site generation), discharge (to the grid and to the load), state-of-charge bounds and state-of-charge dynamics with $\eta^{L}_l=\sqrt{\texttt{BATEFF}}$. It mirrors the grid-storage formulation, with the same efficiency-weighted recursion, initialization options and cyclic closure, but is scoped to one site's private battery:

$$
\begin{aligned}
chg^{Grid\text{-}ES}_{l} + g^{G\text{-}ES}_{l} &\le CAP^{\mathrm{chg}}_l, &
g^{ES\text{-}Grid}_{l} + g^{ES\text{-}LFL}_{l} &\le CAP^{ES}_l, &
STOMIN_l &\le soc^{L}_{l} \le CAP^{\mathrm{chg}}_l \\[2pt]
soc^{L}_{l,d,h,t,y} &= soc^{L}_{l,\mathrm{prev}} + \frac{\eta^{L}_l\big(chg^{Grid\text{-}ES}_{l}+g^{G\text{-}ES}_{l}\big) - \tfrac{1}{\eta^{L}_l}\big(g^{ES\text{-}Grid}_{l}+g^{ES\text{-}LFL}_{l}\big)}{|\mathcal T|}
\end{aligned}
$$

Both charge sources share one charging power limit, and both discharge destinations share one discharge power limit, reflecting a single converter rating. The initialization options and the cyclic closure are the same as for grid storage.

!!! note "One data field serves as both charge power and energy capacity"
    For an LFL's on-site storage, the `Charge_CAP` field bounds both the charging power (MW) and the state-of-charge upper limit (MWh). The discharge power limit reads a separate `CAP` field. Populate `Charge_CAP` with this dual role in mind.

### Hybrid resources (shared interconnection)

A hybrid plant couples a main generator (index $i$) and a co-located storage unit ($i_{ES}$) behind a **shared** interconnection limit. A typical example is a solar-plus-storage plant in which the PV array and the battery connect to the grid through one substation. The pairing is a grid-side asset, unlike the on-site storage of a flexible load, which sits behind the load's meter. The main generator may route power to the grid and to the co-located storage ($g^{G\text{-}ES}_i$), and the two share $INTERCON$:

$$
\begin{aligned}
g_{i} + g^{G\text{-}ES}_{i} + \text{(up-reserves)} &\le CAP_i\,AF_{i,d,h,y} && \text{(on-site gen cap)}\\
\big(g_{i}+\text{up-res}\big) + \big(g_{i_{ES}}+\text{up-res}\big) &\le INTERCON && \text{(shared injection)}\\
chg_{i_{ES}} + \text{(down-reserves)} &\le INTERCON && \text{(shared withdrawal)}
\end{aligned}
\qquad \forall\, i\in\mathcal I^{HYB},\,d,h,t,y
$$

The first row bounds the generator's own output (to the grid plus what it routes to the storage) by its nameplate and availability, so the array can charge the battery even when it exports nothing. The second and third rows couple the pair: the combined export of generator and storage cannot exceed the shared $INTERCON$, which is often smaller than the sum of the two nameplate ratings, and the storage's charging cannot exceed the same limit on the withdrawal side. Without this coupling, a 150 MW array and a 100 MW battery could both export at full rating through a 100 MW interconnection.

Grid charging of the hybrid storage is set to zero when the hybrid's `Grid_Charge` is false, so the storage charges only from its co-located generator. With `Grid_Charge` true, the storage can also arbitrage grid energy. The hybrid storage has its own state-of-charge balance and cyclic closure, with the same efficiency-weighted form as grid storage.

---

## Standalone Operation vs. GTEP-embedded dispatch

The equations above are identical in both models. The differences lie in how the investment quantities are treated and in the outer loop.

- **Standalone Operation** usually fixes the investment quantities to a prior expansion result (`expansion_fix = true`, applied whenever an expansion result is supplied). It disables the multi-round solution process and transmission expansion, and re-solves the dispatch chronologically for each representative day group. Transfer limits therefore treat the expansion increment $u^T_{k,y}$ as a **constant**. See [Operation run modes](./Operation_Execution_and_Run_Modes.md). When no prior expansion result is supplied (`expansion_fix = false`), the model dispatches the data-specified existing fleet with no expansion decision anywhere: the installed unit counts are set to the existing-unit counts in the technology data, $u^G_{i,y}=$ `EXUNITS`, and the storage energy capacity is set to the existing storage energy, $u^{ESE}_{i,y}=$ `ES_MWh`.
- **GTEP-embedded dispatch** applies the same equations to every representative day group of every stage $y$. The operational constraints read the investment **variables** ($u^G_{i,y}$, $u^{ESE}_{i,y}$, $u^T_{k,y}$) directly, and the dispatch cost enters the expansion objective weighted by $\sigma_d\,\omega(\cdot)/|\mathcal T|$. See the [GTEP embedded operation cost](../GTEP/GTEP_Formulation.md#embedded-operation-cost-zmathrmop).
- **Aggregated operating-reserve mode is available only in GTEP-embedded dispatch.** Standalone Operation requires `operating_reserve_modeling_option == "individual"` and stops with an error for any other value, so reserves are always tracked unit by unit. Every reference on this page to an "aggregated" reserve mode (reserve headroom, ramping and storage state-of-charge limits expressed through reserve-group proxies) applies only to the GTEP-embedded dispatch. The aggregated inter-temporal ramping option is a separate case: it is controlled by `include_dispatch_ramping_flag` and `dispatch_ramping_modeling_option`, applies only when `Dispatch_Mode_in_EXP` is not `Unit Commitment`, and is likewise not available in standalone Operation.

---

## Related pages

- [Operation Overview](./Operation_Overview.md) — what the Operation model is for, run modes at a glance.
- [Operation Execution and Run Modes](./Operation_Execution_and_Run_Modes.md) — the three ways to invoke this dispatch core in practice.
- [Operation Outputs](./Operation_Outputs.md) — result files and column definitions produced by the constraints above.
- [GTEP formulation](../GTEP/GTEP_Formulation.md) — shared sets/parameters/variables, and how the expansion objective weights this same dispatch core when embedded.
- [RA formulation](../RA/RA_Formulation.md) — reuses this dispatch core inside each outage and renewable scenario.
