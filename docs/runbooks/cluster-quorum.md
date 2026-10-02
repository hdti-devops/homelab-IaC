# Cluster Quorum

## Context

Cluster `hdti-homelab` has two nodes with one vote each. Quorum requires **2 votes**. HA is not used, but quorum still governs `/etc/pve`.

When one node is down, the surviving node loses quorum:
* running guests keep running;
* starting, creating, or modifying guests fails, and Terraform runs fail;
* after a power outage, guests with `onboot` do **not** start if only one node comes back.

## Check Status

```bash
pvecm status   # look for "Quorate: Yes" and "Total votes: 2"
```

## Workaround: Single Node Down

Run on the **surviving** node only, and only if the other node is really offline:

```bash
pvecm expected 1
```

The node becomes quorate and guests can be started or modified. Quorum expectations reset automatically once the second node rejoins. Check with `pvecm status`.

> ⚠️ Never run this while both nodes are running but unable to see each other (network split): both sides could write to `/etc/pve` independently.

### After a Power Outage

If only one node boots, start the critical guests in this order once `pvecm expected 1` is applied: `dns-01` / `dns-02`, `caddy`, `vault`, then the others.

## Permanent Fix: QDevice (planned with the NAS)

A QDevice provides a third vote from an external host (IP `192.168.2.205`).

```bash
# On the external host (Debian-based)
apt install corosync-qnetd

# On both Proxmox nodes
apt install corosync-qdevice

# On one Proxmox node (requires root SSH to the external host)
pvecm qdevice setup 192.168.2.205
pvecm status   # expected: Total votes 3, Quorum 2
```
