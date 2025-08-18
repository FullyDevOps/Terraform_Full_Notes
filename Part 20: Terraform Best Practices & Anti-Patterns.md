### **20.1 Directory Structure Best Practices**  
**Why it matters:** Poor structure leads to unmanageable code, state conflicts, and team friction.  

#### **Optimal Structure**  
```bash
project-root/
├── modules/                  # Reusable, versioned modules (NEVER environment-specific)
│   ├── vpc/                  # Example: Network module
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── ec2-instance/         # Compute module
├── environments/             # Environment-specific configurations (state isolation)
│   ├── prod/
│   │   ├── main.tf           # ONLY calls modules + sets vars
│   │   ├── terraform.tfvars  # Prod-specific vars
│   │   └── backend.tf        # Prod backend config (e.g., S3 bucket)
│   └── staging/
│       ├── main.tf
│       └── ...               # Same pattern
├── .terraform-version        # Enforces Terraform CLI version
├── README.md                 # Project overview & setup guide
└── providers.tf              # Global provider config (if not in modules)
```

#### **Key Rules**  
1. **Strict Separation**:  
   - `modules/`: Pure infrastructure definitions. **No environment-specific logic**.  
   - `environments/`: Thin wrappers that *only* call modules + set variables.  
   - *Anti-pattern:* Mixing environment logic inside modules (e.g., `if var.env == "prod"`).  

2. **State Isolation**:  
   - Each environment (`prod`, `staging`) **must** have its own state file (via `backend.tf`).  
   - *Anti-pattern:* Using one state file for all environments → accidental prod changes.  

3. **Module Reusability**:  
   - Modules should work *anywhere* (e.g., `modules/vpc` used in `prod` and `staging`).  
   - *Anti-pattern:* Hardcoding region/AZ in modules → breaks multi-region deployments.  

4. **Avoid Top-Level Resources**:  
   - All resources should live in modules. Top-level `environments/` files only compose modules.  
   - *Why?* Prevents "spaghetti code" and enables module reuse across projects.  

---

### **20.2 Naming Conventions**  
**Why it matters:** Consistent naming enables automation, auditing, and reduces human error.  

#### **Resource Names**  
- **Format:** `resource_type_environment_short-description`  
  ```hcl
  # Example: AWS S3 bucket for prod logs
  resource "aws_s3_bucket" "logs_prod_app" {
    bucket = "company-logs-prod-app"
  }
  ```
- **Rules:**  
  - Use **lowercase** + **hyphens** (AWS/GCP requirement).  
  - Include `environment` (prod/staging/dev) **and** `application` (e.g., `app`, `db`).  
  - Avoid generic names like `bucket1` or `server`.  
  - *Anti-pattern:* `resource "aws_s3_bucket" "my_bucket" {}` → Impossible to identify in AWS console.  

#### **Variable/Output Names**  
- **Format:** `descriptive_snake_case`  
  ```hcl
  variable "instance_type" {  # Good
    type = string
  }
  
  variable "instancetype" {   # Bad (ambiguous)
    type = string
  }
  ```
- **Rules:**  
  - Be specific: `vpc_cidr` not `cidr_block`.  
  - Prefix module inputs: `module "vpc" { vpc_cidr = var.prod_vpc_cidr }`.  

#### **File Names**  
- `main.tf`: Resource definitions.  
- `variables.tf`: All inputs.  
- `outputs.tf`: All outputs.  
- `versions.tf`: Required Terraform/provider versions.  
- *Anti-pattern:* `prod-config.tf` or `networking.tf` → Breaks standard tooling.  

---

### **20.3 Version Pinning (Providers, Modules)**  
**Why it matters:** Unpinned versions cause **unpredictable breaks** during `terraform init`.  

#### **Provider Pinning**  
```hcl
# versions.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"  # Pin MAJOR.MINOR; allows PATCH updates
    }
  }
}
```
- **`~>` Operator**:  
  - `~> 5.0` = `>= 5.0.0, < 6.0.0` (allows patch updates, blocks breaking changes).  
  - *Never* use `latest` or omit version.  
- **Why?** Provider v5.1 might break your v4.0 code. Pinning ensures reproducibility.  

#### **Module Pinning**  
```hcl
module "vpc" {
  source  = "git::https://example.com/vpc.git?ref=v1.2.0"
  # OR for Terraform Registry:
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"  # EXACT version
}
```
- **Always pin to a semantic version tag** (e.g., `v1.2.0`).  
- *Anti-pattern:* `ref=main` → Module changes break your environment silently.  

#### **Terraform CLI Pinning**  
- Use `.terraform-version` file (with `tfenv` or similar):  
  ```bash
  # .terraform-version
  1.6.6
  ```
