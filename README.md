# AWS to Azure Cloud Migration

Migrating a Windows Server 2022 workload from AWS EC2 into Azure using Azure Migrate — discovery, assessment, replication, and cutover, with all infrastructure defined in Terraform.

## Video walkthrough

📹 **[Watch the walkthrough](ADD_YOUR_LOOM_LINK_HERE)**

---

## What this project does

Companies move workloads between clouds all the time — after an acquisition, for cost, or for compliance. This project does exactly that: it takes a Windows Server running on AWS EC2 and migrates it into Azure.

Azure Migrate handles the process in four phases:

| Phase | What happens |
|---|---|
| **Discovery** | An appliance in Azure reads the EC2 instance's metadata using scoped AWS credentials |
| **Assessment** | Azure evaluates whether the workload can run in Azure, recommends a VM size, and estimates cost |
| **Replication** | Disk data is continuously copied from AWS to Azure while the source keeps running |
| **Cutover** | Azure builds the target VM from the replicated disk — minimal downtime |

---

## Architecture

```
AWS                                    Azure
─────────────────────────────          ────────────────────────────────────────

VPC  10.0.0.0/16                       rg-migrate-source  (staging)
└── Public subnet 10.0.1.0/24          ├── VNet 10.1.0.0/16
    │                                   │   └── Subnet 10.1.1.0/24
    └── EC2: Windows Server 2022  ─────▶├── Discovery appliance VM
        (source workload)               ├── Replication appliance VM
                                        ├── Storage account (replication cache)
IAM service user                        ├── Log Analytics workspace
└── Scoped policy:              ───────▶└── Recovery Services Vault
    read EC2 + snapshot                       (Azure Site Recovery)

                                       rg-migrate-target
                                       └── Migrated VM (after cutover)
```

**Two appliances, not one.** For AWS migrations Azure Migrate needs a *discovery* appliance for inventory and assessment, plus a separate *replication* appliance running Azure Site Recovery for the disk copying. "Agentless" means nothing is installed on the source EC2 instance — the appliances still run in Azure.

**No private link.** Traffic goes over the public internet, so discovery targets the EC2 instance's public IP. The two address ranges are deliberately different (10.0 vs 10.1) so peering could be added later without overlap.

---

## Repository layout

```
aws-side/       Terraform for the AWS source environment
azure-side/     Terraform for the Azure target environment
```

Two separate Terraform roots so each cloud can be destroyed independently.

### AWS side
VPC, public subnet, internet gateway, route table, security group, IAM role + policy + instance profile, IAM service user with scoped access keys, and a Windows Server 2022 EC2 instance.

### Azure side
Two resource groups, virtual network and subnet, replication cache storage account, Log Analytics workspace, Recovery Services Vault, network security groups, and both appliance VMs.

The Azure Migrate project itself is created manually in the portal — the Terraform AzureRM provider has no resource type for it.

---

## How to run it

**Prerequisites:** Terraform, AWS CLI, Azure CLI, an AWS account on the paid plan, and an Azure subscription with at least 32 vCPUs of quota per VM family.

```bash
# 1. AWS source environment
cd aws-side
cp terraform.tfvars.example terraform.tfvars   # fill in your values
terraform init
terraform apply

# 2. Azure target environment
cd ../azure-side
cp terraform.tfvars.example terraform.tfvars   # fill in your values
terraform init
terraform apply
```

Then in the Azure portal: create the Azure Migrate project → generate an appliance key → install the appliance on the discovery VM → add AWS and Windows credentials → run discovery → run an assessment → set up the replication appliance → enable replication → test migration → cut over.

### Teardown

```bash
cd aws-side && terraform destroy
cd ../azure-side && terraform destroy
az group delete --name rg-migrate-target-<yourname> --yes
```

---

## Results

| | |
|---|---|
| AWS source environment | ✅ Deployed |
| Azure target environment | ✅ Deployed |
| Discovery appliance | ✅ Registered, EC2 instance discovered |
| Assessment | ✅ **100% Azure readiness**, ~$39.60/month estimated |
| Replication appliance | ✅ Deployed, Site Recovery installed |
| Replication + cutover | ⛔ Blocked by subscription quota (see below) |

### The blocker

The Azure Site Recovery replication appliance requires **8 physical CPU cores**. Every VM size available on this subscription is v7-generation, and v7 hardware uses hyperthreading — 4 physical cores presented as 8 logical. The validator counts physical cores, sees 4, and blocks with the Continue button disabled.

Getting 8 physical cores means a 16-vCPU size. This subscription caps every VM family at 10 vCPUs, and the quota increase request was auto-denied — standard for a new pay-as-you-go account with no billing history.

This is a capacity planning finding, not a design problem. In a real engagement, verifying that the target subscription can host the migration tooling is something you do during planning, not on cutover weekend.

---

## Things worth knowing

Problems hit during this build that aren't in most documentation:

- **AWS reserves the `sg-` prefix** for system-generated security group IDs — you can't use it in a user-defined name.
- **Hardcoded AMI IDs go stale.** AWS deregisters Windows images as patches ship. Use a `data "aws_ami"` block instead.
- **AWS split the free tier in July 2025.** New Free Plan accounts can only launch free-tier-eligible instance types, which excludes anything big enough to run Windows Server.
- **Azure Migrate needs WinRM.** Port 5985 must be open on the source security group, and `winrm quickconfig -force` has to run on the instance — otherwise discovery finds the machine but reads nothing about the OS.
- **Recovery Services Vault soft delete can't be disabled** on new vaults. Plan your teardown around it instead of trying to turn it off.
- **SKU availability is per-subscription, per-region, and per-generation.** `az vm list-skus --location <region> --all | grep -v NotAvailable` tells you what you can actually deploy in ten seconds.
- **v7-generation VM sizes require Generation 2 images** (`2022-datacenter-g2`) and premium-capable disks.
- **Azure security defaults block the appliance.** Azure Migrate authenticates as the Azure PowerShell application, and MFA enforcement blocks it with error `AADSTS530035`. Security defaults don't appear in the Conditional Access policy list — check Entra ID → Properties. The right fix is a scoped app exclusion, not disabling the protection.
- **The replication appliance needs 600 GB free**, not the 127 GB commonly documented. And Azure won't expand a disk on a running VM, nor does Windows auto-extend the partition afterward.

---

## Security notes

This is a lab, and some choices reflect that:

- RDP is open to `0.0.0.0/0` — scope it to a specific source IP or use a bastion in production.
- The IAM snapshot policy uses `Resource: "*"` — scope it with resource tags in production.
- Static AWS access keys are used because Azure Migrate can't assume a role across clouds. They live in Terraform state in cleartext, so the state file is gitignored and the service user should be deleted at teardown.
- Azure security defaults were temporarily disabled to complete appliance registration, and re-enabled afterward.

`terraform.tfvars` and `terraform.tfstate` are gitignored — they contain admin passwords and the AWS secret key. Copy the `.example` files and fill in your own values.
