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
- Check locally with `GALVANI_WORKER_MODE=on podman compose up -d php`.

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
