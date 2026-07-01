# PVDeg Methodology and Feature Analysis

This document provides a detailed breakdown of the mathematical models and physical equations used in PVDeg, as well as clarifications on specific feature requests and modeling scopes.

## I. Degradation Methodology: Bottom-Up Traceback

### 1. LETID Outdoor Degradation Model
This model simulates the progression of Light and elevated Temperature Induced Degradation (LeTID) in field-deployed PV modules.

#### **Level 1: Final Performance Metrics**
*   **Normalized Power ($Pmp_{norm}$):**
    $$Pmp_{norm}(t) = \frac{P_{mp}(t)}{P_{mp}(t=0)}$$
*   **Maximum Power ($P_{mp}$):**
    $$P_{mp} = V_{oc} \cdot J_{sc} \cdot FF \cdot \text{Cell Area}$$
*   **Energy Loss:**
    $$\text{Loss} = 1 - \frac{\int_{0}^{T} Pmp_{norm}(t) dt}{T}$$
    *(Source: `pvdeg/letid.py` -> `calc_letid_outdoors`, `calc_energy_loss`)*

#### **Level 2: Device Physics (Electrical Parameters)**
*   **Open Circuit Voltage ($V_{oc}$):** Derived from carrier lifetime ($\tau$) using the Gray model for saturation current density ($J_0$):
    $$V_{oc} = \frac{kT}{q} \ln\left(\frac{J_{sc}}{J_0}\right)$$
*   **Short Circuit Current ($J_{sc}$):** Calculated by integrating the Collection Probability ($CP$) and the Optical Generation Profile ($G(z)$) through the wafer depth ($z$):
    $$J_{sc} = q \int_{0}^{W} G(z) \cdot CP(z, \tau) dz$$
*   **Fill Factor ($FF$):** Estimated using **Green’s empirical expression** based on normalized open-circuit voltage $v$:
    $$v = \frac{V_{oc}}{nkT/q}$$
    $$FF = \frac{v - \ln(v + 0.72)}{v + 1}$$
    *(Source: `pvdeg/collection.py` -> `calculate_jsc_from_tau_cp`, `pvdeg/letid.py` -> `calc_voc_from_tau`, `ff_green`)*

#### **Level 3: Defect Kinetics (3-State Model)**
The effective carrier lifetime ($\tau$) is determined by the fraction of defects in recombination-active **State B ($N_B$)**.
*   **Carrier Lifetime ($\tau$):**
    $$\tau(N_B) = \frac{\tau_0}{(\frac{\tau_0}{\tau_{deg}} - 1) \frac{N_B}{100} + 1}$$
*   **Defect State Evolution:** Transitions between State A (initial), State B (degraded), and State C (regenerated) governed by first-order kinetics:
    $$\frac{dN_A}{dt} = k_{ba} N_B x_{ba} - k_{ab} N_A x_{ab}$$
    $$\frac{dN_C}{dt} = k_{bc} N_B x_{bc} - k_{cb} N_C$$
    $$N_B = 100 - N_A - N_C$$
    *(Source: `pvdeg/letid.py` -> `tau_now`, `calc_letid_outdoors` loop)*

#### **Level 4: Reaction Rates and Acceleration**
*   **Arrhenius Rate Constants ($k_{ij}$):**
    $$k_{ij} = \nu_{ij} \cdot \exp\left(\frac{-E_{a,ij}}{k_B T}\right)$$
*   **Carrier Acceleration Factor ($x_{ij}$):**
    $$x_{ij} = \left(\frac{\Delta n}{\Delta n_{lit}}\right)^{\text{exponent}}$$
    *(Source: `pvdeg/letid.py` -> `k_ij`, `carrier_factor`, constants from `DegradationDatabase.json`)*

#### **Level 5: Physical Inputs**
*   **Cell Temperature ($T$):** Calculated via `pvlib` models (e.g., SAPM) using ambient temperature ($T_{amb}$), wind speed ($WS$), and POA irradiance ($G_{poa}$):
    $$T_{cell} = T_{amb} + G_{poa} \cdot \exp(a + b \cdot WS) + \frac{G_{poa}}{G_0} \cdot \Delta T$$
