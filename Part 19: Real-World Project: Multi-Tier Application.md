### **19.1 Project Scope: VPC, Web, App, DB Tier**
**What it is:** Defining the *boundaries* and *core components* of your infrastructure. This is your architectural blueprint.

**Why it Matters:**
*   **Isolation:** Prevents a compromised web server from directly attacking the database.
*   **Security:** Enforces least-privilege access between tiers.
*   **Scalability:** Allows independent scaling of each tier (e.g., more web servers during traffic spikes).
*   **Resilience:** Failure in one tier (e.g., web) doesn't necessarily cascade to others (e.g., DB).

**The Tiers Explained (AWS Context):**

1.  **VPC (Virtual Private Cloud):**
    *   **Purpose:** Your private, isolated network "fence" in AWS. *Everything lives inside this.*
    *   **Key Components:**
        *   **CIDR Block:** Your private IP range (e.g., `10.0.0.0/16`).
        *   **Subnets:** Segments *within* the VPC. **CRITICAL DISTINCTION:**
            *   **Public Subnets:** Have a Route Table pointing to an **Internet Gateway (IGW)**. Resources here (like Web Servers) can be directly reached *from* the internet (via Load Balancer) and *can reach out* to the internet.
            *   **Private Subnets:** Have a Route Table pointing to a **NAT Gateway** (in a Public Subnet). Resources here (App Servers, DB) *cannot* be reached directly from the internet, but *can* reach the internet (for patches, etc.). **DB Subnets should be *extra* private (no NAT access if possible, use VPC Endpoints).**
        *   **Route Tables:** Define where network traffic flows (to IGW, NAT, VPC Peering, etc.).
        *   **Security Groups (SGs):** Virtual firewalls *at the instance level* (stateful). Define *allowed* inbound/outbound traffic.
        *   **Network ACLs (NACLs):** (Optional) Stateless firewalls *at the subnet level* (usually for broad network security policies).
    *   **Terraform Focus:** `aws_vpc`, `aws_subnet`, `aws_internet_gateway`, `aws_nat_gateway`, `aws_route_table`, `aws_route_table_association`. **GOTCHA:** NAT Gateways cost $$$. Place them strategically (one per AZ for HA, but consider cost vs. resilience).

2.  **Web Tier:**
    *   **Purpose:** Handles direct user traffic, serves static content, terminates SSL/TLS.
    *   **Typical AWS Resources:**
        *   **Application Load Balancer (ALB):** Distributes traffic across Web Servers. *Lives in Public Subnets.* Listens on ports 80/443.
        *   **EC2 Instances (or Auto Scaling Group - ASG):** Web servers (e.g., running Nginx, Apache). *Lives in Public Subnets.* **Security Critical:** Only allow traffic *from* the ALB's SG on port 80/443. Block direct internet access.
    *   **Terraform Focus:** `aws_lb` (ALB), `aws_lb_target_group`, `aws_lb_listener`, `aws_autoscaling_group`, `aws_launch_template`, `aws_instance`. **GOTCHA:** ALB needs Security Groups allowing 80/443 *from the internet* (or specific IPs). Web Server SG must *only* allow traffic *from* the ALB SG.

3.  **App Tier (Application Tier / Middle Tier):**
    *   **Purpose:** Business logic, dynamic content processing. Talks to Web Tier and DB Tier.
    *   **Typical AWS Resources:**
        *   **EC2 Instances (or ASG):** App servers (e.g., running Python/Node.js/Java apps). *Lives in Private Subnets.*
        *   **Security:** SG allowing inbound traffic *only* from the Web Tier SG on the app port (e.g., 8080). Outbound to DB Tier SG on DB port (e.g., 3306 for MySQL).
    *   **Terraform Focus:** `aws_autoscaling_group`, `aws_launch_template`, `aws_instance`. **GOTCHA:** App servers *must not* be publicly accessible. Verify SG rules strictly. Ensure they have outbound access to the internet (via NAT) for updates, but restrict to necessary ports/repos.

