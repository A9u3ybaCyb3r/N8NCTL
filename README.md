# N8NCTL

**Standalone control script for a lone n8n Docker container**
Environment: Kali/Parrot Linux (Docker CLI) · Project path: `~/n8n-project` (flat, single folder)
Status: Built and verified working — single-folder layout + full lifecycle commands (start/stop/update/backup/restore/shell/config/remove)

`n8nctl` runs n8n by itself — no `kali-pentest` tooling container involved.
Analysis tools are reached on the **host machine** directly, via an SSH
node or HTTP Request node inside n8n, using a built-in
`host.docker.internal` mapping. It's the standalone counterpart to
`n8nkali`, for setups that would rather not run a second container at all.

---

## When to use this instead of `n8nkali`

| | `n8nkali` | `n8nctl` |
|---|---|---|
| Manages | n8n + kali-pentest (+ optional caddy, kali-pentest-ad) | n8n only, nothing else |
| Tool execution model | HTTP listener inside a dedicated `kali-pentest` container | SSH or HTTP Request node reaching the host OS directly |
| Use when | You want tool isolation — a disposable, namespaced container | You'd rather not run a second container, and accept the higher trust/risk trade-off of reaching the host directly |

**Container name collision:** both scripts default to naming the container `n8n`. Docker requires unique names on a host — stop one before starting the other:

```bash
n8nkali stop      # if switching to n8nctl
./n8nctl stop     # if switching back to n8nkali
```

---

## Folder layout — flat, single folder

```
~/n8n-project/
├── n8nctl              ← the script
├── n8nctl.env           ← optional overrides, auto-sourced if present
├── n8n-data/            ← docker volume — n8n's persistent state
└── backups/             ← timestamped tar.gz snapshots (created on first `backup`)
```

Everything lives beside the script itself — move the whole folder anywhere and it keeps working with no edits.

### Migrating an existing install

```bash
mkdir -p ~/n8n-project
mv /old/path/bin/n8nctl ~/n8n-project/
mv /old/path/n8n-data ~/n8n-project/      # if it already exists
cd ~/n8n-project
chmod +x n8nctl
./n8nctl config                            # confirm paths before running start
```

### Optional local overrides — `n8nctl.env`

```bash
cat > ~/n8n-project/n8nctl.env <<'EOF'
N8N_TZ="America/New_York"
N8N_IMAGE_TAG="1.68.0"
EOF
```

Auto-sourced at startup, before flags are parsed. **Precedence:** flags > environment variables > `n8nctl.env` > script defaults.

---

## Installation

```bash
chmod +x n8nctl
```

---

## Usage

```
./n8nctl <command> [options]
```

Run `./n8nctl help` any time. Every mistake — unknown command, unknown flag, missing flag value — prints the specific error and then the full help text automatically.

### Commands

| Command | What it does |
|---|---|
| `start` | Create/start n8n |
| `stop` | Stop n8n (container preserved) |
| `restart` | Stop then start |
| `update` | Pull latest (or `--image-tag`) and recreate, preserving data |
| `status` | Show container status |
| `config` | Show effective settings without starting anything |
| `logs` | Follow n8n logs |
| `shell` | Open a shell inside the running container |
| `backup` | Snapshot `n8n-data/` into `backups/` (stops container briefly) |
| `restore` | Restore `n8n-data/` from a backup file (overwrites current data) |
| `remove` | Stop and remove the container (add `--data` to also wipe data) |
| `help` | Show the full reference |

### Options — `start` / `restart` / `update`

| Flag | Effect |
|---|---|
| `-l`, `--lan` | Bind to `0.0.0.0` for LAN access (default: localhost only) |
| `-b`, `--bind <ip>` | Bind to one specific address instead of `127.0.0.1` |
| `--localhost` | Force localhost-only (the default, rarely needed explicitly) |
| `-n`, `--name <name>` | Container name (default: `n8n`) |
| `--no-host-access` | Disable the `host.docker.internal` mapping (default: enabled) |
| `-t`, `--tz <timezone>` | Timezone, e.g. `America/New_York` (default: `America/Puerto_Rico`) |
| `-i`, `--image-tag <tag>` | n8n image tag (default: `latest`, or `N8N_IMAGE_TAG`) |

### Options — `stop` / `status` / `logs` / `shell` / `config`

| Flag | Effect |
|---|---|
| `-n`, `--name <name>` | Target a container name other than the default `n8n` |

### Options — `remove`

| Flag | Effect |
|---|---|
| `-n`, `--name <name>` | Target a container name other than the default `n8n` |
| `--data` | Also delete `n8n-data/` (irreversible — gated behind a `yes` prompt) |

### Options — `restore`

| Flag | Effect |
|---|---|
| `-n`, `--name <name>` | Target a container name other than the default `n8n` |
| `--file <path>` | Backup file to restore (required) |

