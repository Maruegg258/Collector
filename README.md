# HYPE Spot Collector

Platform-neutral HYPE/USDC Spot demand collector used by `HYPE_SWING_LONG_PROTOCOL.md`.

## Scope

This repository contains the Collector runtime, storage adapters, tests, and portable operational tooling. It does **not** contain trading decisions, cloud-provider deployment state, production secrets, or a production monitoring schedule.

The Protocol remains the strategy Single Source of Truth. The Collector only gathers, stores, aggregates, and exposes Hyperliquid Spot evidence.

## Data source

- WebSocket: `wss://api.hyperliquid.xyz/ws`
- Market: HYPE Spot `@107`
- Official aggressor side: `B` = buy, `A` = sell
- USDC notional: `px * sz`
- Primary delta: aggressive-buy notional minus aggressive-sell notional
- Deduplication key: `(time_ms, coin, tid)`

Raw trades are retained for a short rolling window (12 hours by default). Completed UTC 4H aggregates and known continuity-gap metadata are retained durably so the Protocol-facing 24H / 3D windows do not require keeping every raw trade indefinitely.

## Protocol-facing API

- `GET /health` — runtime diagnostics and interface version
- `GET /readiness` — WebSocket, storage, and freshness readiness
- `GET /hype/spot-demand` — latest Protocol-facing Spot snapshot
- `GET /hype/spot-demand?completed_4h_end_ms=<UTC_4H_BOUNDARY_MS>` — deterministic completed-boundary query
- `GET /storage/status` — compaction, archive, retention, and storage-monitoring status

The Spot payload uses:

- `schema_version = HYPE-SPOT-PAYLOAD-v1`
- 4H / 24H / 3D windows
- recent completed 4H buckets for CVD direction
- explicit continuity / gap diagnostics
- `boundary_match` for historical-boundary verification

The Collector does not assign the Monitor's final `ROBUST / MARGINAL / UNKNOWN` Decision Usability.

## Storage

Two backends are supported:

- **SQLite** — default for local development and simple single-instance use.
- **PostgreSQL** — recommended when durable external storage or multiple Collector instances are required.

For PostgreSQL deployments, database capacity must be monitored by the database/platform provider. The Collector intentionally reports `EXTERNAL_MONITOR_REQUIRED` because it cannot inspect an external database service volume directly.

## Configuration

| Variable | Default | Purpose |
| --- | --- | --- |
| `PORT` | `8000` | HTTP listen port used by the Docker command |
| `STORAGE_BACKEND` | `sqlite` | `sqlite` or `postgres` |
| `DB_PATH` | `./data/hype_spot.sqlite3` | SQLite database path |
| `DATABASE_URL` | unset | Required when `STORAGE_BACKEND=postgres` |
| `COLLECTOR_INSTANCE_ID` | `HOSTNAME` or generated UUID | Optional stable/unique PostgreSQL lease identity |
| `HYPE_COIN` | `@107` | Hyperliquid HYPE Spot identifier |
| `HYPERLIQUID_WS_URL` | official mainnet WebSocket | Collector source |
| `RAW_RETENTION_HOURS` | `12` | Raw-trade retention |
| `SUMMARY_INTERVAL_SECONDS` | `60` | Runtime summary interval |
| `STORAGE_MAINTENANCE_INTERVAL_SECONDS` | `3600` | Storage maintenance interval |
| `READINESS_MAX_MESSAGE_AGE_MS` | `30000` | Maximum message age for readiness |
| `LOG_LEVEL` | `INFO` | Application log level |

Copy `.env.example` as a starting point for local configuration. Do not commit real credentials.

## Local run

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt pytest
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

On Windows, activate the virtual environment with the appropriate PowerShell or Command Prompt command.

## Docker

Build:

```bash
docker build -t hype-spot-collector .
```

Run with local SQLite storage:

```bash
docker run --rm -p 8000:8000 -v hype-data:/data hype-spot-collector
```

For PostgreSQL, supply `STORAGE_BACKEND=postgres` and `DATABASE_URL` through the target platform's secret/environment-variable mechanism.

The repository intentionally contains no provider-specific deployment manifest or production domain. A future hosting platform only needs to build the Docker image (or run the Python app), provide persistent storage/database access, and configure environment variables.

## GitHub Actions

### CI

`.github/workflows/ci.yml` runs unit tests, the PostgreSQL storage contract, and a non-blocking Hyperliquid live smoke test.

### Monitor payload proxy

`.github/workflows/monitor-payload-proxy.yml` is retained as a **manual diagnostic transport only**. It has no recurring schedule.

Provide the Collector endpoint either:

1. as the workflow input `base_url`, or
2. as the repository variable `COLLECTOR_BASE_URL`.

The workflow validates the same canonical `/readiness` and `/hype/spot-demand` payload; it does not calculate a second Spot signal.

## Tests

```bash
pip install -r requirements.txt pytest
pytest -q
```

## Repository hygiene

The default branch is intended to stay deployment-provider-neutral:

- no production API keys or database passwords
- no hard-coded production host/domain
- no provider-specific deployment identifiers
- no historical strategy backtest workflows
- no automatic HYPE Monitor schedule

Git history may still contain older implementation details, as normal for a version-controlled project; current deployment should always use the latest default branch.
