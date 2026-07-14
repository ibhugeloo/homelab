# 🏠 Homelab

Infrastructure-as-documentation for my self-hosted lab — a two-node Proxmox setup
running my dev/staging environments, internal tooling, monitoring, a 3-node
Kubernetes lab, a self-hosted LLM agent, and personal media services.

> **Design philosophy:** public production for clients lives on managed cloud
> (Vercel, Supabase Cloud, Cloudflare). Everything else — dev, staging, internal
> tools, and personal apps — runs on the homelab.

This repo is a **sanitized** snapshot of the architecture. Internal addressing,
host identifiers, and domains are intentionally omitted.

---

## 🧱 Compute

Two standalone Proxmox nodes (no cluster — independent hosts, backed up by a
shared Proxmox Backup Server). Both are low-power 8th-gen Intel mini-PCs (35 W
TDP each), so the whole lab stays quiet and cheap to run 24/7.

| Node | Role | CPU | RAM | Storage |
|------|------|-----|-----|---------|
| `pve-infra` | Networking, infra services, monitoring, backup, media & personal apps | Intel Core i5-8500T — 6C / 6T · 2.1 → 3.5 GHz · 35 W | 32 GB DDR4-2933 (2 × 16) | 1 TB NVMe (Crucial P310) + 500 GB 2.5″ SSHD |
| `pve-apps`  | Local production, dev / staging, project databases | Intel Core i3-8100T — 4C / 4T · 3.1 GHz · 35 W | 32 GB DDR4-2667 (2 × 16) | 500 GB NVMe (Samsung 980) + 500 GB 2.5″ HDD |

On each host the NVMe carries the VM / LXC volumes (thin-provisioned LVM) and the
2.5″ spinner holds backups and bulk media. Each host runs a single VLAN-aware
bridge; VMs and LXC containers are split across functional VLANs (one `/24` per
VLAN).

---

## 🌐 Network segmentation

Routing and firewalling are handled by **OPNsense**. Traffic between VLANs is
deny-by-default; only the flows below are allowed.

```mermaid
flowchart TB
    NET([🌍 Internet / WAN])
    CF[☁️ Cloudflare Tunnel<br/>inbound HTTPS · no open ports]
    TS[🔗 Tailscale<br/>remote admin · subnet-router]
    FW{{🛡️ OPNsense<br/>router · firewall<br/>deny-by-default between VLANs}}

    NET --> CF --> FW
    NET <--> TS --> FW

    FW <==>|admin plane · reaches all| MGMT
    FW <==>|ingress + observability| INFRA
    FW -->|tunnel ingress in · WAN out only| PROD
    FW -->|tunnel ingress in · WAN out only| DEV
    FW -->|WAN out only| MEDIA

    PROD ==>|egress| NET
    DEV ==>|egress| NET
    MEDIA ==>|egress| NET

    subgraph MGMT [🟣 Management VLAN]
        direction LR
        N1[pve-infra node]
        N2[pve-apps node]
        PBS[Proxmox Backup Server]
    end
    subgraph INFRA [🔵 Infra VLAN]
        direction LR
        DNS[AdGuard DNS]
        RP[Caddy reverse proxy]
        MON[Prometheus · Grafana]
        SEM[Semaphore · Ansible]
        AI[self-hosted LLM agent]
    end
    subgraph PROD [🟢 Production VLAN]
        direction LR
        PA[local prod apps]
        PDB[(per-project Supabase VMs)]
    end
    subgraph DEV [🟡 Dev / Staging VLAN]
        direction LR
        CO[Coolify worker]
        K3S[k3s HA ×3 · ArgoCD]
        DDB[(dev databases)]
    end
    subgraph MEDIA [🟠 Media & Personal VLAN]
        DK[Docker VM<br/>Jellyfin · Immich · Navidrome …]
    end

    classDef mgmt  fill:#f3e8ff,stroke:#9333ea,color:#3b0764;
    classDef infra fill:#dbeafe,stroke:#2563eb,color:#172554;
    classDef prod  fill:#dcfce7,stroke:#16a34a,color:#052e16;
    classDef dev   fill:#fef9c3,stroke:#ca8a04,color:#422006;
    classDef media fill:#ffedd5,stroke:#ea580c,color:#431407;
    classDef edge  fill:#f1f5f9,stroke:#475569,color:#0f172a;
    class MGMT,N1,N2,PBS mgmt;
    class INFRA,DNS,RP,MON,SEM,AI infra;
    class PROD,PA,PDB prod;
    class DEV,CO,K3S,DDB dev;
    class MEDIA,DK media;
    class NET,CF,TS,FW edge;
```

> **Two ways in, gated by one firewall.** Public services are reached only through
> an outbound **Cloudflare Tunnel** (no inbound ports), admin access only through
> **Tailscale**. Everything inter-VLAN crosses OPNsense, which is deny-by-default —
> the table below is the full allow-list.

| VLAN | Purpose | Egress policy |
|------|---------|---------------|
| 🟣 Management | Proxmox hosts, backup server, OPNsense | Reaches everything (admin plane, VPN-only) |
| 🔵 Infra | Tunnel ingress, DNS, reverse proxy, monitoring, automation | Routes to Prod / Dev / Media |
| 🟢 Production | Local prod apps + project databases | **WAN only** — isolated from other VLANs |
| 🟡 Dev / Staging | Ephemeral app + database environments | **WAN only** — zero access to Prod |
| 🟠 Media & Personal | Single Docker host (see below) | **WAN only** — a compromised torrent client can't reach business apps |

### Edge & core services

- **OPNsense** — router + firewall, inter-VLAN routing.
- **Cloudflare Tunnel** — HTTPS ingress for exposed services (no public ports open).
- **Caddy** — internal reverse proxy.
- **AdGuard Home** — local DNS with ad/tracker blocking.
- **Tailscale** — zero-trust remote access via a subnet-router (advertises the
  internal VLAN ranges, not the home LAN).

