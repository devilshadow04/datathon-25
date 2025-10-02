# Datathon 2025 – Predicting Supply Chain Disruptions

This repository contains our project for the **2025 SUDATA × SUBAA Datathon**.  
Challenge: analyze a real-world logistics dataset and deliver **data-driven strategies** to reduce disruption and improve efficiency across a multi-modal supply chain.

## Business Context
GRB Co., a consumer goods firm, faces rising logistics costs, delays, and strained client relationships due to operational inefficiencies.  
Our goal: identify **disruption drivers** and recommend **actionable interventions** to make the supply chain more reliable and cost-effective.

## Problem Statement
- **Objective:** Predict and mitigate supply chain disruptions.  
- **Primary Target:** `disruption_likelihood_score` (0–1).  
- **Context:** Multi-modal operations (**Truck, Rail, Drone**) with different risk profiles.

## Dataset
- **Region & Period:** Southern California, **2021–2024** (hourly).  
- **Modalities:** Trucks, Rail, Drones (anonymised).  
- **Features:** GPS, congestion, weather, IoT temperature, warehouse ops, supplier reliability, driver behaviour, fatigue, etc.  
- **Targets:** `disruption_likelihood_score`, `delay_probability`, `risk_classification`, `delivery_time_deviation`.  
*(See `Metadata.pdf` for details.)*

## Methodology

### 1. Transport Mode Identification (Unlabelled → Clusters)

The raw data does not include transport mode labels. We infer them via **KMeans (k=3)** on operational signals:

- `fuel_consumption_rate` (efficiency differences)  
- `traffic_congestion_level` (road exposure → trucks)  
- `eta_variation_hours` (arrival variability)  
- `delay_probability` (likelihood of disruption)

![transport_clustering](./transport_cluster.png)

**Steps:**  
1. Median imputation for missing values  
2. Standardise features  
3. Run KMeans (k = 3)  
4. Map clusters → {Truck, Rail, Drone} via mean-profile inspection  
5. Pairplot visualisation to validate separation

In essence, the clusters are defined by a trade-off: Trucks are characterized by high fuel use and high traffic; Rail by very high fuel use and high ETA variation; and Drones by low fuel use, low traffic, and low ETA variation.

### 2. Geospatial View (Southern California)
We examined spatial patterns of disruption:
1. Define a SoCal polygon boundary (approximate region).  
2. Convert GPS (`vehicle_gps_latitude`, `vehicle_gps_longitude`) to a GeoDataFrame.  
3. Filter shipments within the polygon.  
5. **Interactive Folium map:**  
   - Point **colour** = `disruption_likelihood_score`  
   - **Size** scales with the score  
   - **Tooltips** show exact values  
This reveals **corridor-level hotspots** for targeted interventions.

![southcal_map](./southcal_map.png)

### 3. Truck-Mode Modelling (AIC-Selected OLS)

**Why focus on Truck?**  
The truck cluster shows **higher cost exposure and variability**, making it the **best lever** for near-term savings and reliability gains.

**Target:** `disruption_likelihood_score`  
**Model:** OLS with AIC forward selection  

**Key model results (n = 98):**
- **R² = 0.184**, **Adj. R² = 0.121** (signal present but modest → noisy environment)  
- Model **F-test p = 0.00886** (joint significance)  
- Coefficients (signs & p-values):
  - `iot_temperature` **−0.0552**, p=0.065 (cooler temps ↘ disruption; near-sig)  
  - `loading_unloading_time` **+0.0524**, p=0.080 (longer handling ↗ disruption; near-sig)  
  - `customs_clearance_time` **−0.0622**, **p=0.044** (stat. sig.; see note below)  
  - `fatigue_monitoring_score` **+0.0546**, p=0.071 (higher fatigue risk ↗ disruption; near-sig)  
  - `lead_time_days` **−0.0495**, p=0.094 (longer planned lead ↘ disruption; buffer effect)  
  - `driver_behavior_score` **−0.0471**, p=0.115 (safer driving ↘ disruption)  
  - `route_risk_level` **+0.0460**, p=0.127 (riskier routes ↗ disruption)

> **Interpretation notes:**  
> • `customs_clearance_time` shows a **negative** coefficient despite domain expectations, this likely reflects **process re-timing** (e.g. longer planned clearance used as **buffer** to stabilise downstream variability) or multicollinearity with other timing variables. We flag this for **operational review** and **spec sensitivity checks**.  


## Insights 
- **7 drivers** explain a meaningful share of disruption risk in a noisy environment.  
- Operational levers with strongest, actionable signals:  
  - **Dock operations** (`loading_unloading_time`)  
  - **Human factors** (`fatigue_monitoring_score`, `driver_behavior_score`)  
  - **Route planning** (`route_risk_level`)  
  - **Cold-chain control** (`iot_temperature`)  
- Planning variables (`lead_time_days`, `customs_clearance_time`) appear to act as **buffers**, worth re-engineering to reduce variability without sacrificing throughput.

## Business Impact (Recommended Actions)
1. **IoT Temperature Automation**  
   - Real-time alerts, thresholding, and audit trails → fewer spoilage claims; improved SLAs.
2. **Loading/Unloading Optimisation**  
   - Slotting, dock scheduling, and light automation → ↓ detention fees; **+5–10%** asset utilisation.
3. **Customs Digitisation & Pre-Clearance**  
   - Advance docs and broker SLAs → ↓ storage/inspection costs; lower variability.
4. **Fatigue & Behaviour Programs**  
   - Fatigue scoring + enforced rest + incentives → fewer incidents; higher driver retention.
5. **Risk-Aware Routing**  
   - Avoid high-risk corridors at peak; scenario rules for weather/port spikes.


## Limitations
- **Noise / Low R²:** Weak relationships between predictors and targets indicate noise and unobserved factors in the dataset.
- **Missing Transport Mode Labels**: Transport modes are not provided — currently inferred through k-means and domain knowledge, which adds uncertainty.
