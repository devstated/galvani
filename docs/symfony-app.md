# Wiring a Symfony app to the stack

`galvani-init` copies the stack files and adds placeholder values to the app's `.env`.
The real values come from the compose files at runtime. These are the app-side bits.

## Packages

```bash
podman compose run --rm composer require \
  symfony/orm-pack \
  symfony/messenger symfony/amqp-messenger \
  symfony/mercure-bundle
```

`galvani-init` sets `extra.symfony.docker=false` in `composer.json`, so Flex recipes no
longer generate their own `compose.yaml` services. A leftover `compose.yaml.flex-bak` can be
deleted.

## Environment variables provided by the stack

| Variable | Used for |
|---|---|
| `APP_ENV`, `APP_DEBUG`, `APP_SECRET`, `DEFAULT_URI` | Symfony core |
| `TRUSTED_PROXIES` | `framework.trusted_proxies` |
| `DATABASE_URL` | Doctrine (PostgreSQL 18) |
| `REDIS_URL` | Cache / sessions / locks (Valkey) |
| `MESSENGER_TRANSPORT_DSN` | Messenger (RabbitMQ) |
| `MAILER_DSN` | Mailer (Mailpit in dev) |
| `MERCURE_URL`, `MERCURE_PUBLIC_URL`, `MERCURE_JWT_SECRET`, `MERCURE_JWT_ISSUER` | MercureBundle |

Every variable the app references must also exist in the committed `.env` (a placeholder
is enough). Symfony resolves some of them while building the cache inside the image.

## config/packages/framework.yaml

```yaml
framework:
    trusted_proxies: '%env(TRUSTED_PROXIES)%'
    trusted_headers: ['x-forwarded-for', 'x-forwarded-proto', 'x-forwarded-host', 'x-forwarded-port']
```

Without this, the app sees `http://` and the proxy's IP instead of the visitor's.

## config/packages/cache.yaml

```yaml
framework:
    cache:
        prefix_seed: my_vendor/my_app
        app: cache.adapter.redis
        default_redis_provider: '%env(REDIS_URL)%'
```

## config/packages/messenger.yaml

```yaml
framework:
    messenger:
        failure_transport: failed
        transports:
            async:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                retry_strategy: { max_retries: 3, multiplier: 2 }
            failed: 'doctrine://default?queue_name=failed'
        routing:
            'App\Message\MyAsyncMessage': async
```

## Mercure

The Mercure recipe's `config/packages/mercure.yaml` works as is: it reads `MERCURE_URL`,
`MERCURE_PUBLIC_URL`, `MERCURE_JWT_SECRET` and `MERCURE_JWT_ISSUER`, all set by the stack.
The hub runs inside the `php` container, so no separate Mercure service is needed.

- Production: subscribers need a JWT. MercureBundle sets the authorization cookie
  (`mercure()` Twig helper / `Authorization` service). To allow anonymous subscribers for
  public topics, set `MERCURE_EXTRA_DIRECTIVES=anonymous` in Coolify.
- Mercure 1.x subscribers use the `match` query parameter:
  `new EventSource('/.well-known/mercure?match=' + encodeURIComponent(topic))`.

## Sessions in Valkey (optional, recommended with several php replicas)

```yaml
# config/services.yaml
services:
    app.session_redis:
        class: Redis
        factory: ['Symfony\Component\Cache\Adapter\RedisAdapter', 'createConnection']
        arguments: ['%env(REDIS_URL)%']
    Symfony\Component\HttpFoundation\Session\Storage\Handler\RedisSessionHandler:
        arguments: ['@app.session_redis', { prefix: 'sess_', ttl: 86400 }]

# config/packages/framework.yaml
framework:
    session:
        handler_id: Symfony\Component\HttpFoundation\Session\Storage\Handler\RedisSessionHandler
```

When sessions or locks are stored in Valkey, set `VALKEY_MAXMEMORY_POLICY=volatile-lru`, so
only keys with a TTL are evicted.

## Worker-mode hygiene

FrankenPHP keeps the kernel in memory between requests:

- Don't keep request data in service properties, or implement `ResetInterface`.
- On a Linux filesystem, dev runs in worker mode automatically. On a Windows/macOS folder,
  check worker-mode behaviour with `GALVANI_WORKER_MODE=on podman compose up -d php`.

## Hot reload (dev)

On a Linux filesystem, dev enables FrankenPHP's hot reload: when a watched file changes,
the browser updates the page without a manual reload. It is never active in production.
Add this to the `<head>` of the base layout (`templates/base.html.twig`):

```twig
{% set hot_reload_url = app.request.server.get('FRANKENPHP_HOT_RELOAD') %}
{% if hot_reload_url %}
    <meta name="frankenphp-hot-reload:url" content="{{ hot_reload_url }}">
    <script src="https://cdn.jsdelivr.net/npm/idiomorph/dist/idiomorph.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/frankenphp-hot-reload/+esm" type="module"></script>
{% endif %}
```

The variable only exists while hot reload is on, so the block renders nothing in
production. Pin the two packages to a version (`.../npm/frankenphp-hot-reload@x.y.z/+esm`)
once the setup works. With Idiomorph, the page is morphed in place; elements that must
survive an update (the Symfony web debug toolbar is the usual example) can be marked with
`data-frankenphp-hot-reload-preserve`. Without Idiomorph, the page reloads fully.

Watched by default: `src/`, `config/`, `templates/`, `translations/`, `assets/` and the
Tailwind output in `var/tailwind/`. Add more with `GALVANI_HOT_RELOAD_CONFIG`, e.g.
`watch /app/lib/**/*.php`.

## Tailwind CSS and daisyUI (no Node)

```bash
podman compose run --rm composer require symfonycasts/tailwind-bundle
podman compose exec php bin/console tailwind:init
podman compose exec php bin/console cache:clear
podman compose restart tailwind
```

The `tailwind` dev service now rebuilds the CSS on every change (`podman compose logs -f
tailwind`); the prod image build runs `tailwind:build --minify` automatically.

daisyUI 5 ships plugin files for the standalone Tailwind binary. Download them next to
the CSS file, pinned to a release, and commit them:

```bash
cd assets/styles
curl -fsSLO https://github.com/saadeghi/daisyui/releases/download/v5.x.y/daisyui.mjs
curl -fsSLO https://github.com/saadeghi/daisyui/releases/download/v5.x.y/daisyui-theme.mjs
```

```css
/* assets/styles/app.css */
@import "tailwindcss";
@plugin "./daisyui.mjs";
```

Replace `v5.x.y` with the current release from the daisyUI releases page. To update
daisyUI, download the files of the new release.

A JavaScript bundler (Rsbuild, Vite) is only worth adding for substantial JavaScript;
see [ADR-8](decisions.md#adr-8-frontend-assets-without-node-by-default).

## Extra PHP extensions

```dockerfile
# Dockerfile (app), in both the dev and prod stages
USER root
RUN install-php-extensions gd
USER 33        # prod stage only
```

## Production php.ini

`exec`, `shell_exec`, `system`, `passthru`, `popen`, `pcntl_exec` and `dl` are disabled in
prod. `proc_open` (Symfony Process, Mailer sendmail) remains available. If an app needs
more, add `docker/php/zz-app.ini` and `COPY` it to `$PHP_INI_DIR/app.conf.d/` in the prod
stage. Think twice before re-enabling anything.
