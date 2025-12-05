# catvsdog/ai

`catvsdog/ai` is a **real-time analytics and forecasting dashboard** that sits on top of the main `catvsdog.live` classifier.

It:

- subscribes to **live classification events** from `catvsdog-live` via WebSocket (`catvsdog-ai-protocol`)
- aggregates **Cat / Dog / Other** counts into 5-minute buckets
- computes **lifetime baselines**, **trending windows**, and **“seasons”** (who is dominating: cats, dogs, or others)
- tracks **pump.fun launch activity** and detects elevated “pump.fun season”
- runs **Holt–Winters** forecasts for the next hour and the next 24 hours
- serves a **standalone dashboard** at `/ai/` (HTML/CSS/JS) showing all of the above

The live instance currently runs at:

- https://catvsdog.live/ai/

This repo contains the separate backend + frontend source for that dashboard, plus zipped snapshots for archival.

> Note: This repository does **not** include SQLite databases or classified images.  
> Those are produced by the running system and configured via `.env`.

---

## Features

- **Real-time stats**
  - Consumes `image-prediction` messages from `catvsdog-live` and tallies Cat / Dog / Other in 5-minute buckets.
  - Computes aggregated stats over multiple windows: 15m, 1h, 4h, 8h, 12h, 24h, 48h, 72h, 1w, 2w, 3w, 1mo.
  - Implements a session-aware notion of “active hours” so downtime doesn’t skew baselines.

- **Trending indicators**
  - For each window, compares `actual` vs `expected` counts using lifetime baselines.
  - Shows simple labels per category:
    - “Trending Up 🚀”
    - “Trending Down 📉”
    - “Stable 🔄”

- **Season detection (Cat / Dog / Other)**
  - Uses the last complete 24 hours of Cat / Dog / Other counts.
  - Computes expectations from lifetime baselines and active hours.
  - Estimates quasi-Poisson dispersion and standardises surplus counts into z-scores.
  - The category with the highest positive z-score above a threshold becomes the current **season**, otherwise **neutral**.
  - Displays a season banner with emoji, intensity bar, and full metric table.

- **Season forecast (next 24 h)**
  - Once per hour at `:02` (UTC), runs additive Holt–Winters with a **damped trend** over the last 7 days of hourly data.
  - Produces a 24-hour forecast for each category, scaled by predicted uptime per hour.
  - Computes z-scores and a relative lift vs baseline to decide the **forecasted season**.
  - Offers two views:
    - **Rolling forecast** (updated hourly)
    - **Static daily forecast** (one immutable forecast per UTC day, stored in `season_forecast_daily`)

- **Next-hour forecast**
  - Every hour at `:02` UTC:
    - builds a 7-day hourly time series per category (complete hours only)
    - fits / updates a Holt–Winters model (period 24, damped trend)
    - predicts the **next hour** for Cat, Dog, and Other
    - computes 95% prediction intervals using a Poisson or Negative Binomial model, depending on observed dispersion
  - The dashboard shows three cards with `point` and `CI` for each class.

- **Pump.fun Activity Monitor**
  - Tracks the total number of pump.fun launches (via the `pool` field on image-prediction messages).
  - Computes a lifetime baseline launch rate (tokens per active hour).
  - Computes a 24-hour window rate and a **lift %** vs baseline.
  - Declares a “pump.fun season” when:
    - there are at least `PUMP_SEASON_MIN_SAMPLES` launches in 24 h, and
    - the 24-h rate exceeds the baseline by at least `PUMP_SEASON_LIFT_THRESHOLD` percent.
  - Shows a dedicated card with:
    - lifetime count
    - baseline tokens/hr
    - last 24h tokens/hr
    - lift%
    - a text line indicating whether the pump.fun activity is “normal” or in “season”.

- **Season images**
  - Scans the `TOKEN_IMAGES_DIR` for recent 10000-confidence images of the current season (Cat / Dog / Other).
  - Enforces a maximum age (`SAFE_IMAGE_MAX_AGE_MINUTES`) for safety.
  - Randomises and shuffles images, then:
    - serves them via REST (`/ai/api/season-images`)
    - pushes new sets via WebSocket as `season-images-update` messages.
  - The frontend renders them as animated “season circles” in the banner.

---

## Repository Structure

At the root of the `catvsdog-ai` repo you will typically have:

