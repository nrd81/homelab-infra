# Architecture Decisions

This document explains the _why_ behind the Dingwall Homelab design. Not just what's running, but why it's set up this way.

## Cluster Design

### Three Proxmox Nodes

The homelab runs on three physical machines, each chosen for a different strength:

| Node | Hardware | Role |
|------|----------|------|
| pve-ugreen | NAS-style, multiple drive bays | Primary storage, ZFS pools |
| pve-beelink | Mini PC, low power | Lightweight services, always-on |
| pve-lenovo | Higher compute | Heavy workloads |

Each node runs Proxmox VE as the hypervisor. VMs are distributed based on resource needs rather than stacking everything on one box.

### Why Proxmox over bare-metal Docker?

Isolation. If a Docker container misbehaves, it takes down the container. If Docker itself misbehaves on bare metal, it takes down the host. Proxmox gives us a VM boundary — the hypervisor stays stable even if a guest VM has issues. It also enables snapshots before risky changes, which has saved hours of recovery time.

## Storage

### Why ZFS?

Three reasons:

1. **Data integrity** — ZFS checksums every block. Silent data corruption (bit rot) is detected and corrected automatically on redundant pools.
2. **Snapshots** — Before any risky change, `zfs snapshot` captures the state. If something breaks, rollback takes seconds, not hours.
3. **Scrubs** — Scheduled scrubs verify every byte on disk. Problems are caught before they become data loss.

### VirtIO-FS for Host-Guest File Sharing

Media files live on ZFS pools on the Proxmox host. VMs access them via VirtIO-FS mounts, which gives near-native filesystem performance without the overhead of NFS or SMB. This means Jellyfin, the arr stack, and qBittorrent all see the same files without duplicating data across VMs.

## Networking

### Cloudflare Tunnel (Zero-Trust Access)

No ports are open on the firewall. External access to services goes through a Cloudflare Tunnel, which creates an outbound-only connection from the homelab to Cloudflare's edge.

Benefits:
- No port forwarding, no exposed attack surface
- Per-service access policies (email OTP, allowed email lists)
- Free SSL termination and DDoS protection
- Bypass policies for mobile apps that can't handle browser-based auth

### Split DNS with Pi-hole

Internal requests to *.thedingwalls.com resolve to local IPs via Pi-hole DNS overrides. External requests route through Cloudflare. This means services are fast on the local network and secure from outside, using the same domain names in both cases.

### VPN for Download Pipeline

All torrent traffic routes through Gluetun (WireGuard VPN container). The arr stack and qBittorrent can only reach the internet through the VPN tunnel. If the VPN drops, downloads stop — no IP leaks.

## Docker Architecture

### Single Compose File

All 20+ services live in one `docker-compose.yml`. This is a deliberate choice over splitting into multiple compose files:

- One `docker compose up -d` brings everything up
- Service dependencies are explicit (`depends_on`, health checks)
- No orchestration layer to maintain
- Easy to see the full picture in one file

The tradeoff is a large file, but with proper commenting and section organization, it's manageable.

### Secrets Management

The stack uses two approaches depending on the service:

1. **Environment variable references** (`${VAR_NAME}`) — Values live in `.env`, never committed to git. Used for most services.
2. **Docker secrets** — For services that support it (PostgreSQL, pgAdmin), passwords are mounted from host files at `/run/secrets/` on tmpfs. Never written to disk inside the container, never visible in `docker inspect`.

Docker secrets is the stronger approach. Services are migrated to it as time allows.

## Design Principles

1. **Simplicity over cleverness** — One compose file, one VM for Docker, standard tools. No Kubernetes, no Ansible, no Terraform. The homelab should be maintainable at 2am when something breaks.

2. **Snapshot before change** — Every infrastructure change gets a ZFS snapshot or VM snapshot first. Rollback is always one command away.

3. **No open ports** — Cloudflare Tunnel handles all external access. The firewall stays closed.

4. **Secrets never in git** — Environment variables, Docker secrets, and `.gitignore` enforce this. The `.env.example` documents the structure without exposing values.

5. **Document as you go** — Runbooks are written after every non-trivial troubleshooting session. Future-me is the target audience.
