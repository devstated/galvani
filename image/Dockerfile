# syntax=docker/dockerfile:1
#
# Galvani — hardened FrankenPHP base image for Symfony 8 applications.
#
# Targets:
#   prod  -> ghcr.io/devstated/galvani-php:<version>       (runtime, non-root, no build tools)
#   dev   -> ghcr.io/devstated/galvani-php:<version>-dev   (Composer, Xdebug, git; also used as
#                                                         the *build* stage of app images)
#
# Build locally:
#   podman build --target prod -t galvani-php:local     image/
#   podman build --target dev  -t galvani-php:local-dev image/

ARG FRANKENPHP_VERSION=1
ARG PHP_VERSION=8.5
ARG DEBIAN_SUITE=trixie
ARG COMPOSER_VERSION=2

FROM docker.io/library/composer:${COMPOSER_VERSION} AS composer_upstream
FROM docker.io/dunglas/frankenphp:${FRANKENPHP_VERSION}-php${PHP_VERSION}-${DEBIAN_SUITE} AS frankenphp_upstream

# ---------------------------------------------------------------------------
# base: everything shared by dev and prod
# ---------------------------------------------------------------------------
FROM frankenphp_upstream AS base

SHELL ["/bin/bash", "-euo", "pipefail", "-c"]

WORKDIR /app

# Security updates on top of the upstream image + PHP extensions needed by a typical
# Symfony 8 stack (Doctrine/PostgreSQL, Messenger/AMQP, Cache/Valkey, Intl, ...).
# Apps needing more extensions run `install-php-extensions <ext>` in their own Dockerfile.
# hadolint ignore=DL3005,DL3008
RUN <<-EOF
	apt-get update
	apt-get upgrade -y --no-install-recommends
	apt-get install -y --no-install-recommends file
	install-php-extensions \
		amqp \
		apcu \
		igbinary \
		intl \
		opcache \
		pcntl \
		pdo_pgsql \
		redis \
		zip
	apt-get clean
	rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
EOF

# App-level php.ini overrides live in app.conf.d (scanned after conf.d).
# Apps can drop their own `zz-app.ini` there.
ENV PHP_INI_SCAN_DIR=":$PHP_INI_DIR/app.conf.d"

ENV APP_ENV=prod \
    APP_DEBUG=0 \
    GALVANI_HTTP_PORT=8080 \
    GALVANI_WORKER_MODE=on \
    GALVANI_HOT_RELOAD=off \
    GALVANI_TRUSTED_PROXIES=private_ranges \
    GALVANI_MAX_BODY_SIZE=32MB \
    TRUSTED_PROXIES=private_ranges \
    MERCURE_PUBLISHER_JWT_ALG=HS256 \
    MERCURE_SUBSCRIBER_JWT_ALG=HS256

COPY --link conf/php/10-galvani.ini   $PHP_INI_DIR/app.conf.d/10-galvani.ini
COPY --link conf/caddy/                /etc/frankenphp/
COPY --link --chmod=0755 bin/          /usr/local/bin/

# Caddy state (/data, /config) and the Symfony var/ dir must be writable by www-data;
# the application code itself stays owned by root (read-only for the PHP process).
RUN <<-EOF
	install -d -o www-data -g www-data -m 0750 /data/caddy /config/caddy /app/var
	chown www-data:www-data /data /config
	# Upstream grants frankenphp cap_net_bind_service (for ports 80/443) as a file
	# capability. Galvani only listens on :8080, and a file capability makes exec fail under
	# `cap_drop: [ALL]` + no-new-privileges. Re-copying the binary drops the xattr.
	cp /usr/local/bin/frankenphp /usr/local/bin/frankenphp.nocap
	mv -f /usr/local/bin/frankenphp.nocap /usr/local/bin/frankenphp
	chmod 0755 /usr/local/bin/frankenphp
EOF

EXPOSE 8080

ENTRYPOINT ["docker-entrypoint"]
CMD ["frankenphp", "run", "--config", "/etc/frankenphp/Caddyfile", "--adapter", "caddyfile"]

# ---------------------------------------------------------------------------
# dev: local development + build stage for app images
# ---------------------------------------------------------------------------
FROM base AS dev

SHELL ["/bin/bash", "-euo", "pipefail", "-c"]

ENV APP_ENV=dev \
    APP_DEBUG=1 \
    XDEBUG_MODE=off \
    COMPOSER_ALLOW_SUPERUSER=1 \
    COMPOSER_HOME=/tmp/composer \
    COMPOSER_CACHE_DIR=/composer-cache \
    GALVANI_REQUIRED_SECRETS=""

COPY --link --from=composer_upstream /usr/bin/composer /usr/local/bin/composer

# hadolint ignore=DL3008
RUN <<-EOF
	apt-get update
	apt-get install -y --no-install-recommends git unzip
	install-php-extensions xdebug
	apt-get clean
	rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
	cp "$PHP_INI_DIR/php.ini-development" "$PHP_INI_DIR/php.ini"
	git config --system --add safe.directory /app
	# Dev containers may run with an arbitrary UID (DEV_UID): keep state dirs writable.
	install -d -m 1777 /composer-cache /tmp/composer
	chmod 1777 /data /data/caddy /config /config/caddy /app/var
EOF

COPY --link conf/php/20-galvani.dev.ini $PHP_INI_DIR/app.conf.d/20-galvani.dev.ini

# Runs as root by default: under rootless Podman, container root == the host user,
# so files written to the bind mount keep the host user's ownership. Override with `user:` if needed.
USER root

# ---------------------------------------------------------------------------
# prod: hardened runtime
# ---------------------------------------------------------------------------
FROM base AS prod

SHELL ["/bin/bash", "-euo", "pipefail", "-c"]

ENV GALVANI_REQUIRED_SECRETS="APP_SECRET MERCURE_PUBLISHER_JWT_KEY MERCURE_SUBSCRIBER_JWT_KEY"

COPY --link conf/php/20-galvani.prod.ini $PHP_INI_DIR/app.conf.d/20-galvani.prod.ini

RUN <<-EOF
	cp "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini"
	# Drop setuid/setgid bits: nothing in this image needs to gain privileges.
	find / -xdev -perm /6000 -type f -exec chmod a-s {} + 2>/dev/null || true
EOF

# www-data, numeric so runtimes can verify runAsNonRoot.
USER 33

# OCI labels (overridden/extended by CI metadata).
LABEL org.opencontainers.image.title="galvani-php" \
      org.opencontainers.image.description="Hardened FrankenPHP (worker mode + Mercure) base image for Symfony 8" \
      org.opencontainers.image.licenses="MIT"
