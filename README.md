# India CCTS — Marginal Abatement Cost Curve (MACC) Builder

> What this is: A single‑page app that lets you build firm‑level MACCs using sample or custom catalogs (fuels, raw materials, transport, waste, electricity). You can define measures via a Quick form or a Template (catalog) wizard with multi‑line drivers and financing. Multi‑firm is supported with local export/import.

---

## 2) Features Overview

- Multi‑firm: Create/rename/delete firms. Each firm has its own sectors, baselines, measures, currency (₹), carbon price, and catalogs. Data is stored in localStorage and can be exported/imported as JSON.
- Catalog sources: Sample / Custom / Merged. Merged = Sample overridden by Custom (matched by `name` or `state`).
- Measure creation:
  - Quick: name, sector, abatement (tCO2), cost (₹/tCO2).
  - Template (catalog): multi‑line drivers (fuels, raw materials, transport, waste, electricity), drift (% per year), adoption profile, other direct reductions, CAPEX (upfront/financed), OPEX/savings, and finance params.
- MACC visualization:
  - Step chart (rectangles) or Quadratic Fit (smooth MACC) with R².
  - Capacity mode (cumulative tCO2 on X, cost on Y) and Intensity mode (cumulative % reduction).
  - Target slider and Budget calculator.
  - PNG export of the chart.
- Data management:
  - Import/export measures CSV.
  - Import/export catalogs (CSV/JSON) per tab.
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
- Aliases handled: `price_per_unit` or `price` → `price_per_unit_inr`; `ef_t_per_unit` or `ef_t` → `ef_tco2_per_unit`.

### 3.4 Electricity (`electricity.json`)
```json
[
  { "state": "India", "price_per_mwh_inr": 5000, "ef_tco2_per_mwh": 0.71 }
]
```
- Aliases handled: `price_per_mwh` or `price` → `price_per_mwh_inr`; `ef_t_per_mwh` or `ef_t` → `ef_tco2_per_mwh`.

### 3.5 Measures (`measures.csv`)
CSV columns:
```
id,name,sector,abatement_tco2,cost_per_tco2,selected,details
```
- `selected` defaults to `true` if omitted.
- `details` may be a JSON string produced by the Template wizard, e.g. including per‑year series and finance summary.

---

## 4) Using the App

1. Manage Firms → Create “My Firm” (start from Sample or Blank).  
   This seeds per‑firm `sectors`, `baselines`, `measures`, and Custom catalogs into localStorage.
2. Switch Data source (Sample / Custom / Merged) for the Wizard.
3. Add measure using Quick or Template (catalog).
4. View the MACC in Capacity or Intensity mode, adjust Target, and Export PNG.
5. Export your firm JSON for backup or sharing; Import later to restore.

---

## 5) Mathematics (plain text, GitHub‑friendly)

This section restates the formulas in simple symbols with short names. No LaTeX is used.

### 5.0 Notation (kept simple)
- Years considered: 2025, 2030, 2035, 2040, 2045, 2050.
- BASE_YEAR = 2025.
- For a given year i with calendar year `year_i`, define `DeltaYears_i = year_i - 2025`.
- Adoption share in year i: `a_i` (a fraction between 0 and 1).
- For each non‑electricity line `ℓ` (fuel, raw, transport, waste):
  - Base price: `P_l_0` (₹ per unit)
  - Base emission factor (EF): `E_l_0` (tCO2 per unit)
  - Price drift per year: `gP_l` (as a fraction; e.g., 0.05 means +5%/yr)
  - EF drift per year: `gE_l`
- For each electricity line `e`:
  - Base electricity price: `P_elec_e_0` (₹ per MWh)
  - Base electricity EF: `E_elec_e_0` (tCO2 per MWh)
  - Price drift: `gP_e`
  - EF drift: `gE_e`
  - Optional per‑year EF override: `E_elec_e_i_override`

### 5.1 Driver lines (prices, EF, quantities, emissions)

**Non‑electricity lines (fuel/raw/transport/waste):**  
Year‑i effective values:
- `P_l_i = P_l_0 * (1 + gP_l)^(DeltaYears_i)`
- `E_l_i = E_l_0 * (1 + gE_l)^(DeltaYears_i)`

If the template specifies a change in quantity vs BAU: `DeltaQ_l_i` (catalog units), the **adopted change** is:
- `DeltaQ_adopted_l_i = a_i * DeltaQ_l_i`

Emissions change from this line:
- `DeltaEMIS_l_i = DeltaQ_adopted_l_i * E_l_i`

**Electricity lines:**  
Year‑i effective values:
- `P_elec_e_i = P_elec_e_0 * (1 + gP_e)^(DeltaYears_i)`
- `E_elec_e_i = E_elec_e_i_override` if provided; otherwise `E_elec_e_0 * (1 + gE_e)^(DeltaYears_i)`

For a change in electricity use: `DeltaMWh_e_i`. Adopted change:
- `DeltaMWh_adopted_e_i = a_i * DeltaMWh_e_i`

Emissions change:
- `DeltaEMIS_elec_e_i = DeltaMWh_adopted_e_i * E_elec_e_i`

**Other direct reduction series:**  
The UI field “Other direct reduction (tCO2e)” is entered **positive for reductions**. We apply:
- `DeltaEMIS_other_i = - a_i * Other_i`

