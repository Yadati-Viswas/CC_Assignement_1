# CC_Assignement_1

This project runs a small **Python** application together with a dedicated database, both containerized and orchestrated via Docker Compose. The goal is to make it trivial for anyone to build, run, and inspect the app and its outputs without installing Python or the database locally.

## What this stack does

- Builds and runs a Python app from `app/` using its `Dockerfile` and `main.py`.
- Builds and runs a database from `db/` using its `Dockerfile` and initializes schema/data from `init.sql`.
- Wires the app and DB services via `compose.yml`, and exposes an `out/` directory where the app writes result files on the host.

The app typically connects to the DB, performs a query or computation, and writes its results to `out/` before exiting or staying alive depending on how `main.py` is implemented.

---

## Project structure

```text
CC_Assignement_1/
├── app/
│   ├── Dockerfile      # App image definition
│   └── main.py         # Python application entrypoint
├── db/
│   ├── Dockerfile      # Database image definition
│   └── init.sql        # Schema and seed data
├── out/                # Output directory (mounted into app container)
├── compose.yml         # Docker Compose stack definition
├── Makefile            # Helper targets for build/run (if used)
└── .DS_Store           # macOS metadata (can be ignored)
```

---

## Prerequisites

- Docker Engine (20.x or later) installed and running.
- Docker Compose v2 / `docker compose` CLI.
- Git.

No global Python or database installation is required; everything runs inside containers.

---

## How to build and run

From the repository root:

```bash
git clone https://github.com/Yadati-Viswas/CC_Assignement_1.git
cd CC_Assignement_1
```

### Start the stack

Option A – direct Docker Compose:

```bash
# Build images and start services in the foreground
docker compose up --build
```

Option B – via Makefile (if you prefer):

```bash
# Check the Makefile, but typically:
make up        # start services
# or
make build     # build images only
```

This will:

- Build the `db` image from `db/Dockerfile` and run `db/init.sql` at container startup.
- Build the `app` image from `app/Dockerfile` and start the Python application.

### Stop the stack

```bash
# Stop and remove containers, keep volumes
docker compose down

# Stop & also remove volumes (DB data reset)
docker compose down -v
```

or via Makefile (if targets exist):

```bash
make down
```

---

## Example output

A typical successful run looks roughly like:

```text
db-1  | 2026-02-09 04:28:27.906 UTC [48] LOG:  database system is shut down
db-1  |  done
db-1  | server stopped
db-1  | 
db-1  | PostgreSQL init process complete; ready for start up.
db-1  | 
db-1  | 2026-02-09 04:28:28.018 UTC [1] LOG:  starting PostgreSQL 16.11 (Debian 16.11-1.pgdg13+1) on aarch64-unknown-linux-gnu, compiled by gcc (Debian 14.2.0-19) 14.2.0, 64-bit
db-1  | 2026-02-09 04:28:28.019 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
db-1  | 2026-02-09 04:28:28.019 UTC [1] LOG:  listening on IPv6 address "::", port 5432
db-1  | 2026-02-09 04:28:28.025 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
db-1  | 2026-02-09 04:28:28.038 UTC [66] LOG:  database system was shut down at 2026-02-09 04:28:27 UTC
db-1  | 2026-02-09 04:28:28.044 UTC [1] LOG:  database system is ready to accept connections
Container cc_assignment_1-db-1 Healthy 
app-1  | === Summary ===
app-1  | {
app-1  |   "total_trips": 6,
app-1  |   "avg_fare_by_city": [
app-1  |     {
app-1  |       "city": "Charlotte",
app-1  |       "avg_fare": 16.25
app-1  |     },
app-1  |     {
app-1  |       "city": "New York",
app-1  |       "avg_fare": 19.0
app-1  |     },
app-1  |     {
app-1  |       "city": "San Francisco",
app-1  |       "avg_fare": 20.25
app-1  |     }
app-1  |   ],
app-1  |   "top_by_minutes": [
app-1  |     {
app-1  |       "city": "San Francisco",
app-1  |       "minutes": 28,
app-1  |       "fare": 29.3
app-1  |     },
app-1  |     {
app-1  |       "city": "New York",
app-1  |       "minutes": 26,
app-1  |       "fare": 27.1
app-1  |     },
app-1  |     {
app-1  |       "city": "Charlotte",
app-1  |       "minutes": 21,
app-1  |       "fare": 20.0
app-1  |     },
app-1  |     {
app-1  |       "city": "Charlotte",
app-1  |       "minutes": 12,
app-1  |       "fare": 12.5
app-1  |     },
app-1  |     {
app-1  |       "city": "San Francisco",
app-1  |       "minutes": 11,
app-1  |       "fare": 11.2
app-1  |     },
app-1  |     {
app-1  |       "city": "New York",
app-1  |       "minutes": 9,
app-1  |       "fare": 10.9
app-1  |     }
app-1  |   ]
app-1  | }
app-1 exited with code 0
```

- These logs appear in your terminal with `docker compose up --build`.
- Result files are written into `./out/` on your host machine (mapped inside the container, usually to `/out`).

You can inspect them with:

```bash
ls -R out
```

---

## Configuration

The concrete variable names may differ based on your `compose.yml`, but typical values are:

- Database connection:
  - `DB_HOST=db`
  - `DB_PORT=5432`
  - `DB_USER`, `DB_PASSWORD`, `DB_NAME`
- Volumes:
  - `./out` on the host mapped to `/out` in the app container.

To change ports, credentials, or volume mappings, edit `compose.yml` and restart the stack.

---

## Where outputs are written

- On host: `./out/summary.json` in the project root.
- In the app container: whatever directory `./out` is mapped to (commonly `/out`).

Ensure `out/` exists before running:

```bash
mkdir -p out
```

---

## Troubleshooting

### Database not ready / connection refused

**Symptoms:**

- App logs show `Connection refused`, `database system is starting up`, or similar errors.

**Fix:**

- Check DB logs:

  ```bash
  docker compose logs db
  ```

- Restart after DB is healthy:

  ```bash
  docker compose down
  docker compose up --build
  ```

If this happens often, add retry logic in `main.py` (loop + small sleep until DB becomes available) or configure `depends_on` with healthchecks in `compose.yml`.

### Permission errors on `out/`

**Symptoms:**

- Error like `Permission denied` when writing to `/out/...`.

**Fix on host:**

```bash
mkdir -p out
chmod 777 out   # or a more restrictive mode appropriate for your system
```

Then restart:

```bash
docker compose down
docker compose up --build
```

Also verify that `compose.yml` correctly maps `./out` into the app container.

### Port already in use

**Symptoms:**

- Docker error `port is already allocated` when starting.

**Fix:**

- Find what is using the port and stop it, or change the published port in `compose.yml` (e.g. `5433:5432` or similar), then:

```bash
docker compose down
docker compose up --build
```

---

**Note:** If you know what `main.py` currently does (e.g., specific query or computation), you can tune the example output and description to match it exactly.