*   **Excess Carrier Density ($\Delta n$):** Determined by solving the steady-state diffusion equation for a given injection level.
*   **Plane-of-Array (POA) Irradiance:** Derived from weather components:
    $$G_{poa} = G_{beam} \cdot \cos(AOI) + G_{sky\_diffuse} + G_{ground\_diffuse}$$
    *(Source: NSRDB/PVGIS weather data, `pvdeg/temperature.py`, `pvdeg/spectral.py`)*

---

### 2. Van't Hoff Irradiance Degradation Model
This model relates field degradation to accelerated chamber testing.

#### **Level 1: Final Outputs**
*   **Acceleration Factor ($AF$):**
    $$AF = \frac{I_{chamber}^p}{\frac{1}{n} \sum_{t=1}^n \left( POA(t)^p \cdot T_f^{\frac{T(t) - T_{chamber}}{10}} \right)}$$
*   **Environment Characterization ($I_{wa}$):** The irradiance level required in a controlled environment to simulate field degradation.
    $$I_{wa} = \left(\frac{\sum (POA^p \cdot T_f^{\frac{T - T_{eq}}{10}})}{n}\right)^{\frac{1}{p}}$$
    *(Source: `pvdeg/degradation.py` -> `vantHoff_deg`, `IwaVantHoff`)*

#### **Level 2: Degradation Rate Equation**
Assumes degradation is driven by power-law irradiance and exponential temperature:
$$\text{Rate}(t) \propto POA(t)^p \cdot T_f^{\frac{T(t) - T_{ref}}{10}}$$
*   $p$: Irradiance fit parameter (default 0.5).
*   $T_f$: Thermal acceleration factor for every 10°C (default 1.41).

#### **Level 3: Equivalent Temperature ($T_{eq}$)**
Arrhenius-weighted equivalent temperature used for normalization:
$$T_{eq} = \frac{10}{\ln(T_f)} \cdot \ln\left(\frac{\sum T_f^{\frac{T}{10}}}{n}\right)$$

#### **Level 4: Physical Inputs**
*   **Operating Temperature ($T$):** Calculated via `pvdeg/temperature.py`.
*   **POA Irradiance:** Calculated from weather data and module orientation.

---

### 3. Monte Carlo Simulations with Arrhenius Models

PVDeg implements a Monte Carlo framework to propagate parameter uncertainties through degradation models.

#### **Correlated Sampling Methodology**
1.  **Statistical Definition:** Mean ($\mu$) and standard deviation ($\sigma$) are defined for each kinetic parameter (e.g., $E_a$, $R_0$).
2.  **Correlation Matrix ($C$):** Capture dependencies between parameters.
3.  **Cholesky Decomposition:** The matrix $C$ is decomposed into $L$ such that $C = LL^T$.
4.  **Sample Generation:**
    *   Generate independent standard normal variables $Z \sim N(0, 1)$.
    *   Apply correlation: $Y = L Z$.
    *   Scale and shift to physical units: $X = \mu + \sigma \cdot Y$.

#### **Simulation Pipeline**
Correlated samples are applied to the Arrhenius degradation function:
$$D_i = R_{0,i} \int [RH(t)]^n \cdot \exp\left(\frac{-E_{a,i}}{RT(t)}\right) [G(t)]^p dt$$
The resulting distribution of $D$ allows for uncertainty quantification and reliability assessment.
*(Source: `pvdeg/montecarlo.py` -> `generateCorrelatedSamples`, `simulate`)*

---

## II. Feature Clarifications

### 1. Temperature vs. `resistanceM`
PVDeg **does not** calculate temperature as a function of an electrical `resistanceM` parameter.
*   **Implementation:** All temperature models in `pvdeg/temperature.py` (e.g., SAPM, Pvsyst, Faiman, Ross) utilize **environmental variables** (irradiance, ambient temperature, wind speed) as primary inputs.
*   **Resistance Context:** Parameters like "encapsulant resistance" (`Rencap`) exist in symbolic models for mechanisms like Potential Induced Degradation (PID), but they serve as inputs to the **degradation rate** equations, not the thermal models.

### 2. Scope of Module Hotspot Detection
Detection of module hotspots is **outside the scope** of PVDeg.
*   **Focus:** PVDeg is a predictive simulation tool for long-term physical degradation kinetics. It models degradation at the cell or module level based on aggregate climatic stress.
*   **Limitation:** It is not a diagnostic tool for identifying localized thermal anomalies (hotspots) caused by cell mismatch, partial shading, or electrical faults. Such detection typically requires infrared thermography or real-time string-level monitoring data, which is not the data type processed by PVDeg.
