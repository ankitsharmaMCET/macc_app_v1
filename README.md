
 **India CCTS — Marginal Abatement Cost Curve (MACC) Builder**

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

## 5) Mathematics

This section documents the formulas exactly as used in the code (see comments in `MACCApp.jsx`). We index **years** by \(i\) over the set \(\mathcal{Y} = \{2025, 2030, 2035, 2040, 2045, 2050\}\). The **BASE\_YEAR** is \(2025\). Let \(\Delta_i = \text{year}_i - 2025\). The **adoption share** in year \(i\) is \(a_i \in [0,1]\).

### 5.1 Driver lines: price & EF drift, quantities, and emissions
For each line \(\ell\) of type \(k \in \{\text{fuel, raw, transport, waste}\}\) with **base price** \(P_{\ell,0}\) (₹/unit) and **base EF** \(E_{\ell,0}\) (tCO₂/unit),
- If the user overrides base price / EF, those values replace \(P_{\ell,0}\), \(E_{\ell,0}\).
- With price drift \(g_{\ell}^{P}\) and EF drift \(g_{\ell}^{E}\) (fraction per year), the **effective year‑\(i\)** values are:
  $$
  P_{\ell,i} = P_{\ell,0} \,(1+g_{\ell}^{P})^{\Delta_i},
  \quad
  E_{\ell,i} = E_{\ell,0} \,(1+g_{\ell}^{E})^{\Delta_i}.
  $$
If the Template series specifies a change in quantity (vs BAU) \(\Delta q_{\ell,i}\) (in the catalog unit), the **adopted** change is:
$$
\widetilde{\Delta q}_{\ell,i} = a_i \cdot \Delta q_{\ell,i}.
$$
The **emissions delta** from line \(\ell\) in year \(i\) is:
$$
\Delta \text{EMIS}_{\ell,i} = \widetilde{\Delta q}_{\ell,i}\, E_{\ell,i}.
$$

#### Electricity lines
For electricity line \(e\) with base price \(P^{\text{elec}}_{e,0}\) (₹/MWh), drift \(g^{P}_{e}\), base EF \(E^{\text{elec}}_{e,0}\) (tCO₂/MWh), EF drift \(g^{E}_{e}\), optional **per‑year EF override** \( \widehat{E}^{\text{elec}}_{e,i}\):
$$
P^{\text{elec}}_{e,i} = P^{\text{elec}}_{e,0} (1+g^{P}_e)^{\Delta_i},\qquad
E^{\text{elec}}_{e,i} =
\begin{cases}
\widehat{E}^{\text{elec}}_{e,i}, & \text{if provided}\\[2pt]
E^{\text{elec}}_{e,0}(1+g^{E}_e)^{\Delta_i}, & \text{otherwise}
\end{cases}
$$
For electricity delta \( \Delta \text{MWh}_{e,i}\), the adopted delta is \( \widetilde{\Delta \text{MWh}}_{e,i} = a_i \cdot \Delta \text{MWh}_{e,i}\), and emissions delta:
$$
\Delta \text{EMIS}^{\text{elec}}_{e,i} = \widetilde{\Delta \text{MWh}}_{e,i}\, E^{\text{elec}}_{e,i}.
$$

#### “Other direct reduction” series
The UI field “Other direct reduction (tCO₂e)” is entered **positive for reductions**; in the code we apply:
$$
\Delta \text{EMIS}^{\text{other}}_{i} = -\, a_i \cdot \text{Other}_i.
$$

#### Total signed emissions delta and split
For year \(i\):
$$
\Delta \text{EMIS}_i
= \sum_{\ell \in \text{fuel,raw,trans,waste}} \Delta \text{EMIS}_{\ell,i}
  + \sum_{e \in \text{elec}} \Delta \text{EMIS}^{\text{elec}}_{e,i}
  + \Delta \text{EMIS}^{\text{other}}_{i}.
$$
We report **non‑negative** reduction/addition magnitudes:
$$
\text{reduction}_i = \max\{0,\; -\Delta \text{EMIS}_i\},\qquad
\text{addition}_i = \max\{0,\; \Delta \text{EMIS}_i\}.
$$

### 5.2 Cost stack and financing (₹ crore)
Driver spend for year \(i\) (converted to ₹ crore by dividing by \(10^7\)):
$$
\text{driver\_cr}_i
= \frac{1}{10^7}\left(
\sum_{\ell} \widetilde{\Delta q}_{\ell,i} P_{\ell,i}
+ \sum_{e} \widetilde{\Delta \text{MWh}}_{e,i} P^{\text{elec}}_{e,i}
\right).
$$
Let \( \text{opex\_cr}_i, \text{savings\_cr}_i, \text{other\_cr}_i, \text{capex\_upfront\_cr}_i\) be the series from the UI (₹ crore). Let financed CAPEX be \(\text{capex\_financed\_cr}_i\) with loan **interest** \(r_i\) and **tenure** \(n_i\). The app uses the standard **annuity factor**:
$$
\text{AF}(r,n)=
\begin{cases}
\dfrac{r(1+r)^n}{(1+r)^n-1}, & r\neq 0,\, n>0,\\[8pt]
\dfrac{1}{n}, & r=0,\, n>0,\\[8pt]
0, & n=0.
\end{cases}
$$
Annualized financing (₹ crore) in year \(i\):
$$
\text{financedAnnual\_cr}_i = \text{capex\_financed\_cr}_i \cdot \text{AF}(r_i,n_i).
$$

> **Important:** Upfront CAPEX is **not annuitized**; it is treated as a cash outflow **only in the years where it is entered**.

