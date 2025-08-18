### **8.1 Environment Patterns: Dev, Staging, Prod - The *Why* and *What***
**Core Purpose:** Isolate infrastructure to prevent "what breaks in dev" from affecting production.  
**Key Patterns & Rationale:**

| Environment | Purpose | Critical Characteristics | Terraform Requirements |
|-------------|---------|--------------------------|------------------------|
| **Dev** | Rapid iteration, developer sandbox | • Ephemeral (destroy/recreate daily)<br>• Low-cost resources (t3.micro)<br>• No backups<br>• Publicly accessible for testing | • Fast apply/destroy cycles<br>• Minimal state locking<br>• Explicit destroy allowed |
| **Staging** | Pre-production validation | • Mirror of prod (same instance types)<br>• Realistic data (anonymized)<br>• Full monitoring<br>• Weekly backups | • Strict change control<br>• State locking enforced<br>• No auto-destroy |
| **Prod** | Customer-facing systems | • High availability (multi-AZ)<br>• Security hardening (WAF, VPC flow logs)<br>• Audit trails<br>• Zero-downtime deployments | • Immutable infrastructure<br>• Strict IAM policies<br>• State stored remotely (S3+DynamoDB)<br>• Plan/apply via CI/CD |

**Why NOT One Environment?**  
- **Risk:** `terraform apply` in dev accidentally modifies prod (if state isn't isolated)  
- **Compliance:** GDPR/PCI require data segregation  
- **Cost:** Staging can use smaller instances than prod  
- **Speed:** Dev can skip expensive checks (SSL certs, backups)

---

### **8.2 Strategy 1: Directory-per-Environment (The Gold Standard)**
**How it Works:** Separate directory for each environment containing identical Terraform configurations.  
**Directory Structure:**
```bash
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars  # dev-specific values
│   ├── staging/
│   │   ├── main.tf
│   │   └── ... # same files as dev
│   └── prod/
│       └── ... 
└── modules/  # Shared modules (critical!)
    ├── network/
    └── compute/
```

**Implementation Steps:**  
1. Create identical `main.tf` in each environment dir (calls modules from `../modules`)  
2. Define environment-specific variables in `terraform.tfvars` (e.g., `instance_type = "t3.small"` in dev)  
3. Configure **remote state per environment** (non-negotiable!):  
   ```hcl
   # environments/dev/backend.tf
   terraform {
     backend "s3" {
       bucket = "mycompany-terraform-state"
       key    = "dev/terraform.tfstate"  # Unique per env!
       region = "us-east-1"
       dynamodb_table = "terraform-locks"
     }
   }
   ```

**Pros:**  
✅ Absolute state isolation (no risk of cross-env drift)  
✅ Full flexibility (different Terraform versions per env if needed)  
✅ Easy to audit changes (`git diff dev/ staging/`)  
✅ Works with ANY Terraform version  

**Cons:**  
⚠️ Configuration duplication (solved by modules - see 8.7)  
⚠️ Manual env creation (mitigated by scripts)  

**When to Use:** **95% of production use cases.** *Always start here.*  
**Pitfall Alert:** If you forget unique `key` in backend config, all envs share state → **catastrophic failure**.

---

### **8.3 Strategy 2: Workspaces (with Caveats)**
**How it Works:** Terraform creates isolated state *namespaces* within a single configuration directory.  
**Basic Usage:**
```bash
terraform workspace new dev    # Creates "dev" workspace
terraform workspace select dev
terraform apply -var-file=dev.tfvars
```

**Under the Hood:**  
- State stored as `env:/dev/...` in your backend (e.g., S3 path `env:/dev/terraform.tfstate`)  
- `terraform.workspace` variable available in configs (use sparingly!)

**Critical Caveats (Why Most Experts Avoid This):**  
| Issue | Explanation | Real-World Impact |
|-------|-------------|-------------------|
| **State Coupling** | All workspaces share the *same* Terraform config | Change in `main.tf` affects ALL envs simultaneously → staging gets dev changes by accident |
| **No Config Isolation** | Can't have different Terraform versions per env | Prod might break during dev testing of new TF version |
| **Variable Chaos** | `terraform.workspace` encourages conditional logic (anti-pattern) | `count = terraform.workspace == "prod" ? 3 : 1` → brittle, hard to test |
| **Accidental Switching** | `terraform workspace select prod` then `apply` without checking | Dev runs plan in prod workspace → **production outage** |

**When Workspaces *Might* Work:**  
- Temporary environments (e.g., `feature-branch` testing)  
- Environments requiring *identical* infrastructure (e.g., multi-region prod)  
- **Never** for dev/staging/prod separation!

**Expert Verdict:** *"Workspaces are a footgun for environment separation."* – Yoriyasu Yano (ex-TF core team)

---

### **8.4 Strategy 3: Terragrunt (Bonus - The Enterprise Solution)**
**What it Is:** Thin wrapper for Terraform (by Gruntwork) solving DRY and state management.  
**Core Philosophy:** *"Keep your configs DRY, but not your state."*

**How it Solves Directory-per-Env Pain Points:**  
1. **DRY Configs via `generate` & `include`:**  
   ```hcl
   # live/dev/terragrunt.hcl
   include "root" {
     path = find_in_parent_folders()  # Inherit from root terragrunt.hcl
   }
   terraform {
     source = "git::git@github.com:org/modules.git//compute?ref=v1.0"
   }
   inputs = {
     instance_type = "t3.small"
     env           = "dev"
   }
   ```
2. **Automated Remote State:**  
   ```hcl
   # live/terragrunt.hcl (root)
   remote_state {
     backend = "s3"
     config = {
       bucket  = "mycompany-terraform-state"
       key     = "${path_relative_to_include()}/terraform.tfstate"
       region  = "us-east-1"
     }
   }
   ```
   → Auto-generates unique state paths: `dev/compute/terraform.tfstate`  
3. **Dependency Management:**  
   ```hcl
   dependencies {
     paths = ["../vpc"]  # Auto-resolves VPC before compute
   }
   ```

**Pros:**  
✅ Eliminates config duplication (without coupling state)  
✅ Enforces consistent remote state/backend config  
✅ Dependency graph handling  
✅ Built-in safety checks (`prevent_destroy` for prod)  

**Cons:**  
⚠️ Adds complexity (new tool to learn)  
⚠️ Requires disciplined `terragrunt.hcl` structure  
⚠️ Slower execution (minor)  

**When to Adopt:**  
- When directory-per-env becomes unmanageable (10+ environments)  
- For strict compliance (SOC2, HIPAA) requiring audit trails  
- *Not* for small projects – start with Strategy 1 first.

---

### **8.5 Using tfvars per Environment - The Right Way**
**Purpose:** Inject environment-specific values WITHOUT modifying core Terraform code.  
**Implementation:**  
```bash
environments/
├── dev/
│   ├── terraform.tfvars       # Auto-loaded by TF
│   └── secrets.auto.tfvars    # Gitignored (for secrets)
├── staging/
│   └── terraform.tfvars
└── prod/
    └── terraform.tfvars
```

**`terraform.tfvars` Example (dev):**
```hcl
# environments/dev/terraform.tfvars
instance_count = 1
instance_type  = "t3.micro"
environment    = "dev"  # Critical for naming/resources
enable_backup  = false
```

**`secrets.auto.tfvars` (NEVER commit this!):**
```hcl
# environments/prod/secrets.auto.tfvars
db_password = "s3cr3t!"  # Should use Vault/SSM instead!
```

**Critical Rules:**  
1. **NEVER** hardcode values in `main.tf` → always use `var.*`  
2. **ALWAYS** set `environment` variable (used in resource names: `my-app-${var.environment}`)  
3. **NEVER** commit secrets → use `-var-file=secrets.tfvars` (add to `.gitignore`) or Vault  
4. **ALWAYS** define variables in `variables.tf`:  
   ```hcl
   variable "instance_type" {
     description = "EC2 instance type"
     type        = string
     default     = "t3.small" # Dev default
   }
   ```

**How Terraform Loads Variables (Order of Precedence):**  
1. `terraform.tfvars` (highest priority)  
2. `*.auto.tfvars` (in lexical order)  
3. `-var` or `-var-file` CLI flags  
4. Environment variables (`TF_VAR_xxx`)  
5. `variables.tf` default values (lowest priority)

---

### **8.6 Environment-Specific Variables and Overrides - Advanced Patterns**
**Beyond Basic tfvars:** When you need dynamic behavior per environment.

#### **Pattern 1: Variable Mapping (Recommended)**
```hcl
# variables.tf
variable "env_config" {
  type = map(object({
    instance_count = number
    instance_type  = string
  }))
  default = {
    dev = {
      instance_count = 1
      instance_type  = "t3.micro"
    }
    prod = {
      instance_count = 3
      instance_type  = "m5.large"
    }
  }
}

# main.tf
module "app" {
  source         = "./modules/app"
  instance_count = var.env_config[var.environment].instance_count
  instance_type  = var.env_config[var.environment].instance_type
}
```
**Why it works:** Single source of truth for env mappings. Easy to add new environments.

#### **Pattern 2: Conditional Overrides (Use Sparingly!)**
```hcl
# ONLY for rare cases where structure differs
resource "aws_db_instance" "main" {
  # Base config
  allocated_storage = 20

  # Prod-only override
  lifecycle {
    ignore_changes = [
      (var.environment == "prod" ? allocated_storage : null)
    ]
  }
}
```
**Warning:** Overuse leads to "spaghetti infrastructure." Prefer Pattern 1.

#### **Pattern 3: Dynamic Blocks (For Complex Resources)**
```hcl
resource "aws_lb" "main" {
  # Base config

  dynamic "access_logs" {
    for_each = var.environment == "prod" ? [1] : []
    content {
      bucket = "prod-logs-bucket"
    }
  }
}
```
→ Enables/disables access logs ONLY in prod.

**Golden Rule:** *If you're writing `if environment == "prod"`, ask: "Can I model this as a variable instead?"*

---

### **8.7 Module Reuse Across Environments - The Secret Sauce**
**Core Principle:** Modules must be **environment-agnostic**. Environment awareness lives *outside* modules.

#### **Anti-Pattern (Broken!):**
```hcl
# modules/compute/main.tf (WRONG!)
resource "aws_instance" "app" {
  count = var.environment == "prod" ? 3 : 1  # MODULE KNOWS ENV → BAD
}
```

#### **Correct Pattern:**
```hcl
# modules/compute/variables.tf
variable "instance_count" {
  description = "Number of instances"
  type        = number
  default     = 1  # Safe default (dev-friendly)
}

# environments/prod/main.tf (ENV-SETTING HAPPENS HERE)
module "app" {
  source         = "../../modules/compute"
  instance_count = 3  # Prod sets count
  instance_type  = "m5.large"
}
```

#### **Advanced: Environment-Aware Naming**
```hcl
# modules/compute/main.tf
resource "aws_instance" "app" {
  tags = {
    Name = "${var.app_name}-${var.environment}"  # Module accepts environment
  }
}

# environments/prod/main.tf
module "app" {
  source      = "../../modules/compute"
  app_name    = "my-app"
  environment = var.environment  # Passed from env config
}
```
**Why this works:** Module *uses* environment for naming but doesn't *decide* environment logic.

#### **Critical Module Design Rules:**
1. **NO** hardcoded environment names (dev/staging/prod) inside modules  
2. **ALL** environment-specific values come from module inputs  
3. **ALWAYS** set `environment` as a top-level variable (used in tags/names)  
4. **NEVER** put secrets in modules → pass via secure inputs (Vault dynamic secrets)  
5. **TEST** modules in isolation with `count=1` (dev-like config)

---

### **Key Takeaways for Your Reference**
| Strategy | When to Use | State Isolation | Avoid If |
|----------|-------------|-----------------|----------|
| **Directory-per-Env** | Most projects (start here!) | ✅ Absolute | Only if you have <5 envs and no compliance needs |
| **Workspaces** | Temporary envs / identical infra | ❌ Shared config | **Never** for dev/staging/prod |
| **Terragrunt** | 10+ envs / strict compliance | ✅ (via auto-path) | Small projects or Terraform beginners |

**Non-Negotiable Best Practices:**  
1. **Remote state per environment** (S3 + DynamoDB)  
2. **`environment` variable** in ALL resources (for tagging/naming)  
3. **Modules = environment-agnostic** (env logic in root configs)  
4. **tfvars for values, NOT structure** (use mapping objects)  
5. **NEVER commit secrets** → use Vault/SSM Parameter Store  

**Red Flags to Watch For:**  
🔴 `terraform.workspace` used for env separation → **switch to directories**  
🔴 Conditional logic (`if env == "prod"`) inside modules → **move to root config**  
🔴 Same state file for multiple environments → **immediate state migration required**  

> "Environment separation isn't about Terraform features – it's about **risk containment**. Your dev mistakes should never touch prod, and Terraform's job is to enforce that boundary." – *Adapted from HashiCorp's Production Checklist*
