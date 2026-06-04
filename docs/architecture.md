# Architecture

A sanitized overview of the lab. Internal IPs, VM/container IDs, and domains are
omitted on purpose.

## Principles

- **Two standalone Proxmox nodes**, not a cluster. Simpler blast radius; a single
  Proxmox Backup Server protects both.
- **Cloud for what must never go down**, homelab for everything else. Client
  production and critical databases live on managed cloud; the lab carries dev,
  staging, internal tooling, and personal apps.
- **Deny-by-default networking.** VLANs are isolated; only explicit inter-VLAN
  flows are permitted.
- **Nothing exposed via open ports.** Public reachability goes exclusively through
  a Cloudflare Tunnel; remote admin goes through Tailscale.

## Nodes

| Node | Role |
|------|------|
| `pve-infra` | Networking (OPNsense), tunnel ingress, DNS, reverse proxy, monitoring, backup, automation, media & personal apps, remote-access agent |
| `pve-apps`  | Local production apps, dev/staging environments, per-project databases |

Each host uses one VLAN-aware Linux bridge. Workloads are LXC containers for
lightweight single-purpose services and full VMs where isolation or a dedicated
kernel matters (databases, Docker hosts, monitoring).

## VLAN map

| VLAN | Purpose | Inter-VLAN policy |
|------|---------|-------------------|
| 🟣 Management | Proxmox hosts, backup server, firewall mgmt | Reaches all (VPN/Tailscale only) |
| 🔵 Infra | Cloudflare Tunnel, DNS, reverse proxy, monitoring, automation | May route to Prod / Dev / Media |
| 🟢 Production | Local prod apps + project databases | WAN only — isolated from other VLANs |
| 🟡 Dev / Staging | Coolify worker + dev databases | WAN only — no access to Prod |
| 🟠 Media & Personal | Single Docker VM | WAN only — isolated from business apps |

### Firewall matrix (inter-VLAN)

| Source | → Destination | Rule |
|--------|---------------|------|
| 🟣 Management | ALL | Admin plane reaches everything |
| 🔵 Infra | Prod / Dev / Media | Tunnel ingress routes to app VLANs |
| 🟢 Production | WAN only | Prod is sealed from the rest of the lab |
| 🟡 Dev | WAN only | Dev can never touch Prod |
| 🟠 Media | WAN only | A compromised torrent client stays contained |

## Core services

| Function | Service |
|----------|---------|
| Router / firewall | OPNsense |
| Public ingress | Cloudflare Tunnel |
| Internal reverse proxy | Caddy |
| Local DNS + filtering | AdGuard Home |
| Remote access | Tailscale subnet-router (advertises internal VLAN ranges only) |
| Metrics & logs | Prometheus + Grafana + Loki |
| Uptime | Uptime Kuma |
| Dashboard | Glance |
| PaaS (dev/staging) | Coolify |
| Workflow automation | n8n |

## Databases

Large/important projects get a **dedicated database VM per project** on `pve-apps`,
exposed to its cloud-hosted frontend through the Cloudflare Tunnel. Small projects
and POCs share the Coolify worker. This keeps each project's blast radius bounded
and lifts the managed-cloud project cap.

## Backup — 3-2-1

1. Proxmox Backup Server — daily VM/LXC snapshots, both nodes.
2. Second on-site copy on separate media.
3. Offsite to Backblaze B2 via `rclone` for critical data.

Photo data and project databases are treated as 🔴 critical (snapshot **and**
logical dump); re-downloadable media is 🟢.
