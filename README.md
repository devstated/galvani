# Galvani

A hardened container platform for **Symfony 8** applications. Galvani provides:

- a **base image** (FrankenPHP in worker mode with a built-in Mercure hub, PHP 8.5), and
- a **compose template** that gives each app the same stack for **local development**
  (Podman or Docker) and for **production on [Coolify](https://coolify.io)**.

Galvani contains no application code. Each Symfony app lives in its own repository and
pulls in the stack with one command.

| Service | Image | Dev | Prod | Purpose |
|---|---|:-:|:-:|---|
| `php` | `ghcr.io/devstated/galvani-php` (FrankenPHP 1.x, PHP 8.5) | ✅ | ✅ | Web server in **worker mode** + built-in **Mercure** hub |
| `worker` | same image | ✅ | ✅ | Symfony Messenger consumer |
| `composer` | same image (`-dev`) | ✅ | — | Composer with the app's exact PHP + extensions |
| `database` | `postgres:18-alpine` | ✅ | ✅ | PostgreSQL |
| `valkey` | `valkey/valkey:9-alpine` | ✅ | ✅ | Cache (and optionally sessions/locks) |
| `rabbitmq` | `rabbitmq:4.3-alpine` (`-management` in dev) | ✅ | ✅ | Messenger broker |
| `mailpit` | `axllent/mailpit:v1` | ✅ | — | Catches all outgoing mail |
| `adminer` | `adminer:5` | ✅ | — | Database GUI in the browser |

The official image is published by [devstated](https://github.com/devstated). Forks publish
under their own account (see [Running your own fork](#running-your-own-fork)).

## Highlights

- **Secure by default:** non-root containers with all Linux capabilities dropped, no
  published ports in production, data services on an internal network, and startup
  checks that refuse missing, weak or placeholder secrets. See [SECURITY.md](SECURITY.md).
- **Patched automatically:** CI rebuilds the image weekly. Apps follow the major tag
  (`:1`) and pick up fixes on their next deploy.
- **Same stack everywhere:** dev differs from prod only by one override file.
- **Scales from a small VPS to large servers** by changing environment variables only.

## How it fits together

```
galvani (this repo)                               an app (one repo per Symfony app)
├── image/      ──CI builds──▶ ghcr.io/devstated/galvani-php:1, :1-dev
│                                   ▲
└── template/   ──galvani-init──▶   ├── Dockerfile            FROM ghcr.io/devstated/galvani-php:1
                                    ├── compose.yaml          prod (Coolify deploys this)
                                    ├── compose.override.yaml dev  (auto-merged locally)
                                    ├── .dockerignore
                                    └── galvani.env.example   list of prod variables
```

The reasoning behind this layout is in [docs/decisions.md](docs/decisions.md).

## Quick start

**Requirements:** Podman or Docker, plus a compose provider. With Podman, `podman compose`
delegates to `docker-compose` or `podman-compose`; Podman Desktop can install one under
*Settings → Resources → Compose*. With Docker, `docker compose` works as is: replace
`podman` with `docker` in the commands below.

```bash
# 0. New app? (uses the Galvani dev image, so no local PHP is needed)
podman run --rm -v "$PWD:/w" -w /w ghcr.io/devstated/galvani-php:1-dev \
  composer create-project "symfony/skeleton:^8.1" my-app

# 1. Add the stack to the app (writes Dockerfile, compose files, ...)
git clone https://github.com/devstated/galvani.git
galvani/bin/galvani-init ./my-app

# 2. Run it
cd my-app
podman compose up -d --build
podman compose logs -f php        # first start runs composer install
```

`galvani-init` takes the image owner from the clone's `origin` remote (`devstated` for the
official repository, the fork owner for a fork). Pass `--owner <github-owner>` to override it.

| URL | What |
|---|---|
| http://localhost:8000 | App |
| http://localhost:8025 | Mailpit |
| http://localhost:8081 | Adminer (system *PostgreSQL*, server `database`, user/pass/db `app`) |
| http://localhost:15672 | RabbitMQ management (`app` / `app`) |
| `localhost:5432` | PostgreSQL, for IDE database tools |

All ports are bound to `127.0.0.1` only. Change them with `HTTP_PORT`, `MAILPIT_UI_PORT`,
`ADMINER_PORT`, `RABBITMQ_UI_PORT` and `POSTGRES_PORT`.

Everyday commands:

```bash
podman compose run --rm composer require symfony/orm-pack   # Composer
podman compose exec php bin/console doctrine:migrations:migrate
podman compose restart worker                                # pick up handler changes
XDEBUG_MODE=debug podman compose up -d php                  # Xdebug (port 9003)
GALVANI_WORKER_MODE=on podman compose up -d php             # test worker mode locally
```

Next, connect the app to the services: [docs/symfony-app.md](docs/symfony-app.md).

## Production

Coolify deploys `compose.yaml` from the **app repository**, not from this one.
Server preparation, required secrets and sizing are covered in [docs/coolify.md](docs/coolify.md).

## Repository layout

```
image/                        the base image (built and published by CI)
  Dockerfile                  targets: prod (hardened) and dev (Composer, Xdebug)
  conf/caddy/                 Caddyfile (HTTP :8080, Mercure, security headers, worker toggle)
  conf/php/                   php.ini overrides (shared / prod / dev)
  bin/                        docker-entrypoint, galvani-worker, galvani-healthcheck
template/                     files copied into every app by bin/galvani-init
bin/galvani-init              installer for app repositories
.github/workflows/image.yaml  build → smoke test → Trivy scan → push (amd64) → cosign sign
docs/                         decisions, Coolify, updates, Symfony wiring
```

## Releases and versioning

Galvani follows [SemVer](https://semver.org/). Each release tag `vX.Y.Z` publishes
`:X.Y.Z`, `:X.Y` and `:X` (plus the `-dev` variants). A new major version (`v2.0.0`)
signals a breaking change, such as a PHP minor bump, a removed extension or changed
defaults. Apps stay on their major until their `GALVANI_TAG` is raised.

How updates reach apps is described in [docs/updating.md](docs/updating.md). Changes are
listed in [CHANGELOG.md](CHANGELOG.md).

## Running your own fork

Forking is the way to change the base image or to publish it under your own account:

1. Fork the repository and keep it **public**. Public ghcr.io packages have no storage
   or download limits.
2. Push to `main`. The workflow builds, tests and publishes `ghcr.io/<your-account>/galvani-php:edge`.
3. Make the package public once: *your profile → Packages → galvani-php →
   Package settings → Change visibility → Public*.
4. Tag a release so apps get a stable tag:
   ```bash
   git tag -a v1.0.0 -m "v1.0.0" && git push origin v1.0.0
   ```
5. Run `bin/galvani-init` from your fork. It derives the owner from the fork's remote.

CI builds `linux/amd64` only. For ARM servers, see the comment in
[image.yaml](.github/workflows/image.yaml).

## Documentation

- [docs/decisions.md](docs/decisions.md): architecture decisions and their reasoning
- [docs/coolify.md](docs/coolify.md): production deployment and server hardening
- [docs/updating.md](docs/updating.md): how base image and app updates flow
- [docs/symfony-app.md](docs/symfony-app.md): configuration an app needs to use the stack
- [SECURITY.md](SECURITY.md): security model and vulnerability reporting

## Contributing

Issues and pull requests are welcome. Every pull request runs the full build, smoke test
and vulnerability scan. Please report security problems privately, as described in
[SECURITY.md](SECURITY.md).

## License

[MIT](LICENSE). The image also contains third-party software (FrankenPHP, PHP, Debian
packages) under their own licenses.
