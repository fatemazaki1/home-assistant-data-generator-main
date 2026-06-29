<h1 align="center">⚡ HA Energy Data Generator</h1>

<p align="center">
  <b>Bootstrap Home Assistant energy forecasting on day one</b> — generate realistic,
  physics-based synthetic consumption history from a description of your home, load it
  into Home Assistant exactly as if it were recorded, and get a working load forecast
  without waiting months for real data to accrue.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.11+-3776AB?logo=python&logoColor=white" alt="Python">
  <img src="https://img.shields.io/badge/FastAPI-009688?logo=fastapi&logoColor=white" alt="FastAPI">
  <img src="https://img.shields.io/badge/React-18-61DAFB?logo=react&logoColor=black" alt="React">
  <img src="https://img.shields.io/badge/Home%20Assistant-41BDF5?logo=home-assistant&logoColor=white" alt="Home Assistant">
  <img src="https://img.shields.io/badge/Kubernetes-Helmfile-326CE5?logo=kubernetes&logoColor=white" alt="Kubernetes">
  <img src="https://img.shields.io/badge/License-MIT-green.svg" alt="License: MIT">
</p>

<p align="center">
  <img src="report/figs/fig_loadprofile.png" width="78%" alt="Synthetic daily load profile by device">
</p>

---

## 🤔 Why

Energy-forecasting models need **months of history** to train. A fresh Home Assistant
install doesn't have it — so the forecast is useless exactly when you're most curious about it.

This project closes that cold-start gap. You describe the home (rooms, devices, lifestyle,
location); it **simulates a physically-plausible hourly energy history** with genuine daily
cycles, weekday/weekend patterns, and weather-driven heating/cooling — then packages it in the
*exact* formats Home Assistant records natively. HA trains on it as if it were real, and you
immediately get a 48-hour forecast with uncertainty bands, a plain-language explanation, and
anomaly detection.

## ✨ Features

- 🏠 **Describe-your-home wizard** — pick rooms, drop in devices with a kWh estimate, choose a lifestyle.
- 🔬 **Physics-based simulation** — five device archetypes (always-on, thermostatic, occupancy-driven, cyclic, EV) driven by real weather (Open-Meteo) and occupancy models.
- 🎯 **Calibrated** — your kWh estimate sets the *total*; the archetype sets the realistic *shape*.
- 📦 **HA-native output** — `energy_history.csv`, hourly `weather.csv`, per-appliance CSVs, a `home-assistant_v2.db` recorder database, and a ready `apps.yaml`.
- 🤖 **Trains the real model** — verified end-to-end against [ha-energy-forecast](https://github.com/m-zenker/ha-energy-forecast); the backfill reconstructs the data row-for-row and the model trains on it.
- 🔮 **Live forecasting** — today / tomorrow / next-3h totals with P10–P90 bands, plus SHAP explanations and z-score anomaly detection.
- ⚙️ **Production stack** — FastAPI + async SQLAlchemy + Redis, JWT auth (no public signup), background generation with progress/checkpointing.
- ☸️ **Ship it** — Helmfile bundle for Kubernetes, reusing your cluster's Postgres/Redis.

## 🧭 How it works

<p align="center">
  <img src="report/figs/fig_architecture.png" width="92%" alt="System architecture">
</p>

Describe the home → the generator produces an hour-by-hour history → it is exported in HA's own
formats → HA's backfill loads it → the forecasting app trains and publishes sensors → a dashboard
visualises the result. Each device is simulated independently per hour, then summed:

<p align="center">
  <img src="report/figs/fig_pipeline.png" width="92%" alt="Simulation pipeline">
</p>

> 📄 **Full details — the physics, the math (with every variable explained), the model, backfilling,
> live forecasting, and anomaly detection — are in the [technical report](report/paper.pdf).**

## 🔮 The forecast in Home Assistant

The trained model publishes forecast sensors that the dashboard renders. Each numbered element:

<p align="center">
  <img src="report/figs/fig_dashboard.png" width="42%" alt="Energy Forecast dashboard (annotated)">
</p>

1. **Forecast tiles** — predicted energy for **Today** (Heute), **Tomorrow** (Morgen) and the **next 3h**, each with **Min/Max**: the P10–P90 band from two quantile models. A wider band means the model is less certain.
2. **Unusual-consumption status** — the anomaly check. Green **"No unusual consumption"** by default; it flips to a red alert when the residual (actual − predicted) exceeds *k·σ* of recent errors.
3. **"What Drives This Forecast?"** — a plain-language reading of the model's feature contributions: what pushed today's number up or down.
4. **Feature-importance table** — the same, quantified: each input's contribution in kWh. Here `lag_1h` (the most recent hour's consumption) dominates, with small corrections from solar radiation and time-of-day:

