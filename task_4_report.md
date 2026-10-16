# Task 4 — Satellite-Surface Correlation Validation & Final Feature Matrix

**Author:** Youssef Sherif
**Input:** `Merged_cleaned_daily_data_task3.csv`
**Output:** `task4_results.json` and Definitive Feature Matrix for Week 3

## 1. Satellite vs Surface Correlation per District

The goal of this test is to determine if Sentinel-5P columns carry additional spatial-temporal information beyond the surface measurements. I computed the Spearman correlation ($r$) between each satellite column and its corresponding surface reading, using **only rows where real observations occurred** (`s5p_*_is_observed = 1`).

| Pollutant | District | N (Observed) | Spearman r | Decision | 
| ----- | ----- | ----- | ----- | ----- | 
| **NO2** | Downtown | 986 | 0.439 | **KEEP** | 
| **NO2** | Helwan | 1008 | 0.398 | **KEEP** | 
| **NO2** | New Cairo | 999 | 0.497 | **KEEP** | 
| **NO2** | 6th of October | 1001 | 0.602 | **KEEP** | 
| **NO2** | Shobra | 976 | 0.476 | **KEEP** | 
| **O3** | Downtown | 1085 | 0.220 | **KEEP** | 
| **O3** | New Cairo | 1085 | 0.184 | **DROP / SECONDARY** (r < 0.2) | 
| **O3** | 6th of October | 1085 | 0.273 | **KEEP** | 
| **SO2** | All Districts | \~970 | < 0.16 | **DROP** (No signal) | 
| **CO** | All Districts | \~945 | < 0.08 | **DROP** (No signal) | 

**Key Findings:**

1. **NO2 is a strong driver:** The `s5p_no2_col` shows strong validation against surface readings, especially in 6th of October (r=0.60). It must be kept as a primary feature.

2. **O3 is marginal:** `s5p_o3_col` barely crosses the 0.2 threshold in most districts and fails in New Cairo and Shobra. It can be used but might not provide strong predictive power.

3. **SO2 & CO are noise:** The satellite columns for SO2 and CO show near-zero correlation with ground sensors across all districts. They provide no surface-level signal and must be dropped.

## 2. Gap-fill Quality Assessment

For days where the satellite passed over but data was missing (e.g., due to cloud cover), a 3-day forward fill was applied previously. I checked whether these gap-filled days (`is_observed = 0`) differ systematically from real observation days.

| Satellite Column | Variance (Observed) | Variance (Gap-filled) | Variance Shift | 
| ----- | ----- | ----- | ----- | 
| `s5p_no2_col` | $1.24 \times 10^{-8}$ | $1.87 \times 10^{-8}$ | **+ 51.1%** | 
| `s5p_o3_col` | $1.04 \times 10^{-4}$ | $4.23 \times 10^{-5}$ | **- 59.3%** | 

**Conclusion:**
There is a massive systematic shift in variance when using forward-filled data. The gap-filled values have very different distributions.
**Recommendation:** The Week 3 modeling team **must** include the `s5p_*_is_observed` flags as model features so the tree algorithms know when to trust the satellite reading and when it is merely a persisted value. Alternatively, reduce the forward-fill window to 1 day.

## 3. The Definitive Feature Matrix (Week 3 Reference)

Based on Tasks 1–4, this is the authoritative reference for the Week 3 modeling phase. **No one should start modeling until they've applied these dropping/keeping rules.**

| Column Name | Keep or Drop | Role | Reason | 
| ----- | ----- | ----- | ----- | 
| `station_id`, `day` | **Keep** | Identifier | Grouping and CV splitting. | 
| `pm2_5_corrected` | **Keep** | **Primary Target** | Main regression target. | 
| `aqi_category`, `is_health_risk` | **Keep** | **Class. Target** | Targets for the classification task. | 
| `pm2_5` | **DROP** | Drop | Raw target (leaker/uncalibrated). | 
| `pm10`, `nitrogen_dioxide`, `ozone`, `sulphur_dioxide`, `carbon_monoxide` | **DROP** | Drop / Leaker | Same-day actuals leak the target. Models must rely on lags. | 
| `wind_direction_10m_mean` | **DROP** | Drop | Replaced by sin/cos encoding. | 
| All `_lag_1d`, `_lag_2d`, `_roll_avg_` | **Keep** | Feature | Core temporal predictors. | 
| `s5p_no2_col`, `s5p_o3_col` | **Keep** | Feature | Validated satellite drivers. | 
| `s5p_no2_is_observed`, `s5p_o3_is_observed` | **Keep** | Feature | Mandatory to handle gap-fill variance shifts. | 
| `s5p_so2_col`, `s5p_co_col` (and flags) | **DROP** | Drop | Failed the r > 0.2 correlation threshold. | 
| `wind_dir_sin`, `wind_dir_cos` | **Keep** | Feature | Properly encoded circular wind features. | 
| `is_downwind_helwan`, `is_downwind_shobra` | **Keep** | Feature | Industrial exposure flags (Task 3). | 
| `dust_transport_proxy`, `inversion_proxy` | **Keep** | Feature | Physical interaction terms (Task 3). | 
