# Updates

Updates come from two places: the **base image**, maintained in this repository, and
each **app repository**, which owns its copy of the template.

## Base image

| What changes | How it is released | Action needed in apps |
|---|---|---|
| PHP patch (8.5.x), FrankenPHP 1.x, Debian security fixes | Weekly CI rebuild of the latest release tag. `:1`, `:1.x` and `:1.x.y` move | none (picked up on the next deploy) |
| GitHub Action versions | Dependabot pull request, reviewed and merged by maintainers | none |
| Galvani config change (php.ini, Caddyfile, entrypoint) | New minor release `v1.(x+1).0` | none (picked up on the next deploy) |
| Breaking change (PHP 8.6, removed extension, changed defaults) | New major release `v2.0.0` | raise `GALVANI_TAG` to `2`, test, deploy |

A release can never be built from vulnerable layers: if Trivy finds a fixable CRITICAL or
HIGH vulnerability, CI fails and nothing is pushed.

## Apps

1. **Base image updates:** app Dockerfiles use `FROM ghcr.io/devstated/galvani-php:1`.
   Coolify pulls the current `:1` on the **next deploy**. To pick up a security fix
   without code changes, trigger *Redeploy* in Coolify or schedule a regular redeploy.
2. **Symfony / Composer dependencies** are updated in the app repository as usual:
   `podman compose run --rm composer update`, then test, commit and deploy.
3. **Template changes** (new compose features in Galvani) are not pushed into apps,
   because the files belong to the app after `galvani-init`. To re-apply them:
   ```bash
   /path/to/galvani/bin/galvani-init . --force   # keeps *.galvani-bak copies
   git diff                                      # review, keep app-specific changes
   ```
4. **Service images** (`postgres:18-alpine`, `valkey:9-alpine`, `rabbitmq:4.3-alpine`):
   patch releases arrive on redeploy, because the floating tags move. Major and minor
   bumps are a deliberate edit of the app's `compose.yaml` (see the upgrade notes in
   [coolify.md](coolify.md#4-operations)).

## Verifying the image signature

Images are signed keylessly with cosign. To verify one:

```bash
cosign verify ghcr.io/devstated/galvani-php:1 \
  --certificate-identity-regexp 'https://github.com/devstated/galvani/.github/workflows/image.yaml@.*' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```
