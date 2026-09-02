# Homelab

A self-hosted infrastructure lab I designed, built, and operated end to end — a
single-node [Proxmox VE](https://www.proxmox.com/en/proxmox-virtual-environment/overview)
hypervisor running a mix of LXC containers and KVM virtual machines, fronted by a
mesh VPN and monitored continuously.

This repository is the sanitised write-up: architecture, the reasoning behind each
choice, the hardening checklist, and the operating runbook. Addresses, hostnames,
and credentials are deliberately replaced with placeholders.

**Status:** Built and operated through 2026 on a mini-PC host. The hardware has
relocated with me to Budapest; the lab is being rebuilt once home internet is
installed. Rebuild notes will land here as it happens.

---

## Why this exists

I wanted to understand a stack end to end rather than only the layer I write code
in. Reading about DNS is not the same as being the reason your household cannot
resolve anything at 11pm. Every service below was chosen because it teaches
something that shows up in real infrastructure work — and because I actually
wanted to use it.

## Architecture

```
                        ┌─────────────────────────────┐
   Internet ──── Router ─┤ DNS → Pi-hole (LXC)         │
                        │                              │
                        │  Proxmox VE host             │
                        │  ├── LXC: Pi-hole            │
                        │  └── VM: Docker host         │
                        │       ├── Portainer          │
                        │       ├── Reverse proxy      │
                        │       ├── Password vault     │
                        │       ├── Document mgmt      │
                        │       ├── Uptime monitoring  │
                        │       └── Status dashboard   │
                        └─────────────────────────────┘
                                     │
                          Tailscale mesh (subnet router)
                                     │
                     laptop  ·  phone  ·  remote access
```

Four layers, each doing one job:

| Layer | What it is | What runs there |
|---|---|---|
| Hypervisor | Bare-metal virtualisation | Proxmox VE |
| Guests | Isolated workloads | LXC containers, KVM virtual machines |
| Services | The actual applications | Docker containers behind a reverse proxy |
| Access | Reaching it from anywhere | Tailscale mesh with subnet routing |

Full detail in [`docs/architecture.md`](docs/architecture.md).

## Service inventory

| Service | Runs as | Why it's here |
|---|---|---|
| **Proxmox VE** | Host | Snapshots, rollback, and a management UI. The same hypervisor concepts every enterprise data centre runs on. |
| **Pi-hole** | LXC | Network-wide DNS with blocklist filtering and internal name records. Teaches DNS by making you responsible for it. |
| **Docker + Portainer** | VM | Container runtime plus a management UI. Containers are the deployment unit essentially everywhere now. |
| **Nginx Proxy Manager** | Container | Reverse proxy — clean names instead of port numbers, and a practical introduction to HTTP routing and TLS termination. |
| **Vaultwarden** | Container | Self-hosted password vault. The service that forced me to take backups and TLS seriously, because losing it is unrecoverable. |
| **Paperless-ngx** | Container | Document management with OCR. Genuinely useful, and a good example of a multi-component app with its own database. |
| **Uptime Kuma** | Container | Uptime monitoring with alerting. Monitoring is not optional in any operations role. |
| **Homepage** | Container | Single dashboard reading live container health from the Docker socket alongside host CPU, memory, and disk. |

## Security posture

Summarised here, detailed in [`docs/hardening.md`](docs/hardening.md).

- SSH key authentication only; password login disabled on the host
- Default-deny host firewall; management interfaces restricted to the local network
- Remote access exclusively over the Tailscale mesh — nothing published to the open internet
- TLS for anything holding secrets, using automatically renewed certificates
- Signups disabled on the password vault after first-run account creation
- Secrets kept in a root-owned, `chmod 600` environment file outside version control
- Weekly patching across host and guests

## Operations

The maintenance routine that keeps it dependable — monitoring, backup
verification, patching, and log review — is in
[`docs/runbook.md`](docs/runbook.md), along with the failure modes I actually hit
and how each was diagnosed.

## What this maps to

Not a certificate, but the practical half of one:

| Capability | Where it came from |
|---|---|
| Linux administration | Running Debian hosts and containers day to day |
| Virtualisation | Proxmox: provisioning, snapshots, resource allocation |
| DNS & DHCP | Pi-hole as the network's resolver; router DHCP configuration |
| VPN & routing | Tailscale mesh, subnet routing, IP forwarding |
| Containers | Docker, Compose definitions, Portainer, container networking |
| Reverse proxy & TLS | Nginx Proxy Manager, certificate issuance and renewal |
| Monitoring | Uptime Kuma, health checks, alerting thresholds |
| Security hardening | Key auth, firewall policy, least privilege |
| Disaster recovery | Backup scheduling, restore testing, UPS-triggered shutdown |

Currently studying for CompTIA Security+ and CCNA, both of which this lab
directly supports.

---

## A note on sanitisation

Everything in this repository uses placeholders:

- Networks are written as `10.0.0.0/24`
- Tailnet hosts are written as `<host>.<tailnet>.ts.net`
- Internal names use the illustrative `service.home` convention
- No credentials, tokens, or secret file paths appear anywhere

Publishing a homelab write-up is useful. Publishing your network's actual
topology is not.

## Related

- Portfolio: [johnkrentschler.me](https://johnkrentschler.me)
- Other projects: [github.com/rntschlr](https://github.com/rntschlr)