<p align="center">
  <img src="report/figs/fig_shap.png" width="68%" alt="SHAP feature importance">
</p>

## 🚀 Quick start

**1. Dependencies (Postgres + Redis):**
```bash
docker run -d --name hadg-pg    -e POSTGRES_DB=energy -e POSTGRES_USER=app -e POSTGRES_PASSWORD=app -p 5433:5432 postgres:16-alpine
docker run -d --name hadg-redis -p 6380:6379 redis:7-alpine
```

**2. Backend (FastAPI):**
```bash
cd backend
python -m venv .venv && .venv/bin/pip install -e ".[dev]"
cp .env.example .env          # then edit ports/secrets to taste
.venv/bin/alembic upgrade head
.venv/bin/uvicorn src.api.main:app --port 8092
```

**3. Frontend (Vite + React):**
```bash
cd frontend
npm install
VITE_API_PROXY=http://127.0.0.1:8092 npm run dev   # http://localhost:3000
```

Sign in with the seeded admin (`ADMIN_USERNAME` / `ADMIN_PASSWORD` from your `.env`), build a
household in the wizard, and generate a dataset.

## 📦 What you get

| File | Purpose |
|------|---------|
| `energy_history.csv` | Primary training input — hourly `timestamp, gross_kwh` |
| `weather.csv` | Matching hourly weather series |
| `home-assistant_v2.db` | HA recorder statistics — run the forecast app's backfill against it |
| `appliances/*.csv` | Per-device hourly kWh (optional sub-sensor tracking) |
| `apps.yaml` | Ready-to-use AppDaemon config snippet |

## 🧰 Tech stack

| Layer | Tech |
|-------|------|
| **Backend** | FastAPI · SQLAlchemy 2 (async) · PostgreSQL · Redis · Alembic · NumPy/pandas engine |
| **Frontend** | Vite · React · TypeScript · Tailwind · TanStack Query |
| **Infra** | Helm · Helmfile · External Secrets · Vault · cert-manager · ingress-nginx |
| **Forecast model** | Gradient-boosted trees (LightGBM, scikit-learn fallback) on `log1p` consumption |

## ✅ Verification

The generator isn't just plausible-looking — it's **format-faithful**:

- The forecast repo's backfill differences the cumulative meter and recovers `gross_kwh` **row-for-row** (Fig. 4 in the report).
- The actual ha-energy-forecast model **trains successfully** on the generated CSVs.
- Engine unit tests cover archetype calibration, reproducibility (seeded), and the output-contract validator.

```bash
cd backend && .venv/bin/python -m pytest tests/ -q
```

## ☸️ Deployment

A Kubernetes Helmfile bundle (backend + frontend) lives in [`deploy/`](deploy/README.md). It runs
in its own namespace, reuses an existing cluster Postgres/Redis, and exposes the UI and API via
ingress. See [`deploy/README.md`](deploy/README.md) for the step-by-step guide.

## 📁 Repository layout

```
backend/    FastAPI app + simulation engine (see backend/src/domain/ for the engine)
frontend/   Vite + React wizard UI
deploy/     Helmfile bundle + deployment guide
report/     Technical report (PDF + Typst source + figures)
```

## 📄 License

Released under the [MIT License](LICENSE).
