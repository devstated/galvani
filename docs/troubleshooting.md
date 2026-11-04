# Troubleshooting

## Stack and app

**`no configuration file provided: not found`** when running `podman compose up`
: The app has no Galvani files yet. Run `bin/galvani-init <app-dir>` first.

**A Symfony recipe asks "Do you want to include Docker configuration from recipes?"**
: Answer `x` (no, permanently). Galvani provides the Docker setup; `galvani-init` normally
  sets this (`extra.symfony.docker=false`) and prints *ACTION NEEDED* if it could not.

**Code changes do not show up**
: Check the startup line: `podman compose logs php | grep "worker mode"`. With
  `worker mode: on`, PHP changes in `src/` and `config/` restart the workers
  automatically on a Linux filesystem. To restart them by hand:
  `podman compose exec php curl -s -X POST http://127.0.0.1:2019/frankenphp/workers/restart`.

**Hot reload / Tailwind watcher do nothing**
: The startup line shows `hot reload: off` and the filesystem type. On `v9fs`, `virtiofs`
  or `fuse` (folders shared from Windows/macOS) no file-change events arrive. Move the
  project to a Linux filesystem, see [windows.md](windows.md). For hot reload, the page
  also needs the layout snippet from [symfony-app.md](symfony-app.md#hot-reload-dev).

**`tailwind` service: `exec: .../tailwindcss-linux-x64: Permission denied`**
: The binary download was interrupted (e.g. by a restart) and the incomplete file stays
  in place. Delete it and let the service download it again:
  `podman compose exec php rm -rf var/tailwind && podman compose restart tailwind`.

**`tailwind` service: `You must specify a "binary" or "binary_version"`**
: Run `bin/console tailwind:init`, then `bin/console cache:clear`, then restart the service.

**Pages take seconds to load on Windows/macOS**
: Same cause: every request reads files through the slow host share. Use the WSL setup
  ([windows.md](windows.md)) or, as a stopgap, force worker mode with
  `GALVANI_WORKER_MODE=on` and restart workers after PHP changes.

**`php` stays in `starting` for minutes on the first run**
: The first start runs `composer install`, which is slow through host shares. Follow it
  with `podman compose logs -f php`.

**A production container exits with `FATAL: ...`**
: Intended: a required secret is missing, too short or a placeholder, or a forbidden
  setting is active (`APP_DEBUG`, `GALVANI_HOT_RELOAD` in prod). The message names it.

## Podman machine (Windows)

These are problems of the Podman VM, not of Galvani. When several of them appear
together, **recreating the machine** is the reliable fix. It deletes all images,
containers and volumes inside the VM (code in your folders is not affected):

```bash
podman machine stop
podman machine rm -f
podman machine init --rootful
podman machine start
```

**`netavark (exit code 1): nftables error: "nft" did not return successfully`**
: Restart the machine (`podman machine stop`, `podman machine start`); if it returns,
  recreate it.

**Ports are not reachable from Windows** (`NS_ERROR_CONNECTION_REFUSED`), but the app works
: Narrow it down:
  ```bash
  podman compose exec php curl -sI http://127.0.0.1:8080/   # app inside the container
  podman machine ssh "curl -sI http://127.0.0.1:8000/"      # port inside the VM
  curl.exe -sI http://127.0.0.1:8000/                        # port on Windows
  ```
  If only the last one fails, Windows does not receive the forwarded port. A working
  machine shows `conmon` listening on the port: `podman machine ssh "sudo ss -ltnp"`. If
  nothing listens there, recreate the machine.

**`aardvark-dns failed to start ... open resolv.conf: No such file or directory`**
: The VM lost its DNS configuration, typically after toggling
  `podman machine set --user-mode-networking`. Recreate the machine.

**`API forwarding for Docker API clients is not available`** on `podman machine start`
: Docker-API tools cannot use the Windows Docker pipe, often because Docker Desktop holds
  it. Podman commands are not affected. If `podman compose` reports it cannot connect,
  quit Docker Desktop and restart the machine.

**`Cannot connect to Podman ... connectex: No connection could be made`**
: The machine is not running: `podman machine start`.

VPN clients were checked in one of these cases and were not the cause; test without the
VPN only after the steps above.
