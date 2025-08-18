### **6.1 What are Modules?**  
**Core Concept:**  
Modules are **self-contained packages of Terraform configurations** that group resources together to create a single, reusable component. Think of them as "functions" for infrastructure: you define inputs, they produce outputs, and hide internal complexity.

**Why They Exist:**  
- **Reusability:** Deploy the same infrastructure pattern (e.g., VPC, Kubernetes cluster) across environments.  
- **Encapsulation:** Hide implementation details (e.g., "network module" vs. 20 individual resource blocks).  
- **Composition:** Build complex systems by combining simpler modules (e.g., `app` module = `vpc` + `rds` + `eks`).  
- **Team Collaboration:** Teams own specific modules (e.g., networking team owns `vpc` module).  

**Key Analogy:**  
> Like LEGO bricks: A single brick (module) has a defined interface (stud/holes). You snap them together (compose) to build complex structures without knowing how the brick is molded internally.

**Critical Insight:**  
- **Every Terraform configuration is a module** (the root module).  
- Modules prevent "copy-paste infrastructure" – the #1 cause of configuration drift and errors.

---

### **6.2 Module Structure and Layout**  
**Standard Directory Structure (Mandatory for Registry/Public Modules):**  
```bash
my-module/
├── main.tf          # Resource definitions (the "engine")
├── variables.tf     # Input variables (module interface)
├── outputs.tf       # Output values (what the module exposes)
├── README.md        # Critical: Usage instructions, examples, requirements
├── versions.tf      # Terraform & provider version constraints (v1.0+)
├── examples/        # Real-world usage examples (REQUIRED for good modules)
│   └── simple/      # Example 1: Minimal configuration
│       ├── main.tf
│       └── variables.tf
└── tests/           # (Optional but recommended) Terratest integration tests
```

**Why This Structure Matters:**  
- **`versions.tf`** ensures compatibility (e.g., `terraform { required_version = ">= 1.4" }`).  
- **`examples/`** is non-negotiable – it’s how users learn to use your module.  
- **No `terraform init` in modules** – the *caller* runs `init`.  
- **NEVER** include `provider` blocks in modules (caller defines providers).

**Pro Tip:**  
Use `terraform validate` inside the module directory to check syntax *before* publishing.

---

### **6.3 Creating a Local Module**  
**Step-by-Step:**  
1. Create the directory structure (as above).  
2. Define resources in `main.tf` (e.g., an AWS VPC):  
   ```hcl
   # my-module/main.tf
   resource "aws_vpc" "main" {
     cidr_block = var.cidr_block
     tags = { Name = "main-vpc" }
   }
   ```
3. Declare inputs in `variables.tf`:  
   ```hcl
   variable "cidr_block" {
     description = "CIDR block for VPC"
     type        = string
     default     = "10.0.0.0/16"
   }
   ```
4. Expose outputs in `outputs.tf`:  
   ```hcl
   output "vpc_id" {
     description = "ID of the VPC"
     value       = aws_vpc.main.id
   }
   ```

**Calling the Local Module (in root config):**  
```hcl
module "prod_vpc" {
  source = "./my-module"  # Relative path to module
  cidr_block = "10.10.0.0/16"
}
```

**Critical Gotchas:**  
- **Path must be relative** (e.g., `../modules/vpc`). Absolute paths break in CI/CD.  
- **Local modules are copied**, not linked. Changes require `terraform init -upgrade`.  
- **Never commit `.terraform/`** – it’s regenerated on `init`.

---

### **6.4 Calling Modules (Source, Version, Inputs)**  
**Syntax Breakdown:**  
```hcl
module "my_module" {
  source  = "<SOURCE_PATH>"  # Where to get the module
  version = "<VERSION>"      # (Optional) For remote modules
  # Inputs:
  input1 = value1
  input2 = value2
}
```

**`source` Types Explained:**  
| Type                | Example                                      | Use Case                          |
|---------------------|----------------------------------------------|-----------------------------------|
| **Local Path**      | `./modules/vpc`                              | Development, testing              |
| **Git (HTTPS)**     | `git::https://example.com/vpc.git?ref=v1.0`  | Private repos (GitHub, GitLab)    |
| **Git (SSH)**       | `git::ssh://git@github.com/user/vpc.git`     | Authenticated private repos       |
| **Terraform Registry** | `terraform-aws-modules/vpc/aws`            | Public/private registry modules   |
| **HTTP URL**        | `http://example.com/vpc-module.zip`          | Rare (use Git/Registry instead)   |

**`version` Deep Dive:**  
- **Only works for remote sources** (Git tags, Registry versions).  
- **SemVer constraints** (see 6.10):  
  ```hcl
  version = "1.2.0"    # Exact version
  version = "~> 1.2"   # >=1.2.0, <1.3.0 (allows patches)
  version = ">=1.0.0"  # Any version >=1.0.0 (risky!)
  ```
