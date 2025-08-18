### **13.1 Dynamic Blocks for Complex Nested Arguments**
* **What it is**: A mechanism to **dynamically generate nested configuration blocks** (like `ingress`, `egress`, `setting`) within a resource based on variable input. Solves the problem of static, repetitive nested blocks.
* **Why it matters**: Essential for resources requiring variable numbers of nested configurations (e.g., security groups with dynamic rules, Azure resources with complex settings).
* **How it works**:
  ```hcl
  resource "aws_security_group" "example" {
    name = "dynamic-sg"

    dynamic "ingress" {
      for_each = var.ingress_rules
      content {
        from_port   = ingress.value.from_port
        to_port     = ingress.value.to_port
        protocol    = ingress.value.protocol
        cidr_blocks = ingress.value.cidr_blocks
      }
    }

    egress {
      from_port   = 0
      to_port     = 0
      protocol    = "-1"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
  ```
* **Critical Details**:
  - **`for_each` vs `count`**: Use `for_each` with maps/sets (preferred) for stable identifiers. Avoid `count` here (causes index drift).
  - **Variable Structure**: `var.ingress_rules` must be a **map** (e.g., `{ "http" = { from_port = 80, ... } }`) or **set**. Maps allow meaningful keys (`http`, `ssh`).
  - **Nested Content**: The `content` block defines the *structure* of each generated block. Uses `ingress.value` to access map values.
  - **Validation**: Use `precondition` in modules to validate input structure:
    ```hcl
    validation {
      condition     = alltrue([for r in var.ingress_rules : r.from_port <= r.to_port])
      error_message = "from_port must be <= to_port in ingress rules."
    }
    ```
* **When to Use**:
  - Generating multiple identical nested blocks (e.g., SG rules, IAM policy statements).
  - Avoiding `count` inside resources (which causes replacement chaos).
* **Pitfalls**:
  - **Not for top-level resources**: Use `for_each` on the resource itself instead.
  - **Complexity**: Overuse makes configs hard to read. Cap at 2-3 dynamic blocks per resource.
  - **State Bloat**: Each generated block becomes a separate state entry. Avoid for *thousands* of items.

---

### **13.2 `for_each` vs `count` (When to Use Which)**
* **Core Difference**:
  | **Feature**          | `count`                          | `for_each`                          |
  |----------------------|----------------------------------|-------------------------------------|
  | **Input Type**       | Integer (`count = 3`)            | Map or Set (`for_each = {a=..., b=...}`) |
  | **Identifier**       | Numeric index (`[0]`, `[1]`)     | Meaningful key (`"a"`, `"b"`)       |
  | **Drift Risk**       | **High** (index changes break state) | **Low** (keys are stable)         |
  | **Conditional Create**| Hard (`count = var.enabled ? 1 : 0`) | Natural (`for_each = var.enabled ? {...} : {}`) |
  | **Use Case**         | Homogeneous resources (identical copies) | Heterogeneous resources (unique configs) |

* **When to Use `count`**:
  - Creating *identical* resources where order/index doesn't matter (e.g., 3 identical NAT gateways per AZ).
  - **Only if** you **cannot** use a map/set (rare). *Generally avoid*.
  ```hcl
  # Example: Identical resources where index is irrelevant
  resource "aws_instance" "nat" {
    count = length(data.aws_availability_zones.all.names)
    # ... (all instances identical)
  }
  ```

* **When to Use `for_each` (Preferred)**:
  - Resources with **unique configurations** (e.g., per-environment buckets, named subnets).
  - **Conditional creation** (filter map to `{}`).
  - **State stability** (keys persist through changes).
  ```hcl
  # Example: Environment-specific buckets
  locals {
    envs = {
      dev  = { retention = 7 }
      prod = { retention = 30 }
    }
  }

  resource "aws_s3_bucket" "env_buckets" {
    for_each = local.envs
    bucket   = "app-${each.key}-bucket"
    lifecycle_rule {
      expiration { days = each.value.retention }
    }
  }
  ```

