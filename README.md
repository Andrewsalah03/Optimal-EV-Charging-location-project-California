# Charging Ahead: Where California Needs Its Next EV Chargers

A ZIP-level analysis of electric vehicle demand versus public charging supply in California, built in Python. We merge DMV registrations, U.S. Department of Energy charging station data, and U.S. Census demographics to find where charging infrastructure is falling behind, explain what drives EV adoption, and rank ZIP codes by how many new public ports they need.

**Authors:** Andrew Salah & Bradley Shea
**Course:** MSBA 502 – Analytics Programming, University of San Diego (Fall 2026)

---

## The Business Problem

California has about **2.2 million EVs** (7.6% of 29 million passenger vehicles), but public chargers are not keeping pace evenly. Statewide there are **32.5 EVs per public charging port**, yet EV share ranges from under 1% to over 25% by ZIP code, and **253 ZIP codes have EVs but no public chargers at all**.

Charging network operators have limited capital. Building in the wrong place strands money in low-use stations; missing a busy area hands revenue to competitors. This project answers three questions:

1. Which ZIP codes are most underserved today?
2. What drives EV adoption, and where will demand grow next?
3. How many additional public ports does each ZIP need?

**Stakeholders:** charging network operators (primary), utilities planning grid upgrades, the California Energy Commission and city planners allocating grants, and retail site hosts.

## Key Findings

- **Income is the strongest driver of EV adoption.** Each $10K of median household income adds about **1.2 percentage points** of EV share. The regression explains **73%** of the variation across ZIPs (R² = 0.731).
- **Holding income constant, higher home ownership means *lower* EV share.** High-ownership areas skew suburban and rural; wealthy renter-heavy areas adopt faster and depend most on public chargers.
- **K-means clustering finds four market types:** Hot Markets, Affluent Adopters, Emerging & Underserved, and Well-Served.
- **The top 20 ZIP codes need about 3,650 additional public ports** to reach the state average, led by Irvine (92620, +268), San Ramon (94582, +236), and Fremont (94539, +229).
- **San Diego County** mirrors the state (7.8% EV share, 32.1 EVs per port), but Chula Vista (91913) has 4,654 EVs sharing 7 ports, and La Mesa (91941) has 2,023 EVs and no public chargers.

## Repository Structure

```
ev-charging-gap-analysis/
├── data/
│   ├── raw/                                          # Original downloads, renamed
│   │   ├── dmv_vehicle_fuel_type_count_by_zip_2025.csv.gz   # gzipped; pandas reads it directly
│   │   ├── afdc_ev_stations_ca_2026-10-12.csv
│   │   ├── census_acs5_2024_B19013_median_income.csv
│   │   ├── census_acs5_2024_B25003_tenure.csv
│   │   └── census_acs5_2024_B01003_population.csv
│   └── processed/                                    # Created by the notebooks
│       ├── ev_clean_by_zip.csv                       # One row per ZIP (1,388 ZIPs)
│       ├── ev_results_by_zip.csv                     # + predictions, segments, ports needed
│       └── top20_charger_targets.csv                 # Final recommendation
├── notebooks/
│   ├── 01_data_cleaning.ipynb                        # Load, clean, merge
│   └── 02_modeling_and_analysis.ipynb                # Regression, clustering, ranking
├── requirements.txt
└── README.md
```

## Data Sources

