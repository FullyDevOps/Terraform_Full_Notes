### **16.1 CI/CD Pipeline Overview for Terraform**
**Why CI/CD for IaC?**  
Traditional CI/CD focuses on *application code*. With Infrastructure-as-Code (IaC), **infrastructure changes become code changes**, requiring the same rigor:
- **Prevent "Drift"**: Manual `terraform apply` causes untracked state changes.
- **Enforce Governance**: Mandate peer review, testing, and approvals.
- **Reduce Risk**: Catch errors *before* they hit production.
- **Auditability**: Full traceability from commit → infrastructure change.

**Core Pipeline Stages for Terraform:**
1. **Validate**: `terraform validate` (syntax, config errors).
2. **Format Check**: `terraform fmt -check` (enforce style consistency).
3. **Plan**: `terraform plan` (generate execution plan, *never* auto-apply here).
4. **Security Scan**: Tools like `tfsec`, `checkov`, or `snyk iac` (identify misconfigurations).
5. **Approval Gate**: Human review of the plan (critical for production).
6. **Apply**: `terraform apply` (only after approval).
7. **Post-Apply Tests**: Validate infrastructure works (e.g., `curl` endpoints, smoke tests).

**Key Differences from App CI/CD:**
- **State Management**: Terraform state (`terraform.tfstate`) must be secured and locked during operations.
- **Idempotency**: Plans must be repeatable; avoid non-deterministic values.
- **Destructive Changes**: `terraform plan` must explicitly show `destroy` actions for review.
- **Secrets Handling**: Never store secrets in plan outputs; use vaults (HashiCorp Vault, AWS Secrets Manager).

> 💡 **Critical Best Practice**: *Never* automate `apply` for production without manual approval. Treat infrastructure like financial transactions.

---

### **16.2 GitHub Actions for Terraform**
**Why GitHub Actions?**  
Tight integration with GitHub PRs, free for public repos, and easy YAML configuration.

**Workflow Structure (`.github/workflows/terraform.yml`):**
```yaml
name: Terraform CI/CD

on:
  pull_request: # Trigger on PRs (Dev/Staging)
    paths:
      - 'terraform/**'
  push:
    branches: [main] # Trigger on main (Prod)

jobs:
  terraform:
    runs-on: ubuntu-latest
    environment: ${{ github.ref == 'refs/heads/main' && 'production' || 'development' }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.6.6

      - name: Terraform Init
        run: cd terraform && terraform init
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Terraform Plan
        if: github.event_name == 'pull_request'
        id: plan
        run: cd terraform && terraform plan -out=tfplan
        continue-on-error: true # Failures are expected if plan has errors

      - name: Upload Plan
        if: github.event_name == 'pull_request'
        uses: actions/upload-artifact@v3
        with:
          name: tfplan
          path: terraform/tfplan

      - name: Terraform Apply
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        run: cd terraform && terraform apply -auto-approve tfplan
```

**Key Features:**
- **Environments**: Link GitHub Environments (e.g., `production`) to secrets and required reviewers.
- **PR Integration**: Plan output automatically posted as PR comment (use `hashicorp/terraform-github-actions`).
- **Concurrency Control**: Use `concurrency: ${{ github.workflow }}-${{ github.ref }}` to prevent parallel runs.
- **Secrets**: GitHub Secrets encrypted at rest; never exposed in logs.

**Pitfalls to Avoid:**
- ❌ Hardcoding secrets in workflow files.
- ❌ Skipping `terraform validate`/`fmt` checks.
- ❌ Allowing `apply` on PRs (only `plan` should run on PRs).

---

### **16.3 GitLab CI/CD with Terraform**
**Why GitLab CI/CD?**  
Native integration with GitLab, built-in Terraform state storage, and merge request widgets.

