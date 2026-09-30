# Glossary and Notation Quick Reference

Short definitions of the acronyms and notation used across this site. Each entry links to the page that defines it in full.

!!! tip "Looking for math notation instead of acronyms?"
    See [Notation used in the Formulation pages](#notation-used-in-the-formulation-pages)
    at the bottom of this page.

## Model Families

| Term | Meaning |
|---|---|
| **GTEP** | Generation and Transmission Expansion Planning: the multi-stage investment and retirement model. See [GTEP Overview](./models/GTEP/GTEP_Overview.md). |
| **Operation** | The dispatch (unit commitment and economic dispatch) model. It runs standalone, after a GTEP run, or as the representative-day dispatch inside GTEP. See [Operation Overview](./models/Operation/Operation_Overview.md). |
| **RA** | Resource Adequacy: the scenario-based reliability assessment model (generator outage and renewable availability uncertainty). See [RA Overview](./models/RA/RA_Overview.md). |

## Reliability and Capacity-Credit Metrics

| Term | Meaning |
|---|---|
| **EUE** | Expected Unserved Energy: the reliability metric RA aggregates from unserved-energy outcomes across scenarios. See [RA Metrics, ELCC, and DLOL](./models/RA/RA_Metrics_ELCC_and_DLOL.md). |
| **NEUE** | Normalized EUE (EUE expressed relative to annual demand). See [RA Metrics, ELCC, and DLOL](./models/RA/RA_Metrics_ELCC_and_DLOL.md). |
| **LOLH** | Loss of Load Hours: expected number of hours with unserved load. See [RA Metrics, ELCC, and DLOL](./models/RA/RA_Metrics_ELCC_and_DLOL.md). |
| **LOLE** | Loss-of-Load Expectation: LOLH converted to a days-per-year style adequacy measure. See [RA Formulation](./models/RA/RA_Formulation.md). |
| **ENS** | Energy Not Served (unserved energy): demand that cannot be met and is curtailed, priced at VOLL in the objective. See [Operation Formulation](./models/Operation/Operation_Formulation.md) and [Operation Outputs](./models/Operation/Operation_Outputs.md). |
| **VOLL** | Value of Lost Load: the penalty price applied to unserved energy (ENS) in the objective. See [Simulation Configuration Reference](./configuration/Simulation_Configuration_Reference.md). |
| **FOR** | Forced Outage Rate: a generator's probability of being unavailable, used to sample outage scenarios in RA. See [RA Overview](./models/RA/RA_Overview.md) and [RA Formulation](./models/RA/RA_Formulation.md). |
| **ELCC** | Effective Load Carrying Capability: how much additional constant load a resource lets the system serve at the same reliability level (or, in deactivation mode, how much load relief restoring reliability after its removal requires), expressed as a fraction of installed capacity. See [ELCC](./models/RA/RA_Metrics_ELCC_and_DLOL.md#elcc). |
| **DLOL** | Direct Loss of Load: a dispatch-based capacity-credit proxy: a unit's average output during system loss-of-load hours, normalized by its installed capacity (its capacity factor during scarcity). Despite the name, it is not a duration or hours metric. See [DLOL](./models/RA/RA_Metrics_ELCC_and_DLOL.md#dlol). |
| **CAPCRED** | The `Gen Technology` column holding a technology's capacity credit (a literal fraction, or a lookup key into a zonal capacity-credit table). Used to derate `CAP` into `UCAP`. See [Gen Technology Reference](./database/Gen_Technology_Reference.md#capcred). |
| **UCAP / ICAP** | UCAP is capacity-credit-derated capacity (`CAP × CAPCRED`); ICAP is nameplate/installed capacity before that derating. See [Gen Technology Reference](./database/Gen_Technology_Reference.md#capcred) and [Operation Outputs](./models/Operation/Operation_Outputs.md). |
| **PRM** | Planning Reserve Margin: the reliability-capacity constraint requiring accredited (UCAP-weighted) capacity to exceed peak demand by a margin. See [Simulation Configuration Reference](./configuration/Simulation_Configuration_Reference.md) and the [planning reserve margin constraint](./models/GTEP/GTEP_Formulation.md#planning-reserve-margin-reliability). |

## Policy and Financial Terms

| Term | Meaning |
|---|---|
| **RPS** | Renewable Portfolio Standard: a regional renewable-energy-share policy target. See [Policy and Financial Settings](./configuration/Policy_and_Financial_Settings.md#rps). |
| **CEGT** | Clean Energy Generation Target: a regional clean-generation-share policy target (defined over policy zones the same way as RPS). See [Policy and Financial Settings](./configuration/Policy_and_Financial_Settings.md#cegt). |
| **CERT** | Carbon Emission Reduction Target: a regional emissions-reduction policy structure. See [Policy and Financial Settings](./configuration/Policy_and_Financial_Settings.md#cert). |
| **ITC** | Investment Tax Credit: a technology-level capital-cost incentive, folded into the annualized investment cost in the GTEP objective. See [Policy and Financial Settings](./configuration/Policy_and_Financial_Settings.md). |
| **PTC** | Production Tax Credit: a technology-level per-MWh generation incentive, subtracted in the GTEP objective. See [Policy and Financial Settings](./configuration/Policy_and_Financial_Settings.md) and [Production tax credit](./models/GTEP/GTEP_Formulation.md#production-tax-credit-zmathrmptc-subtracted). |
| **ATB** | Annual Technology Baseline: an external technology cost and performance dataset (the bundled data uses ATB 2024) that the `ATB Setting` sheet maps `Gen Technology` rows onto. See [Gen Technology Reference](./database/Gen_Technology_Reference.md). |
| **ESGC** | A label, not an expanded acronym: the storage cost and performance data-source key (`ESGC_Setting_ID`) that joins a `Gen Technology` row to a matching row in `Storage Cost and Performance`. See [Policy and Financial Settings](./configuration/Policy_and_Financial_Settings.md#storage-cost-and-performance). |
| **AET** | Annual Energy Throughput: a storage technology's yearly cycling limit (a proxy for cycle-life/degradation limits). See [Storage Modeling Reference](./database/Storage_Modeling_Reference.md). |

## Power Flow and Network Terms

| Term | Meaning |
|---|---|
| **PTDF** | Power Transfer Distribution Factor: a shift-factor DC power-flow mode (`power_flow_mode_flag`). It does not support DC ties. See [Network Data Reference](./database/Network_Data_Reference.md). |
| **B-theta** | The full DC optimal power flow mode (angle-based, with one reference bus per synchronous island). Asynchronous DC ties are excluded from the angle equation but their transfer limits still apply. See [Network Data Reference](./database/Network_Data_Reference.md) and [Operation Formulation](./models/Operation/Operation_Formulation.md). |
| **Network_Flow** | The aggregate transfer-limit power-flow mode (zonal transfer capacity, no angle or susceptance representation). See `power_flow_mode_flag` in [ALEAF Simulation Setting File Reference](./configuration/ALEAF_Simulation_Setting_File.md). |
| **DC tie / VFT** | An asynchronous DC tie (a back-to-back HVDC link or variable-frequency transformer, VFT), modeled as an angle-decoupled transfer with its own rate limit. See [Network Data Reference](./database/Network_Data_Reference.md). |
| **PTDF reduction** | The network-reduction feature that re-estimates aggregated-corridor reactances from synthetic operating snapshots so that a coarser network reproduces the finer network's flows. It is distinct from the **PTDF** power-flow mode above. See [Network Resolution — PTDF / snapshot network reduction](./database/Network_Resolution.md#ptdf-snapshot-network-reduction). |

## Network Resolution

| Term | Meaning |
|---|---|
| **`subdivision` resolution** | The finest bundled spatial level (`Country subdivision` in the bundled configuration): the sub-state grouping used by the North America datasets, below `BA zone` and `BA`. User-supplied finer levels are supported. See [Network Resolution — The Resolution Ladder](./database/Network_Resolution.md#the-resolution-ladder). |
| **resolution level (aggregation resolution)** | Where a run sits on the ladder `Country subdivision` < `BA zone` < `BA` < `Interconnection` < `Country` in the bundled configuration, selected by `regional_aggregation_resolution_type`/`_level` (and its sub-regional counterpart) in `Network Setting`. See [Network Resolution — Selecting Resolution](./database/Network_Resolution.md#selecting-resolution). |
| **data region** | A (possibly coarser) region at which a time-series profile is read and then applied to the finer network, for example load at `BA` and wind at `subdivision`. See [Network Resolution — Per-Profile Data Resolution](./database/Network_Resolution.md#per-profile-data-resolution). |

## Resource and Load Terms

| Term | Meaning |
|---|---|
| **VRE** | Variable Renewable Energy: resource-availability-limited technologies (wind, solar PV, and similar) whose output follows a time-series profile. See [Gen Technology Reference](./database/Gen_Technology_Reference.md). |
| **LFL** | Large Flexible Load: large, schedulable or curtailable loads (for example data centers and large industrial load) defined in the `demand` sheet. See [Large Load and Demand Response Reference](./database/Large_Load_and_Demand_Response_Reference.md). |
| **SOC** | State of Charge: a storage resource's stored energy level, tracked chronologically subject to charge/discharge and initialization settings. See [Storage Modeling Reference](./database/Storage_Modeling_Reference.md). |
| **UC** | Unit Commitment: the on/off (and, where modeled, startup/shutdown) commitment decision for thermal units, as opposed to pure economic dispatch. See [Operation Formulation](./models/Operation/Operation_Formulation.md). |

## Execution and Runtime Terms

| Term | Meaning |
|---|---|
| **distributed operation solve** | The parallel operation run mode (`run_operation_in_parallel_flag`) in which per-stage representative-day-group subproblems are solved on separate worker processes instead of serially. See [Operation Execution and Run Modes — Serial vs Distributed Operation Solves](./models/Operation/Operation_Execution_and_Run_Modes.md#serial-vs-distributed-operation-solves). |
| **light master reference** | The reduced main-process model used in a distributed operation solve so that the full per-day-group data is not replicated on the main process (a memory saving at nodal scale). See [Operation Execution and Run Modes — Light master reference](./models/Operation/Operation_Execution_and_Run_Modes.md#light-master-reference-nodal-out-of-memory-behavior). |

## Notation Used in the Formulation Pages

The three [Formulation pages](./models/GTEP/GTEP_Formulation.md) (GTEP, [Operation](./models/Operation/Operation_Formulation.md), [RA](./models/RA/RA_Formulation.md)) share one nomenclature, defined in full in
[GTEP Formulation — Notation conventions](./models/GTEP/GTEP_Formulation.md#notation-conventions)
and [Sets and indices](./models/GTEP/GTEP_Formulation.md#sets-and-indices). The
essentials, so an equation can be read without opening that page:

- **Sets** use calligraphic capitals ($\mathcal{I}$, $\mathcal{D}$, ...); superscripts
  mark subsets (e.g. $\mathcal{I}^{ES}$ = storage units).
- **Parameters** are upper-case Latin/Greek with the qualifier as a superscript and
  indices as subscripts (e.g. $C^{\mathrm{inv}}_{i,y}$).
- **Variables** are lower-case (e.g. $g_{i,d,h,t,y}$ for generation); investment/retirement
  decisions use $u$ with a superscript tag.
- **`(d,h,t,y)`** is the core operational index tuple: $d$ = representative day group,
  $h$ = hour within the day, $t$ = sub-hour interval within the hour (usually degenerate
  — most runs set $|\mathcal T|=1$), $y$ = planning **stage** id (not a calendar year —
  see [Planning Horizon and Multi-Round](./models/GTEP/GTEP_Planning_Horizon_and_Multi_Round.md)).
- $\mathcal P$ (policy/RPS/CEGT zones), $\mathcal P^{R}$ (planning-reserve zones),
  $\mathcal Z$ (operating-reserve zones), and $\mathcal S$ (resource-supply-curve zones)
  are four **independent** zonal partitions of the same bus set — they need not
  coincide, even though a single-region case can collapse them all to "the whole
  system."
