# Operations Guide

Sanitized runbooks and day-to-day operations procedures for the homelab.
Addresses and secrets are placeholders — substitute your own.

---

## 1. Media Docker Host Lifecycle

The media stack lives under `/opt/homelab/<service>/` on the consolidated Docker VM.

```bash
ssh root@<media-host>
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

### Stack Management

```bash
# Restart a specific service
cd /opt/homelab/<service> && docker compose restart

# Tail live container logs
docker logs -f <container-name>

# Pull and update all stacks to latest pinned tags
for d in /opt/homelab/*/; do
  (cd "$d" && docker compose pull && docker compose up -d)
done

# Clean dangling images post-update
docker image prune -f
```

### Immich Notes
Immich runs as a multi-container stack (server, machine-learning, PostgreSQL with VectorChord, Redis).
Always start or restart the entire compose folder together:
```bash
cd /opt/homelab/immich && docker compose up -d
```

---

## 2. Infrastructure as Code (OpenTofu)

Infrastructure resources (VMs and LXCs) are managed as code with OpenTofu. State is
stored in a dedicated PostgreSQL backend (`tofu_state`).

```bash
# Initialize with PostgreSQL backend
tofu init

# Dry-run plan
tofu plan

# Target specific resource for safe surgical updates
tofu apply -target=proxmox_virtual_environment_vm.<resource_name>
```

### CI Plan Validation
Pull requests trigger automated `tofu plan` executions through a self-hosted
GitHub Actions runner (`gh-runner`) running within the Infra VLAN, keeping
credentials out of public CI environments.

---

## 3. Fleet Configuration (Ansible & Semaphore)

Configuration management across all hosts, VMs, and LXCs is automated via Ansible
and driven through the Semaphore UI (`https://semaphore.lab`).

- **GitOps Flow:** Edits made to playbooks are pushed to the private repository.
- **Pull Mechanism:** Semaphore pulls updates using a read-only SSH deploy key.
- **Execution:** Tasks are run as idempotent playbooks (base system setup, SSH key injection,
  backup job configurations, Docker updates).

---

## 4. Database Operations & Consistency

### Verifying Guest Agent
All database VMs must have `qemu-guest-agent` active to enable filesystem freeze/thaw
during Proxmox Backup Server snapshots:

```bash
# Check guest agent status on hypervisor
qm guest cmd <vmid> ping
```

### Logical Encrypted Dumps
In addition to image snapshots, databases run scheduled logical dumps encrypted with AES-256-CBC:
```bash
# Verify latest encrypted dump timestamp inside guest
ls -la /var/backups/pgdump/
```

---

## 5. Backup Verification (3-2-1)

### Proxmox Backup Server (PBS)
- Daily snapshots run at `03:00` across all guests.
- Guests on different hypervisors sharing numeric IDs use distinct namespaces (`srv1`, `srv2`)
  to prevent retention policy collisions.

### Offsite Replication (Backblaze B2)
Offsite replication is executed via an `rclone crypt` systemd timer at `04:00` on the PBS node:

```bash
# Check offsite sync timer status
systemctl status pbs-offsite.timer

# Review latest offsite replication logs
journalctl -u pbs-offsite.service -n 50 --no-pager
```

### OPNsense Configuration Backup
An automated export script generates daily encrypted XML configuration archives
and uploads them to offsite B2 storage.

---

## 6. Security Auditing Protocol (Kali Pentest VM)

The security audit machine (`kali-pentest`) possesses a **trunk network interface**
capable of connecting to any VLAN at Layer 2 to evaluate firewall policies.

### Operational Safeguard: Power-Off Rule
To prevent persistent exposure of a trunk interface on the hypervisor:

1. **Power on on-demand:**
   ```bash
   qm start <kali-vmid>
   ```
2. **Execute audit tasks:**
   Run port scans, TLS configuration checks, or segmentation validation from the guest.
3. **Mandatory shutdown:**
   Once testing completes, immediately shut down the VM:
   ```bash
   qm shutdown <kali-vmid>
   ```
   Verify status is `stopped`:
   ```bash
   qm status <kali-vmid>
   ```
