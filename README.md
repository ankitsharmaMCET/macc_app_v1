# India CCTS — Marginal Abatement Cost Curve (MACC) Builder

> **What this is:** A single‑page React app that lets you build firm‑level MACCs using sample or custom catalogs (fuels, raw materials, transport, waste, electricity). You can define measures via a Quick form or a **Template (catalog) wizard** with multi‑line drivers and financing. Multi‑firm is supported with local export/import.

---

## 2) Features Overview

- **Multi‑firm**: Create/rename/delete firms. Each firm has its own sectors, baselines, measures, currency (₹), carbon price, and catalogs. Data is stored in `localStorage` and can be **exported/imported** as JSON.
- **Catalog sources**: **Sample / Custom / Merged**. Merged = Sample overridden by Custom (matched by `name` or `state`).
- **Measure creation**:
  - **Quick**: name, sector, abatement (tCO₂), cost (₹/tCO₂).
  - **Template (catalog)**: multi‑line drivers (fuels, raw materials, transport, waste, electricity), drift (%/yr), adoption profile, other direct reductions, CAPEX (upfront/financed), OPEX/savings, and finance params.
- **MACC visualization**:
  - **Step chart** (rectangles) or **Quadratic Fit** (smooth MACC) with **R²**.
  - **Capacity** mode (cumulative tCO₂ on X, cost on Y) and **Intensity** mode (cumulative % reduction).
  - **Target slider** and **Budget calculator**.
  - **PNG export** of the chart.
- **Data management**:
  - Import/export **measures CSV**.
  - Import/export **catalogs** (CSV/JSON) per tab.
  - Per‑firm sectors & baselines editor.

---

## 3) Data Files (put under `public/data/`)

The app loads these endpoints at startup:
```
/data/measures.csv
/data/sectors.json
/data/baselines.json
/data/fuels.json
/data/raw.json
/data/transport.json
/data/waste.json
/data/electricity.json
```

### 3.1 `sectors.json`
```json
["Steel","Cement","Power","Firm – My Firm"]
```

### 3.2 `baselines.json`
A map from sector to baseline information.
```json
{
  "Steel":   { "production_label": "t steel", "annual_production": 1000000, "annual_emissions": 1800000 },
  "Cement":  { "production_label": "t clinker", "annual_production": 500000, "annual_emissions": 450000 },
  "Power":   { "production_label": "MWh", "annual_production": 100000, "annual_emissions": 71000 },
  "Firm – My Firm": { "production_label": "units", "annual_production": 0, "annual_emissions": 0 }
}
```

### 3.3 Catalogs (`fuels.json`, `raw.json`, `transport.json`, `waste.json`)
```json
[
  { "name": "Coal", "unit": "t", "price_per_unit_inr": 7000, "ef_tco2_per_unit": 2.4 },
  { "name": "Diesel", "unit": "L", "price_per_unit_inr": 95, "ef_tco2_per_unit": 0.00265 }
]
```
- **Aliases handled:** `price_per_unit` or `price` will be read as `price_per_unit_inr`; `ef_t_per_unit` or `ef_t` as `ef_tco2_per_unit`.

### 3.4 Electricity (`electricity.json`)
```json
[
  { "state": "India", "price_per_mwh_inr": 5000, "ef_tco2_per_mwh": 0.71 }
]
```
- **Aliases handled:** `price_per_mwh` or `price` → `price_per_mwh_inr`; `ef_t_per_mwh` or `ef_t` → `ef_tco2_per_mwh`.

### 3.5 Measures (`measures.csv`)
CSV columns:
```
id,name,sector,abatement_tco2,cost_per_tco2,selected,details
```
- `selected` defaults to `true` if omitted.
- `details` may be a JSON string produced by the Template wizard, e.g. including per‑year series and finance summary.

---

## 4) Using the App

1. **Manage Firms** → Create “My Firm” (start from Sample or Blank).  
   This seeds per‑firm `sectors`, `baselines`, `measures`, and **Custom catalogs** into localStorage.
