# AWS to Azure Cloud Migration — EC2 to Azure VM Using Azure Migrate

## 🎬 Video Walkthrough

**[Watch the full walkthrough](https://www.loom.com/share/4946195db2a5416f9a87ad0d1d98d204)**

---

## 📖 Project Overview

This project is an end-to-end cloud-to-cloud migration: a Windows Server 2022 workload running on AWS EC2 is discovered, assessed, and prepared for replication into Azure using **Azure Migrate**, Microsoft's native migration service.

Cross-cloud migrations are among the highest-value engagements in cloud engineering. Organizations move workloads between providers for cost, for compliance, to consolidate after an acquisition, or because a business unit standardized on a different platform. The work is rarely about the tooling — it's about planning, dependency mapping, and knowing what breaks before it breaks.

Infrastructure on both sides is provisioned with **Terraform**, split into two independent roots (`aws-side` and `azure-side`) so each cloud can be destroyed without touching the other. The migration itself — appliance registration, credential entry, discovery, assessment — is portal-driven, because Azure Migrate's appliances require interactive registration that Terraform cannot perform.

### What "agentless" actually means here

Nothing is installed on the EC2 source machine. But unlike VMware agentless migrations, **AWS migrations still require two dedicated appliance VMs running in Azure**:

- A **discovery appliance** — handles inventory, OS-level detail collection, and assessment data
- A **replication appliance** (Configuration Server) — handles disk-level replication, running Azure Site Recovery underneath

Both are mandatory regardless of whether agents run on the source. This is the single most misunderstood part of the architecture.

### How this project ended

Discovery and assessment completed cleanly. The EC2 instance was found from Azure, evaluated, right-sized, and costed — **100% Azure readiness, $39.60/month projected**.

Replication hit a hard subscription constraint. Azure Site Recovery's replication appliance validates against **physical** CPU cores, not vCPUs. Every VM generation available to this subscription is hyperthreaded, so satisfying an 8-physical-core requirement means provisioning a 16-vCPU size — and this subscription caps every VM family at 10 vCPUs. The quota increase request was auto-denied.

That outcome is documented in full rather than hidden, because insufficient quota is one of the most common real blockers in enterprise migration work. See [How This Project Ended](#-how-this-project-ended) for the full diagnosis and [Troubleshooting](#️-troubleshooting) for every individual issue and fix.

---

## 🧠 Skills Demonstrated

- Cross-cloud migration planning and execution (AWS → Azure)
- Dual Terraform root design — independently deployable and destroyable stacks per cloud
- AWS networking fundamentals: VPC, public subnet, internet gateway, route tables — and how each maps to its Azure equivalent
- **Least-privilege IAM design** for a third-party service, including deliberately rejecting `IAMFullAccess` in favor of a scoped inline policy to avoid a privilege-escalation path
- Azure Migrate architecture: why AWS migrations need two appliances, and what each one does
- Azure Site Recovery concepts underlying agentless replication — replication cache, Recovery Services Vault
- Migration assessment interpretation: Azure readiness, performance-based right-sizing, cost projection
- Root-causing an opaque Entra ID error (`AADSTS530035`) to a tenant-level Security Defaults policy using raw sign-in diagnostics rather than guesswork
- Diagnosing SKU availability constraints that are per-subscription, per-region, **and per-generation**
- Terraform state recovery via `terraform import` after a partial apply left resources orphaned
- Working through Azure vCPU quota at both the regional and per-family level
- Recognizing product and documentation drift, and verifying against current vendor docs instead of trial-and-error
- Cross-cloud secrets handling — AWS IAM access keys passed into an Azure-hosted appliance
- Multi-cloud teardown with correct dependency ordering

---

## 🏗️ Architecture

```
AWS Account — Source                    Azure Subscription — Target
─────────────────────────────           ────────────────────────────────────────

Internet Gateway                        Azure Migrate Project
    │                                        │
VPC 10.0.0.0/16                         rg-migrate-source (staging)
└── Public Subnet 10.0.1.0/24           ├── VNet 10.1.0.0/16
    └── Security Group                  │   └── snet-migrate 10.1.1.0/24
        (443 / 3389 / 5985)             │       ├── Discovery Appliance VM
        └── EC2 Windows Server 2022 ◄───┤       └── Replication Appliance VM
            (source workload)           ├── Storage Account (replication cache)
                                        ├── Log Analytics Workspace
IAM Service User ───────────────────────┤   └── Recovery Services Vault
└── Scoped policy:                      │       (Azure Site Recovery)
    ec2:Describe*, CreateSnapshot,      │
    DeleteSnapshot                      rg-migrate-target
                                        ├── Network Security Group
                                        └── Migrated VM (created at cutover)
```

### How the pieces connect

**Discovery.** The discovery appliance in Azure authenticates to AWS using a dedicated, least-privilege IAM access key and reads EC2 instance metadata over the public internet. No agent runs on the EC2 instance. A second credential set — a Windows administrator account over WinRM — collects OS-level detail that the EC2 API doesn't expose.

**Assessment.** Azure Migrate evaluates the discovered machine against Azure's supported configurations, recommends a VM size based on observed CPU and memory utilization, and projects monthly cost.

**Replication.** The replication appliance snapshots the source disk, streams the data into the replication cache storage account, and then continuously syncs deltas. The Recovery Services Vault orchestrates this — Azure Site Recovery is the engine underneath.

**Cutover.** Azure builds the target VM from the most recent replicated disk state. Downtime is measured in minutes, not hours, because the data is already there.

### Why there's no private link

There is no VPN or peering between the AWS VPC and the Azure VNet in this design. All cross-cloud traffic goes over the public internet, which is why the EC2 instance needs a public IP and an internet gateway, and why discovery targets that public address.

The two address ranges (`10.0.0.0/16` and `10.1.0.0/16`) are deliberately non-overlapping so a site-to-site VPN or peering could be added later without renumbering either side. In a production migration you would build that VPN first and keep the source instance private.

### AWS and Azure network models are not the same

Worth understanding before reading the Terraform, because the two clouds solve this differently:

| | AWS | Azure |
|---|---|---|
| Default state | **Closed.** No inbound or outbound until you attach an internet gateway and add a route | **Outbound open by default** via a system route |
| Outbound path | Internet gateway + `0.0.0.0/0` route in the route table | System route, or NAT Gateway for explicit control |
| Inbound path | Internet gateway + route + public IP on the instance | Public IP on the NIC + an NSG rule. That's it |
| "Public" vs "private" subnet | **Real distinction** — created by whether the route table points at the IGW | Exists, but means something different: a "private subnet" (`defaultOutboundAccess = false`) only disables implicit **outbound**. Inbound is never a subnet property |
| Instance firewall | Security Group | Network Security Group |

The practical consequence: in AWS you make a machine private by putting it in a subnet with no IGW route. In Azure you make a machine private by not giving it a public IP — the subnet has nothing to do with it.

---

## ✅ Prerequisites

- **AWS account with programmatic access.** Must be on the **paid plan**, not the Free Plan. AWS split the free tier in July 2025; Free Plan accounts can only launch free-tier-eligible instance types, which excludes anything capable of running Windows Server.
- **Azure subscription** with at least **32 vCPUs of quota in a single VM family** in your target region. This is the constraint that stopped this project. Check before you start:
  ```bash
  az vm list-usage --location eastus --output table | grep -i "Family vCPUs"
  ```
- **Terraform** — `brew install hashicorp/tap/terraform` on macOS
- **AWS CLI** — configured with `aws configure`, verified with `aws sts get-caller-identity`
- **Azure CLI** — authenticated with `az login`, verified with `az account show`
- **Remote Desktop client** — Microsoft Remote Desktop (Mac App Store) or `mstsc` on Windows
- **Budget awareness.** Appliance VMs at the required spec run roughly **$1–1.80/hour** with Windows licensing. Deallocate when you step away; destroy when you're done.
- **8–10 hours**, not the 4–6 a clean run would suggest. Budget for diagnosis, not just execution.

---

## 🏷️ Naming Conventions

| Resource | Value |
|---|---|
| AWS region | `us-east-1` |
| AWS IAM user (Terraform) | `terraform-migrate-lab` |
| AWS VPC | `vpc-migrate-fabrizio` (10.0.0.0/16) |
| AWS subnet | `snet-migrate-fabrizio` (10.0.1.0/24) |
| AWS security group | `migrate-source-sg-fabrizio` — renamed from `sg-migrate-source-…`; AWS reserves the `sg-` prefix |
| AWS EC2 instance | `ec2-migrate-source-fabrizio` (t3.medium, 30 GB gp3) |
| AWS IAM role | `role-azure-migrate-fabrizio` |
| AWS IAM service user | `svc-azure-migrate-fabrizio` |
| Azure region | `East US` |
| Azure source resource group | `rg-migrate-source-fabrizio` |
| Azure target resource group | `rg-migrate-target-fabrizio` |
| Azure VNet | `vnet-migrate-fabrizio` (10.1.0.0/16) |
| Azure subnet | `snet-migrate` (10.1.1.0/24) |
| Azure Migrate project | `migrate-project-fabrizio` |
| Replication cache storage | `stmigratefabrizio` |
| Recovery Services Vault | `rsv-migrate-fabrizio` |
| Log Analytics workspace | `law-migrate-fabrizio` |
| Discovery appliance (portal name) | `appl-mig-fab` — 14-character limit |
| Discovery appliance VM | `vm-mig-appl-fabrizio`, computer name `appl-fabrizio` |
| Discovery appliance size | `Standard_D8s_v7` |
| Replication appliance VM | `vm-mig-repl-fabrizio`, computer name `repl-fabrizio` |
| Replication appliance size | `Standard_D16s_v7` required — blocked by quota |

---

## 🪜 Project Steps

All Terraform lives in `aws-side/` and `azure-side/`. Copy `terraform.tfvars.example` to `terraform.tfvars` in each and fill in real values. **Never commit `terraform.tfvars`** — it holds three admin passwords and is gitignored.

The steps below show the **working final configuration**. Every error and dead end encountered getting there lives in [Troubleshooting](#️-troubleshooting), grouped by category rather than interleaved here.

---

### Part 0 — Local Setup

#### Step 0a: Create the AWS IAM user for Terraform

In the AWS Console: **IAM → Users → Create user**, name it `terraform-migrate-lab`, attach `AmazonEC2FullAccess` and `AmazonVPCFullAccess`, and create access keys.

**These two policies are not enough.** The Terraform in this repo also creates IAM roles, policies, users, and instance profiles, and neither policy covers those. You will get `AccessDenied` on every IAM resource.

The obvious fix is attaching `IAMFullAccess`. **Don't.** That gives your Terraform user the ability to create an administrator role in your own account — a textbook privilege-escalation path. Instead, attach a scoped inline policy limited to the four resource name patterns this project actually creates:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ManageMigrateLabIamRoles",
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole", "iam:GetRole", "iam:DeleteRole",
        "iam:TagRole", "iam:UntagRole", "iam:ListRoleTags",
        "iam:ListRolePolicies", "iam:ListAttachedRolePolicies",
        "iam:ListInstanceProfilesForRole",
        "iam:AttachRolePolicy", "iam:DetachRolePolicy", "iam:PassRole"
      ],
      "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/role-azure-migrate-*"
    },
    {
      "Sid": "ManageMigrateLabInstanceProfiles",
      "Effect": "Allow",
      "Action": [
        "iam:CreateInstanceProfile", "iam:GetInstanceProfile",
        "iam:DeleteInstanceProfile", "iam:AddRoleToInstanceProfile",
        "iam:RemoveRoleFromInstanceProfile", "iam:TagInstanceProfile"
      ],
      "Resource": "arn:aws:iam::<ACCOUNT_ID>:instance-profile/profile-azure-migrate-*"
    },
    {
      "Sid": "ManageMigrateLabPolicies",
      "Effect": "Allow",
      "Action": [
        "iam:CreatePolicy", "iam:GetPolicy", "iam:DeletePolicy",
        "iam:GetPolicyVersion", "iam:ListPolicyVersions",
        "iam:DeletePolicyVersion", "iam:TagPolicy"
      ],
      "Resource": "arn:aws:iam::<ACCOUNT_ID>:policy/policy-azure-migrate-*"
    },
    {
      "Sid": "ManageMigrateLabServiceUser",
      "Effect": "Allow",
      "Action": [
        "iam:CreateUser", "iam:GetUser", "iam:DeleteUser",
        "iam:TagUser", "iam:UntagUser", "iam:ListUserTags",
        "iam:ListUserPolicies", "iam:ListAttachedUserPolicies",
        "iam:ListGroupsForUser", "iam:AttachUserPolicy",
        "iam:DetachUserPolicy", "iam:CreateAccessKey",
        "iam:DeleteAccessKey", "iam:ListAccessKeys"
      ],
      "Resource": "arn:aws:iam::<ACCOUNT_ID>:user/svc-azure-migrate-*"
    }
  ]
}
```

#### Step 0b: Configure the CLIs

```bash
# Terraform
brew tap hashicorp/tap && brew install hashicorp/tap/terraform

