# Workstation Tools Setup

Inventory of the CLI tools required on the local Windows workstation.

> 🇫🇷 French version: [../../fr/docs/runbooks/tools-setup_FR.md](../../fr/docs/runbooks/tools-setup_FR.md)

## Inventory

| Tool | Version | Purpose | Install source | Install command |
|---|---|---|---|---|
| GitHub CLI (`gh`) | 2.102.0 | Repository, PR, and Actions management | winget | `winget install --id GitHub.cli -e` |
| Terraform | 1.14.0 | Infrastructure provisioning | Chocolatey | `choco install terraform -y` |
| Trivy | 0.75.0 | IaC misconfiguration and vulnerability scanning | winget | `winget install --id AquaSecurity.Trivy -e` |
| TFLint | 0.64.0 | Terraform linting | winget | `winget install --id TerraformLinters.tflint -e` |
| Gitleaks | 8.30.1 | Secret scanning | winget | `winget install --id Gitleaks.Gitleaks -e` |
| git-cliff | 2.14.2 | Changelog generation from conventional commits | winget | `winget install --id orhun.git-cliff -e` |

Open a new terminal after installation so that `PATH` is refreshed.

## Configuration

### GitHub CLI
```powershell
gh auth login          # account: hdti-devops
gh auth setup-git      # use gh as git credential helper
gh auth status
```

### Terraform / HCP Terraform
```powershell
terraform login        # stores an HCP Terraform user token for Org "hdti"
```
Workspaces in Project `homelab` use **local execution mode**.

## Verification
```powershell
gh --version
terraform version
trivy --version
tflint --version
gitleaks version
git-cliff --version
```