4.  **DB Tier (Database Tier):**
    *   **Purpose:** Persistent data storage. The most sensitive tier.
    *   **Typical AWS Resources:**
        *   **Amazon RDS (Relational DB):** Managed DB (e.g., MySQL, PostgreSQL, Aurora). *Lives in Private Subnets.* **Strongly Preferred over self-managed EC2 DBs.**
        *   **Security:** SG allowing inbound traffic *only* from the App Tier SG on the DB port (e.g., 3306). **NO** internet access. Consider RDS Security Groups *within* the VPC.
        *   **Optional:** RDS Subnet Group (defines which Private Subnets the DB instances launch into), Multi-AZ for HA, Read Replicas for scaling reads.
    *   **Terraform Focus:** `aws_db_instance`, `aws_db_subnet_group`. **GOTCHA:** **NEVER** put RDS in a Public Subnet. **ALWAYS** use a dedicated DB Subnet Group within Private Subnets. Enable encryption-at-rest and backup retention.

**Key Project Scope Deliverable:** A clear diagram (draw.io, Lucidchart) showing VPC, Subnets (Public/Private), Tiers, Load Balancer, and Security Group relationships. Define CIDR blocks, instance types, DB engine/version upfront.

---

### **19.2 Module Design (Network, Compute, Database)**
**What it is:** Organizing Terraform code into **reusable, self-contained units** (modules) with defined inputs and outputs. *This is non-negotiable for production.*

**Why Modules are Essential:**
*   **Reusability:** Deploy identical network layouts for Dev/Staging/Prod.
*   **Encapsulation:** Hide complex implementation details (e.g., VPC creation).
*   **Consistency:** Enforce standards across environments.
*   **Maintainability:** Fix a bug in one module, fix it everywhere.
*   **State Management:** Cleaner state files (module paths).

**Core Module Structure (Example Layout):**
```bash
project-root/
├── modules/
│   ├── network/               # VPC, Subnets, IGW, NAT, Route Tables, SGs
│   │   ├── main.tf            # Core resources
│   │   ├── variables.tf       # Inputs (cidr, azs, etc.)
│   │   ├── outputs.tf         # Outputs (vpc_id, public_subnet_ids, web_sg_id, db_sg_id, etc.)
│   │   └── ...                # (optional: locals.tf, data.tf)
│   ├── compute/               # Web & App Tier ASGs/Instances
│   │   ├── web/
│   │   │   ├── main.tf        # ALB, Target Group, ASG, Launch Template
│   │   │   ├── variables.tf   # Inputs (vpc_id, subnet_ids, web_sg_id, instance_type, etc.)
│   │   │   └── outputs.tf     # Outputs (alb_dns_name, asg_name)
│   │   └── app/
│   │       ├── main.tf        # App ASG, Launch Template
│   │       ├── variables.tf   # Inputs (vpc_id, subnet_ids, app_sg_id, db_sg_id, db_host, etc.)
│   │       └── outputs.tf
│   └── database/              # RDS
│       ├── main.tf            # DB Subnet Group, RDS Instance
│       ├── variables.tf       # Inputs (vpc_id, private_subnet_ids, db_sg_id, engine, size, etc.)
│       └── outputs.tf         # Outputs (db_address, db_port)
├── environments/
│   ├── dev/
│   │   ├── main.tf            # Root module for Dev env
│   │   ├── terraform.tfvars   # Dev-specific vars (instance_type = "t3.micro")
│   │   └── ...
│   ├── staging/
│   └── prod/
└── ...
```

**Critical Module Design Principles:**

1.  **Single Responsibility:** Each module does *one thing well* (Network, Web Compute, DB).
2.  **Clear Inputs:** Every module needs well-documented variables (`variables.tf`). *Avoid hardcoding.*
    *   *Good:* `vpc_cidr = "10.0.0.0/16"`, `instance_type = "t3.medium"`
    *   *Bad:* `vpc_cidr = "10.0.0.0/16"` hardcoded inside module.
3.  **Meaningful Outputs:** Modules *must* expose critical IDs/resources (`outputs.tf`) for other modules to consume.
    *   Network Module Outputs: `vpc_id`, `public_subnet_ids`, `private_subnet_ids`, `web_sg_id`, `app_sg_id`, `db_sg_id`
    *   Web Module Outputs: `alb_dns_name`, `web_asg_name`
    *   DB Module Outputs: `db_address`, `db_port`
4.  **Environment Agnostic:** Modules *should not* contain environment-specific logic (like `count = var.env == "prod" ? 3 : 1`). Handle scaling/size via *inputs* (`instance_count`, `instance_type`).
5.  **Versioning:** Use Git tags for modules (`?ref=v1.0.0`). **NEVER** point to `main` branch in prod!
6.  **Remote State Dependencies:** Outputs from one module (e.g., Network) become inputs to the next (e.g., Web). Terraform handles the dependency graph automatically.

