# Community Scripts Registry

Registry of [community-scripts.org](https://community-scripts.github.io/ProxmoxVE/) scripts executed on the homelab. Community scripts are imperative and not reproducible by Terraform: every execution must be recorded here.

## Registry

| Date | Script | Target | Options selected | Result |
|---|---|---|---|---|
| ≤ 2026-10-02 | PVE Post Install | `pve` (`192.168.2.200`) | No-subscription repositories enabled, enterprise repositories disabled, subscription nag removed, system updated | ✅ Success |
| ≤ 2026-10-02 | PVE Post Install | `pve2` (`192.168.2.201`) | No-subscription repositories enabled, enterprise repositories disabled, subscription nag removed, system updated | ✅ Success |

## Script Details

### PVE Post Install
* **Source:** `tools/pve/post-pve-install.sh`
* **Run on:** each Proxmox VE node shell, as `root`.
* **Command:**
  ```bash
  bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/pve/post-pve-install.sh)"
  ```
* **Notes:** Re-run after a major Proxmox VE upgrade to restore the nag removal and repository configuration.

## Rules
* Read a script before running it.
* Prefer Terraform-managed LXCs/VMs; use community scripts only for host-level tooling or one-off bootstrap.
* Record date, target, options, and result for every execution.
