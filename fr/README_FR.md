# HDTI HomeLab IaC

Infrastructure as Code du homelab Proxmox VE `hdti` : Terraform (`bpg/proxmox`) avec state distant HCP Terraform, golden templates Packer et GitHub Actions sur runners self-hosted.

> 🇬🇧 English version: [../README.md](../README.md)

## Plateforme

| Nœud | CPU | RAM | Stockage (`local` / `local-lvm`) | IP | Rôle |
|---|---|---|---|---|---|
| `pve` | AMD Ryzen 7 5800H (8C/16T) | 28 GiB | ~94 GiB / ~794 GiB thin | `192.168.2.200` | Calcul : VM Kubernetes, runner CI, builds Packer, suite *arr |
| `pve2` | Intel Core i7-6700T (4C/8T, QuickSync) | 7,5 GiB | ~94 GiB / ~795 GiB thin | `192.168.2.201` | LXC d'infrastructure légers, Jellyfin (transcodage matériel) |

* Cluster `hdti-homelab` (2 nœuds, pas de HA, pas encore de QDevice). Voir le [runbook quorum](../docs/runbooks/cluster-quorum.md).
* State : HCP Terraform, Org `hdti`, Projet `homelab`, workspaces en **mode d'exécution local** (HCP ne peut pas joindre le LAN).
* Justification : [ADR-0001 — Allocation des charges](../docs/adr/0001-node-workload-allocation.md).

### Placement des charges

| Nœud | Charge | Type | vCPU | RAM (limite) |
|---|---|---|---|---|
| `pve` | `k8s-cp-01` | VM | 2 | 4 GiB |
| `pve` | `k8s-wk-01`, `k8s-wk-02` | VM | 4 chacun | 5 GiB chacun |
| `pve` | `gh-runner-01` | LXC | 4 | 3 GiB |
| `pve` | `dns-02` (secondaire) | LXC | 1 | 512 MiB |
| `pve` | Suite *arr (plus tard) | LXC | 2 | 2 GiB |
| `pve` | Marge pour builds Packer | VM (temporaire) | 4 | 4 GiB |
| `pve2` | `dns-01`, `caddy`, `wireguard`, `cloudflare-ddns`, `vault`, `pulse`, `librespeed` | LXC | 1 chacun | ~3,7 GiB au total |
| `pve2` | `jellyfin` (plus tard) | LXC | 2 | 2 GiB |

Les golden templates sont construits par nœud (pas de stockage partagé) : VMID `9000-9099` sur `pve`, `9100-9199` sur `pve2`. Les ISO sont sur le stockage `local` de chaque nœud.

## Réseau

* LAN `192.168.2.0/24`, passerelle `192.168.2.1`, DNS amont `1.1.1.1`.
* Plage statique `192.168.2.200-254` (le pool DHCP du routeur ne doit pas la chevaucher).
* Tunnel WireGuard `10.8.0.0/24`, NATé vers le LAN.

### Plan d'adressage IP

| Plage | Usage | Attributions |
|---|---|---|
| `.200-.209` | Hyperviseurs & gestion | `.200` pve · `.201` pve2 · `.202-.204` futurs nœuds · `.205` QDevice/NAS (futur) |
| `.210-.219` | Réseau cœur | `.210` dns-01 · `.211` dns-02 · `.212` caddy · `.213` wireguard · `.214` cloudflare-ddns |
| `.220-.229` | Plateforme & ops | `.220` vault · `.221` pulse · `.222` gh-runner-01 · `.223` librespeed · `.224` packer-build (temporaire) |
| `.230-.239` | Nœuds Kubernetes | `.230` k8s-cp-01 · `.231` k8s-wk-01 · `.232` k8s-wk-02 · `.239` VIP du control plane |
| `.240-.249` | Media & automatisation | `.240` jellyfin · `.241` arr · `.242` client de téléchargement |
| `.250-.254` | Pool LoadBalancer Kubernetes (MetalLB) | |

Les adresses non listées de chaque plage sont réservées pour la croissance.

### DNS Split-Horizon

Technitium héberge une **Conditional Forwarder Zone** pour `hdti.ca` : les enregistrements internes (`<svc>.hdti.ca`) sont résolus localement vers Caddy, tout le reste (enregistrements publics, autres domaines) est transmis à Cloudflare. Caddy sert un wildcard Let's Encrypt `*.hdti.ca` obtenu via le challenge DNS-01 Cloudflare. Seul WireGuard (UDP 51820) est exposé sur Internet. Voir [ADR-0002](../docs/adr/0002-split-horizon-dns.md).

```
INTERNE (LAN ou VPN)
client ──► Technitium (.210 / .211)
            ├─ <svc>.hdti.ca      → enregistrement local → Caddy (.212) ──TLS *.hdti.ca──► backend
            └─ tout autre nom     → transmis à Cloudflare 1.1.1.1 (DoH)

EXTERNE (Internet)
client ──► DNS public Cloudflare
            ├─ vpn.hdti.ca        → IP WAN (tenue à jour par DDNS .214, DNS only)
            │                        └─► routeur UDP 51820 ──► WireGuard (.213) ──► flux INTERNE
            └─ services internes  → non publiés
```

## Sécurité

* L'automatisation Proxmox utilise des tokens API dédiés à privilèges minimaux avec séparation des privilèges (`terraform@pve!iac`, `packer@pve!build`, `pulse@pve!monitor`), jamais `root@pam`. Voir le [runbook tokens API Proxmox](../docs/runbooks/proxmox-api-tokens.md).
* Les tokens Cloudflare sont limités à `Zone:DNS:Edit` sur `hdti.ca`, un par consommateur (Caddy, DDNS).
* Les secrets sont d'abord stockés en variables sensibles HCP Terraform, puis migrés vers HashiCorp Vault ([ADR-0003](../docs/adr/0003-vault-hosting.md)).
* Analyse avant commit avec `gitleaks`, `tflint` et `trivy`.

## Feuille de route

- [x] Post-installation Proxmox VE sur `pve` et `pve2`, cluster `hdti-homelab`
- [x] Rôles, utilisateurs et tokens API Proxmox
- [ ] Workspaces HCP Terraform (exécution locale) et squelette `terraform/`
- [ ] Technitium DNS (`dns-01`, `dns-02`), puis bascule du DHCP du routeur
- [ ] Caddy avec certificat wildcard, certificats ACME sur les nœuds PVE
- [ ] Runner GitHub self-hosted
- [ ] HashiCorp Vault et migration des secrets
- [ ] Supervision Pulse
- [ ] Cloudflare DDNS, puis WireGuard
- [ ] LibreSpeed Rust
- [ ] Golden templates Packer
- [ ] Cluster Kubernetes
- [ ] Suite Media & automatisation (Jellyfin, *arr)
- [ ] NAS : sauvegardes, QDevice, médiathèque

## Documentation

* Décisions d'architecture : [docs/adr/](../docs/adr/)
* Runbooks : [docs/runbooks/](../docs/runbooks/)
  * [Outils du poste de travail](docs/runbooks/tools-setup_FR.md)
  * [Registre des community scripts](../docs/runbooks/community-scripts.md)
  * [Tokens API Proxmox](../docs/runbooks/proxmox-api-tokens.md)
  * [Quorum du cluster](../docs/runbooks/cluster-quorum.md)
