### **9.1 Introduction to Terraform Cloud (Free & Paid Tiers)**  
**What it is:**  
Terraform Cloud (TFC) is HashiCorp’s **SaaS platform** for managing Terraform workflows. Terraform Enterprise (TFE) is the **self-hosted/private cloud version** of TFC. Both provide collaboration, automation, and governance for infrastructure-as-code (IaC).  

#### **Key Tiers & Differences:**  
| **Feature**               | **Free Tier**                          | **Team & Cloud Business (Paid)**       | **Enterprise (TFE Only)**             |
|---------------------------|----------------------------------------|----------------------------------------|---------------------------------------|
| **Pricing**               | Free (limited resources)               | $20/user/month (min. 5 users)          | Custom quote (self-hosted)            |
| **Workspaces**            | Unlimited                              | Unlimited                              | Unlimited                             |
| **Private Module Registry** | ❌ Public modules only                | ✅ Full access                         | ✅ Full access                        |
| **Team Management**       | ❌ Basic teams (no RBAC)               | ✅ RBAC (Workspace/Org-level)          | ✅ Advanced RBAC                      |
| **Sentinel Policies**     | ❌                                     | ❌                                     | ✅ Policy-as-Code enforcement         |
| **Agent Pools**           | ❌                                     | ❌                                     | ✅ Self-managed runners for air-gapped envs |
| **VCS Integrations**      | ✅ GitHub/GitLab/Bitbucket (limited)   | ✅ Full VCS support                    | ✅ Full VCS support                   |
| **Remote Execution**      | ✅ (HashiCorp-runners)                 | ✅ (HashiCorp-runners)                 | ✅ (Agents or HashiCorp-runners)      |
| **SSO/SAML**              | ❌                                     | ✅ (Team tier)                         | ✅ Full enterprise SSO                |
| **Audit Logging**         | ❌                                     | ✅ (Business tier)                     | ✅ Advanced logging                   |

**Why TFC/TFE?**  
- **Centralized State:** Eliminates local `terraform.tfstate` risks (locking, corruption, secrets leakage).  
- **Collaboration:** Teams work on the same infrastructure without conflicts.  
- **Automation:** Trigger plans/applies via VCS webhooks or APIs.  
- **Governance:** Enforce policies (e.g., "No public S3 buckets") before infrastructure changes.  

> 💡 **Key Insight:** Free tier is for individuals/small teams. **Paid tiers are mandatory for enterprises** needing RBAC, Sentinel, private modules, or audit trails.

---

### **9.2 Setting Up a Workspace**  
A **Workspace** is a container for Terraform configurations, state, variables, and runs.  

#### **Step-by-Step Setup:**  
1. **Create Workspace:**  
   - In TFC/TFE UI → **Organizations** → **Workspaces** → **New Workspace**.  
   - Choose **CLI-driven** (manual `terraform login`) or **VCS-driven** (auto-trigger from Git repo).  

2. **Configure Execution Mode:**  
   - **Remote (Default):** Terraform runs execute on HashiCorp’s runners (secure, isolated).  
   - **Local:** Runs execute on your machine (rarely used; loses TFC benefits).  

3. **Set Variables:**  
   - **Terraform Variables:** Input variables for your `.tf` files (e.g., `region = "us-east-1"`).  
   - **Environment Variables:** Secrets (e.g., `AWS_ACCESS_KEY`). Mark as **"Sensitive"** to hide values.  
   - **Workspace vs. Run Variables:** Workspace vars apply to all runs; run-specific vars override them.  

4. **Advanced Settings:**  
   - **Terraform Version:** Pin exact Terraform CLI version (e.g., `1.6.6`).  
   - **Working Directory:** Subdirectory in repo (e.g., `prod/networking`).  
   - **Auto-Apply:** Skip manual approval for `terraform apply` (use cautiously!).  

> ⚠️ **Critical:** Always use **"Execution Mode = Remote"** for security. Never store state locally in production.

---

### **9.3 VCS Integration (GitHub, GitLab, Bitbucket)**  
Connects TFC/TFE to your Git repository for **automated workflows**.  

#### **How It Works:**  
1. **Link VCS Provider:**  
   - TFC → **Settings** → **VCS Providers** → Connect GitHub/GitLab/Bitbucket via OAuth.  
   - TFC creates a **dedicated OAuth app** in your VCS (with limited permissions).  

2. **Configure Workspace:**  
   - Select repo, branch (e.g., `main`), and working directory.  
   - TFC installs a **webhook** in your repo to trigger runs on `git push`.  

3. **Key Features:**  
   - **Pull Request Previews:** When a PR is opened, TFC runs `terraform plan` and posts results as a comment.  
   - **Branch-Driven Workflows:**  
     - `dev` branch → Auto-applies to dev environment.  
     - `main` branch → Requires manual approval for prod.  
   - **Run Triggers:** Workspace A (e.g., networking) triggers Workspace B (e.g., compute) on successful apply.  

