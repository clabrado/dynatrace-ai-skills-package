---
name: dt-cloud-compliance
description: >
  Verify Dynatrace internal cloud resource compliance after any cloud work.
  Checks tagging policy (dt_owner_email, Owner, ACE:CREATED-BY), Azure security
  guidelines, and optional tags (ShouldBeReserved, SKIPPED_BY_AUTOMATION).
  Run after creating or modifying any AWS, Azure, or GCP cloud resources.
  Produces a compliance report with pass/warn/fail per resource.
args:
  - name: provider
    description: "Cloud provider to check: aws | azure | gcp (default: aws)"
    required: false
  - name: resources
    description: "Comma-separated list of resource IDs/names/ARNs to check. If omitted, checks recently touched resources."
    required: false
  - name: account
    description: "AWS account ID or Azure subscription ID. Defaults to active context."
    required: false
---

# dt-cloud-compliance — Dynatrace Internal Cloud Tagging & Security Compliance Checker

Run this skill after any cloud resource creation, modification, or infrastructure work to verify compliance with Dynatrace's internal R&D policies.

**Policy sources (memory):**
- `reference_cloud_tagging_policy` — mandatory tags, enforcement timelines, service scope
- `reference_azure_guidelines` — Azure-specific tagging, security, and subscription rules

---

## Quick Reference — What Gets Checked

### Mandatory tag check (ALL clouds)
| Tag | Rule |
|---|---|
| `dt_owner_email` | Must be set AND end with `@dynatrace.com` AND be an active employee |
| `Owner` (Azure) | Must be set on both resource group AND individual resource |
| `ACE:CREATED-BY` (Azure) | Must NOT be deleted; auto-managed by ACE automation |

### Service scope — tagging REQUIRED on:
| AWS | Azure | GCP |
|---|---|---|
| EC2, EBS, S3*, RDS, EKS, ECS | VMs, VMSSs, Disk Storage, Blob Storage, SQL DB, AKS | Compute Engine, Persistent Disk, Cloud Storage, Cloud SQL, GKE |

*S3 exempt patterns: `ace-cnc-cfn-*`, `cf-templates-*`, `heimdall-aws-cf-*`, `stackset-*`, `elasticbeanstalk-*`

### Optional tag checks (warn if missing on long-lived resources):
- `ShouldBeReserved` — should be TRUE for resources planned to run >3 months
- `SKIPPED_BY_AUTOMATION` — relevant for detached EBS/Disk/Persistent Disk

---

## Execution Protocol

### Step 0 — Determine scope

If `--resources` provided, check exactly those.
If not provided, determine recently touched resources:
- AWS: `aws resourcegroupstaggingapi get-resources` or `aws ec2 describe-instances --filters Name=instance-state-name,Values=running` scoped to the active account
- Azure: `az resource list --query "[?tags.\"ACE:CREATED-BY\" == '<user>']"` or recently modified resources
- GCP: `gcloud asset search-all-resources` with recent createTime filter

Always confirm the active account/subscription/project before querying.
For AWS, default account is `446130280781` (NORAM SE account) unless `--account` is specified.

### Step 1 — Fetch tags for each in-scope resource

**AWS (use AWS CLI or boto3):**
```bash
# Get tags for a specific resource
aws resourcegroupstaggingapi get-resources \
  --resource-type-filters ec2:instance \
  --query 'ResourceTagMappingList[*].{ARN:ResourceARN,Tags:Tags}'

# Or for a specific ARN
aws resourcegroupstaggingapi get-resources \
  --resource-arn-list <arn1> <arn2>
```

**Azure (use Azure CLI):**
```bash
az resource show --ids <resource_id> --query tags
az group show --name <rg_name> --query tags
```

**GCP (use gcloud):**
```bash
gcloud compute instances describe <name> --zone <zone> --format="value(labels)"
```

### Step 2 — Evaluate each resource

For each resource, evaluate:

