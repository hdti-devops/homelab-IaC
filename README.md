# HDTI HomeLab IaC

Infrastructure as Code for the `hdti` Proxmox VE homelab: Terraform (`bpg/proxmox`, `darkhonor/technitium`) with HCP Terraform remote state, Packer golden templates, and GitHub Actions on self-hosted runners. Core network LXCs are provisioned with community scripts, their configuration managed as code ([ADR-0004](docs/adr/0004-hybrid-provisioning.md)).

> 🇫🇷 French version: [fr/README_FR.md](fr/README_FR.md)

## Platform

| Node | CPU | RAM | Storage (`local` / `local-lvm`) | IP | Role |
|---|---|---|---|---|---|
| `pve` | AMD Ryzen 7 5800H (8C/16T) | 28 GiB | ~94 GiB / ~794 GiB thin | `192.168.2.200` | Compute: Kubernetes VMs, CI runner, Packer builds, *arr suite |
| `pve2` | Intel Core i7-6700T (4C/8T, QuickSync) | 7.5 GiB | ~94 GiB / ~795 GiB thin | `192.168.2.201` | Lightweight infrastructure LXCs, Jellyfin (hardware transcoding) |

* Cluster `hdti-homelab` (2 nodes, no HA, no QDevice yet). See [cluster quorum runbook](docs/runbooks/cluster-quorum.md).
* State: HCP Terraform, Org `hdti`, Project `homelab`, workspaces in **local execution mode** (HCP cannot reach the LAN).
* Rationale: [ADR-0001 — Node workload allocation](docs/adr/0001-node-workload-allocation.md).

### Workload Placement

| Node | Workload | Type | vCPU | RAM (limit) |
|---|---|---|---|---|
| `pve` | `k8s-cp-01` | VM | 2 | 4 GiB |
| `pve` | `k8s-wk-01`, `k8s-wk-02` | VM | 4 each | 5 GiB each |
| `pve` | `gh-runner-01` | LXC | 4 | 3 GiB |
| `pve` | `dns-02` (secondary) | LXC | 1 | 512 MiB |
| `pve` | *arr suite (later) | LXC | 2 | 2 GiB |
| `pve` | Packer build headroom | VM (transient) | 4 | 4 GiB |
| `pve2` | `dns-01`, `caddy`, `wireguard`, `cloudflare-ddns`, `vault`, `pulse`, `librespeed` | LXC | 1 each | ~3.7 GiB total |
| `pve2` | `jellyfin` (later) | LXC | 2 | 2 GiB |

Golden templates are built per node (no shared storage): VMID `9000-9099` on `pve`, `9100-9199` on `pve2`. ISOs live on each node's `local` storage.

## Network

* LAN `192.168.2.0/24`, gateway `192.168.2.1`, upstream DNS `1.1.1.1`.
* Static pool `192.168.2.200-254` (the router DHCP pool must not overlap it).
* WireGuard tunnel `10.8.0.0/24`, NATed to the LAN.

### IP Address Plan

| Range | Purpose | Assignments |
|---|---|---|
| `.200-.209` | Hypervisors & management | `.200` pve · `.201` pve2 · `.202-.204` future nodes · `.205` QDevice/NAS (future) |
| `.210-.219` | Core network | `.210` dns-01 · `.211` dns-02 · `.212` caddy · `.213` wireguard · `.214` cloudflare-ddns |
| `.220-.229` | Platform & operations | `.220` vault · `.221` pulse · `.222` gh-runner-01 · `.223` librespeed · `.224` packer-build (transient) |
| `.230-.239` | Kubernetes nodes | `.230` k8s-cp-01 · `.231` k8s-wk-01 · `.232` k8s-wk-02 · `.239` control-plane VIP |
| `.240-.249` | Media & automation | `.240` jellyfin · `.241` arr · `.242` download client |
| `.250-.254` | Kubernetes LoadBalancer pool (MetalLB) | |

Unlisted addresses in each range are reserved for growth.

### Split-Horizon DNS