### Observability

- **Prometheus + Grafana** — one central metrics stack for the whole lab;
  node_exporter on every host feeds dashboards and the Glance widgets (live WAN
  throughput, backup freshness…).
- **Uptime Kuma** — HTTP checks on every service, **Telegram alert on any
  `DOWN`**.
- **Glance** — at-a-glance dashboard: fleet status, live metrics, topology map.
- The k8s cluster ships its own metrics into the same central Prometheus
  (`kube-state-metrics` + Grafana Alloy `remote_write`) — one Grafana, lab-wide.

> Loki was tried, then **decommissioned**: in a solo lab the logs were never
> read, so it was pure RAM/disk cost. Logs are consulted on demand instead
> (`kubectl logs`, `docker logs`). Observability you don't look at is waste.

---

## ⚙️ Provisioning & automation

Three layers, each doing one job:

| Layer | Tool | Job |
|-------|------|-----|
| Infrastructure | **OpenTofu** (Proxmox provider) | VMs/LXCs as code — adopted **brownfield by `import`**, state on a dedicated PostgreSQL backend |
| Post-provisioning | **Ansible + Semaphore** | Idempotent config of the whole fleet (~17 targets) from a web UI; playbooks versioned on GitHub, pulled through a **read-only deploy key** |
| App deployment | **Coolify** / **ArgoCD** | PaaS for Docker apps · GitOps for the k8s lab |

Coolify runs split: the **orchestrator VM runs zero apps** — apps live on
dedicated worker VMs (one for local prod, one for dev/staging). An app that
saturates a worker can't take down the control plane. Project databases follow
the same isolation rule: **one project = one dedicated self-hosted Supabase VM**,
exposed only through the Cloudflare Tunnel. An **n8n** instance covers
miscellaneous workflow automation.

## ☸️ Kubernetes lab — GitOps

A **3-node HA k3s cluster** (all nodes `control-plane,etcd`, MetalLB in L2 mode)
lives in the Dev VLAN, built for two things: learning the industry-standard
stack on real hardware, and rehearsing the migration path off Coolify.

- Provisioned end-to-end by **one Semaphore button** (cloud-init template →
  Ansible playbook) — the cluster is **disposable and rebuildable**.
- Apps are reconciled by **ArgoCD** from a public repo:
  **[`homelab-k8s`](https://github.com/ibhugeloo/homelab-k8s)** — `git push`
  is the only deploy mechanism. Manifests, ops notes and the monitoring
  federation design live there.
- Total footprint: **~5 GB real RAM for all three nodes** (memory ballooning +
  KSM on the host).

## 🤖 Self-hosted AI agent

An **LLM agent runs 24/7 in its own LXC** (open-weights model, self-hosted
runtime), reachable over Telegram. It has a **read-only clone** of a
git-versioned knowledge base — it can read everything, it can write nothing.
It's one member of a multi-agent personal system whose engine, memory
architecture and eval harness are documented in
**[`manin-porunga`](https://github.com/ibhugeloo/manin-porunga)**.

---

## 🎬 Media & personal stack

All media and personal apps run as Docker containers on a **single VM** in the
Media VLAN (Debian 12 + Docker). No per-service LXCs, no `*arr` automation —
acquisition is manual, playback is via Jellyfin.

| Service | Image | What it does |
|---------|-------|--------------|
| **Jellyfin** | `jellyfin/jellyfin` | Movies / TV / music streaming |
| **Immich** | `ghcr.io/immich-app/immich-server` | Self-hosted photo & video backup (with ML + Postgres + Redis) |
| **qBittorrent** | `lscr.io/linuxserver/qbittorrent` | Torrent client |
| **Navidrome** | `deluan/navidrome` | Music streaming (Subsonic API) |
| **MeTube** | `ghcr.io/alexta69/metube` | yt-dlp web frontend |
| **Filebrowser** | `filebrowser/filebrowser` | Web file manager |

Compose files for each live in [`docker-compose/`](./docker-compose/).

### Data layout

```
/data/
├── files/                ← Filebrowser
├── photos/               ← Immich
└── media/
    ├── movies/           ← Jellyfin (films)
    ├── tv/               ← Jellyfin (series)
    ├── music/            ← Navidrome + MeTube (audio)
    └── downloads/        ← qBittorrent + MeTube (video)
```

---

## 💾 Backup — 3-2-1

1. **Proxmox Backup Server** — daily snapshots of every VM/LXC across both nodes.
2. **Second copy** — replicated to a separate volume/medium.
3. **Offsite** — `rclone` to Backblaze B2 for critical data (prod apps, databases, photos).

Criticality: photos 🔴 (snapshot + DB dump) · Docker configs 🟡 · re-downloadable media 🟢.

---

## 🚀 Deployment rule

- ☁️ **Managed cloud** (Vercel + Supabase Cloud + Cloudflare): critical databases,
  delivered client projects, public production.
- 🏠 **Homelab**: dev/staging, database-less landing pages, internal tooling
  (n8n, monitoring), personal apps (Jellyfin, Immich…).

---

## 📁 Repo layout

```
.
├── docs/                  → architecture & operations notes
└── docker-compose/        → media-VM service stacks (one folder per service)
    ├── jellyfin/
    ├── immich/
    ├── qbittorrent/
    ├── navidrome/
    ├── metube/
    └── filebrowser/
```

Each service folder ships a `docker-compose.yml` and a `.env.example`. Real
secrets live in a gitignored `.env` and are **never** committed.

---

## ⚖️ License

MIT — see [`LICENSE`](./LICENSE). Configs are provided as reference; always check
each image's official docs before reuse.
