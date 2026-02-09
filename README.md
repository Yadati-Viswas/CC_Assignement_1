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
db-1   | Database system is ready to accept connections
db-1   | Executing /docker-entrypoint-initdb.d/init.sql ...
db-1   | Initialization complete
app-1  | Connecting to database at db:5432 ...
app-1  | Running query: SELECT * FROM some_table;
app-1  | Found 3 rows, writing results to /out/results.json
app-1  | Job finished successfully
```

- These logs appear in your terminal with `docker compose up`.
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

## Useful commands

```bash
# Start (foreground)
docker compose up --build

# Start detached
docker compose up -d

# Tail logs from all services
docker compose logs -f

# Tail only app logs
docker compose logs -f app

# Tail only db logs
docker compose logs -f db

# Stop and remove containers (keep volumes)
docker compose down

# Stop and remove containers + volumes (reset DB)
docker compose down -v
```

If you work with the Makefile, you might also use:

```bash
make build   # build images
make up      # start services
make down    # stop services
make logs    # (if defined) follow logs
```

---

## Where outputs are written

- On host: `./out/` in the project root.
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
