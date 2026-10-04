# [ADR-0004] Hybrid provisioning and Terraform root layout

* **Status:** Accepted
* **Date:** 2026-10-02
* **Deciders:** hdti-devops (homelab owner)

## Context
Core network services (Technitium DNS, Caddy, WireGuard) are simple, single-purpose LXCs. The community-scripts.org helper scripts install them in minutes, while reproducing the same result with the `bpg/proxmox` provider (template download, container definition, application install, bootstrap) requires significant effort for little gain. The configuration that actually changes over time (DNS zones and records, reverse proxy routes) lives *inside* these services, not in the container definition.

Terraform is not suited to managing files inside containers: it would require SSH provisioners, which are not idempotent and invisible to state. Technitium, however, exposes an HTTP API with a maintained Terraform provider.

HCP Terraform workspaces run in local execution mode, so a provider that cannot reach its target fails every plan in its root configuration.

## Decision
* **Provisioning, case by case:** core network LXCs (`dns-01`, `dns-02`, `caddy`, `wireguard`) are created manually with community scripts in *Advanced* mode. Each service has a runbook that records every option, and each execution is logged in the [community scripts registry](../runbooks/community-scripts.md) with the script commit SHA. Other workloads are decided per service; Kubernetes VMs and Packer templates remain Terraform/Packer-managed.
* **Ownership tags:** every guest carries a Proxmox tag `managed-by-script` or `managed-by-terraform`. Script-managed guests are never imported into Terraform. VMIDs are auto-assigned and recorded in the service runbook.
* **Configuration layer:**
  * Technitium zones, records, and settings are managed by Terraform with the `darkhonor/technitium` provider, pinned to an exact version (`= 1.2.1`), STIG validation set to warning mode, and connected by IP address (never by a name it resolves itself). The runbook only covers install and bootstrap (admin account, clustering, `terraform` API user and token).
  * The Caddyfile is versioned in `config/caddy/Caddyfile`, deployed manually at first, then by a GitHub Actions workflow on the self-hosted runner.
  * WireGuard keys and peer configurations are secrets: never committed, protected by `vzdump` backups, then Vault.
* **Terraform root layout:** one directory per environment, split by domain, each with its own HCP Terraform workspace (local execution):
  * `terraform/environments/prod/dns/` → workspace `homelab-prod-dns`;
  * `terraform/environments/prod/proxmox/` → workspace `homelab-prod-proxmox`, created with the first Terraform-managed Proxmox resource.
* **Secrets:** in local execution mode, HCP Terraform workspace variables are not injected. Provider credentials come from environment variables on the machine running Terraform (workstation, then GitHub Secrets on the runner), then from Vault ([ADR-0003](0003-vault-hosting.md)). Credentials are never Terraform variables, so they never reach state or plans.

## Alternatives Considered

### Option A: Everything in Terraform
* **Description:** Create every LXC with `bpg/proxmox` and configure applications through Terraform.
* **Verdict:** Rejected
* **Rationale:** High effort for simple pets; in-container configuration would rely on SSH provisioners.

### Option B: Hybrid (scripts for provisioning, API/files as code for configuration)
* **Description:** Described in the Decision.
* **Verdict:** Selected
* **Rationale:** Fast provisioning, while the configuration that changes over time stays declarative and reviewed in Git.

### Option C: Ansible for in-container configuration
* **Description:** Ansible roles deploying configuration files and service settings.
* **Verdict:** Deferred
* **Rationale:** Current scope (one Caddyfile) does not justify a new tool; can be revisited when configuration grows.

### Option D: `kenske/technitium` provider
* **Description:** Alternative community Technitium provider.
* **Verdict:** Rejected
* **Rationale:** Dormant since late 2025, latest release not published on the registry, limited resource coverage (no cluster or server settings).

### Option E: Single `prod` root configuration
* **Description:** One root and workspace for all providers.
* **Verdict:** Rejected
* **Rationale:** A Technitium outage would block every Proxmox plan, and a single state widens the blast radius of each apply.

## Consequences

### Positive Impacts
* Core network services are online quickly, unblocking the router DHCP switch.
* DNS zones and records are reviewed and versioned; drift is detected by `terraform plan`.
* Independent states limit the blast radius of each apply.

### Negative Impacts & Risks
* Script-managed LXCs are not reproducible from code: recovery relies on runbooks, Git configuration, and `vzdump` backups (local only until the NAS exists).
* Community scripts track their `main` branch: re-running a script may produce a different result.
* `darkhonor/technitium` is a fast-moving community provider with breaking changes between minor versions (v1.3 requires Technitium 15.0+).
* Caddy needs a custom binary with the `caddy-dns/cloudflare` module; an unpinned `apt upgrade` silently restores the stock binary.

### Trade-offs & Compromises
* Speed and simplicity of provisioning over full reproducibility of core network containers.

### Next Steps & Action Items
* [ ] Technitium runbook, install `dns-01`/`dns-02`, router DHCP switch.
* [ ] `terraform/environments/prod/dns/` root and `homelab-prod-dns` workspace.
* [ ] Caddy runbook and `config/caddy/Caddyfile`.
* [ ] GitHub Actions: Terraform PR checks/apply, Caddyfile validation and deployment.

## Corrections & Revisions
* 2026-10-04: Terraform scope narrowed by [ADR-0005](0005-internal-dns-subdomain.md): it manages the `home.hdti.ca` zone and its records only, applied to the primary node `dns-01`. Technitium server settings (forwarders, recursion, blocking, clustering) are applied by the runbook and synchronized by the cluster.
* 2026-10-04: The community script installs the latest Technitium release (v15.6 on 2026-10-04). The `= 1.2.1` provider pin is re-evaluated against it when the DNS root is written; `darkhonor/technitium` v1.3.0 (released 2026-10-04) requires Technitium 15.0+.