2. Switch **Data source** (**Sample / Custom / Merged**) for the **Wizard**.
3. **Add measure** using **Quick** or **Template (catalog)**.
4. View the **MACC** in **Capacity** or **Intensity** mode, adjust **Target**, and **Export PNG**.
5. **Export** your firm JSON for backup or sharing; **Import** later to restore.

---

## 5) Mathematics (plain‑text, no LaTeX)

This section documents the formulas exactly as used in the code (see comments in `MACCApp.jsx`). We index **years** by i over the set Y = {2025, 2030, 2035, 2040, 2045, 2050}. The **BASE_YEAR** is 2025. Let Delta_i = year_i - 2025. The **adoption share** in year i is a_i in [0,1].

### 5.1 Driver lines: price & EF drift, quantities, and emissions

For each line `l` of type k in {fuel, raw, transport, waste} with **base price** P_l,0 (₹/unit) and **base EF** E_l,0 (tCO2 per unit):

- If the user overrides base price / EF, those values replace P_l,0 and E_l,0.
- With price drift gP_l and EF drift gE_l (fractions per year), the **effective year-i** values are:

```
P_l,i = P_l,0 * (1 + gP_l)^(Delta_i)
E_l,i = E_l,0 * (1 + gE_l)^(Delta_i)
```

If the Template series specifies a change in quantity (vs BAU) Delta_q_l,i (in the catalog unit), the **adopted** change is:

```
Delta_q_tilde_l,i = a_i * Delta_q_l,i
```

The **emissions delta** from line l in year i is:

```
Delta_EMIS_l,i = Delta_q_tilde_l,i * E_l,i
```

#### Electricity lines

For electricity line `e` with base price P_elec_e,0 (₹/MWh), price drift gP_e, base EF E_elec_e,0 (tCO2/MWh), EF drift gE_e, optional **per‑year EF override** override_E_elec_e,i:

```
P_elec_e,i = P_elec_e,0 * (1 + gP_e)^(Delta_i)
E_elec_e,i = override_E_elec_e,i (if provided), otherwise E_elec_e,0 * (1 + gE_e)^(Delta_i)
```

For electricity delta Delta_MWh_e,i, the adopted delta is:

```
Delta_MWh_tilde_e,i = a_i * Delta_MWh_e,i
Delta_EMIS_elec_e,i = Delta_MWh_tilde_e,i * E_elec_e,i
```

#### “Other direct reduction” series

The UI field “Other direct reduction (tCO2e)” is entered **positive for reductions**; in the code we apply:

```
Delta_EMIS_other_i = - (a_i * Other_i)
```

#### Total signed emissions delta and split

For year i:

```
Delta_EMIS_i
= sum_over_lines(Delta_EMIS_l,i)
+ sum_over_electricity(Delta_EMIS_elec_e,i)
+ Delta_EMIS_other_i
```

We report **non‑negative** reduction/addition magnitudes:

```
reduction_i = max(0, -Delta_EMIS_i)
addition_i  = max(0,  Delta_EMIS_i)
```

### 5.2 Cost stack and financing (₹ crore)

Driver spend for year i (converted to ₹ crore by dividing by 1e7):

```
driver_cr_i = (1/1e7) * [ sum_l(Delta_q_tilde_l,i * P_l,i)
                        + sum_e(Delta_MWh_tilde_e,i * P_elec_e,i) ]
```

Let `opex_cr_i`, `savings_cr_i`, `other_cr_i`, `capex_upfront_cr_i` be the series from the UI (₹ crore). Let financed CAPEX be `capex_financed_cr_i` with loan **interest** r_i and **tenure** n_i. The app uses the standard **annuity factor** AF(r,n):

```
AF(r,n) = r*(1+r)^n / [(1+r)^n - 1], if r != 0 and n > 0
AF(r,n) = 1/n,                         if r == 0 and n > 0
AF(r,n) = 0,                           if n == 0
```

