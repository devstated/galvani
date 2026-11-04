# Changelog

All notable changes to the Galvani base image and app template are documented here.
The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and the
project follows [Semantic Versioning](https://semver.org/). Apps follow the major tag (`:1`).

## [Unreleased]

### Added
- Browser hot reload for development (`GALVANI_HOT_RELOAD=on|off|auto`) through FrankenPHP's
  `hot_reload` and the built-in Mercure hub. Extra watch patterns via `GALVANI_HOT_RELOAD_CONFIG`.
- `auto` value for `GALVANI_WORKER_MODE`: worker mode on Linux filesystems, classic mode on
  host folders shared into a VM (Windows/macOS), where file-change events don't arrive.
- Startup log line with the active worker/hot-reload mode and the detected filesystem.
- Template: `tailwind` dev service (`tailwind:build --watch`, idles until
  `symfonycasts/tailwind-bundle` is installed); the prod build runs `tailwind:build --minify`.
- Docs: Windows guide (WSL setup, PowerShell commands), troubleshooting guide, hot reload
  and Tailwind/daisyUI setup, ADR-8 (frontend without Node).

### Changed
- Dev defaults: `GALVANI_WORKER_MODE` and `GALVANI_HOT_RELOAD` default to `auto` (was:
  worker mode `off`). Worker `watch` is limited to `src/`, `config/`, `translations/`,
  `.env*` and `composer.lock`.
- `galvani-init` runs Composer in the Galvani dev image instead of `composer:2`, and ends
  with an *ACTION NEEDED* summary (exit code 2) when a step could not be done.
- CI smoke test covers all worker/hot-reload combinations and invalid values.

### Security
- The entrypoint refuses `GALVANI_HOT_RELOAD=on` when `APP_ENV=prod`.
- `GALVANI_WORKER_MODE` and `GALVANI_HOT_RELOAD` accept only `on`, `off` or `auto`; the
  values are used in Caddyfile import paths.

## [1.0.0] - 2026-11-04

### Added
- Base image `galvani-php` (FrankenPHP 1.x, PHP 8.5, Debian trixie) with `prod` and `dev` targets.
- PHP extensions: amqp, apcu, igbinary, intl, opcache, pcntl, pdo_pgsql, redis, zip (plus xdebug in dev).
- Caddyfile: HTTP :8080 behind a TLS proxy, worker-mode toggle, built-in Mercure (1.0 issuer configuration), security headers.
- Entrypoint with fail-closed secret checks, optional Composer install (dev) and Doctrine migrations.
- `galvani-worker` (Messenger consumer) and `galvani-healthcheck`.
- App template: Dockerfile, production `compose.yaml` (Coolify), development `compose.override.yaml`
  (Mailpit, Adminer, RabbitMQ UI, Composer), `galvani.env.example`.
- `bin/galvani-init` installer.
- CI: build, smoke test, Trivy scan, amd64 push to ghcr.io, cosign signing, weekly rebuild.
- MIT license.

[Unreleased]: https://github.com/devstated/galvani/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/devstated/galvani/releases/tag/v1.0.0