- **Critical:** Always pin versions in production (`version = "1.2.3"`). Unpinned versions cause silent breaks.

**Input Handling:**  
- Inputs **must match variables** defined in the module.  
- **No partial inputs** – all required variables must be provided.  
- Use `terraform console` to debug input values.

---

### **6.5 Module Inputs and Outputs**  
**Inputs (`variables.tf`):**  
- **Required vs. Optional:**  
  ```hcl
  variable "instance_type" {
    description = "EC2 instance type"
    type        = string
    # No default = REQUIRED
  }
  
  variable "environment" {
    type    = string
    default = "dev"  # Optional (has default)
  }
  ```
- **Validation (v1.2+):**  
  ```hcl
  variable "azs" {
    type    = list(string)
    default = ["us-east-1a"]
    validation {
      condition     = length(var.azs) <= 3
      error_message = "Cannot specify more than 3 AZs."
    }
  }
  ```
- **Sensitivity:** Mark secrets with `sensitive = true` (hides values in logs).

**Outputs (`outputs.tf`):**  
- **Purpose:** Expose resource attributes for other modules/stacks.  
  ```hcl
  output "alb_dns_name" {
    description = "DNS name of the ALB"
    value       = aws_lb.main.dns_name
    # Optional: Hide secrets
    sensitive   = false 
  }
  ```
- **Critical Rule:** Outputs should be **stable and documented**. Changing an output name is a breaking change.

**Debugging Tip:**  
Use `terraform output -json` to see all outputs programmatically.

---

### **6.6 Public vs Private Modules**  
| **Public Modules**                     | **Private Modules**                          |
|----------------------------------------|----------------------------------------------|
| Hosted on **Terraform Registry**       | Hosted in **private Git repos** (GitHub, GitLab, Bitbucket) |
| Accessible to **anyone**               | Require **authentication** (SSH keys, tokens) |
| Example: `terraform-aws-modules/vpc/aws` | Example: `git::ssh://git@internal.git/vpc?ref=v2.0` |
| **No version pinning risk** (but still pin!) | **Must manage auth** (see below)          |

**Private Module Authentication:**  
- **SSH:** Use SSH keys in `~/.ssh/config` (preferred for automation).  
- **HTTPS:** Use personal access tokens (store in `~/.terraformrc`):  
  ```hcl
  credentials "app.terraform.io" {
    token = "YOUR_REGISTRY_TOKEN"
  }
  credentials "github.com" {
    token = "YOUR_GITHUB_TOKEN"
  }
  ```
- **Never hardcode tokens** in source files!

**When to Use Private:**  
- Company-specific compliance rules  
- Proprietary infrastructure patterns  
- Modules containing sensitive logic

---

### **6.7 Module Composition and Reusability**  
**Composition = Building Blocks**  
```hcl
# root/main.tf
module "network" {
  source = "./modules/network"
  cidr   = "10.0.0.0/16"
}

module "database" {
  source       = "./modules/rds"
  vpc_id       = module.network.vpc_id  # Output from network module
  subnet_ids   = module.network.private_subnet_ids
}
```

**Reusability Principles:**  
1. **Single Responsibility:** One module = one purpose (e.g., `vpc`, not `vpc_and_rds`).  
2. **Avoid Over-Parameterization:** Don’t expose every resource attribute. Expose only what callers *need*.  
3. **Idempotency:** Running the module twice with same inputs produces same result.  
4. **Stateless:** Modules should not depend on external state (except inputs/outputs).  

**Anti-Patterns to Avoid:**  
- ❌ Modules that manage unrelated resources (e.g., "app module" creating IAM + VPC + Lambda).  
- ❌ Hardcoding region/account IDs inside modules (use inputs!).  
- ❌ Optional resources (e.g., `create_rds = false` – split into separate modules).

---

### **6.8 Using Remote Modules (GitHub, Terraform Registry)**  
**From GitHub (HTTPS):**  
```hcl
module "vpc" {
  source  = "git::https://github.com/terraform-aws-modules/terraform-aws-vpc.git?ref=v3.14.0"
  version = "3.14.0" # Optional but recommended
  cidr    = "10.0.0.0/16"
}
```