### Options — any command

| Flag | Effect |
|---|---|
| `-h`, `--help` | Show the full reference and exit |

---

## Quick reference — commands

```bash
./n8nctl start                          # localhost only, all defaults
./n8nctl start --lan                    # LAN access, plain HTTP
./n8nctl start -b 192.168.1.50          # bind to one specific address
./n8nctl start --no-host-access         # no host.docker.internal mapping
./n8nctl start -n n8n-test -t America/New_York

./n8nctl update                         # pull latest, keep current settings
./n8nctl update -i 1.68.0               # pin to a specific version

./n8nctl backup                         # snapshot to backups/
./n8nctl restore --file backups/n8n-data_2026-07-16_143012.tar.gz

./n8nctl remove --data                  # nuke container + volume

./n8nctl config
./n8nctl restart --lan                  # re-apply a bind change
./n8nctl stop
./n8nctl status
./n8nctl logs
./n8nctl shell

NO_COLOR=1 ./n8nctl status              # disable colors, e.g. for logging
```

### Verify host access after start

```bash
docker exec n8n cat /etc/hosts | grep host.docker.internal
```

---

## Connecting n8n to the host machine via SSH

**Why `localhost` doesn't work:** `-p` publishes container → host, for things *outside* Docker to reach *in*. It has no effect on what `localhost` resolves to *inside* the container — that's always the container's own loopback. An SSH node pointed at `localhost` reaches port 22 inside the n8n container itself, not the host.

### Confirm the mapping is present

```bash
docker inspect n8n --format '{{json .HostConfig.ExtraHosts}}'
# expect: ["host.docker.internal:host-gateway"]
```

### Enable SSH on the host (run on the host, not in any container)

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
sudo systemctl status ssh --no-pager
sudo ss -lntp | grep ':22'
```

Test the account directly on the host first: `ssh <your-username>@127.0.0.1`

### n8n SSH credential

| Field | Value |
|---|---|
| Host | `host.docker.internal` |
| Port | `22` |
| Username | your host account |
| Password / Key | your host account's credential |

**Do not use:** `localhost`, `127.0.0.1`, `::1` (all resolve to the container's own loopback), or `kali-pentest` (a different container, only relevant if it exists and runs its own SSH server).

### Raw gateway IP (alternative to `host.docker.internal`)

```bash
docker inspect n8n --format '{{range .NetworkSettings.Networks}}{{.Gateway}}{{end}}'
```

The gateway IP can shift if the Docker network is ever recreated, while `host.docker.internal` doesn't — prefer the hostname for anything beyond a one-off test.

---

## Raw docker command equivalents

Everything `n8nctl` does is a thin wrapper around a fixed set of `docker` invocations — useful for debugging, running without the script, or confirming what's happening under the hood. All paths assume `~/n8n-project/`.

### `start` (localhost-only, host access on)

```bash
mkdir -p ~/n8n-project/n8n-data ~/n8n-project/backups
sudo chown -R 1000:1000 ~/n8n-project/n8n-data

docker run -d --name n8n --restart unless-stopped \
  -p 127.0.0.1:5678:5678 \
  -v ~/n8n-project/n8n-data:/home/node/.n8n \
  -e GENERIC_TIMEZONE="America/Puerto_Rico" \
  -e TZ="America/Puerto_Rico" \
  --add-host=host.docker.internal:host-gateway \
  docker.n8n.io/n8nio/n8n:latest
```

### `start --lan`

```bash
docker run -d --name n8n --restart unless-stopped \
  -p 0.0.0.0:5678:5678 \
  -v ~/n8n-project/n8n-data:/home/node/.n8n \
  -e GENERIC_TIMEZONE="America/Puerto_Rico" \
  -e TZ="America/Puerto_Rico" \
  -e N8N_SECURE_COOKIE=false \
  --add-host=host.docker.internal:host-gateway \
  docker.n8n.io/n8nio/n8n:latest
```

### `start --no-host-access`

Drop the `--add-host` flag from either command above.

### `stop` / `restart`

```bash
docker stop n8n
docker start n8n
```

**Caveat:** this only works cleanly if bind address / host-access don't need to change — Docker won't let you alter `-p` or `--add-host` on an existing container. To change those, remove and recreate:

```bash
docker rm -f n8n
# then re-run the full `docker run ...` command with new options
```

### `update`

```bash
docker pull docker.n8n.io/n8nio/n8n:latest
docker rm -f n8n
# re-run the same `docker run ...` command from start — the volume
# at ~/n8n-project/n8n-data is untouched by `docker rm`
```

### `status`

```bash
docker ps -a --filter "name=^n8n$" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

### `config` (facts `detect_current_config` gathers)

