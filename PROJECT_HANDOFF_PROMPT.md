# HANDOFF PROMPT — Germany Day-Ahead Electricity Price Forecast

Copy everything below this line into another AI. Follow it strictly. Do not invent publication times, do not claim live skill without ≥60 valid pre-gate days, and do not treat shift-0 results as deployable.

---

## 1. Goal

Forecast **German EPEX Spot day-ahead prices** (bidding zone DE/LU, EUR/MWh) for the next **Berlin delivery day**, under an explicit **gate-closure** information set so that no feature uses information that was unavailable at auction gate closure.

- **Gate closure:** 12:00 CET on day D−1 for delivery day D (winter ≈ 11:00 UTC).
- **Horizon:** direct 24 hourly forecasts for the full Berlin delivery day (not recursive multi-step on predicted prices).
- **Author:** Azizallah Farhadi · farhadmy1974@gmail.com
- **Status:** Research prototype (Nov 2026). MIT code; market data under original terms.

## 2. Two repositories (critical)

| Role | URL / path | Notes |
|------|------------|--------|
| Research / older | https://github.com/AZFARHAD24511/germany-electricity-price-forecast | GitHub Actions often **disabled** on this account |
| Working / Actions | Local: `D:\projects\germany-electricity-price-forecast-v2-fixed-realdata` | May point at org remote (e.g. `irantradeonewindow-eng/...`). Push as `AZFARHAD24511` → **403** unless Write access or own remote |

**Rule:** After code/data pipeline changes, sync the Actions-enabled copy. If `git push` returns 403: `git remote -v`, then either get org Write permission or:

```powershell
git remote set-url origin https://github.com/AZFARHAD24511/<your-repo>.git
# Password prompt → use a classic PAT with scope "repo", not the GitHub login password
git push -u origin main
```

Windows daily automation uses `run_daily.bat` (Task Scheduler ~09:00 Berlin), not only Actions.

## 3. Non-negotiable scientific rules

1. **Never** use delivery-day price(t) or price(t−1) inside gate-closure features.
2. SMARD chart-data is **not** an official BNetzA API (confirmed by SMARD email ~29 Oct 2026). `meta_data.created` is **not** a reliable issue time → SMARD wind/solar stay under **external_shift = 24 h**.
3. Weather under the same 24 h shift by default; ablation **C** uses weather_shift = 36 h (stricter if series is reanalysis/observed).
4. **ENTSO-E day-ahead load forecast** (`load_forecast_mw`) is treated as available for the target hour (**no** 24 h shift), because it is a day-ahead product. Still verify operationally that it is published before gate when claiming live skill.
5. Shift **0** external results are an **upper bound only**, not a primary claim.
6. Forecast files named by **delivery date**: `outputs/forecasts/forecast_YYYY-MM-DD.csv`. Do not overwrite an existing file with `pre_gate=True` by a late post-gate run (save as `late_forecast_...` instead).
7. Live performance claims only after **≥ 60** valid pre-gate, complete, unique-delivery-day scored days.
8. Report **MAE and mean error (ME)**. Recent live bias is **not** one-sided.

## 4. Project structure (canonical)

```text
germany-electricity-price-forecast/   # or ...-v2-fixed-realdata
├── data/raw/
│   ├── germany_electricity_hourly.csv          # price_eur_mwh, timestamp_utc
│   ├── germany_smard_day_ahead_forecasts.csv  # wind/solar day-ahead series
│   ├── germany_weather_forecast_archive.csv   # regional Open-Meteo → avg_wind/temp/rad
│   └── germany_load_forecast.csv              # ENTSO-E load_forecast_mw (needs ENTSOE_API_KEY)
├── src/electricity_forecast/
│   ├── data.py          # load_* + join_all(price, smard, weather, load_path=optional)
│   ├── features.py      # gate vs legacy; lags; context; calendar; holidays; shifts
│   ├── train.py         # HGB + Ridge ensemble
│   ├── evaluate.py      # SN24/SN168, rMAE, DM-HLN, live scoring
│   ├── visualize.py     # charts; x-axis labels rotated 90°
│   └── ...
├── scripts/
│   ├── update_data.py           # SMARD price+forecasts, Open-Meteo, ENTSO-E load
│   ├── forecast_horizon.py      # direct_forecast(...); pre_gate; data_gap_hours
│   ├── bias_correction.py       # errors from RAW yhat; hourly bias; shrink
│   ├── run_daily_forecast.py    # live: update → forecast → score → README block
│   ├── run_ablation.py          # setups A–E → reports/metrics/ablation.csv
│   ├── run_rolling_eval.py      # replay live procedure; --incremental
│   ├── update_readme_results.py # rebuild KEY / RESULTS README sections
│   └── run_pipeline.py          # older backtest entry (if present)
├── tests/
│   ├── test_leakage.py          # gate features invariant to delivery-day price changes
│   └── test_features.py
├── reports/
│   ├── figures/   ablation_mae.png, rolling_eval.png, pred_vs_actual_*.png
│   ├── metrics/   ablation.csv, rolling_eval.csv, backtest_summary.csv
│   └── methodology/  CONSERVATIVE_LEAKAGE.md (or similar)
├── outputs/
│   ├── forecasts/  forecast_*.csv, score_summary.csv, legacy/ (old files)
│   └── figures/    latest_forecast.png, latest_vs_actual.png
├── .github/workflows/daily_forecast.yml   # schedule; contents: write
├── run_daily.bat
├── requirements.txt
├── pyproject.toml
└── README.md   # markers: KEY_*, RESULTS_*, DAILY_FORECAST_*
```