**Terraform Usage (Root Module - `environments/prod/main.tf`):**
```hcl
# 1. Instantiate Network Module
module "network" {
  source = "../../modules/network"
  vpc_cidr = "10.10.0.0/16"
  azs      = ["us-east-1a", "us-east-1b", "us-east-1c"]
  # ... other inputs
}

# 2. Instantiate DB Module (Depends on Network Outputs)
module "database" {
  source          = "../../modules/database"
  vpc_id          = module.network.vpc_id
  private_subnets = module.network.private_subnet_ids
  db_sg_id        = module.network.db_sg_id
  engine          = "postgres"
  instance_class  = "db.m6g.large"
  # ... other inputs
}

# 3. Instantiate App Module (Depends on Network & DB Outputs)
module "app" {
  source           = "../../modules/compute/app"
  vpc_id           = module.network.vpc_id
  private_subnets  = module.network.private_subnet_ids
  app_sg_id        = module.network.app_sg_id
  db_sg_id         = module.network.db_sg_id
  db_host          = module.database.db_address # CRITICAL: Gets DB endpoint from DB module
  db_port          = module.database.db_port
  instance_type    = "t3.xlarge"
  # ... other inputs
}

# 4. Instantiate Web Module (Depends on Network & App Outputs)
module "web" {
  source          = "../../modules/compute/web"
  vpc_id          = module.network.vpc_id
  public_subnets  = module.network.public_subnet_ids
  web_sg_id       = module.network.web_sg_id
  app_sg_id       = module.network.app_sg_id
  app_target_group_arn = module.app.app_target_group_arn # App module must output this!
  instance_type   = "t3.medium"
  # ... other inputs
}
```

**GOTCHAS:**
*   **Circular Dependencies:** Impossible (e.g., Web needs App SG, App needs Web SG). Design SGs to depend *only* on the *next* tier's SG (Web SG -> App SG -> DB SG). Never have App SG depend on Web SG *for inbound*; Web SG allows ALB, App SG allows Web SG.
*   **State Bloat:** Large monolithic state files are fragile. Modules create cleaner state paths.
*   **Input Validation:** Use `validation` blocks in `variables.tf` (e.g., `validate { condition = length(var.azs) >= 2 }`).

---

### **19.3 Environment Management (Dev/Staging/Prod)**
**What it is:** Safely managing *identical* infrastructure configurations across different environments (Development, Staging, Production) with *controlled variations*.

**Why it Matters:** Prevents "works on my machine" disasters. Ensures Prod is tested on an environment mirroring its setup.

**Terraform Strategies (Choose ONE):**