# AWS CLI
brew install awscli
aws configure   # region: us-east-1

# Azure CLI
brew install azure-cli
az login
```

Verify both before continuing:

```bash
aws sts get-caller-identity
az account show
```

**Watch the AWS CLI region.** Terraform reads its region from `var.aws_region`; the CLI reads yours from the profile. If they differ, every manual `aws ec2` verification command silently returns empty and it looks like nothing deployed. Check with `aws configure get region`.

---

### Part 1 — AWS Source Environment

The AWS side provisions a VPC, a public subnet with internet gateway and route table, a security group, an IAM role and instance profile, a dedicated IAM service user with access keys for cross-cloud auth, and the Windows Server 2022 EC2 instance.

#### Key configuration decisions

**Security group name.** Uses `migrate-source-sg-fabrizio`, not `sg-migrate-source-fabrizio`. AWS reserves the `sg-` prefix for system-generated security group IDs and rejects it outright in a user-defined name.

**Three ingress rules, not two:**

| Port | Purpose |
|---|---|
| 443 | Azure Migrate appliance control channel |
| 3389 | RDP, for manual verification |
| **5985** | **WinRM HTTP — required for OS-level discovery** |

Port 5985 is easy to miss. Azure Migrate uses Windows Remote Management to read OS detail the EC2 API doesn't expose, and the setup flow tells you to disable the HTTPS toggle so it falls back to HTTP on 5985. Without that port open, discovery finds the instance but returns no OS information.

The instance's `user_data` also runs `winrm quickconfig -force`, because Windows Server has WinRM running by default but not configured to accept remote connections.

**AMI lookup, not a hardcoded ID.** AWS deregisters Windows AMIs as patched versions ship, so any hardcoded AMI goes stale within months. This uses a data source resolved at plan time:

```hcl
data "aws_ami" "windows_2022" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["Windows_Server-2022-English-Full-Base-*"]
  }

  filter {
    name   = "state"
    values = ["available"]
  }
}
```

Trade-off worth naming: `most_recent = true` means the AMI can change between applies and Terraform will want to replace the instance. Correct for a lab, wrong for production — there you resolve the ID once and update it deliberately.

**IAM scoped to what Azure Migrate actually needs** — read EC2 metadata plus create and delete snapshots, since agentless replication works by snapshotting the source disk, copying, and deleting the snapshot.

#### Deploy

```bash
cd aws-side
cp terraform.tfvars.example terraform.tfvars   # fill in your values
terraform init
terraform validate
terraform apply
```

Expect **14 resources**. The Windows Administrator password must be 12+ characters with uppercase, lowercase, digits, and symbols — AWS rejects weak passwords.

#### Save the outputs

These feed directly into Part 3:

```bash
terraform output ec2_public_ip
terraform output ec2_instance_id
terraform output migrate_access_key_id
terraform output -raw migrate_secret_access_key
```

#### Verify via RDP

Connect with Microsoft Remote Desktop — username `Administrator`, password from your `terraform.tfvars`. **Wait a full 5 minutes after apply completes**; Windows runs `user_data` on first boot and the password isn't set until that finishes.

Run `ipconfig` on the instance and you'll see `10.0.1.x` — the private address. The instance has no idea it has a public IP; the internet gateway does 1:1 NAT on the way in and out.

---

### Part 2 — Azure Target Environment

Provisions two resource groups, the VNet and subnet, replication cache storage, Log Analytics workspace, Recovery Services Vault, and NSGs.

#### Key configuration decisions

**Two resource groups, not one.** `rg-migrate-source` holds the migration tooling; `rg-migrate-target` holds the migrated VM. Separating them means you can delete the tooling after cutover without touching what you just migrated.

**`soft_delete_enabled = true` on the vault.** Many older guides set this to `false` so the vault deletes cleanly at teardown. Azure no longer accepts that — soft delete is now permanently on for new Recovery Services Vaults, a deliberate ransomware protection. The correct response is to adjust your teardown process, not to fight the control.

**Storage account must be Standard / LRS / StorageV2.** Azure Migrate validates these values when you configure replication and fails on anything else. Locally redundant is correct here — this is temporary staging, and if the cache is lost mid-replication you simply restart.

**The `null` provider must be declared.** The Azure Migrate project has no Terraform resource type at all, so it's created manually in the portal. A `null_resource` documents that gap in code:

```hcl
terraform {
  required_providers {
    azurerm = { source = "hashicorp/azurerm", version = "~> 3.0" }
    null    = { source = "hashicorp/null",    version = "~> 3.0" }
  }
}
```

#### Deploy

```bash
cd ../azure-side
cp terraform.tfvars.example terraform.tfvars
terraform init
terraform apply
```

Expect **9 resources**.

---

### Part 3 — Discovery Appliance

#### Step 1: Create the Migrate project and generate a key

Azure portal → **Azure Migrate** → **Start discovery** → **Using appliance** → **For Azure**

- **Are your servers virtualized?** → *Physical or other (AWS, GCP, Xen, etc.)*
- **Project name:** `migrate-project-fabrizio`, resource group `rg-migrate-source-fabrizio`, geography United States
- **Appliance name:** 14 characters maximum. `appl-mig-fab` works; `appliance-migrate-fabrizio` is rejected
- Click **Generate key** and save it immediately — you'll paste it inside an RDP session later

> The portal UI has been substantially reorganized. Older guides reference "Servers, databases and web apps" and "Migration and modernization" — those tiles no longer exist. **Start discovery** and **Execute → Migrations** are the current equivalents.

#### Step 2: Deploy the appliance VM

Microsoft's current requirement is **8 vCPUs, 16 GB RAM, 80 GB storage**. Older guides specifying `Standard_A4_v2` (4 vCPU / 8 GB) will fail the appliance's own prerequisite check.

```hcl
resource "azurerm_windows_virtual_machine" "appliance" {
  name                  = "vm-mig-appl-${var.yourname}"
  computer_name         = "appl-${var.yourname}"   # 15-char NetBIOS limit
  size                  = "Standard_D8s_v7"
  admin_username        = "migrateadmin"
  admin_password        = var.appliance_admin_password
  # ...

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"           # v7 requires premium-capable
    disk_size_gb         = 128
  }

  source_image_reference {
    publisher = "MicrosoftWindowsServer"
    offer     = "WindowsServer"
    sku       = "2022-datacenter-g2"               # v7 requires Generation 2
    version   = "latest"
  }
}
```

Three non-obvious requirements are encoded there: a short `computer_name`, a Generation 2 image SKU, and a premium disk. All three produced separate errors before landing on this configuration.

#### Step 3: Install and register

1. RDP in with `migrateadmin` and your appliance password
2. Turn off IE Enhanced Security Configuration (Server Manager → Local Server) or Edge will block the portal
3. **Download the installer on the VM itself**, not on your Mac — it's ~500 MB and pushing it over RDP is slower than a fresh download
4. **Extract the zip before running anything.** Scripts run from inside a mounted zip fail silently
5. Run the installer from an already-open **Administrator** PowerShell window:

   ```powershell
   cd C:\Users\migrateadmin\Downloads\AzureMigrateInstaller
   Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
   .\AzureMigrateInstaller.ps1
   ```

   Right-click → *Run with PowerShell* closes the window instantly on any error, hiding the message. Always run from an open window.

6. Answer the prompts: `Y` (execution policy) → `3` (Physical or other) → `1` (Azure Public) → `1` (Public endpoint) → `Y`
7. When asked whether to remove Internet Explorer, answer `N` if Edge is already installed — `Y` forces an immediate reboot and drops your session for no benefit
8. The configuration manager should open automatically. If it doesn't, browse to `https://<computer-name>:44368` and click through the self-signed certificate warning
9. Run **Set up prerequisites**, paste the project key, click **Login**

