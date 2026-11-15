# PulsePipe — Deployment

PulsePipe is a single FastAPI process over a file-backed SQLite database, so
deployment is deliberately uncomplicated. This document covers container and
bare-metal options, plus the operational settings that matter in production.

## 1. The unit of deployment

**One process.** Everything — HTTP, dispatch, metrics, alerts, replay — lives
in the ASGI app in `backend/`. Nothing is stateful outside the SQLite file, so
scaling horizontally means scaling **databases**, not processes (see §5).

## 2. Environment (deployment values)

Override any of these at deploy time:

| Variable | Prod recommendation | Notes |
|---|---|---|
| `PULSEPIPE_PORT` / `PULSEPIPE_HOST` | as needed | default `8000` / `0.0.0.0` |
| `PULSEPIPE_DB_PATH` | path to a **persistent volume** | the single source of truth |
| `PULSEPIPE_SEED_ON_START` | `false` after first boot | avoids reseed surprises on restart (it's idempotent anyway) |
| `PULSEPIPE_DISPATCH_MODE` | `async` | decouples ingest latency from subscriber latency |
| `PULSEPIPE_DISPATCH_TIMEOUT` | tune to downstream SLO, e.g. `5.0` | per-subscriber wait |
| `PULSEPIPE_NAME` / `PULSEPIPE_VERSION` | set for your build | shows in `/api/health` |

There is **no built-in auth** — expose PulsePipe only through an authenticated
reverse proxy / API gateway / service mesh, and put it on a private network
segment that your producers and internal subscribers can reach.

## 3. Container (Docker)

Example `Dockerfile` (place at the repo root):

```dockerfile
FROM python:3.14-slim

WORKDIR /srv/pulsepipe
COPY backend/ ./backend/
COPY frontend/ ./frontend/

RUN python -m venv /srv/pulsepipe/backend/.venv \
 && /srv/pulsepipe/backend/.venv/bin/pip install -r backend/requirements.txt

ENV PULSEPIPE_DB_PATH=/data/pulsepipe.db \
    PULSEPIPE_SEED_ON_START=true \
    PULSEPIPE_DISPATCH_MODE=async \
    PULSEPIPE_HOST=0.0.0.0 PULSEPIPE_PORT=8000

# data/ must be writable; mount a persistent volume here.
RUN mkdir -p /data
VOLUME /data
EXPOSE 8000

WORKDIR /srv/pulsepipe/backend
CMD [".venv/bin/uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Run:

```bash
docker build -t pulsepipe .
docker run -d --name pulsepipe \
  -p 8000:8000 \
  -v pulsepipe-data:/data \
  -e PULSEPIPE_DISPATCH_MODE=async \
  pulsepipe
```

> `data/pulsepipe.db` is created on first boot (the `data/` dir is auto-made
> by `config.py`). Mount a volume at `/data` so events and rule state survive
> container recreation.

## 4. Reverse proxy

Terminate TLS and do auth at the edge. Minimal **nginx**:

```nginx
server {
    listen 443 ssl;
    server_name pulsepipe.example.com;
    # ssl_certificate / ssl_certificate_key ...

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 30s;   # leave headroom for sync dispatch
        client_max_body_size 5m;  # cap inbound event payloads
    }
}
```

## 5. Scaling & limits

- **SQLite concurrency** — writes are serialized by the app's `RLock`. For
  modest ingest rates this is fine (thousands of events/sec won't be a
  constraint in most SDR/outbound workloads). If you need horizontal
  parallelism, the app's storage layer (`Database` in `app/database.py`) is the
  single seam to swap out for Postgres — the services only ever call the DAO
  primitives (`execute`/`query`/`query_one`/`dumps`/`loads`) and never touch
  SQLite directly.
- **Single writer** — run exactly one PulsePipe instance writing to a given DB
  file. Multiple read-only copies backed by a shared DB need a network DB.
- **Dispatch timeouts** — `PULSEPIPE_DISPATCH_TIMEOUT` bounds worst-case ingest
  latency when `sync`; use `async` mode for headroom.

## 6. Monitoring & health

- **Liveness/readiness** — use `GET /api/health`; it performs a real
  `SELECT 1` against the DB, so a degraded DB surfaces as a failing probe.
- **Business metrics** — the `/api/metrics/*` endpoints expose throughput,
  counts, latency and queue depth; the dashboard posts them to `/` for humans.
  A production pipeline could scrape these into any monitoring system that can
  call HTTP.
- **Audit trails** — `alerts` and `replay_log` tables give you a point-in-time
  record of alerts fired and replays executed for compliance/observability.

## 7. Backups

Backing up the single `pulsepipe.db` file is the whole game:

- Use `sqlite3 data/pulsepipe.db ".backup backup.db"` for a consistent snapshot.
- Copy the file only when the app is idle for a byte-identical snapshot.
- Include `data/` in your nightly backup and DR restores; the DB is
  self-contained (no separate WAL with default journal settings).

## 8. Common deployment checklist

- [ ] `PULSEPIPE_DB_PATH` points at a persistent, backed-up volume.
- [ ] `PULSEPIPE_DISPATCH_MODE=async` for production ingest.
- [ ] Auth/TLS enforced at the gateway; DB file not publicly reachable.
- [ ] `PULSEPIPE_SEED_ON_START=false` after the first boot.
- [ ] Health check wired to `GET /api/health` in your scheduler/orchestrator.
- [ ] Payload size cap at the proxy to bound memory on `/api/events`.