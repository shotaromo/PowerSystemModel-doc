---
icon: material/draw-pen
---

# Equations

!!! note "Notation conventions"
    Here, capital characters refer to endogenous variables, while lowercase characters indicate exogenous parameters.

??? tip "GAMS symbol naming conventions"
    The model follows systematic naming rules for GAMS symbols to ensure consistency and readability:
    
    **Decision variables (Documentation ↔ GAMS correspondence):**
    
    - All decision variables start with `V` in both documentation and GAMS implementation
    - Documentation uses generic notation ($VX$, $VS$, $VR$, $VP$) with technology-agnostic subscripts
    - GAMS uses technology-specific prefixes where the second letter indicates technology type:
        - `VG*` for generators: $VX_{n,i,r,t}$ → `VG(N,I,R,T)`, $VS_{n,i,r}$ → `VGS(N,I,R)`, $VR_{n,i,r}$ → `VGR(N,I,R)`, $VP_{n,i,r}$ → `VGP(N,I,R)`
        - `VF*` for links: $VX^+_{l,t}$ → `VFF(L,T)`, $VX^-_{l,t}$ → `VFB(L,T)`, $VS_l$ → `VFS(L)`, $VR_l$ → `VFR(L)`, $VP_l$ → `VFP(L)`
        - `VE*` for storage (energy): $VE_{n,i,s,t}$ → `VE(N,I,S,T)`, $VS_{n,i,s}$ → `VES(N,I,S)`, $VR_{n,i,s}$ → `VER(N,I,S)`, $VP_{n,i,s}$ → `VEP(N,I,S)`
        - `VH*` for storage (power): $VX_{n,i,s,t}$ → `VH(N,I,S,T)`
        - `VP*` for energy supply: $VP_{n,i,k}$ → `VP(N,I,K)`
        - `VQ*` for emissions: $VQ_{n,i,g}$ → `VQ(N,I,G)`
    - Third letter in GAMS indicates variable type: `S` (stock/capacity), `R` (new installation/renewal), `P` (replacement), `X` (operation/dispatch), `F`/`B` (forward/backward flow)
    - Special case: Total cost uses `VTC` in both documentation ($VTC$) and GAMS (`VTC`)
    
    **Parameters:**
    
    - Technology-specific parameters start with technology letter: `g` (generator), `f` (link), `e` (storage)
    - Cost parameters: `cc` (capital cost), `foc` (fixed O&M cost), `voc` (variable O&M cost), `rc` (replacement cost), `acc` (annualized capital cost), `arc` (annualized replacement cost)
    - Technical parameters: `mn` (minimum), `mx` (maximum), `fx` (fixed), `lt` (lifetime), `alp` (alpha/discount rate), `eta` (efficiency)
    - Stock parameters: `sc` (stock by vintage), `ssc` (surviving stock), `sr` (replaceable stock), `ssr` (surviving replaceable stock)
    - Examples: `gacc` = generator + annualized capital cost, `gmx` = generator + maximum, `eeta` = storage (energy) + efficiency (self-discharge rate)
    
    **Sets and mappings:**
    
    - Aggregation sets start with `M` followed by base set letter: `MR` (generator group), `ME` (region-sector pair), `MK` (energy type group), etc.
    - Mapping sets start with `M_` followed by set names: `M_MR` (mapping from MR to R), `M_ME` (mapping from ME to N,I)
    - Flag sets start with `FL_`: `FL_IR` (valid region-sector-technology combinations), `FL_IRT` (valid region-sector-technology-timeslice combinations)

## Sets and Indices

The model uses base indices for different dimensions, and aggregation sets that group elements from these base indices.

### Base indices

| Notation | Symbol | Description |
|----------|-------------|-------------|
| $n \in N$ | `N` | region |
| $i \in I$ | `I` | sector |
| $r \in R$ | `R` | generator technology |
| $l \in L$ | `L` | link (transmission/distribution) |
| $s \in S$ | `S` | storage technology |
| $t \in T$ | `T` | timeslice |
| $k \in K$ | `K` | energy type |
| $g \in G$ | `G` | emission type |
| $y \in Y$ | `Y` | year (all years including past and future); simulation period is defined by subset `YEAR(Y)` |

### Aggregation sets

Aggregation sets are used to define groups of technologies, regions, or timeslices for applying constraints at a higher level of aggregation. These sets enable policy makers to impose constraints on aggregated technology groups, regions, or energy types rather than on individual elements.

| Notation | Symbol | Definition | Description |
|----------|-------------|-----------|-------------|
| $MR$ | `MR` | $MR \subseteq R$ | generator group set for generation share constraints |
| $ME$ | `ME` | $ME \subseteq N \times I$ | region-sector pair set for energy supply constraints |
| $MK$ | `MK` | $MK \subseteq K$ | energy type group set for energy supply constraints |
| $MQ$ | `MQ` | $MQ \subseteq N \times I$ | region-sector pair set for emission constraints |
| $MG$ | `MG` | $MG \subseteq G$ | emission type group set for emission constraints |
| $ML2$ | `ML2` | $ML2 \subseteq L$ | link group set for link contribution share constraints |
| $MT$ | `MT` | $MT \subseteq T$ | timeslice-group set for nodal power balance |