#### **Permissions Required:**  
| **VCS**      | **Required Scopes**                                  |
|--------------|------------------------------------------------------|
| **GitHub**   | `repo`, `read:org`, `admin:repo_hook`                |
| **GitLab**   | `api`, `read_repository`, `write_repository`         |
| **Bitbucket**| `Account:Read`, `Repository:Write`, `Webhooks`       |

> 🔒 **Security Note:** TFC **never** sees your repo credentials. It uses short-lived OAuth tokens.

---

### **9.4 Remote State & Plan/Apply in Cloud**  
#### **Remote State Management**  
- **What Happens:**  
  - State (`terraform.tfstate`) is stored **encrypted in TFC/TFE** (not locally).  
  - Uses **state locking** (via DynamoDB-like mechanism) to prevent concurrent runs.  
  - State versions are **versioned** – roll back to any previous state.  
- **How to Configure:**  
  ```hcl
  terraform {
    backend "remote" {
      organization = "my-org"
      workspaces   = { name = "prod-app" }
    }
  }
  ```
  Run `terraform login` to authenticate.  

#### **Remote Plan/Apply Workflow**  
1. **Trigger:** `git push` → VCS webhook → TFC creates a **Run**.  
2. **Plan Phase:**  
   - TFC clones your repo, runs `terraform plan`.  
   - Outputs shown in UI with **resource diff** (e.g., `+ aws_instance.web`).  
3. **Apply Phase:**  
   - **Manual Approval:** User clicks "Confirm & Apply" in UI (default).  
   - **Auto-Apply:** Skips approval (set in workspace settings).  
   - TFC runs `terraform apply` on **isolated, ephemeral runners** (no persistent storage).  

> ✅ **Benefits:**  
> - No secrets on developer laptops.  
> - Consistent CLI/environment (no "works on my machine" issues).  
> - Audit trail of every plan/apply (who, when, what changed).

---

### **9.5 Team & Policy Management**  
#### **Team Management (RBAC)**  
- **Teams:** Groups of users (e.g., `networking-team`, `devs`).  
- **Permissions Hierarchy:**  
  ```mermaid
  graph LR
    A[Organization] --> B[Team]
    B --> C[Workspaces]
    B --> D[Modules]
    B --> E[Registry]
  ```
- **Granular Permissions:**  
  | **Role**          | **Workspace Permissions**                          |
  |-------------------|----------------------------------------------------|
  | **Viewer**        | Read-only (view state, plans)                     |
  | **Plan**          | Run plans, but not apply                           |
  | **Write**         | Plan + Apply (most common for engineers)           |
  | **Admin**         | Manage workspace settings, variables, run triggers |

#### **Policy Management (Pre-Sentinel)**  
- **Workspace Policies:** Restrict Terraform versions, enforce naming conventions via regex.  
- **Organization Policies:** Block public S3 buckets across all workspaces (using HCL constraints).  
  Example:  
  ```hcl
  # org-policy.hcl
  resource "aws_s3_bucket" "public_access" {
    required = true
    enforcement_level = "advisory"
    condition = "${self.acl}" == "public-read"
  }
  ```

> 💡 **Free Tier Limitation:** Teams are basic (no RBAC). Paid tiers enable fine-grained permissions.

---

### **9.6 Sentinel Policy-as-Code (Enterprise Only)**  
Sentinel is HashiCorp’s **policy-as-code language** to enforce governance rules.  

#### **How Sentinel Works:**  
1. **Policy Sets:** Group related policies (e.g., `security`, `cost`).  
2. **Enforcement Levels:**  
   - **Advisory:** Warn but allow apply.  
   - **Soft-Mandatory:** Apply requires override.  
   - **Hard-Mandatory:** Block apply if policy fails.  
3. **Policy Lifecycle:**  
   - **Plan-Time:** Validate against `terraform plan` (e.g., "No public IPs").  
   - **Apply-Time:** Validate against live state (e.g., "Tag compliance").  

#### **Example Policy (Block Public S3):**  
```python
import "tfplan"

# Check all S3 buckets in the plan
public_buckets = rule {
  all tfplan.resource_changes as _, rc {
    rc.type != "aws_s3_bucket" or
    (not has_field(rc.change.after, "acl") or
     rc.change.after.acl != "public-read")
  }
}

# Enforce the rule
main = rule { public_buckets }
```

#### **Key Components:**  
- **Mock Data:** Test policies against sample plans without real runs.  
- **Policy Sets:** Assign to workspaces/orgs.  
- **Overrides:** Admins can bypass policies (with audit log).  

> ⚠️ **Enterprise Exclusive:** Sentinel is **only in TFE or TFC Business tier**. Free/Team tiers use simpler HCL policies.

---