Optional / historical in sandbox artifacts (not always in the live Windows tree):

- `01_baseline_*.py`, `02_add_external_forecasts.py`, `03_ensemble_*.py` — early experiments
- `baseline_results/`, `external_results/`, `ensemble_results/` — early CSVs
- `METHODOLOGY_CONSERVATIVE_LEAKAGE.md` — SMARD letter + 24 h lag rationale
- `germany_forecast_bundle.zip` — snapshot of data.py, features.py, update_data.py, run_daily_forecast.py with load support

## 5. Feature protocol (gate-closure)

| Family | Rule |
|--------|------|
| Price lags 24, 48, 72, 168, 336 h | Always ≥ 24 h old |
| Price context (mean/std/min/max/last of day, 7d mean/std) | Only **complete** Berlin days up to D−1 (`price_info="gate"`) |
| Calendar + German federal holidays (eve/next) | Deterministic |
| SMARD wind/solar | `shift(external_shift)` default **24** |
| Weather avg_wind, avg_temp, avg_rad | Same; optional weather_shift=36 |
| `load_forecast_mw` (ENTSO-E) | **No shift** |

**Legacy (ablation A only):** `shift(1).rolling(...)` — **leaks** delivery-day price(t−1). Forbidden as primary.

`tests/test_leakage.py` must pass: changing any price on the delivery day must leave all gate features unchanged.

## 6. Model and evaluation

- **Model:** `HistGradientBoostingRegressor` weight 0.65 + `Ridge` 0.35.
- **Training:** expanding window; retrain per backtest window / per rolling day / per live run as coded.
- **Baselines:** SN24 (same hour yesterday), SN168 (same hour last week).
- **Tests:** one-sided Diebold–Mariano on **daily** losses with Harvey–Leybourne–Newbold correction.
- **Bias correction:** optional, from **raw** past `yhat` errors, hourly, min_obs≈5, shrink; often inactive (bias=0) when history is short or sign flips day-to-day.

### Latest ablation order-of-magnitude (89 days / 2136 h, Jul–Oct 2026 windows; ~19 Nov 2026 re-run)

| Config | MAE ≈ |
|--------|-------|
| SN24 | 37.79 |
| SN168 | 48.26 |
| A legacy leaky shift24 | ~26.2 |
| **B gate shift24 (PRIMARY)** | **~28.2** |
| C gate weather+36 | ~28.2 |
| D gate shift0 (NOT gate-safe) | ~20.5 |
| E gate no external | ~29.4 (varies by code version; sometimes ~27.9) |

Primary claim: **B beats SN24 by ~26%**, DM p < 0.001.  
Leak size A→B ≈ 1.7 EUR/MWh.  
D is upper bound only.

### Rolling-origin replay (~27 days from 2026-10-24)

- Model MAE ≈ 36 vs SN24 ≈ 51; beats SN24 on ~19/27 days; DM p ≈ 0.001.
- This is a **replay**, not a live pre-gate record.

### Live scored sample (very small)

Example ME (ŷ − y) on complete days:

- 2026-11-17: ME ≈ +55.8, under% 0% (always over)
- 2026-11-18: ME ≈ +45, under% ~12%
- 2026-11-19: ME ≈ −40.3, under% 100% (always under)

→ Bias **sign is unstable**; do not claim “always under/over”.

## 7. Data update (`scripts/update_data.py`)

Requires network. Env: `ENTSOE_API_KEY` for load.

1. SMARD chart_data filter `4169` → prices  
2. SMARD filters 5097 / 3791 / 123 / 125 → wind/solar combined, offshore, onshore, solar  
3. Open-Meteo archive + forecast APIs for 4 DE points (N/E/S/W) → weather CSV  
4. ENTSO-E API documentType A65 processType A01 area 10Y1001A1001A83F → hourly mean load forecast  

Append-only merge on `timestamp_utc`. Do not rewrite history blindly.

## 8. Daily job flow

```text
update_data.py
  → join frame (price + smard + weather + load)
  → direct_forecast (gate features, external_shift=24)
  → optional bias_correction
  → outputs/forecasts/forecast_<delivery>.csv
  → score past forecasts vs realized price (Counted = complete ∩ pre_gate ∩ unique delivery)
  → patch README between DAILY_FORECAST_START/END
  → run_rolling_eval.py --incremental (optional)
  → git add/commit/push (if credentials allow)
```

