# SmartCity Pollution Monitoring — Django Ingestion Service

Django REST-style ingestion service for environmental readings from Raspberry Pi nodes. The service receives the enriched JSON emitted by `read_sensors.py`, validates an API key and readings, and writes raw events and typed time-series data to an **existing PostgreSQL 16 + TimescaleDB** database.

## Contents

1. [Architecture and data flow](#architecture-and-data-flow)
2. [Features](#features)
3. [Project layout](#project-layout)
4. [Prerequisites](#prerequisites)
5. [Database preparation](#database-preparation)
6. [Environment configuration](#environment-configuration)
7. [Authentication and API keys](#authentication-and-api-keys)
8. [Run with Docker Compose](#run-with-docker-compose)
9. [API reference](#api-reference)
10. [Raspberry Pi integration](#raspberry-pi-integration)
11. [Database mapping](#database-mapping)
12. [Tests and verification](#tests-and-verification)
13. [Operations, security, and troubleshooting](#operations-security-and-troubleshooting)
14. [Known limitations](#known-limitations)

## Architecture and data flow

```text
Raspberry Pi nodes (elb_1, elb_2, korca_1, korca_2)
    │  sensors (DHT11, BMP280, UV, rain, sound, SDS011)
    ▼
read_sensors.py
    ├── local CSV backup
    ├── SQLite offline queue and replay
    └── HTTP POST /api/v1/ingest/  (X-Api-Key)
                    │
                    ▼
               Django / Gunicorn
                ├── request validation
                ├── SHA-256 API-key matching and node scoping
                ├── transactional ingestion
                └── idempotency by event_id
                    │
                    ▼
             PostgreSQL 16 + TimescaleDB
                ├── smart_city_schema.ingest_events       (raw payload)
                ├── smart_city_schema.environmental_measurements (typed data)
                ├── cities / locations / gateways / nodes
                └── api_users
```

A successful request returns **HTTP 200** even when its event ID is a duplicate, to maintain compatibility with the Raspberry Pi sender and replay logic.

## Features

- Receives enriched JSON with UUID event IDs, timezone-aware timestamps, node metadata, location, measurements, raw sensor readings, and optional diagnostic/log information.
- Keeps unsupported sensor fields `null` rather than inventing readings.
- Validates measurement field names, numeric types, finite values, latitude/longitude and humidity ranges.
- Authenticates `X-Api-Key` against SHA-256 hashes in `smart_city_schema.api_users`; the authenticated `username` must equal the submitted `node_name`.
- Creates or updates city, location, gateway and node records, then inserts raw events and typed measurements atomically.
- Deduplicates retries using the event UUID; enforces the database's unique `(node_id, measured_at)` measurement rule.
- Serves an OpenAPI 3.1 JSON document and Swagger UI.
- Includes a Docker Compose service and opt-in database integration tests.

## Project layout

```text
smartcity-delivery/
├── README.md                       # overall Raspberry Pi + API documentation
├── django_api/
│   ├── .env.example                # runtime configuration template
│   ├── Dockerfile
│   ├── compose.yaml
│   ├── manage.py
│   ├── requirements.txt
│   ├── openapi.json                 # OpenAPI 3.1 contract
│   ├── config/
│   │   ├── settings.py
│   │   ├── urls.py
│   │   └── wsgi.py
│   ├── monitoring/
│   │   ├── apps.py
│   │   ├── models.py                # unmanaged models for existing tables
│   │   ├── views.py                 # HTTP endpoints and API-key verification
│   │   ├── services.py              # validation and transactional writes
│   │   └── tests/
│   │       ├── test_validation.py
│   │       ├── test_endpoint.py
│   │       └── test_database_integration.py
│   └── sql/
│       ├── original-db-schema.sql
│       ├── fresh-install.sql
│       ├── repair-existing-schema.sql
│       └── provision-api-user.sql
└── raspberry_pi/
    ├── read_sensors.py
    ├── read_sensors.original.py
    ├── deploy-script.yaml
    └── .env.example
```

## Prerequisites

- Python **3.12** for local execution, or Docker Engine and the Docker Compose plugin for container execution.
- The supplied `requirements.txt` pins **Django 6.1.3** and specifies compatible dependency ranges for psycopg, python-dotenv, PyYAML and Gunicorn.
- A reachable **PostgreSQL 16** instance with **TimescaleDB** and the `smart_city_schema` tables installed.
- A PostgreSQL role with appropriate access to the schema and tables; a separate administrative role is needed for the schema repair.
- Network access from the Raspberry Pis to the API host/port, preferably through HTTPS or a secure private network.

## Database preparation

### Existing database (recommended for this project)

The supplied original schema contains an index on `environmental_measurements.gateway_id`, but the column is missing from its table definition. Apply the included targeted repair **after a backup and review**:

```bash
# From django_api/; use your actual DB name, user, and host
psql -h <DB_HOST> -U <DB_ADMIN_USER> -d <DB_NAME> \
  -v ON_ERROR_STOP=1 -f sql/repair-existing-schema.sql
```

The repair adds `gateway_id`, its foreign key to `gateways`, and the corresponding index. It does not delete existing measurement rows.

### New, empty database

`sql/fresh-install.sql` contains a corrected schema for a new installation. Review and run it **only** for a fresh database:

```bash
psql -h <DB_HOST> -U <DB_ADMIN_USER> -d <DB_NAME> \
  -v ON_ERROR_STOP=1 -f sql/fresh-install.sql
```

**Do not** apply the fresh-install script blindly against your existing VM database. The service does not run `manage.py migrate` automatically, and unmanaged models do not create these tables.

## Environment configuration

From `django_api/`:

```bash
cp .env.example .env
```

Edit `.env`:

```dotenv
DJANGO_SECRET_KEY=replace-with-long-random-secret
DJANGO_DEBUG=false
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1,vm-host-or-ip

DB_HOST=host.docker.internal
DB_PORT=5432
DB_NAME=smartcity
DB_USER=smartcity_api
DB_PASSWORD=replace-with-db-password
DB_SSLMODE=prefer

API_REQUIRE_HTTPS=false
API_MAX_BODY_BYTES=262144
```

| Setting | Description |
|---|---|
| `DJANGO_SECRET_KEY` | Random secret; must not use the code's development fallback in deployment |
| `DJANGO_DEBUG` | Keep `false` outside development |
| `DJANGO_ALLOWED_HOSTS` | Comma-separated hostnames/IPs accepted by Django |
| `DB_HOST`, `DB_PORT` | PostgreSQL host and port; `host.docker.internal` maps to the Linux host via Compose `host-gateway` |
| `DB_NAME`, `DB_USER`, `DB_PASSWORD` | Existing database and application credentials |
| `DB_SSLMODE` | psycopg SSL mode (`prefer` by default); configure as required by your DB |
| `API_REQUIRE_HTTPS` | Reject non-secure Django requests when `true`; needs correct TLS/proxy integration |
| `API_MAX_BODY_BYTES` | Maximum request-body size (default 262144 bytes) |

**Connectivity:** PostgreSQL on the VM must listen on an address accessible from the container, permit the application role in `pg_hba.conf`, and have the relevant firewall permissions. Do not expose database port 5432 publicly simply to make containers connect.

## Authentication and API keys

The endpoint uses the `X-Api-Key` request header. It hashes the supplied token with SHA-256 and compares that digest to the active hashes stored in `smart_city_schema.api_users.token_hash`. Create **one token per node** and set the database `username` equal to that node's `NODE_NAME`.

Generate a secure token and hash locally:

```bash
python3 - <<'PY'
import secrets, hashlib
key = secrets.token_urlsafe(32)
print('API_KEY:', key)
print('SHA256:', hashlib.sha256(key.encode()).hexdigest())
PY
```

Insert **only the digest** into PostgreSQL:

```sql
INSERT INTO smart_city_schema.api_users (username, full_name, token_hash, is_active)
VALUES ('elb_1', 'Elbasan Raspberry Pi 1', '<SHA256_HEX_DIGEST>', true);
```

Put the corresponding raw token in the `API_KEY` environment variable of **that** Raspberry Pi. Repeat for `elb_2`, `korca_1` and `korca_2` if those nodes are deployed. Never commit raw tokens to Git or include them in log output.

## Run with Docker Compose

```bash
cd django_api
docker compose up --build -d
docker compose ps
docker compose logs -f api
```

The container runs Gunicorn on port `8000`, and Compose publishes `8000:8000` on the host.

Verify service process availability:

```bash
curl -i http://localhost:8000/health/
```

Expected HTTP 200 body:

```json
{"status":"ok"}
```

**Note:** `/health/` is a process/HTTP health check; it does **not** confirm PostgreSQL availability. Verify an authenticated ingestion request separately. In production, place the API behind a TLS reverse proxy and restrict network access as appropriate.

Local Python alternative:

```bash
cd django_api
python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python manage.py check
python manage.py runserver 127.0.0.1:8000
```

`runserver` is for development, not production.

## API reference

| Method | Endpoint | Purpose | Authentication |
|---|---|---|---|
| `GET` | `/health/` | HTTP liveness status | None |
| `POST` | `/api/v1/ingest/` | Accept one Raspberry Pi event | `X-Api-Key` |
| `GET` | `/api/openapi.json` | OpenAPI 3.1 specification | None |
| `GET` | `/api/docs/` | Swagger UI | None |

Swagger UI uses CDN-hosted JavaScript and CSS; the OpenAPI JSON endpoint remains readable offline.

### Example ingestion request

This is a compact **valid-shape example**, not an exhaustive rendering of all measurements. The running Pi script generates the richer payload.

```bash
curl -i -X POST 'http://localhost:8000/api/v1/ingest/' \
  -H 'Content-Type: application/json' \
  -H 'X-Api-Key: <RAW_NODE_API_KEY>' \
  --data '{
    "event_id":"5d2584c2-3aa4-4e95-9434-b6c5f60fedab",
    "schema_version":"2.1",
    "source_type":"raspberry_pi_api",
    "event_time":"2026-11-23T12:00:00+01:00",
    "received_at":"2026-11-23T12:00:01+01:00",
    "node":{"node_name":"elb_1","node_external_id":"elb_1","microcontroller_model":"Raspberry Pi"},
    "location":{"city_name":"Elbasan","country_name":"Albania","location_name":"Pilot Site 1","latitude":41.1125,"longitude":20.0822},
    "measurements":{"temperature_c":23.4,"humidity_pct":55.2,"pressure_hpa":1013.2,"pm25_ug_m3":9.1,"pm10_ug_m3":16.8,"noise_db":null},
    "raw_sensor_data":{"example":"device readings and diagnostics"}
  }'
```

The request's `node.node_name` must match the API user's `username` or the service rejects it. When reusing the example, generate a **new UUID per new measurement** and a realistic event timestamp. Retain the *same* UUID and payload only when retrying an earlier event.

Typical successful body:

```json
{"event_id":"5d2584c2-3aa4-4e95-9434-b6c5f60fedab","status":"created","node_name":"elb_1","measurement_id":1}
```

Possible `status` values on HTTP 200 are `created`, `duplicate`, and `measurement_already_exists`. If an existing `event_id` is submitted with a different payload, the server rejects it.

### HTTP responses

| Status | Meaning |
|---|---|
| `200` | New event accepted, duplicate event replayed, or measurement timestamp already exists |
| `400` | Invalid JSON |
| `401` | Invalid/missing API key |
| `403` | HTTPS required when that option is enabled |
| `405` | Incorrect HTTP method |
| `413` | Request body exceeds configured maximum |
| `422` | Invalid payload, measurement data, timestamp or node authorization |
| `503` | Database error/unavailable |

## Raspberry Pi integration

The corresponding client is `../raspberry_pi/read_sensors.py`. Set its existing `.env` settings, including:

```dotenv
NODE_NAME=elb_1
PLACE=Pilot Site 1
API_URL=https://YOUR_API_HOST/api/v1/ingest/
API_KEY=<RAW_NODE_API_KEY>
```

Refer to `../raspberry_pi/.env.example` for all required and optional settings (location, altitude, connectivity checks, SMTP alerts, etc.). In the deployed repository, preserve your original Raspberry Pi `requirements.txt` and its package versions: that file was not included in the original upload.

Existing behavior is preserved: five-minute sampling, CSV export, local SQLite buffer, reconnect flush, API retries, SMTP alerts and GPIO cleanup. Buffered events retain their event IDs so replay is idempotent. For the supplied GitHub Actions workflow, place the revised script and existing `requirements.txt` at the repository root, since its SCP step references those paths.

## Database mapping

| Table | API behavior |
|---|---|
| `api_users` | Reads active token hashes; updates `last_used_at` |
| `cities` | Upserts city metadata where provided |
| `locations` | Finds/creates or updates location metadata |
| `gateways` | Upserts gateway only when identified in payload |
| `nodes` | Upserts node identity, location association and metadata |
| `ingest_events` | Inserts event UUID, raw JSON, raw sensor snapshot, decoded JSON, timestamps and auth attribution |
| `environmental_measurements` | Inserts typed sensor values into the Timescale hypertable |
| `alert_rules`, `alerts`, `anomaly_events` | Models exist; ingestion does not generate alerts or anomaly records |

`environmental_measurements` has a composite primary key `(measured_at, id)` and a unique `(node_id, measured_at)` index. Ingestion uses raw SQL compatible with the schema. Do not use the Django measurement model's `.save()` or `.delete()` expecting a simple `id` primary key. The service stores original sensor snapshots, including diagnostic information, in `ingest_events` instead of fabricating columns for fields absent from the schema.

**Timestamp distinction:** `measured_at` is the sensor event time; request `received_at` is supplied by the Pi; database `ingested_at` is generated server-side. Legacy timestamps lacking an offset are interpreted in `Europe/Tirane`.

## Tests and verification

Run checks with Python 3.12 and installed dependencies:

```bash
cd django_api
python3.12 -m pip install -r requirements.txt
python3.12 manage.py check
python3.12 manage.py test monitoring.tests.test_validation monitoring.tests.test_endpoint
```

An opt-in database integration test exists:

```bash
RUN_DB_INTEGRATION=1 python3.12 manage.py test \
  monitoring.tests.test_database_integration --keepdb
```

**Warning:** Run integration tests only with a dedicated, correctly initialized test database. Django's test runner may create a new test database lacking the external TimescaleDB schema; prepare an isolated test environment first. Do not target production for tests that create and delete records.

The provided source delivery includes unit tests, but this README does not assert that Django runtime tests or a live database integration test have passed in your VM.

## Operations, security, and troubleshooting

**Logs and containers**

```bash
docker compose ps
docker compose logs --tail=100 api
docker compose restart api
docker compose down
```

**Common problems**

| Symptom | What to check |
|---|---|
| `400 DisallowedHost` | Add the incoming hostname/IP to `DJANGO_ALLOWED_HOSTS` |
| `401 Invalid API key` | Compare raw Pi token, DB SHA-256 digest and `is_active` |
| `422 API credential is not authorized for node` | DB `api_users.username` must match `node.node_name` exactly |
| `422 Invalid numeric value` | Replace missing/non-finite sensor readings with JSON `null` |
| `503 Database unavailable` | DB host/port, credentials, PostgreSQL listening address, `pg_hba.conf`, firewall, schema permissions |
| Database failure involving `gateway_id` | Apply/review `sql/repair-existing-schema.sql` |
| `/health/` succeeds but ingestion fails | Health check does not query PostgreSQL; test authenticated POST |
| Swagger UI not loading | External Swagger UI assets require CDN connectivity; use `/api/openapi.json` |
| Raspberry Pi queue keeps growing | API address/reachability, node token, response status, Pi logs and `.env` |

**Production hardening checklist**

- Use HTTPS termination, restrict inbound traffic, and configure Django's proxy security settings correctly before enabling `API_REQUIRE_HTTPS=true` behind a reverse proxy.
- Rotate node API keys; protect `.env` files and avoid committing raw tokens or DB passwords.
- Create least-privilege database accounts, and back up the existing PostgreSQL database and Pi SQLite buffers.
- Consider rate limiting, enhanced request logging/metrics and dedicated readiness checks before treating the deployment as production-ready.
- Restrict or protect API documentation endpoints if operational details should not be public.

## Known limitations

- No hardware packages were repinned because the original Raspberry Pi `requirements.txt` was not supplied.
- The provided Compose project runs **the API only**, connecting to PostgreSQL already hosted in your VM; it does not start a new PostgreSQL container.
- `api_users` does not carry an explicit node foreign key; authorization instead requires an exact match between the stored username and payload `node_name`.
- Only the `raspberry_pi_api` source type is accepted by this endpoint.
- Alert/anomaly generation, dashboard UI, sensor calibration and administrative CRUD are outside this implementation.
- A successful liveness check and syntactically valid project files are not evidence of a live end-to-end deployment.

---

**Reference files:** `openapi.json` is the machine-readable API contract; `sql/` contains database setup/repair scripts; `monitoring/services.py` is the definitive implementation of accepted fields and transactional ingestion behavior.
