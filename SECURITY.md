# Security

## Supported versions

Only the latest release of the current major version (`:1`) receives fixes. The weekly
rebuild applies upstream security patches to that release automatically.

## Reporting a vulnerability

Please do not open a public issue. Use GitHub's private vulnerability reporting:
<https://github.com/devstated/galvani/security/advisories/new>
Include the affected image tag or file, the impact and steps to reproduce.

## Security model

**This repository contains no secrets and no application code.** The published image
can be public. Production secrets exist only in Coolify's environment configuration.

### Image (`image/`)

- Based on the official FrankenPHP Debian image, with `apt-get upgrade` at build time,
  rebuilt weekly from scratch (no cache).
- Runs as `www-data` (UID 33), numeric `USER` for `runAsNonRoot` checks.
- No file capabilities (upstream `cap_net_bind_service` removed), no setuid/setgid binaries.
- Works with `cap_drop: [ALL]`, `no-new-privileges` and a read-only root filesystem.
- Prod image contains no Composer, no git, no Xdebug.
- `expose_php=Off`, dangerous functions disabled, `allow_url_include=Off`,
  `opcache.validate_timestamps=0`, secure session cookie flags.
- Application code is copied as `root:root`: the PHP process cannot modify it.
- Caddy admin API disabled. The health endpoint listens on loopback only.
- Mercure tokens, `Authorization` headers and cookies are redacted from access logs.
- Security headers by default: `nosniff`, `SAMEORIGIN`, `Referrer-Policy`, COOP,
  `Permissions-Policy`, HSTS. The `Server` header is removed.
- Mercure: no anonymous subscribers and no subscription API unless explicitly enabled.
  Issuer bound to the app's domain.

### Startup checks (fail closed)

With `APP_ENV=prod` the entrypoint refuses to start when:

- a required secret (`APP_SECRET`, Mercure keys) is missing, shorter than 32 characters,
  or contains a placeholder (`ChangeMe`, `dev-only`, …),
- `APP_DEBUG` is enabled,
- `GALVANI_HOT_RELOAD` is enabled (a development feature that publishes file changes),
- `GALVANI_WORKER_MODE` or `GALVANI_HOT_RELOAD` has a value other than `on`, `off` or `auto`
  (checked in every environment).

Valkey and RabbitMQ refuse to start without a password of at least 32 characters.
PostgreSQL refuses to initialise without a password.

### Stack (`template/compose.yaml`)

- No published ports. The reverse proxy reaches `php` over the container network.
- Data services run on an `internal` network with no egress.
- Every container: `no-new-privileges`, `cap_drop: [ALL]`, memory limits, log rotation.
- PostgreSQL, Valkey and RabbitMQ run as their unprivileged users from the start (no
  root entrypoint).
- PostgreSQL: SCRAM-SHA-256 only, data checksums on (PG 18 default).
- Valkey: password via an ACL file in tmpfs (not on the command line);
  `CONFIG`, `DEBUG`, `MODULE`, `MONITOR`, `SHUTDOWN`, `ACL` and replication commands are denied.
- Secrets are runtime-only environment variables, never build arguments.

### Supply chain

- GitHub Actions pinned to commit SHAs and updated by Dependabot.
- Workflow permissions: `{}` by default. The publish job gets only `contents: read`,
  `packages: write` and `id-token: write`.
- Trivy blocks publication on fixable CRITICAL/HIGH findings.
- Images ship with SBOM + SLSA provenance and are signed keylessly with cosign.

### Dev (`template/compose.override.yaml`)

- Every port is bound to `127.0.0.1`.
- Dev credentials are intentionally invalid for prod.
- The dev app container adds back `CHOWN`, `DAC_OVERRIDE`, `FOWNER` and `FSETID`. These are
  needed for bind mounts from Windows/macOS hosts and are not present in prod.
- Hot reload publishes the names of changed files over Mercure, which allows anonymous
  subscribers in dev only. Both are refused or off in prod.