The **net cost** (₹ crore) for MACC per‑ton calculations is:
$$
\text{net\_cost\_cr}_i
= \text{driver\_cr}_i + \text{opex\_cr}_i + \text{other\_cr}_i - \text{savings\_cr}_i
+ \text{financedAnnual\_cr}_i + \text{capex\_upfront\_cr}_i.
$$

### 5.3 Cash flows, Carbon price, and Per‑ton costs
Let the **carbon price** be \(p^{\text{CO2}}\) (₹ per tCO₂).  
Yearly **cash flow without CO₂** (₹):
$$
\text{CF}^{\neg \text{CO2}}_i
= 10^7\big(\text{savings\_cr}_i - \text{opex\_cr}_i - \text{driver\_cr}_i - \text{other\_cr}_i - \text{financedAnnual\_cr}_i\big)
- 10^7\cdot \text{capex\_upfront\_cr}_i.
$$
We add CO₂ **benefit only on reduced tonnes**:
$$
\text{CF}^{\text{CO2}}_i = \text{CF}^{\neg \text{CO2}}_i + p^{\text{CO2}} \cdot \text{reduction}_i.
$$

Per‑ton **representative‑year** costs (the app picks a representative index \(i^\*\), by default the first year with \(\text{reduction}_i>0\); fallback is 2035):
$$
\text{cost}^{\neg \text{CO2}}_{i^\*}
=
\begin{cases}
\dfrac{10^7\cdot \text{net\_cost\_cr}_{i^\*}}{\text{reduction}_{i^\*}}, & \text{if }\text{reduction}_{i^\*}>0,\\[6pt]
0, & \text{otherwise},
\end{cases}
$$
$$
\text{cost}^{\text{CO2}}_{i^\*}
=
\begin{cases}
\dfrac{10^7\cdot \text{net\_cost\_cr}_{i^\*} - p^{\text{CO2}}\cdot \text{reduction}_{i^\*}}{\text{reduction}_{i^\*}}, & \text{if }\text{reduction}_{i^\*}>0,\\[6pt]
0, & \text{otherwise}.
\end{cases}
$$


### 5.4 Effective cost under carbon price
For each measure \(m\) with **input cost** \(c_m\) (₹/tCO₂) and abatement \(A_m\) (tCO₂), we display an **effective** cost for the MACC that adjusts for carbon price in one of two ways:
- If the measure’s **saved cost** already **included** a carbon price \(p^{\text{CO2}}_{\text{save}}\), we subtract only the **difference** from the current price \(p^{\text{CO2}}_{\text{now}}\):
  $$
  c^{\text{eff}}_m = c_m - \big(p^{\text{CO2}}_{\text{now}} - p^{\text{CO2}}_{\text{save}}\big).
  $$
- Otherwise, subtract the full current carbon price:
  $$
  c^{\text{eff}}_m = c_m - p^{\text{CO2}}_{\text{now}}.
  $$
The **step MACC** uses bars of width \(A_m\) and height \(c^{\text{eff}}_m\).

### 5.5 Capacity vs Intensity X‑axis
Let the **baseline emissions** for the selected sector be \(E^{\text{base}}\) (tCO₂/yr). Sorting measures by increasing \(c^{\text{eff}}_m\), define cumulative abatement
$$
S_j = \sum_{m\le j} A_m.
$$
- **Capacity** mode X = \(S_j\) (tCO₂).  
- **Intensity** mode X = \(100\cdot S_j/E^{\text{base}}\) (percent).

### 5.6 Quadratic MACC fit and \(R^2\)
If **Quadratic Fit** is selected, we fit a polynomial \(y = a + b x + c x^2\) to \((x_j,y_j)\) points where \(x_j\) is cumulative abatement (capacity or intensity) and \(y_j\) is \(c^{\text{eff}}_{m(j)}\). The coefficients are computed by **normal equations**:
$$
\begin{bmatrix}
n & \sum x & \sum x^2 \\[3pt]
\sum x & \sum x^2 & \sum x^3 \\[3pt]
\sum x^2 & \sum x^3 & \sum x^4
\end{bmatrix}
\begin{bmatrix} a\\ b\\ c \end{bmatrix}
=
\begin{bmatrix}
\sum y\\ \sum xy\\ \sum x^2 y
\end{bmatrix}.
$$
The app solves this \(3\times3\) system via Cramer’s rule (determinants).  
Goodness of fit:
$$
R^2 = 1 - \frac{\sum_j (y_j - \widehat{y}_j)^2}{\sum_j (y_j - \bar{y})^2},\qquad
\widehat{y}_j = a + b x_j + c x_j^2,\quad \bar{y}=\frac{1}{n}\sum_j y_j.
$$

### 5.7 Target slider and budget computation
Let the **target** be \(T\%\). We convert it to an **X‑axis target** \(X^\*\) as:
- Capacity mode: \(X^\* = E^{\text{base}}\cdot (T/100)\) (tCO₂).  
- Intensity mode: \(X^\* = T\) (percent).

We then traverse measures in ascending \(c^{\text{eff}}\), accumulating progress \(X\) until \(X\ge X^\*\). For the **budget** (₹), we convert the taken share back to **tons** (even in intensity mode) and sum **\(c^{\text{eff}}\times \text{tons}\)**:
$$
\text{Budget} = \sum_{\text{taken }m} c^{\text{eff}}_m \cdot \text{tons taken from } m.
$$

---

## 6) Accessibility & UX details

- **Keyboard**: Press **Esc** to close the Measure Wizard.
- **Hover overlay** on MACC steps for precise rectangle details.
- **PNG export** from the MACC section.
- **LocalStorage** keys are namespaced per firm (e.g., `macc_firm_1_measures`).

---



---

## 8) License

© 2025 Shunya Lab. All rights reserved.  