- **Why?** Newer Terraform versions may deprecate syntax (e.g., `count` → `for_each`).  

---

### **20.4 Avoiding Large Monolithic Configurations**  
**Why it matters:** Monoliths cause slow plans, state corruption risks, and team bottlenecks.  

#### **Symptoms of a Monolith**  
- One `main.tf` with 500+ lines.  
- All resources in a single state file.  
- `terraform apply` takes >10 minutes.  

#### **How to Split**  
1. **By Component**:  
   ```bash
   environments/prod/
   ├── network/       # VPC, subnets, route tables
   ├── compute/       # EC2, EKS, ASG
   └── main.tf        # Composes network + compute
   ```
2. **By Environment**:  
   - Separate state for `prod`, `staging`, `dev` (as in 20.1).  
3. **By Team Ownership**:  
   - `team-a/` (owns databases), `team-b/` (owns frontend).  

#### **Critical Rules**  
- **State Boundaries = Deployment Units**:  
  - If resources can be updated independently, they need separate state.  
- *Anti-pattern:* `prod/main.tf` defining VPC, RDS, *and* Lambda → One broken resource blocks all updates.  
- **Use `terraform_remote_state`** to share data between states (e.g., VPC ID for compute layer).  

---

### **20.5 Using `for_each` Over `count` When Possible**  
**Why it matters:** `count` causes resource replacement on index shifts; `for_each` is stable.  

#### **When to Use `for_each`**  
- **Dynamic Resource Sets** (e.g., multiple similar resources with unique IDs):  
  ```hcl
  # GOOD: for_each with map
  resource "aws_s3_bucket" "logs" {
    for_each = {
      app  = "app-logs"
      audit = "audit-logs"
    }
    bucket = "company-${each.key}-${var.env}"
  }
  ```
- **Why better than `count`?**  
  - If you add a bucket (`"metrics"`), only *new* buckets are created.  
  - With `count`, adding an item **replaces all resources** (index shift → `bucket[0]` becomes `bucket[1]`).  

#### **When `count` is Acceptable**  
- Fixed-size lists (e.g., 3 identical subnets per AZ).  
- *But prefer:* `for_each = toset(var.azs)` to avoid index issues.  

#### **Anti-Patterns**  
```hcl
# BAD: count with dynamic list
resource "aws_instance" "web" {
  count = length(var.instance_types)  # If var.instance_types changes, ALL instances replaced!
  instance_type = var.instance_types[count.index]
}
```

---

### **20.6 Minimizing Use of Provisioners**  
**Why it matters:** Provisioners create **state drift**, break immutability, and complicate recovery.  

#### **Provisioner Types & Risks**  
| Type          | Example                     | Risk                                  |
|---------------|-----------------------------|---------------------------------------|
| `local-exec`  | Run `kubectl apply`         | Fails silently; no state tracking     |
| `remote-exec` | SSH into VM to configure OS | Breaks if SSH fails; no idempotency   |
| `file`        | Copy config to VM           | Drift if VM modified manually         |

#### **When to Avoid Provisioners**  
- **For OS Configuration**: Use cloud-init (AWS) or startup scripts (GCP).  
- **For App Deployment**: Use Kubernetes manifests or serverless (Lambda).  
- **For Secrets**: Use Vault/Terraform Cloud variables, **never** `local-exec` with `aws ssm put-parameter`.  

#### **Only Use Provisioners When**  
- **No Native Terraform Resource Exists** (e.g., legacy on-prem API call).  
- **As a Last Resort**:  
  ```hcl
  resource "aws_instance" "app" {
    # ...
    provisioner "local-exec" {
      when    = destroy  # ONLY for cleanup
      command = "curl -X DELETE https://legacy-api.example.com/${self.id}"
    }
  }
  ```

#### **Anti-Pattern**  
```hcl
# NEVER: Using provisioners for core config
resource "aws_instance" "db" {
  # ...
  provisioner "remote-exec" {
    inline = ["sudo apt-get install mysql"]  # Breaks immutability!
  }
}
```
→ **Fix:** Bake AMI with Packer; deploy pre-configured image.  

---

### **20.7 Immutable Infrastructure Principles**  
**Why it matters:** Mutable infra leads to "snowflake servers" and unrepeatable deployments.  

#### **Core Principles**  
1. **No In-Place Changes**:  
   - Never SSH to modify a running server. Update the AMI/instance config and replace.  
2. **Versioned Artifacts**:  
   - AMIs, Docker images, and Terraform modules **must** have immutable versions (e.g., `ami-123abc`).  
3. **Declarative State**:  
   - Terraform config = single source of truth. Manual changes = drift (detected via `terraform plan`).  

