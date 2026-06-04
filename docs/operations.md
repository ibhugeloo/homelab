# Operations

Common day-to-day commands for the media Docker host. Addresses are placeholders
— substitute your own.

## Compose layout

Each service lives in its own folder under `/opt/homelab/<service>/` on the VM,
mirroring this repo's `docker-compose/` tree.

```bash
ssh root@<media-host>
docker ps --format 'table {{.Names}}\t{{.Status}}'
```

## Lifecycle

```bash
# Restart one service
cd /opt/homelab/<service> && docker compose restart

# Tail logs
docker logs -f <container>

# Update every stack to the latest images
for d in /opt/homelab/*/; do (cd "$d" && docker compose pull && docker compose up -d); done

# Prune dangling images after updates
docker image prune -f
```

## First-run notes

- **qBittorrent** generates a temporary admin password on first boot:

  ```bash
  docker logs qbittorrent 2>&1 | grep -i password
  ```

  Set a permanent one in the WebUI, then it stays put.

- **Immich** is a multi-container stack (server + machine-learning + Postgres +
  Redis). Bring the whole folder up together; don't start `immich-server` alone.

## Bootstrapping a new service

1. Create `/opt/homelab/<service>/` with a `docker-compose.yml` and `.env`.
2. Mount data under `/data/...` to keep everything on the media volume.
3. `docker compose up -d`.
4. Add it to the dashboard and (if it needs remote access) the reverse proxy /
   tunnel config.

## Backups

The media VM is snapshotted nightly by Proxmox Backup Server. Photos additionally
get a logical Postgres dump before the snapshot; Docker configs are versioned in
this repo. Bulk re-downloadable media is not backed up offsite.