* **Critical Best Practices**:
  1. **Never use `count` for heterogeneous resources** (causes index drift).
  2. **Use maps, not sets**: Sets lack keys for meaningful references (`aws_s3_bucket.env_buckets["dev"]`).
  3. **Filter with `for` expressions**: Conditionally create resources:
     ```hcl
     for_each = { for k, v in local.envs : k => v if v.enabled }
     ```
  4. **Avoid `count` with modules**: Modules with `count` are fragile. Use `for_each` in root module.

---

### **13.3 Conditional Resource Creation**
* **The Problem**: How to create a resource *only* under certain conditions (e.g., only in `prod`).
* **Solutions**:
  1. **`count` (Legacy, Avoid)**:
     ```hcl
     count = var.env == "prod" ? 1 : 0
     ```
     *Pitfall*: Causes index drift if condition changes later.

  2. **`for_each` with Empty Map (Recommended)**:
     ```hcl
     for_each = var.create_rds ? { "rds" = {} } : {}
     ```
     *Why better*: Stable key (`"rds"`), no state drift.

  3. **Module-Level Condition**:
     ```hcl
     module "rds" {
       source = "./rds"
       count  = var.create_rds ? 1 : 0
     }
     ```
     *Warning*: Only safe if module has **no outputs** used elsewhere (else breaks dependencies).

* **Advanced Patterns**:
  - **Conditional Nested Blocks**: Combine with `dynamic` (Section 13.1).
  - **Toggles via Variables**:
    ```hcl
    variable "create_lb" {
      type    = bool
      default = false
    }
    resource "aws_lb" "main" {
      for_each = var.create_lb ? { "main" = {} } : {}
      # ...
    }
    ```
* **Critical Rules**:
  - **Never use `count` inside resources** for conditionals. Use `for_each`.
  - **Outputs must handle absence**: Use `try()` or `length()=0` checks:
    ```hcl
    output "lb_dns" {
      value = length(aws_lb.main) > 0 ? aws_lb.main["main"].dns_name : null
    }
    ```
  - **Avoid conditional providers**: Leads to complex state management.

---

### **13.4 Meta-Arguments (`depends_on`, `lifecycle`, `providers`)**
#### **`depends_on`**
* **Purpose**: Explicitly declare **hidden dependencies** (when Terraform's graph can't infer order).
* **When to Use**:
  - Resource requires another resource to be *fully created* before it starts (e.g., Lambda needs EFS mount target `available` state).
  - **Not for**: Standard dependencies (Terraform auto-detects via references).
* **Anti-Patterns**:
  - Overuse (indicates poor design).
  - Using for resources that *should* reference outputs (e.g., `vpc_id = aws_vpc.main.id` is better than `depends_on`).
* **Example**:
  ```hcl
  resource "aws_efs_mount_target" "mt" {
    # ...
  }

  resource "aws_lambda_function" "app" {
    # ...
    depends_on = [aws_efs_mount_target.mt] # Wait for mount target to be ACTIVE
  }
  ```

#### **`lifecycle`**
* **Critical Rules**:
  - `create_before_destroy`: **Essential** for resources requiring zero-downtime replacement (e.g., load balancers, databases). *Use cautiously* (can cause quota issues).
    ```hcl
    lifecycle {
      create_before_destroy = true
    }
    ```
  - `prevent_destroy`: Protect critical resources (e.g., prod VPC). **Use sparingly** (can block legitimate changes).
  - `ignore_changes`: **Dangerous**. Only use for:
    - Attributes mutated externally (e.g., `kubernetes_deployment` pod count).
    - *Never* for core config (e.g., `ami` in EC2). Causes config drift.
    ```hcl
    lifecycle {
      ignore_changes = [metadata] # Only if metadata is managed externally
    }
    ```

#### **`providers`**
* **Purpose**: Explicitly assign a provider configuration to a resource/module.
* **When Needed**:
  - Multi-region deployments (e.g., `aws.us-east-1`, `aws.eu-west-1`).
  - Multiple accounts (e.g., `aws.management`, `aws.prod`).