Windows:

```powershell
cd D:\projects\germany-electricity-price-forecast-v2-fixed-realdata
$env:ENTSOE_API_KEY = "<token>"
.\run_daily.bat
Get-Content logs\daily_windows.log -Tail 50
```

## 9. Strengths

- Explicit **gate-closure** design and documented SMARD limitation (third-party chart_data).
- Leakage test suite and ablation A/B that **quantifies** the cost of leaky rolling features.
- Direct delivery-day forecast (no recursive use of own price predictions as inputs).
- Transparent baselines (SN24/SN168), rMAE, DM tests, `pre_gate` / `data_gap_hours` flags.
- Automated daily path (bat + optional Actions) and README auto-sections.
- Conservative externals (24 h) so primary numbers are not silently optimistic.

## 10. Weaknesses / work to prioritize

1. **Live sample too small** — no live skill claim yet; target ≥60 pre-gate days.  
2. **Load not always wired into ablation CLI** — `run_ablation.py` may not pass `load_path`; re-measure B vs E vs B+load before claiming load helps. Latest evidence does **not** prove load improved MAE.  
3. **External skill at 24 h lag is weak/ambiguous** — sometimes E ≈ B; shift0 helps a lot but is not gate-safe without proven issue times.  
4. **Weather may be reanalysis** — risk of mild leakage if treated as forecast; use weather_shift=36 in strict variants.  
5. **Bias correction weak** on live days (sign flips; many zeros in Bias column).  
6. **Regime coverage short** — one season in main windows; gas/CO2 shocks not modeled.  
7. **Hourly only** — market moved toward 15-minute day-ahead products (from Oct 2025); not yet modeled.  
8. **Point forecasts only** — no quantiles / no economic value (trading PnL).  
9. **Git/Actions fragility** — 403 on org remotes; Actions disabled on personal account historically.  
10. **FIGARO IO sectoral shock module** — planned, separate track, not integrated into price model.

## 11. What the next AI should do (safe order)

1. Confirm `git remote -v`, `git status`, and that `data.py` / `features.py` have correct indentation and `load_forecast_mw` (no shift).  
2. Set `ENTSOE_API_KEY`, run `py scripts\update_data.py`, verify load/weather/price last timestamps.  
3. Run `py scripts\run_daily_forecast.py --no-update` once; check delivery date naming and `pre_gate`.  
4. Wire load into `run_ablation.py` / `join_all` if missing; re-run ablation; update README claims only if numbers move.  
5. Keep collecting live pre-gate days; extend rolling eval with `--incremental`.  
6. Do not weaken the 24 h SMARD/weather shift without a **verifiable** publication time source.  
7. Optional research: Δprice vs lag-24 target, gas/CO2 features, quantiles, 15-min products, FIGARO intensities.

## 12. Common failure modes (fix these)

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Forecast horizon jumps months into the past | weather/load NaNs + aggressive dropna | ffill external with limit; dropna only on core lags/price |
| IndentationError in `data.py` | empty `if load_path` body | ensure `parts.append(load_load_forecast(...))` indented under `if` |
| Bias always 0 | too few errors / min_obs / conflicting signs | expect this early; do not force |
| Push 403 | wrong remote or no Write | set-url to own repo + PAT |
| Score table missing a day | incomplete actuals or not pre_gate | wait for full day; check Counted flags |
| UnicodeEncodeError on Windows | arrow character `→` in print | use ASCII `->` |
| Actions not running | account Actions disabled / unpaid PA features | use `run_daily.bat` |

## 13. README markers (do not delete)

- `<!-- KEY_START -->` … `<!-- KEY_END -->`
- `<!-- RESULTS_START -->` … `<!-- RESULTS_END -->`
- `<!-- DAILY_FORECAST_START -->` … `<!-- DAILY_FORECAST_END -->`

Scripts rewrite only those blocks. Hand-written sections 1–4 and 6–9 stay manual.

## 14. One-paragraph summary for papers / LinkedIn

Leakage-aware forecasting of German day-ahead electricity prices under a gate-closure information set: price lags and complete-day context up to D−1 only; SMARD wind/solar and weather lagged 24 h because issue times on the unofficial chart-data endpoint are unverifiable; optional ENTSO-E day-ahead load without lag. Ensemble HGB+Ridge, direct 24 h horizon, seasonal-naive baselines and DM tests. In an 89-day expanding backtest the gate-safe model MAE is about 28 EUR/MWh versus about 38 for SN24 (~26% lower). Live pre-gate evaluation is ongoing and not yet large enough for a performance claim.

---

End of handoff. Prefer changing evaluation and data provenance over changing the model class until leakage and availability rules are settled.
