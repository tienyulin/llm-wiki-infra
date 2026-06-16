# llm-wiki-infra

Shared infrastructure for the [LLM Wiki platform](https://github.com/tienyulin/llm-wiki-mcp):
**one MinIO + one Postgres (pgvector)** on a shared docker network
(`llm-wiki-net`). Start it once; every service attaches to it, so you can run
all services at the same time — they share data and there's no port clash on the
infra (only the apps' own ports 8001/8002/8003 are exposed by the services).

## Start / stop
```bash
docker compose up -d        # creates network llm-wiki-net + minio + pg
docker compose ps
docker compose down         # stop (keep data)
docker compose down -v      # stop + WIPE all shared data (minio + pg volumes)
```

## What it exposes
| | host | in-network hostname |
|---|---|---|
| MinIO API | localhost:9000 | `minio:9000` |
| MinIO console | localhost:9001 (minioadmin/minioadmin) | — |
| Postgres + pgvector | localhost:5432 (wiki/wikipass, db `wiki`) | `pg:5432` |

Services reach these as `minio` / `pg` — their compose env already defaults to
`MINIO_ENDPOINT=minio:9000` and `PG_DSN=postgresql://wiki:wikipass@pg:5432/wiki`,
so no service config is needed.

`db/init/01-extension.sql` enables `vector` + `pg_trgm` on first boot; table DDL
is created by wiki-processor at runtime.

## How services attach
A service compose joins the network like:
```yaml
services:
  wiki-processor:
    networks: [llm-wiki-net]
networks:
  llm-wiki-net:
    external: true
```
So **start this infra before** starting (or opening a dev container for) a service.

## Start everything together
From the platform repo: `scripts/dev-up.sh` brings up this infra + all services
in one shot. See [llm-wiki-mcp](https://github.com/tienyulin/llm-wiki-mcp).
