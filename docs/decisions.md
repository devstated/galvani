# Architecture decisions

Each record states the context, the decision and its consequences. New decisions are
appended; a decision that is replaced is marked as superseded rather than deleted.

## ADR-1: One repository for dev and prod

**Context:** the stack needs a development setup (tools, throw-away credentials,
published ports) and a production setup (hardened, secrets from the platform). They
could live in separate repositories or together.

**Decision:** one repository. Production and development are separated by file, not by
repository.

**Reasoning:**

- Dev and prod share most of their definition: the same image (different build target),
  the same services, the same environment variable names. Separate repositories would
  duplicate this and drift apart, and drift between dev and prod is a common source of
  production bugs.
- The parts that differ are small and isolated:
  - `compose.yaml` is the production definition and the **only** file Coolify reads.
  - `compose.override.yaml` adds dev tools, dev credentials and published ports. Only a
    local `compose up` merges it.
  - The image has separate `prod` and `dev` targets. `prod` contains no Composer, no
    Xdebug and no git.
- A split would add no security. The repository contains **no secrets and no application
  code**: production secrets live only in the deployment platform, and the published
  image is safe to be public.

**Safeguards:**

- Every production secret is required and fails closed: if a secret is missing, too
  short or still a placeholder, its service refuses to start (`docker-entrypoint` checks
  plus per-service command guards).
- Dev credentials are deliberately invalid for prod. They contain `dev-only` or are
  shorter than 32 characters, so the prod checks reject them.
- All dev ports bind to `127.0.0.1`. If the override were used on a server by mistake,
  nothing would be exposed publicly.
- Dev-only services (`mailpit`, `adminer`, `composer`) and the RabbitMQ management UI
  exist only in the override file.

**Consequences:** a team that needs different write access for the production
definition than for the dev tooling should protect `template/compose.yaml` with branch
protection or code owners, or move it to a separate repository.

## ADR-2: Valkey instead of Redis

**Context:** Symfony needs a key-value store for cache, and optionally for sessions and
locks. It has to run on a small VPS and scale to high-traffic servers with little
maintenance.

**Decision:** `valkey/valkey:9-alpine`.

| | Valkey 9 | Redis 8 |
|---|---|---|
| License | BSD-3, Linux Foundation | AGPLv3 / RSALv2 / SSPLv1 (tri-license) |
| Protocol / clients | Redis-compatible: phpredis and Symfony `RedisAdapter` / `redis://` DSNs work unchanged | native |
| Scaling up | Multi-threaded I/O (`--io-threads`) scales to many cores and very high request rates on one instance | mostly single-threaded command execution; I/O threads available |
| Memory | New hashtable since 8.1: noticeably less overhead per key | standard |
| Small VPS | Same footprint as Redis (a few MB idle); `maxmemory` caps it | same |
| Maintenance | Official image, frequent patch releases, drop-in upgrades within a major | same |

**Consequences:** on a small server with the defaults (`maxmemory 256mb`, 1 I/O thread),
Valkey behaves like Redis. Larger servers raise `VALKEY_IO_THREADS` and `VALKEY_MAXMEMORY`
without other changes, and Valkey Cluster is protocol-compatible beyond a single machine.
The permissive license removes legal questions for commercial or hosted use. The PHP
extension is still called `redis` (phpredis), and Symfony's DSN stays `redis://`.

## ADR-3: Adminer as the database GUI

**Context:** developers need a browser-based database tool in dev. The candidates were
Adminer and pgAdmin.

**Decision:** Adminer.

**Reasoning:** Adminer is one small container (~30 MB RAM) that starts instantly and
covers browsing, querying, editing and import/export. pgAdmin needs ~200–300 MB RAM plus
its own configuration and storage. The tool exists only in dev, where a light footprint
matters more than pgAdmin's extra features.

## ADR-4: Base image plus per-app template

**Context:** several apps should share one patched runtime, while each app can still
have its own needs.

**Decision:**

- The **base image** (`image/`) holds everything that should be identical and centrally
  patched: PHP, FrankenPHP, extensions, php.ini, Caddyfile, entrypoint and hardening.
  CI rebuilds it weekly.
- The **template** (`template/`) is copied into each app once and then belongs to the
  app. It is deliberately small, because an app may need an extra extension or a
  different Messenger setup.
- Apps follow the **major tag** (`:1`).

**Consequences:** security patches reach apps with their next deploy, and breaking
changes require an explicit bump to `:2`. Template changes are not pushed into apps; they
are re-applied with `galvani-init --force` (see [updating.md](updating.md)). PHP
extensions are compiled once in CI, not on the production server during each deploy.

## ADR-5: TLS terminates at the proxy, the container serves HTTP on :8080

**Context:** Coolify runs a reverse proxy (Traefik) that manages certificates.

**Decision:** FrankenPHP serves plain HTTP on port 8080. TLS, certificates and HTTP/3 are
handled by the proxy.

**Consequences:** the container needs no root, no `cap_net_bind_service` and no ACME
state, so it runs with `cap_drop: [ALL]`. `trusted_proxies` in Caddy and `TRUSTED_PROXIES`
in Symfony are set to private ranges, so client IPs, `https` detection and the Mercure
token audience are correct behind the proxy.

## ADR-6: Worker mode on in prod, off by default in dev

**Context:** worker mode keeps the Symfony kernel in memory between requests, which
greatly increases throughput. Code changes then require a worker restart. FrankenPHP's
`watch` option restarts workers automatically, but relies on inotify, which does not work
through Windows/macOS bind mounts (Podman machine and Docker Desktop use 9p/virtiofs).

**Decision:** worker mode is on in production. Dev defaults to classic mode
(`GALVANI_WORKER_MODE=off`).

**Consequences:** every code change is visible immediately in dev on all host systems.
Behaviour that only appears in worker mode (state leaking between requests) is tested by
running dev with `GALVANI_WORKER_MODE=on`, which is also the recommended setting on native
Linux hosts.

## ADR-7: Mercure secure by default

**Context:** FrankenPHP includes a Mercure hub. Its defaults decide whether anyone can
subscribe to updates.

**Decision:** the hub uses the Mercure 1.0 `issuer` configuration. In production there are
no anonymous subscribers and no subscription API, and the issuer is bound to
`https://<APP_DOMAIN>`. Anything more is an explicit opt-in via `MERCURE_EXTRA_DIRECTIVES`.

**Consequences:** subscribers need a JWT, which MercureBundle provides. The hub keeps its
history in the default local (bolt) transport inside the `php` container. Running several
`php` replicas requires a shared transport (Mercure Enterprise / Cloud).