Annualized financing (₹ crore) in year i:

```
financedAnnual_cr_i = capex_financed_cr_i * AF(r_i, n_i)
```

Important: Upfront CAPEX is **not annuitized**; it is treated as a cash outflow **only in the years where it is entered**.

The **net cost** (₹ crore) for MACC per‑ton calculations is:

```
net_cost_cr_i = driver_cr_i + opex_cr_i + other_cr_i - savings_cr_i
              + financedAnnual_cr_i + capex_upfront_cr_i
```

### 5.3 Cash flows, Carbon price, and Per‑ton costs

Let the **carbon price** be pCO2 (₹ per tCO2).  
Yearly **cash flow without CO2** (₹):

```
CF_noCO2_i = 1e7*(savings_cr_i - opex_cr_i - driver_cr_i - other_cr_i - financedAnnual_cr_i)
           - 1e7*capex_upfront_cr_i
```

We add CO2 **benefit only on reduced tonnes**:

```
CF_withCO2_i = CF_noCO2_i + pCO2 * reduction_i
```

Per‑ton **representative‑year** costs (the app picks a representative index i*, by default the first year with reduction_i > 0; fallback is 2035):

```
cost_noCO2_i* = (1e7 * net_cost_cr_i*) / reduction_i*   if reduction_i* > 0, else 0
cost_CO2_i*   = (1e7 * net_cost_cr_i* - pCO2 * reduction_i*) / reduction_i*   if reduction_i* > 0, else 0
```

### 5.4 Effective cost under carbon price

For each measure m with **input cost** c_m (₹/tCO2) and abatement A_m (tCO2), we display an **effective** cost for the MACC that adjusts for carbon price in one of two ways:

```
If the measure’s saved cost already included a carbon price pCO2_save:
    c_eff_m = c_m - (pCO2_now - pCO2_save)
Else:
    c_eff_m = c_m - pCO2_now
```

The **step MACC** uses bars of width A_m and height c_eff_m.

### 5.5 Capacity vs Intensity X‑axis

Let the **baseline emissions** for the selected sector be E_base (tCO2/yr). Sorting measures by increasing c_eff_m, define cumulative abatement:

```
S_j = sum_{m<=j} A_m
```

- **Capacity** mode X = S_j (tCO2).  
- **Intensity** mode X = 100 * S_j / E_base (percent).

### 5.6 Quadratic MACC fit and R^2

If **Quadratic Fit** is selected, we fit a polynomial y = a + b*x + c*x^2 to points (x_j, y_j) where x_j is cumulative abatement (capacity or intensity) and y_j is c_eff of the j‑th measure. The coefficients are computed by **normal equations** described via sums of powers of x and products with y (3x3 linear system), solved by Cramer’s rule in the app.

Goodness of fit:

```
R^2 = 1 - [ sum_j (y_j - yhat_j)^2 ] / [ sum_j (y_j - ybar)^2 ]
yhat_j = a + b*x_j + c*x_j^2
ybar   = (1/n) * sum_j y_j
```

### 5.7 Target slider and budget computation

Let the **target** be T% (percent). We convert it to an **X‑axis target** X* as:

```
Capacity mode: X* = E_base * (T/100)     (tCO2)
Intensity mode: X* = T                   (percent)
```

We then traverse measures in ascending c_eff, accumulating progress X until X >= X*. For the **budget** (₹), we convert the taken share back to **tons** (even in intensity mode) and sum c_eff * tons:

```
Budget = sum_over_taken_measures( c_eff_m * tons_taken_from_m )
```

---

## 6) Accessibility & UX details

- **Keyboard**: Press **Esc** to close the Measure Wizard.
- **Hover overlay** on MACC steps for precise rectangle details.
- **PNG export** from the MACC section.
- **LocalStorage** keys are namespaced per firm (e.g., `macc_firm_1_measures`).

---

## 8) License

© 2025 Shunya Lab. All rights reserved.
