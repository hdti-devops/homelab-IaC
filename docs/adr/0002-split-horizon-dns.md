# [ADR-0002] Split-horizon DNS with Technitium, Caddy wildcard TLS, and WireGuard

* **Status:** Accepted, partially superseded by [ADR-0005](0005-internal-dns-subdomain.md)
* **Date:** 2026-10-02
* **Deciders:** hdti-devops (homelab owner)

## Context
Internal services need friendly names under the public domain `hdti.ca` (Cloudflare) with valid TLS certificates, without exposing them to the Internet. Remote access must be possible. Public records of `hdti.ca` (e.g. `www`, `MX`) must remain resolvable from the LAN.

## Decision
* **Technitium DNS** (`dns-01` `192.168.2.210` on `pve2`, `dns-02` `192.168.2.211` on `pve`, clustered) is the LAN resolver, distributed by the router DHCP.
* `hdti.ca` is configured as a **Conditional Forwarder Zone**: internal records `<svc>.hdti.ca` resolve locally to Caddy; unmatched names are forwarded to Cloudflare (`1.1.1.1`, DoH).
* **Caddy** (`192.168.2.212`), built with the `caddy-dns/cloudflare` module, obtains a Let's Encrypt wildcard `*.hdti.ca` through the DNS-01 challenge and reverse-proxies all internal services. No inbound port is opened for Caddy.
* **WireGuard** (`192.168.2.213`, tunnel `10.8.0.0/24`) is the only Internet-exposed service (UDP 51820). VPN clients use Technitium as DNS.
* **Cloudflare DDNS** (`192.168.2.214`) keeps `vpn.hdti.ca` (DNS-only, not proxied) pointed at the WAN IP.
* Cloudflare API tokens are scoped to `Zone:DNS:Edit` on `hdti.ca`, one per consumer (Caddy, DDNS).
* Proxmox nodes get their own certificates via the native PVE ACME Cloudflare DNS plugin.

## Alternatives Considered

### Option A: Technitium primary zone for hdti.ca
* **Description:** Technitium is authoritative for the whole domain internally.
* **Verdict:** Rejected
* **Rationale:** Shadows public records; every public record would need to be duplicated internally.

### Option B: Dedicated internal subdomain (lab.hdti.ca)
* **Description:** Internal zone `lab.hdti.ca`, wildcard `*.lab.hdti.ca`.
* **Verdict:** Rejected
* **Rationale:** Valid, but the Conditional Forwarder Zone achieves the same isolation with shorter names and allows true split-horizon (same name, different answer inside and outside) if a service is published later.

### Option C: Conditional Forwarder Zone for hdti.ca
* **Description:** Described in the Decision.
* **Verdict:** Selected
* **Rationale:** No record shadowing, short names, single wildcard certificate.

## Consequences

### Positive Impacts
* Valid TLS everywhere without exposing services.
* Minimal attack surface: a single UDP port.
* Identical experience on the LAN and over the VPN.

### Negative Impacts & Risks
* DNS becomes a critical dependency for the whole LAN; mitigated by two Technitium instances on different nodes.
* Caddy requires a custom build (xcaddy) instead of the stock package.
* `1.1.1.1` must not be handed to clients as a secondary resolver, or split-horizon resolution becomes inconsistent.

### Trade-offs & Compromises
* Single Caddy instance: simpler, but a single point of failure for HTTPS access.

### Next Steps & Action Items
* [ ] Confirm the router DHCP pool does not overlap `192.168.2.200-254`.
* [ ] Create Cloudflare API tokens (Caddy, DDNS).
* [ ] Deploy Technitium, then switch router DHCP DNS to `.210` / `.211`.

## Corrections & Revisions
* 2026-10-02: `dns-01`, `dns-02`, `caddy`, and `wireguard` are provisioned with community scripts; the `hdti.ca` Forwarder zone, records, and Technitium settings are managed by Terraform (`darkhonor/technitium`); the Caddyfile is versioned in `config/caddy/Caddyfile`. See [ADR-0004](0004-hybrid-provisioning.md).
* 2026-10-02: The community script xCaddy addon builds without extra modules: build with `xcaddy build --with github.com/caddy-dns/cloudflare`, replace the stock binary, and hold the package (`apt-mark hold caddy`) so upgrades do not restore it.
* 2026-10-04: Superseded in part by [ADR-0005](0005-internal-dns-subdomain.md). The internal zone is the Primary zone `home.hdti.ca` (not a Forwarder zone for `hdti.ca`), services are named `<svc>.home.hdti.ca`, and Caddy's wildcard is `*.home.hdti.ca` with `resolvers 1.1.1.1` for the DNS-01 propagation check. The Caddy Cloudflare token also needs `Zone:Zone:Read`. PVE ACME certificates on the nodes are deferred. Technitium, Caddy, WireGuard, DDNS, and the single exposed UDP port are unchanged.