### **9.7 Private Module Registry**  
A **private repository** for your organization’s Terraform modules (vs. public [Terraform Registry](https://registry.terraform.io/)).  

#### **Why Use It?**  
- **Reusability:** Share validated modules (e.g., `aws-vpc`, `k8s-cluster`) across teams.  
- **Versioning:** Semantic versioning (e.g., `v1.2.0`) for safe upgrades.  
- **Governance:** Modules must pass Sentinel policies before publishing.  

#### **Setup Workflow:**  
1. **Publish Module:**  
   - Tag Git repo with `vX.Y.Z` (e.g., `v1.0.0`).  
   - TFC/TFE auto-discovers modules from repos named `terraform-<PROVIDER>-<NAME>`.  
2. **Consume Module:**  
   ```hcl
   module "vpc" {
     source  = "app.terraform.io/my-org/vpc/aws"
     version = "1.0.0"
   }
   ```
3. **Permissions:**  
   - Control who can publish/consume modules via **Team Access**.  

#### **Free Tier Limitation:**  
- Free tier **only supports public modules** (e.g., `hashicorp/vpc/aws`).  
- **Private modules require Team/Business tier.**

---

### **9.8 Running Terraform in CI/CD via Cloud**  
Integrate TFC/TFE into pipelines (e.g., GitHub Actions, GitLab CI) **without local Terraform**.  

#### **Two Approaches:**  
1. **Trigger Runs via API (Recommended):**  
   - CI pipeline calls TFC API to create a run → TFC handles plan/apply.  
   - **Steps:**  
     ```bash
     # In CI pipeline
     curl --header "Authorization: Bearer $TFC_TOKEN" \
          --request POST \
          --data '{"data": {"attributes": {"is_destroy":false}}}' \
          https://app.terraform.io/api/v2/runs
     ```
   - **Pros:** No Terraform CLI in CI; full audit trail in TFC.  

2. **Terraform CLI in CI (Legacy):**  
   - CI runs `terraform apply` using TFC remote backend.  
   - Requires `TF_API_TOKEN` in CI secrets.  
   - **Cons:** Secrets exposed in CI logs; loses TFC UI benefits.  

#### **Best Practices:**  
- Use **Workspace Run Triggers** (not CI) for VCS-driven workflows.  
- For complex pipelines (e.g., testing before apply), use **API-triggered runs**.  
- **Never store state locally** – always use remote backend.  

> 🔐 **Security Tip:** Use **TFC Workspace API Tokens** (scoped to one workspace) instead of user/org tokens in CI.

---

### **9.9 Agent Pools (Enterprise Only)**  
#### **What Problem Does It Solve?**  
TFC’s default runners **can’t access private networks** (e.g., on-prem datacenters, air-gapped clouds).  

#### **How Agent Pools Work:**  
1. **Agents:** Lightweight binaries you install **inside your private network**.  
2. **Agent Pools:** Logical groups of agents (e.g., `aws-us-east`, `on-prem`).  
3. **Workflow:**  
   - TFC queues a run → Assigns to an agent in the pool.  
   - Agent executes `terraform plan/apply` **inside your network**.  
   - Communicates with TFC over **outbound HTTPS** (no inbound ports).  

#### **Key Features:**  
- **Isolation:** Agents run in your infra; TFC never touches your network.  
- **Scalability:** Add agents to handle concurrency (e.g., 50 agents = 50 concurrent runs).  
- **Secrets Management:** Agents use your existing secrets (Vault, AWS Secrets Manager).  

#### **Setup Steps:**  
1. In TFE → **Organization Settings** → **Agent Pools** → **New Pool**.  
2. Download agent token and config file.  
3. Run agent binary:  
   ```bash
   terraform Enterprise agent -config=config.hcl
   ```
4. Assign workspace to use the agent pool.  

> ⚠️ **Enterprise Exclusive:** Agent pools are **only in TFE (self-hosted)** or TFC **Business tier**. Free/Team tiers use only HashiCorp runners.

---

### **Summary Cheat Sheet**  
| **Topic**               | **Free Tier**      | **Paid Tiers**     | **Enterprise Only** |  
|-------------------------|--------------------|--------------------|---------------------|  
| **Remote State**        | ✅                 | ✅                 | ✅                  |  
| **VCS Integration**     | ✅ (Limited)       | ✅                 | ✅                  |  
| **Private Modules**     | ❌                 | ✅                 | ✅                  |  
| **Team RBAC**           | ❌ (Basic teams)   | ✅                 | ✅                  |  
| **Sentinel Policies**   | ❌                 | ❌                 | ✅                  |  
| **Agent Pools**         | ❌                 | ❌ (TFC Business)  | ✅ (TFE)            |  

**When to Upgrade:**  
- **Team Tier ($20/user):** For RBAC, private modules, SSO.  
- **Business Tier ($70/user):** For Sentinel, audit logs, advanced VCS.  
- **Enterprise (TFE):** For air-gapped networks, full data residency.  
