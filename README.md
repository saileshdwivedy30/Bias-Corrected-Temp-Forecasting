# Bias-Corrected Temperature Forecasting

An **end-to-end, online bias-correction pipeline** that

* ingests **GFS** forecasts and **URMA** analyses,
* produces short-range (lead **+3...+8 h**) 2-m temperature forecasts on the **CONUS 0.25°** grid, and
* **learns online from incoming URMA data** to correct systematic GFS errors in real time by modeling residuals on top of GFS.

This enables continuously updated, bias-corrected predictions as new truth arrives.

*Demo window:* **2024-10-01** (one day, due to compute). More days will stabilize and typically increase performance.

---

## Results at a glance (2024-10-01)

> The online bias-corrector beats GFS **most of the time**, with the strongest gains late in the **3–8 h** window and clear **regional pockets** of improvement.

1. **Scoreboard (overall MAE)**
   >Model trims aggregate MAE from **1.25 to 1.24 °C** (**+0.9%**). Positive, consistent with other views.
   
   ![Scoreboard](./scoreboard.png)

2. **MAE by Cycle (00Z/06Z/12Z/18Z)**
   > Beats GFS in **3/4 cycles** (Delta **–0.03 to –0.04 °C** at 06Z/12Z and **–0.02 °C** at 18Z).
   **00Z** is slightly worse (**+0.04 °C**)

   ![MAE by Cycle](./mae_cycle.png)

3. **Spatial Win-rate Map (day aggregate)**
   > Blue = model wins more often; computed per grid cell across the day.”

   > Clear **regional regimes**: broad blue zones = reliable improvement; red pockets = where GFS holds an edge.
   
   ![Winrate Map](./win_rate_map.png)

4. **Win-rate vs GFS (by lead 3–8 h)**

   > Greater than 50% wins at every lead, peaking **\~60–61%** at **+7–8 h**; near parity at **+5 h**.

   ![Winrate by Lead](./model_beat_gfs.png)


---

## System overview

```
Herbie (GFS/URMA)  ─┐
                    ├─► Bronze  (raw grids + ingest log)
Regrid URMA → GFS ──┤
                    ├─► Silver  (URMA on GFS grid + engineered features)
Bias MLP (online) ──┤
                    └─► Gold    (predictions, training log, daily metrics, views)
```

**Bronze:** `bronze_gfs_grid`, `bronze_urma_grid`, `bronze_ingest_log`

**Silver:** `silver_urma_on_gfs` (URMA regridded), `silver_gfs_features` (GFS + time/geo feats)

**Gold:** `predictions_grid_online`, `online_training_log`, `daily_metrics` (+ helper views)

**Block-aware timeline**

* **B0** predicts **+3,+4,+5 h**; **B1** predicts **+6,+7,+8 h**.
* As URMA lands for a block, update the model **online** (Adam + SmoothL1).
* **Canary guardrail:** if Model MAE > GFS MAE + **0.05 °C**, **rollback**.

---

## Modeling notes

* **Target:** residual learning on top of GFS (`Y ≈ GFS + f(features)`), predict a **3-lead block** jointly.
* **Features:** local GFS temps (context & targets), lat/lon trig terms, hour-of-day & day-of-year sin/cos, cycle index.
* **Network:** MLP (64×2, dropout 0.1), sized for fast online updates.
* **Scaler:** streaming Welford variance; persisted to JSON.
* **Why blocks?** Low-latency predictions for the next 3 h while truth for the previous block arrives means **tight feedback, safer updates**.

---

## Ops sanity & data quality

* **Completion:** per-cycle `b0_done/b1_done` and cycle/blocks-done tallies.
* **Lead health:** 5% per-lead row-count deviation alert.
* **Nulls/dupes:** explicit scans; merges are keyed/idempotent.
* **Coverage:** URMA regridded to the GFS CONUS grid (consistent truth).
* **Ingest log:** per asset/hour status (`started/success/skip/failed`) for audit.

---

## Limitations & next steps

* **Scope:** One-day demo; extend to weeks/months for stable skill and regime analysis.
* **Features:** add **terrain/elevation, land-use, coastal distance, dT/dt**, cloud/radiation proxies.
* **Modeling:** try **cycle/region-aware heads** or mixture-of-experts; add **uncertainty (quantiles)**.

---

### Glossary

* **URMA:** Unrestricted Mesoscale Analysis (verification truth).
* **Lead hour:** Forecast hour relative to cycle time.
* **Canary:** Quick verification that gates model updates (with rollback).
* **Win-rate:** % of points where **|Model−Obs| < |GFS−Obs|**.

