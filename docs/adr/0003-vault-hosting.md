# [ADR-0003] HashiCorp Vault hosting in an unprivileged LXC with integrated Raft storage

* **Status:** Accepted
* **Date:** 2026-10-02
* **Deciders:** hdti-devops (homelab owner)

## Context
The homelab needs a central secrets store for Terraform, Packer, GitHub Actions, and services (Proxmox tokens, Cloudflare tokens, etc.). Vault will run on `pve2`, which has ~6 GiB of allocatable RAM shared with all infrastructure services. There is no cloud KMS for auto-unseal.

## Decision
* Run Vault in an **unprivileged LXC** on `pve2` (`vault`, `192.168.2.220`, 1 vCPU, 1 GiB limit).
* Use **integrated storage (Raft)**, single node.
* Set `disable_mlock = true` (mlock is unavailable in unprivileged LXCs and recommended off with Raft).
* **Manual Shamir unseal**; unseal keys and root token stored offline.
* Expose the UI/API as `vault.hdti.ca` through Caddy.
* Schedule `vault operator raft snapshot save` backups.
* Bootstrap: secrets live in HCP Terraform sensitive variables until Vault is deployed, then migrate.

## Alternatives Considered

### Option A: Dedicated VM
* **Description:** Vault in a full VM with its own kernel.
* **Verdict:** Rejected
* **Rationale:** ~1 GiB extra RAM on an already tight node for an isolation gain of limited value in a homelab.

### Option B: Unprivileged LXC with Raft
* **Description:** Described in the Decision.
* **Verdict:** Selected
* **Rationale:** ~300 MB footprint, no external storage backend, snapshot-based backups.

### Option C: OpenBao
* **Description:** MPL-licensed, API-compatible fork of Vault.
* **Verdict:** Rejected (for now)
* **Rationale:** Vault's BSL license is not a constraint for homelab use; can be revisited.

## Consequences

### Positive Impacts
* Lightweight, simple to back up and restore.
* Centralized secrets for all automation.

### Negative Impacts & Risks
* Manual unseal after every restart of the container or node.
* Single instance: no high availability.
* Shared kernel with the host (weaker isolation than a VM).

### Trade-offs & Compromises
* Resource efficiency over isolation and availability.

### Next Steps & Action Items
* [ ] Deploy the Vault LXC with Terraform.
* [ ] Migrate Proxmox and Cloudflare tokens from HCP Terraform variables to Vault.
* [ ] Automate Raft snapshots to the future NAS.

## Corrections & Revisions
* 2026-10-02: HCP Terraform workspaces use local execution mode, where workspace variables are not injected. Bootstrap secrets live in environment variables (workstation, then GitHub Secrets) until Vault is deployed. See [ADR-0004](0004-hybrid-provisioning.md).
* 2026-10-02: Provisioning method for the Vault LXC (Terraform or community script) is decided per service, see [ADR-0004](0004-hybrid-provisioning.md).
* 2026-10-04: The UI/API is exposed as `vault.home.hdti.ca` (not `vault.hdti.ca`), following the internal zone of [ADR-0005](0005-internal-dns-subdomain.md).