**Pipeline Configuration (`.gitlab-ci.yml`):**
```yaml
stages:
  - validate
  - plan
  - apply

validate:
  stage: validate
  image: hashicorp/terraform:1.6.6
  script:
    - terraform validate
    - terraform fmt -check
  rules:
    - if: $CI_MERGE_REQUEST_ID

plan:
  stage: plan
  image: hashicorp/terraform:1.6.6
  script:
    - terraform init
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - tfplan
  rules:
    - if: $CI_MERGE_REQUEST_ID

apply-prod:
  stage: apply
  image: hashicorp/terraform:1.6.6
  script:
    - terraform init
    - terraform apply -auto-approve tfplan
  environment: production
  when: manual # Requires manual approval
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

**GitLab-Specific Advantages:**
- **Terraform State Storage**: Use GitLab's built-in state backend (`terraform { backend "http" { ... } }`).
- **Merge Request Widgets**: Plan output embedded directly in MR UI.
- **Approval Rules**: Require approvals from specific groups before `apply`.
- **Protected Environments**: Restrict `apply` to users with "Maintainer" role.

**Critical Setup Steps:**
1. Store Terraform state in GitLab:  
   ```hcl
   terraform {
     backend "http" {
       address = "https://gitlab.com/api/v4/projects/<PROJECT_ID>/terraform/state/<STATE_NAME>"
       lock_address = "${address}/lock"
       unlock_address = "${address}/lock"
       username = "gitlab-ci-token"
       password = "$CI_JOB_TOKEN"
       lock_method = "POST"
       unlock_method = "DELETE"
       retry_wait_min = "5"
     }
   }
   ```
2. Configure **Protected Environments** for `production` with approval rules.

---

### **16.4 Jenkins Pipeline for Terraform**
**Why Jenkins?**  
Enterprise-grade scalability, extensive plugin ecosystem, and complex workflow orchestration.

**Declarative Pipeline (`Jenkinsfile`):**
```groovy
pipeline {
  agent any
  environment {
    TF_STATE = "s3://my-terraform-state/${ENVIRONMENT}/terraform.tfstate"
  }
  stages {
    stage('Validate') {
      steps {
        sh 'terraform validate'
        sh 'terraform fmt -check'
      }
    }
    stage('Plan') {
      when { expression { env.BRANCH_NAME != 'main' } }
      steps {
        sh 'terraform init -backend-config="bucket=my-terraform-state"'
        sh 'terraform plan -out=tfplan'
        archiveArtifacts 'tfplan'
      }
    }
    stage('Apply to Prod') {
      when { expression { env.BRANCH_NAME == 'main' } }
      steps {
        input { message "Apply to PRODUCTION?" }
        sh 'terraform init -backend-config="bucket=my-terraform-state"'
        sh 'terraform apply -auto-approve tfplan'
      }
    }
  }
  post {
    failure {
      slackSend channel: '#infra-alerts', message: "Terraform failed in ${env.JOB_NAME}"
    }
  }
}
```

**Jenkins-Specific Power Features:**
- **Shared Libraries**: Reuse Terraform logic across pipelines (e.g., `@Library('terraform-lib') _`).
- **Approval Gates**: Integrate with Jenkins' `input` step or tools like **Jira Service Management**.
- **Parallel Environments**: Run plans for dev/staging in parallel:
  ```groovy
  parallel {
    stage('Dev Plan') {
      steps { ... }
    }
    stage('Staging Plan') {
      steps { ... }
    }
  }
  ```
- **Audit Trail**: Jenkins logs capture *who* approved changes.

**Must-Have Plugins:**
- **Terraform Plugin**: Syntax highlighting, version management.
- **Lockable Resources**: Prevent concurrent state access.
- **HashiCorp Vault Plugin**: Inject secrets dynamically.

---

### **16.5 Pull Request Workflow with Plan**
**The Gold Standard for Safety**  
*Never apply changes without reviewing the plan.*

**How It Works:**
1. Developer opens PR with Terraform changes.
2. CI pipeline runs `terraform plan` automatically.
3. Plan output is **posted as a PR comment** (e.g., via GitHub Actions or GitLab MR widget).
4. Team reviews:
   - ✅ Expected resource changes
   - ❌ Unexpected `destroy` actions
   - ⚠️ Security warnings from `tfsec`
5. PR approved → Merged → Triggers production pipeline (with approval gate).

**Example PR Comment (GitHub):**
```
Terraform Plan for `dev` environment:
+ 3 resources to add
~ 1 resource to update
- 0 resources to destroy

Security Scan (tfsec):
[OK] No critical issues found