Technitium is authoritative for the private zone **`home.hdti.ca`**: service names (`<svc>.home.hdti.ca`) resolve to Caddy through a wildcard record, host names (`pve`, `pve2`, `dns-01`, ...) to their own IP, and every other name is forwarded to Cloudflare. Caddy serves a Let's Encrypt wildcard `*.home.hdti.ca` obtained through the Cloudflare DNS-01 challenge. `hdti.ca` stays public in Cloudflare and only publishes `vpn.hdti.ca`; WireGuard (UDP 51820) is the only service exposed to the Internet. The same URLs work on the LAN and over the VPN. See [ADR-0002](docs/adr/0002-split-horizon-dns.md) and [ADR-0005](docs/adr/0005-internal-dns-subdomain.md).

```
INTERNAL (LAN or VPN)
client ──► Technitium cluster (dns-01 .210 primary / dns-02 .211)
            ├─ <svc>.home.hdti.ca   → wildcard → Caddy (.212) ──TLS *.home.hdti.ca──► backend
            ├─ <host>.home.hdti.ca  → host record → host IP
            └─ any other name       → forwarded to Cloudflare 1.1.1.1 / 1.0.0.1

EXTERNAL (Internet)
client ──► Cloudflare public DNS
            ├─ vpn.hdti.ca          → WAN IP (kept current by DDNS .214, DNS-only)
            │                          └─► router UDP 51820 ──► WireGuard (.213) ──► INTERNAL flow
            └─ *.home.hdti.ca       → not published
```

## Security

* Proxmox automation uses dedicated, least-privilege API tokens with privilege separation (`terraform@pve!iac`, `packer@pve!build`, `pulse@pve!monitor`), never `root@pam`. See [Proxmox API tokens runbook](docs/runbooks/proxmox-api-tokens.md).
* Cloudflare tokens are scoped to `hdti.ca`, one per consumer (Caddy, DDNS), with `Zone:DNS:Edit`; the Caddy token also has `Zone:Zone:Read`.
* Secrets are bootstrapped as environment variables (workstation, then GitHub Secrets on the runner), since HCP Terraform local execution does not inject workspace variables, then migrated to HashiCorp Vault ([ADR-0003](docs/adr/0003-vault-hosting.md), [ADR-0004](docs/adr/0004-hybrid-provisioning.md)).
* Pre-commit scanning with `gitleaks`, `tflint`, and `trivy`.

## Roadmap

- [x] Proxmox VE post-install on `pve` and `pve2`, cluster `hdti-homelab`
- [x] Proxmox API roles, users, and tokens
- [x] Hybrid provisioning decision ([ADR-0004](docs/adr/0004-hybrid-provisioning.md))
- [x] Internal DNS zone `home.hdti.ca` and naming convention ([ADR-0005](docs/adr/0005-internal-dns-subdomain.md))
- [ ] Technitium DNS cluster (`dns-01`, `dns-02`) via community scripts, then router DHCP switch
- [ ] Terraform DNS root (`terraform/environments/prod/dns/`, workspace `homelab-prod-dns`): zone `home.hdti.ca` and records
- [ ] Caddy via community script, versioned Caddyfile, wildcard `*.home.hdti.ca`
- [ ] GitHub self-hosted runner
- [ ] HashiCorp Vault and secrets migration
- [ ] Pulse monitoring
- [ ] Cloudflare DDNS, then WireGuard
- [ ] LibreSpeed Rust
- [ ] Packer golden templates
- [ ] Kubernetes cluster
- [ ] Media & automation suite (Jellyfin, *arr)
- [ ] NAS: backups, QDevice, media library

## Documentation

* Architecture decisions: [docs/adr/](docs/adr/)
* Runbooks: [docs/runbooks/](docs/runbooks/)
  * [Workstation tools setup](docs/runbooks/tools-setup.md)
  * [Community scripts registry](docs/runbooks/community-scripts.md)
  * [Proxmox API tokens](docs/runbooks/proxmox-api-tokens.md)
  * [Cluster quorum](docs/runbooks/cluster-quorum.md)
