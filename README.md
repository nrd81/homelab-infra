# 🏠 Dingwall Homelab

A self-hosted infrastructure running on a multi-node Proxmox cluster, managed as Infrastructure as Code. Everything behind Cloudflare tunnels with zero-trust access controls.

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                   Cloudflare Tunnel                  │
│              *.thedingwalls.com (Access)             │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│              Proxmox VE Cluster                      │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │pve-ugreen│  │pve-beelink│ │pve-lenovo│          │
│  └────┬─────┘  └─────┬────┘  └────┬─────┘          │
│       │              │             │                 │
│       └──────────────┼─────────────┘                 │
│                      │                               │
│          ┌───────────▼───────────┐                   │
│          │   srv-media-stack     │                   │
│          │   (Docker Host VM)    │                   │
│          │                       │                   │
│          │  ┌─────────────────┐  │                   │
│          │  │ Docker Compose  │  │                   │
│          │  │  20+ services   │  │                   │
│          │  └─────────────────┘  │                   │
│          └───────────────────────┘                   │
└──────────────────────────────────────────────────────┘
```

## Services

| Category | Services | Purpose |
|----------|----------|---------|
| **Media** | Jellyfin, Jellyseerr | Media server & request management |
| **Automation** | Radarr, Sonarr, Lidarr | Media library automation |
| **Downloads** | qBittorrent, Gluetun (VPN), Jackett | Secure download pipeline |
| **Smart Home** | Home Assistant, Ecobee, Hue, Sonos | Home automation & dashboards |
| **Network** | Pi-hole, Cloudflare Tunnel, Tinyproxy | DNS, reverse proxy, ad blocking |
| **Infrastructure** | Proxmox VE, ZFS, VirtIO-FS | Hypervisor, storage, file sharing |

## Tech Stack

- **Hypervisor:** Proxmox VE (multi-node cluster)
- **Storage:** ZFS pools with scheduled scrubs
- **Containers:** Docker Compose (single stack)
- **Networking:** Cloudflare Tunnel + Access (zero-trust), Pi-hole (local DNS), split-DNS
- **File Sharing:** VirtIO-FS between host and VMs
- **Monitoring:** Home Assistant Command Center dashboard

## Repo Structure

```
homelab-infra/
├── README.md
├── .gitignore
├── .env.example              # Template with placeholder values
├── docker-compose.yml        # Full service stack
├── docs/
│   ├── architecture.md       # Detailed architecture decisions
│   ├── network-diagram.md    # Network topology
│   └── runbooks/             # Troubleshooting guides
│       ├── vpn-troubleshoot.md
│       └── container-recovery.md
└── configs/
    └── cloudflare/
        └── config.yml.example
```

## Key Design Decisions

**Why a single compose file?** Simplicity. One `docker-compose up -d` brings everything up. Dependencies between services (VPN → download clients → arr apps) are managed with `depends_on` and health checks.

**Why Cloudflare Tunnel over a traditional reverse proxy?** No open ports on the firewall. Zero-trust access policies per service. Email OTP authentication for sensitive services. No SSL certificate management.

**Why ZFS?** Data integrity with checksumming, snapshots before risky changes, and easy pool expansion as drives are added.

## Getting Started

This repo documents my personal infrastructure. It's not meant to be cloned and run as-is, but the patterns, configs, and documentation may be useful if you're building something similar.

1. Copy `.env.example` to `.env` and fill in your values
2. Review `docker-compose.yml` and adjust paths/ports
3. `docker compose up -d`

## License

This project is shared for educational and portfolio purposes.
