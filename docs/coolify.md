# Production with Coolify

Coolify deploys the **app repository** (which contains the Galvani template files),
not this repository.

## 1. Prepare the server (once)

- Recent Debian/Ubuntu LTS, unattended security upgrades enabled.
- SSH: keys only, no root password login (`PasswordAuthentication no`).
- Firewall: allow only `22`, `80` and `443` from the internet. Allow `8000` (Coolify
  UI) only from administrator IPs, or put the UI behind its own domain with HTTPS.
- Install Coolify, enable 2FA for every Coolify user.
- Nothing in the stack publishes a port. Only Coolify's proxy is reachable from outside.

## 2. Create the resource

1. *Projects → New → Resource → Git repository* (public, or private with a GitHub App or deploy key).
2. **Build Pack:** `Docker Compose`. **Docker Compose Location:** `/compose.yaml`.
3. **Domains:** set the domain only on the `php` service, with the container port:
   `https://app.example.com:8080`. Give the other services no domain.
4. **Environment variables:** add every variable listed in `galvani.env.example`. Generate
   each secret with `openssl rand -hex 32`.
   - **Do not** tick *Build Variable* for any secret. Secrets are needed at runtime only and
     must never end up in an image layer.
   - Required: `APP_DOMAIN`, `APP_SECRET`, `MERCURE_JWT_SECRET`, `POSTGRES_PASSWORD`,
     `VALKEY_PASSWORD`, `RABBITMQ_PASSWORD`, `MAILER_DSN`.
5. Deploy. The `php` service waits for Postgres, Valkey and RabbitMQ to become healthy,
   runs Doctrine migrations (`GALVANI_MIGRATE=1`) and only then reports healthy. The
   `worker` starts after `php`.

If a secret is missing or weak, the affected container exits with a clear `FATAL:` line
in its logs. This is intended.

The base image is public on ghcr.io, so no registry credentials are needed.

## 3. Sizing

The defaults target a **2 vCPU / 4 GB** VPS (limits are ceilings, not reservations):

| Variable | 2c / 4 GB | 8c / 32 GB | 32c / 128 GB |
|---|---|---|---|
| `PHP_MEM_LIMIT` | 1536M | 8G | 24G |
| `FRANKENPHP_WORKER_CONFIG` | *(auto: 2×CPU)* | `num 32` | `num 128` |
| `WORKER_MEM_LIMIT` | 512M | 2G | 4G |
| `POSTGRES_MEM_LIMIT` | 1G | 12G | 64G |
| `POSTGRES_SHARED_BUFFERS` | 256MB | 3GB | 16GB |
| `POSTGRES_EFFECTIVE_CACHE_SIZE` | 768MB | 9GB | 48GB |
| `POSTGRES_WORK_MEM` | 8MB | 32MB | 64MB |
| `POSTGRES_MAX_CONNECTIONS` | 100 | 200 | 300 (+ PgBouncer) |
| `VALKEY_MEM_LIMIT` / `VALKEY_MAXMEMORY` | 384M / 256mb | 4G / 3gb | 16G / 12gb |
| `VALKEY_IO_THREADS` | 1 | 4 | 8 |
| `RABBITMQ_MEM_LIMIT` | 512M | 2G | 4G |

Each FrankenPHP worker holds a PostgreSQL connection, so keep
`num` workers + Messenger consumers below `POSTGRES_MAX_CONNECTIONS`.

For millions of visitors, the next steps beyond one big box are a CDN in front of the
static assets, PgBouncer, a managed or dedicated Postgres, more `worker` replicas, and
a shared Mercure transport when running several `php` replicas.

## 4. Operations

- **Logs:** Coolify UI, or `docker logs`. Every service logs to stdout/stderr. Docker
  logs rotate at 5×10 MB per container.
- **Backups:** the data lives in the named volumes `database_data`, `valkey_data`
  (cache, disposable) and `rabbitmq_data`. Back up PostgreSQL with `pg_dump` at least
  daily, for example as a Coolify *Scheduled Task* on the `database` service:
  `pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB" -Fc > /var/lib/postgresql/backup.dump`
  then ship the file off-site (S3). Test a restore regularly.
- **Scheduler:** set `MESSENGER_TRANSPORTS="async scheduler_default"` to run Symfony
  Scheduler in the worker.
- **More consumers:** add more worker services or raise replicas. The work is
  distributed by RabbitMQ.
- **Postgres major upgrades** (18 → 19) need `pg_upgrade` or dump/restore. They never
  happen automatically, because the image tag pins the major.
- **RabbitMQ upgrades:** one minor at a time (4.3 → 4.4 → …). Enable all feature flags
  before each step: `rabbitmqctl enable_feature_flag all`.

## 5. Network model

```
internet ──443──▶ Coolify proxy (Traefik, TLS) ──▶ php:8080
                                                    │
               default network ── php, worker ──────┤ (egress allowed: mail, APIs)
               backend (internal: true) ── php, worker, database, valkey, rabbitmq
```

The data services are only on the `backend` network, which is internal (no route to
the internet). Coolify additionally attaches its own per-resource network so the proxy
can reach `php`.