1.  **Workspaces (Recommended for Simplicity):**
    *   **How it Works:** Terraform creates *separate state files* per workspace (`dev`, `staging`, `prod`) within *one* set of configuration files.
    *   **Implementation:**
        *   Root module (`environments/main.tf`) uses `terraform.workspace` to get current env name.
        *   Define environment-specific variables using `locals` or `tfvars` files named after workspace.
        ```hcl
        # environments/main.tf
        locals {
          # Map workspace name to instance type
          instance_type_map = {
            dev     = "t3.micro"
            staging = "t3.medium"
            prod    = "m6g.large"
          }
          instance_type = local.instance_type_map[terraform.workspace]
        }

        module "web" {
          source        = "../modules/compute/web"
          instance_type = local.instance_type
          # ... other inputs
        }
        ```
        *   Run commands: `terraform workspace new dev`, `terraform apply -var-file="dev.tfvars"`, `terraform workspace select prod`, `terraform apply -var-file="prod.tfvars"`.
    *   **Pros:** Simple, single codebase, easy to manage.
    *   **Cons:** State files are separate but *not* isolated (a `terraform destroy` in `dev` won't touch `prod` state, but misconfiguration can cause issues). Harder to enforce strict access controls per env.
    *   **GOTCHA:** **NEVER** run `terraform apply` without confirming the active workspace (`terraform workspace show`). Use a pre-command hook or prompt.

2.  **Directory-per-Environment (Recommended for Strict Isolation):**
    *   **How it Works:** Duplicate the root module directory for each environment (`environments/dev/`, `environments/staging/`, `environments/prod/`). Each has its *own* `terraform.tfstate`.
    *   **Implementation:**
        ```bash
        environments/
        ├── dev/
        │   ├── main.tf          # References modules, sets dev-specific vars
        │   └── terraform.tfvars # instance_type = "t3.micro", ...
        ├── staging/
        │   ├── main.tf
        │   └── terraform.tfvars # instance_type = "t3.medium", ...
        └── prod/
            ├── main.tf
            └── terraform.tfvars # instance_type = "m6g.large", critical = true, ...
        ```
        *   Each `main.tf` is nearly identical, differing only in variable values (often via `terraform.tfvars`).
    *   **Pros:** Maximum isolation (separate state, separate config files). Easier to enforce different IAM permissions per environment directory (via CI/CD). Simpler for large teams.
    *   **Cons:** Slightly more code duplication (mitigated by modules!). Requires discipline to keep environments in sync (use CI/CD to deploy changes sequentially: dev -> staging -> prod).
    *   **GOTCHA:** **MUST** version control *all* environment directories. Accidentally modifying `prod/main.tf` directly is a major risk – enforce changes *only* via CI/CD pipeline.

**Critical Environment Management Practices:**

*   **Progressive Promotion:** Changes flow Dev -> Staging -> Prod. Test thoroughly in Staging (which mirrors Prod) before Prod deployment.
*   **Environment-Specific Variables:** Use `terraform.tfvars` files *per environment* for differences:
    *   `instance_count` (Prod: 3, Dev: 1)
    *   `instance_type` (Prod: larger)
    *   `enable_backup` (Prod: true, Dev: false)
    *   `critical = true` (Prod only - triggers stricter approval steps in CI/CD)
*   **Strict Access Control:**
    *   **Dev:** Developers can deploy freely.
    *   **Staging:** Developers + QA can deploy; maybe requires PR merge.
    *   **Prod:** **ONLY** via CI/CD pipeline after Staging approval. Manual `terraform apply` in Prod is **FORBIDDEN**.
*   **State File Isolation:** Prod state must be in a *separate* S3 bucket (or prefix with strict IAM) from Dev/Staging. **Prod state is crown jewels.**
*   **Drift Detection:** Run `terraform plan` regularly in Prod (via CI/CD) to detect manual changes.

---

### **19.4 CI/CD Pipeline Setup**
**What it is:** Automating the testing, validation, and deployment of your Terraform infrastructure changes.

**Why it Matters:** Eliminates manual errors, enforces process (reviews, approvals), provides audit trail, enables safe & frequent deployments.

**Pipeline Stages (Typical Flow):**

1.  **Trigger:** Code pushed to `main` branch (or specific branch like `terraform-main`).
2.  **Validate & Format:**
    *   `terraform validate` (Checks syntax and basic configuration)
    *   `terraform fmt -check` (Ensures consistent style)
    *   *Fails fast* on errors.
3.  **Plan (for Target Environment):**
    *   For `dev` branch push: Plan against **Dev** environment.
    *   For `staging` branch push (or merge to `staging`): Plan against **Staging**.
    *   For `prod` branch push (or merge to `prod`): **DO NOT APPLY YET!** Plan against **Prod** and **POST PLAN OUTPUT AS COMMENT** on the PR/MR. *Requires human review.*
    *   Uses correct `terraform.tfvars` and state backend config for the target env.
    *   *Outputs the `terraform plan` diff for review.*
4.  **Approval (Prod Only):** Manual approval step in pipeline (e.g., GitHub Approvals, GitLab Merge Request approval) *after* plan review.
5.  **Apply (Target Environment):**
    *   After approval (or automatically for Dev/Staging after tests), runs `terraform apply -auto-approve` (for Dev/Staging) or just `terraform apply` (Prod after approval).
    *   **CRITICAL:** Uses `-var-file` specific to the environment.
6.  **Post-Apply Tests (Optional but Recommended):** Run smoke tests against the deployed environment (e.g., `curl http://alb-dns-name` should return 200).

**Key CI/CD Components (AWS Focus):**

*   **Remote State Backend:** **MANDATORY.** Use `backend "s3" { ... }` configured with:
    ```hcl
    terraform {
      backend "s3" {
        bucket         = "my-terraform-state-prod" # DIFFERENT BUCKET FOR PROD!
        key            = "env/prod/terraform.tfstate"
        region         = "us-east-1"
        dynamodb_table = "terraform-locks" # For state locking!
        encrypt        = true
      }
    }
    ```
    *   **DynamoDB Lock Table:** Prevents concurrent `apply`/`plan` causing state corruption. **NON-NEGOTIABLE FOR TEAMS.**
*   **Pipeline Tool:** GitHub Actions, GitLab CI, AWS CodePipeline, Jenkins.
*   **Credentials:** Use short-lived IAM roles assumed by the CI/CD service (e.g., GitHub Actions OIDC with `aws-actions/configure-aws-credentials`). **NEVER** store long-lived AWS keys in CI/CD secrets.
*   **Terraform Version:** Pin the exact Terraform version in the pipeline (`hashicorp/setup-terraform` action).

**Example GitHub Actions Snippet (Prod Apply Stage):**
```yaml
jobs:
  deploy_prod:
    runs-on: ubuntu-latest
    environment: production # GitHub Env w/ approval rules
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActions-Prod-Deploy
          aws-region: us-east-1

      - name: Terraform Init
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.6.5
        run: terraform init -backend-config="bucket=my-terraform-state-prod"

      - name: Terraform Plan
        run: terraform plan -var-file="environments/prod/terraform.tfvars"
        # Output captured & posted as PR comment by previous step

      - name: Terraform Apply
        run: terraform apply -auto-approve -var-file="environments/prod/terraform.tfvars"
        # Only runs AFTER manual approval on the GitHub Environment
```

**GOTCHAS:**
*   **State Locking Failure:** If DynamoDB lock fails, `terraform apply` will error. Fix the lock table permissions!
*   **Accidental Prod Apply:** Double-check the environment name in the pipeline config and state backend config. Use distinct bucket names.
*   **No Plan Review:** Never auto-apply to Prod without a human-reviewed plan diff. This is your last safety net.
*   **Secrets in Plan Output:** Ensure Terraform variables containing secrets (DB passwords) use `sensitive = true` and are *not* output in plan logs. Use AWS Secrets Manager/Parameter Store instead of Terraform vars for secrets.

---

### **19.5 Security Hardening (Security Groups, IAM)**
**What it is:** Applying the **principle of least privilege** at every layer to minimize attack surface.

**Why it Matters:** Terraform makes it *easy* to deploy insecure configs. Hardening is not optional for production.

**Critical Hardening Areas:**

1.  **Security Groups (The #1 Mistake Area):**
    *   **Web Tier (ALB):**
        *   **Inbound:** ONLY `0.0.0.0/0` on `443` (HTTPS), `80` (HTTP - redirect to HTTPS). *Avoid broad 0.0.0.0/0 if possible (use CloudFront IP ranges).*
        *   **Outbound:** ONLY to App Tier SG on App Port (e.g., `8080`).
    *   **Web Tier (Instances - behind ALB):**
        *   **Inbound:** ONLY from ALB SG on `80`/`443` (or App Port if ALB talks directly to instances). **NEVER `0.0.0.0/0` here!**
        *   **Outbound:** ONLY to App Tier SG on App Port.
    *   **App Tier:**
        *   **Inbound:** ONLY from Web Tier SG on App Port (e.g., `8080`).
        *   **Outbound:** ONLY to DB Tier SG on DB Port (e.g., `3306`). *Optionally:* Allow outbound to `0.0.0.0/0` on `443` *only* for specific package repos (use NACLs or Security Group egress rules tightly).
    *   **DB Tier (RDS):**
        *   **Inbound:** ONLY from App Tier SG on DB Port (e.g., `3306`). **NEVER `0.0.0.0/0`!**
        *   **Outbound:** Typically `0.0.0.0/0` (for patches) but restrict to necessary ports/repos if possible.
    *   **General SG Best Practices:**
        *   **Name SGs Clearly:** `sg-web-alb`, `sg-app-servers`, `sg-rds-mysql`.
        *   **Use SG References:** `security_groups = [module.network.web_sg_id]` NOT hardcoded IDs.
        *   **Avoid "Default" SG:** Create explicit SGs for everything.
        *   **Minimize Rules:** One rule per *necessary* connection. Don't open port `0-65535`.
        *   **Use Prefix Lists:** For AWS service IPs (e.g., `com.amazonaws.us-east-1.s3`).

2.  **IAM (Identity and Access Management):**
    *   **Terraform Execution Role:**
        *   **Principle:** Terraform should run with the *absolute minimum* permissions needed to deploy the *current* configuration.
        *   **How:** Create IAM Roles *per environment* (`Terraform-Dev-Role`, `Terraform-Prod-Role`). Attach *only* the necessary policies.
        *   **Policy Example (Prod - VERY TIGHT):**
            ```json
            {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Action": [
                            "ec2:Describe*",
                            "ec2:CreateTags",
                            "ec2:DeleteTags",
                            "ec2:RunInstances", // Only if creating EC2 directly
                            "ec2:TerminateInstances", // Only if destroying
                            "elasticloadbalancing:Describe*",
                            "elasticloadbalancing:Create*",
                            "rds:Describe*",
                            "rds:CreateDBInstance",
                            "rds:ModifyDBInstance",
                            "s3:GetObject",
                            "s3:PutObject",
                            "dynamodb:GetItem",
                            "dynamodb:PutItem"
                        ],
                        "Resource": "*"
                    },
                    {
                        "Effect": "Allow",
                        "Action": "sts:AssumeRole",
                        "Resource": "arn:aws:iam::123456789012:role/terraform-assumable-role-*" // For modules needing roles
                    }
                ]
            }
            ```
        *   **NEVER** use `AdministratorAccess` for Terraform! Start restrictive and add only what `terraform plan` complains about.
    *   **Resource IAM Roles (for EC2/RDS):**
        *   **EC2 Instances (Web/App):** Use Instance Profiles (`aws_iam_instance_profile`). Attach roles with *only* permissions the app needs (e.g., `s3:GetObject` for specific bucket, `ssm:SendCommand` for SSM). **Avoid `PowerUserAccess`!**
        *   **RDS:** Enable IAM Database Authentication *if* your app supports it (more secure than password). Otherwise, use Secrets Manager for passwords.
    *   **State Bucket Permissions:** Prod state bucket should have:
        *   Block Public Access: **ON**
        *   Bucket Policy: Deny all except specific IAM roles (Terraform Prod role, maybe backup role). Encrypt with KMS CMK.

3.  **Other Critical Hardening:**
    *   **Enable VPC Flow Logs:** Capture network traffic for security analysis. Send to CloudWatch Logs or S3.
    *   **Encrypt Everything:** EBS volumes (default now), RDS (enable), S3 buckets (SSE-S3 or SSE-KMS). Use KMS CMKs for prod.
    *   **Disable Public IP for Private Instances:** `associate_public_ip_address = false` in Launch Template/ASG for App/DB tiers.
    *   **Use Latest AMIs:** Reference AMIs by name via `data "aws_ami" {}` with filters (e.g., `most_recent = true`, `owners = ["amazon"]`, `name_regex = "amzn2-ami-hvm-2.0.*-x86_64-gp2"`).
    *   **Secrets Management:** **NEVER** store DB passwords in Terraform code/state. Use:
        *   `aws_secretsmanager_secret` + `aws_secretsmanager_secret_version` (Terraform creates secret, outputs ARN).
        *   App servers retrieve secrets at startup via Secrets Manager API (using their Instance Profile role).
        *   *Alternative:* Parameter Store (SSM) with `SecureString`.

**GOTCHAS:**
*   **Overly Permissive SGs:** The #1 cloud breach vector. Audit SGs monthly.
*   **Terraform Running as Root:** Using root credentials or overly permissive roles for Terraform.
*   **Hardcoded Secrets:** `db_password = "P@ssw0rd!"` in `.tf` files. Terraform state is *not* secret-safe.
*   **Unencrypted State:** S3 bucket without encryption. Enable `encrypt = true` in backend config.
*   **Missing Flow Logs:** Flying blind on network activity.

---

### **19.6 Monitoring & Alerting (CloudWatch Integration)**
**What it is:** Proactively observing infrastructure health and performance, and getting notified of issues *before* users do.

**Why it Matters:** "If you aren't monitoring it, it's broken and you don't know it." Essential for uptime and performance.

**Core Monitoring Targets:**

1.  **Infrastructure Health:**
    *   **EC2 ASGs:** `HealthyHostCount` (should equal `DesiredCapacity`), `UnHealthyHostCount`.
    *   **ALB:** `HTTPCode_ELB_5XX`, `HTTPCode_Backend_5XX`, `TargetResponseTime`, `HealthyHostCount`.
    *   **RDS:** `CPUUtilization`, `FreeStorageSpace`, `FreeableMemory`, `ReadIOPS`, `WriteIOPS`, `DatabaseConnections`, `ReplicaLag`.
    *   **General:** `StatusCheckFailed` (System + Instance) for EC2.

2.  **Application Performance:**
    *   **Custom Metrics:** Push app-specific metrics via CloudWatch Agent or SDK (e.g., `app_errors_per_minute`, `queue_length`).

**Terraform Implementation (CloudWatch Alarms):**

```hcl
# Example: CPU Utilization Alarm for App ASG (Prod)
resource "aws_cloudwatch_metric_alarm" "app_cpu_high" {
  alarm_name          = "prod-app-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2" # Or AWS/ApplicationELB, AWS/RDS
  period              = "300"     # 5 minutes
  statistic           = "Average"
  threshold           = "80"
  unit                = "Percent"

  dimensions = {
    AutoScalingGroupName = module.app.asg_name # Get ASG name from App module output!
  }

  alarm_actions = [aws_sns_topic.prod_alerts.arn] # Send to SNS Topic
  ok_actions    = [aws_sns_topic.prod_alerts.arn]
}

# SNS Topic for Alerts (Prod)
resource "aws_sns_topic" "prod_alerts" {
  name = "prod-infrastructure-alerts"
}

# Subscription to send alerts to email (or Lambda/Slack/etc.)
resource "aws_sns_topic_subscription" "email_alert" {
  topic_arn = aws_sns_topic.prod_alerts.arn
  protocol  = "email"
  endpoint  = "ops-team@yourcompany.com"
}
```

**Key Monitoring Components:**

1.  **CloudWatch Alarms:** Define thresholds and actions (SNS notification).
2.  **SNS Topic:** Central notification hub. Subscribe email, Lambda (for Slack/Teams), PagerDuty.
3.  **CloudWatch Dashboards:** Visualize key metrics (CPU, Latency, Errors) in one place. (Can be Terraform'd via `aws_cloudwatch_dashboard`).
4.  **CloudWatch Logs:** Aggregate logs from EC2 (via CloudWatch Agent), ALB, RDS. Use Insights for querying.
5.  **Optional - X-Ray:** For distributed tracing of app requests (requires code changes).

**Critical Alerting Practices:**

*   **Meaningful Thresholds:** Don't alert on `CPU > 50%` if 70% is normal peak. Base on historical data.
*   **Alert Tiers:**
    *   **P1 (Critical):** Site down, data loss (e.g., `HealthyHostCount = 0` on ALB). Page 24/7.
    *   **P2 (High):** Degraded performance (e.g., `TargetResponseTime > 2s`). Slack/email during business hours.
    *   **P3 (Medium):** Warning (e.g., `FreeStorageSpace < 20%`). Email digest.
*   **Avoid Alert Fatigue:** Too many alerts = ignored alerts. Tune aggressively. Use SNS filter policies.
*   **Test Alerts:** Run `terraform apply` on an alarm with a low threshold to verify notifications work.

**GOTCHAS:**
*   **Missing Dimensions:** Alarm without correct `AutoScalingGroupName` or `DBInstanceIdentifier` won't trigger.
*   **No Notification Channel:** Alarm set but no SNS subscription = silent failure.
*   **Ignoring OK Actions:** Not sending "resolved" notifications causes confusion.
*   **Not Monitoring State File Bucket:** Monitor `BucketSizeBytes` and `NumberOfObjects` for your Terraform state bucket!

---

### **19.7 Documentation & READMEs**
**What it is:** Creating clear, actionable documentation so anyone (including Future You) can understand, deploy, and troubleshoot the infrastructure.

**Why it Matters:** "Works on my machine" infrastructure is useless. Documentation is part of the deliverable. Critical for onboarding and disaster recovery.

**Essential Documentation Components:**

1.  **Project README.md (Root of Repo):**
    *   **Purpose:** One-stop shop for *everything*.
    *   **MUST Include:**
        *   **Overview:** What this project deploys (Multi-tier app: VPC, Web, App, DB).
        *   **Architecture Diagram:** Link to or embed the diagram (draw.io source + PNG).
        *   **Prerequisites:** AWS account, Terraform >=1.6, `aws` CLI configured, S3 bucket for state created.
        *   **Getting Started (Dev):**
            ```bash
            cd environments/dev
            terraform init -backend-config="bucket=dev-tfstate-bucket"
            terraform plan -var-file="terraform.tfvars"
            terraform apply -var-file="terraform.tfvars"
            ```
        *   **Environment Management:** How to deploy to Staging/Prod (via CI/CD link or commands).
        *   **Key Outputs:** `terraform output` after apply (ALB DNS, DB endpoint).
        *   **Security Notes:** How secrets are managed, IAM roles used.
        *   **Troubleshooting:** Common errors (`State lock acquired by X`, `Invalid SG reference`) and fixes.
        *   **Contributing:** Branching strategy, PR process, required checks.
        *   **Important Links:** CI/CD pipeline URL, CloudWatch Dashboard URL, Slack channel.

2.  **Module READMEs (`modules/*/README.md`):**
    *   **Purpose:** Document *each reusable module*.
    *   **MUST Include:**
        *   **What it does:** "Creates a VPC with Public/Private subnets, NAT, IGW, and tier SGs".
        *   **Inputs Table:** Every variable (`name`, `description`, `type`, `default`, `required`).
        *   **Outputs Table:** Every output (`name`, `description`).
        *   **Example Usage:** Snippet of how to call the module.
        *   **Requirements:** AWS provider version, other modules it depends on.

3.  **Environment READMEs (`environments/*/README.md`):**
    *   **Purpose:** Document environment-specific quirks.
    *   **MUST Include:**
        *   **Purpose:** "Production environment - handles live customer traffic".
        *   **Critical Variables:** Why `instance_type = "m6g.2xlarge"`? What's the backup retention?
        *   **Deployment Process:** *Exactly* how to deploy (CI/CD trigger, manual command if allowed).
        *   **Contacts:** Who owns this environment? (Slack handle, email)
        *   **Special Notes:** "This env uses KMS key `prod-key` for RDS encryption".

4.  **Runbook / Troubleshooting Guide (`docs/troubleshooting.md`):**
    *   **Purpose:** Step-by-step fixes for known issues.
    *   **Examples:**
        *   "ALB returning 502: Check App SG allows traffic from ALB SG, check App server logs, verify Target Group health checks".
        *   "Terraform state locked: `aws dynamodb delete-item --table-name terraform-locks --key '{\"LockID\": {\"S\": \"<ID>\"}}'`".
        *   "RDS connection timeout: Verify DB SG allows App SG, check NACLs, ensure app uses correct endpoint".

**Documentation Best Practices:**

*   **Keep it Updated:** Documentation rot is worse than no documentation. Make updating docs part of the PR process.
*   **Be Specific:** "Deploy to Prod" is bad. "Merge to `prod` branch triggers CI/CD pipeline; requires 2 approvals" is good.
*   **Include Commands:** Copy-pasteable commands are gold.
*   **Use Diagrams:** A picture is worth 1000 words (and 1000 `terraform graph` outputs).
*   **Audience:** Write for a new hire with AWS/Terraform basics, not for yourself 6 months ago.
*   **Location:** All docs live *in the repo* (not Confluence/wiki that might get lost). Versioned with code.

**GOTCHAS:**
*   **"I'll document it later":** It never happens. Document as you build.
*   **Outdated Diagrams:** The diagram shows 2 AZs, but code uses 3. Causes confusion.
*   **Missing Critical Steps:** "Just run `terraform apply`" (but forgot you need the state bucket created first).
*   **No Troubleshooting Guide:** Team wastes hours on known issues.

---

## **Your Action Plan (Summary)**

1.  **Scope Rigorously:** Define VPC CIDR, Subnets, Tiers, Security Groups *on paper* first.
2.  **Build Modules:** Network -> DB -> App -> Web. Test modules in isolation (Dev).
3.  **Isolate Environments:** Use Directory-per-Env or Workspaces *with strict controls*. Prod state is sacred.
4.  **Automate Deployment:** CI/CD pipeline with Plan Review for Prod. **No manual Prod deploys.**
5.  **Harden Relentlessly:** SGs = minimal rules, IAM = least privilege, Secrets = Secrets Manager, Encrypt everything.
6.  **Monitor Proactively:** CloudWatch Alarms -> SNS -> Slack/Email. Test alerts.
7.  **Document Obsessively:** READMEs for Project, Modules, Environments, and Troubleshooting. Keep it updated.

**Remember:** Terraform is infrastructure *as code*. Treat it like application code: version control, code reviews, testing, CI/CD, and documentation are **mandatory** for production.
