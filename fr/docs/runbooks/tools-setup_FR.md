# Installation des outils du poste de travail

Inventaire des outils CLI requis sur le poste Windows local.

> 🇬🇧 English version: [../../../docs/runbooks/tools-setup.md](../../../docs/runbooks/tools-setup.md)

## Inventaire

| Outil | Version | Usage | Source | Commande d'installation |
|---|---|---|---|---|
| GitHub CLI (`gh`) | 2.102.0 | Gestion du dépôt, des PR et des Actions | winget | `winget install --id GitHub.cli -e` |
| Terraform | 1.14.0 | Provisionnement de l'infrastructure | Chocolatey | `choco install terraform -y` |
| Trivy | 0.75.0 | Analyse de configuration IaC et de vulnérabilités | winget | `winget install --id AquaSecurity.Trivy -e` |
| TFLint | 0.64.0 | Linting Terraform | winget | `winget install --id TerraformLinters.tflint -e` |
| Gitleaks | 8.30.1 | Détection de secrets | winget | `winget install --id Gitleaks.Gitleaks -e` |
| git-cliff | 2.14.2 | Génération du changelog à partir des commits conventionnels | winget | `winget install --id orhun.git-cliff -e` |

Ouvrir un nouveau terminal après l'installation pour rafraîchir le `PATH`.

## Configuration

### GitHub CLI
```powershell
gh auth login          # compte : hdti-devops
gh auth setup-git      # utiliser gh comme gestionnaire d'identifiants git
gh auth status
```

### Terraform / HCP Terraform
```powershell
terraform login        # enregistre un token utilisateur HCP Terraform pour l'Org "hdti"
```
Les workspaces du Projet `homelab` utilisent le **mode d'exécution local**.

## Vérification
```powershell
gh --version
terraform version
trivy --version
tflint --version
gitleaks version
git-cliff --version
```