!!! tip "Usage example"

    - **Generation share constraints**: An aggregation set $MR_0$ might contain renewable technologies (wind, solar, hydro) while $MR_1$ contains all fossil fuel generators. A constraint can then enforce that renewable generation must exceed fossil generation by a certain ratio.
    - **Energy supply constraints**: An aggregation set $ME$ groups multiple region-sector pairs (e.g., industrial sectors across regions) to enforce aggregate biofuel blending requirements.
    - **Emission constraints**: An aggregation set $MQ$ groups specific regions while $MG$ targets particular gases (e.g., CO₂ only), allowing regional or gas-specific emission caps.
    - **Timeslice groups**: An aggregation set $MT$ might represent "peak hours" or "night hours", allowing constraints specific to these time periods.

### Timeslice parameters

Throughout this document, the following timeslice-related parameters are used:

| Notation | Symbol | Description | Unit |
|----------|-------------|-------------|------|
| $\omega$ | `wt(T)` | Timeslice weight (hours in each timeslice) | hours |
| $\Delta t$ | `rt(T)` | Timeslice granularity (hours per timeslice, used for ramping and storage dynamics) | hours |
| $\Delta y$ | `t_int` | Time interval between simulation years | years |

!!! note "Note"

    In the equations, $\omega$ and $\Delta t$ are often treated as constants across $t$ for readability. If time-varying weights or durations are used, the notation can be generalized by reintroducing the subscript $t$. In the GAMS implementation, both `wt(T)` and `rt(T)` are indexed by timeslice $t$ to allow flexibility.

## Objective function

### Total system cost

The objective function minimizes the total system cost, expressed as the sum of **CAPEX** (${C^{\text{CAPEX}}}$), **Fixed OPEX** (${C^{\text{Fixed OPEX}}}$), and **Variable OPEX** (${C^{\text{Variable OPEX}}}$) across all technologies, regions, and sectors:

$$
{VTC} = {C^{\text{CAPEX}}} + {C^{\text{Fixed OPEX}}} + {C^{\text{Variable OPEX}}} \to \min
$$

**Capital expenditure (CAPEX):**

$$
{C^{\text{CAPEX}}} = {C^{\text{CAPEX,new}}} + {C^{\text{CAPEX,replace}}}
$$

$$
{C^{\text{CAPEX,new}}} = \sum_{n,i,r}{c^{\text{new}}}_{n,i,r}\cdot {VR}_{n,i,r}
+ \sum_{l}{c^{\text{new}}}_{l}\cdot {VR}_{l}
+ \sum_{n,i,s}{c^{\text{new}}}_{n,i,s}\cdot {VR}_{n,i,s}
$$

$$
{C^{\text{CAPEX,replace}}} = \sum_{n,i,r}{c^{\text{replace}}}_{n,i,r}\cdot {VP}_{n,i,r}
+ \sum_{l}{c^{\text{replace}}}_{l}\cdot {VP}_{l}
+ \sum_{n,i,s}{c^{\text{replace}}}_{n,i,s}\cdot {VP}_{n,i,s}
$$

**Fixed operating expenditure (Fixed OPEX):**

$$
{C^{\text{Fixed OPEX}}} = \sum_{n,i,r}{o^{\text{fix}}}_{n,i,r}\cdot {VS}_{n,i,r}
+ \sum_{l}{o^{\text{fix}}}_{l}\cdot {VS}_{l}
+ \sum_{n,i,s}{o^{\text{fix}}}_{n,i,s}\cdot {VS}_{n,i,s}
$$

**Variable operating expenditure (Variable OPEX):**

$$
{C^{\text{Variable OPEX}}} = \sum_{n,i,r,t}{\omega}\cdot {o^{\text{variable}}}_{n,i,r}\cdot {VX}_{n,i,r,t}
+ \sum_{l,t}{\omega}\cdot {o^{\text{variable}}}_{l}\cdot \left({VX}^{+}_{l,t}+{VX}^{-}_{l,t}\right)
$$

<details markdown="1">
<summary><b>Parameters used in Total system cost</b></summary>

| Parameter | Symbol | Description | Unit |
|-----------|--------|-------------|------|
| $c^{\text{new}}_{n,i,r}$ | `gacc(N,I,R)` | Annualized capital cost for generator new installation | million USD/GW |
| $o^{\text{fix}}_{n,i,r}$ | `gfoc(N,I,R)` | Fixed O&M cost of generator | million USD/GW |
| $c^{\text{replace}}_{n,i,r}$ | `garc(N,I,R)` | Annualized replacement cost of generator | million USD/GW |
| $o^{\text{variable}}_{n,i,r}$ | `gvoc(N,I,R)` | Variable O&M cost of generator | million USD/GWh |
| $c^{\text{new}}_{l}$ | `facc(L)` | Annualized capital cost for link new installation | million USD/GW |
| $o^{\text{fix}}_{l}$ | `ffoc(L)` | Fixed O&M cost of link | million USD/GW |
| $c^{\text{replace}}_{l}$ | `farc(L)` | Annualized replacement cost of link | million USD/GW |
| $o^{\text{variable}}_{l}$ | `fvoc(L)` | Variable O&M cost of link | million USD/GWh |
| $c^{\text{new}}_{n,i,s}$ | `eacc(N,I,S)` | Annualized capital cost for storage new installation | million USD/GWh |
| $o^{\text{fix}}_{n,i,s}$ | `efoc(N,I,S)` | Fixed O&M cost of storage | million USD/GWh |
| $c^{\text{replace}}_{n,i,s}$ | `earc(N,I,S)` | Annualized replacement cost of storage | million USD/GWh |
| $\omega$ | `wt(T)` | Timeslice weight (hours in each timeslice) | hours |

</details>

<details markdown="1">
<summary><b>Decision variables used in Total system cost</b></summary>

