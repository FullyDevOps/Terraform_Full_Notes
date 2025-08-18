### **14.1 Securing Terraform State (Encryption at Rest, Access Control)**

**Why State Security is Critical**  
Terraform state (`terraform.tfstate`) is the **single most sensitive artifact** in your IaC workflow. It contains:
- Full resource configurations (including metadata)
- **Secrets** (even if passed via variables, they're often *stored* in state)
- Resource dependencies and relationships
- Sensitive resource attributes (e.g., database endpoints, IP addresses)

**If compromised, an attacker gains full control over your infrastructure.**

#### **A. Encryption at Rest**
*   **Problem:** Local state files (`*.tfstate`) are plaintext JSON by default. Remote state backends (like S3) store state unencrypted unless configured.
*   **Solution:**
    1.  **ALWAYS use a Remote Backend:** Never use local state for production.
        ```hcl
        # AWS S3 Backend Example (with encryption)
        terraform {
          backend "s3" {
            bucket         = "my-secure-state-bucket"
            key            = "path/to/state.tfstate"
            region         = "us-east-1"
            encrypt        = true  # CRITICAL: Enables SSE-S3
            dynamodb_table = "terraform-locks" # For state locking
          }
        }
        ```
    2.  **Enable Backend Encryption:**
        - **AWS S3:** `encrypt = true` (uses S3-managed keys - SSE-S3) **OR** specify KMS key:
          ```hcl
          kms_key_id = "arn:aws:kms:us-east-1:123456789012:key/abcd1234-a123-4567-8abc-def123456789"
          ```
        - **Azure Storage:** Enable "Secure transfer required" and "Encryption at rest" in Storage Account settings. Use `access_key` or `sas_token` securely (see 14.2).
        - **GCP Cloud Storage:** Bucket-level CMEK (Customer-Managed Encryption Key) via Cloud KMS.
        - **Terraform Cloud/Enterprise:** State is *always* encrypted at rest using AES-256-GCM.
    3.  **Rotate Encryption Keys:** Periodically rotate KMS keys used for state encryption (backend-specific process).

#### **B. Access Control**
*   **Problem:** Unrestricted access to state files allows infrastructure takeover.
*   **Solution:**
    1.  **Principle of Least Privilege (Backend IAM):**
        - **AWS S3:** Restrict S3 bucket access *only* to Terraform execution roles/users. Deny `s3:GetObjectVersion` for non-Terraform principals.
          ```json
          {
            "Version": "2012-10-17",
            "Statement": [
              {
                "Effect": "Allow",
                "Principal": {"AWS": "arn:aws:iam::123456789012:role/terraform-exec-role"},
                "Action": [
                  "s3:GetObject",
                  "s3:PutObject",
                  "s3:ListBucket"
                ],
                "Resource": [
                  "arn:aws:s3:::my-secure-state-bucket",
                  "arn:aws:s3:::my-secure-state-bucket/*"
                ]
              },
              {
                "Effect": "Deny",
                "NotPrincipal": {"AWS": "arn:aws:iam::123456789012:role/terraform-exec-role"},
                "Action": "s3:GetObject",
                "Resource": "arn:aws:s3:::my-secure-state-bucket/*"
              }
            ]
          }
          ```
        - **Azure:** Use RBAC on Storage Account (e.g., `Storage Blob Data Contributor` only for Terraform service principal).
        - **GCP:** Use IAM roles on Cloud Storage bucket (`roles/storage.objectAdmin` for Terraform service account).
    2.  **State Locking:** *Mandatory* to prevent concurrent operations corrupting state.
        - **AWS:** Use DynamoDB table (configured in `backend "s3"` block).
        - **Azure:** Use Blob Leases (enabled by default in `azurerm` backend).
        - **GCP:** Use Cloud Storage object metadata locking.
        - **Terraform Cloud/Enterprise:** Built-in locking.
    3.  **Versioning:** Enable versioning on remote state backend (S3, Azure Blob, GCS) to recover from accidental deletion/corruption. **BUT:** Old versions *also contain secrets* – manage access tightly!

**Critical Pitfall:** State files often contain **plaintext secrets** even if you use variables! Terraform *must* store the final resolved value. **Never commit state files to Git!**

---

### **14.2 Managing Secrets**

**Hardcoded Secrets are the #1 Terraform Security Anti-Pattern**

#### **A. Avoiding Hardcoded Secrets**
*   **NEVER DO THIS:**
    ```hcl
    # TERRIBLE! Secrets in code!
    resource "aws_db_instance" "prod" {
      password = "Super$3cureP@ss!" # 🔥 BURNING FIRE 🔥
    }
    ```
*   **Why it's Bad:**
    - Secrets leak into Git history, CI/CD logs, state files, `terraform plan` output.
    - Impossible to rotate without code changes.
    - Violates compliance (PCI-DSS, HIPAA, SOC 2).
*   **Immediate Action:** Scan repos with `git-secrets`, `trufflehog`, or `gitleaks` to find leaked secrets.

#### **B. Secure Secret Management Patterns**

| Tool                     | Best For                          | How Terraform Integrates                                                                 | Key Advantages                                                                 |
| :----------------------- | :-------------------------------- | :------------------------------------------------------------------------------------- | :----------------------------------------------------------------------------- |
| **HashiCorp Vault**      | **Cross-Cloud, Advanced Use Cases** | `vault` provider + `vault_*` data sources<br>```data "vault_generic_secret" "db" { path = "secret/data/db" }``` | Dynamic secrets, short TTLs, audit trails, PKI, cloud auth, leasing, revocation |
| **AWS Secrets Manager**  | **AWS-Native Workloads**          | `aws_secretsmanager_secret_version` data source<br>```data "aws_secretsmanager_secret_version" "db" { secret_id = "my-db-secret" }``` | Automatic rotation (RDS), fine-grained KMS encryption, CloudTrail logging      |
| **AWS SSM Parameter Store** | **Simple AWS Secrets (Cost Effective)** | `aws_ssm_parameter` data source<br>```data "aws_ssm_parameter" "db_pass" { name = "/prod/db/password" with_decryption = true }``` | Free tier, integrates with KMS, hierarchical paths, audit via CloudTrail       |
| **Azure Key Vault**      | **Azure-Native Workloads**        | `azurerm_key_vault_secret` data source<br>```data "azurerm_key_vault_secret" "db" { name = "db-password" key_vault_id = azurerm_key_vault.example.id }``` | HSM-backed keys, RBAC, audit logs, certificate management                      |
| **Google Secret Manager**| **GCP-Native Workloads**          | `google_secret_manager_secret_version` data source<br>```data "google_secret_manager_secret_version" "db" { secret = "db-password" }``` | IAM integration, audit logging, replication, versioning                        |

*   **Critical Implementation Steps:**
    1.  **Store Secrets ONLY in Secret Manager:** Never in `.tf`, `.tfvars`, or environment variables (unless strictly temporary in CI).
    2.  **Use Data Sources (NOT Variables):** Fetch secrets *at apply time* using data sources (examples above). Avoid `local_file` hacks.
    3.  **Restrict Secret Manager Access:**
        - Vault: Use Vault policies limiting paths.
        - AWS: Use KMS key policies + Secrets Manager resource policies.
        - Azure: Use Key Vault Access Policies + RBAC.
        - GCP: Use Secret Manager IAM roles.
    4.  **Rotate Secrets:** Automate rotation (e.g., AWS Secrets Manager rotation lambdas, Vault dynamic DB roles).
    5.  **Avoid Outputting Secrets:** Never `output` secrets from modules! Terraform Cloud/Enterprise redacts them, but local state does not.

**Pro Tip:** Use **Terraform Cloud/Enterprise's Sentinel Policies** (14.5) to *block* plans containing hardcoded secrets (e.g., regex on `password = ".*"`).

---

### **14.3 Principle of Least Privilege (IAM Roles)**

**Terraform != God Mode**

*   **Problem:** Terraform execution often uses overly permissive IAM roles (e.g., `AdministratorAccess`), enabling catastrophic damage if compromised.
*   **Solution: Granular Permissions per Environment/Module**
    1.  **Identify Required Permissions:**
        - Run `terraform plan` with a *minimal* role and note `403` errors.
        - Use tools like `iamlive` (AWS) to capture real-time permissions during a dry-run.
    2.  **Create Dedicated Roles:**
        - **Per Environment:** `prod-terraform`, `dev-terraform` (prod has stricter permissions).
        - **Per Module (Advanced):** `vpc-terraform`, `eks-terraform` (requires module-aware CI/CD).
    3.  **Craft Minimal Policies:**
        - **AWS Example (EC2 Only):**
          ```json
          {
            "Version": "2012-10-17",
            "Statement": [
              {
                "Effect": "Allow",
                "Action": [
                  "ec2:Describe*",
                  "ec2:CreateTags",
                  "ec2:RunInstances",
                  "ec2:TerminateInstances",
                  "ec2:ModifyInstanceAttribute"
                ],
                "Resource": "*",
                "Condition": {
                  "StringEquals": {
                    "aws:ResourceTag/ManagedBy": "Terraform"
                  }
                }
              }
            ]
          }
          ```
        - **Key Techniques:**
          - Use `Condition` blocks with tags (e.g., `aws:ResourceTag/Environment: prod`).
          - Restrict `Resource` ARNs where possible (e.g., specific S3 buckets, VPCs).
          - **DENY** destructive actions outside maintenance windows (use `Deny` statements with time conditions).
    4.  **Temporary Credentials:**
        - Use short-lived tokens (AWS STS `AssumeRole`, Azure AD tokens, GCP short-lived credentials).
        - Never use long-term static keys in CI/CD.
    5.  **Terraform Cloud/Enterprise:** Use its built-in role-based access control (RBAC) to limit *who* can run plans/applies.

**Golden Rule:** Terraform should **only** have permissions to manage resources it *declares*. No `iam:*`, `secretsmanager:*`, or `kms:*` unless absolutely necessary for the module.

---

### **14.4 Audit Logging & Drift Detection**

#### **A. Audit Logging**
*   **Why:** Track *who* changed *what* and *when*. Critical for forensics and compliance.
*   **Implementation:**
    1.  **Cloud Provider Logs:**
        - **AWS:** CloudTrail (filter for `eventName` = `RunInstances`, `CreateBucket`, etc.). Enable **Organization Trails** for multi-account.
        - **Azure:** Azure Activity Log + Diagnostic Settings to Log Analytics/Storage.
        - **GCP:** Cloud Audit Logs (Admin Activity, Data Access).
    2.  **Terraform Execution Logs:**
        - **Terraform CLI:** Set `TF_LOG=INFO` or `DEBUG` and capture output in CI/CD (e.g., GitHub Actions, GitLab CI). **Redact secrets!**
        - **Terraform Cloud/Enterprise:** Built-in audit log (user actions, run states, policy checks).
    3.  **State Change Logs:** Configure remote backend to log access (S3 server access logging, Azure Storage logging, GCS access logs).
*   **Critical:** Correlate Terraform run IDs (from CLI `-run-<id>` or Cloud) with CloudTrail event IDs.

#### **B. Drift Detection**
*   **Problem:** Manual changes outside Terraform break IaC's desired-state model, causing inconsistencies and failures.
*   **Solutions:**
    1.  **`terraform plan` (Manual):** The primary drift detection tool. Run regularly (e.g., nightly in CI).
        ```bash
        terraform plan -out=tfplan # Shows drift as "±" changes
        ```
    2.  **Automated Drift Detection:**
        - **Terraform Cloud/Enterprise:** Scheduled runs (e.g., daily) that trigger drift detection and alerting.
        - **Custom Scripts:** Use `terraform state list` + `terraform state show` to compare against known-good state.
        - **Cloud-Native Tools:**
          - AWS Config: Rules to detect resource configuration changes.
          - Azure Policy: `AuditIfNotExists` policies for drift.
          - GCP Security Command Center: Config Validator.
    3.  **Drift Remediation:** **NEVER** apply without understanding the drift cause! Manual changes should be:
        - Documented and approved (rarely acceptable)
        - Imported back into state (`terraform import`)
        - Or reverted via Terraform apply

**Pro Tip:** Use Sentinel policies (14.5) to **block applies** if significant drift is detected (e.g., >5 resources changed).

---

### **14.5 Terraform Sentinel Policies (Compliance & Governance)**

**Sentinel = Policy-as-Code for Terraform**

*   **What it is:** HashiCorp's proprietary policy language (in Terraform Enterprise/Cloud) to enforce rules on Terraform configurations *before* apply.
*   **Why:** Automate compliance (HIPAA, PCI), security standards, and organizational best practices.
*   **How it Works:**
    1.  Policies are written in `.sentinel` files.
    2.  Enforced during Terraform Cloud/Enterprise runs (plan phase).
    3.  Fail the run if policy is violated.

#### **Key Policy Types & Examples**
| Policy Goal                  | Sentinel Code Example                                                                 | Explanation                                                                 |
| :--------------------------- | :---------------------------------------------------------------------------------- | :-------------------------------------------------------------------------- |
| **Block Public S3 Buckets**  | ```main = rule { all s3 in tfconfig.resources.s3 as _, bucket_config | not bucket_config.public_access_block_configuration.block_public_acls }``` | Checks all S3 buckets for disabled public ACLs.                             |
| **Enforce Resource Tagging** | ```required_tags = ["Environment", "Owner"]<br>main = rule { all tfconfig.resources as _, rs | all rs as r | all required_tags as tag | exists r.config.tags[tag] }``` | Requires all resources to have specific tags.                               |
| **Restrict Instance Types**  | ```allowed_types = ["t3.micro", "t3.small"]<br>main = rule { all tfconfig.resources.aws_instance as _, inst | inst.instance_type in allowed_types }``` | Only allows small instance types in dev environments.                       |
| **Prevent Secret Leakage**   | ```import "tfconfig"<br>main = rule { not any tfconfig.resources as _, r | r.type is "aws_db_instance" and exists r.config.password }``` | Blocks plans with hardcoded `password` in RDS instances.                    |

*   **Implementation Workflow:**
    1.  Write policy in `.sentinel` file.
    2.  Associate policy with a **Policy Set** in Terraform Cloud/Enterprise.
    3.  Assign Policy Set to a **Workspace** (or Organization).
    4.  Policies run automatically during `terraform plan`.
*   **Advanced Features:**
    - **Mock Data:** Test policies against sample configurations.
    - **Policy Overrides:** Allow exceptions (with justification) for critical fixes.
    - **Soft-Mandatory Policies:** Warn but don't fail (for advisory rules).

**Limitation:** Only available in **Terraform Cloud (Team/Enterprise)** or **Terraform Enterprise**. Open-source Terraform users must use alternatives (Checkov, tfsec - see 14.6).

---

### **14.6 Static Code Analysis (Checkov, tfsec, tflint)**

**Scan Terraform Code *Before* it Runs**

*   **Why:** Catch security misconfigurations, compliance violations, and errors early in CI/CD.
*   **Tool Comparison:**

| Tool      | Focus                          | Strengths                                                                 | Weaknesses                                     | Example Command                                  |
| :-------- | :----------------------------- | :------------------------------------------------------------------------ | :--------------------------------------------- | :----------------------------------------------- |
| **Checkov** | **Broad Compliance** (CIS, PCI, HIPAA) | Huge rule library (2000+), multi-cloud, IaC-focused, SARIF output          | Can be noisy, some rules need tuning           | `checkov -d . --framework terraform`             |
| **tfsec**   | **Cloud Security**             | Fast, focused on cloud-specific risks (AWS/Azure/GCP), detailed findings  | Fewer compliance frameworks than Checkov       | `tfsec . --exclude-check=AWS001`                 |
| **tflint**  | **Syntax & Best Practices**    | Catches typos, deprecated args, region/AMI checks, plugin-aware           | Limited security focus, requires plugins       | `tflint --module --deep --aws-region=us-east-1` |

#### **How to Implement Effectively**
1.  **Integrate into CI/CD Pipeline:**
    ```yaml
    # GitHub Actions Example
    jobs:
      terraform:
        steps:
          - name: Checkov Scan
            run: |
              docker run --rm -v $(pwd):/src bridgecrew/checkov -d /src
          - name: tfsec Scan
            run: tfsec .
          - name: tflint Scan
            run: tflint --module
    ```
2.  **Customize Rules:**
    - **Exclude False Positives:** `tfsec . --exclude-check=AWS001` or `.tfsec.hcl` config.
    - **Write Custom Rules:**
      - **Checkov:** Extend with Python (custom policies).
      - **tfsec:** Write custom rules in JSON (`.tfsec/` directory).
      - **tflint:** Write rules in HCL (`.tflint.hcl`).
3.  **Fail Builds on Critical Issues:** Configure CI to fail on `HIGH` severity findings.
4.  **Scan State Files:** `tfsec` and `checkov` can scan `.tfstate` for exposed secrets/resources.
5.  **Output Formats:** Use `--format junit` for CI integration or `--output json` for reporting.

**Pro Tip:** Combine with Sentinel (14.5) – use static analysis for *pre-commit* checks and Sentinel for *pre-apply* governance in Terraform Cloud.

---

### **14.7 Preventing Accidental Destruction (`prevent_destroy`)**

**Stop Catastrophic Deletes Before They Happen**

*   **Problem:** `terraform destroy` or `terraform apply` with resource removal can delete critical infrastructure.
*   **Solution: `lifecycle { prevent_destroy = true }`**
    ```hcl
    resource "aws_db_instance" "prod" {
      # ... configuration ...
      lifecycle {
        prevent_destroy = true # 🔒 CANNOT BE DELETED BY TERRAFORM
      }
    }
    ```
*   **How it Works:**
    - Terraform will **fail the apply** if it detects an attempt to destroy the resource.
    - Output: `Error: Instance cannot be destroyed...`
*   **Critical Use Cases:**
    - Production databases
    - Root DNS zones
    - Critical IAM roles/users
    - State storage buckets (S3, etc.)
*   **Limitations & Workarounds:**
    - **Does NOT prevent manual deletion** in cloud console (use IAM Deny policies!).
    - **To destroy:** Temporarily remove `prevent_destroy`, run `terraform apply`, then re-add it.
    - **Not for modules:** Must be set *within* the resource block (cannot be set via module input).
    - **Doesn't prevent replacement:** If resource attributes change requiring replacement (e.g., `instance_type` change for stateful DB), destruction *can* still happen. Use `create_before_destroy` cautiously.
*   **Advanced Protection:**
    - **Cloud Provider Safeguards:**
      - AWS: S3 Bucket Versioning + MFA Delete, RDS Deletion Protection.
      - Azure: Resource Locks (`CanNotDelete`).
      - GCP: Resource Deletion Prevention (beta).
    - **Terraform Cloud Workspaces:** Enable "Execution Mode" = **Remote** and set "Operations" to **Manual Run** for production workspaces.

**Golden Rule:** `prevent_destroy` is a **last line of defense**, not a replacement for RBAC and process controls.

---

### **14.8 Environment Isolation & Network Security**

**Compartmentalize to Contain Breaches**

#### **A. Environment Isolation**
*   **Why:** Prevent dev mistakes from impacting prod; limit blast radius of compromises.
*   **Implementation Strategies:**
    1.  **Separate Cloud Accounts:**
        - **AWS:** Use AWS Organizations (Prod Account, Dev Account, Shared Services Account).
        - **Azure:** Use separate Azure AD Tenants or Subscriptions.
        - **GCP:** Use separate Projects within an Organization.
        - *Best Practice:* **Mandatory** for production workloads.
    2.  **Terraform Workspaces (Limited Isolation):**
        - Use `terraform workspace new dev` / `prod`.
        - *Limitation:* Same state file backend, same IAM permissions. **Not sufficient for prod isolation!**
    3.  **Directory-per-Environment:**
        - `/prod/`, `/dev/`, `/staging/` directories with separate `.tf` code.
        - *Advantage:* Enforces separate state files and backend configs.
    4.  **Terraform Cloud/Enterprise Workspaces:**
        - Dedicated workspace per environment with isolated state, variables, and access controls.

#### **B. Network Security**
*   **Critical Principles:**
    - **Zero Trust:** No implicit trust between environments/networks.
    - **Least Privilege Networking:** Only allow necessary traffic.
*   **Key Tactics:**
    1.  **VPC Design:**
        - **Per-Environment VPCs:** Prod and Dev in separate VPCs.
        - **Subnet Isolation:** Public, Private, Data subnets. Place DBs in private subnets.
        - **Network Segmentation:** Use separate VPCs for sensitive workloads (e.g., PCI).
    2.  **Firewall Rules (NSGs, Security Groups):**
        - **Default Deny:** Start with all traffic blocked.
        - **Least Privilege:** Only open ports needed (e.g., `80/443` for ALB, `22` only from jump host).
        - **Tag-Based Rules:** Use tags like `Role = WebServer` for dynamic membership.
        ```hcl
        # AWS Security Group Example
        resource "aws_security_group" "web" {
          ingress {
            from_port   = 443
            to_port     = 443
            protocol    = "tcp"
            cidr_blocks = ["0.0.0.0/0"]
          }
          egress {
            from_port   = 0
            to_port     = 0
            protocol    = "-1"
            cidr_blocks = ["0.0.0.0/0"]
          }
          tags = { Environment = "prod" }
        }
        ```
    3.  **Private Connectivity:**
        - **AWS:** VPC Peering, Transit Gateway, PrivateLink.
        - **Azure:** VNet Peering, Private Endpoints.
        - **GCP:** VPC Peering, Private Service Connect.
        - *Never* expose databases to public internet.
    4.  **Terraform Execution Network:**
        - Run Terraform from a **private subnet** (not public internet).
        - Use a dedicated "CI/CD" VPC or secure bastion host.
        - Restrict cloud API access to specific IPs (AWS VPC Endpoints, Azure Service Endpoints, GCP Private Google Access).

**Pro Tip:** Use **Sentinel policies** (14.5) to enforce network rules (e.g., "All EC2 instances must have security groups with no open SSH from 0.0.0.0/0").

---

### **Critical Cross-Cutting Themes (The Unwritten Rules)**

1.  **Never Store Secrets in Git:** Use secret managers + `.gitignore` for `.tfvars`.
2.  **Immutable Infrastructure:** Treat infrastructure as disposable; avoid manual changes.
3.  **Automate Security:** Integrate scans (14.6) into every pull request.
4.  **Least Privilege Everywhere:** Applies to IAM (14.3), network (14.8), state access (14.1), and secret access (14.2).
5.  **Assume Breach:** Design so a compromised dev environment doesn't grant prod access (14.8).
6.  **Audit Everything:** Logs (14.4) are useless if not reviewed/retained.
7.  **Test Security Policies:** Validate Sentinel/checkov rules against real misconfigurations.
