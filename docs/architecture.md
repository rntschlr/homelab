# Architecture

## Hardware

| Component | Role |
|---|---|
| Mini PC (quad-thread mobile CPU, 16 GB RAM, 512 GB SSD, 2.5 GbE) | The server — runs everything |
| Consumer router | Gateway and DHCP server, with DNS pointed at Pi-hole |
| UPS (~600 VA) | Keeps host and router alive through short outages; triggers clean shutdown on long ones |
| Laptop / phone | Management and remote access over the mesh VPN |

One mini PC comfortably runs 8–12 lightweight services concurrently. The
constraint that shapes every decision below is **16 GB of RAM**, not CPU.

## Layer 1 — Hypervisor

Proxmox VE on bare metal. The alternative was plain Debian with Docker on top,
which would have worked, but gives up three things worth having:

- **Snapshots.** A restore point before every risky change. This single feature
  changed how fast I was willing to experiment.
- **Mixed workloads.** LXC containers for lightweight services, full KVM virtual
  machines where kernel isolation matters — on the same box.
- **Transferable concepts.** Hypervisors are how enterprise compute works.

## Layer 2 — Guests

**LXC containers** share the host kernel. Cheap: a DNS server runs comfortably in
512 MB. Used for single-purpose, low-risk services.

**KVM virtual machines** are fully isolated with their own kernel. More expensive
and worth it when a workload should not be able to affect the host — the Docker
host is a VM for exactly this reason.

### Resource allocation

Budgeting matters at 16 GB. Roughly:

| Workload | Type | RAM |
|---|---|---|
| Hypervisor host | — | ~2 GB |
| DNS filtering | LXC | 512 MB |
| Docker host (all containers) | VM | ~4 GB |
| Mesh VPN | Host-level | negligible |
| **Always-on subtotal** | | **~7 GB** |
| On-demand lab VMs | VM | 2–4 GB each |

Always-on sits near 7 GB, leaving headroom to spin up lab VMs without contention.
On-demand workloads stay powered off unless in use.

## Layer 3 — Services

All application services run as Docker containers on the Docker host VM, each
with its own Compose definition. Containers are reached either by port on the
local network or by name through the reverse proxy.

Grouping:

- **Network** — DNS filtering
- **Vault & documents** — password manager, document management
- **Monitoring** — uptime checks, status dashboard
- **Infrastructure** — hypervisor UI, container management, reverse proxy

## Layer 4 — Access

[Tailscale](https://tailscale.com) creates a WireGuard mesh between the
hypervisor, the Docker VM, laptop, and phone. Two things make it more than a
convenience:

**Subnet routing.** The hypervisor advertises the local network (`10.0.0.0/24`)
to the mesh and the route is approved in the admin console, with IP forwarding
enabled on the host. Every device on the mesh can then reach local addresses
directly — *the same URLs work at home and remotely, with no reconfiguration.*

**Mesh-only TLS.** For services holding secrets, `tailscale serve` terminates
HTTPS with an automatically renewed certificate, reachable only from within the
mesh.

Nothing is exposed to the public internet. There are no port forwards on the
router.

### Name resolution

Pi-hole holds internal DNS records mapping readable names to local addresses
(the `service.home` convention), so services are reachable by name rather than by
address and port. The router hands out Pi-hole as the DNS server over DHCP, with
**no secondary DNS configured** — a populated secondary lets clients silently
round-robin past your filtering, which defeats the point.

## Data flow: a DNS query

1. A device asks for `example.com`.
2. Pi-hole checks its blocklists. If the domain is an ad or tracker, it answers
   with a null address and stops.
3. Otherwise it forwards upstream to the configured resolver.
4. The answer is cached, so the next identical query is served locally.

This is also why DNS is the highest-blast-radius service in the lab: when it is
down, everything looks broken. It gets the most monitoring attention for that
reason.
