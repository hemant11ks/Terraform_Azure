# Azure Terraform — Complete DevOps & Interview Guide

> **Goal:** Learn Infrastructure as Code (IaC) on Microsoft Azure with Terraform, from basic resources to secure remote state, reusable modules, and Azure DevOps CI/CD.
>
> **Updated:** November 2026 · **Audience:** DevOps Engineers, Cloud Engineers, SREs, Platform Engineers, and interview candidates.
>
> **Version note:** Examples target the **HashiCorp AzureRM provider 5.x**. Provider 5.x introduced breaking changes versus 4.x, including a change to resource-provider registration defaults. Always check your locked provider version before copying an older tutorial.

---

## Table of Contents

1. [Terraform and Azure: fundamentals](#1-terraform-and-azure-fundamentals)
2. [Architecture and resource mapping](#2-architecture-and-resource-mapping)
3. [Prerequisites and installation](#3-prerequisites-and-installation)
4. [Authentication and permissions](#4-authentication-and-permissions)
5. [First hands-on Azure deployment](#5-first-hands-on-azure-deployment)
6. [Terraform workflow and commands](#6-terraform-workflow-and-commands)
7. [Remote state with Azure Blob Storage](#7-remote-state-with-azure-blob-storage)
8. [Reusable modules and environments](#8-reusable-modules-and-environments)
9. [Azure DevOps CI/CD pipeline](#9-azure-devops-cicd-pipeline)
10. [Security, governance, and production practices](#10-security-governance-and-production-practices)
11. [Advanced Terraform patterns](#11-advanced-terraform-patterns)
12. [Common errors and troubleshooting](#12-common-errors-and-troubleshooting)
13. [Interview questions and answers](#13-interview-questions-and-answers)
14. [Practical projects and roadmap](#14-practical-projects-and-roadmap)
15. [Useful links and references](#15-useful-links-and-references)

---

## 1. Terraform and Azure: fundamentals

### What is Terraform?

Terraform is a declarative Infrastructure as Code tool. You describe the **desired infrastructure** in `.tf` files; Terraform determines the changes required to create, update, or delete resources.

**Example:** instead of manually opening the Azure Portal to create a resource group, virtual network, and subnet, commit their HCL definitions to Git and deploy them through a controlled pipeline.

| Concept | What it means | Azure example |
|---|---|---|
| IaC | Infrastructure described in versioned code | VNet and NSG defined in `.tf` |
| Provider | Plugin that calls a cloud or service API | `hashicorp/azurerm` |
| Resource | Object Terraform creates/manages | `azurerm_resource_group` |
| Data source | Reads an existing object | `data.azurerm_resource_group` |
| Variable | Input to a configuration | `location = "centralindia"` |
| Output | A result exposed by a module | VNet ID |
| State | Mapping between configuration and real infrastructure | `terraform.tfstate` |
| Backend | Where state is stored | Azure Blob Storage |
| Module | Reusable collection of Terraform files | `modules/network` |
| Plan | Preview of infrastructure changes | `terraform plan` |
| Drift | Real infrastructure differs from expected configuration | Portal change to subnet |

### Declarative vs. imperative

```text
Imperative:  "Create a resource group, then a VNet, then this subnet."
Declarative: "These resources must exist with these properties."
```

Terraform constructs a dependency graph from resource references, determines the operations, and executes independent operations in parallel when safe.

### Terraform vs. Bicep vs. ARM templates

| Tool | Best suited for | Key difference |
|---|---|---|
| Terraform | Multi-cloud / multi-provider IaC | HCL, state-driven planning, modules |
| Azure Bicep | Azure-native infrastructure | Compiles to ARM deployment templates |
| ARM JSON | Azure resource declarations | Verbose, low-level Azure-native syntax |
| Ansible | Configuration management / automation | Often imperative tasks; complements Terraform |

**Typical approach:** Terraform provisions Azure infrastructure, while cloud-init, Ansible, or application-deployment tools configure workloads.

---

## 2. Architecture and resource mapping

```text
Developer / Pull Request
          |
          v
Azure Repos or GitHub
          |
          v
Azure DevOps Pipeline
  fmt -> validate -> plan -> approval -> apply
          |
          | Workload Identity Federation (OIDC)
          v
Azure Resource Manager API
          |
          +--> Resource Group
          |      +--> Virtual Network / Subnet / NSG
          |      +--> App Service / VM / AKS / Key Vault
          |
          +--> Remote Terraform State
                 Azure Storage Account -> Blob Container
                 (RBAC, encryption, blob lease locking)
```

### Frequently used Azure resources

| Azure service | Terraform resource / data source | Common use case |
|---|---|---|
| Resource Group | `azurerm_resource_group` | Logical resource organization |
| Virtual Network | `azurerm_virtual_network` | Private network |
| Subnet | `azurerm_subnet` | Segment a VNet |
| Network Security Group | `azurerm_network_security_group` | Network access rules |
| Public IP | `azurerm_public_ip` | Internet-facing entry point |
| Virtual Machine | `azurerm_linux_virtual_machine` | Compute workloads |
| Storage Account | `azurerm_storage_account` | Blob/file/queue/table storage |
| Blob Container | `azurerm_storage_container` | Objects and Terraform state |
| Key Vault | `azurerm_key_vault` | Secret and key management |
| App Service Plan | `azurerm_service_plan` | Compute for web apps |
| Linux Web App | `azurerm_linux_web_app` | Host an API/web app |
| Container Registry | `azurerm_container_registry` | Store container images |
| AKS | `azurerm_kubernetes_cluster` | Managed Kubernetes |
| Log Analytics Workspace | `azurerm_log_analytics_workspace` | Central log analytics |
| Role Assignment | `azurerm_role_assignment` | Azure RBAC |
| Existing RG lookup | `data "azurerm_resource_group"` | Read existing infrastructure |

> The AzureRM provider manages Azure infrastructure. To manage **Azure DevOps organizations/projects/repositories/pipelines as resources**, consider the separate `microsoft/azuredevops` Terraform provider. For newer/unsupported Azure APIs, evaluate the `Azure/azapi` provider.

---

## 3. Prerequisites and installation

You need:

- An Azure subscription with permission to create resources and, for remote state, configure storage/RBAC.
- Azure CLI (`az`), Terraform CLI, and Git.
- A terminal such as Bash on Ubuntu or Azure Cloud Shell.
- An Azure DevOps organization/project only for the CI/CD exercise.

Check installation:

```bash
az version
terraform version
git --version
```

Official installation instructions:

- [Terraform CLI](https://developer.hashicorp.com/terraform/install)
- [Azure CLI on Linux](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-linux)
- [Azure Cloud Shell](https://learn.microsoft.com/en-us/azure/cloud-shell/overview)

**Important:** Cloud Shell may already include required tooling, but confirm versions. Prefer officially supported installers over downloading unverified binaries.

### Configure Azure CLI

```bash
az login
az account list --output table
az account set --subscription "<YOUR_SUBSCRIPTION_ID>"
az account show --output table

# Required for the provider configuration below
export ARM_SUBSCRIPTION_ID="$(az account show --query id -o tsv)"
```

For AzureRM **v5**, explicitly setting `ARM_SUBSCRIPTION_ID` keeps the target subscription unambiguous. Do not place a real subscription or secret in a public example repository.

---

## 4. Authentication and permissions

| Environment | Recommended authentication | Notes |
|---|---|---|
| Local development | `az login` (Azure CLI) | Good for interactive use |
| Azure DevOps / GitHub Actions | Workload Identity Federation (OIDC) | Avoid long-lived client secrets |
| Azure VM / managed runtime | Managed identity | Avoid secrets where supported |
| Legacy automation | Service principal secret/certificate | Prefer OIDC; rotate credentials if used |

**Two separate permission paths:**

1. **Management plane:** create/read/update Azure resources (for example, Contributor scoped to the lab resource group).
2. **Data plane for state:** read/write blob state (usually `Storage Blob Data Contributor` on the state container).

A Contributor role on the subscription/resource group **does not automatically grant blob data-plane access**.

### AzureRM provider authentication

```hcl
# versions.tf
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 5.0" # Allow 5.x, not 6.x
    }
  }
}

provider "azurerm" {
  features {}
  subscription_id = var.subscription_id
}
```

The lab uses an input `subscription_id`, or you can omit the provider argument and use the `ARM_SUBSCRIPTION_ID` environment variable. **Use one consistent subscription source**, and keep credentials out of source code.

**AzureRM v5 caveat:** automatic resource-provider registration defaults changed. If your role cannot register resource providers, ask the subscription administrator to register the required namespaces. For the networking lab, check `Microsoft.Network`:

```bash
az provider show --namespace Microsoft.Network --query registrationState -o tsv
# Run only if you have appropriate permissions:
az provider register --namespace Microsoft.Network
```

---

## 5. First hands-on Azure deployment

### Lab objective

Provision a small Azure network using Terraform:

```text
Resource Group: rg-tf-lab-dev
  └─ Virtual Network: vnet-tf-lab-dev (10.20.0.0/16)
      └─ Subnet: snet-app (10.20.1.0/24)
          └─ Associated Network Security Group: nsg-app
```

This lab intentionally creates **no VM, AKS cluster, public IP, or App Service**. It minimizes costs, although Azure billing and policy effects still depend on the subscription.

### Folder structure

```text
azure-terraform-lab/
├── versions.tf
├── variables.tf
├── main.tf
├── outputs.tf
├── terraform.tfvars.example
├── .gitignore
└── README.md
```

### File 1 — `versions.tf`

```hcl
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 5.0"
    }
  }
}

provider "azurerm" {
  features {}
  subscription_id = var.subscription_id
}
```

### File 2 — `variables.tf`

```hcl
variable "subscription_id" {
  description = "Azure subscription where this lab is deployed"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "test", "prod"], var.environment)
    error_message = "Allowed values: dev, test, prod."
  }
}

variable "location" {
  description = "Azure region"
  type        = string
  default     = "centralindia"
}

variable "project" {
  description = "Project name for resource naming"
  type        = string
  default     = "tf-lab"
}
```

### File 3 — `main.tf`

```hcl
locals {
  prefix = "${var.project}-${var.environment}"
  tags = {
    project     = var.project
    environment = var.environment
    managed_by  = "terraform"
  }
}

resource "azurerm_resource_group" "lab" {
  name     = "rg-${local.prefix}"
  location = var.location
  tags     = local.tags
}

resource "azurerm_virtual_network" "lab" {
  name                = "vnet-${local.prefix}"
  address_space       = ["10.20.0.0/16"]
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name
  tags                = local.tags
}

resource "azurerm_subnet" "app" {
  name                 = "snet-app"
  resource_group_name  = azurerm_resource_group.lab.name
  virtual_network_name = azurerm_virtual_network.lab.name
  address_prefixes     = ["10.20.1.0/24"]
}

resource "azurerm_network_security_group" "app" {
  name                = "nsg-app-${local.prefix}"
  location            = azurerm_resource_group.lab.location
  resource_group_name = azurerm_resource_group.lab.name
  tags                = local.tags
}

resource "azurerm_subnet_network_security_group_association" "app" {
  subnet_id                 = azurerm_subnet.app.id
  network_security_group_id = azurerm_network_security_group.app.id
}
```

Note that **no inbound allow rule** is defined in the NSG. Add explicit least-privilege rules only when the workload requires them; avoid opening SSH/RDP to the world.

### File 4 — `outputs.tf`

```hcl
output "resource_group_name" {
  value       = azurerm_resource_group.lab.name
  description = "Created resource group"
}

output "vnet_id" {
  value       = azurerm_virtual_network.lab.id
  description = "Created virtual network ID"
}

output "subnet_id" {
  value       = azurerm_subnet.app.id
  description = "Created application subnet ID"
}
```

### File 5 — `terraform.tfvars.example`

```hcl
subscription_id = "00000000-0000-0000-0000-000000000000" # Replace locally
location        = "centralindia"
project         = "tf-lab"
environment     = "dev"
```

Make a local copy and edit it:

```bash
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars to set your real Azure subscription ID.
```

### File 6 — `.gitignore`

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfplan
crash.log
crash.*.log
override.tf
override.tf.json
*_override.tf
*_override.tf.json
terraform.tfvars
*.auto.tfvars
*.auto.tfvars.json
!*.tfvars.example
```

**Commit `.terraform.lock.hcl`** for reproducible provider selections. Do not indiscriminately ignore it. Sensitive variables or secret outputs can still appear in state or saved plans; never commit either.

### Execute the lab

```bash
terraform init
terraform fmt -recursive
terraform fmt -check -recursive
terraform validate
terraform plan -out=tfplan
terraform apply tfplan
terraform output

# Verify in Azure
az group show --name rg-tf-lab-dev --output table
az network vnet list --resource-group rg-tf-lab-dev --output table
```

Before using an unfamiliar lab, review `terraform plan`. In real environments, **require approval** before any destructive change.

### Clean up to avoid charges

```bash
terraform plan -destroy
terraform destroy
```

Type `yes` only after reviewing the resources to be removed. Destroying the lab resource group deletes resources owned by that configuration. **Never run `destroy` against a shared or production state by accident.**

---

## 6. Terraform workflow and commands

### Core lifecycle

```text
Write HCL -> terraform fmt -> terraform init -> terraform validate
       -> terraform plan -> review/approve -> terraform apply
       -> monitor drift -> update / destroy when appropriate
```

| Command | Purpose |
|---|---|
| `terraform init` | Download providers/modules and initialize backend |
| `terraform fmt -recursive` | Format all `.tf` files |
| `terraform fmt -check -recursive` | Fail CI if formatting is wrong |
| `terraform validate` | Check configuration syntax/consistency |
| `terraform plan` | Preview proposed changes |
| `terraform plan -out=tfplan` | Save exact plan for review/apply |
| `terraform apply tfplan` | Apply a previously saved plan |
| `terraform destroy` | Delete resources tracked in current state |
| `terraform output` | Display root module outputs |
| `terraform state list` | List tracked resource addresses |
| `terraform state show <address>` | Inspect a resource recorded in state |
| `terraform providers` | Show provider dependencies |
| `terraform console` | Experiment with HCL expressions |
| `terraform show` | Human-readable state or plan details |
| `terraform import ADDRESS ID` | Bring existing resource under state management |
| `terraform plan -refresh-only` | Preview state refresh without config changes |
| `terraform apply -replace=ADDRESS` | Explicitly recreate a resource |
| `terraform init -upgrade` | Re-evaluate provider/module selections |

**Plan result codes for CI:** `terraform plan -detailed-exitcode` returns `0` for no changes, `2` for changes present, and `1` for an error. A nonzero code `2` is not a failure when your CI script handles it intentionally.

### HCL reference examples

```hcl
# Expression and reference
name = "rg-${var.project}-${var.environment}"

# Local value
locals { common_tags = { owner = "devops" } }

# Data source: read an existing resource (does not create it)
data "azurerm_resource_group" "existing" {
  name = "rg-existing"
}

# Explicit dependency (only when implicit references are insufficient)
# depends_on = [azurerm_role_assignment.example]
```

**State safety:** do not manually edit `.tfstate`; use supported import/move/state commands with backup and change control. Removing something from state **does not delete the real Azure resource**.

---

## 7. Remote state with Azure Blob Storage

### Why remote state?

Local state is fragile for teams and pipelines. Azure Blob Storage provides centralized state, encryption at rest, and **blob lease-based locking** to reduce concurrent-write risk.

```text
Terraform Client / Agent
        |
        | Microsoft Entra ID auth
        v
Azure Storage Account (dedicated for state)
        └── Private blob container: tfstate
              ├── dev.network.tfstate
              ├── test.network.tfstate
              └── prod.network.tfstate
```

### Step A — Bootstrap the backend outside the workload state

Run once with appropriately privileged Azure account. Use a storage account name that is globally unique (3–24 lowercase alphanumeric characters).

```bash
az login
az account set --subscription "<YOUR_SUBSCRIPTION_ID>"

STATE_RG="rg-terraform-state"
STATE_LOCATION="centralindia"
STATE_SA="tfstate$(openssl rand -hex 5)"
STATE_CONTAINER="tfstate"

az group create --name "$STATE_RG" --location "$STATE_LOCATION"

az storage account create \
  --name "$STATE_SA" \
  --resource-group "$STATE_RG" \
  --location "$STATE_LOCATION" \
  --sku Standard_LRS \
  --kind StorageV2 \
  --allow-blob-public-access false \
  --min-tls-version TLS1_2

STATE_SA_ID="$(az storage account show \
  --resource-group "$STATE_RG" --name "$STATE_SA" --query id -o tsv)"

# Blob data-plane permission for the signed-in user; requires RBAC assignment rights.
MY_OBJECT_ID="$(az ad signed-in-user show --query id -o tsv)"
az role assignment create \
  --assignee-object-id "$MY_OBJECT_ID" \
  --assignee-principal-type User \
  --role "Storage Blob Data Contributor" \
  --scope "$STATE_SA_ID"

# RBAC propagation can take a few minutes.
az storage container create \
  --account-name "$STATE_SA" \
  --name "$STATE_CONTAINER" \
  --auth-mode login

printf 'Backend storage account: %s\n' "$STATE_SA"
```

**Security:** for production, enable blob versioning/soft delete, diagnostics, network restrictions or private endpoints, and tightly scoped RBAC. A hosted public agent cannot reach a private endpoint unless network connectivity is provided. Grant the **pipeline identity** its own blob data-plane access; the local user's role assignment does not carry over.

### Step B — Add a backend block

In the lab, add to `versions.tf` inside the existing `terraform { ... }` block:

```hcl
backend "azurerm" {
  resource_group_name  = "rg-terraform-state"
  storage_account_name = "<REPLACE_WITH_UNIQUE_STATE_ACCOUNT>"
  container_name       = "tfstate"
  key                  = "dev.network.tfstate"
  use_azuread_auth     = true
}
```

You **cannot** use `var.*` references inside a Terraform backend block. Backend configuration is processed during `terraform init` before regular variables are evaluated. Alternatively, leave an empty `backend "azurerm" {}` block and pass non-secret configuration using `-backend-config` or a checked-in, non-secret `.hcl` file.

### Step C — Migrate local state

```bash
# For your local Azure CLI login, use Entra ID authentication to state.
export ARM_USE_CLI=true
export ARM_USE_AZUREAD=true
export ARM_SUBSCRIPTION_ID="$(az account show --query id -o tsv)"

terraform init -migrate-state
terraform state list
```

Confirm the state blob exists in the expected container. Treat the state file as confidential even if an HCL variable was marked `sensitive = true`.

### State locking and recovery

- Azure Blob lease locking prevents two supported Terraform clients from modifying the same state concurrently.
- On a legitimate stuck lock, investigate the old process before considering `terraform force-unlock <LOCK_ID>`.
- **Never** break another team's active lock.
- Use separate **state keys** (or separate accounts/containers) for independently deployable environments.
- Bootstrap state storage separately; do not make the backend depend on resources stored **inside itself**.

---

## 8. Reusable modules and environments

### Recommended repository structure

```text
infrastructure/
├── modules/
│   ├── network/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── app-service/
│   └── key-vault/
├── environments/
│   ├── dev/
│   │   ├── versions.tf
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── test/
│   └── prod/
└── azure-pipelines.yml
```

Each environment should have its **own backend state key, approval policy, and appropriate permissions**.

### Example network module interface

`modules/network/variables.tf`:

```hcl
variable "name"                { type = string }
variable "location"            { type = string }
variable "resource_group_name" { type = string }
variable "address_space"       { type = list(string) }
```

`modules/network/main.tf`:

```hcl
resource "azurerm_virtual_network" "this" {
  name                = var.name
  location            = var.location
  resource_group_name = var.resource_group_name
  address_space       = var.address_space
}
```

`modules/network/outputs.tf`:

```hcl
output "vnet_id" { value = azurerm_virtual_network.this.id }
```

Call it from an environment root's `main.tf`:

```hcl
resource "azurerm_resource_group" "this" {
  name     = "rg-demo-dev"
  location = "centralindia"
}

module "network" {
  source              = "../../modules/network"
  name                = "vnet-demo-dev"
  location            = azurerm_resource_group.this.location
  resource_group_name = azurerm_resource_group.this.name
  address_space       = ["10.30.0.0/16"]
}

output "vnet_id" {
  value = module.network.vnet_id
}
```

**Note:** This is an **alternative root configuration** to the first lab. Do not paste this example into the same root as the original VNet unless you intend to create a second VNet.

### Environment isolation options

| Strategy | Pros | Risks / notes |
|---|---|---|
| Separate root folders/states | Clear separation and review | Some code duplication; modules reduce it |
| Terraform workspaces | Convenient for simple variants | Same code/backend context; weaker separation |
| Separate subscriptions | Stronger security and billing boundary | More governance and identity setup |

For production systems, prefer separate states and strict identity boundaries over relying on workspaces alone.

---

## 9. Azure DevOps CI/CD pipeline

### Goal

Automate validation and review, then apply **only after approval**:

```text
Commit / PR -> fmt -> init -> validate -> plan
                                      |
                                Review + approval
                                      |
                                   apply
```

### Prerequisites

1. Create an **Azure Resource Manager service connection** in Azure DevOps using **Workload Identity Federation**.
2. Give that identity scoped Azure management permissions and `Storage Blob Data Contributor` on the state container.
3. Set up the remote backend in advance (Section 7).
4. Install the [Terraform extension for Azure Pipelines](https://marketplace.visualstudio.com/items?itemName=ms-devlabs.custom-terraform-tasks) if using the `TerraformInstaller@1` task below.
5. Configure a protected Azure DevOps **Environment** called `terraform-dev` and add its approval check in the Azure DevOps UI. Merely naming an environment in YAML **does not create an approval**.
6. Keep the Terraform configuration under `infrastructure/environments/dev` (or change the YAML `workingDirectory`).

### Example `azure-pipelines.yml`

> Adapt resource names, service connection, and file paths. This is a reference pipeline, not a preconfigured deployment. Its saved plan artifact may contain sensitive data; restrict access/retention appropriately.

```yaml
trigger:
  branches:
    include:
      - main

pr:
  branches:
    include:
      - main

pool:
  vmImage: ubuntu-24.04

variables:
  azureServiceConnection: 'sc-azure-terraform-wif'  # Create this first
  tfWorkingDir: 'infrastructure/environments/dev'
  tfVersion: '1.11.3'  # Example pinned version; update deliberately

stages:
  - stage: Plan
    displayName: Validate and plan
    jobs:
      - job: TerraformPlan
        steps:
          - checkout: self

          - task: TerraformInstaller@1
            inputs:
              terraformVersion: '$(tfVersion)'

          - task: AzureCLI@2
            displayName: Terraform fmt, validate and plan
            inputs:
              azureSubscription: '$(azureServiceConnection)'
              addSpnToEnvironment: true
              scriptType: bash
              scriptLocation: inlineScript
              workingDirectory: '$(tfWorkingDir)'
              inlineScript: |
                set -euo pipefail
                export ARM_CLIENT_ID="$servicePrincipalId"
                export ARM_TENANT_ID="$tenantId"
                export ARM_SUBSCRIPTION_ID="$(az account show --query id -o tsv)"
                export ARM_USE_OIDC=true
                export ARM_OIDC_TOKEN="$idToken"
                export ARM_USE_AZUREAD=true

                terraform fmt -check -recursive
                terraform init -input=false
                terraform validate
                terraform plan -input=false \
                  -out="$(Build.ArtifactStagingDirectory)/tfplan"

          - task: PublishPipelineArtifact@1
            displayName: Publish reviewed plan
            inputs:
              targetPath: '$(Build.ArtifactStagingDirectory)/tfplan'
              artifact: 'tfplan'

  - stage: Apply
    displayName: Apply approved changes
    dependsOn: Plan
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: TerraformApply
        environment: 'terraform-dev'  # Configure manual approval check in UI
        strategy:
          runOnce:
            deploy:
              steps:
                - checkout: self

                - download: current
                  artifact: tfplan

                - task: TerraformInstaller@1
                  inputs:
                    terraformVersion: '$(tfVersion)'

                - task: AzureCLI@2
                  displayName: Apply saved plan
                  inputs:
                    azureSubscription: '$(azureServiceConnection)'
                    addSpnToEnvironment: true
                    scriptType: bash
                    scriptLocation: inlineScript
                    workingDirectory: '$(tfWorkingDir)'
                    inlineScript: |
                      set -euo pipefail
                      export ARM_CLIENT_ID="$servicePrincipalId"
                      export ARM_TENANT_ID="$tenantId"
                      export ARM_SUBSCRIPTION_ID="$(az account show --query id -o tsv)"
                      export ARM_USE_OIDC=true
                      export ARM_OIDC_TOKEN="$idToken"
                      export ARM_USE_AZUREAD=true

                      terraform init -input=false
                      terraform apply -input=false \
                        "$(Pipeline.Workspace)/tfplan/tfplan"
```

**Pipeline behavior:** PRs run the Plan stage only; `main` also reaches Apply. Apply is gated by the **Environment approval configured in Azure DevOps**. If new commits change the infrastructure after approval, create a fresh reviewed plan. Use branch protection and deployment concurrency controls.

**OIDC note:** this example uses the short-lived `idToken` exposed by `AzureCLI@2` to keep the script readable. For unusually long-running operations, configure Azure DevOps OIDC request-token/refresh support as described in the official AzureRM provider and backend documentation rather than relying on one expiring token.

### Recommended production pipeline improvements

- Add policy/security validation (`tflint`, `tfsec`/Trivy, Checkov or organization-approved tools).
- Use private module registries or pinned module revisions, and commit `.terraform.lock.hcl`.
- Restrict who can approve production applies and who can download plan artifacts.
- Use separate service connections for dev/test/prod and least-privilege RBAC.
- Require pull requests and human review for infrastructure changes.
- Detect drift on a schedule with read-only plans and alert on unexpected changes.
- For strict reproducibility, pin the Terraform CLI and provider versions in both plan and apply stages.

---

## 10. Security, governance, and production practices

### Checklist

- [ ] Use Entra ID / OIDC / managed identities; avoid committed client secrets.
- [ ] Store remote state in a dedicated, access-controlled Azure Storage account.
- [ ] Restrict state data-plane and management-plane permissions independently.
- [ ] Enable soft delete/versioning for state and test recovery procedures.
- [ ] Protect `.tfstate`, plan artifacts, and CI logs as potentially sensitive.
- [ ] Set network rules and private endpoints where operationally feasible.
- [ ] Use Azure RBAC least privilege, Azure Policy, resource locks where appropriate.
- [ ] Tag resources (`owner`, `project`, `environment`, `cost_center`).
- [ ] Set budgets/alerts to avoid unexpected Azure spending.
- [ ] Review all deletes/replacements; require approvals for production.
- [ ] Back up state and plan migrations before upgrading provider major versions.
- [ ] Never modify Azure resources manually without reconciling Terraform drift.
- [ ] Never run multiple writers against the same state without locking.

### Sensitive values example

```hcl
variable "api_token" {
  type      = string
  sensitive = true
}
```

`sensitive = true` reduces routine CLI display of the value; it **does not guarantee encryption or absence from state**. Use Key Vault/secret-management processes and minimize secrets in Terraform-managed arguments.

### Common enterprise additions

- Azure landing zones, management groups, subscription organization.
- Azure Policy / Defender for Cloud alignment.
- Private networking, DNS, Private Link, and firewall strategy.
- Diagnostic settings, Log Analytics, Azure Monitor alerts.
- Cost management: rightsizing, environment shutdown, lifecycle cleanup.
- Multi-subscription management with provider aliases and separate identities.

---

## 11. Advanced Terraform patterns

### A. `for_each` vs. `count`

```hcl
# Use stable keys for instances that evolve over time.
resource "azurerm_resource_group" "team" {
  for_each = toset(["dev", "test"])
  name     = "rg-team-${each.key}"
  location = "centralindia"
}
```

- `count` identifies instances by numeric index; removing an early list item can shift addresses.
- `for_each` identifies instances by stable keys; generally better for named environments/resources.

### B. Data sources

```hcl
data "azurerm_resource_group" "shared" {
  name = "rg-shared-services"
}

output "shared_rg_id" {
  value = data.azurerm_resource_group.shared.id
}
```

### C. Import an existing resource

```bash
# First define the matching resource block in HCL.
terraform import azurerm_resource_group.lab \
  "/subscriptions/<SUB_ID>/resourceGroups/rg-tf-lab-dev"

terraform plan # Reconcile any property differences
```

**Terraform import is not automatic reconciliation of all child resources.** Define and import each object you intend to manage (or use supported import blocks/generation workflows).

### D. Move a resource address without destroying it

```hcl
moved {
  from = azurerm_resource_group.old_name
  to   = azurerm_resource_group.lab
}
```

Use `moved` blocks to refactor addresses safely when compatible. Cross-state moves and imports require additional care.

### E. Lifecycle behavior

```hcl
# Illustrative only — place inside a suitable resource block.
lifecycle {
  prevent_destroy = true
}
```

`prevent_destroy` helps block a Terraform plan that would destroy the **resource while its block remains in configuration**. It is not a substitute for RBAC, backups, or Azure resource locks.

### F. Provider aliases across subscriptions

```hcl
provider "azurerm" {
  alias           = "shared"
  subscription_id = var.shared_subscription_id
  features {}
}

# Example usage in an appropriate resource/data block:
# provider = azurerm.shared
```

Ensure the identity has permission in every targeted subscription and that modules explicitly receive required alias mappings.

---

## 12. Common errors and troubleshooting

| Symptom / error | Likely reason | What to check |
|---|---|---|
| Subscription ID missing | AzureRM provider config/environment incomplete | `az account show`; `ARM_SUBSCRIPTION_ID` or provider `subscription_id` |
| `403 AuthorizationFailed` during resource create | Missing Azure management permission | Correct scope, role, subscription, propagation |
| `403` on state blob | Missing storage **data-plane** role | `Storage Blob Data Contributor`, auth mode, firewall |
| Provider not registered | Namespace not registered or insufficient rights | `az provider show` / admin registration |
| Resource already exists | Resource created outside current state | Inspect, define HCL, import before managing |
| State lock error | Another execution is active or left a lock | Verify active runs; force-unlock only if safe |
| Storage account name unavailable | Global name collision or invalid syntax | Use a different lowercase alphanumeric name |
| Wrong region/SKU | Azure service/VM SKU not available there | Choose supported region/SKU and check quotas |
| Unauthorized pipeline | Service connection not granted access | WIF trust, service connection authorization, roles |
| OIDC token expired | Pipeline ran too long with static token | Use token-refresh compatible OIDC auth |
| Unexpected replacement | Immutable property changed or provider upgrade | Review provider changelog and plan carefully |
| Changes made in Portal | Drift from Terraform configuration | Compare plan; update code or revert manual changes |
| `terraform init` backend mismatch | Backend config/state location changed | Use `-migrate-state` or `-reconfigure` appropriately |
| Unexpected Azure bill | Billable resources left deployed | Review Cost Management; destroy lab resources |

Useful diagnostics:

```bash
az account show -o json
az provider show --namespace Microsoft.Network --query registrationState -o tsv
terraform version
terraform providers
terraform validate
terraform state list
terraform plan -refresh-only
```

For an access error, examine **whether it concerns the Azure control plane or the storage data plane** before changing role assignments.

---

## 13. Interview questions and answers

### Fundamentals

**1. What problem does Terraform solve?**  
It turns infrastructure into reviewable, repeatable, version-controlled code and calculates a change plan before applying it.

**2. What is the AzureRM provider?**  
A Terraform plugin for provisioning and managing resources through Azure Resource Manager APIs.

**3. What is the difference between a provider and a backend?**  
A provider manages infrastructure APIs; a backend stores Terraform state and may provide locking.

**4. What is Terraform state, and why is it needed?**  
State maps resource addresses in HCL to actual remote objects and records metadata needed to plan changes.

**5. What is the difference between `plan` and `apply`?**  
`plan` previews proposed changes; `apply` executes them. A saved plan allows you to apply exactly what was reviewed, provided the infrastructure has not invalidated it.

**6. Does `validate` confirm Azure permissions or quota?**  
No. It primarily checks configuration validity. Most API, permission, capacity, and policy constraints require plan/apply or separate validation.

**7. How do you handle variables across dev, test, and prod?**  
Use separate root configurations, state keys, environment-specific inputs, and scoped identities. Reuse common modules.

**8. What is the difference between resource blocks and data sources?**  
A resource is managed by Terraform; a data source reads information from an existing object.

### State and collaboration

**9. Why use Azure Blob as a backend?**  
Centralized state storage, encryption at rest, and blob-lease locking enable safer team workflows.

**10. What is state locking?**  
A mechanism preventing conflicting concurrent Terraform state writes. AzureRM backend uses Azure Blob leases.

**11. How do you recover from a stuck state lock?**  
Check active jobs and identify the lock owner first; only force-unlock a verified abandoned lock following your change process.

**12. Is a `sensitive` Terraform variable absent from state?**  
No. It may still be present in the state and plan; restrict those artifacts.

**13. How do you migrate state from local to Azure?**  
Bootstrap backend storage, configure `backend "azurerm"`, authenticate, run `terraform init -migrate-state`, and verify.

**14. How do you import manually created Azure resources?**  
Define the matching HCL, import by Azure resource ID, then run plan to reconcile attributes.

**15. What is drift?**  
A difference between real infrastructure and Terraform's intended configuration, often from manual Portal/CLI changes.

### Azure and security

**16. How should Terraform authenticate in Azure DevOps?**  
Prefer Azure Resource Manager service connections using Workload Identity Federation (OIDC) rather than long-lived service principal secrets.

**17. Why might a pipeline create a VNet but fail to read the state blob?**  
Azure management-plane permissions and Storage Blob data-plane permissions are distinct.

**18. What changed in AzureRM 5.0?**  
It introduced breaking changes including resource-provider registration behavior; the default moved to `none`. Read the 5.0 upgrade guide before upgrading.

**19. How do you deploy across Azure subscriptions?**  
Configure provider aliases and scoped credentials, with explicit module provider mappings and appropriate state separation.

**20. How do you reduce the blast radius of Terraform?**  
Separate subscriptions/states, use least privilege, narrow resource ownership, require plan approvals, and avoid broad automated destroys.

### CI/CD and advanced concepts

**21. Why store a saved Terraform plan?**  
To separate plan/review from apply and reduce the chance that apply recalculates a different set of operations.

**22. Why use modules?**  
To standardize and reuse infrastructure patterns, enforce naming/tagging policies, and reduce repeated code.

**23. `count` vs `for_each`?**  
`count` uses numeric indices; `for_each` uses stable keys and is usually better for sets of named resources.

**24. When do you use `depends_on`?**  
When a dependency exists but cannot be inferred from references. Prefer implicit dependencies when possible.

**25. How do you manage Terraform upgrades?**  
Pin versions, read release notes/upgrade guides, test plans in non-production, review breaking changes, and roll out deliberately.

**26. Terraform vs. Ansible?**  
Terraform primarily manages infrastructure lifecycles; Ansible often handles OS/application configuration and operational tasks.

**27. What are `moved` blocks?**  
Declarative state address refactors that help prevent unintended destroy/recreate when renaming compatible Terraform resources.

**28. How do you avoid exposing secrets in CI?**  
Use OIDC/managed identity, secret stores, masked inputs, minimal output, access-controlled state and plan artifacts, and short retention.

**29. What happens if the Azure Portal resource is deleted manually?**  
On refresh/plan, Terraform generally detects the missing object and plans to recreate it if it remains in configuration.

**30. How should production approvals work?**  
Branch-protected PR review for the configuration, a saved plan, and a separately enforced deployment approval/check before apply.

---

## 14. Practical projects and roadmap

### Project 1 — Azure network lab (beginner)

Create an RG, VNet, subnet, and NSG. Practice `init`, `validate`, `plan`, `apply`, `output`, and `destroy`. Verify in the Portal/CLI.

### Project 2 — Remote state and modules (intermediate)

Set up a dedicated Storage backend with Entra ID, then refactor the network into a reusable module. Deploy `dev` and `test` with separate state keys and inspect the outputs.

### Project 3 — Azure DevOps pipeline (intermediate)

Connect Azure Repos to `azure-pipelines.yml`. Use a workload identity service connection; run `fmt`, `validate`, and `plan` on PRs and gated `apply` on main.

### Project 4 — Containerized API on Azure (intermediate/advanced)

Build a small FastAPI image, push it to Azure Container Registry, provision hosting (such as an Azure Linux Web App with the appropriate plan), configure identity/secrets, and deploy from Azure DevOps. **App hosting, registry, and traffic may be billable.**

### Project 5 — AKS platform (advanced)

Use modules to create network, AKS, container registry, managed identities, Azure Monitor/Log Analytics, RBAC, and private access controls. Add Helm and GitOps. **AKS and associated networking/monitoring can incur significant costs; use a constrained sandbox and destroy after testing.**

### Suggested four-week learning plan

| Week | Objective | Deliverable |
|---|---|---|
| 1 | HCL, resources, variables, outputs, Terraform workflow | Working VNet lab |
| 2 | State, backend, locking, modules | Remote state + module refactor |
| 3 | Azure RBAC, OIDC, Azure DevOps pipeline | PR plan + gated apply |
| 4 | Drift, import, monitoring, IaC security, interview prep | Mini project + documented runbook |

---

## 15. Useful links and references

**Official Terraform documentation**

- [Terraform language documentation](https://developer.hashicorp.com/terraform/language)
- [AzureRM provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [AzureRM 5.0 upgrade guide](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/guides/5.0-upgrade-guide)
- [AzureRM backend and Azure AD/OIDC auth](https://developer.hashicorp.com/terraform/language/backend/azurerm)
- [Azure Terraform get-started tutorials](https://developer.hashicorp.com/terraform/tutorials/azure-get-started)
- [Azure DevOps Terraform provider](https://registry.terraform.io/providers/microsoft/azuredevops/latest/docs)

**Microsoft documentation**

- [Terraform on Azure](https://learn.microsoft.com/en-us/azure/developer/terraform/)
- [Store Terraform state in Azure Storage](https://learn.microsoft.com/en-us/azure/developer/terraform/store-state-in-azure-storage)
- [Azure DevOps Pipelines](https://learn.microsoft.com/en-us/azure/devops/pipelines/)
- [AzureCLI@2 task](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/azure-cli-v2)
- [Azure RBAC](https://learn.microsoft.com/en-us/azure/role-based-access-control/overview)
- [Azure pricing calculator](https://azure.microsoft.com/en-us/pricing/calculator/)

---

## Quick Revision Cheat Sheet

```bash
# 1. Log in and choose Azure subscription
az login
az account set --subscription "<SUB_ID>"

# 2. Standard IaC validation workflow
terraform init
terraform fmt -recursive
terraform fmt -check -recursive
terraform validate
terraform plan -out=tfplan

# 3. Apply reviewed changes
terraform apply tfplan

# 4. Inspect infrastructure/state
terraform output
terraform state list
terraform plan -refresh-only

# 5. Destroy only a disposable lab with the correct state
terraform destroy
```

**Remember for interviews:** *provider ≠ backend*, *state can contain secrets*, *Azure RBAC management permissions ≠ Blob data permissions*, *OIDC is preferred for automation*, and *every production apply should follow a reviewed plan and approval*.
