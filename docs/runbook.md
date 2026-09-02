# Runbook

## Maintenance schedule

| Frequency | Task | How |
|---|---|---|
| Continuous | Service availability | Uptime monitoring alerts on failure |
| Weekly | Patch host and guests | `apt update && apt upgrade` on each |
| Weekly | Review DNS query log | Unusual lookups can indicate compromised devices |
| Weekly | Check UPS battery health | Via the UPS monitoring daemon |
| Monthly | Verify a backup by restoring it | An unverified backup is a hypothesis, not a backup |
| Quarterly | Reclaim disk space | `docker system prune -a` |

## Setup order

Each step depends on the one before it. Snapshot after each.

1. Install the hypervisor
2. Configure networking, remove the subscription nag
3. Connect the UPS, install and configure NUT
4. Deploy DNS filtering as an LXC container
5. Point the router's DHCP-advertised DNS at it
6. Create the Docker host VM, install Docker and Portainer
7. Deploy the reverse proxy
8. Deploy monitoring
9. Install the mesh VPN on the host, enable subnet routing
10. Set up SSH keys, disable password authentication
11. Configure the firewall
12. Schedule backups

Roughly four hours for steps 1–12 first time through.

> **Snapshot before every step.** A snapshot is a save point. If step 7 goes
> wrong you roll back to the end of step 6 in seconds instead of rebuilding. This
> is the single biggest time-saver in lab work.

---

## Failure modes actually encountered

Real problems and how they were diagnosed — the useful part of any runbook.

### DNS records not applying after a major version upgrade

**Symptom:** Local DNS records added through the admin UI silently failed to
resolve.

**Cause:** The major version bump moved custom DNS host storage out of the legacy
list file and into the new structured config, under a dedicated hosts key. Edits
to the old path were being ignored entirely — no error, no warning.

**Fix:** Set records through the current configuration tooling rather than the
legacy file. Verify with a direct query against the resolver, not through a
client that may be caching.

**Lesson:** When a config change appears to do nothing after a major upgrade,
suspect the storage location moved before suspecting your syntax.

### SSH host key changed

**Symptom:** `REMOTE HOST IDENTIFICATION HAS CHANGED` connecting host-to-guest.

**Cause:** A package update bumped the default host key type, so the server
presented a different key algorithm than the one cached.

**Fix:**

```bash
ssh-keygen -f ~/.ssh/known_hosts -R <host>
ssh -o StrictHostKeyChecking=accept-new user@<host>
```

**Lesson:** This warning is designed to look like an attack because sometimes it
is one. Confirm you have an explanation — a recent update, a rebuild — before
clearing the key. Clearing it reflexively is exactly the habit an attacker
relies on.

### Monitoring UI silently dropping saved configuration

**Symptom:** Some monitors saved through the web UI never persisted.

**Cause:** Flaky client-side event handling under automation; the save appeared
to succeed but never committed.

**Fix:** Write directly to the application's SQLite database and restart the
container. Note that HTTPS checks against self-signed certificates need the
"ignore TLS error" flag set, or they alert continuously.

**Lesson:** When a UI is unreliable, the datastore underneath is usually stable
and documented. Knowing where an application keeps its state is worth the ten
minutes it takes to find out.

### Self-assigned address on a USB ethernet adapter

**Symptom:** Wired interface stuck on a link-local `169.254.x.x` address.

**Cause:** Physical — DHCP requests received no reply. Cable fault, not
configuration.

**Fix:** Nothing in software. The OS correctly deprioritised the dead interface
and routed over Wi-Fi.

**Lesson:** A link-local address means the DHCP conversation never happened.
That is a layer 1 or layer 2 problem, and no amount of reconfiguring layer 3
will fix it. Check the cable first.

---

## Recovery

**A service is down.** Check the container is running, then its logs, then
whether its dependencies (DNS, the Docker daemon, the host) are healthy. Work
down the stack, not across it.

**DNS is down.** Everything appears broken simultaneously — that symptom is
itself the diagnosis. Point a client at a public resolver temporarily to confirm,
then fix DNS. Anything that takes out DNS deserves a same-day fix.

**The host will not boot.** Restore the most recent backup to new storage. This
is why backups get verified monthly rather than assumed.
