# Proxmox API Tokens

Dedicated, least-privilege service accounts for automation. `root@pam` is never used by tooling.

## Accounts

| Token | Role | Consumer |
|---|---|---|
| `terraform@pve!iac` | `TerraformProv` (custom) | Terraform `bpg/proxmox` provider |
| `packer@pve!build` | `PackerProv` (custom) | Packer `proxmox-iso` builder |
| `pulse@pve!monitor` | `PVEAuditor` (built-in) | Pulse monitoring (read-only) |

All tokens use **privilege separation** (`privsep=1`): effective permissions are the intersection of the user ACL and the token ACL, so both receive the role.

## Creation Procedure

Run once as `root` on any cluster node (`/etc/pve` is replicated). Check the version first with `pveversion`.

> Privileges below target Proxmox VE 9. On Proxmox VE 8, replace `VM.GuestAgent.Audit` with `VM.Monitor`.

```bash
# 1. Roles
pveum role add TerraformProv -privs "Datastore.Allocate Datastore.AllocateSpace Datastore.AllocateTemplate Datastore.Audit Pool.Allocate Pool.Audit SDN.Use Sys.Audit Sys.Console Sys.Modify VM.Allocate VM.Audit VM.Clone VM.Config.CDROM VM.Config.Cloudinit VM.Config.CPU VM.Config.Disk VM.Config.HWType VM.Config.Memory VM.Config.Network VM.Config.Options VM.Console VM.GuestAgent.Audit VM.Migrate VM.PowerMgmt VM.Snapshot VM.Snapshot.Rollback"
pveum role add PackerProv -privs "Datastore.AllocateSpace Datastore.AllocateTemplate Datastore.Audit SDN.Use Sys.Audit VM.Allocate VM.Audit VM.Clone VM.Config.CDROM VM.Config.Cloudinit VM.Config.CPU VM.Config.Disk VM.Config.HWType VM.Config.Memory VM.Config.Network VM.Config.Options VM.Console VM.GuestAgent.Audit VM.PowerMgmt"

# 2. Users (pve realm, no password => no interactive login)
pveum user add terraform@pve --comment "Terraform IaC automation"
pveum user add packer@pve    --comment "Packer golden image builds"
pveum user add pulse@pve     --comment "Pulse monitoring (read-only)"

# 3. User ACLs
pveum acl modify / --users terraform@pve --roles TerraformProv
pveum acl modify / --users packer@pve    --roles PackerProv
pveum acl modify / --users pulse@pve     --roles PVEAuditor

# 4. Tokens (the secret is displayed ONCE) + token ACLs
pveum user token add terraform@pve iac     --privsep 1 --comment "homelab-IaC"
pveum user token add packer@pve    build   --privsep 1
pveum user token add pulse@pve     monitor --privsep 1
pveum acl modify / --tokens 'terraform@pve!iac'  --roles TerraformProv
pveum acl modify / --tokens 'packer@pve!build'   --roles PackerProv
pveum acl modify / --tokens 'pulse@pve!monitor'  --roles PVEAuditor
```

### GUI equivalent
Datacenter → Permissions:
1. **Roles** → Create (name + privileges above).
2. **Users** → Add (realm *Proxmox VE authentication server*, no password).
3. **API Tokens** → Add (keep *Privilege Separation* checked). Copy the secret immediately.
4. **Permissions** → Add → *User Permission*, then *API Token Permission*, path `/`, propagate enabled.

## Verification

```bash
pveum user token permissions terraform@pve iac --path /
curl -k -H "Authorization: PVEAPIToken=terraform@pve!iac=<secret>" https://192.168.2.200:8006/api2/json/version
```

## Consumption

Terraform reads the token from environment variables, stored as **sensitive** HCP Terraform variables (later Vault):

```text
PROXMOX_VE_ENDPOINT=https://192.168.2.200:8006/
PROXMOX_VE_API_TOKEN=terraform@pve!iac=<secret>
```

Never commit token secrets. Start with this privilege set, validate with the first `terraform plan`/`apply`, then tighten.

## Known Limitations

API tokens cannot perform operations restricted to `root@pam`:
* host bind mounts in LXCs (`mp0: /host/path,...`);
* device passthrough (e.g. `/dev/dri` for Jellyfin);
* some LXC `features` flags.

These are handled over SSH (Ansible) or manually. The `bpg/proxmox` provider also needs SSH access to upload cloud-init snippets; a dedicated SSH user will be documented when required.

## Rotation & Revocation

```bash
pveum user token remove terraform@pve iac
pveum user token add terraform@pve iac --privsep 1 --comment "homelab-IaC"
pveum acl modify / --tokens 'terraform@pve!iac' --roles TerraformProv
```
Update the secret in HCP Terraform (or Vault) right after rotation.