```
RESULT = PASS | WARN | FAIL

FAIL conditions (blocks policy compliance):
  - Resource type is in mandatory scope AND dt_owner_email is missing
  - dt_owner_email is present but does NOT end with @dynatrace.com
  - Azure: Owner tag is missing from resource OR its resource group
  - Azure: ACE:CREATED-BY tag was deleted (check with az resource show)

WARN conditions (not blocking but should be addressed):
  - dt_owner_email is set correctly but dt_owner_team is also absent (belt-and-suspenders)
  - Resource appears to be running >3 months and ShouldBeReserved is not set to TRUE
  - Detached EBS / Disk / Persistent Disk without SKIPPED_BY_AUTOMATION = TRUE
  - GCP: dt_owner_email not converted to GCP label format (chris_labrado-dynatrace_com)

PASS: All mandatory tags present and valid
```

### Step 3 — Remediation commands

For any FAIL or WARN, generate the exact CLI command to fix it.

**AWS — add/update tag:**
```bash
aws ec2 create-tags \
  --resources <instance-id> \
  --tags Key=dt_owner_email,Value=chris.labrado@dynatrace.com
```

**Azure — add/update tag:**
```bash
az resource tag \
  --ids <resource-id> \
  --tags dt_owner_email=chris.labrado@dynatrace.com Owner=chris.labrado@dynatrace.com
# Also tag the resource group:
az group update \
  --name <rg-name> \
  --tags Owner=chris.labrado@dynatrace.com
```

**GCP — add/update label (note substitution: . → _ and @ → -):**
```bash
gcloud compute instances add-labels <name> \
  --zone <zone> \
  --labels dt_owner_email=chris_labrado-dynatrace_com
```

**Terraform — add to ignore_tags block to protect ACE auto-tags (Azure):**
```hcl
provider "azurerm" {
  ignore_tags {
    key_prefixes = ["ACE:"]
  }
}
```

### Step 4 — Output compliance report

Format:

```
## Cloud Compliance Report — <date> — <provider> — <account/subscription>

| Resource | Type | dt_owner_email | Owner | ACE:CREATED-BY | ShouldBeReserved | Result |
|---|---|---|---|---|---|---|
| <name/id> | EC2 | chris.labrado@dynatrace.com | — | — | — | ✅ PASS |
| <name/id> | EBS | MISSING | — | — | — | ❌ FAIL |

### Remediation Actions Required
1. [FAIL] <resource> — missing dt_owner_email
   Fix: aws ec2 create-tags --resources <id> --tags Key=dt_owner_email,Value=chris.labrado@dynatrace.com

### Enforcement Reminder
Resources with invalid tags will be STOPPED in 14 days and TERMINATED in 28 days.
S3/Blob/Cloud Storage will be DELETED in 30 days.
Contact #help-finops with questions.
```

---

## Azure Security Checklist (run when `--provider azure`)

In addition to tagging, verify:

- [ ] No SSH (22) or RDP (3389) NSG rules allowing `0.0.0.0/0` — should be restricted to Dynatrace network IPs
- [ ] No storage accounts with `publicNetworkAccess = Enabled` without ACE exemption
- [ ] No Visual Studio / Free Trial / PAYG subscriptions in scope
- [ ] Service principal secrets are not hardcoded in code or config files

**CLI checks:**
```bash
# Check for open SSH/RDP rules
az network nsg list --query "[].securityRules[?destinationPortRange=='22' || destinationPortRange=='3389']"

# Check storage account public access
az storage account list --query "[?publicNetworkAccess=='Enabled'].{name:name,rg:resourceGroup}"
```

---

## Chris LaBrado defaults
- AWS account: `446130280781` (NORAM SE)
- Default owner email for remediation: `chris.labrado@dynatrace.com`
- GCP label format: `chris_labrado-dynatrace_com`
- Azure `Owner` tag value: `chris.labrado@dynatrace.com`
- dtctl context for SE work: `sprint-<your-tenant>` or `<your-dtctl-context>`

---

## Support contacts
| Issue | Contact |
|---|---|
| Tagging policy questions | `#help-finops` |
| Azure general questions | `#help-gotc` |
| Cloud IAM/access | Cloud Access Engineering: https://teams.internal.dynatrace.com/teams/461835 |
| Cost optimization | ACE FinOps: https://teams.internal.dynatrace.com/teams/462163 |
