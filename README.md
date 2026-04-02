# quadlet-loki

Quadlet setup for [Grafana Loki](https://grafana.com/oss/loki/) — a log aggregation system (`docker.io/grafana/loki`).

This project was created with the help of Claude Code and https://github.com/mkoester/quadlet-my-guidelines/blob/main/new_quadlet_with_ai_assistance.md.

## Files in this repo

| File | Description |
|---|---|
| `loki.container` | Quadlet unit file |
| `loki.env` | Default environment variables |
| `loki.override.env.template` | Template for local overrides |
| `loki.yaml.template` | Loki configuration template (single-binary, filesystem storage) |
| `loki-backup.service` | Systemd service: rsync data directory to backup location |
| `loki-backup.timer` | Systemd timer: triggers the backup daily |
| `loki-healthcheck.service` | Systemd service: checks `/ready` endpoint, restarts Loki if unhealthy |
| `loki-healthcheck.timer` | Systemd timer: triggers the health check every 30 seconds |

## Setup

```sh
# 1. Create service user (regular user, home in /var/lib)
sudo useradd -m -d /var/lib/loki -s /usr/sbin/nologin loki

REPO_URL=https://github.com/mkoester/quadlet-loki.git
REPO=~loki/quadlet-loki
```

```sh
# 2. Enable linger
sudo loginctl enable-linger loki

# 3. Clone this repo into the service user's home
sudo -u loki git clone $REPO_URL $REPO

# 4. Create quadlet, config, and data directories
sudo -u loki mkdir -p ~loki/.config/containers/systemd
sudo -u loki mkdir -p ~loki/config
sudo -u loki mkdir -p ~loki/data

# 5. Copy the Loki config template into the config directory and adjust if needed
sudo -u loki cp $REPO/loki.yaml.template ~loki/config/local-config.yaml

# 6. Create .override.env from template
sudo -u loki cp $REPO/loki.override.env.template $REPO/loki.override.env

# 7. Symlink quadlet files from the repo
sudo -u loki ln -s $REPO/loki.container ~loki/.config/containers/systemd/loki.container
sudo -u loki ln -s $REPO/loki.env ~loki/.config/containers/systemd/loki.env
sudo -u loki ln -s $REPO/loki.override.env ~loki/.config/containers/systemd/loki.override.env

# 8. Reload and start
sudo -u loki XDG_RUNTIME_DIR=/run/user/$(id -u loki) systemctl --user daemon-reload
sudo -u loki XDG_RUNTIME_DIR=/run/user/$(id -u loki) systemctl --user start loki

# 9. Verify
sudo -u loki XDG_RUNTIME_DIR=/run/user/$(id -u loki) systemctl --user status loki
```

## Configuration

### Environment variables

`loki.env` contains the defaults:

| Variable | Default | Description |
|---|---|---|
| `TZ` | `Europe/Berlin` | Container timezone |

`loki.override.env` (created from template) has no required values; it can override `TZ`.

To apply changes after editing the override file:

```sh
sudo -u loki nano ~loki/quadlet-loki/loki.override.env
sudo -u loki XDG_RUNTIME_DIR=/run/user/$(id -u loki) systemctl --user restart loki
```

### Loki configuration

The file `~loki/config/local-config.yaml` is mounted into the container at `/etc/loki/`. It is copied from `loki.yaml.template` in this repo during setup and is **not** managed by symlink, so you can adjust it per host without affecting the repo.

The default config uses single-binary mode with local filesystem storage — suitable for a single-node setup.

## Reverse proxy (Caddy)

Add a site block to your Caddyfile:

```
loki.example.com {
    reverse_proxy localhost:3100
}
```

After editing the Caddyfile, reload Caddy:

```sh
sudo systemctl reload caddy
```

## Backup

Loki stores all data (chunks and index) in `/loki` inside the container (`~loki/data/` on the host). See the [general backup setup](https://github.com/mkoester/quadlet-my-guidelines#backup) for the one-time server-wide setup.

The backup service uses `rsync` to copy the entire data directory to the backup location.

```sh
# 1. Create backup staging directory (owned by loki, readable by backup-readers group)
sudo mkdir -p /var/backups/loki
sudo chown loki:backup-readers /var/backups/loki
sudo chmod 750 /var/backups/loki

# 2. Symlink the backup service and timer from the repo
sudo -u loki mkdir -p ~loki/.config/systemd/user
sudo -u loki ln -s $REPO/loki-backup.service ~loki/.config/systemd/user/loki-backup.service
sudo -u loki ln -s $REPO/loki-backup.timer ~loki/.config/systemd/user/loki-backup.timer

# 3. Enable and start the timer
sudo -u loki XDG_RUNTIME_DIR=/run/user/$(id -u loki) systemctl --user daemon-reload
sudo -u loki XDG_RUNTIME_DIR=/run/user/$(id -u loki) systemctl --user enable --now loki-backup.timer
```

### On the remote (backup) machine

```sh
rsync -az backupuser@loki-host:/var/backups/loki/ /path/to/local/backup/loki/
```

## Health check

The Loki container image is distroless (no shell or HTTP tools), so Podman's built-in `HealthCmd` cannot run inside the container. Instead, a systemd timer checks the `/ready` endpoint from the host every 90 seconds and restarts Loki if it fails.

```sh
# 1. Symlink the health check service and timer from the repo
sudo -u loki mkdir -p ~loki/.config/systemd/user
sudo -u loki ln -s $REPO/loki-healthcheck.service ~loki/.config/systemd/user/loki-healthcheck.service
sudo -u loki ln -s $REPO/loki-healthcheck.timer ~loki/.config/systemd/user/loki-healthcheck.timer

# 2. Enable and start the timer
sudo -u loki XDG_RUNTIME_DIR=/run/user/$(id -u loki) systemctl --user daemon-reload
sudo -u loki XDG_RUNTIME_DIR=/run/user/$(id -u loki) systemctl --user enable --now loki-healthcheck.timer
```

## Notes

- Port `3100` is bound to `127.0.0.1` only — Caddy handles TLS termination.
- All persistent data is stored at `~loki/data/` on the host.
- The Loki API (used by Promtail, Alloy, or other log shippers) is available at `http://localhost:3100/loki/api/v1/push`.
- `AutoUpdate=registry` is enabled; activate the timer once to get automatic image updates:
  ```sh
  sudo -u loki XDG_RUNTIME_DIR=/run/user/$(id -u loki) systemctl --user enable --now podman-auto-update.timer
  ```
- To prune old images automatically, enable the system-wide prune timer (see [image pruning setup](https://github.com/mkoester/quadlet-my-guidelines#image-pruning)). Replace `30` with the desired retention period in days:
  ```sh
  sudo -u loki XDG_RUNTIME_DIR=/run/user/$(id -u loki) systemctl --user enable --now podman-image-prune@30.timer
  ```
