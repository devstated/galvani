# Changelog

All notable changes to the Galvani base image and app template are documented here.
The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and the
project follows [Semantic Versioning](https://semver.org/). Apps follow the major tag (`:1`).

## [Unreleased]

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