| Variable | Symbol | Description | Unit |
|----------|--------|-------------|------|
| $VTC$ | `VTC` | Total system cost to be minimized | million USD |
| $VR_{n,i,r}$ | `VGR(N,I,R)` | New installation of generator | GW |
| $VS_{n,i,r}$ | `VGS(N,I,R)` | Installed capacity of generator | GW |
| $VP_{n,i,r}$ | `VGP(N,I,R)` | Replacement capacity of generator | GW |
| $VX_{n,i,r,t}$ | `VG(N,I,R,T)` | Power output of generator | GW |
| $VR_{l}$ | `VFR(L)` | New installation of link | GW |
| $VS_{l}$ | `VFS(L)` | Installed capacity of link | GW |
| $VP_{l}$ | `VFP(L)` | Replacement capacity of link | GW |
| $VX^{+}_{l,t}$ | `VFF(L,T)` | Forward energy flow via link | GW |
| $VX^{-}_{l,t}$ | `VFB(L,T)` | Backward energy flow via link | GW |
| $VR_{n,i,s}$ | `VER(N,I,S)` | New installation of storage | GWh |
| $VS_{n,i,s}$ | `VES(N,I,S)` | Installed energy capacity of storage | GWh |
| $VP_{n,i,s}$ | `VEP(N,I,S)` | Replacement capacity of storage | GWh |

</details>

### Annualization of capital costs

New installation and replacement costs are annualized based on technology discount rate and lifetime. For generators, the annualized costs (${c^{\text{new}}}_{n,i,r}$ and ${c^{\text{replace}}}_{n,i,r}$) are computed from initial capital costs (${b^{\text{new}}}_{n,i,r}$ and ${b^{\text{replace}}}_{n,i,r}$) using discount rate $\alpha_{n, i, r}$ and lifetime $\tau_{n, i, r}$:

$$
{c^{\text{new}}}_{n, i, r} = {b^{\text{new}}}_{n, i, r} \cdot \frac{\alpha_{n, i, r} \cdot \left(1 + \alpha_{n, i, r}\right)^{\tau_{n, i, r}}}
{\left(1 + \alpha_{n, i, r}\right)^{\tau_{n, i, r}}-1} \cdot \Delta y
$$

$$
{c^{\text{replace}}}_{n, i, r} = {b^{\text{replace}}}_{n, i, r} \cdot \frac{\alpha_{n, i, r} \cdot \left(1 + \alpha_{n, i, r}\right)^{\tau_{n, i, r}}}
{\left(1 + \alpha_{n, i, r}\right)^{\tau_{n, i, r}}-1} \cdot \Delta y
$$

<details markdown="1">
<summary><b>Parameters used in Annualization of capital costs</b></summary>

| Parameter | Symbol | Description | Unit |
|-----------|--------|-------------|------|
| $b^{\text{new}}_{n,i,r}$ | `gcc(N,I,R)` | Initial capital cost for generator new installation (before annualization) | million USD/GW |
| $b^{\text{replace}}_{n,i,r}$ | `grc(N,I,R)` | Initial capital cost for generator replacement (before annualization) | million USD/GW |
| $c^{\text{new}}_{n,i,r}$ | `gacc(N,I,R)` | Annualized capital cost for generator new installation | million USD/GW |
| $c^{\text{replace}}_{n,i,r}$ | `garc(N,I,R)` | Annualized replacement cost of generator | million USD/GW |
| $\alpha_{n,i,r}$ | `galp(N,I,R)` | Discount rate for generator | - |
| $\tau_{n,i,r}$ | `glt(N,I,R)` | Lifetime of generator | years |
| $\Delta y$ | `t_int` | Time interval between simulation years | years |

</details>

## Generator operation & capacity

### Minimum/Maximum operation of generator

The output of each generator is bounded by its installed capacity and technology-specific dispatch limits:

$$
{{\gamma^{\min}}}_{n, i, r, t} \cdot {VS}_{n, i, r} \leq {VX}_{n, i, r, t} \leq {{\gamma^{\max}}}_{n, i, r, t} \cdot {VS}_{n, i, r}
$$

<details markdown="1">
<summary><b>Parameters used in Minimum/Maximum operation</b></summary>

| Parameter | Symbol | Description | Unit |
|-----------|--------|-------------|------|
| $\gamma^{\min}_{n,i,r,t}$ | `gmn(N,I,R,T)` | Minimum output factor of generator | 0-1 |
| $\gamma^{\max}_{n,i,r,t}$ | `gmx(N,I,R,T)` | Maximum output factor (availability) | 0-1 |

</details>

<details markdown="1">
<summary><b>Decision variables used in Minimum/Maximum operation</b></summary>

| Variable | Symbol | Description | Unit |
|----------|--------|-------------|------|
| $VX_{n,i,r,t}$ | `VG(N,I,R,T)` | Power output of generator | GW |
| $VS_{n,i,r}$ | `VGS(N,I,R)` | Installed capacity of generator | GW |

</details>

!!! tip "Handling of curtailment"
    For non-dispatchable generators (wind/solar), ${VX}_{n,i,r,t}$ represents actual generation, which may be less than potential generation ${{\gamma^{\max}}}_{n,i,r,t} \cdot {VS}_{n,i,r}$ due to economic curtailment. The curtailed capacity is calculated in postprocessing as ${{\gamma^{\max}}}_{n,i,r,t} \cdot {VS}_{n,i,r} - {VX}_{n,i,r,t}$.

### Ramping constraints (optional)