**From Terraform Registry:**  
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "3.14.0"
  cidr    = "10.0.0.0/16"
}
```

**Critical Steps for Remote Modules:**  
1. **Verify the module:** Check stars, issues, and last update date.  
2. **Pin the version** (never omit `version`).  
3. **Review inputs/outputs** in the module’s `README.md`.  
4. **Check dependencies** (does it require a specific provider version?).  

**Why Registry > Raw Git:**  
- Version browsing  
- Verified publishers (e.g., `hashicorp/aws`)  
- Dependency resolution  

**Gotcha:**  
Registry modules use `source = "namespace/name/provider"`, **not** Git URLs.

---

### **6.9 Publishing Modules to Terraform Registry**  
**Prerequisites:**  
- **Public Registry:** GitHub account + public repo.  
- **Private Registry (HCP Terraform):** HCP Terraform account.  

**Step-by-Step (Public Registry):**  
1. **Structure your repo** correctly (see 6.2).  
2. **Tag a release** in Git:  
   ```bash
   git tag -a v1.0.0 -m "First release"
   git push --tags
   ```
3. **Publish to Registry:**  
   - Go to [registry.terraform.io](https://registry.terraform.io)  
   - Click "Publish" → "GitHub" → Select repo  
   - **Verification:** Terraform scans repo for `versions.tf` and structure.  

**Private Registry (HCP Terraform):**  
1. Create a "Registry Module" in HCP Terraform UI.  
2. Use `terraform login` to authenticate.  
3. Publish via CLI:  
   ```bash
   terraform registry publish -module-name=my-org/vpc/aws \
                             -namespace=my-org \
                             -provider=aws \
                             -version=1.0.0 \
                             ./path/to/module
   ```

**Non-Negotiables for Publishing:**  
- ✅ `README.md` with usage examples  
- ✅ `examples/` directory  
- ✅ `versions.tf` with constraints  
- ✅ Semantic version tags (vX.Y.Z)  

---

### **6.10 Module Versioning (SemVer)**  
**SemVer Rules for Terraform Modules:**  
| Version Part | Change Type                  | Example Change                          | Breaking? |
|--------------|------------------------------|-----------------------------------------|-----------|
| **MAJOR**    | Incompatible API changes     | Remove `subnet_ids` output              | ✅ YES    |
| **MINOR**    | Backward-compatible features | Add `ipv6_enabled` input                | ❌ NO     |
| **PATCH**    | Backward-compatible fixes    | Fix typo in README, security patch      | ❌ NO     |

**Critical SemVer Scenarios:**  
- **Adding an input with a default** = MINOR (non-breaking).  
- **Changing an input type** (e.g., `string` → `list(string)`) = MAJOR (breaking).  
- **Changing an output value** (e.g., `vpc_id` → `vpc_arn`) = MAJOR.  

**Version Constraints in Practice:**  
```hcl
# Safe: Allows patches (1.2.0 → 1.2.5)
version = "~> 1.2"

# Dangerous: Allows minors (1.2.0 → 1.3.0 – may break!)
version = ">= 1.2.0"

# Safest for production: Pin exact version
version = "1.2.3"
```

**Why SemVer Matters:**  
Without it, `terraform init` could silently upgrade to a breaking version. **Always pin versions in production.**

---

### **6.11 Module Best Practices**  
**Non-Negotiables:**  
1. **Input Validation (v1.2+):**  
   ```hcl
   variable "instance_type" {
     type    = string
     validation {
       condition     = contains(["t3.micro", "t3.small"], var.instance_type)
       error_message = "Only t3.micro or t3.small allowed."
     }
   }
   ```
2. **Comprehensive Documentation:**  
   - `README.md` must include:  
     - **Usage example** (copy-paste ready)  
     - **Input/Output tables** (with descriptions, types, defaults)  
     - **Requirements** (Terraform version, provider versions)  
     - **Examples directory structure**  
3. **Examples Directory:**  
   - `examples/simple/`: Minimal working config  
   - `examples/complete/`: All inputs configured  
   - Test these examples with `terraform validate`!  

**Advanced Best Practices:**  
- **Sensitivity:** Mark outputs with secrets as `sensitive = true`.  
- **Idempotency Tests:** Use [Terratest](https://terratest.gruntwork.io/) to verify re-runs don’t change state.  
- **Deprecations:** Use `description` to mark inputs as deprecated:  
  ```hcl
  variable "old_input" {
    description = "(DEPRECATED) Use new_input instead. Will be removed in v2.0."
  }
  ```
- **Avoid `count` in modules:** Use `for_each` for predictable resource addressing.  
- **No `terraform` blocks in modules:** Provider configuration belongs to the caller.  

**The Golden Rule:**  
> **"If it’s not in the examples directory, it doesn’t work."**  
> Users will copy-paste from examples – ensure they’re battle-tested.

---

### **Key Takeaways for Production**  
1. **Modules = Terraform’s superpower.** Without them, you’re scripting, not doing IaC.  
2. **Pin all versions** – unpinned modules are time bombs.  
3. **Public registry is for generic patterns** (VPCs, ALBs); **private registry for org-specific logic**.  
4. **SemVer is not optional** – breaking changes break production.  
5. **Documentation is code** – if it’s not documented, it doesn’t exist.  
6. **Test your examples** – they’re your module’s API contract.  

> 💡 **Pro Tip:** Run `tflint` on your modules to catch anti-patterns early. Use `terraform module gen-changelog` (community tool) to auto-generate changelogs from Git history.