* **Example**:
  ```hcl
  provider "aws" {
    alias  = "east"
    region = "us-east-1"
  }

  resource "aws_s3_bucket" "east_bucket" {
    provider = aws.east
    # ...
  }
  ```
* **Best Practice**: Declare provider aliases in root module. Avoid hardcoding in modules.

---

### **13.5 Handling Large Configurations (Module Decomposition)**
* **The Problem**: Monolithic configs become unmaintainable (>500 lines).
* **Solution: Strategic Module Decomposition**
  * **Principles**:
    1. **Single Responsibility**: One module = one logical unit (e.g., `vpc`, `eks-cluster`, `rds-instance`).
    2. **Input/Output Boundaries**: Strictly define inputs (configuration) and outputs (references for other modules).
    3. **Reusability**: Design modules for multiple consumers (e.g., `network` module used by `app` and `db`).
    4. **Avoid Over-Nesting**: Max 2-3 module layers deep. Flatter is better.

* **Decomposition Strategy**:
  ```bash
  modules/
  ├── vpc/                # Core networking
  │   ├── main.tf
  │   ├── variables.tf
  │   └── outputs.tf
  ├── eks/                # EKS cluster (depends on vpc)
  │   ├── main.tf
  │   └── ...
  └── rds/                # RDS (depends on vpc)
      └── ...
  ```

* **Critical Practices**:
  - **Versioned Modules**: Use Git tags or Terraform Registry for production modules.
  - **Input Validation**: Use `validation` blocks (Section 13.1) to enforce constraints.
  - **Avoid "Kitchen Sink" Modules**: Don't combine unrelated resources (e.g., VPC + EKS + RDS).
  - **Composition over Inheritance**: Build complex systems by *combining* small modules, not nesting deeply.
  - **State Management**: Use remote state with `terraform_remote_state` for cross-module dependencies.

* **When NOT to Module**:
  - Unique, one-off resources (e.g., a specific Lambda function).
  - Configs < 100 lines with no reuse potential.

---

### **13.6 Cross-Module Dependencies**
* **The Challenge**: Module B needs an output from Module A. How to structure without circular dependencies?
* **Correct Pattern**:
  ```hcl
  # Root module
  module "vpc" {
    source = "./modules/vpc"
    # ...
  }

  module "eks" {
    source = "./modules/eks"
    vpc_id = module.vpc.vpc_id  # Explicit dependency
    # ...
  }

  module "rds" {
    source = "./modules/rds"
    vpc_id = module.vpc.vpc_id  # Also depends on vpc
    # ...
  }
  ```
* **Key Rules**:
  1. **Outputs are Contracts**: Module outputs must be stable and well-documented.
  2. **No Direct Module-to-Module References**: *Always* route dependencies through the root module. Prevents hidden coupling.
  3. **Avoid Circular Dependencies**: If `module A` needs `module B` and vice versa, **refactor** into a shared module (e.g., `networking`).
  4. **Use `terraform_remote_state` Sparingly**: Only for *truly* separate state files (e.g., shared network in another Terraform run). Prefer direct module references.

* **Anti-Pattern**:
  ```hcl
  # modules/eks/main.tf (WRONG!)
  module "vpc" {
    source = "../vpc" # Direct module reference = hidden dependency
  }
  ```
  *Why bad*: Breaks encapsulation, makes root module unaware of dependency.

---

### **13.7 `null_resource` and `local-exec` / `remote-exec` (Use Cases & Warnings)**
* **What it is**: A resource with **no real infrastructure**, used to trigger provisioners.
* **When to Use (Rarely)**:
  - **Bootstrapping**: Initial setup before resources exist (e.g., create S3 bucket for remote state *before* `terraform init`).
  - **One-time actions**: Database schema migration *after* RDS creation.
  - **External system integration**: Trigger a webhook when infra changes.

