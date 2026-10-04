# HDTI HomeLab IaC

Infrastructure as Code du homelab Proxmox VE `hdti` : Terraform (`bpg/proxmox`, `darkhonor/technitium`) avec state distant HCP Terraform, golden templates Packer et GitHub Actions sur runners self-hosted. Les LXC du réseau cœur sont provisionnés par community scripts, leur configuration est gérée en code ([ADR-0004](../docs/adr/0004-hybrid-provisioning.md)).

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

Technitium fait autorité sur la zone privée **`home.hdti.ca`** : les noms de services (`<svc>.home.hdti.ca`) pointent vers Caddy via un enregistrement wildcard, les noms d'hôtes (`pve`, `pve2`, `dns-01`, ...) vers leur propre IP, et tout autre nom est transmis à Cloudflare. Caddy sert un wildcard Let's Encrypt `*.home.hdti.ca` obtenu via le challenge DNS-01 Cloudflare. `hdti.ca` reste public chez Cloudflare et ne publie que `vpn.hdti.ca` ; WireGuard (UDP 51820) est le seul service exposé sur Internet. Les mêmes URL fonctionnent en LAN et via le VPN. Voir [ADR-0002](../docs/adr/0002-split-horizon-dns.md) et [ADR-0005](../docs/adr/0005-internal-dns-subdomain.md).

```
INTERNE (LAN ou VPN)
client ──► cluster Technitium (dns-01 .210 primaire / dns-02 .211)
            ├─ <svc>.home.hdti.ca   → wildcard → Caddy (.212) ──TLS *.home.hdti.ca──► backend
            ├─ <hôte>.home.hdti.ca  → enregistrement d'hôte → IP de l'hôte
            └─ tout autre nom       → transmis à Cloudflare 1.1.1.1 / 1.0.0.1

EXTERNE (Internet)
client ──► DNS public Cloudflare
            ├─ vpn.hdti.ca          → IP WAN (tenue à jour par DDNS .214, DNS only)
            │                          └─► routeur UDP 51820 ──► WireGuard (.213) ──► flux INTERNE
            └─ *.home.hdti.ca       → non publié
```

## Sécurité

* L'automatisation Proxmox utilise des tokens API dédiés à privilèges minimaux avec séparation des privilèges (`terraform@pve!iac`, `packer@pve!build`, `pulse@pve!monitor`), jamais `root@pam`. Voir le [runbook tokens API Proxmox](../docs/runbooks/proxmox-api-tokens.md).
* Les tokens Cloudflare sont limités à `hdti.ca`, un par consommateur (Caddy, DDNS), avec `Zone:DNS:Edit` ; le token de Caddy a aussi `Zone:Zone:Read`.
* Les secrets sont d'abord fournis en variables d'environnement (poste de travail, puis GitHub Secrets sur le runner), car l'exécution locale HCP Terraform n'injecte pas les variables de workspace, puis migrés vers HashiCorp Vault ([ADR-0003](../docs/adr/0003-vault-hosting.md), [ADR-0004](../docs/adr/0004-hybrid-provisioning.md)).
* Analyse avant commit avec `gitleaks`, `tflint` et `trivy`.

## Feuille de route

- [x] Post-installation Proxmox VE sur `pve` et `pve2`, cluster `hdti-homelab`
- [x] Rôles, utilisateurs et tokens API Proxmox
- [x] Décision de provisioning hybride ([ADR-0004](../docs/adr/0004-hybrid-provisioning.md))
- [x] Zone DNS interne `home.hdti.ca` et convention de nommage ([ADR-0005](../docs/adr/0005-internal-dns-subdomain.md))
- [ ] Cluster Technitium DNS (`dns-01`, `dns-02`) via community scripts, puis bascule du DHCP du routeur
- [ ] Root Terraform DNS (`terraform/environments/prod/dns/`, workspace `homelab-prod-dns`) : zone `home.hdti.ca` et enregistrements
- [ ] Caddy via community script, Caddyfile versionné, wildcard `*.home.hdti.ca`
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