- `README.md` – this file
- `LICENSE` – MIT (or your chosen OSS license)
- `catvsdogai_backend.zip` – zipped backend snapshot (Node + SQLite schema)
- `catvsdogai_frontend.zip` – zipped frontend snapshot (HTML/CSS/JS)
- optionally: extracted `backend/` and `frontend/` directories with the same code as the zips

Inside the backend snapshot:

- `backend/`
  - `aiServer.js` – main Express + WebSocket server (ports `PORT` and `STATS_WS_PORT`)
  - `aiDatabase.js` – SQLite schema, time-series rollups, baseline, forecasts, and dispersion estimation
  - `.env.example` – configuration template
  - `package.json` – dependencies (`express`, `ws`, `sqlite3`, `dotenv`, etc.)
  - `database/ai_prediction.db` – SQLite database (if you include a snapshot)
  - scripts and helpers for maintenance and shutdown

Inside the frontend snapshot:

- `frontend/`
  - `index.html` – main dashboard under `/ai/` (uses `<base href="/ai/">`)
  - `css/styles.css` – the full dashboard styling (banner, metrics, forecast cards)
  - `js/aiScript.js` – client logic:
    - initial REST fetches
    - WebSocket handling for stats / tokens / forecasts / season images
    - DOM updates for banner, metrics, counters, and forecast cards
  - `images/` – logo and background assets used by the dashboard

---

## Runtime Requirements

- Node.js 18+ (tested with Node 18.x)
- SQLite 3 (standard on most Linux distros)
- A running `catvsdog-live` backend that:
  - exposes a WebSocket server which accepts the `catvsdog-ai-protocol` subprotocol and emits `image-prediction` messages
  - writes classified token images into `TOKEN_IMAGES_DIR`

Recommended Node dependencies (see `package.json`):

- `express`
- `ws`
- `sqlite3`
- `dotenv`

---

## Configuration

1. Copy the example env file:

   cp `.env.example` to `.env` in the backend directory.

2. Edit `.env` and update the following:

   - `PORT`, `STATS_WS_PORT` – ports for HTTP and stats WebSocket
   - `OTHER_BACKEND_WS_URL` – WebSocket URL of the main `catvsdog-live` backend (e.g. `ws://127.0.0.1:9000`)
   - `DB_PATH` – path to `ai_prediction.db` (relative to backend or absolute)
   - `TOKEN_IMAGES_DIR` – directory where `catvsdog-live` stores classified token images
   - `SAFE_IMAGE_MAX_AGE_MINUTES` – max age for season images
   - `PUMP_SEASON_LIFT_THRESHOLD`, `PUMP_SEASON_MIN_SAMPLES` – pump.fun season detection thresholds

3. Ensure that:

   - `DB_PATH` directory exists and is writable by the backend process.
   - `TOKEN_IMAGES_DIR` points to a location the AI backend can read (but not necessarily write).

---

## Running Locally

Typical development run (backend only):

1. Install dependencies:

   - change to the backend folder
   - run `npm install`

2. Configure:

   - copy `.env.example` to `.env`
   - adjust `.env` for local paths and WebSocket URLs

3. Start the AI backend:

   - `node aiServer.js` (or `pm2 start aiServer.js --name catvsdog-ai` in production)

4. Serve the frontend:

   - `aiServer.js` already serves the static frontend from its `frontend` directory
   - for Nginx: proxy `/ai/` and `/ai/ws/` and `/ai/predict-ws/` to the backend ports

Once both `catvsdog-live` and `catvsdog-ai` are running, visit:

- `https://your-domain/ai/` to see the dashboard.

---

## Relation to `catvsdog-live` and `GENESIS_MACHINE`

- `catvsdog-live`  
  - runs the real-time **classifier** and sentiment engine  
  - produces `image-prediction` events and classified token images  

- `catvsdog/ai` (this project)  
  - subscribes to those events  
  - aggregates, baseline-corrects, forecasts, and visualises the **statistics over time**

- `GENESIS_MACHINE`  
  - uses live **trending words** from `catvsdog-live` to generate synthetic tokens and images  
  - can be used in parallel with `catvsdog/ai`, but is not required

All three projects talk to each other via **HTTP / WebSocket APIs** and **shared image folders** only; each has its own codebase and GitHub repository.

---

## License

This project is released under the MIT License. See the `LICENSE` file for details.