**Total signed emissions change and reporting:**  
Total (can be positive or negative):
- `DeltaEMIS_i = sum_l(DeltaEMIS_l_i) + sum_e(DeltaEMIS_elec_e_i) + DeltaEMIS_other_i`

We also report non‑negative magnitudes:
- `reduction_i = max(0,  - DeltaEMIS_i)`
- `addition_i  = max(0,    DeltaEMIS_i)`

### 5.2 Cost stack and financing (₹ crore)

Driver spend for year i (converted from ₹ to ₹ crore by dividing by 10,000,000):
- `driver_spend_cr_i = ( sum_l(DeltaQ_adopted_l_i * P_l_i) + sum_e(DeltaMWh_adopted_e_i * P_elec_e_i) ) / 10,000,000`

Inputs from UI (₹ crore): `opex_cr_i`, `savings_cr_i`, `other_cr_i`, `capex_upfront_cr_i`.

Financed CAPEX (₹ crore): `capex_financed_cr_i` with loan **interest** `r_i` and **tenure** `n_i` (years).  
Standard annuity factor `AF(r, n)`:
- if `r != 0` and `n > 0`: `AF = r * (1+r)^n / ((1+r)^n - 1)`
- if `r = 0` and `n > 0`: `AF = 1 / n`
- if `n = 0`: `AF = 0`

Annualized financing (₹ crore) in year i:
- `financed_annual_cr_i = capex_financed_cr_i * AF(r_i, n_i)`

Important: **Upfront CAPEX is not annuitized.** Count it only in the years where it is entered.

Net cost (₹ crore) used for per‑ton costs:
- `net_cost_cr_i = driver_spend_cr_i + opex_cr_i + other_cr_i - savings_cr_i + financed_annual_cr_i + capex_upfront_cr_i`

### 5.3 Cash flows, carbon price, and per‑ton costs

Let carbon price be `pCO2` (₹ per tCO2).

Yearly cash flow **without** CO2 (₹):
- `CF_noCO2_i = 10^7 * (savings_cr_i - opex_cr_i - driver_spend_cr_i - other_cr_i - financed_annual_cr_i) - 10^7 * capex_upfront_cr_i`

Add CO2 benefit **only on reduced tonnes**:
- `CF_withCO2_i = CF_noCO2_i + pCO2 * reduction_i`

Per‑ton representative‑year costs: pick index `i*` = first year where `reduction_i > 0` (fallback: 2035).
- if `reduction_i* > 0`:
  - `cost_noCO2 = (10^7 * net_cost_cr_i*) / reduction_i*`
  - `cost_withCO2 = (10^7 * net_cost_cr_i* - pCO2 * reduction_i*) / reduction_i*`
- else: both costs are `0`.

### 5.4 Effective cost under a carbon price

For each measure `m` with input cost `c_m` (₹/tCO2) and abatement `A_m` (tCO2):
- If the measure’s saved cost already included a carbon price `pCO2_saved`, adjust by the **difference** from the current price `pCO2_now`:
  - `c_eff_m = c_m - (pCO2_now - pCO2_saved)`
- Otherwise, subtract the full current carbon price:
  - `c_eff_m = c_m - pCO2_now`

The step MACC uses bars of width `A_m` and height `c_eff_m`.

### 5.5 Capacity vs. intensity X‑axis

Let baseline emissions for the sector be `E_base` (tCO2 per year).  
Sort measures by increasing `c_eff_m`. Define cumulative abatement:
- `S_j = sum_{m <= j} A_m`

Then:
- Capacity mode X = `S_j` (tCO2)
- Intensity mode X = `100 * S_j / E_base` (percent)

### 5.6 Quadratic MACC fit and R^2

If Quadratic Fit is selected, we fit `y = a + b*x + c*x^2` to points `(x_j, y_j)` where:
- `x_j` is cumulative abatement (capacity or intensity)
- `y_j` is `c_eff` of the j‑th measure

Coefficients are computed by the **normal equations** (using sums of `x`, `x^2`, `x^3`, `x^4`, `y`, `x*y`, `x^2*y`) and we solve the resulting `3x3` linear system (the code uses determinants).

Goodness of fit:
- `R2 = 1 - ( sum( (y_j - yhat_j)^2 ) / sum( (y_j - ybar)^2 ) )`
- `yhat_j = a + b*x_j + c*x_j^2`
- `ybar = average of y_j`

### 5.7 Target slider and budget computation

Let the target be `T` percent. Convert to an X‑axis target `X*`:
- Capacity mode: `X* = E_base * (T/100)` (tCO2)
- Intensity mode: `X* = T` (percent)

Traverse measures in ascending `c_eff`, accumulating abatement until `X >= X*`.  
Budget (₹) = sum over taken measures of `c_eff_m * tons_taken_from_m`.

---

## 6) Accessibility & UX details

- Keyboard: Press Esc to close the Measure Wizard.
- Hover overlay on MACC steps for precise rectangle details.
- PNG export from the MACC section.
- LocalStorage keys are namespaced per firm (e.g., `macc_firm_1_measures`).

---

## 8) License

© 2025 Shunya Lab. All rights reserved.