* **Example**:
  ```hcl
  resource "null_resource" "db_migrate" {
    triggers = {
      db_endpoint = aws_db_instance.main.endpoint
      script_hash = filemd5("${path.module}/migrate.sql")
    }

    provisioner "local-exec" {
      command = "psql ${self.triggers.db_endpoint} -f migrate.sql"
    }
  }
  ```

* **Critical Warnings**:
  1. **Not for Configuration Management**: Use Ansible/Puppet/Chef instead. Terraform ≠ CM tool.
  2. **Idempotency**: `local-exec`/`remote-exec` scripts **MUST** be idempotent (safe to run repeatedly).
  3. **State Drift**: Terraform **cannot** detect changes made by provisioners. Outputs won't reflect reality.
  4. **Security Risks**: `remote-exec` uses SSH (exposes credentials). Prefer SSM Session Manager.
  5. **Destroy-Time Actions**: Use `when = destroy` cautiously (e.g., cleanup S3 before bucket delete). Risky if fails.

* **Best Practices**:
  - **Always use `triggers`**: Force re-run when inputs change (e.g., script hash).
  - **Prefer `local-exec` over `remote-exec`**: Safer, uses local machine credentials.
  - **Log Verbosely**: `command = "set -x; your_script.sh"` to debug failures.
  - **Use Alternatives First**: Cloud-init, AWS Lambda, EventBridge.

---

### **13.8 Provisioners (Best Practices & Anti-Patterns)**
* **Provisioners**: Scripts run *during* resource creation/destruction (`file`, `local-exec`, `remote-exec`).
* **When to Use (Last Resort)**:
  - **Bootstrapping**: Install cloud-init dependencies on first boot.
  - **Final Configuration**: Small tweaks not supported by native resource (e.g., set hostname).
  - **Destroy Cleanup**: Delete external data (e.g., deregister from load balancer).

* **Anti-Patterns (Avoid These)**:
  - **Managing Application State**: Deploying apps, running migrations. *Use CI/CD pipelines*.
  - **Replacing Native Terraform Features**: If AWS supports it via API (e.g., IAM policies), **don't** use `remote-exec`.
  - **Long-Running Scripts**: Provisioners timeout (default 5m). Use background services.
  - **Assuming Order**: `depends_on` doesn't guarantee sequential execution of provisioners.

* **Best Practices**:
  1. **Use `connection` securely**:
     ```hcl
     provisioner "remote-exec" {
       connection {
         type        = "ssh"
         user        = "ubuntu"
         private_key = file("~/.ssh/id_rsa")
         host        = self.public_ip
       }
       # ...
     }
     ```
  2. **Idempotency is mandatory**: Scripts must handle re-runs.
  3. **Fail Fast**: `when = failure` for diagnostics, but **never** for critical path.
  4. **Prefer `file` provisioner**: Copy config files instead of inline scripts.
  5. **Destroy Provisioners**: Only for cleanup. Test thoroughly!
     ```hcl
     provisioner "local-exec" {
       when    = destroy
       command = "aws s3 rm s3://${var.bucket} --recursive"
     }
     ```

* **Golden Rule**: If you can do it with a native Terraform resource, **DO THAT INSTEAD**.

---

### **13.9 Terraform Expressions & Functions (Advanced)**
* **Beyond Basics**: Master these for complex logic.
* **Critical Functions**:
  - **`try()`**: Safely access optional attributes/outputs:
    ```hcl
    value = try(module.optional_db.endpoint, "localhost:5432")
    ```
  - **`coalesce()`**: Return first non-null value:
    ```hcl
    value = coalesce(var.env_specific, var.default_value, "fallback")
    ```
  - **`flatten()`**: Collapse nested lists:
    ```hcl
    flatten([for subnet in aws_subnet.private : subnet.id])
    ```
  - **`setproduct()`**: Generate combinations (e.g., AZs x Instance Types):
    ```hcl
    setproduct(["us-east-1a", "us-east-1b"], ["t3.micro", "t3.small"])
    # => [["us-east-1a", "t3.micro"], ["us-east-1a", "t3.small"], ...]
    ```
  - **`regexall()`**: Extract data from strings (e.g., parse ARNs):
    ```hcl
    regexall("arn:aws:iam::123456789012:user/(.*)", aws_iam_user.example.arn)[0][0]
    ```

