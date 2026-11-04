# Developing on Windows

There are two ways to run the Galvani dev stack on Windows:

| | **A: WSL (recommended)** | B: Podman Desktop + Windows folder |
|---|---|---|
| Code lives in | the Linux home folder of a WSL distro | a normal Windows folder |
| Page load (Symfony dev) | fast (worker mode) | ~1–3 s per request |
| Worker restart on code change, browser hot reload, Tailwind watcher | yes | no (classic mode, reload manually) |
| Editor | VS Code with the WSL extension | any |

The difference comes from file sharing: a Windows folder reaches the containers through
a network-style share (9p) that is slow and delivers no file-change events. Galvani detects
this and falls back to classic mode automatically (see [ADR-6](decisions.md#adr-6-worker-mode-and-hot-reload-chosen-by-filesystem-in-dev)).

## A: WSL with Podman inside the distro

1. Install a distro (PowerShell): `wsl --list --online`, then for example
   `wsl --install -d Ubuntu-24.04` (or the newest Ubuntu LTS listed).
2. In the distro, install Podman and git, and check the version (5.x recommended):
   ```bash
   sudo apt update && sudo apt install -y podman git curl
   podman --version
   ```
3. Install a compose provider (Docker Compose v2 as a standalone binary) and enable the
   Podman API socket it talks to:
   ```bash
   mkdir -p ~/.local/bin
   curl -fsSL -o ~/.local/bin/docker-compose \
     https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64
   chmod +x ~/.local/bin/docker-compose
   systemctl --user enable --now podman.socket
   podman compose version
   ```
   (`~/.local/bin` is on the `PATH` after a new login on Ubuntu.)
4. Keep projects in the Linux home folder, **not** under `/mnt/c/...`:
   ```bash
   mkdir -p ~/projects && cd ~/projects
   git clone https://github.com/devstated/galvani.git
   ```
5. Continue with the [quick start](../README.md#quick-start). Open the app in VS Code with
   `code .` from the distro (installs the WSL extension on first use).

### Working with the files from Windows

The project stays in Linux; Windows sees the same files live (no copy, no sync) under:

```
\\wsl.localhost\<distro>\home\<user>\projects
```

Open it in Explorer (`explorer.exe .` from the distro opens the current folder) and pin
it to Quick Access, or map it as a network drive.

- Edit with VS Code connected to WSL (`code .` in the distro): writes go straight to the
  Linux filesystem at full speed.
- Windows apps can also open and save files through the path above. This is slower,
  but the change happens on the Linux side, so worker restarts and hot reload still
  pick it up.
- Use git inside the distro (VS Code's WSL terminal), not Windows Git on the
  `\\wsl.localhost` path: that is slow and fails with "dubious ownership" errors.

To move an existing project from a Windows folder into WSL, clone it (uncommitted changes
stay behind, so commit first):

```bash
git clone /mnt/c/Users/<you>/Documents/Projects/my-app ~/projects/my-app
```

`vendor/` and `var/` are not in git; the first `podman compose up` recreates them.

Containers run rootless as your Linux user, so files created by Composer or
`bin/console` belong to you. Published ports (`localhost:8000`, …) are reachable from
Windows browsers through WSL's localhost forwarding.

A Podman Desktop machine is not needed for this setup and can be stopped.

## B: Podman Desktop with a Windows folder

Works without extra setup; expect slower pages and no automatic reloading.

1. Podman Desktop: start the Podman machine; under *Settings → Resources → Compose* run
   *Setup*.
2. Use **Git Bash** or **PowerShell**; the syntax differs:

| Task | Git Bash | PowerShell |
|---|---|---|
| Run a container with the current folder mounted | `MSYS_NO_PATHCONV=1 podman run --rm -v "$(pwd -W):/w" -w /w IMAGE ...` | `podman run --rm -v "${PWD}:/w" -w /w IMAGE ...` |
| Run `galvani-init` | `galvani/bin/galvani-init ./my-app` | `& "C:\Program Files\Git\bin\bash.exe" -c "galvani/bin/galvani-init ./my-app"` |
| Set a variable for one command | `XDEBUG_MODE=debug podman compose up -d php` | `$env:XDEBUG_MODE="debug"; podman compose up -d php` (stays set for the session) |
| Continue a long command on the next line | `\` | `` ` `` |

In PowerShell, do not call plain `bash`: it may start WSL's bash instead of Git's.

`podman machine ssh` run from Git Bash leaves a file named `NUL` in the current folder,
because Git Bash's `ssh` treats Windows' `NUL` device as a file name. Run it from
PowerShell, or from a folder outside your repositories.

Problems with the Podman machine itself (networking, ports, DNS) are covered in
[troubleshooting.md](troubleshooting.md).
