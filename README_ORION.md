# Orion Nexus — deployment notes

Companion notes for `stack-compose.yml`. Covers what was added, what was
fixed, and the two things that will still break until they are addressed
outside this repo.

## Topology

| Service | Image | Port | Networks | Public host |
|---|---|---|---|---|
| `cebba-web` | `michaelmurwayi/cebba-website:v1.1` | 80 | `CEBBA` | `cebba.ke` |
| `orion-dashboard` | `orion-dashboard:latest` | 8080 | `CEBBA`, `ORION` | `orion.cebba.ke` |
| `orion-api` | `orion:latest` | 8000 | `CEBBA`, `ORION` | `api.orion.cebba.ke` |
| `orion-postgres` | `postgres:16-alpine` | — | `ORION` only | *none* |

Traefik does TLS and HTTP→HTTPS for every public host. Nothing publishes a
host port, so the only way in is through the proxy.

`orion-postgres` is attached to `ORION` and *not* to `CEBBA`. That is
deliberate: it puts the database one network hop away from the edge and off
the proxy's network entirely.

Source repos: API `~/Code/orion/api`, dashboard `~/Code/nexus`.

## Deploy

```bash
export POSTGRES_PASSWORD='...'      # A-Za-z0-9-._~ only — see below
export SECRET_KEY="$(openssl rand -hex 32)"
docker stack deploy -c stack-compose.yml cebba
```

Point `api.orion.cebba.ke` and `orion.cebba.ke` at the Traefik entrypoint
before deploying, or Let's Encrypt will fail.

`POSTGRES_PASSWORD` is interpolated straight into `DATABASE_URL` with no
URL-encoding, so `@ : / ? # @` in the password will corrupt the connection
string. Generate one from a restricted alphabet.

## Known blockers

These are outside this repo and are **not** fixed by this stack.

### 1. `/farmers` routes return HTTP 500 — Redis

`~/Code/orion/api/core/cache.py` builds its client with literals:

```python
redis_client = redis.Redis(host="localhost", port=6379, decode_responses=True)
```

It ignores `REDIS_HOST` / `REDIS_PORT` from `core/config.py`. Inside a
container `localhost` is the API container itself, so the address is never
reachable. `services/farmer_service.py` then calls `set_cache`,
`get_cache` and `delete_cache` with no `try/except` at lines 67, 111, 125,
174, 177, 200, 203, 309 and 425 — each one raises `ConnectionError` and
surfaces as a 500.

No Redis service is deployed, and `REDIS_HOST`/`REDIS_PORT` are left empty
on purpose: adding a Redis container would not fix this, because the
hostname is hardcoded. The suggested fix is to read the settings values and
treat cache failure as non-fatal:

```python
from core.config import settings

redis_client = redis.Redis(
    host=settings.REDIS_HOST or "localhost",
    port=settings.REDIS_PORT,
    db=settings.REDIS_DB,
    decode_responses=True,
)
```

and wrap the call sites in `farmer_service.py` so a cache outage degrades to
a database read instead of an error. Everything except `/farmers` works today.

### 2. `VITE_API_URL` is compiled into the dashboard image

`~/Code/nexus/Dockerfile` bakes the value into the JS bundle at build time.
A container environment variable cannot correct it. If the running image was
built against `http://localhost:8000`, the browser will call its own
machine. Rebuild with the real origin:

```bash
cd ~/Code/nexus
docker build --build-arg VITE_API_URL=https://api.orion.cebba.ke \
  -t orion-dashboard:latest .
```

`CORS_ORIGINS` in `.env` must list the **dashboard** origin
(`https://orion.cebba.ke`), not the API origin. It is parsed as JSON by
pydantic-settings, so it must stay valid JSON.

## Also worth knowing

- The API exposes `/docs` and `/openapi.json` on the public hostname. Add a
  Traefik `stripprefix`/IP-white-list middleware if the schema is sensitive.
- `TEST_DATABASE_URL` points at `orion_test`, which this stack never creates.
  That is fine — only the test suite connects to it — but the API refuses to
  boot if the variable is unset at all.
- The dashboard calls `/mills` and `/warehouses`, but no router is registered
  for either in `api/main.py` (only the ORM models exist). Those calls 404
  regardless of this stack.
