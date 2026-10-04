# [ADR-0005] Internal DNS zone home.hdti.ca and naming convention

* **Status:** Accepted
* **Date:** 2026-10-04
* **Deciders:** hdti-devops (homelab owner)

## Context
[ADR-0002](0002-split-horizon-dns.md) selected a Technitium Conditional Forwarder Zone for the whole `hdti.ca` domain, with internal services named `<svc>.hdti.ca`. Before deploying Technitium, the design was reviewed against the requirements:

* Publicly trusted HTTPS certificates for LAN-only services, with no custom CA on client devices.
* The same HTTPS URLs on the LAN and over WireGuard.
* No internal service exposed to the Internet; Cloudflare remains the public DNS.
* Terraform used only where it adds real value; simplicity and reliability first.

Findings of the review:

* Forwarder zone support in the `darkhonor/technitium` provider only became functional in v1.2.1, which also introduced a breaking change on FWD records; v1.3.0 still fixes forwarder issues. Primary zones are the most mature path.
* In a Forwarder zone, any name without a local record is forwarded to Cloudflare, including typos of internal names. A Primary zone answers authoritatively with NXDOMAIN.
* Sharing `hdti.ca` between internal and future public names blurs which names are private.
* Technitium clustering (v14.0+) replicates only zones that are members of the cluster catalog zone; configuration changes are accepted only on the primary node.
* Technitium installed by the community script is v15.6 (2026-10-03).

## Decision
* **Internal zone:** `home.hdti.ca` is a **Primary zone** in Technitium, member of the cluster catalog zone so that `dns-01` and `dns-02` both serve it. It is not delegated publicly: Cloudflare publishes nothing under `home.hdti.ca` except the transient `_acme-challenge` TXT records.
* **Namespace boundary:**
  * `*.home.hdti.ca` is private, reachable only from the LAN or through WireGuard.
  * `*.hdti.ca` is public, managed in Cloudflare; today it holds only `vpn.hdti.ca`.
  * Internal services get no external name: remote access always goes through WireGuard, whose clients use Technitium and therefore the same URLs. A service published later (e.g. through Cloudflare Tunnel) is named under `hdti.ca`; Proxmox, Vault, and DNS are never published.
* **Naming convention:**
  * **Service names** (daily HTTPS access): resolved by the wildcard record `*.home.hdti.ca` → Caddy (`192.168.2.212`), e.g. `proxmox.home.hdti.ca`, `vault.home.hdti.ca`. Adding a service only requires a Caddyfile change.
  * **Host names** (SSH, DNS tests, troubleshooting): explicit A records to the host IP, which take precedence over the wildcard: `pve`, `pve2`, `dns-01`, `dns-02`, `caddy`, `wireguard`.
  * A host name never equals a service name, otherwise it shadows the service.
  * `proxmox.home.hdti.ca` is reverse-proxied by Caddy to `pve` with failover to `pve2` (either node manages the whole cluster).
* **TLS:** Caddy obtains a Let's Encrypt wildcard `*.home.hdti.ca` (plus `home.hdti.ca` only if needed) through the Cloudflare DNS-01 challenge. The `tls` block sets `resolvers 1.1.1.1`, because Caddy's own resolver (Technitium) is authoritative for `home.hdti.ca` and would never see the challenge record created at Cloudflare. The Caddy token is scoped to `hdti.ca` with `Zone:DNS:Edit` and `Zone:Zone:Read`, as required by the `caddy-dns/cloudflare` module.
* **Proxmox node certificates:** native PVE ACME certificates are deferred. Break-glass access when Caddy is down uses `https://<node IP>:8006` with the self-signed certificate.
* **Resolution:** Technitium forwards all other names to Cloudflare (`1.1.1.1`, `1.0.0.1`). The cluster domain is `cluster.home.hdti.ca`; its zone is created by Technitium and never managed by Terraform.
* **Terraform scope:** the `home.hdti.ca` zone and its records only, applied to the primary node `dns-01` by IP. Server settings (forwarders, recursion, blocking, clustering) are applied once by the Technitium runbook and synchronized by the cluster. If the provider cannot set the cluster catalog membership, the zone is created manually as a catalog member and Terraform manages its records only.
* **Resolver bypass guards:** the router hands out only `192.168.2.210` and `192.168.2.211` (no public secondary, no ISP DNS through IPv6 router advertisements or DHCPv6), and Technitium blocks the Firefox canary domain `use-application-dns.net` to disable default DNS-over-HTTPS.

## Alternatives Considered

### Option A: Conditional Forwarder Zone for hdti.ca (ADR-0002)
* **Description:** Local records `<svc>.hdti.ca` in a Forwarder zone, other names forwarded to Cloudflare.
* **Verdict:** Rejected (supersedes the ADR-0002 choice)
* **Rationale:** Works, but relies on the least mature part of the Terraform provider, leaks unknown internal names upstream, and mixes private and public names. Its main benefit (same name answering differently inside and outside) is not needed since no internal service is published.

### Option B: Primary zone for hdti.ca
* **Description:** Technitium authoritative for the whole domain.
* **Verdict:** Rejected
* **Rationale:** Shadows every public record (`www`, `MX`, `TXT`) for LAN and VPN clients unless duplicated internally.

### Option C: Primary zone for the home.hdti.ca subdomain
* **Description:** Described in the Decision.
* **Verdict:** Selected
* **Rationale:** No shadowing, mature provider support, authoritative answers, clear private/public boundary, single wildcard certificate.

### Option D: Separate external names for internal services (e.g. pve.hdti.ca)
* **Description:** A public name for remote access next to the internal one.
* **Verdict:** Rejected
* **Rationale:** Without VPN the name points to nothing reachable; making it reachable would expose the hypervisor. Over WireGuard the internal name already works, so a second name only doubles records and certificates.

### Option E: PVE ACME certificates on the nodes
* **Description:** Each node obtains its own certificate through the PVE ACME Cloudflare plugin.
* **Verdict:** Deferred
* **Rationale:** Needs a Cloudflare token on each hypervisor and publishes host names in Certificate Transparency logs, for a rarely used break-glass path.

## Consequences

### Positive Impacts
* Public records of `hdti.ca` keep resolving from the LAN and VPN without duplication.
* Unknown internal names fail fast and stay inside the LAN.
* New services need only a Caddyfile change; Terraform changes are limited to host records.
* The wildcard certificate keeps service names out of Certificate Transparency logs.

### Negative Impacts & Risks
* Longer names (`<svc>.home.hdti.ca`).
* Unknown names under `home.hdti.ca` resolve to Caddy through the wildcard and fail there instead of at the DNS level.
* Clients that bypass Technitium (manual DNS, Android Private DNS set to a public provider, browser DNS-over-HTTPS) cannot resolve internal names.
* Break-glass access to Proxmox shows a certificate warning.

### Trade-offs & Compromises
* Clarity and provider maturity over shorter names.

### Next Steps & Action Items
* [ ] Technitium runbook: cluster `cluster.home.hdti.ca`, forwarders, canary block, router DHCP switch with IPv6 check.
* [ ] Terraform DNS root: zone `home.hdti.ca`, wildcard record, host records; validate the provider version against Technitium 15.6 and its catalog membership support.
* [ ] Caddy: wildcard `*.home.hdti.ca`, `resolvers 1.1.1.1`, `proxmox.home.hdti.ca` with failover.
* [ ] Cloudflare token for Caddy with `Zone:DNS:Edit` and `Zone:Zone:Read`.

## Corrections & Revisions
* None.
