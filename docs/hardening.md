# Hardening

The lab is a security lab, so it should not itself be soft. Ordered roughly by
return on effort.

## 1. SSH key authentication

Password authentication over SSH is brute-forceable. Keys are not, in any
practical sense.

```bash
# Generate a key pair on the workstation
ssh-keygen -t ed25519 -C "homelab"

# Install the public key on the host
ssh-copy-id root@<host>

# Verify key login works BEFORE disabling passwords
ssh root@<host>
```

Then in `/etc/ssh/sshd_config`:

```
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
```

```bash
sshd -t
systemctl reload ssh
```

> **Order matters.** Confirm key authentication works before disabling password
> login, or you will lock yourself out of your own machine. Keep a second session
> open while making this change. On Debian/Proxmox the unit is normally `ssh`;
> confirm the unit name on other distributions. Inspect effective settings with
> `sshd -T`, including relevant `Match` blocks, and verify a fresh login after reload.

## 2. Firewall policy

Default deny, with explicit allows:

| Traffic | Source | Action |
|---|---|---|
| SSH (22) | Local network only | Allow |
| Hypervisor UI (8006) | Local network only | Allow |
| Mesh VPN (UDP 41641) | Any | Allow |
| Everything else | Any | Drop |

Remote administration goes over the mesh VPN, which is why management ports
never need to face the internet.

## 3. No public exposure

There are no port forwards on the router. Nothing in the lab is published to the
open internet. Every remote path in or out runs through the mesh VPN.

This is the single largest reduction in attack surface available to a home
setup, and it costs nothing.

## 4. TLS where it matters

The password vault refuses to operate over plain HTTP for good reason — it
handles credentials. `tailscale serve` terminates HTTPS with an automatically
renewed certificate, reachable only from inside the mesh.

> **Worth knowing:** hostnames issued public TLS certificates appear in the
> public Certificate Transparency log permanently. That is a privacy trade-off,
> not a security flaw, but it is worth making deliberately rather than by
> accident.

## 5. Least privilege on accounts

- Vault signups disabled after creating the first account, so the instance
  cannot accumulate unexpected users
- Default credentials on every deployed service changed at install, not later
- Admin interfaces bound to the local network rather than all interfaces

## 6. Secrets handling

Service secrets live in a single root-owned file, `chmod 600`, on the host —
outside any container mount and outside version control. Nothing sensitive is in
a Compose file that could end up committed.

Keep an offline recovery kit in a physically secure location, separate from
the encrypted vault backup. Include the recovery codes and the information needed
to restore access. A password manager cannot recover a forgotten master password
from its encrypted database. Test recovery without exposing the real secret.

## 7. Power protection

A UPS with a USB data connection to the host, running
[NUT](https://networkupstools.org/) (Network UPS Tools). Short outage: the lab
rides through it. Long outage: NUT triggers a clean shutdown before the battery
dies.

Power loss during a disk write corrupts filesystems. This is the cheapest
insurance in the entire build.

## 8. Patching

Weekly `apt update && apt upgrade` across host and every guest. Review release notes and take verified backups before major upgrades. Validate
service health after patching and keep a tested rollback path.

## Deliberately not done

Honest list of what is missing:

- **No SIEM yet.** Wazuh is planned but was not run continuously — the manager
  and dashboard want ~4 GB, which is most of the spare headroom. Intended as an
  on-demand workload rather than always-on.
- **No network segmentation.** Everything shares one VLAN. Segmenting the lab
  from personal devices is the next meaningful hardening step.
- **No off-site backups.** Backups were local-only pending an external drive.
  A local-only backup does not survive theft, fire, or flood.

## References

- [OpenSSH server configuration](https://man.openbsd.org/sshd_config.5)