Ramp-up and ramp-down constraints limit the rate of change of generator output between adjacent timeslices $(t, t')$:

$$
 -{{\Delta t}}\cdot {{\rho^{\text{down}}}}_{n,i,r}\cdot {{\gamma^{\max}}}_{n,i,r,t}\cdot {VS}_{n,i,r}
\leq {VX}_{n,i,r,t'}-{VX}_{n,i,r,t}
\leq {{\Delta t}}\cdot {{\rho^{\text{up}}}}_{n,i,r}\cdot {{\gamma^{\max}}}_{n,i,r,t}\cdot {VS}_{n,i,r}
$$

<details markdown="1">
<summary><b>Parameters used in Ramping constraints</b></summary>

| Parameter | Symbol | Description | Unit |
|-----------|--------|-------------|------|
| $\Delta t$ | `rt(T)` | Timeslice granularity (hours per timeslice) | hours |
| $\rho^{\text{up}}_{n,i,r}$ | `gru(N,I,R)` | Ramp-up rate coefficient | - |
| $\rho^{\text{down}}_{n,i,r}$ | `grd(N,I,R)` | Ramp-down rate coefficient | - |
| $\gamma^{\max}_{n,i,r,t}$ | `gmx(N,I,R,T)` | Maximum output factor | 0-1 |

</details>

<details markdown="1">
<summary><b>Decision variables used in Ramping constraints</b></summary>

| Variable | Symbol | Description | Unit |
|----------|--------|-------------|------|
| $VX_{n,i,r,t}$ | `VG(N,I,R,T)` | Power output of generator | GW |
| $VS_{n,i,r}$ | `VGS(N,I,R)` | Installed capacity of generator | GW |

</details>

### Generation share constraints

The model can impose minimum/maximum generation shares between generator groups. For two groups $(MR_0,MR_1)$:

$$
{{\theta^{\min}}}_{n,i,MR_0,MR_1}\cdot \sum_{r\in MR_1}\sum_{t}{VX}_{n,i,r,t}
\leq \sum_{r\in MR_0}\sum_{t}{VX}_{n,i,r,t}
\leq {{\theta^{\max}}}_{n,i,MR_0,MR_1}\cdot \sum_{r\in MR_1}\sum_{t}{VX}_{n,i,r,t}
$$

<details markdown="1">
<summary><b>Parameters used in Generation share constraints</b></summary>

| Parameter | Symbol | Description | Unit |
|-----------|--------|-------------|------|
| $\theta^{\min}_{n,i,MR_0,MR_1}$ | `gmmn(N,I,MR0,MR1)` | Minimum generation share coefficient | - |
| $\theta^{\max}_{n,i,MR_0,MR_1}$ | `gmmx(N,I,MR0,MR1)` | Maximum generation share coefficient | - |

</details>

<details markdown="1">
<summary><b>Decision variables used in Generation share constraints</b></summary>

| Variable | Symbol | Description | Unit |
|----------|--------|-------------|------|
| $VX_{n,i,r,t}$ | `VG(N,I,R,T)` | Power output of generator | GW |

</details>

### Capacity bounds for generator

The installed capacity of generators is constrained by technology-specific limits:

$$
{{\lambda^{\min}}}_{n, i, r} \leq {VS}_{n, i, r} \leq {{\lambda^{\max}}}_{n, i, r}
$$

New installation of generators is also bounded:

$$
{{\nu^{\min}}}_{n, i, r} \leq {VR}_{n, i, r} \leq {{\nu^{\max}}}_{n, i, r}
$$

<details markdown="1">
<summary><b>Parameters used in Capacity bounds</b></summary>

| Parameter | Symbol | Description | Unit |
|-----------|--------|-------------|------|
| $\lambda^{\min}_{n,i,r}$ | `gsmn(N,I,R)` | Minimum installed capacity | GW |
| $\lambda^{\max}_{n,i,r}$ | `gsmx(N,I,R)` | Maximum installed capacity | GW |
| $\nu^{\min}_{n,i,r}$ | `grmn(N,I,R)` | Minimum new installation per year | GW |
| $\nu^{\max}_{n,i,r}$ | `grmx(N,I,R)` | Maximum new installation per year | GW |

</details>

<details markdown="1">
<summary><b>Decision variables used in Capacity bounds</b></summary>

| Variable | Symbol | Description | Unit |
|----------|--------|-------------|------|
| $VS_{n,i,r}$ | `VGS(N,I,R)` | Installed capacity of generator | GW |
| $VR_{n,i,r}$ | `VGR(N,I,R)` | New installation of generator | GW |

</details>


## Link operation & capacity

### Link flow operation

Energy flows via each link are bounded by installed capacity and dispatch limits. Forward and backward flows are treated separately to represent lossy transport[^1]:

$$
{{\gamma^{\min}_{+}}}_{l, t} \cdot {VS}_{l} \leq {VX}^{+}_{l, t} \leq {{\gamma^{\max}_{+}}}_{l, t} \cdot {VS}_{l}
$$

$$
{{\gamma^{\min}_{-}}}_{l, t} \cdot {VS}_{l} \leq {VX}^{-}_{l, t} \leq {{\gamma^{\max}_{-}}}_{l, t} \cdot {VS}_{l}
$$

Fixed flows can be represented by setting bounds equal:

$$
{VX}^{+}_{l,t} = {{\gamma^{\text{fix}}_{+}}}_{l,t}\cdot {VS}_{l},\qquad {VX}^{-}_{l,t} = {{\gamma^{\text{fix}}_{-}}}_{l,t}\cdot {VS}_{l}
$$

<details markdown="1">
<summary><b>Parameters used in Link flow operation</b></summary>

| Parameter | Symbol | Description | Unit |
|-----------|--------|-------------|------|
| $\gamma^{\min}_{+,l,t}$ | `ffmn(L,T)` | Minimum forward flow factor | 0-1 |
| $\gamma^{\max}_{+,l,t}$ | `ffmx(L,T)` | Maximum forward flow factor | 0-1 |
| $\gamma^{\min}_{-,l,t}$ | `fbmn(L,T)` | Minimum backward flow factor | 0-1 |
| $\gamma^{\max}_{-,l,t}$ | `fbmx(L,T)` | Maximum backward flow factor | 0-1 |
| $\gamma^{\text{fix}}_{+,l,t}$ | `fffx(L,T)` | Fixed forward flow factor | 0-1 |
| $\gamma^{\text{fix}}_{-,l,t}$ | `fbfx(L,T)` | Fixed backward flow factor | 0-1 |

</details>

<details markdown="1">
<summary><b>Decision variables used in Link flow operation</b></summary>

| Variable | Symbol | Description | Unit |
|----------|--------|-------------|------|
| $VX^{+}_{l,t}$ | `VFF(L,T)` | Forward energy flow via link | GW |
| $VX^{-}_{l,t}$ | `VFB(L,T)` | Backward energy flow via link | GW |
| $VS_{l}$ | `VFS(L)` | Installed capacity of link | GW |

</details>

!!! note "Bidirectional links and Unidirectional links"
    Links are classified as either  **bidirectional links** (e.g., inverter) or **unidirectional links** (e.g., electrolyser). For unidirectional links, ${{\gamma^{\max}_{-}}}_{l,t} = 0$, restricting energy flow to one direction only.

### Link capacity bounds

Link installed capacity is constrained by:

$$
{{\lambda^{\min}}}_{l} \leq {VS}_{l} \leq {{\lambda^{\max}}}_{l}
$$

and new installation by:

$$
{{\nu^{\min}}}_{l} \leq {VR}_{l} \leq {{\nu^{\max}}}_{l}
$$

<details markdown="1">
<summary><b>Parameters used in Link capacity bounds</b></summary>

| Parameter | Symbol | Description | Unit |
|-----------|--------|-------------|------|
| $\lambda^{\min}_{l}$ | `fsmn(L)` | Minimum link capacity | GW |
| $\lambda^{\max}_{l}$ | `fsmx(L)` | Maximum link capacity | GW |
| $\nu^{\min}_{l}$ | `frmn(L)` | Minimum new installation per year | GW |
| $\nu^{\max}_{l}$ | `frmx(L)` | Maximum new installation per year | GW |

</details>

<details markdown="1">
<summary><b>Decision variables used in Link capacity bounds</b></summary>

| Variable | Symbol | Description | Unit |
|----------|--------|-------------|------|
| $VS_{l}$ | `VFS(L)` | Installed capacity of link | GW |
| $VR_{l}$ | `VFR(L)` | New installation of link | GW |

</details>

### Link contribution share constraints

The model can impose constraints on the contribution of link groups to nodal flows. Share constraints for two link groups $(ML2_0,ML2_1)$ based on their net nodal injections/withdrawals are:

$$
\theta^{\min}_{n,i,ML2_0,ML2_1} \cdot \sum_{l\in ML2_1}\sum_{t}\Big(-{k^{+}}_{n,i,l}\cdot {VX}^{+}_{l,t}+{k^{-}}_{n,i,l}\cdot {VX}^{-}_{l,t}\Big)
\leq \sum_{l\in ML2_0}\sum_{t}\Big(-{k^{+}}_{n,i,l}\cdot {VX}^{+}_{l,t}+{k^{-}}_{n,i,l}\cdot {VX}^{-}_{l,t}\Big)
\leq \theta^{\max}_{n,i,ML2_0,ML2_1} \cdot \sum_{l\in ML2_1}\sum_{t}\Big(-{k^{+}}_{n,i,l}\cdot {VX}^{+}_{l,t}+{k^{-}}_{n,i,l}\cdot {VX}^{-}_{l,t}\Big)
$$

The incidence matrix coefficients ${k^{+}}_{n,i,l}$ and ${k^{-}}_{n,i,l}$ encode both network topology (connectivity) and link efficiency for forward and backward directions. These coefficients are constructed from the link efficiency parameter and topology information:

- **Forward flow incidence matrix**: ${k^{+}}_{n,i,l}$ represents the contribution of forward flow via link $l$ to node $(n,i)$
    - Source node (`N0`, `I0`): ${k^{+}}_{n_0,i_0,l} = 1$ (energy leaves the node)
    - Destination node (`N1`, `I1`): ${k^{+}}_{n_1,i_1,l} = -\eta_l$
    - Unconnected nodes: ${k^{+}}_{n,i,l} = 0$

- **Backward flow incidence matrix**: ${k^{-}}_{n,i,l}$ represents the contribution of backward flow via link $l$ to node $(n,i)$
    - Source node (`N0`, `I0`): ${k^{-}}_{n_0,i_0,l} = \eta_l$
    - Destination node (`N1`, `I1`): ${k^{-}}_{n_1,i_1,l} = -1$ (energy arrives at the node)
    - Unconnected nodes: ${k^{-}}_{n,i,l} = 0$

<details markdown="1">
<summary><b>Parameters used in Link contribution share constraints</b></summary>

| Parameter | Symbol | Description | Unit |
|-----------|--------|-------------|------|
| $k^{+}_{n,i,l}$ | `kff(N,I,L)` | Incidence matrix for forward flow | - |
| $k^{-}_{n,i,l}$ | `kbf(N,I,L)` | Incidence matrix for backward flow | - |
| $\theta^{\min}_{n,i,ML2_0,ML2_1}$ | `fmmn(N,I,ML2_0,ML2_1)` | Minimum link group share coefficient | - |
| $\theta^{\max}_{n,i,ML2_0,ML2_1}$ | `fmmx(N,I,ML2_0,ML2_1)` | Maximum link group share coefficient | - |

</details>

<details markdown="1">
<summary><b>Decision variables used in Link contribution share constraints</b></summary>

| Variable | Symbol | Description | Unit |
|----------|--------|-------------|------|
| $VX^{+}_{l,t}$ | `VFF(L,T)` | Forward energy flow via link | GW |
| $VX^{-}_{l,t}$ | `VFB(L,T)` | Backward energy flow via link | GW |

</details>

## Storage operation & capacity

### Minimum/Maximum operation of storage

Storage operation involves two decision variables with different constraints: charge/discharge (${VX}_{n,i,s,t}$) and energy level (${VE}_{n,i,s,t}$).

$$
-\infty < {VX}_{n, i, s, t} < +\infty
$$

$$
{{\xi^{\min}}}_{n, i, s, t} \cdot {VS}_{n, i, s} \leq {VE}_{n, i, s, t} \leq {{\xi^{\max}}}_{n, i, s, t} \cdot {VS}_{n, i, s}
$$

<details markdown="1">
<summary><b>Parameters used in Minimum/Maximum operation of storage</b></summary>

| Parameter | Symbol | Description | Unit |
|-----------|--------|-------------|------|
| $\xi^{\min}_{n,i,s,t}$ | `emn(N,I,S,T)` | Minimum energy level factor | 0-1 |
| $\xi^{\max}_{n,i,s,t}$ | `emx(N,I,S,T)` | Maximum energy level factor | 0-1 |

</details>

<details markdown="1">
<summary><b>Decision variables used in Minimum/Maximum operation of storage</b></summary>

| Variable | Symbol | Description | Unit |
|----------|--------|-------------|------|
| $VE_{n,i,s,t}$ | `VE(N,I,S,T)` | Energy level (state-of-charge) of storage, constrained by $\xi^{\min}/\xi^{\max}$ | GWh |
| $VS_{n,i,s}$ | `VES(N,I,S)` | Installed energy capacity of storage | GWh |
| $VX_{n,i,s,t}$ | `VH(N,I,S,T)` | Charge/discharge power (unbounded: $-\infty$ to $+\infty$; positive: discharge) | GW |

</details>

### Storage capacity bounds

The installed energy capacity of storage is constrained by minimum and maximum bounds:

$$
{{\lambda^{\min}}}_{n, i, s} \leq {VS}_{n, i, s} \leq {{\lambda^{\max}}}_{n, i, s}
$$

The new installaion of storage is constrainted by the minimum and maximum capacity.

$$
{{\nu^{\min}}}_{n, i, s} \leq {VR}_{n, i, s} \leq {{\nu^{\max}}}_{n, i, s}
$$

<details markdown="1">
<summary><b>Parameters used in Storage capacity bounds</b></summary>

| Parameter | Symbol | Description | Unit |
|-----------|--------|-------------|------|
| $\lambda^{\min}_{n,i,s}$ | `esmn(N,I,S)` | Minimum installed energy capacity | GWh |
| $\lambda^{\max}_{n,i,s}$ | `esmx(N,I,S)` | Maximum installed energy capacity | GWh |
| $\nu^{\min}_{n,i,s}$ | `ermn(N,I,S)` | Minimum new installation | GWh |
| $\nu^{\max}_{n,i,s}$ | `ermx(N,I,S)` | Maximum new installation | GWh |

</details>

<details markdown="1">
<summary><b>Decision variables used in Storage capacity bounds</b></summary>

| Variable | Symbol | Description | Unit |
|----------|--------|-------------|------|
| $VS_{n,i,s}$ | `VES(N,I,S)` | Installed energy capacity of storage | GWh |
| $VR_{n,i,s}$ | `VER(N,I,S)` | New installation of storage | GWh |

</details>

### Time-series state of charge of storage

The energy level (SoC) at each timeslice depends on the energy level at the previous timeslice, self-discharge, and charge/discharge operations during the current timeslice:

$$
{VE}_{n, i, s, t} =  {{\eta}_{n, i, s}}^{{{\Delta t}}} \cdot {VE}_{n, i, s, t-1} - {{\Delta t}} \cdot {VX}_{n, i, s, t}
$$

<details markdown="1">
<summary><b>Parameters used in Time-series state of charge of storage</b></summary>

| Parameter | Symbol | Description | Unit |
|-----------|--------|-------------|------|
| $\eta_{n,i,s}$ | `eeta(N,I,S)` | Self-discharge rate per hour | - |
| $\Delta t$ | `rt(T)` | Timeslice granularity (hours per timeslice) | hours |

</details>

<details markdown="1">
<summary><b>Decision variables used in Time-series state of charge of storage</b></summary>

| Variable | Symbol | Description | Unit |
|----------|--------|-------------|------|
| $VE_{n,i,s,t}$ | `VE(N,I,S,T)` | Energy level (state-of-charge) of storage, bounded by $\xi^{\min}/\xi^{\max}$ | GWh |
| $VX_{n,i,s,t}$ | `VH(N,I,S,T)` | Charge/discharge power (unbounded; positive: discharge, negative: charging) | GW |

</details>

!!! note "Cyclic boundary condition"
    
    This constraint enforces that the energy level at the first timeslice ($t_1$) equals the energy level at the last timeslice ($t_{|T|}$), ensuring cyclic operation across the modeled period. This prevents the model from arbitrarily draining or filling storage between simulation years, maintaining inter-annual consistency.

    $$
    {VE}_{n, i, s, t_1} = {VE}_{n, i, s, t_{|T|}}
    $$

### Power-Energy capacity of storage

Some storage technologies (e.g., pumped hydro storage) relate power capacity (e.g., pump) to energy capacity (e.g., reservoir). For each link $l$:

$$
{VS}_{l} = \sum_{(n,i,s)\in ISL_l} {{\chi}_{n,i,s,l}}\cdot {VS}_{n,i,s}
$$

where $ISL_l$ is the set of storage technology instances $(n,i,s)$ related to link $l$.

<details markdown="1">
<summary><b>Parameters used in Power-Energy capacity of storage</b></summary>

| Parameter | Symbol | Description | Unit |
|-----------|--------|-------------|------|
| $\chi_{n,i,s,l}$ | `ecrt(N,I,S,L)` | Power-to-energy ratio | GW/GWh |

</details>

<details markdown="1">
<summary><b>Decision variables used in Power-Energy capacity of storage</b></summary>

| Variable | Symbol | Description | Unit |
|----------|--------|-------------|------|
| $VS_{l}$ | `VFS(L)` | Installed capacity of link | GW |
| $VS_{n,i,s}$ | `VES(N,I,S)` | Installed energy capacity of storage | GWh |

</details>

## Nodal power balance

The nodal power balance ensures supply equals demand at each node for each timeslice group:

$$
\sum_{r}\sum_{t\in MT}{VX}_{n,i,r,t}+\sum_{s}\sum_{t\in MT}{VX}_{n,i,s,t}
-\sum_{l}\sum_{t\in MT}{k^{+}}_{n,i,l}{VX}^{+}_{l,t}+\sum_{l}\sum_{t\in MT}{k^{-}}_{n,i,l}{VX}^{-}_{l,t}
= \sum_{t\in MT}{d}_{n,i,t}+\delta_{n,i,MT}
$$

**Supply side (left-hand side):**

- **Generator output**: $\sum_{r}\sum_{t\in MT}{VX}_{n,i,r,t}$ aggregates generation from all generators
- **Storage discharge/charge**: $\sum_{s}\sum_{t\in MT}{VX}_{n,i,s,t}$ (positive: discharge, negative: charge)
- **Net link flows**: $-\sum_{l}\sum_{t\in MT}{k^{+}}_{n,i,l}{VX}^{+}_{l,t}+\sum_{l}\sum_{t\in MT}{k^{-}}_{n,i,l}{VX}^{-}_{l,t}$ represents net energy imports (incidence matrix ${k^{+}}_{n,i,l}$ and ${k^{-}}_{n,i,l}$ encode network topology and efficiency)

**Demand side (right-hand side):**

- **Exogenous demand**: $\sum_{t\in MT}{d}_{n,i,t}$ is the time-varying power demand
- **Slack variable**: $\delta_{n,i,MT}$ allows residual imbalance (typically penalized in objective function; zero in normal solutions)

<details markdown="1">
<summary><b>Parameters used in Nodal power balance</b></summary>

| Parameter | Symbol | Description | Unit |
|-----------|--------|-------------|------|
| ${d}_{n,i,t}$ | `dmd(N,I,T)` | Exogenous demand for power | GW |
| $k^{+}_{n,i,l}$ | `kff(N,I,L)` | Incidence matrix for forward link flows | - |
| $k^{-}_{n,i,l}$ | `kbf(N,I,L)` | Incidence matrix for backward link flows | - |

</details>

<details markdown="1">
<summary><b>Decision variables used in Nodal power balance</b></summary>

| Variable | Symbol | Description | Unit |
|----------|--------|-------------|------|
| $VX_{n,i,r,t}$ | `VG(N,I,R,T)` | Power output of generator | GW |
| $VX_{n,i,s,t}$ | `VH(N,I,S,T)` | Storage charge/discharge power | GW |
| $VX^{+}_{l,t}$ | `VFF(L,T)` | Forward energy flow via link | GW |
| $VX^{-}_{l,t}$ | `VFB(L,T)` | Backward energy flow via link | GW |
| $\delta_{n,i,MT}$ | `RES_NPB(N,I,MT)` | Slack variable for nodal power balance | GW |

</details>

!!! tip "Timeslice aggregation"
    The constraint aggregates over timeslice group $MT$ rather than individual timeslices $t$, allowing flexibility in temporal resolution. Common groupings:
    
    - $MT$ = {all timeslices}: annual energy balance
    - $MT$ = {single timeslice $t$}: hourly or sub-hourly balances
    
    This design enables the model to enforce balances at different temporal scales depending on the application.

## Technology stock

Installed capacity comprises new installations, replacements, and existing vintage stock:

$$
{VS}_{n,i,r} = ({VR}_{n,i,r}+{VP}_{n,i,r})\cdot \Delta y + \sum_{y}{sc}_{n,i,r,y}
$$

Replacements are limited by replaceable stock:

$$
{sr}_{n,i,r} \geq {VP}_{n,i,r}
$$

Vintage stocks $sc_{n,i,r,y}$ are tracked by year $y$. Replaceable stock ${sr}_{n,i,r}$ comprises vintages that have reached lifetime $\tau_{n,i,r}$ (age $a_y = y^* - y$, where $y^*$ is simulation year), scaled to the multi-year step:

$$
{sr}_{n,i,r} = \frac{\sum_{y\;:\; a_y \geq {\tau}_{n,i,r}} {sc}_{n,i,r,y}}{\Delta y}
$$

where the subscripts $(n,i,r)$, $(l)$, and $(n,i,s)$ apply analogously to generators, links, and storage.

<details markdown="1">
<summary><b>Parameters used in Technology stock</b></summary>

| Parameter | Symbol | Description | Unit |
|-----------|--------|-------------|------|
| $sc_{n,i,r,y}$ | `gssc(N,I,R,Y)` | Stock by vintage year (generators/links/storage) | GW / GW / GWh |
| $sr_{n,i,r}$ | `gssr(N,I,R)` | Replaceable stock (generators/links/storage) | GW / GW / GWh |
| $\tau_{n,i,r}$ | `glt(N,I,R)` | Lifetime of generator | years |
| $\Delta y$ | `t_int` | Time interval between simulation years | years |

</details>

<details markdown="1">
<summary><b>Decision variables used in Technology stock</b></summary>

| Variable | Symbol | Description | Unit |
|----------|--------|-------------|------|
| $VS_{n,i,r}$ | `VGS(N,I,R)` | Installed capacity of generator | GW |
| $VR_{n,i,r}$ | `VGR(N,I,R)` | New installation of generator | GW |
| $VP_{n,i,r}$ | `VGP(N,I,R)` | Replacement capacity of generator | GW |
| $VS_{l}$ | `VFS(L)` | Installed capacity of link | GW |
| $VR_{l}$ | `VFR(L)` | New installation of link | GW |
| $VP_{l}$ | `VFP(L)` | Replacement capacity of link | GW |
| $VS_{n,i,s}$ | `VES(N,I,S)` | Installed energy capacity of storage | GWh |
| $VR_{n,i,s}$ | `VER(N,I,S)` | New installation of storage | GWh |
| $VP_{n,i,s}$ | `VEP(N,I,S)` | Replacement capacity of storage | GWh |

</details>

## Energy supply

Energy supply tracks primary energy consumption or fuel use:

$$
{VP}_{n, i, k} = \sum_{r, t} \left[ {{\omega}} \cdot {{\eta}_{n, i, r, k}} \cdot {VX}_{n, i, r, t} \right]
$$

Energy supply is constrained by minimum/maximum bounds or fixed values for aggregated region-sector groups ($ME$) and energy types ($MK$):

$$
{p^{\min}}_{ME,MK} \leq \sum_{(n,i)\in ME}\sum_{k\in MK} {VP}_{n,i,k}
\leq {p^{\max}}_{ME,MK}
$$

$$
{p^{\text{fix}}}_{ME,MK} = \sum_{(n,i)\in ME}\sum_{k\in MK} {VP}_{n,i,k}
$$

<details markdown="1">
<summary><b>Parameters used in Energy supply</b></summary>

| Parameter | Symbol | Description | Unit |
|-----------|--------|-------------|------|
| $\eta_{n,i,r,k}$ | `geta(N,I,R,K)` | Conversion coefficient (energy type $k$ per unit output) | - |
| $\omega$ | `wt(T)` | Timeslice weight | hours |
| $p^{\min}_{ME,MK}$ | `pmin(ME,MK)` | Minimum annual energy supply for aggregated group | GWh |
| $p^{\max}_{ME,MK}$ | `pmax(ME,MK)` | Maximum annual energy supply for aggregated group | GWh |
| $p^{\text{fix}}_{ME,MK}$ | `pfix(ME,MK)` | Fixed annual energy supply for aggregated group | GWh |

</details>

<details markdown="1">
<summary><b>Decision variables used in Energy supply</b></summary>

| Variable | Symbol | Description | Unit |
|----------|--------|-------------|------|
| $VP_{n,i,k}$ | `VP(N,I,K)` | Annual energy supply of type $k$ | GWh |
| $VX_{n,i,r,t}$ | `VG(N,I,R,T)` | Power output of generator | GW |

</details>

## Emission

Emissions are computed using emission factors:

$$
{VQ}_{n,i,g} = \sum_{k} {{\epsilon}_{n,i,k,g}}\cdot {VP}_{n,i,k}
$$

Emission caps limit total emissions for aggregated regions ($MQ$) and gas types ($MG$):

$$
\sum_{(n,i)\in MQ}\sum_{g\in MG} {VQ}_{n,i,g} \leq {q^{\max}}_{MQ,MG}
$$

<details markdown="1">
<summary><b>Parameters used in Emission</b></summary>

| Parameter | Symbol | Description | Unit |
|-----------|--------|-------------|------|
| $\epsilon_{n,i,k,g}$ | `gas(N,I,K,G)` | Emission factor for energy type $k$ and gas $g$ | e.g., MtCO₂/GWh |
| $q^{\max}_{MQ,MG}$ | `qmax(MQ,MG)` | Maximum annual emissions for aggregated group | e.g., MtCO₂ |

</details>

<details markdown="1">
<summary><b>Decision variables used in Emission</b></summary>

| Variable | Symbol | Description | Unit |
|----------|--------|-------------|------|
| $VQ_{n,i,g}$ | `VQ(N,I,G)` | Annual emissions of type $g$ | e.g., MtCO₂ |
| $VP_{n,i,k}$ | `VP(N,I,K)` | Annual energy supply of type $k$ | GWh |

</details>

[^1]: [Neumann, F., Hagenmeyer, V. & Brown, T. Assessments of linear power flow and transmission loss approximations in coordinated capacity expansion problems. Appl. Energy 314, 118859 (2022).](https://doi.org/10.1016/j.apenergy.2022.118859)