```bash
docker inspect n8n --format \
  '{{range $p, $conf := .HostConfig.PortBindings}}{{range $conf}}{{.HostIp}}{{end}}{{end}}'   # bind
docker inspect n8n --format '{{json .HostConfig.ExtraHosts}}'                                  # host-access mapping
docker inspect n8n --format '{{range .Config.Env}}{{println .}}{{end}}' | grep GENERIC_TIMEZONE # timezone
docker inspect n8n --format '{{.Config.Image}}'                                                 # image + tag
```

### `logs` / `shell`

```bash
docker logs -f n8n
docker exec -it n8n sh
```

### `backup`

```bash
docker stop n8n
tar -czf ~/n8n-project/backups/n8n-data_"$(date +%Y-%m-%d_%H%M%S)".tar.gz \
  -C ~/n8n-project n8n-data
docker start n8n
```

### `restore --file <path>`

```bash
docker stop n8n
rm -rf ~/n8n-project/n8n-data/*
tar -xzf ~/n8n-project/backups/n8n-data_2026-07-16_143012.tar.gz \
  -C ~/n8n-project
sudo chown -R 1000:1000 ~/n8n-project/n8n-data
docker start n8n
```

### `remove` / `remove --data`

```bash
docker rm -f n8n
# only if you also want the volume gone (irreversible):
rm -rf ~/n8n-project/n8n-data
```

### Health check (what `wait_for_healthy` polls)

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:5678/healthz
# 200 once n8n has finished booting
```

---

## Design notes

- **Path resolution** reuses the hard-won `command -v` + `readlink -f` logic from `n8nkali` so the script works whether invoked as `./n8nctl`, from a `$PATH` symlink, or as a bare command — minus the `../` since the layout is now flat.
- **`fail()` is the single funnel** for every user mistake: unknown command, unknown flag, missing flag value all print the specific problem and then the full help text before exiting non-zero.
- **Precedence:** flags override environment variables, which override `n8nctl.env`, which overrides script defaults. A bad flag is caught immediately; a typo'd env var or config-file key silently does nothing.
- **`HOST_ACCESS` defaults to `true`**, not opt-in — reaching the host is the point of this script. The `--add-host` mapping only writes a static `/etc/hosts` entry; what's actually reachable depends on what's listening on the host side.
- **`update` reads the running container's own metadata** (`detect_current_config`) instead of asking you to retype flags — bind, host-access, timezone, and image tag are all pulled from `docker inspect` so the recreated container matches the old one, just on the new image.
- **`backup`/`restore` stop the container first.** n8n's default storage is SQLite; tarring or overwriting that file while it's open risks a corrupted snapshot. Both commands restart automatically afterward if the container was running.
- **`remove --data` and `restore` both require a typed `yes`**, not just a flag — a second, independent confirmation that can't be triggered by an accidental copy-paste.
- **`wait_for_healthy` polls `/healthz`** for up to 30s after start, because "the container exists" and "n8n is actually serving requests" are different facts.
- **Colors are conditional on `[ -t 1 ]`** and respect `NO_COLOR` — piping `logs` or `status` into a file or another tool no longer embeds raw ANSI bytes.
- **Bind-address and host-access mismatch both trigger a recreate.** Docker can't alter `-p` or `--add-host` on an existing container — `docker start` just relaunches it as-is — so `cmd_start` checks both the bind address and the `ExtraHosts` mapping against what's currently requested and recreates if either has drifted.

---

## Bug fixed (historical) — help text printed literal escape codes

**Symptom:** colored headers in `cmd_help` printed as literal text (`\033[0;36mn8nctl\033[0m...`) instead of rendering as color.

**Root cause:** color variables were originally defined with plain single quotes, storing the literal `\033...` characters rather than the real escape byte. `echo -e` interpreted them fine, but `cat << EOF` heredocs only perform variable substitution, never escape interpretation.

**Fix:** ANSI-C quoting (`CYAN=$'\033[0;36m'`), storing the real escape byte at definition time so both `echo -e` and heredocs render correctly.

---

## Verified test results

- Help output renders real ANSI escape bytes in an interactive terminal (confirmed via `cat -v`)
- Colors suppressed when stdout isn't a tty (`./n8nctl status > file` contains zero escape bytes)
- Unknown command → specific error + full help, exit code `1`
- Missing flag value (e.g. `start --bind`) caught before ever reaching Docker
- `config` reflects the actual running container's state via `docker inspect`, not just passed flags
- `update` preserves non-default settings (timezone, host-access) across a version bump
- `backup` produces a restorable `tar.gz` containing `n8n-data/database.sqlite` and related files
- `restore` refuses to run without `--file`
- `remove --data` is gated behind a typed `yes` — data survives on any other input
- Host-access mismatch detection tested against three scenarios (no mapping + access requested; mapping present + already correct; mapping present + `--no-host-access` requested) — recreates only when needed