#### **How Terraform Enables Immutability**  
- **Replace, Don’t Patch**:  
  ```hcl
  resource "aws_instance" "app" {
    ami           = "ami-123abc"  # Versioned AMI
    instance_type = "t3.medium"
    # Changing ami triggers NEW instance (old one destroyed)
  }
  ```
- **Auto Scaling Groups (ASG)**:  
  - Use `terraform apply` to update ASG launch template → ASG replaces instances.  

#### **Anti-Patterns**  
- Using `user_data` to run `apt upgrade` → Mutable OS.  
- Manually scaling ASG instances → Drift from Terraform state.  

---

### **20.8 Handling Destructive Changes Safely**  
**Why it matters:** Accidental `destroy` can delete production databases.  

#### **Destructive Change Workflow**  
1. **Identify Destruction**:  
   ```bash
   terraform plan -out=tfplan
   # LOOK FOR: `-/+` (replace) or `-` (destroy)
   ```
2. **Confirm Intent**:  
   - For critical resources (RDS, S3), add `lifecycle` warnings:  
     ```hcl
     resource "aws_db_instance" "prod" {
       # ...
       lifecycle {
         prevent_destroy = true  # BLOCKS terraform destroy
       }
     }
     ```
3. **Isolate Destructive Changes**:  
   - Split into separate `terraform apply` (e.g., update networking first, then compute).  
4. **Backup First**:  
   - For databases: `aws rds create-db-snapshot` before `apply`.  
5. **Use `-target` Sparingly**:  
   ```bash
   terraform apply -target=module.db  # ONLY update DB (bypasses dependencies)
   ```
   → *Warning:* Only for emergencies; breaks dependency graph.  

#### **Critical Anti-Patterns**  
- Running `terraform apply` without reviewing `plan` output.  
- Not using `prevent_destroy` on stateful resources (RDS, S3).  
- Storing state in local file (`terraform.tfstate`) → accidental deletion.  

---

### **20.9 Documentation in Code (Comments, READMEs)**  
**Why it matters:** Undocumented Terraform = unmaintainable infrastructure.  

#### **Where to Document**  
1. **Inline Comments (for complex logic)**:  
   ```hcl
   # WHY: Route 53 health check required for failover (AWS docs: https://...)
   # HOW: This record only updates if primary endpoint is unhealthy
   resource "aws_route53_record" "app" {
     # ...
   }
   ```
   - *Rule:* Only explain **why**, not **what** (code should be self-documenting).  

2. **Module `README.md` (MANDATORY)**:  
   ```markdown
   ## Module: `modules/vpc`
   ### Purpose
   Creates a production-grade VPC with public/private subnets.
   
   ### Inputs
   | Name          | Description                     | Type   | Default     |
   |---------------|---------------------------------|--------|-------------|
   | `env`         | Environment (prod/staging)      | string | -           |
   | `cidr`        | VPC CIDR block                  | string | "10.0.0.0/16" |
   
   ### Outputs
   | Name          | Description                     |
   |---------------|---------------------------------|
   | `vpc_id`      | ID of the VPC                   |
   
   ### Usage
   ```hcl
   module "prod_vpc" {
     source = "./modules/vpc"
     env    = "prod"
     cidr   = "10.10.0.0/16"
   }
   ```
   ```

3. **Environment `README.md`**:  
   - Deployment commands: `terraform init -backend-config=prod.backend`  
   - Owner/team contact info.  
   - Disaster recovery steps.  

#### **Anti-Patterns**  
- Comments that duplicate code: `# Create S3 bucket` (redundant).  
- No `README.md` for modules → new engineers can’t use them.  
- Outdated comments (e.g., comment says `cidr="10.0.0.0/16"` but code uses `10.1.0.0/16`).  

---

### **Summary: Golden Rules**  
1. **Structure**: Modules + environments = isolated, reusable code.  
2. **Naming**: Be specific, include environment, follow cloud provider rules.  
3. **Pinning**: Lock versions for providers/modules/Terraform CLI.  
4. **Split Monoliths**: One state file per deployable unit.  
5. **Prefer `for_each`**: Avoid resource replacement from index shifts.  
6. **Avoid Provisioners**: They break immutability; use native config.  
7. **Immutable Infra**: Replace, don’t patch; version everything.  
8. **Destroy Safely**: `prevent_destroy`, backups, and targeted applies.  
9. **Document Relentlessly**: `README.md` for modules + environments.  

> **Pro Tip**: Run `terraform validate` and `checkov` in CI to enforce these rules automatically.  
> **Remember**: Terraform is *infrastructure as code* – treat it like application code (version control, PR reviews, testing).  
