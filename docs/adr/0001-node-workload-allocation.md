# [ADR-0001] Node workload allocation between pve and pve2

* **Status:** Accepted
* **Date:** 2026-10-02
* **Deciders:** hdti-devops (homelab owner)

## Context
The homelab has two Proxmox VE nodes in cluster `hdti-homelab`:
* `pve`: AMD Ryzen 7 5800H (8C/16T), 28.31 GiB RAM, ~794 GiB `local-lvm` thin.
* `pve2`: Intel Core i7-6700T (4C/8T, Intel QuickSync HD 530), 7.51 GiB RAM, ~795 GiB `local-lvm` thin.

Planned workloads: infrastructure LXCs (Technitium, Caddy, WireGuard, Cloudflare DDNS, Vault, Pulse, LibreSpeed, GitHub runner), a Kubernetes cluster (1 control plane, 2 workers), Packer golden template builds, and later a media/automation suite (Jellyfin, *arr). RAM is the binding constraint; storage is sufficient on both nodes.

## Decision
* **`pve` = compute node:** Kubernetes VMs (`k8s-cp-01` 2 vCPU/4 GiB, `k8s-wk-01/02` 4 vCPU/5 GiB each), `gh-runner-01` (3 GiB), secondary DNS `dns-02`, Packer build headroom (~4 GiB transient), *arr suite later.
* **`pve2` = lightweight infrastructure node:** `dns-01`, `caddy`, `wireguard`, `cloudflare-ddns`, `vault`, `pulse`, `librespeed` (~3.7 GiB total), and `jellyfin` later, using QuickSync passthrough for hardware transcoding.
* Golden templates are built per node (VMID `9000-9099` on `pve`, `9100-9199` on `pve2`) because local storage templates can only be cloned on their own node.

## Alternatives Considered

### Option A: Infrastructure LXCs on pve, Kubernetes on pve2
* **Description:** Initial plan.
* **Verdict:** Rejected
* **Rationale:** Three Kubernetes nodes (≥ 2 GiB each) would consume all of `pve2`'s ~6 GiB allocatable RAM, leaving ~1.2 GiB per worker for pods and nothing for Packer or media, while `pve` would stay ~85% idle.

### Option B: Compute on pve, infrastructure on pve2
* **Description:** Inverted allocation described in the Decision.
* **Verdict:** Selected
* **Rationale:** Matches workload weight to node capacity, keeps ~3 GiB margin on `pve`, and puts Jellyfin on the node with the best transcoding iGPU.

## Consequences

### Positive Impacts
* Kubernetes and CI get sufficient CPU and RAM.
* Infrastructure services are isolated from compute-heavy workloads.
* Jellyfin benefits from Intel QuickSync.

### Negative Impacts & Risks
* `pve2` runs close to its RAM limit (~5.7 / ~6 GiB of configured limits once Jellyfin is added).
* Most critical services (DNS, proxy, Vault) depend on `pve2`; `dns-02` on `pve` mitigates DNS only.
* Two-node cluster quorum: losing one node blocks guest management (see `docs/runbooks/cluster-quorum.md`).

### Trade-offs & Compromises
* Simplicity over redundancy: no HA, single instances except DNS.

### Next Steps & Action Items
* [ ] Re-evaluate KSM, VM ballooning, and lightweight Kubernetes distributions (k3s, Talos) before deploying Kubernetes.
* [ ] Add a QDevice and backup target when the NAS is available.

## Corrections & Revisions
* None.