Plan file: [Download tfplan](https://.../artifacts/tfplan)
```

**Critical Configuration:**
- **Mask Secrets**: Use `TF_LOG=error` to hide sensitive values in logs.
- **Cost Estimation**: Integrate `infracost` to show cost impact in PRs.
- **Auto-Comment**: Tools like [Atlantis](#168-using-atlantis-for-automated-terraform-in-git) automate this.

> ⚠️ **Never** show full plan output if it contains secrets (e.g., RDS passwords). Use `terraform plan -compact-warnings` and scrub logs.

---

### **16.6 Automated apply with Approval Gates**
**Balancing Automation & Safety**  
*Automate everything except production applies.*

**Implementation Patterns:**
| **Tool**       | **Approval Mechanism**                                  | **Example**                                                                 |
|----------------|---------------------------------------------------------|-----------------------------------------------------------------------------|
| **GitHub**     | Environment protection rules + Required reviewers       | Require 2 approvals from "Infra Team" before deploying to `production` env. |
| **GitLab**     | Merge request approvals + Protected environments        | `apply-prod` job requires 1 approval from "Maintainers" group.              |
| **Jenkins**    | `input` step + Slack/email integration                  | `input message: 'Approve PROD deploy?', submitter: 'infra-team'`            |
| **Generic**    | Webhook to Slack/MS Teams with approval buttons         | Use **Shubot** or **Jira Service Management** for approvals.                |

**Approval Gate Checklist:**
1. Plan shows **no unexpected destroys**.
2. Security scan passed (0 critical issues).
3. Cost impact is acceptable (via `infracost`).
4. Compliance checks passed (e.g., AWS Config rules).
5. Rollback plan documented (e.g., `terraform state rm` commands).

**Advanced: Time-Locked Approvals**  
Require approvals to be valid for ≤ 24 hours to prevent stale approvals.

---

### **16.7 Environment Promotion (Dev → Staging → Prod)**
**Avoid "Snowflake Environments"**  
Use **identical code** across environments with parameterization.

**Strategy 1: Folder-per-Environment (Recommended)**
```
terraform/
├── dev/
│   ├── main.tf
│   └── terraform.tfvars
├── staging/
│   ├── main.tf (symlink to ../modules)
│   └── terraform.tfvars
└── prod/
    ├── main.tf (symlink to ../modules)
    └── terraform.tfvars
```
- **Pros**: Simple, explicit, easy to isolate state.
- **Cons**: Slight duplication; fix via symlinks to shared modules.

**Strategy 2: Workspaces (Use with Caution)**
```hcl
# main.tf
resource "aws_instance" "app" {
  count = terraform.workspace == "prod" ? 3 : 1
}
```
- **Pros**: Single codebase.
- **Cons**: Hard to manage complex differences; state isolation issues.

**Promotion Workflow:**
1. PR to `dev` → Auto-apply on merge.
2. PR from `dev` to `staging` → Manual approval → Apply.
3. PR from `staging` to `main` (prod) → Strict approval → Apply.

**Critical Requirements:**
- **State Isolation**: Separate state files for each environment (e.g., `dev/terraform.tfstate`).
- **Immutable Artifacts**: Pin Terraform module versions (no `latest`).
- **Drift Detection**: Run `terraform plan` daily in prod to detect manual changes.

> 💡 **Golden Rule**: *Promote the exact same commit hash* from dev → staging → prod.

---

### **16.8 Using Atlantis for Automated Terraform in Git**
**What is Atlantis?**  
Open-source tool that **enforces Terraform workflows via PRs** (no custom CI needed).

**How It Works:**
1. Developer opens PR with `.tf` changes.
2. Atlantis (running as a server) detects PR → runs `terraform plan`.
3. Plan output posted as **PR comment**.
4. Team reviews/approves PR.
5. On merge, Atlantis runs `terraform apply`.

**Key Features:**
- **Automatic State Locking**: Uses backend lock (S3, Consul).
- **Policy as Code**: Integrate with Sentinel/Open Policy Agent.
- **Multi-Project Support**: Handle multiple Terraform configs in one repo.
- **Custom Workflows**: Define per-repo workflows (e.g., custom plan commands).

**Minimal `atlantis.yaml` Configuration:**
```yaml
version: 3
projects:
  - dir: terraform/prod
    terraform_version: v1.6.6
    autoplan:
      when_modified: ["*.tf", "../modules/**.tf"]
    apply_requirements: [approved, mergeable] # Require PR approval
```

**Setup Steps:**
1. Deploy Atlantis (Kubernetes, EC2, or managed service like **Runatlantis**).
2. Configure GitHub/GitLab webhook to Atlantis server.
3. Add `atlantis.yaml` to repo root.

**Why Teams Love Atlantis:**
- Eliminates custom CI pipeline complexity.
- PR-centric workflow matches developer habits.
- Built-in security (no secrets stored in Atlantis).

**Limitations:**
- Less flexible for complex pipelines (e.g., pre-apply tests).
- Requires server maintenance (vs serverless GitHub Actions).

---

### **16.9 Integrating with Argo CD / Flux (GitOps)**
**Terraform + GitOps: The Perfect Pair**  
- **Terraform**: Manages *cloud infrastructure* (VPCs, clusters, DBs).
- **Argo CD/Flux**: Manages *application deployments* on the infrastructure.

**Why Combine Them?**
| **Layer**          | **Tool**       | **Responsibility**                          |
|--------------------|----------------|---------------------------------------------|
| Cloud Foundation   | Terraform      | Create EKS cluster, VPC, IAM roles          |
| Cluster Add-ons    | Terraform      | Install ingress-nginx, cert-manager         |
| Application Deploy | Argo CD/Flux   | Deploy apps into the cluster (Helm/Kustomize)|

**Workflow Integration:**
1. Terraform pipeline provisions EKS cluster.
2. Terraform **outputs** cluster endpoint and CA cert:
   ```hcl
   output "cluster_endpoint" {
     value = aws_eks_cluster.this.endpoint
   }
   ```
3. Argo CD **consumes outputs** via:
   - **Secrets Manager**: Terraform writes outputs to AWS Secrets Manager.
   - **ConfigMap**: Terraform creates a Kubernetes Secret via `null_resource`.
4. Argo CD uses these outputs to connect to the cluster and deploy apps.

**Example: Terraform → Argo CD Handoff**
```hcl
# terraform/eks/main.tf
resource "aws_eks_cluster" "main" { ... }

resource "null_resource" "argocd_bootstrap" {
  provisioner "local-exec" {
    command = <<EOT
      kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
      kubectl patch secret argocd-secret -n argocd --type='json' -p='[{"op": "add", "path": "/data/admin.password", "value": "${base64encode(random_password.admin.password)}"}]'
    EOT
  }
  depends_on = [aws_eks_cluster.main]
}
```

**Critical Considerations:**
- **State Separation**: Terraform state ≠ GitOps app state. Never let Argo CD manage Terraform state.
- **Drift Handling**: 
  - Terraform drift: Use `atlantis plan` on schedule.
  - Argo CD drift: Automatic sync or manual reconciliation.
- **Security**: 
  - Terraform runs with cloud IAM roles.
  - Argo CD uses Kubernetes RBAC (least privilege).

**When NOT to Use This:**
- For ephemeral environments (use Terraform alone).
- If infrastructure changes hourly (GitOps is for declarative app deployment).

---

### **Critical Cross-Cutting Concerns (Must Implement!)**
1. **State Security**:
   - Always use **remote state** (S3 + DynamoDB lock).
   - Encrypt state at rest (S3 SSE-KMS) and in transit (TLS).
   - Restrict state access via IAM policies.

2. **Secrets Management**:
   - **NEVER** commit secrets to Git.
   - Use **HashiCorp Vault** or cloud KMS (AWS Secrets Manager) with dynamic secrets.
   - Example: `aws_secretsmanager_secret_version.this` data source.

3. **Policy as Code**:
   - **Sentinel** (Terraform Enterprise) or **Open Policy Agent (OPA)**.
   - Enforce rules: "No public S3 buckets", "All RDS instances encrypted".

4. **Cost Control**:
   - `infracost` in PRs to show cost impact.
   - Budget alerts via cloud provider APIs.

5. **Disaster Recovery**:
   - Versioned state buckets (S3 versioning).
   - Backup state to another region weekly.

---

### **Anti-Patterns to Avoid**
| **Anti-Pattern**               | **Why It's Bad**                          | **Fix**                                  |
|--------------------------------|-------------------------------------------|------------------------------------------|
| `terraform apply` in PRs       | Bypasses review; destroys prod           | Only run `plan` on PRs                   |
| Hardcoded secrets in `.tfvars` | Secrets leaked to Git history            | Use Vault/cloud secrets                  |
| Single state file for all envs | Accidental prod changes from dev         | Separate state per environment           |
| No plan review                 | Uncaught destructive changes             | PR comments with plan output             |
| Auto-apply to production       | No human oversight for critical changes  | Strict approval gates                    |

---

### **Final Checklist for Production-Ready Terraform CI/CD**
- [ ] Remote state with locking enabled
- [ ] `terraform plan` on every PR with output in comments
- [ ] Security scanning (tfsec/checkov) in pipeline
- [ ] Manual approval required for production
- [ ] Environment promotion via identical commit hashes
- [ ] Secrets stored in Vault/KMS, not Git
- [ ] Policy checks (Sentinel/OPA) blocking non-compliant PRs
- [ ] Cost estimation in PRs (`infracost`)
- [ ] Daily drift detection in production
- [ ] Backup and DR for state files