* **Advanced Patterns**:
  - **Conditional Maps**:
    ```hcl
    tags = merge(
      { Environment = var.env },
      var.env == "prod" ? { Critical = "true" } : {}
    )
    ```
  - **Dynamic Defaults**:
    ```hcl
    variable "instance_type" {
      type    = string
      default = "t3.${var.env == "prod" ? "large" : "small"}"
    }
    ```
  - **Error Handling**:
    ```hcl
    validation {
      condition     = can(regex("^(prod|dev)$", var.env))
      error_message = "env must be 'prod' or 'dev'."
    }
    ```

* **Pro Tips**:
  - Use `terraform console` to test expressions interactively.
  - Prefer `for` expressions over `map()`/`filter()` for readability.
  - Avoid deeply nested expressions (>3 levels). Break into `locals`.

---

### **13.10 Working with JSON & YAML (`jsonencode`, `yamlencode`)**
* **Why**: Terraform often requires JSON/YAML strings as input (e.g., IAM policies, Kubernetes manifests).
* **`jsonencode()`**:
  - Converts HCL values to **canonical JSON** (sorted keys, no whitespace).
  - **Critical for**: IAM policies, CloudWatch alarms, Lambda environment variables.
  ```hcl
  resource "aws_iam_policy" "example" {
    policy = jsonencode({
      Version = "2012-10-17"
      Statement = [
        {
          Action   = ["s3:GetObject"]
          Effect   = "Allow"
          Resource = "arn:aws:s3:::my-bucket/*"
          Sid      = "AllowRead"
        }
      ]
    })
  }
  ```
* **`yamlencode()`** (Terraform 1.2+):
  - Converts HCL to **YAML** (for Kubernetes, Ansible, etc.).
  ```hcl
  resource "kubernetes_config_map" "example" {
    data = {
      config.yaml = yamlencode({
        database = {
          host = "db.example.com"
          port = 5432
        }
        features = {
          new_ui = true
        }
      })
    }
  }
  ```

* **Key Nuances**:
  - **`null` Handling**: `jsonencode(null)` → `null` (not `"null"`). Use `try()` to avoid errors.
  - **Type Conversion**:
    - HCL `bool` → JSON `true`/`false`
    - HCL `number` → JSON number (no quotes)
    - HCL `string` → JSON string (quotes added)
  - **Order Matters**: IAM policies require specific key order (use `jsonencode` to ensure correctness).
  - **Escaping**: `jsonencode` handles quotes/backslashes automatically. **Never** manually escape.
  - **Validation**: Use `jsondecode()` in `precondition` to validate structure:
    ```hcl
    validation {
      condition     = can(jsondecode(var.policy_json))
      error_message = "Invalid JSON policy."
    }
    ```

* **When NOT to Use**:
  - For simple strings (e.g., `"{\"key\":\"value\"}"` → just use `jsonencode({key="value"})`).
  - If native resource attributes exist (e.g., `aws_iam_policy_document` data source).

---

### **Summary: Golden Rules for Advanced Terraform**
1. **Prefer Native Features**: Avoid `null_resource`/provisioners if Terraform has a resource.
2. **Stability First**: Always use `for_each` over `count` for resources.
3. **Decompose Early**: Break configs into modules *before* they become unmanageable.
4. **Outputs are Contracts**: Design module outputs carefully; treat them as APIs.
5. **Idempotency is Non-Negotiable**: Everything must be safe to re-run.
6. **State Drift is Your Enemy**: Never let provisioners modify state outside Terraform's knowledge.
7. **Test Expressions**: Use `terraform console` for complex logic.
8. **JSON/YAML via Encoding**: Never write raw JSON strings; use `jsonencode`.