| Dataset | Source | Description | Raw size |
| --- | --- | --- | --- |
| Vehicle Fuel Type Count by ZIP Code (as of Dec 31, 2025) | [California DMV via data.ca.gov](https://data.ca.gov/dataset/vehicle-fuel-type-count-by-zip-code) | Registered vehicles by ZIP, fuel type, make, model year, and duty | 504,168 rows |
| Alternative Fuel Stations, California, electric (Oct 12, 2026) | [U.S. DOE Alternative Fuels Data Center](https://afdc.energy.gov/data_download) | Location, access, status, and port counts for every EV station | 21,814 stations |
| B19013 – Median Household Income | [U.S. Census ACS 5-Year, 2020–2024](https://data.census.gov) | Median household income by ZIP Code Tabulation Area | 33,772 ZCTAs |
| B25003 – Tenure | U.S. Census ACS 5-Year, 2020–2024 | Owner- vs. renter-occupied households | 33,772 ZCTAs |
| B01003 – Total Population | U.S. Census ACS 5-Year, 2020–2024 | Population | 33,772 ZCTAs |

## How to Run

```bash
git clone https://github.com/<your-username>/ev-charging-gap-analysis.git
cd ev-charging-gap-analysis
pip install -r requirements.txt
jupyter notebook
```

Run the notebooks in order from the `notebooks/` folder:

1. **`01_data_cleaning.ipynb`** reads `data/raw/` and writes `data/processed/ev_clean_by_zip.csv`.
2. **`02_modeling_and_analysis.ipynb`** reads the clean table and writes `ev_results_by_zip.csv` and `top20_charger_targets.csv`.

## Methodology

### 1. Data cleaning and transformation

- **DMV:** removed out-of-state records ("OOS"), kept light-duty vehicles only, grouped 9 fuel types into BEV, PHEV, Hybrid, Gas, Diesel, and Other, and pivoted to one row per ZIP.
- **Charging stations:** kept public, currently open stations (20,180 of 21,814) and summed Level 1, Level 2, and DC fast ports by ZIP (69,285 ports). Ports are counted instead of stations because one station can have many plugs.
- **Census:** skipped the second header row, converted `ZCTA5 #####` to plain ZIP codes, filtered to California (ZIPs 900–961), and recoded text values in income (`250,000+` → 250,000; `2,500-` → 2,500; `-` → missing).
- **Merge:** joined all sources on ZIP; ZIPs with no chargers are kept with 0 ports. Dropped ZIPs with fewer than 500 vehicles or 1,000 residents and ZIPs missing income. **Final table: 1,388 ZIPs.**
- **New variables:** EV share, EVs per port, ports per 1,000 people, ports per 100 EVs, home ownership rate.

### 2. Linear regression (statsmodels OLS)

`EV share (%) ~ median income ($10K) + home ownership (%) + population (10K)`, n = 1,387 (ZIP 94304, Palo Alto, removed as a likely fleet-registration outlier at 57% EVs).

| Variable | Coefficient | Std. error | t | p-value |
| --- | --- | --- | --- | --- |
| Intercept | 0.831 | 0.317 | 2.62 | 0.009 |
| Median income ($10K) | **+1.175** | 0.019 | 60.83 | < 0.001 |
| Home ownership (%) | **−0.095** | 0.004 | −21.59 | < 0.001 |
| Population (10K) | −0.067 | 0.037 | −1.80 | 0.071 |

R² = 0.731 · Adjusted R² = 0.730 · F = 1,250 (p < 0.001) · Condition number = 259

Residuals (actual minus predicted EV share) identify ZIPs adopting faster or slower than their demographics predict.

### 3. K-means clustering (scikit-learn)

Clustered ZIPs on EV share, log ports per 100 EVs, regression residual, and income, all standardized. The elbow method supported k = 4 (`n_init=10`, `random_state=42`).

| Segment | ZIPs | Avg. EV share | Median ports per 100 EVs | Strategy |
| --- | --- | --- | --- | --- |
| Hot Markets | 187 | 14.7% | 1.6 | Build now: adopting faster than income predicts |
| Affluent Adopters | 195 | 14.2% | 1.4 | Build selectively: highest income, more home charging |
| Emerging & Underserved | 625 | 4.6% | 0.7 | Build ahead: thinnest coverage, 226 with no public charger |
| Well-Served | 380 | 3.9% | 8.4 | Skip for now |

### 4. Ranking: ports needed

The benchmark is the statewide average of **32.5 EVs per public port** (2,175,946 EVs ÷ 66,878 ports).

```
Ports needed = (EVs in ZIP ÷ 32.5) − existing public ports
```

Example: Irvine (92620) has 9,716 EVs and 31 ports → 9,716 ÷ 32.5 ≈ 299 → **268 more ports needed**.

## Limitations

- **Registration ≠ charging location.** Fleet and lease registrations can inflate some ZIPs.
- **ZIP boundaries are not driving limits.** Drivers use chargers in neighboring ZIPs, so some gaps are overstated.
- **No home-charging or utilization data.** Affluent, high-ownership ZIPs may need fewer public ports than the ranking suggests.
- **Single snapshot and mismatched dates.** DMV data is from Dec 2025, chargers from Oct 2026; "emerging" demand is inferred, not measured over time.
- **Correlation, not causation.** Income may stand in for unmeasured factors such as commute length or local incentives.
- **Statistical caveats.** Residuals are moderately right-skewed (0.90) and spatially correlated (Durbin-Watson 1.36); cluster separation is modest (silhouette ≈ 0.29); DC fast and Level 2 ports are weighted equally.

## Future Work

- Forecast EV share by ZIP through 2030 using DMV snapshots back to 2018.
- Measure charger access within a drive radius using station coordinates instead of ZIP boundaries.
- Add utilization and pricing data to estimate revenue and payback per site.
- Weight DC fast chargers more heavily and incorporate grid capacity.

## Tools

Python (pandas, NumPy, statsmodels, scikit-learn, Matplotlib, seaborn), Jupyter, and Tableau for presentation visuals.