> If sign-in fails with "You don't have access to this" despite being Global Administrator and subscription Owner, see [Authentication issues](#authentication-issues). It is a tenant policy, not a permissions problem.

#### Step 4: Add credentials and start discovery

Two credential sets are required. Friendly names accept letters and digits only — no hyphens.

| | Credential 1 | Credential 2 |
|---|---|---|
| **Source type** | Windows Server | Windows Server |
| **Friendly name** | `awsmigratesvc` | `ec2winadmin` |
| **Username** | your `migrate_access_key_id` | `Administrator` |
| **Password** | your `migrate_secret_access_key` | your EC2 `admin_password` |

Then:

1. Turn the **HTTPS slider off** — this permits WinRM's HTTP fallback on port 5985
2. **Add discovery source** with the EC2 instance's **public** IP (there is no private link between the clouds)
3. Map it to `ec2winadmin`
4. **Save** → **Revalidate** → wait for *Validation successful*
5. **Start discovery** (5–15 minutes)

---

### Part 4 — Assessment

Azure Migrate → your project → **Create assessment**

- **Type:** Azure VM
- **Target location:** East US
- **Storage type:** Automatic
- **Sizing criteria:** Performance-based — uses the CPU and memory data the appliance actually collected, rather than just matching the source instance's specs
- **Reserved instances:** None
- **Add workloads** → select the discovered EC2 instance

#### Results

| Metric | Value |
|---|---|
| Azure readiness | **100% — Ready for Azure** |
| Inventory covered | 100% |
| Estimated monthly cost | **$39.60** |

**Why this phase matters most commercially.** Readiness is what catches blockers before you schedule a cutover window — unsupported OS versions, boot type mismatches, disk configurations Azure won't accept. "Ready with conditions" or "Not ready" tells you exactly what to remediate first. And the cost projection is the number a business actually approves the migration on.

A migration project that stalls after assessment has still delivered something real: a validated inventory, a readiness determination, a right-sized target, and a budget.

---

### Part 5 — Replication Appliance

This is a **second, separate VM**. It runs Azure Site Recovery's Configuration Server and handles disk-level replication — entirely distinct from the discovery appliance.

Portal path: **Execute → Migrations → Start execution**

- What do you want to migrate? → *Servers or Virtual machines (VM)*
- Where to? → *Azure VM*
- How will you select workloads? → *From replication appliance (Physical or others)*
- Click **"Click here to set up"** → target region East US → **Generate key**

#### Requirements, which are higher than most guides state

| Requirement | Value | Notes |
|---|---|---|
| CPU | **8 physical cores** | Not 8 vCPUs — see below |
| RAM | 16 GB+ | |
| **Free disk space** | **600 GB** | Commonly documented as 127 GB. The installer aborts immediately below 600 |
| OS | Windows Server 2022 | 2019 fails during install |

#### Installing

```powershell
cd C:\Users\replicationadmin\Downloads\DRAppliance\DRAppliance
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
.\DRInstaller.ps1
```

Note the doubled folder — the zip extracts `DRAppliance` inside `DRAppliance`.

**Before running the installer, extend the partition.** Resizing the Azure managed disk does not resize the NTFS partition inside Windows, and Azure won't expand a disk on a running VM at all. The full sequence is deallocate → resize → start → then:

```powershell
$max = (Get-PartitionSupportedSize -DriveLetter C).SizeMax
Resize-Partition -DriveLetter C -Size $max
Get-PSDrive C | Select-Object Free
```

The installer runs about 15 minutes — IIS, the URL Rewrite module, all the Site Recovery agents, Process Server, and Configuration Manager. It completes with `Installation completed successfully` and the configuration manager at `https://<computer-name>:44368`.

**And then the prerequisite check fails.** See below.

---

### Part 6 — Test Migration and Cutover (Not Reached)

Neither was reached, since replication never started. The full sequence is documented in [How This Project Ended](#-how-this-project-ended) for anyone completing this with sufficient quota.

---

## 🛑 How This Project Ended

Discovery and assessment completed cleanly. Replication hit a hard subscription constraint, diagnosed as follows.

### The chain

**1. The installer's prerequisite check failed** with *"The server has only 4 CPU cores. We recommend 8"* — on a VM provisioned with 8 vCPUs.

**2. Windows confirmed the discrepancy:**

```powershell
Get-CimInstance Win32_Processor | Select-Object NumberOfCores, NumberOfLogicalProcessors
# NumberOfCores: 4    NumberOfLogicalProcessors: 8
```

The v7 VM generation uses simultaneous multithreading — 4 physical cores presenting as 8 logical processors. **The Site Recovery validator counts physical cores.** Despite the word "recommend," the Continue button stays disabled. It's a hard block.

**3. Switching CPU vendors didn't help.** Moved from `Standard_D8as_v7` (AMD) to `Standard_D8ds_v7` (Intel) — same 4/8 split. The entire v7 generation is hyperthreaded.

**4. Satisfying 8 physical cores means a 16-vCPU size** — `Standard_D16s_v7` or equivalent.

**5. Quota blocked it:**

```
Standard Dsv7 Family vCPUs     8     10
Standard Ddsv7 Family vCPUs    8     10
Standard Dadsv7 Family vCPUs   0     10
```

Every family caps at **10 vCPUs**. A 16-vCPU VM does not fit in any of them. The earlier *regional* quota increase to 32 had been approved, but regional and per-family quotas are separate ceilings.

**6. The family quota increase was auto-denied.** Requested 32 for `Standard Dsv7 Family vCPUs` in East US. Rejected without review — standard for a pay-as-you-go subscription with no billing history.

**7. Alternatives were exhausted:**
- **East US 2** — same 10-vCPU cap on every family
- **Older generations** (`D8_v3`, `D8s_v3`, `E8s_v3`, `F8s_v2`) that might expose 8 physical cores at 8 vCPUs — **none available to this subscription**. SKU availability turned out to be gated per-generation, not just per-region

Getting past this requires a formal Azure support ticket with a 24–48 hour turnaround.

### Why this is worth documenting

Insufficient quota is one of the most common blockers in real enterprise migrations. Large organizations routinely file quota increase tickets weeks before a migration window opens, precisely because approval isn't instant.

Verifying that **the target subscription can host the migration tooling** is a planning-phase question. Discovering it during execution is exactly how a migration weekend gets derailed. Hitting it personally, and diagnosing it down to the physical-versus-logical core distinction in the installer's own validation logic, is a more honest demonstration of migration engineering than an uninterrupted walkthrough would be.

### What completing this looks like from here

**1. Resolve the quota.** File an Azure support request for 32 vCPUs in the `Dsv7` family in the target region. Deploy `Standard_D16s_v7`.

**2. Finish appliance registration.** Prerequisites go green, select connectivity, register against the Recovery Services Vault and subscription. About 15 minutes to show as connected.

**3. Enable replication.** Portal → Migrations → Replicate:
- Target resource group: `rg-migrate-target-fabrizio`
- Cache storage account: `stmigratefabrizio`
- VNet: `vnet-migrate-fabrizio`, subnet `snet-migrate`
- Compute: accept the assessment's recommendation
- Tags: `project = azure-migrate-lab`

**4. Monitor initial replication.** The first full disk copy takes 20–60 minutes; a 30 GB Windows disk typically 30–45. Status moves from *Initial replication in progress* to **Protected** once the appliance is keeping up with delta syncs. From then on the target stays current while the source keeps serving traffic — that's what makes near-zero-downtime cutover possible.

**5. Run a test migration.** Never skip this. It builds a temporary copy of the VM in an isolated network so you can verify it boots, the hostname and OS match, and workload-specific things came across. RDP in, confirm, then **Clean up test migration**. This doesn't disturb ongoing replication. Cutting over untested is how migration weekends go wrong.

**6. Cut over.** **Shut down the source first** in any real migration — otherwise writes land on both copies between the final sync and cutover, and the data diverges. That's split-brain. Then **Migrate**; Azure runs a final sync and builds the target VM (5–10 minutes).

**7. Verify and re-point networking.** Attach a public IP, RDP in with the original credentials, confirm hostname and OS version match the source. **The IP address changes** — you moved between clouds. Update DNS records, load balancer backends, and any hardcoded references. In a real cutover the DNS TTL is lowered days in advance so the change propagates quickly.

**8. Decommission the source.** Keep the EC2 instance stopped as a rollback option through the validation period, then terminate it and tear down the AWS-side infrastructure.

---

## 🛠️ Troubleshooting

Every issue actually encountered, grouped by category.

### AWS issues

| Issue | Cause | Resolution |
|---|---|---|
| `terraform apply` fails: *invalid value for name (cannot begin with sg-)* | AWS reserves the `sg-` prefix for system-generated security group IDs | Rename to `migrate-source-sg-<name>` |
| `AccessDenied` on every IAM resource — role, policy, user | The Terraform user only had EC2 and VPC permissions | Attach the scoped inline IAM policy from Step 0a. Avoid `IAMFullAccess` — it creates a privilege-escalation path |
| `collecting instance settings: empty result` | The hardcoded AMI ID no longer exists; AWS deregisters Windows images as patches ship | Replace with a `data "aws_ami"` block that resolves at plan time |
| `InvalidParameterCombination: The specified instance type is not eligible for Free Tier` | AWS split the free tier in July 2025. Free Plan accounts can only launch free-tier-eligible types | Upgrade to the paid plan. Promotional credits carry over and are consumed first |
| `aws ec2` commands return empty even though resources exist | CLI region (`us-east-2`) didn't match Terraform's region (`us-east-1`) | Pass `--region us-east-1` explicitly, or fix the CLI profile. Terraform carries its own region and is unaffected |

### Azure infrastructure issues

| Issue | Cause | Resolution |
|---|---|---|
| `BMSUserErrorDisablingSoftDeleteStateNotAllowed` on the Recovery Services Vault | Azure made soft delete permanently on for new vaults as a ransomware control; `false` is no longer a legal value | Set `soft_delete_enabled = true` and adjust the teardown process instead |
| Re-running apply: *a resource with this ID already exists — needs to be imported* | The failed apply created the vault in Azure before erroring, so it exists in Azure but not in Terraform state | `terraform import azurerm_recovery_services_vault.main "/subscriptions/<id>/resourceGroups/<rg>/providers/Microsoft.RecoveryServices/vaults/<vault>"`, then apply again |
| `provider does not support resource type azurerm_migrate_project` | Azure Migrate projects have no Terraform resource type at all | Create the project manually in the portal; document the gap with a `null_resource` |
| `Unexpected "output" block` in `terraform.tfvars` | An output block was pasted into the wrong file | `terraform.tfvars` holds only `key = value` pairs. Blocks go in `outputs.tf` |

### VM sizing and availability issues

| Issue | Cause | Resolution |
|---|---|---|
| `computer_name can be at most 15 characters, got 20` | Windows NetBIOS names cap at 15; Terraform defaults `computer_name` to the resource name | Set an explicit short `computer_name` (`appl-fabrizio`, `repl-fabrizio`) |
| `SkuNotAvailable` for `D8s_v3`, `D8s_v5`, `D8as_v5`, `D8_v4`, `B4ms` — every size tried | SKU availability is per-subscription, per-region, **and per-generation**. This subscription only had **v7** enabled; v3, v4, and v5 were all blocked | Query it rather than guessing: `az vm list-skus --location eastus --resource-type virtualMachines --all -o table \| grep -vi NotAvailable` |
| `cannot boot Hypervisor Generation '1'` | v7 sizes only support Generation 2 images; the default Windows SKU string is Gen 1 | Use `sku = "2022-datacenter-g2"` |
| `exceeding approved Total Regional Cores quota` | Default regional cap was 10 | Portal → Subscriptions → Usage + quotas → request an increase. Raised to 32, approved |
| `exceeding approved StandardDsv7Family Cores quota` **after** raising the regional quota | Regional and per-family quotas are **separate ceilings**. Regional was 32; the family was still 10 | Switch to a different family (each has its own bucket) or request a family-level increase |

### Replication appliance issues

| Issue | Cause | Resolution |
|---|---|---|
| `Aborting the installation, as at least 600 GB free space is required` | The real requirement is 600 GB, not the 127 GB commonly documented | Resize the OS disk to 700 GB |
| Disk resized in Azure but Windows still reports the old free space | Azure won't expand a disk on a running VM, and Windows doesn't auto-extend the NTFS partition | Deallocate → `terraform apply` → start → then `Resize-Partition -DriveLetter C -Size (Get-PartitionSupportedSize -DriveLetter C).SizeMax` |
| `DRInstaller.ps1` opens and closes instantly | Right-click → *Run with PowerShell* closes the window on any error, hiding the message | Run from an already-open Administrator PowerShell window so errors stay visible |
| `The term '.\DRInstaller.ps1' is not recognized` | The zip extracted a folder inside a folder | `cd DRAppliance` one level deeper. Confirm with `Get-ChildItem -Recurse -Filter *.ps1` |
| *Memory and CPU validation failed — the server has only 4 CPU cores* on an 8-vCPU VM | The validator counts **physical** cores. v7 hardware is hyperthreaded: 4 physical, 8 logical | Requires a 16-vCPU size. See [How This Project Ended](#-how-this-project-ended) |

### Authentication issues

| Issue | Cause | Resolution |
|---|---|---|
| Appliance registration: *"Your sign-in was successful but you don't have permission to access this resource"* — despite Global Administrator **and** subscription Owner | Not a permissions problem. Entra sign-in logs showed error **`AADSTS530035` — "Access has been blocked by security defaults."** Security defaults enforce MFA for admin roles, and the appliance authenticated with a password only | Entra ID → **Properties** → **Manage security defaults** → Disabled. Re-enable after the project. The production-correct fix is a scoped Conditional Access exclusion for the Azure PowerShell app, not disabling the protection |
| `az role assignment list --assignee <email>` returns *Cannot find user in graph database* | The account is a **guest** (`#EXT#`) in the tenant; guest UPNs are mangled (`user_gmail.com#EXT#@tenant.onmicrosoft.com`) | Resolve the real identity with `az ad signed-in-user show --query id -o tsv` and query by object ID |
| Security defaults don't appear in the Conditional Access policy list | Security defaults are a separate Microsoft-managed baseline, not a CA policy | Check Entra ID → Properties. The **sign-in logs** name the exact cause; don't diagnose by clicking around |

**The lesson from this one:** Azure Migrate's appliance authenticates as the **Microsoft Azure PowerShell** application (`1950a258-227b-4e31-a9cf-717495945fc2`). Any tenant-level MFA enforcement, Conditional Access policy, or device-compliance requirement will block migration tooling. In an enterprise tenant this is a planning-phase conversation with the identity team.

### Portal and product-drift issues

| Issue | Cause | Resolution |
|---|---|---|
| "Servers, databases and web apps" and "Migration and modernization" tiles don't exist | The Azure Migrate portal was substantially reorganized | **Start discovery → Using appliance → For Azure** replaces the first; **Execute → Migrations** replaces the second |
| Appliance name rejected | 14-character limit not stated in most guides | `appl-mig-fab`, not `appliance-migrate-fabrizio` |
| `${var.yourname}` rejected in the portal project name field | Terraform interpolation syntax pasted literally into a portal form | Substitute the value by hand in every portal field |
| Configuration manager doesn't open after install | The installer skips launching the browser in some paths | Browse to `https://<computer-name>:44368` manually and accept the self-signed certificate |

---

## 🧹 Cleanup

Order matters. AWS first, then Azure, then the target resource group.

```bash
# 1. AWS
cd aws-side
terraform destroy

# 2. Azure
cd ../azure-side
terraform destroy

# 3. Target resource group (Terraform doesn't own it)
az group delete --name rg-migrate-target-fabrizio --yes

# 4. Azure auto-creates this one
az group delete --name NetworkWatcherRG --yes
```

**If `terraform destroy` fails on the Recovery Services Vault** with *"vault is not empty"* — that's the soft-delete consequence. Portal → the vault → **Replication items** and **Backup items** → delete everything listed → retry.

**If it fails on the resource group** because the Azure Migrate project created resources Terraform doesn't track (a Key Vault, discovery sites, a second vault), add this to the provider block:

```hcl
provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = false
    }
  }
}
```

That deletes the group via the Azure API, sweeping up portal-created resources alongside Terraform-managed ones.

**Verify nothing survives:**

```bash
az group list --output table
aws ec2 describe-instances --region us-east-1 \
  --filters "Name=tag:project,Values=azure-migrate-lab" \
  --query "Reservations[].Instances[?State.Name!='terminated'].InstanceId" --output text
```

**Don't forget:**
- Delete the scoped IAM inline policy from `terraform-migrate-lab`
- Confirm the `svc-azure-migrate-*` access key is gone — long-lived keys should never outlive the project
- **Re-enable Azure security defaults.** That account holds Owner on a live subscription

---

## 🔐 Security Notes

Deliberate lab shortcuts, and what production requires:

| Lab | Production |
|---|---|
| RDP open to `0.0.0.0/0` on three machines | Restrict to a specific source IP (`cidr_blocks = ["<your-ip>/32"]`), or remove public RDP entirely in favor of a bastion |
| `Resource: "*"` on `ec2:CreateSnapshot` / `ec2:DeleteSnapshot` | Tag-scope it with a condition on `aws:ResourceTag/project`. Only `ec2:Describe*` genuinely can't be resource-scoped — that's an AWS limitation |
| Windows Administrator password in EC2 `user_data` | `user_data` is readable from the Instance Metadata Service by anything running on the box. Use a key pair and decrypt the generated password instead |
| Static AWS access key with no expiry | Can't be eliminated — Azure Migrate cannot assume a role cross-cloud. So: scheduled rotation, and delete the key at cutover |
| Unencrypted root EBS volume | Enable EBS encryption. The disk gets snapshotted and shipped across a cloud boundary |
| `soft_delete_enabled = false` attempted on the vault | Azure now blocks this outright, and correctly — it's a ransomware control |
| Security defaults disabled for appliance registration | Scoped Conditional Access exclusion for the Azure PowerShell app instead |
| No private connectivity between clouds | Site-to-site VPN or ExpressRoute, with the source instance kept private |

`terraform.tfvars` and `terraform.tfstate` are gitignored. They contain three admin passwords and the AWS secret access key in cleartext. Copy the `.example` files and supply your own values.

---

## 💡 Key Takeaways

**Discovery and assessment are a complete deliverable on their own.** Even without reaching cutover, this produced a real cross-cloud inventory, a readiness determination, a right-sized recommendation, and a cost projection. A migration that stalls here has still answered the question the business was paying for.

**"Agentless" describes the source machine, not the migration.** AWS-to-Azure still requires two appliance VMs in Azure. Getting this wrong means budgeting for one VM and needing two — at 8 vCPUs each.

**Quota is not one number.** Regional vCPU quota and per-family vCPU quota are separate ceilings. Raising one does nothing for the other, and the error messages don't make that obvious.

**"8 cores" is not 8 vCPUs.** Azure Site Recovery's installer counts physical cores. Modern Azure VM families expose 2 vCPUs per physical core via SMT, so a documented 8-core minimum means a 16-vCPU VM. Confirming this with `Get-CimInstance Win32_Processor` rather than assuming was the difference between guessing and knowing.

**SKU availability is gated per-generation, not just per-region.** This subscription had only v7 enabled; every v3, v4, and v5 family was blocked in both regions tried. Nothing surfaces that until a deploy fails. `az vm list-skus --all | grep -v NotAvailable` answers it in ten seconds and should be the first command run in any new subscription.

**Error messages point at symptoms; logs point at causes.** "You don't have access to this" looked like an RBAC problem and consumed real time. The Entra sign-in log named it immediately: `AADSTS530035`, security defaults. When a permissions error doesn't match the permissions you can see, go to the logs.

**Infrastructure as code and portal configuration differ in what survives teardown.** Everything in `main.tf` rebuilds with one command. The Azure Migrate project, discovery data, and appliance registrations do not — they're recreated by hand every time. That distinction is worth living through once.

**Documentation goes stale faster than anyone updates it.** Roughly fifteen blockers in this project, and none were in the guide it was based on — the AWS free tier changed, portal navigation changed, appliance requirements grew, VM generations rolled over, and a security control became non-optional. The job isn't following steps. It's diagnosing why the documented path doesn't work in your environment.

---

**Author:** Fabrizio Mastrogiovanni
**Stack:** Terraform · AWS CLI · Azure CLI · Azure Migrate · Azure Site Recovery
**Time to complete:** ~8 hours across two sessions (original estimate: 4–6)
